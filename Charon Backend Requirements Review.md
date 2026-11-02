# Charon Backend Requirements Review

Nov 2, 2026 · @K

## Verdict

The core design is sound and a bank would recognise it. Rollups for screens, position and rows committed together, idempotent loads keyed on request id, decimals as text, database time for every decision, a cap decision and its outbound command saved in one transaction, absolute amounts with read-back, a week of shadow mode, a kill switch, a guard against an empty directory read, and a tool that is never in the request path: these are the things that fail in practice, and the document gets every one of them right.

Around that core it is not what a senior backend developer would build for a first release. Roughly half of the 159 Charon-owned requirements and two thirds of the 63 Risk IT asks are layered mechanism: two or three controls on nearly every failure mode, cryptography on internal tables whose writers already hold the keys, eleven programs and thirteen database logins, seventeen two-person approvals for a team of three to five, and a request that Risk IT turn two plain views and a "set a number" call into a commit-ordered, epoch-versioned, sequence-checked distributed system. Each extra interlock is one more way for a legitimate uplift to sit stuck. The uniform depth is the tell: the changelog's five reviews each added mechanism and none removed any.

Three changes remove most of the excess without touching the core: a level-triggered reconcile loop in place of the twelve-state command machine, a database of Charon's own in place of sixteen shared-server safeguards, and loading by an ingest timestamp in place of positions, epochs and revisions. With those, release 1 is two deployables, three logins, about ten asks of Risk IT, and a read-only shadow period before any cap is sent. Release 2 is enforcement.

## How this was read

The document (Draft v2.1 of BACKEND-BUILD-REQUIREMENTS.html, 222 requirements) was read from a seven-minute phone video of the screen, not from the file. One frame every two seconds was cropped, upscaled and transcribed; roughly 60 percent of the text is legible.

| Part of the document | Legibility |
| --- | --- |
| Purpose, architecture principles, flows, failure table, design choices | Good |
| RR, DS, CC, GV (Risk IT domains) and LD (Loading) | Good; five passages blurred |
| ST, SV, IA, SD | Fair; many words guessed |
| EN (Enforcement), OP (Operations), TV (Tests) | Poor; several bodies unreadable, IDs inferred from order |
| Data Shapes tables, test cases 1 to 14, glossary middle, margin notes | Poor to unreadable |

Every number quoted in this review may be a misread. The figures listed under "Figures that disagree" must be checked against the HTML before anything is changed on their account. A second pass by two independent reviewers (one for over-engineering, one for gaps and contradictions) was run on the same transcripts; their findings are folded into the sections below.

## What the document gets right

The hard invariants are correct, and they are the ones first-time backend builders almost always get wrong. A reviewer on Risk IT will trust the author on the strength of these alone.

- **Time.** Every decision uses database time, not host time (OP-13). Usage days are UTC with the midnight-straddle test written out (RR-2, CC-14, ST-1).
- **Loading.** The feed position commits in the same transaction as the rows (LD-3). Page loads are idempotent by request id and revision (LD-7). A bad row goes to quarantine instead of halting the feed (LD-4, LD-5). An explicit column list is always sent (LD-12). An empty page is neither failure nor proof of health (LD-6).
- **Money.** Cost travels as text and lands in a decimal, never a float (RR-8, LD-13, ST-2, SV-9).
- **Concurrency.** Leases expire on database time and carry fencing tokens (LD-2). Feed epochs and safe positions catch rebuilds and late writers (DS-2, DS-3, LD-10, LD-11).
- **Safety.** When uncertain, do nothing and alert (principle 7). The empty-directory guard disables nobody on a bad read (LD-20). Shadow mode before enforcement (EN-8). The sender starts paused after any restore and reconciles with the gateway first (OP-11). Held spend is kept and shown, never dropped (LD-15).
- **Cap calls.** Exact amounts, idempotency keys and an expected-value check make repeats and races safe (CC-1, CC-2). A restore anchor at the gateway detects a database rolled back under Charon (CC-15).

