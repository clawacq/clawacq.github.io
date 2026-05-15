---
title: "FileFix：伪装文件修复的社工攻击手法"
date: 2026-05-15
tags: ["FileFix", "Social Engineering", "Red Team", "Malware Delivery"]
categories: ["安全研究"]

# 封面图
cover:
  image: ""
  alt: "FileFix"

# 摘要
summary: "FileFix 是一种通过伪造文件修复/转换服务，诱导用户下载运行恶意文件的社工攻击手法。与 ClickFix 不同，FileFix 不要求用户手动执行命令，而是让用户主动下载并运行\"修复后\"的文件。本文详解其原理、攻击流程、与 ClickFix 的区别及防御思路。"

# 作者
authors:
  - laoxu

# 许可证
license: "MIT"

# 标签
tags: ["FileFix", "Social Engineering", "Red Team", "Malware Delivery"]

# 分类
categories: ["安全研究"]
---

## 概述

| 项目 | 内容 |
|------|------|
| **类型** | 社工攻击手法（Social Engineering） |
| **活跃时间** | 2025 年中至今 |
| **目标** | Windows 用户，企业为主 |
| **与 ClickFix 的区别** | ClickFix 让用户手动执行命令，FileFix 让用户下载运行恶意文件 |

---

## FileFix vs ClickFix

| 对比维度 | ClickFix | FileFix |
|---------|---------|---------|
| **诱导方式** | "请按 Win+R，粘贴命令，按回车" | "您的文件已损坏，点击下载修复版" |
| **用户操作** | 在 Run 对话框粘贴执行命令 | 下载文件并运行 |
| **最终手段** | 用户执行 PowerShell/cmd 命令 | 用户运行恶意程序 |
| **检测难度** | 低（用户操作看似正常） | 中（文件落地，但用户主动运行） |
| **适用场景** | 钓鱼邮件 + HTML 页面 | 钓鱼邮件 + 虚假文件转换网站 |

---

## 攻击原理

### 核心思路

FileFix 的本质是**伪造文件服务**，让用户以为自己需要"修复"或"转换"某个文件，从而主动下载攻击者提供的恶意文件。

常见场景：

```
攻击者发送钓鱼邮件："您的文档已损坏，请点击修复"
           ↓
邮件包含链接或 HTML 附件，引导用户访问"文件修复网站"
           ↓
网站显示"正在修复您的 docx 文件..."（假的）
           ↓
网站弹出"修复完成，点击下载"
           ↓
用户下载并运行"修复后的文档.exe"
           ↓
恶意软件落地
```

### 为什么能绑过安全软件

1. **用户主动下载** — 不是漏洞利用，不是脚本自动执行
2. **文件看起来正常** — 恶意载荷在"修复后文件"里，不是明显的病毒
3. **杀软对"用户自己下载运行"的检测最弱** — 这是正常业务流程
4. **文件每次不同** — 多形态（polymorphic）payload，每次下载 hash 不同

---

## 常见变种

### 变种一：虚假文档转换

伪造 Google Docs、Adobe Acrobat 风格的页面：

```
⚠️ 您的 Office 文档已过期/损坏

为防止数据丢失，请点击下方按钮下载修复版本。

[下载修复版本]
```

实际上下载的是一个 `.exe` 或 `.msi`，图标伪装成 Word/Excel。

### 变种二：邮件附件"预览版"

攻击者发送正常的邮件，但附件是一个 HTML 文件（不是 EXE/DOC）：

```
Subject: Invoice #2026 - Please Review
Attachment: invoice_preview.html
```

HTML 打开后显示：

```
此文件需要启用宏才能预览完整内容
[启用宏]
```

用户点击"启用宏"后，宏代码下载恶意 payload 并执行。

### 变种三：虚假文件解码/解压

页面显示：

```
您的文件已编码，需要解码才能打开
[解码文件]

正在解码中...
[████████░░░░░░░] 60%

解码完成！点击下方按钮下载
[下载解码后文件]
```

用户下载的是加壳的远控木马。

### 变种四：OneDrive/Google Drive 钓鱼

伪造云盘分享页面：

```
您的好友分享了一个文件给您

invoice_2026.pdf.exe

[下载] [取消]
```

用户以为是在下载 PDF，实际是一个 `.exe`（Windows 默认隐藏扩展名）。

---

## 攻击流程图

