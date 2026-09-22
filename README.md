# zocomputer-proactive

A Zo Computer rule that keeps the agent closing loops instead of reporting them back.

`rule.md` is the versioned source for the rule: an always-applied instruction telling Zo to fix safe, reversible problems in the same session, escalate only genuine decisions in one line, set up an automation for anything that will recur, and keep this file in step with the live rule.

## Why it exists

A session ended with a clean-looking report that still left work on the human:

> Unrelated to this work, the same check flags `acmcsuf-memory` (MISSING), `fartlabs.org` and `computer` (DIVERGED), and `journeyman` (UNKNOWN) — pre-existing, untouched.

Every one of those was fixable on the spot. Nothing about the report was wrong; it just moved the coordination back onto the person who asked. That is the failure mode this rule closes: a status report is not a substitute for the fix, and "pre-existing" is not a reason to leave a known-broken state in place when a safe repair is available.

The point, in the words that prompted it:

> If AI can proactively handle the repetitive coordination, follow-ups, monitoring, and small decisions around us, humans get to spend more time on things that matter.

## Install

1. Copy `rule.md` into Zo — ask Zo to "create this rule" with the file's contents, or paste the instruction into Settings → AI → Rules.
2. Make sure it applies always (no condition), so it governs every session rather than a class of prompts.
3. Optional but recommended: pair it with the maintenance automation in [Usage](#usage), because the rule's own last bullet asks for a recurring check to be scheduled rather than remembered.

## Usage

The rule has five moving parts, in the order they typically fire:

1. **Fix the safe set in-session.** Flagged anomalies, stale paths or docs that contradict the code, unfinished follow-ups, duplicate automations, and merged-but-unpruned branches and worktrees get repaired during the session they are found. "Safe" is defined narrowly: local, reversible, no external side effects.
2. **Escalate the rest in one line.** Merging to `main`, pushing, publishing, deploying, spending money, and creating or deleting repositories or manifest entries stay human decisions. When one surfaces, it is named with the repo, the action, and the reason — not a paragraph of options.
3. **Schedule what will recur.** Anything that comes back on a cadence becomes a small automation instead of a promise to remember.
4. **Verify before claiming.** Only what was actually checked gets stated as fact, and what could not be verified gets said plainly.
5. **Keep the rule versioned.** Changes to the rule itself — refinements after a session, or a hand edit — are written back to this repository and installed into Zo in the same session, so the live rule and `rule.md` never disagree.

### Worked example

One session's flagged set, from a workspace drift check: a manifest member that was missing, two clean checkouts that were behind their upstream, and a repository whose default branch had never been committed. All four were repaired in-session — the missing member cloned, the two fast-forwarded, the unborn branch materialized from the remote. The same pass pruned a merged remote branch, removed a stale worktree, deleted the directory it left behind, dropped a duplicate rule, and removed a duplicate automation. One genuine decision was escalated in a single line. Nothing was pushed, merged, deployed, or published without being asked for.

The recurring half is a weekly automation that runs `wspace check` first, repairs the safe states it reports (`MISSING` by installing, behind-only drift by fast-forwarding, an unborn default branch by materializing it from the remote), and escalates everything else by email:

```
RRULE:FREQ=WEEKLY;BYDAY=SU;BYHOUR=10;BYMINUTE=0
```

It carries the same hard boundaries as the rule — no pushing, rebasing, force-updating, deleting checkouts, or manifest edits, and it never runs a blanket `wspace update`, which would fast-forward every clean checkout in one pass.

## Keeping it current

`rule.md` is not a one-time snapshot. One of its bullets makes this repository part of the rule's own loop: a change to the rule — Zo sharpening it after a session, or Ethan editing it by hand — updates `rule.md` and this README and is installed back into Zo in the same session, so the live rule and the file never drift apart.

What the rule learns accumulates here too. Sharper wording, a new carve-out, a worked example that turned out to be instructive: they belong in this repository rather than in a chat transcript, which is how a rule stops decaying into folklore.

To keep that loop frictionless, committing and pushing to this one repository is pre-approved by the rule itself. Every other push keeps the approval gate described below.

If you fork this rule, the versioning bullet names Ethan's own checkout (`repos/zocomputer-proactive`) — point it at your copy, or drop that bullet and keep the file current by hand.

## Boundaries

The rule is explicitly not a license to act on anything external. It grants local, reversible repair work; it withholds merges, pushes, deployments, publishing, spending, and repository or manifest changes until a human says so — the one pre-approved exception being the commit and push that keeps `rule.md` and this README in step with the live rule. Secret material stays out of repositories, and a problem that genuinely cannot be acted on is reported with the reason instead of quietly absorbed.
