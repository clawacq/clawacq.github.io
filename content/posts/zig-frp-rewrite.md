---
title: "从零用 Zig 重写 FRP：一步一步记录（完整版）"
date: 2026-05-02
draft: false
tags: ["zig", "frp", "reverse-proxy", "networking", "tutorial"]
---

# 从零用 Zig 重写 FRP：一步一步记录

## 前言

之前写的太简陋了，重新整理一遍，记录从零搭建的完整过程。

**注意：本文基于 Zig 0.17.0-dev，API 可能随版本变化。**

## 环境准备

### 1. 安装 Zig

```bash
cd /tmp
wget https://ziglang.org/builds/zig-x86_64-linux-0.17.0-dev.224+c166c49b1.tar.xz
tar -xf zig-x86_64-linux-0.17.0-dev.224+c166c49b1.tar.xz
export PATH=/tmp/zig-x86_64-linux-0.17.0-dev.224+c166c49b1:$PATH
zig version
```

### 2. 创建项目

```bash
mkdir -p zig-frp/src/client zig-frp/src/server zig-frp/src/msg zig-frp/src/util
cd zig-frp
git init
```

## 第一步：Hello World - 最简单的 TCP Server

**目标**：验证 Zig 环境 + 理解 Linux syscall 基本用法

`src/server/main.zig`:

```zig
const std = @import("std");
const linux = std.os.linux;

const CONTROL_PORT: u16 = 7000;

pub fn main() void {
    // 创建 socket：IPv4 + TCP
    const sock = @as(linux.fd_t, @intCast(linux.socket(
        linux.AF.INET,
        linux.SOCK.STREAM,
        0
    )));
    if (sock < 0) {
        std.debug.print("[!] socket failed\n", .{});
        return;
    }
    defer _ = linux.close(sock);

    // 绑定端口
    var addr: linux.sockaddr.in = .{
        .family = linux.AF.INET,
        .port = CONTROL_PORT,
        .addr = 0,  // 0.0.0.0
        .zero = [_]u8{0} ** 8,
    };
    if (linux.bind(sock, @ptrCast(&addr), @sizeOf(linux.sockaddr.in)) < 0) {
        std.debug.print("[!] bind failed\n", .{});
        return;
    }

    // 监听
    if (linux.listen(sock, 128) < 0) {
        std.debug.print("[!] listen failed\n", .{});
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

        std.debug.print("[+] New connection from {}\n", .{client_addr.addr});
        _ = linux.close(client_fd);
    }
}
```

**编译运行**:
```bash
zig build-exe -lc -OReleaseSafe src/server/main.zig --name frps
./frps
```

**关键点**:
- `linux.socket()` 返回 `usize`，需要 `@as(linux.fd_t, @intCast(...))` 转换
- `linux.sockaddr.in` 的 `.port` 需要网络字节序（Zig 自动处理）
- 错误时返回负数，不是 error union

## 第二步：客户端连接

`src/client/main.zig`:

```zig
const std = @import("std");
const linux = std.os.linux;

const SERVER_PORT: u16 = 7000;

pub fn main() void {
    const sock = @as(linux.fd_t, @intCast(linux.socket(
        linux.AF.INET, linux.SOCK.STREAM, 0
    )));
    if (sock < 0) {
        std.debug.print("[!] socket failed\n", .{});
        return;
    }
    defer _ = linux.close(sock);

    // 连接服务器
    var addr: linux.sockaddr.in = .{
        .family = linux.AF.INET,
        .port = SERVER_PORT,
        .addr = @as(u32, 0x0100007f),  // 127.0.0.1 大端
        .zero = [_]u8{0} ** 8,
    };
    if (linux.connect(sock, @ptrCast(&addr), @sizeOf(linux.sockaddr.in)) < 0) {
        std.debug.print("[!] connect failed\n", .{});
        return;
    }

    std.debug.print("[+] Connected to 127.0.0.1:{}\n", .{SERVER_PORT});
}
```

**测试**:
```bash
zig build-exe -lc -OReleaseSafe src/client/main.zig --name frpc
./frps &  # 后台运行服务端
./frpc    # 客户端连接
```

## 第三步：自定义消息协议

FRP 用的是 Length-prefixed 协议：
- 1 字节：消息类型（如 `'o'` = Login）
- 4 字节：大端长度
- N 字节：JSON payload

`src/msg/msg.zig`:

```zig
//! 消息类型定义
pub const TypeLogin: u8 = @intCast('o');
pub const TypeLoginResp: u8 = @intCast('1');
pub const TypeNewProxy: u8 = @intCast('p');
pub const TypeNewProxyResp: u8 = @intCast('2');
pub const TypeStartWorkConn: u8 = @intCast('s');
pub const TypePing: u8 = @intCast('h');
pub const TypePong: u8 = @intCast('4');

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

pub const LoginResp = struct {
    version: []const u8 = "",
    run_id: []const u8 = "",
    error_msg: []const u8 = "",
};
```

## 第四步：发送和接收消息

添加消息 I/O 辅助函数：

