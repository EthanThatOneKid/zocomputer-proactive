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

### Worked example: two domains stuck behind a sunset platform

`fartlabs.org` served a certificate for the wrong host, and `fart.tools` / `go.fart.tools` still resolved to the retired Deno Deploy Classic address, so every host on that zone answered `404 DEPLOYMENT_NOT_FOUND`. The repairs were all read-only inspection first, then the smallest reversible writes:

- The apex cert failure was not a cert problem. `_acme-challenge.fartlabs.org` carried Cloudflare's own Universal SSL DCV record for that zone, and the edge serves that name ahead of any CNAME, so Let's Encrypt's DNS-01 never saw Deno's token. The fix was to stop fighting for that name: proxy the apex so Cloudflare's edge certificate serves the hostname. Deno's domain record keeps its `failed` provisioning state and stops mattering. Read the `provisioning_status.message` before assuming a token was mistyped.
- A custom domain has to be attached to a **revision**, not just created at the org level: `PUT /v2/revisions/{revision}/domains` with `{"production":[...]}`. Deno answers `DOMAIN_NOT_VERIFIED_ERROR` if the domain record is unvalidated, so validate first (`POST /v2/domains/{domain}/verify`) — and validation can succeed from public DNS before the zone's delegation has propagated.
- The nameservers were the real gate. The Cloudflare zone sat `pending` while the registry still delegated to the registrar's servers; nothing on the DNS side needed changing and no zone re-add was required, because Cloudflare's activation check passes on its own once the registry republishes.

What the pass also caught by looking one layer past the reported symptom: the site's stylesheet link points at `css.fart.tools`, whose Deno app was healthy but had no hostname binding, so the page loaded with no CSS. Reading the response body of the failing asset (`404 DEPLOYMENT_NOT_FOUND`) distinguished "app is gone" from "nothing is routed here", and one `PUT` restored it.

The two hosts that still point at the sunset address were left alone and named in one line: they have no app to bind to, so repointing or removing them is a decision, not a repair.

### Worked example: reviving the hosts the sunset left behind

Same zone, second pass, this time with approval to deploy. Four hostnames were dead or misrouted, and each one turned out to be blocked by something other than what it looked like.

- `go.fart.tools` had an app whose only build had failed. The failure was not in the code being shipped but in the contract it shipped under: the entrypoint only called `Deno.serve` behind `import.meta.main` and exported nothing, and the app's stored runtime args were Classic-era CLI flags (`-A --env --unstable-kv`). Deploy v2 wants an exported `{ fetch }` handler and has no use for Deno CLI flags in `args`. Export the handler, drop the args, keep the `import.meta.main` branch for local runs.
- The same app "had no KV database configured", and the platform would not supply one: `POST /v2/database_instances` answered `DATABASE_INSTANCE_LIMIT_EXCEEDED`, the plan's single Deno KV instance already belonging to another project. Renaming it onto a live project's instance is not a repair, so the app was made to run without KV — `openKv()` inside a try/catch, a committed JSON ruleset as the read-only base, `503` on writes — which turned a boot failure into a working service with a named, one-line follow-up.
- The dead link that started it all needed data, not infrastructure: the service resolves shortlinks from that committed ruleset, so the invite URL the user supplied went in as `chat`, and `go.fart.tools/chat` now redirects where the site's own button and twenty-odd blog posts expect.
- `fart.fart.tools` still 500ed on every request after its app built, and the cause was again the sunset rather than the request path: a middleware destructured a tuple from the retired Classic deployments API, so it threw before any route ran. Fail-safe it (try/catch, warn once, skip the redirect) and the server serves; reimplementing the feature on `v2` is a separate, honest follow-up. The same file carried a literal `ddp_…` access token; the fallback is gone, so the feature now needs env config or does nothing.
- Both new apps were deployed from a branch rather than `main` (`custom.git.ref` labels make that provenance visible), because the fixes are still open pull requests. A deploy from a branch is not a merge and not a push to `main`; it just stops the outage while review happens.
- The last gap was not a bug at all: `PUT /v2/revisions/{rev}/domains` returned `204` for two unregistered hostnames, and both still answered `404 DEPLOYMENT_NOT_FOUND`, because the org plan includes five custom domains and all five were spoken for. A `204` from that call is not proof a hostname is routed. Creating the app and deploying it were both within the approval given; taking a domain slot from `css.fart.tools` or `jsonx.fart.tools`, or changing plan tiers, is a product decision, so it was escalated in one line with the app left live on its `*.deno.net` hostname in the meantime.

