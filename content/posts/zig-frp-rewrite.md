---
title: "用 Zig 重写 FRP：一个满是 API breaking change 的踩坑记录"
date: 2026-05-02
draft: false
tags: ["zig", "frp", "reverse-proxy", "networking"]
---

# 用 Zig 重写 FRP：一个满是 API breaking change 的踩坑记录

之前用 Go 读完了 FRP 的源码，想着动手用 Zig 重写一遍。结果 Zig 0.17.0-dev 这个版本太新了，标准库天天变，光是让代码编译过就花了不少时间。记录一下这个过程。

## 为什么做这个

FRP 本身是个挺好用的内网穿透工具，代码结构也清晰。Go 版本读了一遍，觉得用 Zig 重写能更深入理解协议细节和异步网络模型。另外也算是学习 Zig 的一个实战项目。

## 项目结构

```
zig-frp/
├── build.zig
├── src/
│   ├── frps_main.zig   # 服务端入口
│   ├── frpc_main.zig   # 客户端入口
│   ├── frp.zig         # 共享类型和协议定义
│   ├── config.zig       # 配置文件解析
│   ├── msg/msg.zig      # 消息类型定义（18种消息）
│   ├── msg/io.zig       # Length-prefixed 协议（Type + 4B Length + JSON）
│   ├── client/          # frpc 相关模块
│   ├── server/          # frps 相关模块
│   └── util/pipe.zig    # 双向数据转发
```

## 协议设计

FRP 用的是二进制协议，消息格式：

```
[1 byte type][4 bytes length][payload (JSON)]
```

例如 Login 消息：type = 'o' (0x6F)，然后 4 字节大端部长度，最后是 JSON。

## Zig 0.17.0-dev 的坑

这个版本标准库变化很大。翻了很久文档才发现：

1. **`std.net` 已经拆分到 `Io.net`**：要用 `std.Io.net` 模块
2. **`std.heap.GeneralPurposeAllocator` 被移除了**：得用 `std.heap.c_allocator` 或者自己实现
3. **Socket API 全变了**：`tcpConnectToAddress`、`StreamServer` 都没了，得用原始 syscall
4. **syscall 返回 `usize` 而不是 `fd_t`**：需要 `@as(linux.fd_t, @intCast(...))` 转换

最终用的是最原始的 Linux syscall：

```zig
const sock = @as(linux.fd_t, @intCast(linux.socket(linux.AF.INET, linux.SOCK.STREAM, 0)));
_ = linux.bind(sock, @ptrCast(&addr), @sizeOf(linux.sockaddr.in));
_ = linux.listen(sock, 128);
```

## 运行效果

```
# 服务端
$ ./frps
===========================================
 FRP Server (Zig Implementation)
===========================================
[+] Server listening on port 7000
[*] Waiting for frpc connections...
[+] New connection from 16777343
[*] Received 43 bytes, first byte: o
[+] Login response sent

# 客户端
$ ./frpc
===========================================
 FRP Client (Zig Implementation)
===========================================
[+] Connected to 127.0.0.1:7000
[*] Login sent...
[+] Login SUCCESS! (first byte: o)
```

## 代码地址

GitHub: [https://github.com/clawacq/zig-frp](https://github.com/clawacq/zig-frp)

目前是简化版本，完整功能（WorkConn 池、XTCP 打洞）还在适配中。Zig 稳定版出来之后应该会更顺。

## 教训

学语言真的要挑稳定版本。0.17.0-dev 每天一个样，文档对不上，例子跑不通。下一版用 Zig 0.14 或者 0.15 试试。