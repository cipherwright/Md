# Charon backend build spec

Oct 28, 2026 · @K

## 1. The shape

Three processes, one database. The poller is separate and pulls on its own timer; the app never talks to a source. This is the answer to "will we have a different polling system that pulls independently and then connect the app to it": yes, and the connection is the database, nothing else.

| Process | Built by | Writes | Reads | Runs |
| --- | --- | --- | --- | --- |
| Feed poller | Risk IT, to the requirements in section 3 | Landing tables (one per feed), feed\_health | Dremio, LDAP, AD, the gateway | Their scheduler, every 5 minutes (directory, groups, models every 6 hours) |
| Charon ingest | Us (this spec) | Base tables, rollups, closures, ingest\_status, cap sync\_state | Landing tables, outbox | Scheduler every 5 minutes, exits when done |
| Charon API | Us (this spec) | cap, cap\_change, uplift tables, alert state, notification reads, audit\_log | Base tables, rollups, closures, ingest\_status | 4 worker processes behind the SSO reverse proxy |

The E2 command path is the one place the poller writes back to a source: Charon's API inserts a cap\_change row with sync\_state pending; Charon ingest copies pending rows into a command landing table (enforcement\_command); the poller reads that table, calls the gateway, and writes the result row (enforcement\_result); Charon ingest reads results and sets sync\_state applied or failed. Everything crosses in tables, both directions.

Stack, fixed for this build: Python 3.11, FastAPI with sync endpoints (def, not async def), SQLAlchemy 2 Core and ORM with pyodbc and ODBC Driver 17 or 18, Alembic for migrations, pydantic v2 for feed row validation, React front end unchanged. SQL Server 2019 or later in production, SQLite only for unit tests that do not touch SQL dialect features. No Redis, no queue, no APScheduler inside the API.

Until Risk IT's poller exists, the simulator (section 7) writes the same landing tables, so Charon ingest and the API are built and tested against the real shape from day one.

## 2. Repository layout

One repository, one Python package, three entry points. The domain and service layer imports nothing from FastAPI; routes are a thin shell. Keep the existing tests and models where they are and move code into this layout rather than rewriting it.

```markdown
charon/
  pyproject.toml
  alembic/                      migrations, SQL Server dialect, schema name from settings
  charon/
    settings.py                 pydantic BaseSettings, env prefixed CHARON_
    db.py                       engine factory (pyodbc, fast_executemany=True), session factory, schema
    models/                     SQLAlchemy ORM models, one module per area (org, callers, spend, caps, control, landing)
    feeds/
      contract.py               pydantic row models per feed (S1Row, S2Row, C1Row, E1Row, P1Row, P2Row, A1Row, M1Row, O1Row)
      landing.py                read pages from landing tables after a watermark
      http.py                   same interface over the HTTPS feed (simulator today, optional later)
    ingest/
      run.py                    entry point: charon-ingest --feeds spend,activity | directory,groups,models | all
      lease.py                  take and release the per-feed lease row
      stage.py                  truncate staging, bulk insert a validated page
      merge/                    one module per feed: merge_s1.py, merge_s2.py, merge_c1.py, merge_e1.py, merge_p1.py, merge_p2.py, merge_a1.py, merge_m1.py
      rollup.py                 recompute spend_daily, function_day, function_day_dim for touched groups
      closure.py                rebuild person_closure and function_closure
      placement.py              walk the manager chain to the first function head; HQ fallback; cycle guard
      outbox.py                 copy pending cap changes to enforcement_command; read enforcement_result
      reconcile.py              S1 versus S2 daily control totals; marks days reconciled or not
      archive.py                move months older than 13 from spend_record to spend_record_archive
      health.py                 write ingest_status and freshness
    domain/
      states.py                 alert and envelope state rules (pure functions)
      caps.py                   effective cap, uplift expiry (pure functions)
      forecast.py               current month forecast (pure function)
    services/
      scope.py                  Scope dataclass from a signed-in person; the SQL fragments it yields
      cache.py                  per-worker PayloadCache keyed by (name, scope_key, watermark, version)
      overview.py               build_overview(session, scope) and the three tables
      people.py                 build_people(session, scope)
      chart.py                  build_chart_13m(session, scope)
      attention.py              build_attention(session, scope)
      caps.py                   set_cap, request_uplift, decide_uplift (write paths, audit, outbox row)
      freshness.py              as_of(session, feeds) from ingest_status
    web/
      app.py                    FastAPI app factory, middleware, routers
      auth.py                   SSO cookie to person; role and scope resolution; dev bypass
      routes/                   overview.py, people.py, chart.py, attention.py, caps.py, system.py
      timing.py                 middleware writing request_log rows
  simulator/                    existing simulator plus a landing-table writer (section 7)
  tests/
    unit/                       domain and services against SQLite where dialect allows
    sqlserver/                  merge, rollup, concurrency and plan tests; skipped unless CHARON_TEST_MSSQL_URL is set
    load/                       locust or k6 scripts for section 8 marks
```

