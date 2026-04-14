
# A Guide to Codex's SubAgent Source Code

_I've been building my own agent CLI and recently hit the subagent implementation. Without a clear direction, I turned to Codex's source code for inspiration — here's what I found._

## Overall Architecture

Codex takes an approach inspired by Unix's `fork()` — the model calls a tool named `spawn_agent`, which triggers the Codex runtime (written in Rust as `codex-rs`) to create a new session thread. Each thread has its own isolated context window and a UUID (v1) or path address (v2). The parent thread then uses `wait_agent` to synchronously collect results.

## The Spawn Entry Point

Source location: `codex/codex-rs/core/src/agent/control.rs`, line 141.

All spawn behavior ultimately flows through `spawn_agent_internal`. Inside this function, it checks whether the new agent should inherit the parent's context, branching into either "inherit context" or "fresh context" paths.

This article focuses on the **fresh context** subagent. If you want to understand the inheritance path, I'll leave that as an exercise — honestly, I was just too lazy to write it up. One quick note: whether to inherit the parent's context is decided by the model itself, passed via the `fork_context` parameter in `SpawnAgentArgs`.

## The Fresh Context Flow

The relevant code looks like this:

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

## API Design Pattern

This is a classic pattern: provide a convenience function with fewer parameters to cover 80% of use cases, while keeping a lower-level function with the full parameter set for the remaining 20%.

After this, `spawn_thread_with_source` calls the fundamental `Codex::spawn` to instantiate a brand-new `Codex` instance.

## Aside: Why `tokio::spawn` Instead of System `fork()`

`Codex::spawn` uses the classic `tokio::spawn` for async Rust rather than a system-level `fork()`.

A system fork copies the entire process memory space, but an agent "fork" really only needs to copy (or not copy) the conversation history data structure. Using `tokio::spawn` to spin up a new async task, combined with `Arc` for communication, is a much better fit for an application like Codex.