Four sections carry the whole argument and should move to the front of whatever Risk IT receives: the eight architecture principles, Table 6 (what users see and what the system does when each part fails), the Data Shapes tables with their Today column, and the table of cases the non-production data must contain.

## Split it, phase it, re-prioritise it

Risk IT should receive one document of about 70 requirements, not 222. The other 150 are the Charon team's own build spec and describe mechanism (claim tokens, signatures, closure lists, lock order, column types) rather than outcomes.

| Document | Contents | Reader |
| --- | --- | --- |
| Interface contract | RR, DS, CC, GV; Data Shapes tables; the non-production cases table; the open questions that are theirs; a one-page summary at the front | Risk IT, the database owner, the Global Head |
| Build spec | LD, ST, SV, EN, IA, SD, OP, TV, QA; architecture; timers | The Charon team only |

Inside the interface contract, separate *things to build* from *things to confirm in writing*. About ten of the Risk IT items are statements for the file, not code (RR-4, RR-13, CC-9, CC-15, DS-13, DS-14, GV-4, GV-5 among them). Two short lists get answered; one long mixed list gets parked.

**Phase the release.** The document already contains the natural line and never draws it. With 192 of 222 as Must, a reader concludes the author does not know what matters.

| Release | What it does | Domains | What Risk IT must deliver first |
| --- | --- | --- | --- |
| 1: read-only, shadow | Dashboards from loaded data; caps computed and compared with the gateway, nothing sent | RR, DS, LD, ST, SV, IA, SD, the monitoring half of OP, TV, QA | The data service and the non-production copy |
| 2: enforcement | Cap commands, uplifts, restores, drift resolution, the kill switch | CC, EN, the command half of OP | The cap calls and the command identity |

That halves the Must count for the first delivery and lets Risk IT ship the data service months before the cap calls.

**Working figures.** Table 5 holds over fifty, and the text says none is agreed. Pick the ten that change the design (load cycle, freshness thresholds, cap-in-force latency, pacing rate, mass-change share, shadow period, uplift ceiling, retention, volume, response time) and get the Global Head to sign them. Make the rest settings with defaults, listed once in the build spec.

## The asks of Risk IT, and what to say if they cannot build them

The 63 Risk IT requirements amount to a platform rebuild specified by the consumer, and the document states no fallback for the ones the enforcement design cannot do without. If any of them is refused or late, release 1 is blocked. Below, each major ask against the document's own Today notes.

