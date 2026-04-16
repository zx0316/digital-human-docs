# Task: system_interrupt_v1

## Goal

实现系统中断处理模块

---

## Input

- 客户端中断请求
- 服务端处理状态

---

## Output

- 中断处理结果
- 会话状态重置

---

## Dependencies

- dataflow/interrupt_flow.md
- server/session_manager.md

---

## Requirements

- 支持快速中断
- 支持会话状态重置
- 支持幂等性

---

## Non-Goals

- 不实现客户端检测
- 不实现业务逻辑

---

## Acceptance Criteria

- 中断响应时间 < 100ms
- 可以正确重置会话状态

---

## Trae Prompt

实现一个 Go 系统中断处理模块：

- 输入：客户端中断请求、服务端处理状态
- 输出：中断处理结果、会话状态重置
- 支持快速响应、幂等性
- 代码结构清晰（package + interface）

参考：
- dataflow/interrupt_flow.md