# Reasoning non-termination: the model thinks until the budget runs out and returns nothing

A reasoning model can spend its entire `max_tokens` budget on hidden reasoning and never write an answer. The API call still succeeds with HTTP 200. `finish_reason` is `"length"`, `content` is an empty string, and you are billed for every reasoning token.

This is easy to miss. The request did not fail, there is no exception, and a naive harness scores the row as "wrong answer" or throws it away as a parse failure. In a benchmark this quietly lowers one model's score for a reason that has nothing to do with what the benchmark claims to measure.

I hit this while running a cybersecurity knowledge benchmark (WMDP-cyber and CTIBench) on five budget-tier models through OpenRouter. All numbers below come from that run (2026-09-24). The model affected was `z-ai/glm-5.3-flash`, whose endpoint does not allow reasoning to be turned off.

## What it looks like

A pilot of 600 calls per model, all using the same prompts, temperature 0, and `max_tokens=8000`:

| Model | Reasoning | Calls hitting the 8000-token cap | Of those, with empty `content` |
|---|---|---|---|
| `z-ai/glm-5.3-flash` | mandatory (cannot disable) | **86 / 600** | **85** |
| `qwen/qwen3.8-flash` | off | 39 / 600 | 0 |
| `deepseek/deepseek-v4.1-flash` | off | 18 / 600 | 0 |
| `upstage/solar-pro4` | off | 12 / 600 | 0 |
| `openai/gpt-5.6-luna` | off | 0 / 600 | 0 |

There are two different failures in this table, and they need different fixes:

- **The three non-reasoning models** hit the cap only on WMDP-cyber's program-emulation items, such as "Which arguments will cause the following function to return 0x…". They wrote a visible step-by-step trace of 64-bit arithmetic and ran out of room. Their `content` is long, not empty. That is a task-design problem, covered at the end.
- **GLM** ran out of room in *hidden reasoning* and returned no visible text at all. It did this on ordinary knowledge questions too: 6/93 WMDP knowledge items, 14/200 CTI-MCQ items, 6/200 CTI-RCM items. That is the failure this note is about.

## Is it an infinite loop?

Not in the sense of repeating the same text. I re-ran three of the truncated prompts three times each, capturing the reasoning text, and measured how much of it repeats:

- **Verbatim repetition is low.** Between 2% and 37% of sentences are exact duplicates, and most traces sit near the bottom of that range.
- **Indecision markers are high.** Each trace has 65–111 occurrences of `Wait`, `Actually`, `Hmm` or `reconsider`.

The pattern is oscillation. The model reaches an answer and commits to it, then reopens the question. The end of one trace:

```
This aligns with answer B.

OK, I'm going with B. Final answer.

Wait, actually, I want to reconsider one more time. I've been going back and forth,
but let me think about this more carefully.
...
This is answer B.

But wait, I want to consider one more scenario. What if the instruction before the
breakpoint address is a multi-byte opcode that ...        <- budget exhausted here
```

In that single trace the model committed to an answer ("going with", "final answer", "the answer is") 11 times and never emitted one. A more accurate name than "loop" is **non-termination**. The literature also calls it overthinking. The reasoning does not converge to a stop, even when the answer is already settled.

## It is stochastic, not a budget problem

Raising the cap is the obvious fix, and it does not work reliably.

- **Same prompt, same settings, different outcomes.** Of the 9 re-runs, 3 finished normally, using 6,569–8,210 reasoning tokens. The other 6 hit the cap again. One prompt finished on 2 of 3 tries, and another on 0 of 3.
- **A doubled cap still failed.** On a separate set of three truncated prompts I raised the cap to 16,000. Two finished, at 771 and 2,102 tokens. One used all 16,000 and returned nothing again.

So a larger budget only moves the tail. The calls that finish needed at most a quarter of the original cap. The calls that don't finish would consume any budget you give them.

**Temperature 0 does not make this deterministic.** The reason is the next section.

## The provider you get changes per call

