# Autonomous mode

For an agent producing a PR from a bug report with no developer to question. This file replaces steps 1 and 3 of [`SKILL.md`](SKILL.md). Steps 2 and 4, the register, the budget, and the three sections are unchanged. Two elements exist only in this mode — *how it was checked* and *where to look hardest*: an unsupervised run volunteers what a developer would have been asked for.

The reviewer inherits the questions the author would have answered. Write every claim so the reviewer can check it without trusting you.

## The anchor is the bug report

Quote the report's observed behaviour and its expected behaviour. The purpose is the gap between them.

A report that states no expected behaviour leaves the purpose incomplete. Say so in section 2, state the expectation the fix assumes, and mark it as an assumption for the reviewer to confirm.

## Answers come from the run, as facts

With no developer to ask, the run supplies the answers. Keep each one checkable:

- **The argument** closes the report's gap: show how the fix turns the observed behaviour into the expected behaviour.
- **How it was checked** names exactly what was run — the commands, the tests — so a reviewer can re-run them. What was not run is *not verified*, unsoftened. Never claim a check the reviewer cannot reproduce.

## Where to look hardest is derived, never self-assessed

An agent that misread the report is confident about the part it got wrong, so its own doubt is not evidence. Derive the element from the diff instead:

- changed files with no test coverage
- paths reachable only in configurations the run never exercised
- anything touched that the description says hasn't changed

Report what the derivation found. When it finds nothing, write *nothing derived* — a hunch invented to fill the element is worth less than the admission.

## Evidence

Capture screenshots where tooling allows. Where it does not, name the missing shot — *not captured: the dialog after rename* — so the gap is visible to the reviewer rather than silently absent.
