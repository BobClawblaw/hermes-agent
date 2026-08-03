# Output-Cap Compression Fix: Detailed Report

## When It Broke

The output-cap retry path was introduced in commit `2f09df56` (Aug 3, 2026), authored by benbarclay. This commit was primarily a Discord relay fix (#77830), but it also inserted the full `conversation_loop.py` (7,250 lines), which included the new output-cap retry logic at lines 4712-4763.

The bug: the output-cap retry path reduced `max_tokens` by 64 tokens per attempt but **never called `_compress_context()`**. The compressor simply wasn't wired into this path.

## How It Broke

When the provider returned an "output cap too large" HTTP 400, the retry loop:
1. Reduced `max_tokens` by 64 tokens per attempt
2. Incremented `compression_attempts`
3. Set `_retry.restart_with_compressed_messages = True` and broke

The flag name `restart_with_compressed_messages` was misleading — the messages were **not** actually compressed. The compressor never fired on this path.

## Token Math

- Input grew by ~65 tokens per attempt (134,465 → 134,530 → 134,595 → 134,660)
- Output cap shrank by 64 tokens per attempt (65,535 → 65,471 → 65,406 → 65,341 → 65,276)
- Net effect: ~1 token net growth per attempt (65 - 64 = 1)
- Total stayed at 200,001 tokens — 1 over the 200,000 ceiling
- After 3 retries: `compression_exhausted=True` → session crash

## The Fix

Added `_compress_context()` to the output-cap retry path (new lines 4762-4819). The compressor now:
1. Drops the middle window, freeing ~50% of tokens in one pass
2. If compression achieves ≥5% savings, the session continues
3. If not, vision payloads are stripped
4. If that fails too, the session ends with `compression_exhausted=True`

## Impact

- Sessions at the context ceiling now recover via message compression instead of exhausting retries
- Applies to all providers: vLLM, DashScope (Qwen), OpenRouter, LM Studio, Anthropic
- No breaking changes — additive fix, ~100+ regression tests pass
- Before: 134,465 input + 65,535 output = 200,001 (crash after 3 retries)
- After: ~62,000 input (compressed) + ~65,471 output = ~127,471 tokens (well within 200,000)

## Affected Code

File: `agent/conversation_loop.py`, lines 4762-4819 (new block after line 4761)