# Meta Task: system_bootstrap_v1

## 🎯 任务目标

实现数字人系统 MVP，打通端云协同基础链路：

- 客户端（C++）↔ 服务端（Go）WebSocket 双向通信
- audio_chunk 传输 → tts_chunk 返回
- 全链路流式处理能力

---

## 📁 输出要求

### 1️⃣ 客户端仓库

- **路径**：`/Users/matrix/Work/Github/digital-human-client`
- **语言**：C++

**核心模块**：
- audio/ - 音频采集
- network/ - WebSocket 通信
- player/ - 音频播放
- core/ - 会话管理
- main/ - 程序入口

### 2️⃣ 服务端仓库

- **路径**：`/Users/matrix/Work/Github/digital-human-server`
- **语言**：Go

**核心模块**：
- ws/ - WebSocket 服务器
- session/ - 会话管理
- pipeline/ - 处理管道（后续扩展）

---

## ✅ 验收标准

- 客户端能连接服务端
- 客户端持续发送 audio_chunk
- 服务端能接收并处理
- 服务端返回 tts_chunk
- 全链路稳定运行

---

## 📋 关键约束

- 严格遵守 `protocol/message_schema.json`
- 保持 streaming 优先设计
- 支持 session_id 和 trace_id
- 可中断设计（interrupt 支持）

---

## 📚 参考文档

- `architecture/overview.md` - 系统架构
- `protocol/message_schema.json` - 协议定义
- `dataflow/realtime_pipeline.md` - 数据流设计
- `client/client_architecture.md` - 客户端架构
- `server/server_architecture.md` - 服务端架构

---

## 🔄 开发流程

1. **阅读参考文档**：理解系统架构和协议设计
2. **实现客户端**：在 `digital-human-client` 实现各模块
3. **实现服务端**：在 `digital-human-server` 实现各模块
4. **联调测试**：打通端云链路
5. **提交推送**：代码提交到 GitHub

---

## 🚀 提交指南

```bash
# 客户端
cd /Users/matrix/Work/Github/digital-human-client
git add . && git commit -m "feat: MVP implementation" && git push

# 服务端
cd /Users/matrix/Work/Github/digital-human-server
git add . && git commit -m "feat: MVP implementation" && git push
```

---

## 🔮 后续任务

- `tasks/server/server_asr_stream_v1.md` - ASR 集成
- `tasks/server/server_tts_stream_v1.md` - TTS 集成
- `tasks/server/server_agent_v1.md` - Agent 实现
- `tasks/system/system_interrupt_v1.md` - 中断功能