| Ask | What the document assumes | What exists today | Fallback stated | Release-1 form to ask for instead |
| --- | --- | --- | --- | --- |
| Positions and safe positions (DS-1, DS-2, LD-10) | A commit-ordered 64-bit position per row, never undercut under concurrent writers, on every view | Dremio views that return whole days of every column; no position, no load time | None | A dedicated, column-restricted view filtered to Risk with an ingest timestamp; Charon reads with an overlap window and re-reads three days nightly |
| Feed epochs (DS-3) | An epoch that changes on any rebuild, with republication of the same ids | Nothing | Partial (LD-11 catches a backwards position) | "Tell us before any rebuild" (DS-13), and the nightly trailing re-read |
| Revisions, withdrawals, redactions as new entries (RR-12, RR-14, DS-6) | Append-only entries with kind, revision and previous values; daily summaries republished with a final flag; a listed person's history republished | No revision, withdrawn flag or kind on any row; the daily view agrees with the request view only to the eleventh decimal | None | Corrections republish the row with a new ingest timestamp; withdrawals set a flag; redaction is a runbook procedure until the retention rule (ST-17) defines erasure |
| Hourly manifest from the write path (DS-15) | A second, independent count per hour | Missing | None | Drop it. Compare against the existing daily view (LD-28) |
| Cap-call semantics (CC-1 to CC-19) | Idempotency keys kept 7 days, expected-value check, per-caller sequence numbers, status-by-key, clear-cap, default-setting, bulk read, restore anchor, snapshot and bulk restore, freeze, overlays, two identities, stated rates | A set call that returns Done and a read that returns a number, one person per call | Only for bulk read, overlays and history | The existing set and read under a non-admin service identity scoped to Risk (CC-4, CC-10), honest failure reasons (CC-7), a gateway-side maximum and no unlimited value (CC-11), a default cap that is never unlimited (CC-5), the fail-open or fail-closed statement (CC-9), Risk IT's own block (CC-18). Overlays with an end time (CC-19) are the one release-2 ask to push for, because they remove Charon's uplift revert entirely |
| The published list (LD-17, DS-6, CC-4) | Risk IT names a route, filters data on the list, scopes the command identity to it, republishes a joiner's history | No route; the service presumably returns all departments | None, and the list's format, acknowledgement and failure handling are unspecified anywhere | Risk IT filters on department (plus agreed principals for non-persons); Charon holds anything unattributable. Revisit if contractors outside the department attribute are common |
| Environment identity (GV-3) | Each service answers an environment call; EN-7 refuses to start without it | Missing | None, and EN-7 makes it a hard start condition in every environment | Separate addresses and credentials, test credentials refused by production. Charon checks its own database tag and configured addresses at start |
| Non-production copy (DS-16, GV-2) | Production schema, a day of production volume, synthetic ids, every listed case, cap calls with a controllable clock | About 1,600 rows a day, one service identity, apparently real ids | Partial (a missing case is a recorded gap) | Keep, with synthetic ids. This is the one ask that must land before anything else |
| 12-month read log (GV-1), OpenAPI under change control (DS-17) | Platform logging and a contract | Nothing | None | Drop. A written column list, example payloads and contract tests on the calls Charon uses |

That is about ten asks instead of 63. The two that gate everything are the dedicated view with an ingest timestamp and the non-production copy; GV-2's four delivery dates should be sought for those first, with the cap calls as a separate, later milestone.

Two further dependencies sit with the database owner. Every migration dry run needs their recorded approval (SD-5), which ties release cadence to one person's calendar; a database of Charon's own removes that. And whether production may sit on the shared development server (EN-7) has no decision, no owner and no date, while "none exists yet" is the document's own answer to a server of Charon's own.

## Over-engineering: what a senior developer would cut, downgrade or keep

The pattern is consistent: the document adopts the cheap structural fix for a failure mode (absolute amounts, a unique constraint, position in the transaction, database time) and then stacks two or three procedural or cryptographic layers on top as if the structural fix might not hold. Every extra layer is one more way for caps to stop moving while a head is on the phone asking why. The uniform depth is itself the tell: CSV formula escaping, WCAG, the budget remainder rule and HMAC key retirement are all specified at the same density, and the changelog's five reviews each added mechanism and removed none.

| Failure mode | Mechanisms stacked in the document | Keep one |
| --- | --- | --- |
| Two writers or out-of-order caps | Per-caller sequence numbers seeded from the gateway (EN-1, CC-2); expected-value compare-and-set (CC-2); claim tokens (EN-3); sender lease and hot standby (EN-4); recompute-and-supersede (EN-5); 5-minute staleness refusal (EN-5); 10-minute signed expiry (EN-9) | One sender that recomputes desired state from the database immediately before each send and sends an absolute amount (EN-5) |
| A forged command row | HMAC command key and a second application key (EN-9); audit-match before signing (EN-9); per-process grants (SD-3); ceilings (EN-5, CC-11) | Database permissions plus a hard maximum at the gateway (CC-11) |
| A cap applied twice | 7-day idempotency keys (CC-1); outcome-unknown state and read-back (EN-2); status-by-key (CC-7) | Absolute amounts plus read-back |
| Two loaders at once | Lease with fencing number on every row (LD-2); position in the transaction (LD-3); idempotent upsert (LD-7) | An application lock plus LD-3 and LD-7 |
| Stale feed unnoticed | In-app alert (SV-6); watchdog on a separate scheduler (OP-5); custom monitor (OP-6); source status (DS-8) | One external check of last successful load, in the firm's existing monitoring |
| Audit tampering | Append-only grants (SD-3); hash chain, chaining job, hourly off-box head hash, nightly verifier (ST-8); nightly export to an immutable store (ST-8) | Deny update and delete, plus the nightly export |
| A mass wrong cap change | Directory shrink guard (LD-20); two-window mass guard with two-person release (EN-10); auto-pause on failures or mismatches (EN-10); 6 changes per caller per day (EN-16); kill switch (EN-6) | LD-20, one threshold ("more than N changes in one pass: hold and alert"), and the kill switch |
| Database restored under Charon | Gateway restore anchor checked every 15 minutes, audit-hash mismatch, sequence reseed, two-person resume (OP-11, CC-15) | Database owner notice, a runbook step, a drift report on start. Do not run production on a server that gets refreshed |
| Clock problems | Database time (OP-13); gateway clock measured every 15 minutes; sender refuses above 5 seconds or after 30 minutes (OP-13) | Database time for every decision |

