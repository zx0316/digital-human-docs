# Task: system_memory_v1

## Goal

实现系统记忆管理模块

---

## Input

- 会话数据
- 用户输入
- 系统响应

---

## Output

- 记忆存储
- 记忆检索

---

## Dependencies

- agent/memory_spec.md
- server/session_manager.md

---

## Requirements

- 支持短期记忆
- 支持长期记忆
- 支持语义检索

---

## Non-Goals

- 不实现业务逻辑
- 不实现 LLM 调用

---

## Acceptance Criteria

- 可以存储和检索记忆
- 检索结果相关度高

---

## Trae Prompt

实现一个 Go 系统记忆管理模块：

- 输入：会话数据、用户输入、系统响应
- 输出：记忆存储、记忆检索
- 支持短期和长期记忆、语义检索
- 代码结构清晰（package + interface）

参考：
- agent/memory_spec.md