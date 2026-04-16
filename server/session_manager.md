# Session Manager

## Overview

Session Manager 负责管理客户端会话的生命周期，包括创建、更新和销毁会话。每个客户端连接对应一个会话，会话在整个对话过程中保持不变。

---

## Core Concepts

### Session

会话是对一次客户端连接的抽象，包含以下信息：

- **SessionID**：全局唯一标识，整个对话过程保持不变
- **TraceID**：一轮对话的 ID，每次用户提问都会生成新的 TraceID
- **Context**：Go context，用于取消和超时控制
- **State**：会话状态
- **CreatedAt**：会话创建时间

### Trace

Trace 是一轮对话的抽象，从用户提问到数字人回复完成：

- **TraceID**：全局唯一标识
- **SessionID**：所属会话
- **StartTime**：开始时间
- **EndTime**：结束时间

---

## Directory Structure

```
server/
├── session/
│   ├── manager.go      # Session 管理器
│   ├── session.go      # Session 实体
│   └── trace.go        # Trace 管理
```

---

## Data Structures

### Session

```go
type Session struct {
    ID        string
    TraceID   string
    State     SessionState
    CreatedAt time.Time
    Context   context.Context
    Cancel    context.CancelFunc
}

type SessionState int

const (
    StateIdle SessionState = iota
    StateRecording
    StatePlaying
    StateInterrupted
)
```

### Trace

```go
type Trace struct {
    ID        string
    SessionID string
    StartTime time.Time
    EndTime   time.Time
}
```

---

## Manager Interface

```go
type Manager interface {
    // 创建新会话
    CreateSession(conn *websocket.Conn) *Session

    // 获取会话
    GetSession(sessionID string) *Session

    // 移除会话
    RemoveSession(sessionID string)

    // 开始新一轮对话
    StartNewTrace(sessionID string) string

    // 取消当前对话
    CancelTrace(sessionID string)

    // 获取所有活跃会话
    GetActiveSessions() []*Session
}
```

---

## Session Lifecycle

### 1. 创建会话

当客户端连接时：

```go
func (m *Manager) CreateSession(conn *websocket.Conn) *Session {
    sessionID := generateUUID()
    ctx, cancel := context.WithCancel(context.Background())

    session := &Session{
        ID:        sessionID,
        TraceID:   generateUUID(),
        State:     StateIdle,
        CreatedAt: time.Now(),
        Context:   ctx,
        Cancel:    cancel,
    }

    m.sessions[sessionID] = session
    return session
}
```

### 2. 开始新对话

当用户开始新的提问时：

```go
func (m *Manager) StartNewTrace(sessionID string) string {
    session := m.sessions[sessionID]
    if session == nil {
        return ""
    }

    // 生成新的 TraceID
    session.TraceID = generateUUID()
    session.State = StateRecording

    return session.TraceID
}
```

### 3. 中断处理

当收到 interrupt 消息时：

```go
func (m *Manager) CancelTrace(sessionID string) {
    session := m.sessions[sessionID]
    if session == nil {
        return
    }

    // 取消当前 context
    session.Cancel()

    // 重置状态
    session.State = StateInterrupted

    // 生成新的 context
    session.Context, session.Cancel = context.WithCancel(context.Background())
}
```

### 4. 销毁会话

当客户端断开连接时：

```go
func (m *Manager) RemoveSession(sessionID string) {
    session := m.sessions[sessionID]
    if session != nil {
        session.Cancel()
        delete(m.sessions, sessionID)
    }
}
```

---

## Trace Management

每个 Session 维护一个 Trace 列表：

```go
type Manager struct {
    sessions map[string]*Session
    traces   map[string][]*Trace
    mu       sync.RWMutex
}

func (m *Manager) StartNewTrace(sessionID string) string {
    m.mu.Lock()
    defer m.mu.Unlock()

    session := m.sessions[sessionID]
    if session == nil {
        return ""
    }

    traceID := generateUUID()
    trace := &Trace{
        ID:        traceID,
        SessionID: sessionID,
        StartTime: time.Now(),
    }

    session.TraceID = traceID
    m.traces[sessionID] = append(m.traces[sessionID], trace)

    return traceID
}
```

---

## Context Propagation

Session 的 Context 用于传递给 Pipeline 的各个 Stage：

```go
func (m *Manager) CreatePipeline(sessionID string) *Pipeline {
    session := m.GetSession(sessionID)
    if session == nil {
        return nil
    }

    return NewPipeline(session.Context, session.TraceID)
}
```

当 interrupt 发生时：

```go
func (m *Manager) CancelTrace(sessionID string) {
    session := m.GetSession(sessionID)
    if session != nil {
        session.Cancel()
    }
}
```

---

## Thread Safety

使用读写锁保护共享数据：

```go
type Manager struct {
    sessions map[string]*Session
    mu       sync.RWMutex
}

func (m *Manager) GetSession(sessionID string) *Session {
    m.mu.RLock()
    defer m.mu.RUnlock()
    return m.sessions[sessionID]
}

func (m *Manager) RemoveSession(sessionID string) {
    m.mu.Lock()
    defer m.mu.Unlock()
    delete(m.sessions, sessionID)
}
```

---

## Implementation Checklist

- [ ] Session 创建和销毁
- [ ] Trace 管理和生命周期
- [ ] Context 传播和取消
- [ ] State 状态机转换
- [ ] Thread-safe 并发控制
- [ ] 与 Pipeline 集成
- [ ] 与 WebSocket Gateway 集成