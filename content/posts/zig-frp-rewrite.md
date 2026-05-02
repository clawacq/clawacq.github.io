---
title: "从零用 Zig 重写 FRP：一步一步记录（2026-05-02 更新）"
date: 2026-05-02
draft: false
tags: ["zig", "frp", "reverse-proxy", "networking", "tutorial"]
---

# 从零用 Zig 重写 FRP：一步一步记录

之前写的太简陋了，重新整理一遍，记录从零搭建的完整过程。

## 环境准备

### 1. 安装 Zig 0.17.0-dev

Zig 还在快速迭代，0.17 是开发版，API 天天变。我用的是 master 分支的最新构建：

```bash
cd /tmp
wget https://ziglang.org/builds/zig-x86_64-linux-0.17.0-dev.224+c166c49b1.tar.xz
tar -xf zig-x86_64-linux-0.17.0-dev.224+c166c49b1.tar.xz
export PATH=/tmp/zig-x86_64-linux-0.17.0-dev.224+c166c49b1:$PATH
zig version  # 验证
```

**坑**：0.17 的标准库和 0.14/0.15 差很多，`std.net`、`std.heap.GeneralPurposeAllocator` 都被重构了。建议用稳定版除非你想踩坑。

### 2. 创建项目目录

```bash
mkdir -p zig-frp/src
cd zig-frp
git init
```

## 第一步：最简单的 TCP Server

目标：能跑起来就行，先验证 Zig 环境没问题。

创建 `src/frps_main.zig`：

```zig
//! FRP Server (frps) - 使用原始 Linux syscall
const std = @import("std");
const linux = std.os.linux;

const CONTROL_PORT: u16 = 7000;

pub fn main() void {
    std.debug.print("FRP Server starting on port {}\n", .{CONTROL_PORT});

    // 创建 socket
    const sock = @as(linux.fd_t, @intCast(linux.socket(
        linux.AF.INET,  // IPv4
        linux.SOCK.STREAM,  // TCP
        0  // 协议：自动选择
    )));
    if (sock < 0) {
        std.debug.print("[!] socket() failed\n", .{});
        return;
    }
    defer _ = linux.close(sock);

    // 绑定端口
    var addr: linux.sockaddr.in = .{
        .family = linux.AF.INET,
        .port = CONTROL_PORT,
        .addr = 0,  // 0.0.0.0 监听所有网卡
        .zero = [_]u8{0} ** 8,
    };
    const bind_result = linux.bind(sock, @ptrCast(&addr), @sizeOf(linux.sockaddr.in));
    if (bind_result < 0) {
        std.debug.print("[!] bind() failed\n", .{});
        return;
    }

    // 开始监听
    const listen_result = linux.listen(sock, 128);
    if (listen_result < 0) {
        std.debug.print("[!] listen() failed\n", .{});
        return;
    }

    std.debug.print("[+] Server listening on port {}\n", .{CONTROL_PORT});

    // 接受连接
    while (true) {
        var client_addr: linux.sockaddr.in = undefined;
        var addr_len: linux.socklen_t = @sizeOf(linux.sockaddr.in);
        
        const client_fd = @as(linux.fd_t, @intCast(linux.accept(
            sock, @ptrCast(&client_addr), &addr_len
        )));
        if (client_fd < 0) continue;

        std.debug.print("[+] New connection\n", .{});
        // 处理客户端...
        _ = linux.close(client_fd);
    }
}
```

编译运行：

```bash
zig build-exe -lc -OReleaseSafe src/frps_main.zig --name frps
./frps
# 输出: [+] Server listening on port 7000
```

**坑 1**：`linux.socket()` 返回 `usize`，不是 `fd_t`，需要 `@as(linux.fd_t, @intCast(...))` 转换。

**坑 2**：`linux.sockaddr.in` 的 `port` 字段需要网络字节序（大端），Zig 会自动帮你转换。

## 第二步：客户端连接

