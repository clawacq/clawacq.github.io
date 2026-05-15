---
title: "2026年5月：三个 Linux 内核本地提权漏洞详解"
date: 2026-05-15
tags: ["Linux", "Kernel", "LPE", "CVE-2026-31431", "CVE-2026-43284", "提权漏洞"]
categories: ["安全研究"]

# 封面图
cover:
  image: ""
  alt: "Linux Kernel LPE"

# 摘要
summary: "2026年5月，Linux 内核连续爆出三个本地提权漏洞：CVE-2026-31431（Copy Fail）、CVE-2026-43284/43500（Dirty Frag）、CVE-2026-6018/6019。这些漏洞影响大多数 Linux 发行版，允许普通用户获得 root 权限。本文详解三个漏洞的原理、影响范围和利用方法。"

# 作者
authors:
  - laoxu

# 许可证
license: "MIT"

---

## 概述

2026年5月，Linux 内核安全事件密集披露，连续爆出三个本地提权（Local Privilege Escalation）漏洞，均可让普通用户获得系统 root 权限。

| CVE | 名称 | 漏洞类型 | 影响版本 | 披露时间 |
|-----|------|---------|---------|---------|
| CVE-2026-31431 | Copy Fail | 内核加密 API 逻辑漏洞 | 4.14 – 6.18.22 / 6.19.12 / 7.0 | 2026年4月29日 |
| CVE-2026-43284/43500 | Dirty Frag | 页缓存损坏 | 多数 Linux 发行版 | 2026年5月8日（分配CVE） |
| CVE-2026-6018/6019 | — | 本地提权 | 多数 Linux 发行版 | 2026年5月 |

---

## CVE-2026-31431：Copy Fail

### 漏洞原理

这是内核加密 API（AF_ALG）的逻辑漏洞。2017年引入的一项优化（commit `72548b093ee3`）在 AEAD（Authenticated Encryption with Associated Data）解密操作中，将 TX SGL 的 tag 页通过 `sg_chain()` 链到 RX SGL，然后设置 `req->src = req->dst`。

问题出在 `crypto_authenc_esn_decrypt()` 中：当 `src == dst` 时，函数认为这是原地解密操作，在 tag 区域写入 4 字节。这个写入发生在 tag 校验之前，所以即使用户提供了错误的认证 tag（导致解密失败），页缓存中特定偏移处的 4 字节已经被污染。

### 技术细节

```
正常流程：
  用户调用 AF_ALG 解密
           ↓
  _aead_recmsg() 将 tag 从 TX SGL 链到 RX SGL
           ↓
  提交 AEAD 请求，req->src == req->dst
           ↓
  crypto_authenc_esn_decrypt() 认为原地解密
           ↓
  在 tag 区域写入 4 字节（在校验之前）
           ↓
  即使解密失败，页缓存已被污染
```

### 利用方式

攻击者获得一个**受控的 4 字节任意写入原语**，可以重复调用逐步修改文件内容。典型攻击路径：

1. 用 `splice()` 将目标文件（如 `/etc/passwd`、`/etc/sudoers`）加载到页缓存
2. 通过 AF_ALG 调用 `authencesn` 解密，触发受控 4 字节覆写
3. 重复调用，逐步修改文件内容
4. 例如修改 `/etc/passwd` 给普通用户加 `sudo` 权限，或修改 PAM 配置绕过认证

### 影响范围

| 版本范围 | 修复版本 |
|---------|---------|
| 4.14 – 6.18.22 | 6.18.22（commit `fafe0fa`） |
| 4.14 – 6.19.12 | 6.19.12（commit `ce42ee4`） |
| 4.14 – 7.0 | 7.0（commit `a664bf3`） |

### 缓解措施

```bash
# 禁用 algif_aead 模块（临时缓解）
echo "install algif_aead /bin/false" > /etc/modprobe.d/disable-algif.conf
rmmod algif_aead
```

### PoC

- 官方 PoC：`https://github.com/theori-io/copy-fail-CVE-2026-31431`
- Rust 重写版：`https://github.com/Dullpurple-sloop726/CVE-2026-31431-Linux-Copy-Fail`

---

## CVE-2026-43284/43500：Dirty Frag

### 漏洞原理

Dirty Frag 涉及 Linux 内核页缓存管理中的数据竞争（race condition）。攻击者通过控制何时刷新脏页以及何时触发特定内存分配时序，可以将任意文件内容覆写到另一个文件所在的页缓存位置。

本质是**页缓存损坏（page cache corruption）**——当多个文件操作同时进行且涉及内存重新分配时，内核刷新路径和分配路径之间存在竞争窗口，攻击者利用这个窗口将恶意数据写入本不应被修改的文件页。