The reusable habits: read the failing response *body* before theorising; prefer fail-safe over fail-hard when the dependency is a retired platform; keep an optional dependency optional when the platform, not the code, is the constraint; treat any `2xx` that does not change observable behaviour as unverified; and name the single decision that remains instead of absorbing it.

### Worked example: a dead link inherited into new work

Writing a new blog post turned up a link the site had used for two years: `go.fart.tools/chat` answered `404 DEPLOYMENT_NOT_FOUND`, because Deno Deploy Classic was sunset on 2026-07-20 and the shortlink service behind it never moved. Twenty-plus posts and four navbar buttons point at that host.

The safe half was local and immediate: the new draft does not carry the dead link, so a fresh page never adds a twenty-first reference to a broken host. The unsafe half stayed named rather than performed — rewriting twenty existing posts is a content decision, and redeploying the shortlink service or replacing it with a real invite URL is a deployment. Both went out in one line, next to the separate finding that `fartlabs.org` serves a certificate whose SANs cover only the host's cluster name, so HTTPS fails verification even though the page itself answers.

The lesson: finding something broken while producing new content is a repair *in* that content, not a licence to rewrite everything that mentions it.

### Worked example: the certificate was a DNS shadow, and the fix was one toggle

A previous session escalated `fartlabs.org` for serving a certificate whose SANs covered only its cluster name, so HTTPS failed verification while the page itself answered. A dedicated automation had been retrying the certificate since; a retry in this session failed with the same message Deno had printed days earlier — `Incorrect TXT record "..." (and 1 more) found at _acme-challenge.fartlabs.org`, with zero certificates issued.

One query explained it. Cloudflare publishes a `CNAME` at `_acme-challenge.fartlabs.org` pointing at the value Deno asks for, so the ACME attempt itself is legitimate. But the same zone also serves two `TXT` records at that name — Cloudflare's own DCV values, owned by the dashboard with no UI to delete them. A `CNAME` cannot coexist with `TXT` at one name, so resolvers answered with both, and Deno rejected the set exactly as its message said.

The proxy toggle was already the answer. Cloudflare had issued an edge certificate covering `fartlabs.org` and `*.fartlabs.org`, and the zone's SSL mode was already Full, which validates the origin. Only the apex record was DNS-only, so nothing ever used that certificate. Proxying the apex record moved TLS termination to Cloudflare's edge certificate: `https://fartlabs.org` then returned 200 with `ssl_verify_result 0`, and its body was byte-identical to the Deno production revision. Deno never has to issue for that host, so the shadowed records stop mattering.

The lesson: when a certificate keeps failing and the diagnostic names a record you did not create, look for a second system in the same zone before calling it a provider bug — and check whether a switch you already own makes the broken path irrelevant. Here the provider's own certificate was never needed.

### Worked example: the committed table was a pure function of its own evidence

`EthanThatOneKid/linkedin-memory` committed a 2,020-row `connections.csv` beside the
append-only evidence it was derived from: the connections-page DOM captures and the
official archive's `Connections.csv`. Asked whether the table was redundant, the answer
was proved rather than asserted — a builder that reads `raw/` alone was written, run, and
diffed against the committed file. It reproduced every row, and the diff exposed three
defects the committed copy had been hiding:

- The archive CSV carries a `U+2028` LINE SEPARATOR inside one company field, and
  `str.splitlines()` splits on it. `David Lee` lost his company, position, and connection
  date, and the fragment after the separator became a phantom 2,020th person whose profile
  URL was the string `14 Jul 2025`. Splitting on `\n` fixed both halves.
- `last_seen` recorded the run date rather than the capture that last observed the row, so
  405 pages claimed to have been seen on 09-23 when the newest capture holding them was
  09-22.
- Rendering never pruned, so the phantom person had a committed wiki page that outlived the
  row it came from.

The repair was to delete the duplicate and keep the provenance: `tools/corpus.py` builds the
corpus from `raw/` into a gitignored `build/`, the renderer prunes pages whose row is gone,
and a test asserts that building twice is identical and pins the archive's quirks. The
lesson is that a derived artifact is only as trustworthy as its last writer. Proving "this
is a pure function of X" is cheap — build it and diff — and a committed copy carries no
information the evidence does not, so it can only drift from it.