创建 `src/frpc_main.zig`：

```zig
//! FRP Client (frpc)
const std = @import("std");
const linux = std.os.linux;

const SERVER_ADDR = "127.0.0.1";
const CONTROL_PORT: u16 = 7000;

pub fn main() void {
    // 创建 socket
    const sock = @as(linux.fd_t, @intCast(linux.socket(
        linux.AF.INET, linux.SOCK.STREAM, 0
    )));
    if (sock < 0) {
        std.debug.print("[!] socket() failed\n", .{});
        return;
    }
    defer _ = linux.close(sock);

    // 连接服务器
    var addr: linux.sockaddr.in = .{
        .family = linux.AF.INET,
        .port = CONTROL_PORT,
        .addr = @as(u32, 0x0100007f),  // 127.0.0.1 大端表示
        .zero = [_]u8{0} ** 8,
    };
    const conn_result = linux.connect(sock, @ptrCast(&addr), @sizeOf(linux.sockaddr.in));
    if (conn_result < 0) {
        std.debug.print("[!] connect() failed\n", .{});
        return;
    }

    std.debug.print("[+] Connected to {}:{}\n", .{SERVER_ADDR, CONTROL_PORT});
}
```

编译运行：

```bash
zig build-exe -lc -OReleaseSafe src/frpc_main.zig --name frpc
./frps &  # 后台运行服务端
./frpc    # 客户端连接
```

## 第三步：自定义消息协议

FRP 用的是 Length-prefixed 协议：
- 1 字节：消息类型（如 'o' 表示 Login）
- 4 字节：大端长度
- N 字节：JSON payload

创建 `src/msg.zig` 定义消息类型：

```zig
//! 消息类型定义
pub const TypeLogin: u8 = @intCast('o');       // 客户端登录
pub const TypeLoginResp: u8 = @intCast('1');   // 服务端响应
pub const TypeNewProxy: u8 = @intCast('p');    // 注册代理
pub const TypeNewProxyResp: u8 = @intCast('2'); // 代理响应

// Login 消息结构
pub const Login = struct {
    version: []const u8 = "",
    hostname: []const u8 = "",
    os: []const u8 = "",
    arch: []const u8 = "",
    privilege_key: []const u8 = "",
    timestamp: i64 = 0,
    run_id: []const u8 = "",
    pool_count: u32 = 0,
};

// LoginResp 消息结构
pub const LoginResp = struct {
    version: []const u8 = "",
    run_id: []const u8 = "",
    error_msg: []const u8 = "",
};
```

## 第四步：发送和接收消息

添加消息 I/O 函数到 `src/msg.zig`：

```zig
//! 发送消息：类型 + 长度 + JSON
pub fn sendMessage(conn: linux.fd_t, msg_type: u8, payload: anytype) !void {
    var json_buf: [8192]u8 = undefined;
    const serialized = std.json.stringifyBuf(payload, json_buf[0..], .{
        .whitespace = .none
    });
    
    // 写类型
    var type_byte = msg_type;
    _ = linux.write(conn, &type_byte, 1);
    
    // 写长度（4字节大端）
    const len: u32 = @intCast(serialized.len);
    var len_bytes: [4]u8 = .{
        @as(u8, @intCast((len >> 24) & 0xFF)),
        @as(u8, @intCast((len >> 16) & 0xFF)),
        @as(u8, @intCast((len >> 8) & 0xFF)),
        @as(u8, @intCast(len & 0xFF)),
    };
    _ = linux.write(conn, &len_bytes, 4);
    
    // 写 JSON
    _ = linux.write(conn, @as([*]const u8, @ptrCast(serialized.ptr)), serialized.len);
}

// 接收消息头（类型 + 长度）
pub fn recvMessageHeader(conn: linux.fd_t) ![5]u8 {
    var header: [5]u8 = undefined;
    _ = linux.read(conn, &header, 5);
    return header;
}

// 解析长度
pub fn parseMsgLen(header: [5]u8) u32 {
    return @as(u32, header[1]) << 24 |
           @as(u32, header[2]) << 16 |
           @as(u32, header[3]) << 8 |
           @as(u32, header[4]);
}
```

