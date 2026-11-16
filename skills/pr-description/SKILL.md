---
name: pr-description
description: Write a pull request description. Use when opening a PR, writing or updating a PR description, or when asked what to put in a PR body.
---

# PR descriptions

A PR description is a **claim the reviewer can falsify against the diff**. Its job is to let a reviewer — human or agent — find where the code diverges from what was intended.

That matters most when the code was written by an agent. A large generated diff no longer evidences intent: plausible code may be sitting in it that nobody actually decided on, and review degrades from *does this do what was intended, and only that?* to *does this look reasonable?* — which generated code passes easily.

The diff already shows what changed. The description says what was **meant**.

## Writing one is a conversation

Draft what the work supports. **Ask the author for everything else.** Their answers are the parts a reviewer most needs and the parts no diff contains — what was actually run, where the risk sits, which requirement this serves.

Every sentence in the finished description comes from the work or from the author. The only placeholders that survive are for artifacts the author must attach — screenshots, video, metrics.

With no author to ask — an agent running autonomously against a bug report — follow [AUTONOMOUS.md](AUTONOMOUS.md) in place of steps 1 and 3.

## Step 1 — Name the anchor

Find the originating requirement from **outside this session**: the linked issue, work item, CR, or the request as it was first made.

The description and the code come out of the same session, so they encode the same understanding of the task. If that understanding was wrong, a description written from the work alone describes the mistake faithfully, and the reviewer finds perfect agreement between two things that are both wrong. The anchor is what breaks that loop.

Where the work reveals no anchor, ask the author which requirement this serves. Section 2 is written once they have named it, or confirmed there is none.

## Step 2 — Draft what the work supports

The change, the argument, and any conditional element the session actually evidences. Read [EXAMPLE.md](EXAMPLE.md) for a worked bug fix at the right level of detail before drafting.

Write in Simplified Technical English: one idea per sentence, active voice, domain terms as the repo's `CONTEXT.md` defines them, where one exists. Write concrete: every sentence names something the reviewer can open — a symbol, a file, a behaviour, the anchor. The exception is the argument in section 3 — where a causal chain needs a longer sentence to stay whole, spend it.

Budget: a reviewer reads the whole description in about two minutes. For most PRs that is 150–300 words. A large change earns more only for what hasn't changed and the risk — never for inventories. Summarise test coverage in a sentence; the diff shows the rest. An `n/a` gives its reason in one.

List the user-visible surfaces the diff adds or changes — controls, states, dialogs, progress. These become the screenshot requests in step 3.

## Step 3 — Ask for the rest

Put the outstanding questions to the author in one message, specific enough to answer in a sentence each. Typically:

- When the anchor gives no reason for the change, what is it for?
- For each user-visible surface from step 2, a named shot: "the deploy button disabled during processing", not "attach screenshots".

Ask only what the work leaves open, and use the answers as given rather than smoothing them into stronger claims.

**Done when** every required element is filled from the work or from the author's answers, and no text placeholder remains.

## Step 4 — Check before posting

- Every required element has a sentence, or `n/a` with a reason.
- Section 3 traces the requirement to the mechanism: a reviewer can follow how the change produces what section 2 asks for.
- Section 1 talks about behaviour and structure. Changed files and line counts are already in the diff view.
- Every claim in section 3 is one the author will stand behind, not one the diff merely implies.
- A Linear-tracked anchor is named with a magic word, closing (`Fixes`) only where the merge completes the issue.
- *What hasn't changed* appears whenever the diff modifies existing code.
- Every sentence names something the reviewer can open. A sentence about the description or its elements is cut.

## The three sections

Every PR carries all three headings, written as `##` markdown headings. A section may be a single sentence. Where a conditional element doesn't apply, leave it out; where a required one genuinely has nothing to say, write `n/a` and why — a visible gap can be challenged, a silent one can't.

Each section opens with its lead element as plain prose — the change, the purpose, the argument. Every element after the lead opens with its name in bold, ended with a colon: **What hasn't changed:**, **Feature switch — `Name`:**, **Upgraders:**. The colon and the heading level keep the two tiers distinct, a missing element is visible at a glance, and an agent reviewer can anchor on the label instead of parsing prose.

### 1. What's the change?

**Required — the change.** What the product or the code now does differently, in terms of behaviour or structure, at a level someone could check against the diff.

**Conditional — what hasn't changed.** Applies when the changed code can reach behaviour that existed before the PR; a purely additive change — new files, plus inert wiring like a docs row — leaves it out. Where it applies, claim the behaviour the change could plausibly reach and deliberately does not alter. This is the element a reviewer can recover no other way: from inside a diff, every touched file looks intended, and on a large or generated diff this claim decides whether a reviewer knows where to spot-check or has no way to tell mechanical from meaningful. A fact the diff's file list already proves — untouched files stay untouched — does not belong here.

**Conditional — feature switch.** Name it, and give the behaviour with it off and on. A switch defaulted off is a statement about what doesn't change for existing customers, which is why it belongs beside *what hasn't changed*.

**Conditional — upgraders.** What upgrades (resource, warehouse), and the expected defaults for new settings on an existing installation.

### 2. Why are we making it?

**Required — the purpose.** The reason the change is wanted — the problem it solves or the capability it delivers, tied to the anchor. An anchor can name a deliverable without giving a reason; a CR that says "add X" supplies the what, and the description already has the what twice. When the anchor states no reason, get one from the author. *Provide a way to deliver Power BI reports to customers using tabular solutions* is a purpose; the CR title repeated is not. An anchor tracked in Linear is named with a [magic word](https://linear.app/docs/github#link-through-pull-requests) — `Fixes DEV-123` when the merge completes the issue, `Part of DEV-123` when it does not — so GitHub links the PR and moves the issue on merge.
### 3. How does it satisfy the requirement?

**Required — the argument.** How the change closes the gap section 2 names: the mechanism, traced from the requirement to the code's behaviour. An agent reviewer can genuinely evaluate this, which is where most of the review value sits. Evidence — screenshots, video, metrics — attaches here as proof of a claim, rather than standing in place of one.

## Not yet covered

Refactors with no intended behaviour change. The format asks whether a change achieves its intended behaviour, which has no purchase when there isn't one, and the three sections collapse into restating "nothing changes". This is a deliberate omission to be revisited once the format has been used in practice — not an oversight. PR titles are likewise not covered yet.