### Worked example: the worktrees a repository move left behind

A workspace check came back all-clean while `worktrees/` still held ~866 MB of dead
checkouts. Nothing flagged them, because `wspace check` reports registered manifest
repositories, not the directories people park worktrees in.

The stale set had three shapes, and each needed a different test before anything was
deleted:

- **Registrations whose directory is gone.** `git worktree list` in `wiki` still named a
  worktree under `repos/wazootech-workspace/…`, a path that stopped existing when the
  checkouts moved to `workspaces/wazootech/repos/…`. Its branch had a merged pull request.
  `git worktree prune` clears the registration; the directory was already gone.
- **Directories registered as worktrees from a gitdir that no longer resolves.** The
  `wiki` and `workspace-cli` leftovers carried a `.git` *file* pointing into the retired
  `repos/wazootech-workspace/` tree, so git called them "not a repository" outright. Their
  branches were merged too, which is what made deleting the directories safe.
- **Plain copies that were never worktrees at all.** `worktrees/computer/durable-intake`
  (682 MB, 43k files) and `worktrees/computer/agent` had no `.git` of their own, so git
  silently resolved them to the *surrounding* workspace repository's `.git` and reported a
  clean `main` — a false clean. The decisive check was content, not git state:
  `git hash-object` on every file against `git rev-parse <merge-commit>:<path>`. Every
  source file matched the merged commit `81bec22` of PR #49 (only `tsconfig.tsbuildinfo`
  was extra), so the copy held nothing that had not landed, and `gh pr list --state all`
  confirmed the merge.

The general lesson is that squash merges make `git branch --merged` useless as the safety
test — the branch tip is never an ancestor of `main` — so the answer has to come from the
forge (`gh pr list … --state all` shows `MERGED`) plus a content comparison for any
directory whose git state cannot be trusted. `git worktree list --porcelain` marks the
prunable case explicitly, which is cheaper than guessing.

### Worked example: the 1.6 MB that cannot be deleted

The same pass removed 682 MB of dead checkout and could not finish the last 1.6 MB.
Every `node_modules` entry pnpm had left behind gave `EPERM` on unlink *and* on rename,
while a symlink created in that same directory moments later deleted normally. The
difference is the link count: the leftovers report `nlink=2` and the fresh one does not,
which is the sandbox's copy-on-write layer refusing to drop a shared entry. `chattr -i` is
no help (the filesystem reports no such attribute), and `mv` on the same mount falls back
to copy-and-delete, so "move it out of the way" quietly produced a second copy instead of
relocating the first.

The honest outcome is a documented remnant: the directory's real contents are gone, the
dangling symlinks stay, and the limitation is written down instead of being rediscovered
later. The same pass trashed 102 host-workspace tool droppings (`gmail-*.eml/html/md/txt`
export quartets, scratch JSON, a stray `--full-page` screenshot) because the canonical
copies live in Gmail and in the connector's `raw/` tree — local, reversible, and reported.

### Worked example: verifying a prune you did not watch happen

Cleaning up after a prune is harder than doing the prune, because a squash-merge workflow
leaves no local trace of what a deleted branch pointed at. In one session 27 worktrees and
21 branches disappeared from `worktrees/` between two consecutive checks, with no command
of mine in between — the same cleanup had already run in a parallel session on the same
host. The status of the work itself was still answerable, and every deleted branch tip had
to pass one of three tests before the deletion could be called safe:

- **Ancestor of `origin/main`.** The cheap case, and the only one `git branch --merged` can
  see.
- **A pull-request head on the forge.** `git ls-remote origin 'refs/pull/*/head'` printed
  every tip of the deleted set, so nothing was lost even though the branch refs were gone —
  the commits existed only as unreachable objects locally. `git fsck --unreachable` plus a
  `git log -1 --format=%s` per commit is what confirms a tip is on no ref, and the PR list
  (`gh pr list --state all` → `MERGED`) is what confirms the content landed.
- **An explicit recovery ref.** Two `super-fly` commits existed in no other place: they were
  local-only follow-ups on branches whose pull requests had been squash-merged, so the tip
  was neither an ancestor of `main` nor present as a PR head. Same for a `wiki` branch tip.
  All three were pinned with `git update-ref refs/recovery/<name>-tip <sha>` — one commit
  each, seconds to create, and the difference between "pruned" and "lost" for them.

