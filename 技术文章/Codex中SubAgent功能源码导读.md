# Codex 中 SubAgent 功能源码导读

最近在写一个自己的 agent CLI，刚好实现到了 subagent 的部分。一直没有好的思路，于是去查看了 Codex 的源码获取点灵感。

## 整体架构

Codex 采用了类似于 Unix `fork()` 的思路——模型通过调用 `spawn_agent` 这个 tool，触发 Codex runtime（Rust 写的 `codex-rs`）创建一个新的 session thread，该 thread 有独立的 context window 和 UUID（v1）或者路径地址（v2），由 parent thread 的 `wait_agent` 同步收集结果。

## Spawn 的入口

源码位置：`codex/codex-rs/core/src/agent/control.rs` 第 141 行。

所有的 spawn 行为最终都会调用 `spawn_agent_internal` 函数。在这之中会判断是否需要继承父 agent 的 context，分为**继承**和**开新 context** 两个分支。

本文着重叙述开新 context 的 subagent，读者如需知晓继承细节可自行查看源码（实际上是我懒得写）。这里稍微提一嘴——是否需要继承父 agent 的 context 由模型决定，通过 `SpawnAgentArgs` 中的 `fork_context` 参数传递。

## 开新 Context 的流程

相关代码如下：

```rust
state
    .spawn_new_thread_with_source(
        config,        // 子 agent 的完整配置（model、sandbox、instructions...）
        self.clone(),
        session_source,
        /*persist_extended_history*/ false,
        /*metrics_service_name*/ None,
        inherited_shell_snapshot,
        inherited_exec_policy,
    )
    .await?
```

其中 `spawn_new_thread_with_source` 是 `spawn_thread_with_source` 的一层薄包装，为最常见的场景设计——普通 spawn，空历史，用默认 auth，没有 dynamic tools 和 tracing。

`spawn_thread_with_source` 是完整版，`fork_context: true` 的路径、需要传递 tracing context、需要 dynamic tools 的场景都使用这个函数。

> [!tip] API 设计模式
> 这是一个很常见的模式：提供一个参数少的便利函数覆盖 80% 的场景，保留一个参数全的底层函数覆盖剩下 20%。

之后便在 `spawn_thread_with_source` 中调用最基础的 `Codex::spawn` 来新生成一个 Codex 实例。

## 题外话：为什么用 `tokio::spawn` 而不是系统 `fork()`

`Codex::spawn` 底层使用的是经典的 `tokio::spawn` 实现异步 Rust，而非系统级别的 `fork()`。

系统 fork 会复制整个进程的内存空间，而 agent 的"fork"本质上只需要复制（或不复制）conversation history 这个数据结构。用 `tokio::spawn` 派生出一个新的异步 task，配合 Arc 做通信更加适合 Codex 这种应用。