**Three design changes remove most of the stack.**

1. **Level-triggered reconcile instead of an edge-triggered command machine.** The database holds each caller's desired cap by the EN-12 precedence rule. A worker loop every 10 to 15 seconds reads the gateway caps, diffs them against desired caps, sets the ones that differ with absolute amounts, reads back and records the result. A command-log row written in the decision's transaction (EN-1, the outbox) serves the screen and the audit, not correctness. Restarts, lost answers, superseded decisions, expired uplifts, joiners and drift all collapse into "the next pass fixes it". One column, the last value Charon set and when, tells drift from a pending change. This removes the 12-state machine (EN-2), claims and standby (EN-3, EN-4), signatures and expiry (EN-9), the retry ladders (EN-16), the gateway sequence and compare-and-set rules (CC-2, CC-3), status-by-key (CC-7) and most of OP-11. This is how production reconcilers are normally built.
2. **Charon's own database on the same instance.** A ten-minute task for the database owner. Permissions then bound Charon, row-versioning reads can be turned on (SV-4's degraded mode disappears), backup and restore are Charon's alone, and the parser-based migration runner (SD-5), the statement scanner (SD-4) and the object listings (SD-8) become plain Alembic plus a review of its SQL output. EN-7 already admits production on the shared development server is undecided; decide it, do not mitigate it with 16 requirements.
3. **Load by an ingest timestamp instead of positions.** Risk IT adds a loaded\_at column to a dedicated view. Charon reads everything with loaded\_at later than its last read minus an overlap of about 30 minutes, upserts idempotently (LD-7), and re-reads the trailing three days each night. Late commits and corrections are handled without safe positions (DS-2), epochs (DS-3), revisions with previous values (RR-12), history republishing (DS-6) or the hourly manifest (DS-15). If positions ever exist, "stop if the high-water mark goes backwards" is one line.

**Cut for release 1** (redundant, or the threat does not apply):

- EN-9 signed command and switch rows, both keys, rotation, the compromise procedure, and OP-20's key ceremony. The attacker it stops, with database write access but no host access, can insert the decision and audit rows the reconciler will then sign. An unsigned switch row reading as paused means one rotation mistake halts enforcement.
- ST-8's hash chain, chaining job, hourly head hash and nightly verifier: four scheduled jobs and four logins. The nightly export is the tamper evidence.
- CC-2 sequence numbers and compare-and-set, CC-1 7-day keys, CC-7 status-by-key, CC-12 snapshot restore, CC-15 restore anchor, EN-3 claims, EN-4 standby and pacing classes, EN-5 staleness refusal, LD-2 fencing numbers on every row.
- The watchdog on a separate scheduler (OP-5), the monitor's own copy of uplift end times (EN-13), the daily acknowledged alive message (OP-6), the gateway clock interlock (OP-13), the settings fingerprint across workers (OP-1).
- EN-10 automatic pause on read-back mismatches, EN-16's 6 system changes per caller per day, IA-14 refused sign-in alerts (the firm's single sign-on sees those, Charon does not).
- LD-34 per-department scope rules and ST-6's enforcement test: multi-department is out of scope. LD-32 and LD-35: the normal loader catches up.
- DS-2, DS-3, DS-15, DS-17, DS-6's republishing, CC-11's 500-caller bulk call, GV-1's 12-month read log.