Rules the coding model follows everywhere:

1. Services take a SQLAlchemy Session and plain arguments, return plain dicts or dataclasses. No Request, no Depends, no HTTP status codes below web/.
2. Ingest merge and rollup steps are hand-written SQL strings in the merge modules, parameterised, executed with session.execute(text(...)). No ORM in the hot path.
3. Every timestamp column is UTC, DATETIME2(6), named \*\_ts or \*\_at. Days are DATE columns named usage\_day or day.
4. Every table carries department\_id. Every Charon-owned table that users edit carries version INT for optimistic concurrency.
5. Money is NUMERIC(18,8) named \*\_usd.

## 3. Database

One SQL Server database, schema name from settings (default charon). Two settings are applied by the first migration and the build fails its SQL Server test suite if they are off:

```sql
ALTER DATABASE CURRENT SET READ_COMMITTED_SNAPSHOT ON WITH ROLLBACK IMMEDIATE;
ALTER DATABASE CURRENT SET ALLOW_SNAPSHOT_ISOLATION ON;
```

### 3.1 Landing tables (the contract Risk IT's poller writes to)

These are the requirement handed to Risk IT. One table per feed, append-only from the poller's side, columns exactly as the feed in Appendix A plus the four control columns. The poller never updates a landing row; a correction is a new row with a higher load\_seq and revision. Charon ingest reads and never deletes; a monthly job removes landing rows older than 35 days whose load\_seq is below every feed's watermark.

```sql
CREATE TABLE charon.landing_s1_spend (
  load_seq           BIGINT        NOT NULL,   -- monotonic per feed, poller assigns
  loaded_at          DATETIME2(6)  NOT NULL,
  request_id         VARCHAR(64)   NOT NULL,
  event_ts           DATETIME2(3)  NOT NULL,
  caller_id          VARCHAR(128)  NOT NULL,   -- uid for a person, principal otherwise
  caller_uid         VARCHAR(64)   NULL,
  caller_kind        VARCHAR(16)   NOT NULL,   -- person | non_person | unknown
  caller_type        VARCHAR(32)   NULL,       -- agentic_workflow | pipeline | application | unknown
  caller_principal   VARCHAR(256)  NULL,
  attribution_tag    VARCHAR(128)  NULL,
  owning_department  VARCHAR(64)   NULL,
  model_id           VARCHAR(128)  NOT NULL,
  api                VARCHAR(32)   NULL,
  input_tokens       BIGINT        NULL,
  output_tokens      BIGINT        NULL,
  cache_read_tokens  BIGINT        NULL,
  cache_write_tokens BIGINT        NULL,
  cost_usd           NUMERIC(18,8) NULL,
  cost_basis         VARCHAR(32)   NULL,       -- computed_list_price | billed | estimate
  long_context       BIT           NULL,
  outcome            VARCHAR(32)   NOT NULL,   -- served | blocked_by_cap | throttled | error
  cap_in_force_usd   NUMERIC(18,8) NULL,       -- required when outcome = blocked_by_cap
  session_id         VARCHAR(128)  NULL,
  client             VARCHAR(64)   NULL,
  account_id         VARCHAR(64)   NULL,
  region             VARCHAR(32)   NULL,
  revision           INT           NOT NULL DEFAULT 1,
  withdrawn          BIT           NOT NULL DEFAULT 0,
  schema_version     SMALLINT      NOT NULL,
  CONSTRAINT pk_landing_s1 PRIMARY KEY CLUSTERED (load_seq, request_id)
);
```

The other landing tables follow the same pattern with their own fields: landing\_s2\_daily (caller\_id, usage\_day, model\_id, requests, tokens, cost\_usd, blocked\_requests, is\_final), landing\_c1\_caller, landing\_e1\_enforcement, landing\_p1\_person, landing\_p2\_group\_member, landing\_a1\_activity, landing\_m1\_model, landing\_o1\_feed\_health (feed, last\_success\_at, newest\_event\_ts, expected\_cadence\_s, rows\_last\_load, schema\_version, status, reason). Every one has load\_seq, loaded\_at, revision, withdrawn, schema\_version and a clustered primary key on (load\_seq, natural key).

