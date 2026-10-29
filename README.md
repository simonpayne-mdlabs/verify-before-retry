# verify-before-retry

**A tool that says it failed is not evidence that it failed.**

An agent skill for browser automation, remote execution, and any other
channel where a content filter, renderer, or acknowledgment timeout sits
between an action and the report of that action.

When a tool call comes back `[BLOCKED: ...]`, or "0 of N actions
dispatched", or times out after sending a large input, the natural move is
to run it again. That move is wrong often enough to be dangerous: the
blocked or failed *report* frequently sits on top of a side effect that
landed perfectly. Retrying applies it twice.

This skill replaces the reflex with a three-question gate.

## The model

Every automation tool call has three separable stages:

| Stage | What it is |
|---|---|
| **Execution** | The code, keystrokes, or edit actually running in the target |
| **Side effect** | The change it made — file edited, command run, form submitted |
| **Report** | What the tool tells you afterwards |

Filters and timeouts sit between **side effect** and **report**. They can
destroy the report without touching the side effect — and they do it in
both directions: a failure report on a landed action, a blocked report on
a healthy one.

**The report is a hint. Only inspected real state is evidence of outcome.**

## The four rules

1. **A blocked result proves nothing about the side effect.** Content
   filters match on the shape of the returned payload, not on what the
   call did.
2. **A reported failure proves nothing either.** Large inputs exceed
   acknowledgment timeouts while still fully landing.
3. **Separate DO from VERIFY by design.** Every mutation needs a probe
   that can distinguish *landed once* from *landed twice* — anchor counts,
   not "contains X".
4. **Blocked reads are payload-shape problems, not channel problems.**
   Reshape — smaller slices, extracted summaries, boolean assertions —
   before abandoning a channel that works.

Plus a pre-flight gate that must be answered in writing before any retry.
Full detail in [`SKILL.md`](SKILL.md).

## When it triggers

`[BLOCKED` · `output blocked` · `tool reported failure` · `action failed,
retrying` · `timed out, trying again` · `can't read the result` · `did it
actually run?`

## Install

**Claude Code** — clone into your skills directory:

```
git clone https://github.com/simonpayne-mdlabs/verify-before-retry.git
```

Then move or symlink the folder into `.claude/skills/` in your project, or
into your user-level skills directory.

**Any other agent** — `SKILL.md` is plain Markdown with YAML frontmatter.
Drop it wherever your agent loads instructions from, or paste its contents
into a system prompt. There are no dependencies and nothing to run.

## Scope

Applies to tool calls that change something in a target system, or read
from one through a filterable channel. Not relevant to pure local
computation, where the return value and the outcome are the same object.

## Licence

CC BY 4.0 — share and adapt for any purpose, including commercially, with
credit. See [`LICENSE`](LICENSE).

## Author

Simon Payne — [ModusData Labs](https://mdlabs.com.au)

Issues and pull requests welcome. If you hit a failure mode in this class
that the four rules don't cover, open an issue with the observed report and
the actual outcome — that gap is the most useful thing you can send.
