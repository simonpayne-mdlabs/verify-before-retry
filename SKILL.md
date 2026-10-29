---
name: verify-before-retry
description: >
  Forensics discipline for browser automation and remote-execution tools
  whose reported results are unreliable: a blocked, filtered, timed-out, or
  "failed to dispatch" tool result does NOT mean the action didn't execute —
  and a clean-looking result doesn't prove it did. Use whenever a tool
  returns "[BLOCKED: ...]" or any content-filter/DLP redaction, reports a
  type/dispatch/timeout failure after sending input, or you are about to
  RETRY an action whose first attempt's outcome is unknown. Also use when a
  read keeps getting blocked and you're tempted to abandon the channel.
  Trigger on: "[BLOCKED", "output blocked", "tool reported failure", "action
  failed, retrying", "timed out, trying again", "can't read the result",
  "did it actually run?".
---

# Verify Before Retry — Tool Reports Are Not Ground Truth

A forensics discipline for agents driving tools whose reports can lie in
both directions: browser automation, remote shells, web IDEs, any channel
where a content filter, renderer, or acknowledgment timeout sits between
the action and the answer.

It exists because the expensive failures in this class are not caused by
tools breaking. They are caused by agents believing a tool's report about
whether something happened — and then retrying a mutation that had already
landed, or abandoning a channel that was working fine.

## Scope

Applies to any tool call that *changes something* in a target system
(typing, editing, executing, submitting) or *reads* from one through a
filterable channel. It does not apply to pure local computation, where the
return value and the outcome are the same object.

## The core model

Every automation tool call has three separable stages:

1. **Execution** — the code, keystrokes, or edit actually running in the
   target.
2. **Side effect** — the change the execution made (file edited, model
   mutated, command run, form submitted).
3. **Report** — what the tool tells you afterwards.

Filters, timeouts, and renderers sit between stages 2 and 3. They can
destroy the REPORT without touching the SIDE EFFECT — in both directions:
a "failure" report on a fully-landed action, and a garbled or blocked
report on a healthy one.

Therefore: **the report is a hint; only inspected real state is evidence of
outcome.**

## Rule 1 — A blocked result proves nothing about the side effect

Output content filters (DLP-style) block returns whose SHAPE looks
sensitive — config-like `KEY: 'value'` blocks, SQL-ish text, long raw
source. They match on the returned payload, not on what the call did.

Observed case: a mutation call (an editor-model `applyEdits`) whose
returned snippet came back as `[BLOCKED: Cookie/query string data]` had in
fact applied its edit, exactly once. The block applied to the report, not
to the execution.

**Required behaviour:** never conclude from a blocked result that the
executed code failed. Re-probe with a minimal-shape return and decide from
that.

## Rule 2 — A reported failure proves nothing either

Large inputs can exceed a client-side acknowledgment timeout while still
fully landing. Observed case: a multi-kilobyte typed input reported as
"failed / 0 of N actions dispatched" had typed completely and run
correctly.

A blind retry double-applies the input. A blind "recovery" keystroke
sequence on top of that is how misdelivery incidents start.

**Required behaviour:** after ANY reported type, dispatch, or timeout
failure, inspect actual state — screenshot, file-content check, command
output, minimal probe — BEFORE retrying. Ask: *"if the first attempt
actually landed, what would a retry do?"* If the answer is "apply it
twice," verification is mandatory, not optional.

## Rule 3 — Separate DO from VERIFY by design

Design every mutation call so its success is checkable by a SEPARATE
minimal probe, not only by its own return value. Minimal-entropy probes —
booleans, counts, lengths — pass any content filter.

```js
// Pattern; adapt to the tool at hand.
// DO:     model.applyEdits([...])            // return may be blocked
// VERIFY: content.split(anchor).length - 1   // expect exactly 2
//         content.length                     // expect old length + delta
//         /expected phrase/.test(content)    // expect true
```

The probe must distinguish **landed once** from **landed twice**. Anchor
counts do this. "Does it contain X" alone does not — it returns true in
both cases, which is precisely the case you are trying to tell apart.

## Rule 4 — Blocked reads are payload-shape problems, not channel problems

Filter blocks on pure reads are usually triggered by the payload's size or
shape, not by the channel. Observed: a 200-character slice passed where a
400-character slice of the same region blocked; a regex-extracted
function-signature map passed where raw source failed.

Escalate through reshaping before abandoning an otherwise-working channel:

1. **Smaller slices** of the same content.
2. **Extracted summaries** — signature maps, matched-line lists,
   structural outlines instead of raw dumps.
3. **Boolean or count assertions** about the content instead of the
   content itself.
4. **Non-text fallback** — scroll the target into view programmatically
   and read it from screenshots. Keep this in reserve for content the
   filter never passes.

## Pre-flight gate (enforcement)

These rules are exactly the kind skipped under retry pressure, when the
obvious move is to just run it again. Before ANY retry of a failed or
blocked action, and before declaring any blocked channel unusable, state
explicitly — in the working log or in chat, not mentally:

1. What side effect was the first attempt supposed to make?
2. What minimal probe checks whether it already happened?
3. Probe result — landed / not landed / landed twice?

Only then retry, recover, or reshape. A retry issued without a written
answer to item 3 violates this skill.

## Anti-patterns

- Retrying a "failed" type action without inspecting state → double input.
- Re-requesting the same large blocked dump verbatim → same block, wasted
  cycles. Reshape instead (Rule 4).
- Treating a mutation's own blocked return as its only success signal →
  an unverifiable action. Design the separate probe first (Rule 3).
- Concluding "the tool can't read this page" after one blocked read →
  usually payload shape. The channel is fine.