Two command tables cross the other way:

```sql
CREATE TABLE charon.enforcement_command (
  command_id       UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,  -- Charon's idempotency key
  command          VARCHAR(32) NOT NULL,   -- set_cap | set_tier_default | disable | reinstate
  caller_id        VARCHAR(128) NOT NULL,
  amount_usd       NUMERIC(18,8) NULL,
  effective_from   DATETIME2(6) NOT NULL,
  effective_until  DATETIME2(6) NULL,
  reference        VARCHAR(128) NULL,
  reason           NVARCHAR(512) NULL,
  created_at       DATETIME2(6) NOT NULL
);
CREATE TABLE charon.enforcement_result (
  command_id       UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,
  status           VARCHAR(16) NOT NULL,   -- applied | pending | failed
  applied_amount_usd NUMERIC(18,8) NULL,
  applied_at       DATETIME2(6) NULL,
  reason           NVARCHAR(512) NULL,
  load_seq         BIGINT NOT NULL
);
```

### 3.2 Base tables that change from the SQLite version

Keep the entity model as it is, with these specific changes.

| Table | Change |
| --- | --- |
| spend\_record | Primary key request\_id NONCLUSTERED. Clustered index on (event\_ts, request\_id). Add load\_id BIGINT, revision INT, withdrawn BIT, person\_uid VARCHAR(64) NULL. Nonclustered (caller\_id, event\_ts DESC) INCLUDE (model\_id, cost\_usd, outcome, cap\_in\_force\_usd, withdrawn). Nonclustered (load\_id). Nonclustered (function\_id, event\_ts). No partitioning in this build |
| spend\_record\_archive | Same columns, same clustered index, no nonclustered indexes. Empty until the archive job runs |
| spend\_daily | Primary key (caller\_id, usage\_day, model\_id). Add source\_feed VARCHAR(8) (S1 or S2), reconciled BIT, blocked\_requests INT. Index (usage\_day, caller\_id) INCLUDE (cost\_usd, calls, blocked\_requests). Index (function\_id, usage\_day) |
| function\_day (new) | (function\_id, usage\_day, caller\_kind) primary key; cost\_usd, calls, input\_tokens, output\_tokens, cache\_read\_tokens, cache\_write\_tokens, blocked\_requests, active\_callers. Index (usage\_day) |
| function\_day\_dim (new) | (function\_id, usage\_day, source\_type, model\_tier, region\_id, model\_id) primary key; cost\_usd, calls, tokens |
| function\_closure (new) | (ancestor\_function\_id, descendant\_function\_id) primary key, depth INT |
| person\_closure (new) | (manager\_uid, person\_uid) primary key, depth INT. Includes depth 0 (self) rows so a lead's own spend is in the subtree |
| ingest\_status | One row per feed: feed, lease\_until, lease\_owner, watermark\_seq, last\_run\_at, last\_success\_at, newest\_event\_ts, rows\_last\_load, run\_seconds, status, error, version |
| ingest\_load | One row per ingest run per feed: load\_id IDENTITY, feed, started\_at, finished\_at, from\_seq, to\_seq, rows\_inserted, rows\_updated, status |
| cap\_change | Add sync\_state (pending, sent, applied, failed), command\_id UNIQUEIDENTIFIER, sent\_at, result\_at, result\_reason |
| settings | Add rows: request\_level\_from DATE, data\_version BIGINT (bumped by every user write), ingest\_interval\_s, overlap\_seq (default 0) |
| request\_log (new) | id IDENTITY, at, route, person\_uid, status, ms, cache\_hit BIT. Index (at). Trimmed to 7 days |

Filtered unique indexes carry over as they are in the entity model (one open head per function, one open function history row per person, one dedupe\_key per alert per department).

## 4. Ingest process

charon-ingest is a command line program that runs one cycle for a set of feeds and exits. The scheduler starts it every 5 minutes with --feeds spend,activity and every 6 hours with --feeds directory,groups,models. A cycle is: for each feed in order (C1, M1, P1, P2, S1, S2, E1, A1, O1, then outbox, then reconcile), take the lease, pull pages, stage, merge, rollup, advance watermark, write health, release.

### 4.1 Run loop, per feed