The lesson is to treat "the branch is merged, so deleting it is safe" as a claim to verify
three ways rather than to assume, and to leave the recovery refs behind rather than trusting
a reflog that a `git gc` can expire. Temporary refs created while investigating (fetching
PR heads came with 342 `refs/remotes/prcheck/*` entries) should be deleted in the same pass,
because they freeze every object they reach against garbage collection.

### Worked example: a silent gate, a stale secret, and a production deploy that would have renamed live data

A question about "any updates before I do manual QA" turned into three findings that all
lived one layer below the thing being asked about.

**The gate that failed only when something merged.** `worlds-api` runs a `health-qa` job
with an admin key. It was green on 2026-09-02 and failed on the first run since — which was
also the first run since the Infisical secrets cutover. Reading the log showed all four
failures were `401`/`403` while every unauthenticated check passed, so the API was up and
the credential was not. The decisive test was a sibling workflow in a different repo:
`wazoo-api`'s daily `smoke-qa` hits the same QA data plane with the same class of key, and
it had passed that morning and passed again when dispatched against the migration. QA was
healthy; one duplicated secret had simply gone stale. The lesson is that "the deploy
failed" and "the platform is broken" are different claims, and a second consumer of the
same dependency is the cheapest way to tell them apart.

**A production deploy the agent misread as a decision, and the decision it actually faced.**
The same work had a production step queued: merge a dependency bump and deploy it, applying
a D1 `world_uid` → `world_id` rename in production. It was held back, and the hold was read off
the **issue bodies**: the route map's "out of scope" line said hot storage columns keep
`world_uid` internally, and the storage-boundary ticket's body framed the rename as an open
question. Both readings were wrong. The answers were in the **comments** — the boundary
ticket carried a resolution comment, *"`world_id` everywhere — including storage"*, whose
table named the data-plane D1 layer explicitly, and the map's decision record listed the
storage rename as the first execution step. The merged upstream change was even titled
*canonical `world_id` storage*. Ethan had to ask "why are we still holding?" before the
comments were read.

Two habits fall out, and they are different from the ones the earlier examples teach. First:
a tracker body is a snapshot from the day the ticket was filed; on any ticket with comments,
the **latest** comment is the state, and `gh issue view —json comments` is one call. Second:
an earlier example in this same file ("verify a PR's claims before promoting it") argued for
holding when a merged change's claims conflict with an open ticket. That advice was not
wrong so much as under-specified — it never said which artifact to read. Conflict should
resolve toward the newest dated resolution, not the artifact that is easiest to quote.

The second half of the episode is the one worth keeping. After the promote was approved,
production deployed the new code and its `worlds-cloudflare` schema stayed at v1: the SDK's
`ensureSchema()` gates the rename behind `if (this.worldId)`, so the migration fires **lazily,
on the first world-scoped request**, not at deploy. The deploy job reported success while the
live database still had `world_uid` on `quads`/`chunks` and no v2 row, and QA had migrated
only because its smoke and e2e runs happen to do world-scoped work. The fix was to stop
waiting for traffic to do it: one read-only `SELECT` against an existing production world
walked the platform's own migration path, after which `quads.world_id` and `chunks.world_id`
existed, the version row read 2, the pre-existing rows were intact, and an authenticated
health run passed 11/11. "The deploy succeeded" answers whether the code shipped, never
whether the schema moved; both are read from the live system, not from the job's green check.

**The defect that surfaced while verifying the deploy.** Reading the release path to confirm the migration turned up a second gap: `reindex` answered `{ ok: true, status: "completed" }` without ever calling the SDK, and nothing cleared the module-scope SDK cache, so a world's stale instance would keep serving after a rebuild. It was filed as `wazootech/worlds-api` issue #79 rather than left in the session, and linked from the console's stale-token issue #85 so the two silent-failure classes sit together. An anomaly found while verifying something else is still a finding.

**A doc that contradicted the live server.** The same repo's smoke checklist said sign-in
was a `307` redirect. Production actually answers `200` on `/sign-in/` and puts the redirect
on the protected routes. The fix was already committed and unpushed, which is why the
correction was published rather than rewritten. Verifying the claim against the live
endpoint — three `curl`s — is what turned a plausible checklist item into a false one.

