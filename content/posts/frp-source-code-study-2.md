---
title: "FRP 源码学习笔记（二）：服务端架构、消息协议与插件机制"
date: 2026-04-29
draft: false
---

这是 FRP 源码学习笔记系列的第二篇，涵盖服务端启动、消息协议和插件机制。

## 服务端整体架构

`server/service.go` 是 frps 的核心入口：

```go
type Service struct {
    muxer              *mux.Mux          // 多协议复用分发器
    listener           net.Listener       // TCP 监听
    kcpListener        net.Listener      // KCP（UDP）
    quicListener       *quic.Listener    // QUIC（UDP）
    websocketListener  net.Listener      // WebSocket
    tlsListener        net.Listener      // TLS
    sshTunnelListener  *netpkg.InternalListener  // SSH 隧道

    ctlManager         *ControlManager    // 管理所有客户端会话
    clientRegistry     *registry.ClientRegistry  // 客户端注册表
    pxyManager         *proxy.Manager     // 管理所有代理
    pluginManager      *plugin.Manager     // 插件管理器
    httpVhostRouter    *vhost.Routers     // HTTP 虚拟主机路由
    rc                 *controller.ResourceController  // 资源控制器
}
```

**同时监听多种协议**：TCP、KCP、QUIC、WebSocket、TLS、SSH 隧道。核心靠 `mux.Mux` 把不同连接路由到对应处理器。

## 连接分发机制

连接到来时，先读取第一个字节判断协议类型：

```go
// handleConnection 根据首字节判断类型
switch firstByte {
case TypeLogin:
    RegisterControl()      // 客户端登录
case TypeNewWorkConn:
    RegisterWorkConn()     // frpc 注册 workConn
case TypeNewVisitorConn:
    RegisterVisitorConn()  // 访客连接
}
```

支持 **TCPMux**（yamux）复用：一条 TCP 连接上开多个 stream，每个 stream 当独立连接处理。

## Control —— 客户端会话

每个 frpc 客户端登录后，frps 会创建一个 `Control` 与之对应：

```go
type Control struct {
    sessionCtx     *SessionContext  // 连接上下文
    workConnCh     chan net.Conn   // workConn 池
    proxies        map[string]proxy.Proxy  // 这个客户端的代理
    poolCount      int             // 连接池大小
    msgDispatcher  *msg.Dispatcher // 消息分发
}
```

**Start()** 时会预建 `poolCount` 个 workConn 发给 frpc，放入池中备用。

**GetWorkConn()** —— 用户请求来了，frps 从池里取一个 workConn，池空则发 `ReqWorkConn` 等 frpc 新建。

## 消息协议

frps 和 frpc 之间所有通信都走同一条连接（TCP/yamux/QUIC），用单字符类型码标识消息：

| 类型码 | 消息 | 方向 | 说明 |
|--------|------|------|------|
| `'o'` | Login | → | 登录 |
| `'1'` | LoginResp | ← | 登录结果 |
| `'p'` | NewProxy | → | 注册代理 |
| `'2'` | NewProxyResp | ← | 注册结果 |
| `'r'` | ReqWorkConn | ← | 服务器请求新建 workConn |
| `'s'` | StartWorkConn | ← | 告诉 frpc 这个 workConn 给哪个 proxy |
| `'w'` | NewWorkConn | → | frpc 完成 workConn 注册 |
| `'h'` | Ping | → | 心跳 |
| `'4'` | Pong | ← | 心跳响应 |

**完整用户请求流程：**

```
用户 ──▶ frps:6000

frps 从池里取 workConn
frps ──StartWorkConn──────▶ frpc  (ProxyName, SrcAddr, DstAddr)
frpc ──NewWorkConn────────▶ frps  (完成注册)
frpc InWorkConn(workConn, msg)
  连接本地服务 127.0.0.1:22
  libio.Join(userConn, workConn)  // 双向拷贝
```

## 插件机制

frps 的插件系统让你在关键节点拦截和修改请求。接口只有三个方法：

```go
type Plugin interface {
    Name() string
    IsSupport(op string) bool
    Handle(ctx, op, content) (res, retContent, err)
}
```

**6 个可拦截的操作点：**

| Op | 触发时机 |
|----|---------|
| OpLogin | frpc 登录时 |
| OpNewProxy | frpc 注册新代理时 |
| OpCloseProxy | frpc 关闭代理时 |
| OpPing | 收到心跳时 |
| OpNewWorkConn | frpc 新建 workConn 时 |
| OpNewUserConn | 用户连接进来时 |

**插件链式执行**：所有插件串行调用，上一个的输出是下一个的输入。插件可以：
- **Reject**：拒绝请求
- **修改 content**：把修改后的内容传给下一个插件

**HTTP Plugin**：插件只需实现 HTTP 接口，frps 会发 POST 请求到你的插件服务：

```
frps ──POST /?version=0.1.0&op=NewProxy──▶ 你的插件服务
        Body: {"version": "0.1.0", "op": "NewProxy", "content": {...}}
◀── Response: {"reject": false, "unchange": false, "content": {...}}
```

## 关键设计思路

**1. 连接池预建**
frpc 提前建好 workConn 放入池中，frps 需要用时直接取，零延迟响应用户请求。

**2. 多路复用**
- 单 TCP 连接上通过 yamux/QUIC 复用多个 stream
- msgDispatcher 按类型码分发消息到不同 handler
- 避免每个代理都建一条独立连接

**3. 插件链式拦截**
- 统一的 Plugin 接口，支持任意实现（HTTP、Go 直接注册等）
- 链式调用支持请求修改和拒绝
- 插件可以组合使用，各司其职

（未完待续）