```markdown
run_feed(feed):
  with session.begin():
    if not lease.take(session, feed, owner=hostname+pid, seconds=600): return SKIPPED
  status = ingest_status.get(feed)
  from_seq = max(0, status.watermark_seq - settings.overlap_seq)
  load = ingest_load.start(feed, from_seq)
  try:
    while True:
      page = adapter.read_page(feed, after_seq=from_seq, limit=10000)   # ordered by load_seq, natural key
      if not page.rows: break
      rows = [FeedRow.model_validate(r) for r in page.rows]            # ValidationError aborts the run, watermark intact
      with session.begin():
        stage.truncate(session, feed)
        stage.bulk_insert(session, feed, rows, load.load_id)            # fast_executemany
        touched = merge_<feed>.merge(session, load.load_id)             # returns touched group keys
        rollup.recompute(session, touched)                              # only for S1, S2, A1
        ingest_status.set_watermark(session, feed, page.max_seq, rows=len(rows))
      from_seq = page.max_seq
      if not page.has_more: break
    ingest_load.finish(load, OK)
    health.write(session, feed, newest_event_ts=..., status=OK)
  except Exception as e:
    ingest_load.finish(load, FAILED, str(e)); health.write(..., status=FAILED, error=str(e)); raise
  finally:
    lease.release(session, feed)
```

The watermark and the rows it covers commit in the same transaction. A crash leaves the watermark at the last committed page and the next run continues from there. The validate step runs before the transaction so a bad page never opens one.

### 4.2 Adapters

feeds/landing.py implements read\_page by SELECT TOP (:limit) \* FROM landing\_\<feed> WHERE load\_seq > :after ORDER BY load\_seq, \<natural key>. has\_more is true when the page is full. feeds/http.py implements the same signature against the simulator's HTTPS feed with bearer token, schema\_version check, 503 raised as FeedUnavailable. Both return Page(rows: list\[dict\], max\_seq: int, has\_more: bool). The adapter is chosen by settings.feed\_transport.

### 4.3 Merge for S1 (the pattern every merge follows)

```sql
-- 1. stamp function, region and person at ingest time
UPDATE s SET
  s.person_uid  = CASE WHEN s.caller_kind = 'person' THEN s.caller_uid END,
  s.function_id = COALESCE(p.function_id, c.function_id, :hq_function_id),
  s.region_id   = COALESCE(p.region_id, :unknown_region_id)
FROM charon.stage_s1 s
LEFT JOIN charon.caller c ON c.id = s.caller_id
LEFT JOIN charon.person p ON p.uid = s.caller_uid;

-- 2. unseen callers get a caller row (every identity seen gets a row)
INSERT INTO charon.caller (id, source_type, person_uid, function_id, status, attribution_state, first_seen_at, last_seen_at, department_id)
SELECT DISTINCT s.caller_id, s.caller_kind, s.person_uid, s.function_id, 'active',
       CASE WHEN s.person_uid IS NOT NULL THEN 'attributed' ELSE 'unattributed' END, s.event_ts, s.event_ts, :dept
FROM charon.stage_s1 s WHERE NOT EXISTS (SELECT 1 FROM charon.caller c WHERE c.id = s.caller_id);

-- 3. corrections: higher revision or newly withdrawn wins
UPDATE t SET
  t.event_ts = s.event_ts, t.model_id = s.model_id, t.cost_usd = s.cost_usd, t.cost_basis = s.cost_basis,
  t.input_tokens = s.input_tokens, t.output_tokens = s.output_tokens,
  t.cache_read_tokens = s.cache_read_tokens, t.cache_write_tokens = s.cache_write_tokens,
  t.outcome = s.outcome, t.cap_in_force_usd = s.cap_in_force_usd,
  t.revision = s.revision, t.withdrawn = s.withdrawn, t.load_id = :load_id
OUTPUT deleted.caller_id, CAST(deleted.event_ts AS DATE), deleted.model_id,
       inserted.caller_id, CAST(inserted.event_ts AS DATE), inserted.model_id
  INTO #touched (caller_id, usage_day, model_id, caller_id2, usage_day2, model_id2)
FROM charon.spend_record t JOIN charon.stage_s1 s ON s.request_id = t.request_id
WHERE s.revision > t.revision OR (s.withdrawn = 1 AND t.withdrawn = 0);

-- 4. new rows
INSERT INTO charon.spend_record (request_id, event_ts, caller_id, person_uid, function_id, region_id, model_id, ... , revision, withdrawn, load_id, department_id)
OUTPUT inserted.caller_id, CAST(inserted.event_ts AS DATE), inserted.model_id INTO #touched (caller_id, usage_day, model_id)
SELECT s.request_id, s.event_ts, s.caller_id, s.person_uid, s.function_id, s.region_id, s.model_id, ... , s.revision, s.withdrawn, :load_id, :dept
FROM charon.stage_s1 s
WHERE NOT EXISTS (SELECT 1 FROM charon.spend_record t WHERE t.request_id = s.request_id);
```