**Downgrade or defer** (right idea, wrong release):

- LD-2 to an application lock and the scheduler's no-overlap setting; LD-4 and LD-5 thresholds to "quarantine, count on the Attention list, skip a page by runbook"; LD-6 and SV-6's six feed states to fresh, delayed, failed.
- LD-9's compare-and-repair to a nightly rebuild of the last 14 days from rows. LD-23 and LD-24's rewrite-with-preview to caller-day rollups with function totals derived at query time by joining the person's current function; at 300 people that join is trivial and "past spend follows the person" comes free. LD-22 redaction to a runbook procedure until ST-17 defines erasure. LD-31 history to the same loader started from the beginning, off-peak.
- ST-13's dual dialect to SQL Server only (LocalDB or Developer edition locally): the document already says every risky behaviour must be tested on SQL Server. ST-10 revisions to a simple history table if corrections prove common.
- SV-3 and TV-5's 200 or 300 virtual users to 50 concurrent at measured volume; SV-5's scoped cache and SV-15's key table deferred (rollups answer in milliseconds; the natural unique constraint covers double clicks); SV-1's grant-level split to a code-review rule.
- EN-10 to one threshold and one admin release; EN-14 to every caller every 30 minutes (about 10 reads a minute); EN-15 to release 2 once the counted cost exists; EN-19 asset caps to release 2, since non-person attribution does not exist today (RR-5, RR-6 margin notes) and non-person spend can sit in the held bucket meanwhile.
- IA-4's eleven two-person actions to four: uplift second approval and no self-approval (EN-21), administrator grants (IA-3, better delegated to the firm's access-request process), release approver not the author (OP-14), and raising a hard ceiling (OP-1, EN-5). Resuming after a pause must be one person, audited; a kill switch that needs two people to re-enable stays off at six on a Friday. IA-11's dated exception machinery to "no admin grants from the directory until LDAPS exists", which IA-3 already says.
- SD-3's thirteen logins to three: web (data only, never calls the gateway), worker (data plus the only holder of the command credential), migrate (schema only). Both runtime logins denied update and delete on audit. Those are the two privilege boundaries an auditor asks about.
- OP-6's custom monitor to the firm's existing monitoring polling the readiness endpoint and one SQL health query (last load older than 20 minutes, any uplift overdue 15 minutes, any failed command older than 10 minutes), in business hours. Charon is not in the request path; if it dies, caps stay as they are. OP-8's 24-hour rota and QA-3's measured 99 percent likewise.
- OP-11 to notice, pause, restore, drift report, resume.

**Keep, and in several cases make Must** (a bank requires these, and most are cheap):