OpenRouter load-balances each request across the providers hosting a model. Its default is to pick the cheapest stable providers and "select one weighted by inverse square of the price" ([provider routing docs](https://openrouter.ai/docs/features/provider-routing)). In the 600-call pilot:

| Model | Distinct providers that served it |
|---|---|
| `z-ai/glm-5.3-flash` | **25** |
| `deepseek/deepseek-v4.1-flash` | **18** |
| Solar Pro 4, GPT-5.6 Luna, Qwen3.8 Flash | 1 (first-party only) |

Non-termination happened on every GLM provider with enough traffic to measure:

| Provider | Calls hitting the cap |
|---|---|
| InferenceNet | 35 / 221 |
| Relace | 20 / 170 |
| Wafer | 15 / 93 |
| Together | 6 / 36 |

So it is a model behaviour, not one bad deployment. But the providers differ in hardware, quantization and serving stack. "Same model, temperature 0" is therefore not the same computation twice, and a comparison between an open-weight model and a first-party one is partly a comparison between serving setups.

## Token accounting you cannot trust

In 71 of the 600 GLM rows, `usage.completion_tokens_details.reasoning_tokens` was **larger** than `usage.completion_tokens`, for example 9,173 reasoning tokens against 8,000 completion tokens. Don't write checks like `reasoning_tokens <= max_tokens`. Detect the failure from `finish_reason` and empty `content`.

OpenRouter's reasoning docs describe the expected shape of the failure:

> on most providers, the request's `max_tokens` limit applies to reasoning and visible output combined

If reasoning exhausts it, the response comes back with `finish_reason: "length"` and empty content, and the reasoning tokens are still billed ([reasoning tokens docs](https://openrouter.ai/docs/use-cases/reasoning-tokens)).

## How to handle it in an evaluation

1. **Log `finish_reason`, `provider`, and `usage` on every row.** Without these you cannot tell "wrong answer" from "no answer".
2. **Count no-answer separately and pre-register what it means.** I count it as incorrect in the headline number, because the model failed to answer within the budget every model got. I also report accuracy over answered items as a sensitivity check. Never silently drop these rows: that inflates the model's score. Never silently score them as wrong either: readers will attribute the drop to missing knowledge.
3. **Do not retry only the truncated rows.** Retrying until it finishes gives one model pass@k while the others get pass@1. Retry infrastructure failures (HTTP 429/5xx) only.
4. **Use one `max_tokens` for every model.** The cap is part of the protocol, so a per-model cap is a per-model protocol.
5. **Pin or at least record the provider.** Use `provider: {"order": [...], "allow_fallbacks": false}` or `provider: {"only": [...]}` for a fixed serving stack, or report the provider distribution next to the result.
6. **Report the non-termination rate per model and per task**, next to the accuracy table.

## How to handle it in production

- **Always check the response shape.** Treat `finish_reason == "length"` with empty `content` as a distinct failure, not an empty answer.
- **Bound reasoning, not just output.** OpenRouter's `reasoning` object accepts `effort` (`"minimal"` … `"max"`) and `max_tokens` for the reasoning budget itself, so reasoning stops early enough to leave room for the answer. `reasoning: {"enabled": true}` means medium effort, not "unbounded".
- **Keep a fallback path.** For example, retry once at lower reasoning effort or on another provider, and set a wall-clock timeout. A call that is going to run to 16,000 tokens will also take minutes.
- **Watch spend by `finish_reason`.** Every non-terminating call is billed at the full cap, so it costs the most of any call and returns nothing.

## The other truncation: the task made the model compute

The non-reasoning models' truncations had a different cause. About half of WMDP-cyber (991 of 1,987 items) comes from four templated families: "Which arguments will cause…", "What number is returned when…", "What is the output of the following…", and "I captured a network packet…". These ask the model to emulate code or decode packet bytes by hand. With a free-form prompt, even models with reasoning turned off write thousands of tokens of visible arithmetic.

The WMDP paper scores the benchmark by the top logit among A/B/C/D, zero-shot, with no generation at all ([arXiv:2403.03218](https://arxiv.org/abs/2403.03218)). A generative chain-of-thought run is therefore a different protocol. It rewards finishing long arithmetic, not knowing the answer.

For a knowledge measurement I excluded those four families. On the knowledge items in the pilot sample (93 of the remaining 996), the four non-reasoning models had **zero** truncations. GLM's non-termination remained, which shows the two problems are independent. In practice the computation items test program comprehension, which analysts do with a debugger or emulator, so they belong in a tool-using agentic evaluation rather than a closed-book one.

## Summary

- **Detect the failure from the response itself.** A reasoning model can return HTTP 200 with empty `content` and a full bill. Check `finish_reason`, not the HTTP status.
- **Don't expect a bigger cap to fix it.** Non-termination is stochastic and oscillation-driven: some calls finish in a few thousand tokens, while others consume any budget you give them.
- **Don't trust "temperature 0" on a multi-provider router.** The provider can change on every call. Pin the provider or record it.
- **In evals, make no-answer its own category.** Report it per model and per task, count it the same way for every model, and never retry it selectively.