The merge function returns the distinct (caller\_id, usage\_day, model\_id) triples from #touched (both old and new keys of a corrected row, so a row that moved day recomputes both days).

### 4.4 Rollup recompute

```sql
-- spend_daily for touched groups, from base rows, withdrawn excluded
MERGE is not used. Two statements per batch of touched keys passed as a table valued parameter or a temp table #k:
UPDATE d SET cost_usd = x.cost_usd, calls = x.calls, input_tokens = x.input_tokens, ... , blocked_requests = x.blocked, source_feed = 'S1', reconciled = 0
FROM charon.spend_daily d JOIN (SELECT r.caller_id, CAST(r.event_ts AS DATE) usage_day, r.model_id, SUM(r.cost_usd) cost_usd, COUNT(*) calls, ..., SUM(CASE WHEN r.outcome='blocked_by_cap' THEN 1 ELSE 0 END) blocked
  FROM charon.spend_record r JOIN #k k ON k.caller_id = r.caller_id AND k.model_id = r.model_id AND CAST(r.event_ts AS DATE) = k.usage_day
  WHERE r.withdrawn = 0 GROUP BY r.caller_id, CAST(r.event_ts AS DATE), r.model_id) x ON x.caller_id = d.caller_id AND x.usage_day = d.usage_day AND x.model_id = d.model_id;
INSERT ... same SELECT ... WHERE NOT EXISTS in spend_daily;
DELETE d FROM charon.spend_daily d JOIN #k k ON ... WHERE NOT EXISTS (SELECT 1 FROM charon.spend_record r WHERE ... AND r.withdrawn = 0) AND d.source_feed = 'S1';
```

Then function\_day and function\_day\_dim are recomputed for the distinct (function\_id, usage\_day) pairs those callers map to, from spend\_daily joined to caller and model\_tier, with the same update, insert, delete pattern. The function\_id used is the one stamped on spend\_record for that day, read through spend\_daily.function\_id, which is set from the stamped rows in the same step. A day is "today" or any past day; the code never special-cases today.

### 4.5 Directory, closures and placement

P1 merge upserts person by uid. After the merge, placement.py runs for every person whose manager, function override or active flag changed in this load: walk manager uid upward at most 20 steps until a uid with an open function\_head\_assignment is found; that head's function is the person's function; no head found means HQ; an administrator override wins. A changed function closes the open person\_function\_history row and opens a new one in the same transaction. Then closure.py rebuilds person\_closure in full (910 people, under a second) with a recursive CTE, truncating and reinserting inside the transaction; function\_closure likewise whenever a function row changes.

### 4.6 Outbox drain and results

After the feeds: INSERT INTO enforcement\_command SELECT ... FROM cap\_change WHERE sync\_state = 'pending' AND command\_id NOT IN enforcement\_command, then set sync\_state = 'sent', sent\_at = now. Then read enforcement\_result rows with load\_seq above the outbox watermark and set the matching cap\_change and cap rows to applied (with applied\_amount, result\_at) or failed (with reason). E1 confirms independently; if E1 shows a cap that differs from Charon's applied cap for more than two intervals, open an alert of kind cap\_drift.

### 4.7 Reconcile and archive

reconcile.py runs at the end of the 5 minute cycle for days where S2 has is\_final = 1 and spend\_daily.reconciled = 0: compare SUM(cost\_usd) and SUM(tokens) per (caller, day, model) between spend\_daily (S1) and landing S2; within 0.5 percent on cost and exact on tokens sets reconciled = 1, otherwise an alert of kind reconcile\_mismatch keyed on the day. Days before request\_level\_from are loaded straight from S2 into spend\_daily with source\_feed = 'S2' and reconciled = 1.

archive.py runs monthly: INSERT INTO spend\_record\_archive SELECT ... WHERE event\_ts < first day of (current month minus 13), in batches of 50,000 by event\_ts range, then DELETE the same range, each batch its own transaction. Rollups are untouched; they are kept for all history.

## 5. Service layer

Every screen is one build\_\* function that takes a Session and a Scope and returns a dict ready to serialise. The function first asks the cache, then runs its queries, then applies the pure domain rules, then stores the result.

### 5.1 Scope

