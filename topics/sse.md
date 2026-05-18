---
tags: [web, protocol, sse, networking]
created: 2026-05-18
source: conversation
---

# 事件流与连接保活机制

> 理解 [[Server-Sent Events|SSE]]、TCP Keep-Alive、HTTP Keep-Alive 三个概念的差异与相互关系，及其在 [[LLM]] 流式推理场景中的应用。

## 背景

大模型 API 的流式输出（Streaming）是目前 SSE（Server-Sent Events）最广泛的应用场景之一。理解其底层机制和工程实现细节，有助于排查生产环境中连接超时、首 token 延迟等常见问题。

---

## 第一部分：SSE 协议

### 核心机制

SSE（Server-Sent Events，服务器推送事件）是 [[WebSocket]] 之外另一种服务端向客户端推送数据的方案：

- **协议格式**：服务端设置 `Content-Type: text/event-stream`，保持 HTTP 长连接，持续推送格式化的文本数据块
- **数据块格式**：以 `data:` 开头，每段以 `\n\n` 分隔，支持 `event`、`id`、`retry` 等字段
- **浏览器原生支持**：`EventSource` API 一行代码即可消费

### 与 WebSocket 对比

| | SSE | WebSocket |
|---|---|---|
|:--|:--|:--|
| 方向 | 服务端→客户端（单向） | 全双工 |
| 协议 | 纯 HTTP | 需要升级到 ws:// |
| 自动重连 | 浏览器内置（`Last-Event-ID`） | 手动实现 |
| 天然 API | `EventSource` | `new WebSocket()` |

### 典型场景

1. **实时通知**：新邮件、审批提醒
2. **AI 流式输出**：ChatGPT、Claude 逐字返回
3. **实时仪表盘**：股票行情、监控指标
4. **日志推送**：部署流水线实时日志
5. **进度更新**：文件处理、视频转码进度条

---

## 第二部分：LLM 场景下的 SSE 实现细节

### 流格式

OpenAI 兼容的流式 API（`stream: true`）的每个 chunk 格式：
```
data: {"id":"chatcmpl-xxx","choices":[{"delta":{"content":"你好"},"index":0}]}

data: [DONE]
```

关键差异：LLM 流式 API 使用 **POST + stream**，而非传统的 GET + EventSource。

### 客户端实现

不能用 EventSource（不支持 POST + 自定义请求头），改用 **fetch + ReadableStream**：

```javascript
const response = await fetch(url, {
  method: 'POST',
  headers: { 'Authorization': 'Bearer xxx', 'Content-Type': 'application/json' },
  body: JSON.stringify({ stream: true, ... }),
});
const reader = response.body.getReader();
```

### SSE 行解析要点

1. **跨 chunk 边界**：`\n\n` 可能被切在两个 chunk 中，需 buffer + 滑动窗口
2. **keepalive/注释行**：`:` 开头的注释行应忽略，硬解析 JSON 会报错
3. **`[DONE]` 标识**：OpenAI 用此标记结束，但有的服务端直接关闭连接
4. **错误恢复**：某行解析失败不能导致整个流中断

### 服务端实现要点

- **响应头**：`Content-Type: text/event-stream` + `Cache-Control: no-cache`
- **断连检测**：`req.on('close')` 通知上游取消推理
- **背压**：LLM token 生成速度远低于传输速度，极少出现
- **代理兼容**：nginx 默认会 buffer 整个响应，需设 `proxy_buffering off;`

### 工程坑点

| 问题 | 表象 | 原因 |
|:--|:--|:--|
| 首个 token 很慢 | 用户看到长时间空白 | 服务端先算完 KV cache 才输出 |
| 前端超时 | 连接被断开 | 思考模型长间隔无 token 输出 |
| Express buffer | 文字整段突然出现 | 未调 `res.flush()` |
|  |
| 代理吞 stream | 经过 nginx 不流了 | `proxy_buffering` 默认开启 |

---

## 第三部分：三个 Keep-Alive

### 1. HTTP Keep-Alive（连接复用）

- **层级**：HTTP 协议层
- **目的**：复用 TCP 连接处理多个请求，减少三次握手开销
- **机制**：请求头 `Connection: keep-alive`，响应头 `Keep-Alive: timeout=5, max=1000`
- **HTTP/1.1** 默认行为，**HTTP/1.0** 需要显式声明
- **SSE 的冲突**：SSE 是长连接独占，不需要复用。但 HTTP/1.1 自动开启了它，不冲突但也无意义

### 2. TCP Keep-Alive（死连接检测）

- **层级**：TCP 传输层（操作系统内核）
- **目的**：检测死连接（对端崩溃、网络断开），非"保持活跃"。
- **机制**：内核定期发无数据探测包，对面回 ACK 则活着，回 RST 则挂了，无回复则关闭连接
- **默认**：大部分 OS 关闭，开启后默认 2 小时才发第一个 probe
- **SSE 的价值**：在长思考间隔（30s+ 无数据）时，防止中间路由标记连接为 stale

### 3. 应用层心跳（SSE Ping）

- **层级**：应用层业务代码
- **目的**：保中间件不超时（nginx `proxy_read_timeout`、AWS ALB idle timeout），同时告知客户端服务端仍在运行
- **机制**：服务端在无数据时主动发空 event：`data: {"type": "ping"}` 或 `: ping`
- **配置建议**：每 15-30s 发一次，可附带 `sse-plus` 等协议扩展

### 三者对比

| | HTTP Keep-Alive | TCP Keep-Alive | 应用层心跳 |
|:--|:--|:--|:--|
| 层级 | HTTP 协议层 | TCP 传输层（内核） | 应用层代码 |
| 目的 | 复用连接 | 探测死连接 | 防中间件超时 |
| 谁发起 | 客户端/服务端协商 | 内核自动 | 服务端主动 |
| SSE 有用吗 | 无意义 | 有用（防连接存活检测） | 有用（代理超时防御） |

---

## 总结

三个 Keep-Alive 名字相同但解决的问题完全不同：

- **TCP Keep-Alive** 回答"对面还在不在"
- **HTTP Keep-Alive** 回答"这条连接还能复用吗"
- **应用层心跳** 回答"服务端的进程还在跑吗"

在 LLM 流式 SSE 场景中，正确的组合是：**短间隔有数据推不需要额外机制，长间隔（30s+）加应用层心跳，同时调大内核 tcp_keepalive_time，并通过代理层时确保 proxy_buffering 关闭**。

---

## 相关概念

[[Server-Sent Events|SSE]] · [[WebSocket]] · [[HTTP Keep-Alive]] · [[LLM]] · [[Backpressure]]