- EN-1 outbox, EN-5 recompute and ceilings, EN-6 one-person kill switch that fails closed, EN-8 shadow mode, EN-12 precedence, EN-13 uplift revert with an overdue alert, EN-14 drift never overwritten silently, EN-18 joiner baseline, EN-21 maker-checker on uplifts in full.
- LD-1, LD-3, LD-4 core, LD-7, LD-8, LD-9 core, LD-12, LD-13, LD-14, LD-15 held bucket, LD-16, LD-18, LD-19, LD-20 simplified, LD-21 core, LD-26, LD-27 simplified, LD-28 against the existing daily view, LD-30.
- ST-1, ST-2, ST-3, ST-4, ST-5, ST-7, ST-8 core (same transaction, append-only by grant, exported nightly), ST-9, ST-11, ST-14, ST-15, ST-17, ST-18, ST-19 as a runbook query.
- SV-2 with TV-7 (a scope leak of per-person spend is the worst defect this product can have), SV-7, SV-8, SV-9, SV-11, SV-13, SV-14, SV-16 timeouts, SV-17.
- IA-1, IA-2, IA-3 simplified, IA-5, IA-6 core, IA-7, IA-8, IA-9, IA-10, and IA-13 quarterly access review raised from Should to Must. A document that makes HMAC rotation a Must and access recertification a Should has its priorities inverted.
- SD-6, SD-7, SD-9, SD-10, SD-11, SD-15. OP-1 core, OP-2, OP-3 short list, OP-5 core, OP-7, OP-9, OP-10, OP-12 short, OP-13 core, OP-14, OP-15, OP-18, OP-19, OP-20 without the ceremony, OP-21. TV-1, TV-2 scoped, TV-3, TV-7, QA-1, QA-4 once OQ-2 is answered with one count query.
- From Risk IT: RR-1, RR-2, RR-3, RR-4, RR-7, RR-8, RR-9, RR-13, RR-15; DS-4, DS-5, DS-7, DS-9, DS-13, DS-16; CC-1, CC-3, CC-4, CC-5, CC-7 core, CC-9, CC-10, CC-11, CC-13, CC-14, CC-18; GV-3, GV-4, GV-5, GV-6.

The release-1 shape that falls out: two deployables (the web app and one singleton worker running the cap-sync loop, the five-minute load and a nightly job) and three logins, in Charon's own database with row-versioning on.

## Gaps: what the backend needs that the document does not cover

For 222 requirements the document is thin in the places a build actually starts from. Each gap below is marked genuinely missing, probably in an illegible passage, or legitimately in the screens document.