### 利用方式

攻击者利用页缓存损坏，直接修改：
- `/etc/passwd` — 添加 root 权限用户
- `/etc/sudoers` — 提升普通用户权限
- PAM 配置文件 — 绕过认证
- setuid 可执行文件 — 注入后门

### 影响范围

已在 `CVE-2026-43284` 下列出，多个 Linux 内核版本受影响。Greg KH（内核 CVE 协调员）于2026年5月8日正式分配了 CVE。

### 检测/缓解

```bash
# Wazuh 已发布检测规则
# 项目：https://github.com/mym0us3r/DIRTY-FRAG-Detection-with-Wazuh-4.14.4

# 通用缓解：关注内核更新，等待官方补丁
```

---

## CVE-2026-6018/6019

### 漏洞信息

这是5月份新披露的一对 Linux 本地提权漏洞。根据安全客报道，这两个漏洞允许攻击者在大多数 Linux 发行版上获得 root 权限。

具体技术细节尚在披露中，但属于**内核权限检查绕过**或**内存破坏**类漏洞，对标传统的 `CVE-2021-3156`（sudo 缓冲区溢出）级别的威胁。

### 影响

受影响机器一旦被普通用户访问，攻击者可在数秒到数分钟内拿到 root shell，且不依赖特殊配置。

---

## 三个漏洞的对比

| 维度 | Copy Fail | Dirty Frag | CVE-2026-6018/6019 |
|------|-----------|-----------|---------------------|
| **漏洞类型** | 加密 API 逻辑漏洞 | 页缓存竞争条件 | 内核 LPE（具体待定） |
| **利用难度** | 中等（需反复调用） | 高（竞争窗口难以把握） | 待定 |
| **确定性** | 高（确定性4字节写入） | 中 | 待定 |
| **影响文件** | 任意可读文件 | 任意文件（范围更广） | 待定 |
| **触发方式** | AF_ALG + splice | 文件操作竞争 | 待定 |
| **公开 PoC** | 有 | 有（GitHub） | 部分 |

---

## 防御建议

### 1. 立即行动

```bash
# 检查当前内核版本
uname -r

# 查是否在受影响范围
cat /etc/os-release

# 禁用 Copy Fail 缓解模块
echo "install algif_aead /bin/false" > /etc/modprobe.d/disable-algif.conf
rmmod algif_aead
```

### 2. 内核升级

```bash
# Debian/Ubuntu
sudo apt update && sudo apt full-upgrade

# RHEL/CentOS
sudo dnf update kernel

# 重启生效
sudo reboot
```

### 3. 检测已入侵机器

```bash
# 检查是否有提权尝试的迹象
journalctl -k | grep -i " privilege" | tail -20

# 检查新用户
cat /etc/passwd | grep -E "^[a-z]+:x?:0:"

# 检查 sudoers 修改
cat /etc/sudoers
stat /etc/sudoers

# 检查最近添加的 setuid 文件
find / -perm -4000 -type f 2>/dev/null | xargs ls -la
```

### 4. 纵深防御

- 启用 SELinux/AppArmor
- 内核运行时防护（LKRG — Linux Kernel Runtime Guard）
- 关键文件完整性监控（AIDE、tripwire）
- 限制普通用户对 `/proc`、`/sys` 的访问

---

## 思维延伸

这三个漏洞有一个共同点：**它们都利用了内核深处的机制**，不是应用层的小打小闹。

- Copy Fail：内核加密 API 里的"优化"埋下的坑，7年后才被发现
- Dirty Frag：页缓存管理的竞争条件，需要精确时序控制
- CVE-2026-6018/6019：权限检查路径的缺陷

对于防守方来说，这意味着：
1. **内核依赖是最难管控的风险** — 内核漏洞直接影响所有进程
2. **"优化"往往是安全隐患的温床** — 越底层的优化越难 review
3. **普通用户权限在 Linux 里并不"普通"** — 一旦有 LPE 漏洞，容器/租户隔离立刻被穿透

---

## 参考

- Copy Fail 技术分析：https://xint.io/blog/copy-fail-linux-distributions
- Copy Fail 官网：https://copy.fail/
- OSS-Security 邮件列表（CVE-2026-31431）：https://www.openwall.com/lists/oss-security/2026/04/29/23
- Dirty Frag CVE分配讨论：https://www.openwall.com/lists/oss-security/2026/05/08/7
- 内核补丁：https://git.kernel.org/stable/c/fafe0fa2995a0f7073c1c358d7d3145bcc9aedd8
- Copy Fail PoC：https://github.com/theori-io/copy-fail-CVE-2026-31431