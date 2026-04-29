---
title: "FRP 源码深度解析：从门卫系统理解完整架构"
date: 2026-04-29
draft: false
---

# FRP 源码深度解析：从门卫系统理解完整架构

做安全研究二十多年，我一直觉得好项目的代码比文档有意思多了。今天拿 FRP（Fast Reverse Proxy）开刀，聊聊它的源码架构。

FRP 是什么？简单说就是：**让你能从外网访问内网机器的工具**。比如你在家想连公司服务器的 SSH，但公司网络是 NAT 的，进不去， FRP 就是那个帮你"穿墙"的工具。

---

## 运行流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           frps（服务端）                                  │
│                      监听 TCP/KCP/QUIC/WebSocket                         │
│                         端口：7000（控制连接）                           │
│                      端口：7001（HTTP）  端口：7002（TCP）               │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                                │ ① frpc 主动连接上来，建立控制连接
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           frpc（客户端）                                  │
│                                                                          │
│  1. login() ──▶ 发送 Login 消息（OS/架构/版本/runID）                     │
│                    │                                                    │
│                    ▼                                                    │
│  2. 收到 LoginResp，得到 RunID（服务器给的唯一身份）                        │
│                    │                                                    │
│                    ▼                                                    │
│  3. Start() ──▶ 预建 poolCount 个 workConn，发给 frps 放池里              │
│                    │                                                    │
│                    ▼                                                    │
│  4. 启动 proxy.Manager，把所有代理注册到 frps（NewProxy）                  │
│                    │                                                    │
│                    ▼                                                    │
│  5. 进入 keepControllerWorking() 死循环，保持连接，断了自动重连           │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                      Control 控制层                              │    │
│  │  msgDispatcher：按类型码分发消息（Login/ReqWorkConn/Pong 等）     │    │
│  │  workConnCh   ：workConn 池                                      │    │
│  │  proxy.Manager：管理所有代理（TCP/HTTP/XTCP/STCP 等）            │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                                │ 用户来了！连接 frps:7002
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         frps 收到用户连接                                 │
│                                                                          │
│  6. 从对应 proxy 的池里取 workConn（没有就发 ReqWorkConn 等 frpc 新建）    │
│                    │                                                    │
│                    ▼                                                    │
│  7. 发 StartWorkConn 给 frpc（包含 ProxyName、用户地址）                  │
│                    │                                                    │
│                    ▼                                                    │
│  8. frpc 找到对应 Proxy.InWorkConn()                                    │
│                    │                                                    │
│                    ▼                                                    │
│  9. frpc 连接本地服务 127.0.0.1:22（SSH）                                │
│                    │                                                    │
│                    ▼                                                    │
│ 10. libio.Join(userConn, workConn) ──▶ 数据双向拷贝                       │
│                                                                          │
│  ┌─────────── 整个过程中 ───────────┐                                   │
│  │  心跳维持：Ping/Pong              │                                   │
│  │  配置热更新：/api/reload         │                                   │
│  │  断线重连：指数退避，最长 20s     │                                   │
│  └──────────────────────────────────┘                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 先用一个比喻理解 FRP 的整体思路

把 FRP 想成一个**快递站**：

- **frpc**（客户端）就像快递员，他住在你家里（内网），负责把家里的东西（本地服务）打包好交给快递站
- **frps**（服务端）就是快递站，他有个公开的收货地址（公网 IP），谁想给你寄东西就送到这里
- **workConn**（工作连接）就是快递员和快递站之间的专属通道，提前建好一堆通道，要用的时候直接拿，不用临时敲门

用户要访问你家的服务，流程是这样的：

```
用户 ──▶ 快递站（frps）
          │
          │ 从池里拿一条专属通道
          ▼
      快递员（frpc）
          │
          │ 走专属通道
          ▼
      你家服务（127.0.0.1:22）
```

接下来我们看代码是怎么实现这个的。

---

## 项目结构一览

```
frp-code/
├── cmd/          # 入口，frps 和 frpc 两个 main 文件
├── pkg/          # 公共库：消息定义、配置、工具函数
├── server/       # 服务端实现
├── client/       # 客户端实现
├── web/          # Dashboard 前端
└── test/         # 测试相关
```

两条线并行：一条是 frps（服务端），一条是 frpc（客户端）。两个都是独立的进程，各自跑 `Run()` 方法。

---

## 入口：程序是怎么启动的

### frpc 客户端

```go
func main() {
    system.EnableCompatibilityMode()
    sub.Execute()
}
```