```markdown
@dataclass(frozen=True)
class Scope:
    person_uid: str
    role: Role                         # global_head | function_head | regional_head | team_lead | user | central
    function_ids: frozenset[int]       # the function subtree(s) this role may see, resolved through function_closure
    region_id: int | None              # regional heads only
    lead_uid: str | None               # team leads: subtree root in person_closure
    only_self: bool                    # ordinary users
    can_write_caps: bool
    def key(self) -> str               # stable string for the cache key, e.g. 'fh:12' or 'tl:abc123' or 'all'
```

scope.resolve(session, person\_uid) builds this from role\_assignment, function\_head\_assignment, regional\_head\_assignment and person. Each build\_\* function gets its WHERE fragment from scope.spend\_daily\_filter() or scope.function\_filter(), which return SQLAlchemy clauses: a join to person\_closure on manager\_uid = lead\_uid for team leads, function\_id IN (...) for function heads, region\_id = ... for regional heads, caller.person\_uid = ... for ordinary users, no filter for global head and central. There is no code path that builds a screen without a Scope.

### 5.2 Cache

```markdown
class PayloadCache:
    def get(self, name, scope_key, watermark, data_version) -> dict | None
    def put(self, name, scope_key, watermark, data_version, payload) -> None
    # dict in process memory, max 500 entries, LRU, no TTL: the key changes every cycle
```

watermark is ingest\_status.watermark\_seq for S1 (or the max over the feeds a screen uses); data\_version is settings.data\_version, bumped by every user write (cap, envelope, head, role). Both are read on every request with one indexed query each, so a payload can never be served against stale user writes. One PayloadCache per worker process, created in the app factory.

### 5.3 Builders and their queries

| Function | Queries, in order | Then |
| --- | --- | --- |
| build\_overview(session, scope) | function tree in scope with heads; function\_day from first of previous month to today joined to function\_closure, grouped by ancestor (tree totals); envelope for the current period; counts from uplift\_request (pending) and blocked\_daily (today) per function; spend\_daily today joined to person\_closure for the teams table (one query grouped by manager\_uid); the people rows via build\_people | domain.forecast.month\_forecast(month\_to\_date, elapsed\_days, days\_in\_month, same\_days\_last\_month); domain.states.envelope\_state(spend, envelope, threshold\_pct, forecast); share and change on last month in Python |
| build\_people(session, scope) | person join caller join cap join spend\_daily (today, grouped by caller) left join blocked\_daily today, filtered by scope; one query | domain.caps.effective\_cap(base, uplift\_amount, uplift\_expiry, now); domain.states.person\_state(spend\_today, effective\_cap, blocked) |
| build\_chart\_13m(session, scope) | function\_day summed by month for 13 months, scope applied via function\_closure; one query | forecast for the current month |
| build\_attention(session, scope) | alert where state = open in scope; uplift\_request pending in scope; blocked\_daily today in scope; three small queries | sort by severity then age |
| build\_my\_spend(session, person\_uid) | spend\_daily for the person's callers, last 31 days, by day and model; one query |  |
| build\_breakdowns(session, scope, node) | function\_day\_dim for the node's subtree and the window | grouped four ways in Python |
| build\_drawer\_requests(session, scope, caller\_id, limit=200) | spend\_record by (caller\_id, event\_ts DESC), scope checked first | not cached |
| freshness.as\_of(session, feeds) | ingest\_status rows for the feeds | min(last\_success\_at), max lag, state fresh, delayed, not\_updating |

### 5.4 Write paths

services/caps.py: set\_cap(session, scope, caller\_id, amount\_usd, until, reason) checks can\_write\_caps and that the caller is in scope, inserts cap\_change (sync\_state pending, command\_id uuid4), updates cap (version check, optimistic), writes audit\_log with before and after JSON, bumps settings.data\_version, all in one transaction. request\_uplift and decide\_uplift follow the same shape; a decision that approves calls set\_cap internally. Nothing in the write path talks to the gateway.

Domain rules in charon/domain are pure functions with no session and are the only place the thresholds (75, 90, 100 percent; function.alert\_threshold\_pct default 80) are interpreted. They are unit tested with no database.

## 6. Web layer

FastAPI with plain def endpoints so each runs in the thread pool and uses sync SQLAlchemy. Served by uvicorn with --workers 4 on Linux or by waitress behind IIS or the SSO reverse proxy on Windows; the app factory is the same. No async def anywhere in the project.

