---
title: "Vercel 供应链攻击事件分析：OAuth 成为新的突破口"
date: 2026-05-15
tags: ["Vercel", "Supply Chain Attack", "OAuth", "Lumma Stealer", "Credential Leak"]
categories: ["安全研究"]

# 封面图
cover:
  image: ""
  alt: "Vercel Supply Chain Attack"

# 摘要
summary: "2026年4月19日，Vercel 披露了一起供应链安全事件。攻击者通过被 Lumma Stealer 恶意软件感染的第三方 AI 工具 Context.ai，窃取了 Google Workspace OAuth 令牌，进而渗透进 Vercel 内部系统，枚举并窃取了客户项目的环境变量。本文详解攻击链、事件时间线、平台设计问题以及对 SaaS 安全的启示。"

# 作者
authors:
  - laoxu

# 许可证
license: "MIT"

# 标签
tags: ["Vercel", "Supply Chain Attack", "OAuth", "Lumma Stealer", "Credential Leak"]

# 分类
categories: ["安全研究"]
---

## 概述

| 项目 | 内容 |
|------|------|
| **事件名称** | Vercel April 2026 Security Incident |
| **披露时间** | 2026年4月19日 |
| **初始感染** | 约2026年2月（Context.ai 员工中 Lumma Stealer） |
| **攻击者手法** | 通过第三方 OAuth 应用横向移动 |
| **影响范围** | 部分客户项目的非敏感环境变量被读取 |
| **攻击者定性** | 高复杂度，AI 加速攻击速度（CEO Rauch 原话） |
| **协同调查方** | Google Mandiant、GitHub、Microsoft、npm、Socket |

---

## 事件时间线

| 时间 | 事件 |
|------|------|
| **约2026年2月** | Context.ai 员工感染 Lumma Stealer（下载 Roblox 漏洞脚本），凭据、Session Token、OAuth Token 被窃 |
| **约2026年3月** | 攻击者进入 Context.ai 的 AWS 环境，窃取消费者用户的 OAuth Token（包括一名 Vercel 员工的 Google Workspace Token） |
| **2026年3月** | 攻击者使用窃取的 OAuth Token 进入 Vercel 员工的 Google Workspace 账号 |
| **2026年3-4月** | 攻击者从 Google Workspace 账号 pivot 到 Vercel 内部系统，开始枚举客户环境变量 |
| **2026年4月10日** | OpenAI 向某 Vercel 客户发出泄露 API Key 通知（客户报告） |
| **2026年4月19日** | Vercel 发布安全公告，CEO Rauch 在 X 上详细披露攻击链 |
| **2026年4月19日后** | 客户通知、凭证轮换指导、仪表盘安全更新 |

---

## 攻击链详解（MITRE ATT&CK）

### Stage 1：第三方 OAuth  compromise（T1199）

Context.ai 是一家 AI 可观测性工具厂商，其 Google Workspace OAuth 应用已被 Vercel 员工授权使用。

攻击路径：

```
Context.ai 员工下载 Roblox 漏洞脚本
           ↓
感染 Lumma Stealer 窃密木马
           ↓
窃取：浏览器凭据 + Session Token + Google OAuth Token
           ↓
攻击者进入 Context.ai AWS 环境
           ↓
窃取 OAuth Token（包含 Vercel 员工对应的 Token）
```

### Stage 2：OAuth 持久化访问（T1550.001）

Google OAuth 应用一旦授权，访问令牌长期有效，不需要密码，且在密码轮换后依然有效。

```
OAuth App: 110671459871-30f1spbu0hptbs60cb4vsmv79i7bbvqj.apps.googleusercontent.com
```

这个 OAuth 应用在 Vercel 员工授权后，攻击者就获得了"无需密码的长期访问权限"。

### Stage 3：横向移动到 Vercel（T1550.001）

```
使用窃取的 Google OAuth Token 登录 Google Workspace 账号
           ↓
利用 Google Workspace session 进入 Vercel 员工账号
           ↓
从员工账号 pivot 到 Vercel 内部环境
           ↓
枚举并读取客户项目环境变量
```

### Stage 4：环境变量窃取（T1213.003）

Vercel 的环境变量模型中，**未被标记为"sensitive"的值会以明文形式存储和传输**。攻击者利用内部访问权限读取了这些明文环境变量，其中常包含：

- OpenAI API Key
- 数据库凭证
- 第三方服务 Token
- 部署保护 Token

---

## 为什么影响这么大

### 1. OAuth 的"信任链"问题

