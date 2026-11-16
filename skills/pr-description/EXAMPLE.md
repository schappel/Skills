# Worked example

A bug fix, at the level of detail and in the register the format expects. Reference for [`SKILL.md`](SKILL.md).

The change is two lines. The description is three short sections, and the claim that makes it most useful — what hasn't changed — is absent from the diff. The description, exactly as posted:

```markdown
## What's the change?

`AuditingChatClient.GetStreamingResponseAsync` now starts the `cachedInputTokens` and `reasoningTokens` counters at `0`, not `null`. The `+=` in the usage loop can now accumulate.

**What hasn't changed:** Only the streaming path changes. The non-streaming path reads its counts directly from `response.Usage` and is untouched. `inputTokens` and `outputTokens` were never nullable and do not change. One visible effect: a streaming turn with no cached or reasoning tokens now logs `0` where it logged `null`, so a consumer that treats `null` as "not reported" will now see zeros.


## Why are we making it?

Fixes DEV-217. `long? += long` yields `null` when the left side is `null`. The counters started at `null`, so every addition was discarded, and streaming turns always logged `null` for both. That hides prompt-cache effectiveness and reasoning cost in turn diagnostics — the numbers the prompt-caching work needs.

## How does it satisfy the requirement?

The counters are nullable only so `ReportTurn` can mark "not reported"; the accumulation never needed nullability, and seeding at `0` is what makes `+=` add rather than propagate null — the accumulated counts now reach the turn diagnostics.
```

## What the diff could not have supplied

- **What hasn't changed:** That the non-streaming path is untouched, and that `null` becoming `0` is visible to downstream consumers. Both required reading beyond the two changed lines — the non-streaming path shares `AuditingChatClient` with them, so no file list proves either claim.

## The register

Every sentence above names an artifact — a method, an operator, a value, a consumer. None is about the description itself.