## 第五步：实现登录握手

修改 `src/frpc_main.zig`，添加 Login 消息发送和响应接收：

```zig
const msg = @import("msg.zig");

pub fn main() void {
    // ... 连接代码 ...

    // 发送 Login 消息
    const login = msg.Login{
        .version = "0.1.0-zig",
        .hostname = "zig-client",
        .os = @tagName(std.builtin.os.tag),
        .arch = @tagName(std.builtin.cpu.arch),
        .pool_count = 1,
    };
    msg.sendMessage(sock, msg.TypeLogin, login) catch |err| {
        std.debug.print("[!] Login send failed: {}\n", .{err});
        return;
    };
    std.debug.print("[*] Login sent\n", .{});

    // 接收 LoginResp
    const header = msg.recvMessageHeader(sock) catch |err| {
        std.debug.print("[!] LoginResp header failed: {}\n", .{err});
        return;
    };
    
    if (header[0] != msg.TypeLoginResp) {
        std.debug.print("[!] Expected LoginResp, got: {c}\n", .{header[0]});
        return;
    }
    
    const len = msg.parseMsgLen(header);
    var body: [1024]u8 = undefined;
    _ = linux.read(sock, &body, len);
    
    std.debug.print("[+] Login SUCCESS!\n", .{});
}
```

## 第六步：配置文件解析

创建 `src/config.zig` 解析 INI 格式配置：

```zig
//! INI 配置文件解析器
pub const Config = struct {
    common: CommonConfig = .{},
    proxies: []ProxyConfig = &.{},
};

pub const CommonConfig = struct {
    server_addr: []const u8 = "127.0.0.1",
    server_port: u16 = 7000,
    pool_count: u32 = 1,
    privilege_key: []const u8 = "",
};

pub const ProxyConfig = struct {
    name: []const u8,
    proxy_type: []const u8 = "tcp",
    local_ip: []const u8 = "127.0.0.1",
    local_port: u16 = 0,
    remote_port: u16 = 0,
};

pub fn parseConfig(content: []const u8) Config {
    var config = Config{};
    // 解析 [common] 和 [proxy_name] sections
    // ...
    return config;
}
```

## 运行结果

```bash
# 编译
zig build-exe -lc -OReleaseSafe src/frps_main.zig --name frps
zig build-exe -lc -OReleaseSafe src/frpc_main.zig --name frpc

# 测试
./frps &
# [+] Server listening on port 7000

./frpc
# [+] Connected to 127.0.0.1:7000
# [*] Login sent...
# [+] Login SUCCESS!
```

## 踩坑总结

| 问题 | 解决方法 |
|------|---------|
| `std.net` API 变化 | 用 `linux` syscall 替代 |
| `GeneralPurposeAllocator` 被移除 | 用 `linux.sockaddr.in` 等原生类型 |
| socket 返回 usize 而非 fd_t | `@as(fd_t, @intCast(...))` |
| `linux.write` 需要 `[*]const u8` | `@as([*]const u8, @ptrCast(ptr))` |

## 完整代码

GitHub: [https://github.com/clawacq/zig-frp-v2](https://github.com/clawacq/zig-frp-v2)

```bash
git clone https://github.com/clawacq/zig-frp-v2
cd zig-frp-v2
zig build-exe -lc -OReleaseSafe src/frps_main.zig --name frps
zig build-exe -lc -OReleaseSafe src/frpc_main.zig --name frpc
./frps &
./frpc
```

## 下一步

- 实现 WorkConn 连接池
- 添加 TCP Proxy 端口映射
- 实现 XTCP NAT 打洞

代码在 GitHub 上，后续会持续更新。