| Route | Service call | Cache | Notes |
| --- | --- | --- | --- |
| GET /api/overview | build\_overview | yes | Payload includes as\_of |
| GET /api/people?filter= | build\_people | yes | Filters applied in Python on the cached 900 rows |
| GET /api/chart/13m?node= | build\_chart\_13m | yes |  |
| GET /api/attention | build\_attention | yes, data\_version in key |  |
| GET /api/me/spend | build\_my\_spend | no | Small |
| GET /api/breakdowns?node=&window= | build\_breakdowns | yes |  |
| GET /api/callers/{id}/requests | build\_drawer\_requests | no | Scope check before query |
| POST /api/caps/{caller\_id} | set\_cap | bumps data\_version | Returns the cap row with sync\_state pending |
| POST /api/uplifts, POST /api/uplifts/{id}/decision | request\_uplift, decide\_uplift | bumps data\_version |  |
| GET /api/system/ingest | ingest\_status rows, last 20 ingest\_load rows | no | Central role only |
| GET /api/system/metrics | request\_log p95 by route for 5 min and 1 h, cache hits, outbox pending age | no | Plain JSON for Risk IT monitoring to poll |

Auth: web/auth.py reads the SSO session cookie the same way Bow Wow does (reuse their module if it can be imported, otherwise copy the cookie validation), maps it to a person by uid, resolves Scope once per request, and caches the Scope in process for 60 seconds keyed by uid and data\_version. A dev bypass header CHARON\_DEV\_UID is honoured only when settings.env = dev.

Timing middleware writes one request\_log row per request after the response is sent, with route, status, milliseconds and whether the cache hit. Errors return a JSON body {error, reason}; a FeedUnavailable from a service becomes 503 with reason; an empty result is 200 with empty arrays.

Every response carries as\_of: {at, state, feeds: \[{feed, last\_success\_at, lag\_s, state}\]} so the React chip needs no second call.

## 7. Simulator

The simulator keeps its HTTPS feed and gains a second output: it writes the landing tables of section 3.1 directly, with the same load\_seq numbering and the same page shape, so Charon ingest runs against the landing adapter from the first day. Add simulator --target landing --db \<url> --advance 5m and --target landing --rebuild --days 45. The seed, the S curve, the weekday rhythm and the case list in Appendix A stay. Two additions: it writes enforcement\_result rows for every enforcement\_command it finds (applied after 30 to 60 seconds, a configurable share failed), and it emits landing\_o1\_feed\_health rows every cycle so freshness logic can be tested without a real poller.

The simulator's load\_seq must be monotonic per feed with occasional gaps (it already numbers pages; add a random skip of 1 to 50 every few pages) so the gap handling is exercised, and it must re-serve a corrected row with a higher load\_seq and revision exactly as rule 4 says.

## 8. Tests the build must pass

The existing 589 tests stay green. These are added, grouped by where they run. The SQL Server group runs only when CHARON\_TEST\_MSSQL\_URL is set and is required before anything is handed to Risk IT.

| Group | Test | Pass condition |
| --- | --- | --- |
| unit | Domain states for every threshold edge (74.99, 75, 89.99, 90, 99.99, 100, 100.01 percent) | Exact state per Appendix B |
| unit | Effective cap with uplift active, expired, future | Appendix B rule |
| unit | Forecast on day 1, mid month, last day, no data last month | No division by zero, sensible numbers |
| unit | Placement walk: head at depth 1, 5, none, cycle, override | Correct function or HQ; cycle terminates |
| sqlserver | RCSI assertion | Fails the suite if READ\_COMMITTED\_SNAPSHOT is off |
| sqlserver | Merge idempotency: same page twice | Row counts and rollups identical after the second run |
| sqlserver | Correction then withdrawal of one row | spend\_daily and function\_day equal SUM over spend\_record for every touched group |
| sqlserver | Row that moves day on correction | Both days recomputed |
| sqlserver | Crash between page commits (raise after page 2 of 3) | Watermark at page 2; rerun completes; no duplicates |
| sqlserver | 23:59:59.999 and 00:00:00.000 UTC rows | Land in the right usage\_day |
| sqlserver | Reconciliation test from Appendix B on every screen | Sum of caller spend equals function spend per function |
| sqlserver | Scope leak, one test per role | Payload contains no person, function or region outside scope |
| sqlserver | Concurrent reads during a 10,000 row merge (threads) | No read waits over 100 ms |
| sqlserver | Outbox: 20 concurrent set\_cap during a drain | No deadlock, each command sent once, each result applied once |
| sqlserver | Query plans for build\_overview and build\_people | No clustered index scan on spend\_daily or spend\_record (check sys.dm\_exec\_query\_plan or SET SHOWPLAN\_XML) |
| sqlserver | Backfill 2.5 million rows through staging | Under 5 minutes on the test server; time recorded |
| sqlserver | Archive moves one month | Counts match; drawer for a day in the archived month returns rows from the archive table |
| load | Section 6 of the review marks, with k6 or locust scripts in tests/load | p95 within marks at fixture F1; results written to a markdown report |

