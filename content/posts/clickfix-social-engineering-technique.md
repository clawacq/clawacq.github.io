---
title: "ClickFix：让用户亲手执行恶意命令的社工手法"
date: 2026-05-15
tags: ["ClickFix", "Social Engineering", "Red Team", "Defense Evasion"]
categories: ["安全研究"]

# 封面图
cover:
  image: ""
  alt: "ClickFix"

# 摘要
summary: "ClickFix 是一种通过欺骗用户手动复制粘贴执行恶意命令的社工攻击手法。近年来因绕过安全软件能力强、检测难度大而广泛流行。本文详解其原理、常见变种、利用流程和防御思路。"

# 作者
authors:
  - laoxu

# 许可证
license: "MIT"

---

## 概述

| 项目 | 内容 |
|------|------|
| **类型** | 社工攻击手法（Social Engineering） |
| **活跃时间** | 2025 年至今（越来越流行） |
| **目标** | Windows 用户，尤其企业环境 |
| **绕过安全软件** | 是（用户主动操作，安全软件不拦截） |

---

## 传统攻击 vs ClickFix

| 对比维度 | 传统攻击 | ClickFix |
|---------|---------|---------|
| **执行方式** | 恶意进程自动运行 | 用户手动 Win+R → 粘贴 → 回车 |
| **EDR 行为检测** | 触发告警 | 不触发（用户正常操作） |
| **用户参与度** | 无感 | 主动参与（以为是"修复步骤"） |
| **邮件附件** | EXE/DOC 直接触发 | HTML 引导用户操作 |
| **成功率** | 低（安全软件拦截） | 高（用户自己执行） |

---

## 攻击原理

### 核心思路

安全软件的检测逻辑：**程序自动执行可疑操作 → 报警**。

ClickFix 的核心突破：**让用户自己动手执行，而不是程序自动跑**。

```
攻击者精心设计一个钓鱼页面/附件
           ↓
引导用户"复制命令到剪贴板"
           ↓
诱导用户"按 Win+R，粘贴命令，按回车"
           ↓
用户以为是"修复步骤"，实际在执行恶意命令
           ↓
恶意进程启动，但这是用户主动操作的，安全软件不报警
```

### 为什么能绑过安全软件

EDR/杀软的检测维度：
- 进程行为异常（如 PowerShell 执行编码命令）→ 报警
- 脚本块执行恶意内容 → 报警
- **用户自己在键盘上敲命令** → **不报警**

安全软件不区分"用户自愿操作"和"程序自动化"，只要是用户自己在 Run 对话框输入的命令，系统认为是正常操作。

---

## 常见变种

### 变种一：虚假 Flash/浏览器更新（最常见）

钓鱼页面显示：

```
⚠️ 您的 Adobe Flash Player 已过期

请按以下步骤更新：
1. 按 Win+R 打开运行框
2. 复制下方命令并粘贴
3. 按回车执行

[复制命令]
```

复制到剪贴板的是：
```powershell
powershell -enc SQBFAFgAIABOAEMAOgBQAFMAVgBFAFgAIAA...
```
（base64 编码的恶意 PowerShell）

### 变种二：伪装 Google reCAPTCHA

页面嵌入一个不可见的 `reCAPTCHA` 图层，用户点击"验证"时触发恶意剪贴板操作：

```javascript
// 监听用户点击
document.addEventListener('click', function() {
    // 把恶意命令写入剪贴板
    navigator.clipboard.writeText('powershell -enc ...');
    
    // 弹出"请粘贴到运行框"提示
    alert('请按 Win+R，粘贴命令，按回车');
});
```

### 变种三：钓鱼邮件附件（HTML）

邮件附件是个 HTML 文件，打开后显示：

```
您的文档已损坏，请按以下步骤修复：
[开始修复] → 点击后恶意命令入剪贴板 → 引导用户执行
```

附件本身不是恶意文件，HTML 也不直接执行任何东西，所有"危险操作"都是用户完成的。

### 变种四：配合 CDN 静默下载

```html
<div style="display:none" onclick="copyToClipboard('powershell -enc ...')">
  <img src="x" onerror="eval(atob(base64))">
</div>
```

用户点击页面任何地方，恶意命令就被写入剪贴板。

---

## 攻击流程图