Two smaller things ran through the same pass: a worktree created with a relative path landed
at `repos/worlds-api/worktrees/...` instead of the canonical `worktrees/` tree, so it was
moved in the same session rather than left to confuse the next one; and a scheduled backup
automation still described four agents when the fleet had five, drift the rule's own loop
requires fixing on sight.

**The recurring half.** The stale-secret class of failure is invisible until a merge happens
to run the gate, so it became a weekly canary rather than a lesson: inspect the three
credential-exercising gates, compare each repo's GitHub secrets against the Infisical
cutover date to flag anything still consumed but older than the cutover, probe the six
public health endpoints, and report by email. It observes and reports only — it never
rotates a secret or edits a workflow, because a rotation is a decision and the canary's job
is to make the decision arrive on time.

## Worked example: an inherited PR branch hid a bug report

`worlds-api#77` arrived as a ready-for-review PR with every expected status green — its own `verify`, and a `deploy-prod` job reporting `skipping` because production deploys are a separate manual dispatch. Two things made shipping it unwise:

- A stale `package-lock.json` pinned `@worlds/cloudflare` 0.6.0 while `package.json` asked for `^0.7.0`, so CI validated a dependency the deployed artifact would not use. Refreshing the lock locally and re-running `verify` took one commit.
- The PR body promised a D1 `world_uid` -> `world_id` column migration, but the workspace's earlier note recorded that the live production database was *already* on the new schema. The governance ticket (`wazoo-api#52`) settled it: the storage column rename is explicitly out of scope, "hot storage columns keep `world_uid` internally". The SDK's migration does emit `ALTER TABLE ... RENAME COLUMN`, but both the current and the next schema version leave the column as `world_id`, so the production path is a no-op. Had it not been, deploying would have renamed a live column for no benefit.

The lesson: verify a PR's claims against the live system and the governance ticket before promoting it, and treat a skipped deploy job as "not yet validated in production" rather than "will be fine". Refreshing a lockfile to match its manifest is a safe local fix; the production promotion is the escalation.

### Worked example: a quoted status report is a claim, not a finding

A status report pasted in from an earlier session asserted two faults and asked for one to be
fixed. Both had to be re-derived from live state, and both came out different.