`main.go` 就一行代码，实际逻辑在 `cmd/frpc/sub/root.go` 里。用 **Cobra 命令行框架** 处理参数（`--config`、`--config_dir` 等），然后创建 `client.NewService()`，最后 `svr.Run(ctx)` 跑起来。

### frps 服务端

一样套路：`root.go` 里解析配置，调用 `server.NewService(cfg)`，然后 `svr.Run(ctx)`。

**关键点**：frps 和 frpc 的配置格式不同，但都支持 YAML/JSON/TOML，告别了老旧的 INI 格式。

---

## frpc 客户端：它是怎么工作的

### Service 是客户端的主心骨

`client/service.go` 里的 `Service` 结构管理整个客户端生命周期：

```go
type Service struct {
    ctl       *Control     // 和 frps 的控制连接
    runID     string       // 服务器给客户端发的"身份证号"
    auth      *auth.ClientAuth  // 认证模块
    webServer *http.Server // admin API 和 Dashboard
    proxyCfgs []v1.ProxyConfigurer  // 所有代理配置
    visitorCfgs []v1.VisitorConfigurer  // 访客配置
}
```

`Run()` 方法做的事：
1. 设置自定义 DNS（如果配了）
2. 启动 admin API 服务器（如果有）
3. 连接 frps，直到登录成功
4. 保持连接，断了自动重连

### 登录：拿到 runID

```go
func (svr *Service) login() (conn net.Conn, connector Connector, err error) {
    connector = svr.connectorCreator(svr.ctx, svr.common)
    connector.Open()

    conn, err = connector.Connect()

    loginMsg := &msg.Login{
        Arch: runtime.GOARCH, Os: runtime.GOOS,
        Hostname: hostname, PoolCount: svr.common.Transport.PoolCount,
        Version: version.Full(), RunID: svr.runID,
    }
    msg.WriteMsg(conn, loginMsg)

    var loginRespMsg msg.LoginResp
    msg.ReadMsgInto(conn, &loginRespMsg)
    svr.runID = loginRespMsg.RunID  // 服务器生成的唯一ID
}
```

发登录消息给服务器，服务器返回 `LoginResp`，里面有个 `RunID`。之后所有通信都用这个 ID 标识身份。

### 断线重连机制

frp 的重连不是"等断了再重连"，而是**提前准备好**：

```go
func (svr *Service) keepControllerWorking() {
    wait.BackoffUntil(func() (bool, error) {
        svr.loopLoginUntilSuccess(20*time.Second, false)
        if svr.ctl != nil {
            <-svr.ctl.Done()  // 等控制连接断开
            return false  // 继续重试
        }
        return true, nil
    }, ..., svr.ctx.Done())
}
```

指数退避策略：前几次重试间隔很短（200ms），然后指数增长，最大 20 秒。避免频繁重试把服务器打挂。

### Connector：连 frps 的三种方式

```go
type Connector interface {
    Open() error
    Connect() (net.Conn, error)
    Close() error
}
```

这是个接口，屏蔽了底层细节。frpc 支持三种连接方式：

| 模式 | 说明 | 比喻 |
|------|------|------|
| **TCP** | 每次 `Connect()` 新建一个 TCP 连接 | 每次都打电话 |
| **TCPMux**（yamux） | 一条 TCP 连接上开多个"频道" | 一根电话线接多个分机 |
| **QUIC** | 基于 UDP，多 stream，更低延迟 | 走快递专列，比 TCP 更快 |

---

## Control：frpc 的控制中枢

`client/control.go` 里的 `Control` 是登录成功后 frps 和 frpc 之间的"翻译官"：

```go
type Control struct {
    sessionCtx     *SessionContext  // 连接上下文
    pm             *proxy.Manager  // 管理所有代理
    vm             *visitor.Manager  // 管理访客
    msgTransporter                  // 多路复用（像交换机）
    msgDispatcher   *msg.Dispatcher // 消息分发
    workConnCh      chan net.Conn   // workConn 池
}
```

**收到服务器消息后的处理：**

| 消息 | 处理函数 | 干什么 |
|------|---------|--------|
| `ReqWorkConn` | `handleReqWorkConn` | 服务器要一个新连接 → 新建 workConn |
| `NewProxyResp` | `handleNewProxyResp` | 代理注册结果 |
| `Pong` | `handlePong` | 心跳响应 |
| `NatHoleResp` | `handleNatHoleResp` | NAT 打洞响应 |

**工作连接建立的流程**（这是 frp 的核心）：

```
服务器发 ReqWorkConn
  → frpc 新建一个 TCP 连接到服务器
  → 发 NewWorkConn 消息
  → 服务器返回 StartWorkConn（包含 ProxyName 等信息）
  → 把这个连接分发给对应的 Proxy 处理
```

