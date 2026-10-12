# Qwen3.8-Flash-Next 双 DGX Spark 配方（维护分叉）

> 本仓库是 [MiaAI-Lab/Qwen3.8-Flash-Dual-DGX-Sparks-TensorFold](https://github.com/MiaAI-Lab/Qwen3.8-Flash-Dual-DGX-Sparks-TensorFold) 的维护分叉。
>
> 上游配方是 MiaAI-Lab 的工作，一切功劳归他们。这个分叉存在的唯一原因是：上游作者把这个账号拉黑了，下面的修复无法以 PR 形式回赠。这是不得已的维护，不是竞争性项目。
>
> 基线：上游 `6730319`（v1.1 + 撤回 PR #2，保留 v1.1 的 CUDA graph 恢复）。在此之上，本分叉恢复了两个由这里编写、但上游目前未采纳的修复：
>
> - **上下文准入**（`patches/0007-serving.patch`）：提示词加请求的输出预算等于标称上下文时予以准入，推测工作区不计入公开窗口（原上游 PR #2，后被撤回）。`tools/context_boundary.py` 一并恢复。
> - **长记录传输顺序**（`patches/0011-record-notification-first.patch`）：rank 0 先发线路通知再发控制链路长负载，发送缓冲区满时两机不再死锁（原上游 PR #4，关闭未合并）。
>
> 上游 issue [#8](https://github.com/MiaAI-Lab/Qwen3.8-Flash-Dual-DGX-Sparks-TensorFold/issues/8)（全词表采样导致两 rank 退出）**不被这些修复解决**：我们在有无这两个修复的情况下都复现了它。实测证据见 `CHANGELOG.md`。
>
> **维护承诺**：我会维护这个分叉。针对它报告的其他 bug 欢迎提出，我会尽力修复，包括我自己改动里的错误。我日常主用的是 27B 级 Qwen（`Qwen3.8-27B`）那套，所以这里的 Flash-Next 38B 配方是作为次要工作顺带维护的。

## 文档

所有安装、参数、性能数据和部署细节都以上游为准——见
[MiaAI-Lab 配方 README](https://github.com/MiaAI-Lab/Qwen3.8-Flash-Dual-DGX-Sparks-TensorFold#readme)。
本分叉不重复这些内容，只在上游 `6730319` 之上携带下面两个修复。

## 本分叉已修复的两个 bug

### 1. 上下文准入会拒绝恰好填满窗口的请求

**现象**：提示词加上请求的 `max_tokens` 恰好等于标称 `--context` 时被 `PromptTooLong` 拒绝，尽管它放得下。lane 准入把内部推测工作区计入了公开窗口，导致可用窗口比标称少 16 个 token。

**根因**：`Lanes.preparePrefill`（由 `patches/0007-serving.patch` 引入）在比较前把 `self.window`（推测行，16）加进了公开限额。

**本分叉的修复**：直接用"提示词 + 输出"对 `--context` 比较；推测行本就在引擎分配中位于窗口之外。回归测试覆盖恰好填满窗口的情况，`tools/context_boundary.py` 通过 HTTP 复核（草稿开/关、流式/非流式，16/16 通过）。

原上游 [PR #2](https://github.com/MiaAI-Lab/Qwen3.8-Flash-Dual-DGX-Sparks-TensorFold/pull/2)，合并后被上游撤回；本分叉恢复。

### 2. 长控制链路记录可能让两 rank 死锁

**现象**：超长提示词的请求（如约 262143 个输入 token）挂起，随后两 rank 在 `TF_TP_TIMEOUT_S` 看门狗超时后退出。客户端看到 SSE `LinkWriteFailed`。

**根因**：`Records.put`（`cuda_mirror.zig`）中，rank 0 在发布线路通知**之前**就发送超大记录的 TCP 负载，而接收方要等通知到达才开始读负载。发送缓冲区满时，两端互相等待。

**本分叉的修复**：先发通知，再发负载（`patches/0011-record-notification-first.patch`）。一个有界回环回归用 4 KiB 强制发送缓冲区驱动 1 MiB 记录；此前失败的 262143-token 输入随后能完成。

原上游 [PR #4](https://github.com/MiaAI-Lab/Qwen3.8-Flash-Dual-DGX-Sparks-TensorFold/pull/4)，关闭未合并；本分叉收录。

## 已知未修复 bug（此处未解决）

上游 issue [#8](https://github.com/MiaAI-Lab/Qwen3.8-Flash-Dual-DGX-Sparks-TensorFold/issues/8)：全词表采样（`top_p 1.0`、`top_k -1`）下，两 rank 在几条请求内退出——草稿/窗口抽样中的 `NucleusUncovered` 或同路径 `OutOfMemory`。我们在本分叉基线上、无论有无上述两个修复都复现了它，因此它与这两者无关、仍然开放。客户端可用显式 `top_k`/`top_p`（如 `top_k 20, top_p 0.95`）避开该路径。

---

*English version: [README.md](README.md)*