```zig
//! 构造消息：类型 + 长度 + JSON
fn makeMessage(msg_type: u8, json_payload: []const u8) []u8 {
    var msg: [8192]u8 = undefined;
    msg[0] = msg_type;
    const len: u32 = @intCast(json_payload.len);
    msg[1] = @as(u8, @intCast((len >> 24) & 0xFF));
    msg[2] = @as(u8, @intCast((len >> 16) & 0xFF));
    msg[3] = @as(u8, @intCast((len >> 8) & 0xFF));
    msg[4] = @as(u8, @intCast(len & 0xFF));
    @memcpy(msg[5..5+json_payload.len], json_payload);
    return msg[0..5+json_payload.len];
}

//! 接收消息
fn recvMessage(sock: linux.fd_t) ![8192]u8 {
    var header: [5]u8 = undefined;
    const n = linux.read(sock, &header, 5);
    if (n < 5) return error.Truncated;
    
    const msg_len = @as(u32, header[1]) << 24 |
                    @as(u32, header[2]) << 16 |
                    @as(u32, header[3]) << 8 |
                    @as(u32, header[4]);
    
    var buf: [8192]u8 = undefined;
    if (msg_len > 8191) return error.TooLarge;
    
    const body_n = linux.read(sock, &buf, msg_len);
    if (body_n < msg_len) return error.Truncated;
    
    var result: [8192]u8 = undefined;
    result[0] = header[0];
    @memcpy(result[1..5], header[1..5]);
    @memcpy(result[5..5+body_n], buf[0..body_n]);
    return result;
}
```

## 第五步：实现登录握手

**客户端** - 发送 Login 并接收响应：

```zig
// 构造 Login JSON
const login_json = "{\"version\":\"0.1.0-zig\",\"hostname\":\"zig-client\"," ++
    "\"os\":\"Linux\",\"arch\":\"x86_64\",\"pool_count\":1}";

// 发送
const login_msg = makeMessage('o', login_json);
_ = linux.write(sock, @as([*]const u8, @ptrCast(login_msg.ptr)), login_msg.len);
std.debug.print("[*] Login sent...\n", .{});

// 接收响应
const resp = recvMessage(sock) catch |err| {
    std.debug.print("[!] Login response failed: {}\n", .{err});
    return;
};

if (resp[0] == '1') {  // LoginResp
    std.debug.print("[+] Login SUCCESS!\n", .{});
}
```

**服务端** - 接收 Login 并发送响应：

```zig
var buf: [8192]u8 = undefined;
const n = linux.read(fd, &buf, buf.len);
if (n > 0 and buf[0] == 'o') {
    // 解析 Login 消息...
    const login_resp = makeMessage('1', 
        "{\"version\":\"0.1.0-zig\",\"run_id\":\"test123\",\"error_msg\":\"\"}");
    _ = linux.write(fd, @as([*]const u8, @ptrCast(login_resp.ptr)), login_resp.len);
    std.debug.print("[+] Login response sent\n", .{});
}
```

## 第六步：消息分发

登录后进入主循环，处理不同类型的消息：

```zig
while (true) {
    const n = linux.read(fd, &buf, buf.len);
    if (n <= 0) break;
    
    switch (buf[0]) {
        'p' => { // NewProxy
            const resp = makeMessage('2', "{\"proxy_name\":\"ssh\",\"error_msg\":\"\"}");
            _ = linux.write(fd, @as([*]const u8, @ptrCast(resp.ptr)), resp.len);
        },
        'h' => { // Ping
            const pong = makeMessage('4', "{\"error_msg\":\"\"}");
            _ = linux.write(fd, @as([*]const u8, @ptrCast(pong.ptr)), pong.len);
        },
        else => {},
    }
}
```

## 运行结果

```bash
# 编译
zig build-exe -lc -OReleaseSafe src/server/main.zig --name frps
zig build-exe -lc -OReleaseSafe src/client/main.zig --name frpc

# 测试
./frps &
[+] Server listening on port 7000 (control)

./frpc
[+] Connected to 127.0.0.1:7000
[*] Login sent...
[+] Login SUCCESS!

# 服务端日志
[+] New connection from 16777343
[*] Received 141 bytes, first byte: o
[+] Login response sent
[*] Message: h
[*] Message: p
```

## 项目结构

```
zig-frp/
├── src/
│   ├── client/
│   │   ├── main.zig       # frpc 主程序
│   │   ├── proxy.zig      # 代理管理
│   │   ├── work_conn.zig  # WorkConn 连接池
│   │   ├── start_work_conn.zig  # StartWorkConn 处理
│   │   └── xtcp.zig       # XTCP NAT 打洞
│   ├── server/
│   │   └── main.zig        # frps 主程序
│   └── msg/
│       ├── msg.zig         # 消息类型定义
│       └── io.zig          # 消息 I/O 辅助函数
└── build.zig
```

## 踩坑总结

| 问题 | 解决 |
|------|------|
| `std.net` API 变化 | 用 `linux` syscall 替代 |
| `GeneralPurposeAllocator` 被移除 | 用简单的栈分配或全局变量 |
| `socket()` 返回 `usize` | `@as(fd_t, @intCast(...))` |
| `linux.write()` 需要 `[*]const u8` | `@as([*]const u8, @ptrCast(ptr))` |
| `linux.read()` 需要 `[*]u8` | `@as([*]u8, @ptrCast(&buf))` |
| `@intCast` 需要显式类型 | 用 `@as(u32, ...)` 显式标注 |

## 完整代码

GitHub: [https://github.com/clawacq/zig-frp-v2](https://github.com/clawacq/zig-frp-v2)

```bash
git clone https://github.com/clawacq/zig-frp-v2
cd zig-frp-v2
zig build-exe -lc -OReleaseSafe src/server/main.zig --name frps
zig build-exe -lc -OReleaseSafe src/client/main.zig --name frpc
./frps &
./frpc
```

## 下一步

- 实现 WorkConn 连接池
- 添加 TCP Proxy 端口映射
- 实现 XTCP NAT 打洞

代码在 GitHub 上持续更新。