1. **Charon's own API and write paths** (missing). SV-1 to SV-18 state properties every route must have, but no route is listed, no request or response body, and no table of which role may call which route, which IA-2's deny-by-default depends on. The record-level specification is absent for uplifts (the state machine from requested through approved, rejected, applied, cancelled, expired; how delegation and an absent head's re-routing are stored), asset registration and retirement (there is no counterpart to LD-21 for an asset, its owner leaving, or a re-owned principal), head appointment and team definition (LD-18 walks to "the first appointed head" but nothing says where heads are appointed), the settings catalogue, and dismissals. The payload contract with the screens team rests on SV-18, a Should that TV-1, a Must, depends on, with no contract artefact, mock server or compatibility policy.
2. **Day one and setup order** (missing; EN-8's body is illegible). There is no inventory of today's caps, no rule for whether existing explicit caps become overrides or are overwritten by tiers at switch-on (the switch-on batch could lower many caps at once, and the signed release bypasses the mass-change guard), no notice to staff whose cap changes, and no stated order for directory read, published list, history load, baseline, reference data, caps. Tier amounts, ceilings and defaults are open questions (OQ-13, OQ-18) with no owner or date.
3. **Hosting** (acknowledged open in OQ-1, with nothing listing what hangs on it). About fifteen requirements cannot be finalised until it is answered: the service manager (OP-7), the second scheduler for the watchdog, the monitor's host (OP-6), how the sign-in identity reaches the API (IA-1, OQ-3), the secrets store, the stop file and off-box audit hash locations, per-process logins, and how several web workers share sessions and cross-site-request state (no session store is named, and the assumptions rule out a cache server). The document needs a decision date and an owner ahead of those builds.
4. **The published list's delivery contract** (missing). Format, full snapshot or change set, acknowledgement that the list is in force (CC-4's "within 5 minutes of delivery" assumes Charon can tell), failure and stale-list handling, and version alignment between the data filter and the cap scope. It also contradicts the constraint "read-only access to every source except the cap call".
5. **Envelopes, budgets and the Attention list** (partly the screens document, partly missing). Nothing says who sets a budget, what counts against an envelope (held spend, asset spend, leavers), or how a retroactive move affects notices already sent. ST-16 is a Should while EN-20, EN-21 and EN-22, all Musts, depend on envelopes existing. The Attention list has no data model, item catalogue or states, although a dozen requirements send items to it.
6. **Data model relationships** (missing). Table 4 lists groups of tables, not entities and keys: person, caller key, caller row, asset, shared principal, function, team, head, envelope, redaction token. ST-18 keys personal data on a pseudonymous person id while every other requirement keys on the Windows id; that is a design-level contradiction, not a wording one.
7. **Alert catalogue** (missing). About forty requirements raise alerts across three channels, with no single list of condition, severity, channel, recipient, deduplication key and runbook entry. Several alerts in the bodies never appear in OP-6's external list (failed command age, pause left on, drift, leaver open cap, quarantine thresholds), so if Charon is down they are silent. Nothing notifies Risk IT automatically when Charon detects a source fault.
8. **Housekeeping** (missing). Staging leftovers, per-cycle records, idempotency keys, export files and expired held batches are all trimmed, and no process in Table 3 owns the job.
9. **Canary** (probably illegible). OQ-24 and the glossary mention a canary identity for cap checks; no legible requirement uses one. If it is absent, nothing proves after go-live that the gateway actually refuses at the cap.
10. **Smaller omissions.** A stated figure with no owner or date in about a dozen places (maximum amount, ceilings, export limits, retention years). Backup of things outside the database (the off-box hash store, the monitor's own state, the signing key). A switch-over plan if the gateway later gains overlays (CC-19). Charon's reaction to the gateway's fail-open window (CC-9). Expected behaviour while Risk IT has frozen the command identity (CC-18) and uplifts keep expiring. Erasure of Charon's own free text (reasons, comments) as opposed to feed data. A local development story (process runner, seed data, directory stub). An escalation path if Risk IT never signs GV-2.

Most of items 1, 2, 5, 6 and 7 are the natural contents of the Charon build spec once it is split from the Risk IT contract, and several shrink under the release-1 simplifications above (fewer processes, fewer alerts, no signing key, no off-box hash).

## Figures that disagree between sections

Check each against the HTML. The second reviewer found 31 disagreements; about 12 are probably misreads of the video, 13 look like real drift, and the rest could be either. The clearest are below, with the likeliest cause.

| Item | One place | Another place | Likely cause |
| --- | --- | --- | --- |
| External stale-feed alert | 20 min (Table 5, LD-6, OP-6) | 30 min (OP-5 acceptance and margin) | Real drift; the arithmetic supports 20 |
| Sender lease | 20 s renewed every 5 s (Table 5) | 30 s renewed every 5 s (EN-4) | Real drift |
| Direct-path total per 30 days | 200 dollars (Table 5, EN-21) | 300 dollars (OQ-14) | Real drift |
| Load-test users | 200 (SV-3, Table 7) | 300 (TV-5, QA-2, index) | Real drift |
| Bulk cap read | 30 s for 2,000 callers (CC-16) | 15 min (Table 11) | Real drift, different units |
| Volume used for sizing | 400,000 a day (DS-10, DS-12 margins) | 650,000 a day grown 50 percent a year (QA-4) | Real drift; Risk IT is sized lower than Charon |
| Cap-call throughput | Risk IT must accept 120 sets and 300 reads a minute (CC-11) | Sender may send up to 140 sets a minute with about three calls each, plus drift reads (EN-4, EN-5, EN-14) | Real drift; the design exceeds the minimum it asks for |
| Pages per cycle against page time | 50 pages in at most 4 min (LD-2) | 5 s per page at the 95th percentile (DS-10), so 250 s | Real drift, minor |
| Batch release rate | 70 a minute (EN-4) | 20 a minute (EN-9 acceptance) | Either |
| Attribution tag length | 100 characters (RR-5, DS-4) | 128 (Table 8) | Either |
| Cap-control numbering | Bodies: CC-2 sequence, CC-3 read, CC-13 cap zero | Index and test-case table: one higher throughout | Probably a renumbering in v2.1 not carried through; several cross-references in Table 19 and OP-18 point at the wrong body |
| Priorities, body against index | LD-35 Could, RR-20 Won't, LD-31 Must | LD-35 Must, RR-20 Could, LD-31 Should | Pale badges misread, or drift; and only one Won't can be found against the header's two |
| Read-only sources | "Read-only access to every source except the cap call" (Constraints) | Charon writes the published list to Risk IT (LD-17, DS-6) | Real drift |
| Envelope priority | ST-16 is Should | EN-20, EN-21, EN-22 Musts depend on envelopes | Real drift, a dependency inversion |
| Person key | Pseudonymous person id for personal data (ST-18) | Windows id as the key everywhere else (RR-4, Table 8, LD-17, ST-3) | Real drift at design level |
| Blocked and throttled cost | Throttled carries cost 0 (ST-2) | RR-8 and RR-9 say nothing about throttled | Real drift, minor |

About a dozen cross-references point at the wrong domain code (CV-3 for GV-3, OC and OB for CC and OP, OA-1 for QA-4, SD for DS-8); those are almost certainly misreads and will not need fixing.

## Smaller fixes before it goes out

- **Tone.** "Every requirement here is a requirement of the owner, not a request" will land badly with a team that has not yet agreed to own anything. Reframe the split as proposed, for agreement, and let GV-2's sign-off make it binding.
- **"sid".** The document uses sid for the Windows login name. To a Windows administrator, SID is the security identifier, a different thing. Confirm what user\_sid actually holds and call it login id throughout.
- **Volume honesty.** The estimate spans 150,000 to 650,000 requests a day and OQ-2 says a count has not been permitted. Say in one line that sizing is a guess until the count is run, and that the load test and retention design will be re-done at the measured figure (QA-4 already allows for this).
- **Stack note.** Async SQLAlchemy 2 on SQL Server depends on aioodbc, which is less mature than the Postgres drivers. Sync SQLAlchemy with a worker pool is the safer choice on this stack; one sentence in the assumptions avoids a nasty surprise in week two.
- **Who is the Charon team.** The document never says. Risk IT will ask, and the answer decides whether the control design is proportionate. If it is one person plus a coding assistant, say so and size the first release to it.
- **Positions column.** Table 5's "Where it is required" column and the per-domain counts in Table 2 could not be read; check they still match the bodies after the next edit.
- **Glossary.** The budget-division rule in ST-16 (parent amount divided by children plus one) is a business rule and belongs in the screens document, not the backend.

## Suggested order of changes

1. **Get two numbers before editing anything.** Run the one count query behind OQ-2 and settle the hosting question (OQ-1) with a date and an owner. Half the open figures and fifteen requirements depend on them.
2. **Decide the three design simplifications** with whoever owns the build: reconcile loop, own database, ingest-timestamp loading. Record each as a line in the Design Choices table with what it replaces.
3. **Split the document.** Pull RR, DS, CC, GV, Data Shapes, the non-production cases and Risk IT's open questions into an interface contract with a one-page summary at the front. Keep everything else as the Charon build spec.
4. **Cut the Risk IT contract to about ten asks**, separated into things to build and things to confirm in writing, with the dedicated view and the non-production copy as the first milestone and the cap calls as the second. State the fallback beside each ask.
5. **Re-prioritise the build spec against two releases.** Must means needed for the read-only shadow release; Should means needed for enforcement; Could means later. Apply the cut and downgrade lists. Raise IA-13 to Must.
6. **Reduce the two-person approvals to four** and the logins to three, and replace the custom monitor with the firm's monitoring polling the readiness endpoint and one health query.
7. **Fix the figures table** against the HTML, then make Table 5 ten agreed values plus a settings register with defaults.
8. **Fill the four gaps the build cannot start without:** the route and permission table, the uplift and asset records, the setup order with today's cap inventory, and the alert catalogue.
9. **Reframe the tone** from "a requirement of the owner, not a request" to a proposed split for agreement, and rename sid to login id.
10. **Send the interface contract to Risk IT** with GV-2's dates requested for the first milestone only.