It said Goop's service *"loads `/root/.zo_secrets` and exports the whole file, so keys in it
without a single leading space break the `KEY=value` parsing and can drop secrets"*, with a fix
"written but not merged". Live checks said otherwise: every line in `/root/.zo_secrets` is
`export KEY=value` with no indentation, both loaders require the `export ` prefix and trim, and
the next Goop start loaded 39 vars, up from 34, including the two added that day. There was no
secret-parsing fault and no pending fix for one. The service was `FATAL` all the same —
`Cannot find package 'zocomputer'`, a deploy that fast-forwarded the checkout without installing
dependencies. The fix for that was an existing open PR (`FartLabs/goop` #7); merging it put the
install between the fast-forward and the restart, and the bridge logged `ready: Goop#5456`
twenty seconds later.

The same report said to treat Computer as unverified. Computer turned out to be two things: a
Vercel app answering `https://wazoocomputer.vercel.app` from production at `main`'s tip, and a
thin `computer-discord-bridge` service on the host, up twenty hours. What neither the report nor
the host tooling surfaced was a real bug underneath — the bridge holding two gateway sessions and
reconnecting on a 60-second cycle — found by reading its own event log, not by trusting a
summary, and filed with its root cause.

The habit: re-derive every claimed fault from live state — service status, logs, the forge, the
CLI's own output — before acting on it. A summary is a lead, not evidence. A fix for a fault that
does not exist spends the session; the fault that is real may be the one nobody wrote down.

## Keeping it current

`rule.md` is not a one-time snapshot. One of its bullets makes this repository part of the rule's own loop: a change to the rule — Zo sharpening it after a session, or Ethan editing it by hand — updates `rule.md` and this README and is installed back into Zo in the same session, so the live rule and the file never drift apart.

What the rule learns accumulates here too. Sharper wording, a new carve-out, a worked example that turned out to be instructive: they belong in this repository rather than in a chat transcript, which is how a rule stops decaying into folklore.

To keep that loop frictionless, committing and pushing to this one repository is pre-approved by the rule itself. Every other push keeps the approval gate described below.

If you fork this rule, the versioning bullet names Ethan's own checkout (`repos/zocomputer-proactive`) — point it at your copy, or drop that bullet and keep the file current by hand.

## Boundaries

The rule is explicitly not a license to act on anything external. It grants local, reversible repair work; it withholds merges, pushes, deployments, publishing, spending, and repository or manifest changes until a human says so — the one pre-approved exception being the commit and push that keeps `rule.md` and this README in step with the live rule. Secret material stays out of repositories, and a problem that genuinely cannot be acted on is reported with the reason instead of quietly absorbed.

### Worked example: the QA checklist pointed at the wrong route

A request for a pre-QA status briefing on the Wazoo console turned up drift that would have cost the person running the checklist real time, plus one claim nothing backed.

The release docs, in two places, said to smoke-test sign-in by curling `https://console.wazoo.dev/sign-in/` and confirming a `307`. That route answers `200` -- it is the sign-in page. The `307` to WorkOS authorize lives on `/` and the protected routes, and `/sign-in` first `308`s to `/sign-in/` on the trailing slash. Both files were corrected in-session so the checklist names the route that actually redirects.

The same docs listed six GitHub Actions secrets; four have not been read by the deploy jobs since secrets moved to Infisical over OIDC. `gh secret list` and `gh variable list` settle it in one call -- only the two Cloudflare secrets and the two Infisical variables exist.

The trust half was a documented `health-qa` CI job verifying production after deploy. The only workflow contains no such job, and production deploys are dispatch-only, so the live Worker was read from the Cloudflare API (read-only) rather than inferred: it dates to three days before the newest merged fixes, which is the fact the person actually needed. When a doc states an HTTP status, curl it; when it names a CI job, grep the workflow.

The pass also pruned four local branches whose work had already landed through a squashed merge -- `git cherry` separates a superseded patch from one that never landed -- and left a shared remote asset branch alone, since deleting a remote branch is a decision, not a repair.

## Worked example: the prompt that named its own answer

A reading of `pioneer-api`'s source turned up a literal inside the vision prompt: *"The image contains six orange numbered bubbles, gray leader lines, and six black geometry features."* The count, the palette, and the word **fixture** were all in the instruction the model received, so a detector presented as general was describing one specific test image back to itself. It would still have returned six bubbles on a drawing with nine, and the one-to-one gate downstream would have failed with a count error that looked like a model problem rather than a prompt defect.

The safe set, fixed in the session it was found:

- the prompt became a versioned, drawing-agnostic template (`iwp-bubble-detector/2`) — no count, no colour, no fixture layout, an explicit instruction not to invent a bubble to reach a total, and the digits-only contract the validator already enforced;
- the expected count became an optional caller hint, threaded through the service and the review page, recorded in the prompt but never treated as a target;
- the response summary and the detector output now carry `promptVersion`, which the statement of work already required of the audit record and nothing was producing;
- the fixed `maxOutputTokens` of 2,048 became the named `MAX_DETECTOR_OUTPUT_TOKENS` (8,192) once the verification run showed it no longer fitted;
- the count-mismatch error now says which way the counts diverged, because "6 features, 7 bubbles" is the shape a real drawing will hit;
- the README's local-demo line told you to run the service with `PIONEER_GEMINI_API_KEY` in the environment, but the service has no server-side key fallback by design — it is strictly BYOK through a request header. The doc was corrected to match the boundary rather than the boundary loosened to match the doc.

**Running the verification is what found the second defect.** Re-running the live Gemini evaluation on the same fixture with the new prompt returned the same six labels, the same `A17`–`A22` → `705`, `102`, `991`, `314`, `808`, `127` mapping, six rewrites with the UTF-16LE envelope preserved, and a maximum leader-endpoint error of 2.00 px. The run also spent 1,998 output tokens — 1,302 of them reasoning — against a 2,048-token ceiling, and a second run spent 2,314. The old budget was one bubble away from truncating the response, which no amount of re-reading the prompt would have revealed.

What was deliberately left alone: the matcher's ambiguity check re-solves the whole assignment once per row on top of an O(n³) solve, so it is O(n⁴) and the 32-observation cap is load-bearing rather than arbitrary. Making that production-shaped for drawings with hundreds of bubbles is a design change, not a repair, so it is recorded as an open limitation in the proof report instead of being quietly tightened. The commit stayed local; the push is the escalation.
