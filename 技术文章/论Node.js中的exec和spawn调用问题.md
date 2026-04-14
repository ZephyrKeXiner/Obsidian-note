---
title: 论 Node.js 中的 exec 和 spawn 调用问题
date: 2026-03-15
tags:
  - Node.js
  - 踩坑记录
  - Claude-Code
---

今天遇到了一个很奇怪的 bug，特此记录。

## 问题现象

在写一个 Node.js 脚本，使用 `exec` 直接调用 Claude 的时候，server 会直接卡住——没有输出，也没有任何错误。

我一直以为是超时问题，但即使是很短的 prompt 也会没有输出。况且直接在 TTY 中执行 `claude -p <prompt>` 很快就能出结果。同时 `stderr` 中也无任何输出。

## 排查过程

有次调试的时候，发现 Claude Code 的 exit code 为 **143**，即 `SIGTERM` 信号——说明进程是被 timeout 杀掉的，而不是自己正常退出的。

> [!tip] 关键线索
> Exit code **143** = 128 + 15（`SIGTERM`），意味着进程并非自行退出，而是被外部终止。

所以猜想是：`claude` 启动了，但一直在等待某个东西，直到超时被杀。

## 根因分析

一个正常的 CLI 工具在 `exec` 中会卡住，最常见的原因就是 **`stdin` 管道未关闭**——程序在等 EOF 信号才知道输入结束。终端会自动处理这个问题，但 `exec` 的管道不会。

> [!warning] exec 的陷阱
> `child_process.exec()` 会创建 `stdin` 管道但不会自动关闭它。如果被调用的程序从 `stdin` 读取输入，它会一直阻塞等待 EOF，永远不会结束。

## 解决方案

不再使用 `exec` 的回调方法，而是换用 `spawn` 生成 `ChildProcess` 对象，手动调用 `child.stdin.end()` 关闭输入流：

```js
const child = spawn('claude', ['-p', prompt]);
child.stdin.end();
```

这样 Node.js 中便可以顺利调用 Claude Code 了。