---

## Proxy：代理是怎么工作的

`client/proxy/proxy.go` 里的 `Proxy` 接口是所有代理类型的抽象：

```go
type Proxy interface {
    Run() error
    InWorkConn(net.Conn, *msg.StartWorkConn)  // 处理工作连接
    SetInWorkConnCallback(func(...) bool)
    Close()
}
```

TCP、HTTP、HTTPS、STCP 等各种代理都实现这个接口，通过**工厂模式**注册：

```go
// init() 里注册
RegisterProxyFactory(reflect.TypeOf(&v1.TCPProxyConfig{}), NewGeneralTCPProxy)
RegisterProxyFactory(reflect.TypeOf(&v1.HTTPProxyConfig{}), NewGeneralTCPProxy)
// ...
```

**BaseProxy** 是所有代理的基类，封装了公共逻辑：

```go
func (pxy *BaseProxy) HandleTCPWorkConnection(workConn net.Conn, m *msg.StartWorkConn, encKey []byte) {
    // 1. wrapWorkConn：给连接加三层套件（限速/加密/压缩）
    remote, recycleFn, _ := pxy.wrapWorkConn(workConn, encKey)

    // 2. 连接本地服务
    localConn, _ := libnet.Dial(net.JoinHostPort(baseCfg.LocalIP, port), ...)

    // 3. 双向拷贝数据
    _, _, errs := libio.Join(localConn, remote)
}
```

**frpc 侧数据流**：

```
workConn 进来
  → 限速检查
  → 加密（如果开了）
  → 压缩（如果开了）
  → 连接本地服务（127.0.0.1:本地端口）
  → libio.Join(localConn, remote)  // 数据双向拷贝
```

---

## XTCP：P2P 直连，不走 frps

XTCP 是 frp 里最特别的一种代理模式。普通 TCP 代理所有数据都经过 frps 中转，XTCP 打了个洞，让两个 frpc **直接通信**，frps 只负责"牵线"，不中转数据。

**为什么需要它？**

想象你要访问公司内网的文件服务器。普通模式：用户 → frps → frpc → 文件服务器，数据走了两趟。如果 frps 带宽小，或者你想访问速度更快，XTCP 就能派上用场——打洞成功后，数据直接在两个 frpc 之间跑，frps 只是个信令交换中介，几乎不吃带宽。

**NAT 打洞的原理：**

内网机器要和外网机器直接通信，最大的障碍是 NAT（网络地址转换）。NAT 只认"我主动发出去的消息"的回复，不认外面主动发进来的连接。

打洞的思路是这样的：

```
Client A（提供服务）           frps（中介）         Client B（想访问）
     │                          │                      │
     │── 注册 XTCP 代理 ──────────▶│                      │
     │◀── 收到 Sid ──────────────│                      │
     │                          │◀── 注册 XTCP 代理 ───│
     │                          │─── 转发 Sid ─────────▶│
     │                          │                      │
     │ STUN 探测，拿到自己公网地址   │                      │
     │◀══════════ NAT Hole Punching ════════════════════▶│
     │      两边同时往对方的公网地址发 UDP包，在 NAT 上打出洞
     │◀═══════════════════════════════════════════════════▶│
     │         直连建立！数据不经过 frps！                   │
```

代码里（`client/proxy/xtcp.go`）分这几步：

1. `nathole.Prepare()` —— 用 STUN 服务器探测自己 NAT 出来的公网地址和端口
2. 发 `NatHoleClient` 消息给 frps，里面包含自己的地址
3. `nathole.ExchangeInfo()` —— 通过 frps 做信令交换，双方交换各自探测到的地址
4. `nathole.MakeHole()` —— **同时往对方地址发 UDP 包**，NAT 设备上会留下"外网也能进来"的映射，洞就打穿了
5. 打洞成功后，用 **QUIC** 或 **KCP** 在这条 UDP 直连上传输数据

**frps 那边只做信令中转**（`server/proxy/xtcp.go`）：`NatHoleController.ListenClient()` 监听 Sid，收到后把信息在两个客户端之间转发。

**局限性：**

- 打洞需要两边都是 UDP 可达，NAT 类型不能是对称型（Symmetric NAT）
- 如果打洞失败（比如对称型 NAT），XTCP 就用不了，还得回落到普通 TCP 模式
- 需要配置 `nat_hole_stun_server`，公开的 STUN 服务器有时候不稳定

---

## frps 服务端：它是怎么接收和处理请求的

### Service：同时监听多种协议