```
              钓鱼邮件/短信/社交媒体消息
                         ↓
         "您的文档已损坏，请点击修复"
                         ↓
              引导至伪造的文件修复网站
                         ↓
           显示"正在修复您的文件..."（假进度条）
                         ↓
              "修复完成，点击下载"
                         ↓
              用户下载 [file.pdf.exe]
                         ↓
              Windows 默认隐藏 .exe 扩展名
              用户以为是 .pdf，实际是 .exe
                         ↓
              用户双击运行
                         ↓
                   恶意软件落地
                         ↓
             远控/勒索/窃密/横向移动
```

---

## 真实案例

### 案例：假冒 PDF 转换服务

攻击流程：

1. 发钓鱼邮件：`Subject: Scanned Document - Action Required`
2. 邮件内容引导访问 `pdf-convert-free[.]com`
3. 网站显示"您的扫描件已上传，需要 Premium 解锁"
4. 用户点击"免费试用"→ 下载 `ScanDocument_FreeTrial.exe`
5. EXE 落地→ Venom 远控木马

### 案例：虚假宏文件

1. 钓鱼邮件带 HTML 附件 `Invoice_2026.html`
2. HTML 显示假 Excel 表格，提示"需要启用编辑才能查看"
3. 用户启用宏 → 宏代码从 C2 下载并执行 Payload
4. Cobalt Strike Beacon 落地

---

## FileFix + ClickFix 组合拳

最危险的用法：**FileFix + ClickFix 叠加**：

```
Step 1: FileFix
用户下载"修复后的文档.exe"（实际是恶意程序）

Step 2: ClickFix
EXE 运行后弹窗：
"Windows 阻止了此程序，请按 Win+R 并运行以下命令解除阻止"

用户执行命令 → 恶意程序获得更高权限
```

这让原本只能低权限运行的恶意软件，通过用户"自愿"执行命令的方式绕过 UAC。

---

## 防御思路

| 防御方向 | 具体措施 |
|---------|---------|
| **扩展名可见** | Windows 设置为显示完整扩展名（.exe 而非隐藏） |
| **邮件网关过滤** | 阻止可疑的文件下载链接，扫描 HTML 附件 |
| **下载权限控制** | 限制浏览器/邮件客户端的下载目录权限 |
| **应用白名单** | AppLocker 禁止从下载目录运行未签名程序 |
| **用户培训** | 不从邮件/网站下载"修复版"文件，直接联系发件人确认 |
| **杀软实时监控** | EDR 监控从浏览器/邮件客户端落地的新文件 |
| **UAC 提示** | 即使用户点了，运行 EXE 仍需 UAC 确认 |
| **宏禁用** | 组策略禁用来源不明文档的宏 |

---

## 检测思路（蓝队）

### 1. 文件落地检测

```powershell
# 监控下载目录新文件（用户 Downloads）
Get-ItemProperty -Path "$env:USERPROFILE\Downloads\*" |
  Where-Object { $_.LastWriteTime -gt (Get-Date).AddHours(-1) } |
  Select-Object Name, Length, LastWriteTime
```

### 2. 进程链检测

```
browser.exe → powershell.exe → network connection (download)
        ↓
   excel.exe → cmd.exe → "$env:USERPROFILE\Downloads\xxx.exe"
        ↓
   xxx.exe → network connection (C2)
```

### 3. HTML 附件宏行为

检查邮件网关是否对 HTML 附件中的以下模式报警：
- `navigator.clipboard.writeText`
- `onclick` 触发文件下载
- `mssaveOrOpenBlob` 强制下载
- `ActiveXObject` / `WScript.Shell`

### 4. 可疑文件名模式

```regex
.*\.(pdf|docx|xlsx)\.exe$
.*\_fixed\.exe$
.*\_converted\.exe$
.*Free\.exe$
```

---

## 思维延伸

FileFix 的核心是**利用用户对"文件损坏"的焦虑感**。

人在面对"文档打不开了"这种情境时，第一反应是"赶紧修复"，而不是"这是不是钓鱼"。攻击者就是抓住这个心理，伪造一个紧迫的场景，让用户放弃思考直接行动。

ClickFix 和 FileFix 的共同点是：**让用户觉得这是正常操作，而不是攻击**。安全培训如果只讲"不要打开陌生附件"已经不够了，要教会用户识别**紧急感制造**和**伪造的官方界面**。

---

## 参考

- mr.d0x / FileFix 原研究：https://github.com/mrd0x/filefix-poc
- Malwarebytes Labs：Social Engineering in 2026
- Microsoft Security Blog：Modern Phishing Techniques
- HP Wolf Security：Spotting ClickFix and FileFix Attacks