## 9. Build order

Each phase ends with a check that must pass before the next starts. The coding model should work one phase per session, keep the existing test suite green throughout, and not touch the React code until phase 6.

1. Phase 1, database. Alembic migration for SQL Server: schema settings, RCSI, landing tables, command tables, base table changes, rollups, closures, ingest\_status, ingest\_load, request\_log, settings rows. Check: migrate up and down on SQL Server and on SQLite (SQLite skips RCSI and filtered index syntax via dialect branches); autogenerate shows no drift.
2. Phase 2, simulator landing output. --target landing writes all landing tables and enforcement\_result. Check: 45 days built; load\_seq monotonic with gaps; a corrected row appears twice with rising revision.
3. Phase 3, ingest for S1 and S2 only. Lease, landing adapter, validate, stage, merge\_s1, merge\_s2, rollup, watermark, health. Check: idempotency, correction, crash and day boundary tests pass; backfill time recorded.
4. Phase 4, ingest for the rest. C1, E1, P1 with placement and closures, P2, A1, M1, O1, outbox drain and results, reconcile, archive. Check: reconciliation and placement tests pass; a full cycle on the 45 day fixture under 60 seconds.
5. Phase 5, services. Scope, cache, the builders, write paths, freshness; domain functions moved to charon/domain. Check: scope leak tests, query plan tests, concurrent read test; build\_overview under 500 ms cold on the fixture.
6. Phase 6, web. App factory, auth, routes, timing middleware, system endpoints; React pointed at the new routes and the as\_of chip added. Check: every screen renders from the landing fixture; 589 plus new tests green.
7. Phase 7, load test and handover. tests/load scripts run at F1 and F5; report written; the Risk IT requirements (section 3.1 DDL, the rules, the SLAs, the case list) exported as a standalone document.

What not to do in any phase: no partitioning, no Redis, no async driver, no APScheduler in the API, no MERGE statement, no ORM in merge or rollup SQL, no screen reading spend\_record except the drawer, no state stored by hand that a rule derives.

## 10. Configuration, scheduling and logging

All settings come from environment variables with prefix CHARON\_ and are read once by charon/settings.py.

| Setting | Default | Meaning |
| --- | --- | --- |
| CHARON\_DB\_URL | none, required | mssql+pyodbc URL with driver and TrustServerCertificate as Risk IT require |
| CHARON\_DB\_SCHEMA | charon | Schema for every table |
| CHARON\_DEPARTMENT\_ID | 1 | The one department in use |
| CHARON\_FEED\_TRANSPORT | landing | landing or http |
| CHARON\_FEED\_HTTP\_URL, CHARON\_FEED\_HTTP\_TOKEN | none | Only for transport http |
| CHARON\_INGEST\_PAGE\_SIZE | 10000 | Rows per page |
| CHARON\_INGEST\_OVERLAP\_SEQ | 0 | Re-read window behind the watermark |
| CHARON\_INGEST\_LEASE\_S | 600 | Lease length |
| CHARON\_FRESH\_INTERVALS\_DELAYED, \_NOT\_UPDATING | 2, 6 | Freshness thresholds in intervals |
| CHARON\_CACHE\_MAX\_ENTRIES | 500 | Per worker |
| CHARON\_ENV | prod | dev enables the bypass header and SQL echo |
| CHARON\_SSO\_COOKIE\_NAME, CHARON\_SSO\_VALIDATE\_URL | from Bow Wow | SSO integration |

Scheduling: two scheduler entries. charon-ingest --feeds spend,activity,outbox,reconcile every 5 minutes; charon-ingest --feeds directory,groups,models every 6 hours; charon-ingest --feeds archive on the first of the month. The program exits non-zero on failure so the scheduler's own alerting fires. If the scheduler cannot guarantee no overlap, the lease makes a second start harmless.

Logging: standard library logging to stdout in JSON lines (one object per line: ts, level, process, feed or route, msg, plus load\_id or request id), so Risk IT's log shipper reads it without a parser. Ingest logs one line per page with rows, inserted, updated, seconds. The API logs nothing per request beyond the request\_log table except errors.

Handover to Risk IT consists of: the repository, the Alembic migrations as SQL scripts (alembic upgrade --sql) for DBA review, the section 3.1 DDL and rules as their build requirement, the two scheduler commands, the environment variable table above, and the load test report.
