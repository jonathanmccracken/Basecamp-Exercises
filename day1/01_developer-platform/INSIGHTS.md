# Key Insights — Building an Agentic Loop on the Claude API

## The agentic loop is just a `while` loop

```
while response.stop_reason == "tool_use":
    run the tools → append assistant turn + tool_results → call again
```

No framework needed. Claude decides the *sequence* (look up → search → resolve); your code just executes tools and shuttles messages. Frameworks like the Agent SDK are orchestration on top of this exact pattern.

## Three things that silently break the loop

1. **`tool_use_id` must pair result→call.** Each `tool_use` block has an `id`; the matching `tool_result` must echo it. Critical with parallel tool calls.
2. **Pass back the *entire* `response.content`.** With thinking enabled, the assistant turn is `[thinking, tool_use]`. Strip the thinking and you corrupt the reasoning chain.
3. **`stop_reason` is the state machine.** `"tool_use"` = keep going; `"end_turn"` = done.

## Structured output is a two-phase pattern

`output_config.format` constrains *all* text Claude emits, so it can't run during the tool loop (Claude would emit JSON instead of calling tools).

- **Phase 1:** run the tool loop unconstrained.
- **Phase 2:** one final call with `format` + `tool_choice={"type":"none"}` to force the JSON.

This phased approach reappears in the thinking and streaming versions unchanged.

## Effort is a budget; adaptive thinking is a switch

`thinking={"type":"adaptive"}` lets Claude decide *whether* to think; `output_config.effort` sets *how much*. Observed on the ambiguous TKT-1046:

| | high | low |
|---|---|---|
| Time | 34.2s | 20.3s |
| Reasoning | weighed rate-limit vs server-side, flagged 2am deploy | jumped to answer |

Route easy tickets to low effort, ambiguous ones to high — a real cost/latency lever at scale.

## Constraining inference costs

Every loop iteration is a **full billed request that re-sends the entire growing message history** — system prompt, tool schemas, all prior tool results, and thinking blocks. So cost scales with *both* the number of tool round-trips *and* the conversation length. Levers:

- **Prompt caching** — the biggest win, and missing from this exercise. The ~2k-token `SYSTEM_PROMPT` + tool schemas are identical on every iteration; mark them with `cache_control` so repeated calls hit cache instead of re-billing input tokens each round.
- **`effort` level** — thinking tokens are billed as output. High effort = more cost + latency. Route simple tickets to low.
- **`max_tokens`** — caps output cost. Note the notebook uses 32000 for the loop but only 8000 for the final structured call.
- **Tool descriptions** — vague schemas cause wrong/extra tool calls, and each call is another billed round-trip. Precise descriptions reduce iterations.
- **`tool_choice={"type":"none"}`** on the final call prevents an extra tool round-trip.
- **Batch API** — ~50% cost reduction for non-interactive ticket processing.

## Streaming = display; `get_final_message()` = control

Two events drive the UI: `content_block_start` (what's starting) and `content_block_delta` (`thinking_delta`, `text_delta`, `input_json_delta`). The loop still needs the assembled `Message` — `stream.get_final_message()` is the bridge. Tool arguments stream as **partial JSON strings**, not a parsed dict.

## The sharpest lesson came from a bug we didn't write

The low-effort TKT-1046 run noticed the ticket was *already* escalated and bailed — because `resolve_ticket` mutates a shared in-memory dict that a prior run had closed. The production lesson: **tools touch shared state, so they must be idempotent or stateless**, and re-running an agent over mutated data produces different behavior.