```go
type Service struct {
    muxer             *mux.Mux  // 多协议复用分发
    listener          net.Listener      // TCP
    kcpListener       net.Listener      // KCP（UDP）
    quicListener      *quic.Listener   // QUIC（UDP）
    websocketListener net.Listener      // WebSocket
    tlsListener       net.Listener      // TLS
    sshTunnelListener *netpkg.InternalListener  // SSH 隧道
}
```

frps 可以同时监听 TCP、KCP、QUIC、WebSocket、TLS、SSH 隧道等多种协议，连接来了通过 `mux.Mux` 分发。

### 连接来了怎么分

```go
func (svr *Service) handleConnection(conn net.Conn) {
    firstByte := make([]byte, 1)
    conn.Read(firstByte)

    switch firstByte[0] {
    case msg.TypeLogin:
        svr.RegisterControl(ctlConn, conn, false)
    case msg.TypeNewWorkConn:
        svr.RegisterWorkConn(workConn, newMsg)
    case msg.TypeNewVisitorConn:
        svr.RegisterVisitorConn(visitorConn, newMsg)
    }
}
```

根据第一个字节判断连接类型，然后分发给不同处理器。

### ControlManager：管理所有客户端

```go
type ControlManager struct {
    ctlsByRunID map[string]*Control  // runID → Control
}
```

每个 frpc 客户端登录后，frps 会创建一个 `Control` 与之对应。`runID` 是 key，可以快速找到对应的客户端会话。

**预建连接池**：`Start()` 时会提前建好 `poolCount` 个 workConn 放在池里，收到用户请求时直接取，不用等新建：

```
用户请求进来
  → frps 从池里取 workConn（如果有的话）
  → 如果池空了，发 ReqWorkConn 通知 frpc 新建
  → 立刻转发，不用等 TCP 握手
```

---

## 消息协议：frps 和 frpc 怎么聊天

所有通信都走同一个连接，用**单字符类型码**标识消息：

| 类型码 | 消息 | 谁发的 | 说明 |
|--------|------|--------|------|
| `'o'` | Login | frpc → frps | 我要登录 |
| `'1'` | LoginResp | frps → frpc | 登录结果 |
| `'p'` | NewProxy | frpc → frps | 我要暴露这个服务 |
| `'2'` | NewProxyResp | frps → frpc | 好的，端口是 xxx |
| `'r'` | ReqWorkConn | frps → frpc | 给我一条新连接 |
| `'s'` | StartWorkConn | frps → frpc | 用这条连接传这个 proxy 的数据 |
| `'w'` | NewWorkConn | frpc → frps | 新连接建好了 |
| `'h'` | Ping | frpc → frps | 心跳 |
| `'4'` | Pong | frps → frpc | 心跳响应 |

**完整用户请求流程**：

```
用户 ──▶ frps:6000（监听端口）

frps 从池里取 workConn
frps ──StartWorkConn──────▶ frpc  (ProxyName=ssh, 用户的IP和端口)
frpc ──NewWorkConn────────▶ frps  (workConn 注册完成)

frpc 找到名为 ssh 的 Proxy.InWorkConn()
  → 连接本地 127.0.0.1:22
  → 双向拷贝数据
```

---

## 插件机制：让 frp 做更多事

frps 的插件系统让你在关键节点拦截请求，做些"额外的事"：

```go
type Plugin interface {
    Name() string
    IsSupport(op string) bool
    Handle(ctx, op, content) (res, retContent, err)
}
```

**6 个可拦截的时机**：

| 操作 | 触发时机 | 用途 |
|------|---------|------|
| OpLogin | frpc 登录时 | 验证身份，拒绝非法客户端 |
| OpNewProxy | 注册新代理时 | 自动改写配置 |
| OpCloseProxy | 关闭代理时 | 记录日志 |
| OpPing | 收到心跳时 | 检查客户端状态 |
| OpNewWorkConn | 新建连接时 | 修改连接属性 |
| OpNewUserConn | 用户连进来时 | 访问控制、记录日志 |

**插件链式调用**：多个插件可以串起来用，上一个的输出是下一个的输入。插件可以拒绝请求（Reject）或者修改内容再传给下一个。

**HTTP Plugin**：插件只需暴露一个 HTTP 接口，frps 会 POST 请求过来：

```
frps ──POST /?version=0.1.0&op=NewUserConn──▶ 你的鉴权服务
        Body: {"op": "NewUserConn", "content": {"user": "Neoaler", "proxy_name": "ssh"}}
◀── Response: {"reject": false, "unchange": false}
```

---

## FRP 的特点、优势与局限

### FRP 的核心特点

作为一个开源的内网穿透工具，FRP 有几个鲜明特点：

