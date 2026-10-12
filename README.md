> **This is a maintained fork of [MiaAI-Lab/Qwen3.8-Flash-Dual-DGX-Sparks-TensorFold](https://github.com/MiaAI-Lab/Qwen3.8-Flash-Dual-DGX-Sparks-TensorFold).**
>
> The upstream recipe is excellent work by MiaAI-Lab, and all credit for it belongs to them. This fork exists only
> because the upstream author blocked this account, so the fixes below could not be offered back as pull requests.
> It is maintained reluctantly, not as a competing project.
>
> Base: upstream `6730319` (v1.1 + the #2 revert, with the v1.1 CUDA graph recovery kept). On top of that base this
> fork restores two fixes that were authored here but are not currently applied upstream:
>
> - **Context admission** (`patches/0007-serving.patch`): a prompt plus its requested reply budget equal to the
>   advertised context is admitted, with the speculative workspace kept outside that public window (previously
>   upstream PR #2, since reverted). `tools/context_boundary.py` is restored alongside it.
> - **Long-record transport ordering** (`patches/0011-record-notification-first.patch`): rank 0 publishes the
>   wire notification before sending a long control-link payload, so a full send buffer can no longer deadlock
>   the two ranks (previously upstream PR #4, closed unmerged).
>
> Upstream issue [#8](https://github.com/MiaAI-Lab/Qwen3.8-Flash-Dual-DGX-Sparks-TensorFold/issues/8) (both ranks
> exit under full-vocabulary sampling) is **not** fixed by these changes: we reproduced it with and without them.
> See `CHANGELOG.md` for the measured evidence.

> **Maintenance.** I intend to keep this fork working: further bugs reported against it are welcome and I will fix
> what I can, including mistakes in my own changes. My own daily driver is the 27B-class Qwen (`Qwen3.8-27B`) setup,
> so the Flash-Next 38B recipe here is maintained as a side effort rather than my primary workload.


*[中文说明](README.zh-CN.md)*

## Documentation

All setup, parameters, performance figures, and deployment details are upstream's — see the
[MiaAI-Lab recipe README](https://github.com/MiaAI-Lab/Qwen3.8-Flash-Dual-DGX-Sparks-TensorFold#readme).
This fork does not duplicate them; it only carries the two fixes below on top of upstream `6730319`.

## Bugs found and fixed in this fork

### 1. Context admission rejects a request that exactly fills the window

**Symptom.** A prompt plus its requested `max_tokens` equal to the advertised `--context` is refused with
`PromptTooLong`, even though it fits. The lane admission charged the internal speculative workspace against the
public window, so the usable window was 16 tokens smaller than advertised.

**Root cause.** In `Lanes.preparePrefill` (added by `patches/0007-serving.patch`), the check added
`self.window` (the speculative rows, 16) into the public limit before comparing with the context.

**Fix here.** Check `prompt + reply` against `--context` directly; the speculative rows already live beyond the
window in the engine's allocation. A regression test covers an exactly-full window, and `tools/context_boundary.py`
re-checks it over HTTP (drafts on/off, streamed or not, 16/16 pass).

This was upstream [PR #2](https://github.com/MiaAI-Lab/Qwen3.8-Flash-Dual-DGX-Sparks-TensorFold/pull/2), merged
then later reverted upstream; it is restored in this fork.

### 2. Long control-link records can deadlock the two ranks

**Symptom.** A request with a very long prompt (e.g. ~262143 input tokens) hangs, then both ranks exit after the
`TF_TP_TIMEOUT_S` deadline. The client sees an SSE `LinkWriteFailed`.

**Root cause.** In `Records.put` (`cuda_mirror.zig`), rank 0 sent an oversized record's TCP payload **before**
publishing its wire notification, while the receiver waits for the notification before reading the payload. A full
send buffer leaves both sides waiting on each other.

**Fix here.** Publish the notification first, then send the payload
(`patches/0011-record-notification-first.patch`). A bounded loopback regression drives a 1 MiB record through a
forced 4 KiB send buffer; the previously-failing 262143-token input then completes.

This was upstream [PR #4](https://github.com/MiaAI-Lab/Qwen3.8-Flash-Dual-DGX-Sparks-TensorFold/pull/4), closed
unmerged; it is included in this fork.

## Known open bug (not fixed here)

Upstream issue [#8](https://github.com/MiaAI-Lab/Qwen3.8-Flash-Dual-DGX-Sparks-TensorFold/issues/8): with
full-vocabulary sampling (`top_p 1.0`, `top_k -1`), both ranks exit within a few requests — `NucleusUncovered` or a
same-path `OutOfMemory` in the drafts'/windows' draws. We reproduced it on this fork's base both with and without
the two fixes above, so it is independent of them and still open. Clients can avoid that path with an explicit
`top_k`/`top_p` (e.g. `top_k 20, top_p 0.95`).