这不是 Vercel 自己的漏洞，而是**第三方供应商的 OAuth 应用被攻破**，导致 Vercel 受到牵连。

现代 SaaS 环境中，企业会授权数十个第三方 OAuth 应用，这些应用持有长期有效的访问令牌，但组织通常没有集中管理这些令牌的机制。

### 2. 环境变量的"非敏感"默认值

Vercel 的设计是：**环境变量默认不加密**，只有标记为"sensitive"的才加密。

这意味着即使用户没有特意标记，任何获得 Vercel 内部访问权限的攻击者都能读取明文环境变量。

### 3. 检测-披露时间差

至少一名客户在 **4月10日** 就收到了 OpenAI 的泄露通知，而 Vercel 直到 **4月19日** 才公开披露，整整晚了 9 天。

如果那个 API Key 只存在于 Vercel，那就说明攻击者在 Vercel 披露之前就已经在黑市/野外使用了这批凭据。

---

## 平台设计的问题

| 设计决策 | 安全影响 |
|---------|---------|
| **OAuth 第三方授权** | 第三方被攻破 = 本方被入侵 |
| **环境变量默认不明文加密** | 内部访问 = 环境变量可读 |
| **非敏感环境变量可枚举** | 攻击者有内部访问权限就能列出所有值 |

---

## 2026 年 SaaS 供应链攻击模式

这不是孤例。2026 年已经发生了多起类似的**开发者平台供应链攻击**：

| 事件 | 手法 |
|------|------|
| **LiteLLM** | 包管理器供应链投毒 |
| **Axios** | npm 包被篡改 |
| **Codecov** | CI/CD 脚本被篡改 |
| **CircleCI** | 客户凭据泄露 |
| **Vercel** | 第三方 OAuth 横向移动 |

攻击者越来越倾向于攻击**开发者生态**——因为这里有大量的长期有效凭据，且平台安全往往不如生产环境严格。

---

## 防御建议

### 对 Vercel 客户

1. **立即轮换所有未标记为 sensitive 的环境变量**（API Key、数据库密码、Token 等）
2. **启用 2FA**（强制使用 Authenticator App 或 Passkey）
3. **将所有敏感凭据标记为 sensitive**
4. **审查最近部署**，删除可疑的 deployment
5. **开启活动日志监控**，检查异常访问
6. **检查部署保护 Token**，如果设置过则轮换

### 对所有 SaaS 平台用户

1. **像管理供应商一样管理 OAuth 应用** — 定期审查已授权的第三方应用，及时撤销不再使用的
2. **限制 OAuth 权限范围** — 只授权业务必需的最小权限
3. **监控 OAuth 异常登录** — 关注来自新设备/新地点的 OAuth 访问
4. **假设平台侧可能受침** — 不要在环境变量里存永久凭据，使用短期 Token 或 secret manager
5. **Google Workspace 日志保留** — 确保日志保留时间超过 6 个月，以便溯源

---

## 思维延伸

### OAuth 是新的"密码"

这件事最值得深思的一点：**OAuth Token 比密码更危险**。

密码可以改，改完就失效。但 OAuth Token 一旦授权，只要不过期就能持续使用，而且**不依赖于用户的密码轮换**。

传统的"密码安全"思维在这里完全失效——你轮换了密码，但 OAuth Token 还在。攻击者拿到 Token 后，可能在长达几个月内都畅通无阻。

### AI 加速攻击

Vercel CEO Rauch 在 X 上明确指出，攻击者表现出"非同寻常的速度和对我司产品 API 的深度理解"，暗示攻击者使用了 AI 辅助。

这意味着：
- 攻击者可以用 AI 快速学习不熟悉的平台 API surface
- 自动化地发现内部系统和环境变量枚举路径
- 将原本需要数周的情报收集工作压缩到数天

---

## IOCs（ Indicators of Compromise）

```
OAuth App Client ID: 110671459871-30f1spbu0hptbs60cb4vsmv79i7bbvqj.apps.googleusercontent.com
```

Google Workspace 管理员应立即检查是否有用户授权过这个 OAuth 应用。

---

## 参考

- Vercel 官方公告：https://vercel.com/kb/bulletin/vercel-april-2026-security-incident
- Trend Micro 分析：https://www.trendmicro.com/en_us/research/26/d/vercel-breach-oauth-supply-chain.html
- GitHub 研究仓库：https://github.com/alecccg03/Supply-Chain-Attack-Mapping
- Context.ai 安全公告：https://context.ai/security-update
- Hudson Rock 调查：https://www.infostealers.com/article/breaking-vercel-breach-linked-to-infostealer-infection-at-context-ai/