```
                    钓鱼页面/附件
                         ↓
              显示"你的软件已过期"等提示
                         ↓
            "请复制下方命令到剪贴板"
                         ↓
             用户点击 [复制] 按钮
                         ↓
           恶意命令（PowerShell）写入剪贴板
                         ↓
          "请按 Win+R，粘贴此命令，按回车"
                         ↓
              用户执行（以为是修复）
                         ↓
              PowerShell 启动并执行恶意命令
                         ↓
                   恶意软件落地
                         ↓
              远控木马/勒索软件/窃密软件
```

---

## 真实案例

### 案例一：North Korean Threat Actors（UNC4899）

朝鲜 APT 组织利用 ClickFix 攻击加密货币企业：

1. 发送钓鱼邮件，伪装成 CoinDesk 订阅确认
2. HTML 附件引导用户"修复 Adobe Flash"
3. 用户执行 PowerShell 命令 → 下载 Kimsuky 窃密木马

### 案例二：Royal勒索软件

Royal 勒索软件团队使用 ClickFix 投递勒索：

1. 伪装成软件破解工具网站
2. 显示"您需要管理员权限"，要求复制命令
3. 用户执行后下载勒索软件

### 案例三：钓鱼邮件 + HTML 附件

```html
Subject: Invoice #2026 - Payment Required
Attachment: invoice_2026.html
```

HTML 内容：
```html
<!DOCTYPE html>
<html>
<head>
  <title>Invoice</title>
</head>
<body>
  <h1>Your invoice is ready</h1>
  <p>This document requires Microsoft Office activation.</p>
  <button onclick="copy()">Copy Activation Command</button>
  <p id="msg"></p>
  <script>
  function copy() {
    navigator.clipboard.writeText('powershell -enc SQBFAFgAI...');
    document.getElementById('msg').innerText = 'Please press Win+R, paste and run.';
  }
  </script>
</body>
</html>
```

---

## 防御思路

| 防御方向 | 具体措施 |
|---------|---------|
| **用户培训** | 看到"按 Win+R 粘贴命令"直接关页面 |
| **禁用 Run 框** | 域策略禁用 Win+R 或限制可执行路径 |
| **应用白名单** | AppLocker / Windows Defender Application Control |
| **PowerShell 限制** | 限制 PowerShell 脚本执行，启用 Constrained Language Mode |
| **AMSI 监控** | 启用 PowerShell AMSI 实时监控 |
| **脚本块日志** | 开启 PowerShell 脚本块日志记录 |
| **浏览器安全** | 限制浏览器写入剪贴板的权限 |
| **EDR 调优** | 对 Win+R + PowerShell 组合添加告警 |
| **邮件过滤** | 阻止 HTML 附件或移除其中的剪贴板操作 JS |
| **网络层** | 阻断境外非必要 PowerShell 目标地址 |

---

## 检测思路（蓝队）

### 1. 进程链检测

```
winword.exe → powershell.exe → network connection
```

关键点：winword 启动 PowerShell 且有网络连接。

### 2. PowerShell 编码命令检测

```powershell
# 检测编码的 PowerShell 命令
Get-WinEvent -FilterHashtable @{LogName="Microsoft-Windows-PowerShell/Operational";ID=4104} | 
  Where-Object {$_.Message -match 'EncodedCommand'}
```

### 3. Win+R 频率监控

如果某台机器 5 分钟内 Win+R 执行超过 3 次，且每次都带 PowerShell，需告警。

### 4. HTML 附件 JS 行为检测

检查企业邮件网关是否检测 HTML 中的：
- `navigator.clipboard.writeText`
- `document.addEventListener('click'`
- `onclick` 触发剪贴板操作

---

## 思维延伸

ClickFix 的本质是**利用用户对"亲手操作"的信任感**。

安全软件的设计逻辑是：**程序自动做的事 = 可能是攻击**，但**用户自己操作 = 信任**。

这给攻防双方都提了个醒：

- **红队**：与其找 0day 绕过 EDR，不如让用户自己动手
- **蓝队**：需要重新审视"用户主动操作"的边界，不能把所有用户操作都当可信

这类手法会越来越多，因为 AI 时代攻击者可以快速生成高质量的钓鱼页面，成本极低。

---

## 参考

- Mandiant/Google TAG 报告：ClickFix Campaign
- Microsoft Security Blog：Defending Against Modern Social Engineering
- Proofpoint：ClickFix Threat Analysis