**1. 多协议支持**
FRP 不只是 TCP 代理。HTTP/HTTPS 代理有虚拟主机路由（根据域名分发到不同内网服务），UDP 代理支持 DNS 转发，STCP/SUDP/XTCP 支持访问内网的 TCP/UDP 服务而不暴露端口到公网。

**2. 多种连接复用方式**
一条 TCP 连接上可以通过 yamux 或 QUIC 复用多个 stream，每个代理的业务数据走自己的 stream。这比每个代理都独立建一条 TCP 连接资源消耗小得多。

**3. 连接池预热**
frpc 提前建好 `pool_count` 条 workConn 放在池里，frps 需要用时直接从池里拿，用户请求几乎是零延迟。不像有些工具是等请求来了才临时建连接。

**4. 插件扩展**
frps 的 6 个操作点都可以挂插件，认证、日志、访问控制都可以在服务端统一处理，不用每个内网服务自己实现。

**5. 配置热更新**
不用重启 frpc，配置改了发个 HTTP 请求就能生效。对于需要长期运行的服务很实用。

---

### FRP 的优势

- **配置简单**：一个 YAML 文件就能跑起来，不需要复杂的依赖
- **跨平台**：Go 写的，Linux/Windows/macOS 都能跑
- **协议透明**：TCP/UDP/HTTP/HTTPS 都支持，不挑业务类型
- **开源可控**：代码量适中，架构清晰，改造成本低
- **社区活跃**：用的人多，遇到问题容易找到解决方案

---

### FRP 的局限

- **所有流量经过 frps**：这是最核心的局限。frps 必须有公网 IP，而且成了所有流量的中转站和单点
- **XTCP 打洞成功率有限**：对称型 NAT 打不了洞，只能回落 TCP 模式
- **没有原生负载均衡**：同一个 proxy 多个 frpc 节点没法定向分发流量
- **不支持多 frps 集群**：想横向扩展得自己想办法
- **HTTP 代理不够完整**：没有请求体缓存、连接复用等完整 HTTP 代理该有的功能

说到底，FRP 是个**单 frps 架构**的工具，设计初衷就不是用来做大规模分发的。想用它做高可用、大规模的内网穿透，得在它前面加一层负载均衡，或者魔改源码。

---

## FRP 的不足和可优化点

说了这么多优点，也该讲讲问题了。

### 1. 单点瓶颈：frps 是所有流量的中转站

frp 的本质是**所有流量都要经过 frps**。用户 → frps → frpc → 本地服务，原路返回。

**问题**：如果 frps 带宽有限，或者部署在低配机器上，所有代理的带宽和性能都会受影响。

**优化方向**：
- P2P 模式：对于 TCP 连接，可以尝试 NAT 打洞直连，frps 只负责建立连接，不中转数据。frp 有 `xtcp` 做这事儿，但需要两边都是主动访问者，且 NAT 类型要合适
- 分布式 frps：让 frps 支持集群，流量分散到多台机器

### 2. 心跳机制浪费资源

frpc 每隔 `heartbeat_interval` 秒发一个 Ping 消息，即使连接空闲也要发。frps 那边的超时是 `heartbeat_timeout`。

**问题**：如果配置了几千个 frpc 客户端，每秒的心跳消息量不小。

**优化方向**：
- 用连接状态检测（SO_KEEPALIVE）代替应用层心跳
- 或者改成"按需心跳"，连接空闲时不发，有数据时捎带

### 3. 配置热更新有局限

frpc 支持配置热更新（通过 `/api/reload`），但本质上是"删掉旧 proxy，建新 proxy"，不是真正的平滑切换。

**问题**：正在处理的请求会中断。

**优化方向**：借鉴 Nginx 的 reload 机制，先不关旧 worker，等现有请求处理完再关。新请求走新配置。

### 4. 缺少流量限制的细粒度控制

frp 支持带宽限制（`bandwidth_limit`），但限制维度只有"每个 proxy"，没法按用户、按 IP、按时间段来限制。

**优化方向**：增加更细粒度的流量控制和配额管理。

### 5. HTTP 代理模式不够"原生"

HTTP 类型的 proxy（`http://:80`），frps 收到请求后通过 HTTP header 里的域名路由到对应的 proxy。如果内网服务返回的 URL 是内网地址，客户端会解析失败。

frp 提供了 `host_header_rewrite` 来改写 Host 头，但某些场景下还是不够用。

### 6. 文档和错误提示

配置出错时，frp 的错误提示有时候不够明确，特别是涉及到端口冲突、权限问题的时候。

**优化方向**：改进错误消息，给出更明确的修复建议。

---

理解了这套架构，你不只是会用 frp，还能根据自己需求魔改它。源码面前，了无秘密。
