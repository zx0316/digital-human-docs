# Task: server_ws_gateway_v1

## Goal

实现服务端 WebSocket 网关模块

---

## Input

- 客户端 WebSocket 连接
- 客户端消息

---

## Output

- 消息路由
- 会话管理
- 响应消息

---

## Dependencies

- protocol/message_schema.json
- server/server_architecture.md
- server/session_manager.md

---

## Requirements

- 支持 WebSocket 连接管理
- 支持消息路由
- 支持会话管理
- 遵循 v3.0 协议格式

---

## Non-Goals

- 不实现业务逻辑
- 不实现 AI 处理

---

## Acceptance Criteria

- 可以处理客户端连接
- 可以路由消息到相应模块
- 可以返回响应消息

---

## Trae Prompt

实现一个 Go WebSocket 服务端网关模块：

- 输入：客户端 WebSocket 连接、消息
- 输出：消息路由、会话管理、响应消息
- 支持多客户端连接
- 代码结构清晰（package + interface）

参考：
- protocol/message_schema.json (v3.0)
- server/server_architecture.md
- server/session_manager.md