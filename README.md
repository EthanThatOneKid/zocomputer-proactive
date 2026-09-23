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

### Worked example: a dead link inherited into new work

Writing a new blog post turned up a link the site had used for two years: `go.fart.tools/chat` answered `404 DEPLOYMENT_NOT_FOUND`, because Deno Deploy Classic was sunset on 2026-07-20 and the shortlink service behind it never moved. Twenty-plus posts and four navbar buttons point at that host.

The safe half was local and immediate: the new draft does not carry the dead link, so a fresh page never adds a twenty-first reference to a broken host. The unsafe half stayed named rather than performed — rewriting twenty existing posts is a content decision, and redeploying the shortlink service or replacing it with a real invite URL is a deployment. Both went out in one line, next to the separate finding that `fartlabs.org` serves a certificate whose SANs cover only the host's cluster name, so HTTPS fails verification even though the page itself answers.

The lesson: finding something broken while producing new content is a repair *in* that content, not a licence to rewrite everything that mentions it.

### Worked example: the certificate was a DNS shadow, and the fix was one toggle

A previous session escalated `fartlabs.org` for serving a certificate whose SANs covered only its cluster name, so HTTPS failed verification while the page itself answered. A dedicated automation had been retrying the certificate since; a retry in this session failed with the same message Deno had printed days earlier — `Incorrect TXT record "..." (and 1 more) found at _acme-challenge.fartlabs.org`, with zero certificates issued.

One query explained it. Cloudflare publishes a `CNAME` at `_acme-challenge.fartlabs.org` pointing at the value Deno asks for, so the ACME attempt itself is legitimate. But the same zone also serves two `TXT` records at that name — Cloudflare's own DCV values, owned by the dashboard with no UI to delete them. A `CNAME` cannot coexist with `TXT` at one name, so resolvers answered with both, and Deno rejected the set exactly as its message said.

The proxy toggle was already the answer. Cloudflare had issued an edge certificate covering `fartlabs.org` and `*.fartlabs.org`, and the zone's SSL mode was already Full, which validates the origin. Only the apex record was DNS-only, so nothing ever used that certificate. Proxying the apex record moved TLS termination to Cloudflare's edge certificate: `https://fartlabs.org` then returned 200 with `ssl_verify_result 0`, and its body was byte-identical to the Deno production revision. Deno never has to issue for that host, so the shadowed records stop mattering.

The lesson: when a certificate keeps failing and the diagnostic names a record you did not create, look for a second system in the same zone before calling it a provider bug — and check whether a switch you already own makes the broken path irrelevant. Here the provider's own certificate was never needed.

## Keeping it current

`rule.md` is not a one-time snapshot. One of its bullets makes this repository part of the rule's own loop: a change to the rule — Zo sharpening it after a session, or Ethan editing it by hand — updates `rule.md` and this README and is installed back into Zo in the same session, so the live rule and the file never drift apart.

What the rule learns accumulates here too. Sharper wording, a new carve-out, a worked example that turned out to be instructive: they belong in this repository rather than in a chat transcript, which is how a rule stops decaying into folklore.

To keep that loop frictionless, committing and pushing to this one repository is pre-approved by the rule itself. Every other push keeps the approval gate described below.

If you fork this rule, the versioning bullet names Ethan's own checkout (`repos/zocomputer-proactive`) — point it at your copy, or drop that bullet and keep the file current by hand.

## Boundaries

The rule is explicitly not a license to act on anything external. It grants local, reversible repair work; it withholds merges, pushes, deployments, publishing, spending, and repository or manifest changes until a human says so — the one pre-approved exception being the commit and push that keeps `rule.md` and this README in step with the live rule. Secret material stays out of repositories, and a problem that genuinely cannot be acted on is reported with the reason instead of quietly absorbed.
