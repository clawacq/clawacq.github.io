---
title: "FRP 源码学习笔记（一）：入口与客户端启动流程"
date: 2026-04-29
draft: false
---

这是 FRP 源码学习笔记系列的第一篇，记录对 frp 项目的阅读和理解。

## 项目结构

```
frp-code/
├── cmd/          # frps 和 frpc 的 main 入口
├── pkg/          # 公共库（message、config、transport、util）
├── server/       # 服务端实现
├── client/       # 客户端实现
├── web/          # Dashboard 前端
├── conf/         # 默认配置示例
└── test/         # 测试相关
```

## cmd 入口

**frps**（服务端）入口极简：

```go
func main() {
    system.EnableCompatibilityMode()
    Execute()  // 实际逻辑在 root.go
}
```

`root.go` 中用 Cobra 框架解析命令行参数（`--config`、`--version` 等），然后调用 `server.NewService(cfg)` 创建服务，最后 `svr.Run(ctx)` 启动。

**frpc**（客户端）类似，但更复杂一点：
- 支持 `--config_dir` 运行多个独立实例（用于测试）
- 支持配置聚合（Aggregator）：从多个来源加载配置，实现热更新不用重启
- 最后调用 `client.NewService()` 创建客户端服务

## 客户端 Service 结构

`client/service.go` 是 frpc 的核心，管理整个生命周期：

```go
type Service struct {
    ctl       *Control          // 与 frps 的控制连接
    runID     string            // 服务器分配的客户端唯一ID
    auth      *auth.ClientAuth  // 认证相关
    webServer *http.Server      // admin API 和 Dashboard
    proxyCfgs []v1.ProxyConfigurer   // 代理配置
    visitorCfgs []v1.VisitorConfigurer // 访客配置
    // ...
}
```

关键流程：

1. **NewService** 初始化认证、加载配置、创建 admin 服务器
2. **Run** 启动：设置 DNS、启动虚拟网络、启动 admin API、连接服务器、保持重连
3. **login()** 与 frps 建立连接，发送 Login 消息，拿到 RunID

## 登录与断线重连

登录流程：
1. 创建 Connector（屏蔽 TCP/QUIC 底层差异）
2. 建立连接，发送 `msg.Login`（包含 OS、架构、hostname、版本等）
3. 等待 `LoginResp`，拿到 RunID

断线重连使用指数退避：
- 前3次重试间隔 200ms
- 之后指数增长，最大 20 秒
- 直到重新登录成功

## Control 结构

`client/control.go` 是登录成功后 frps 和 frpc 之间的控制层：

```go
type Control struct {
    sessionCtx     *SessionContext  // 连接上下文
    pm             *proxy.Manager   // 管理所有代理
    vm             *visitor.Manager // 管理访客
    msgTransporter                  // 多路复用消息
    msgDispatcher   *msg.Dispatcher // 消息分发
}
```

消息分发机制类似 HTTP/2，同一条 TCP 连接上同时处理多种消息：
- `ReqWorkConn` → 服务器请求新建工作连接
- `NewProxyResp` → 代理启动结果
- `Pong` → 心跳响应
- `NatHoleResp` → NAT 打洞响应

## 工作连接（Work Conn）的建立

frp 的代理原理核心在这里：

```
frpc 收到服务器的 ReqWorkConn 请求
  → frpc 建立新的 TCP 连接到 frps（workConn）
  → 发送 NewWorkConn 消息
  → frps 返回 StartWorkConn（包含要代理的 proxyName）
  → 把 workConn 分发给对应的 proxy 处理
```

controlConn 只传控制信令，workConn 才是真正搬运业务数据的连接。

## 下一步

接下来看：
1. `proxy.Manager` 怎么管理各个 proxy（tcp、udp、http、https、stcp 等）
2. `Connector` 具体怎么建立连接（TCP/QUIC）
3. 服务端 server 怎么接收连接和路由流量

（未完待续）
