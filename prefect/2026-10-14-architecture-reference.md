# Prefect Architecture Reference (2026-10-14)

Diagrams + file references for the systems architecture. Each section is self-contained; use the reference index at the bottom to retrieve details without re-reading the whole doc.

---

## 1. High-Level System Diagram

```mermaid
flowchart TB
    subgraph USERS["User Surfaces"]
        SDK["Python SDK<br/>@flow / @task<br/>src/prefect/flows.py, tasks.py"]
        CLI["CLI (cyclopts)<br/>src/prefect/cli/"]
        UI["React UI (ui-v2)<br/>REST only, openapi-fetch"]
    end

    subgraph EXEC["Execution Plane (client-side)"]
        FE["Flow Engine<br/>flow_engine.py"]
        TE["Task Engine<br/>task_engine.py"]
        W["Workers (work pools)<br/>workers/base.py"]
        R["Runner (prefect serve)<br/>runner/runner.py"]
        TW["TaskWorker (deferred tasks)<br/>task_worker.py"]
        B["Bundles<br/>cloudpickle flow → subprocess<br/>bundles/"]
    end

    subgraph SERVER["Orchestration Plane — src/prefect/server/"]
        API["FastAPI app<br/>api/server.py::create_app"]
        ORCH["Orchestration rules<br/>orchestration/rules.py, core_policy.py"]
        SVC["Docket perpetual services<br/>scheduler, late-runs, foreman,<br/>cancellation, vacuums"]
        EVT["Event pipeline<br/>events/ → triggers → actions"]
        TQ["TaskQueue<br/>server/task_queue.py"]
    end

    subgraph DATA["Data Plane"]
        DB[("SQLAlchemy ORM<br/>server/database/orm_models.py<br/>PG + SQLite (lockstep)")]
        BLK[("Blocks: server-persisted config<br/>blocks/core.py")]
        RES[("Result storage blocks<br/>results.py ResultStore")]
    end

    SDK --> FE
    SDK --> TE
    CLI --> API
    UI -->|REST| API
    API --> ORCH --> DB
    FE -->|"REST: propose states"| API
    TE -->|"task_run.* events (WS)"| EVT
    W -->|"WS worker channel + REST"| API
    API -->|"scheduled flow runs"| W
    W --> B --> FE
    R -->|"poll + subprocess"| API
    TW -->|subscribe| TQ
    TQ -->|"scheduled task runs"| TW
    SVC --> DB
    EVT --> DB
    BLK --> DB
```

**The core architectural split:** flow-run states are *proposed to the server* (rules ACCEPT/REJECT/WAIT/ABORT); task-run states are *advanced locally* by the task engine and only *reported* via `prefect.task_run.*` events, which `TaskRunRecorder` upserts into the DB. See `src/prefect/AGENTS.md:28-29`.

---

## 2. Flow Execution — Lifecycle Sequence

```mermaid
sequenceDiagram
    participant U as User code
    participant F as @flow (flows.py)
    participant FE as FlowRunEngine (flow_engine.py)
    participant C as PrefectClient (client/)
    participant S as Server API (server/api/)
    participant OR as Orchestration rules (server/orchestration/)

    U->>F: my_flow(args)
    F->>FE: run_flow (flow_engine.py:2202)
    Note over FE: dispatch: sync/async × plain/generator<br/>(flow_engine.py:2070-2199)
    FE->>C: create_flow_run
    C->>S: POST /flow_runs
    FE->>S: propose Running → POST /flow_runs/{id}/set_state
    S->>OR: run rules (CoreFlowPolicy)
    alt ACCEPT
        OR-->>FE: validated state
    else REJECT
        OR-->>FE: server's state (may raise Pause)
    else WAIT
        OR-->>FE: retry after delay_seconds
    else ABORT
        OR-->>FE: raise Abort
    end
    Note over FE: enter FlowRunContext (context.py:528)<br/>run user fn → tasks/subflows
    FE->>S: propose terminal state (Completed/Failed/Crashed)
    S->>OR: rules validate + commit → state event emitted
```

- State proposal logic: `src/prefect/utilities/engine/__init__.py:514` (sync), `:612` (async).
- Endpoint: `src/prefect/server/api/flow_runs.py:820`.
- Rules: `BaseOrchestrationRule` = async context manager per rule, nested via AsyncExitStack around the DB commit — `src/prefect/server/orchestration/rules.py:534`.
- Policies: `CoreFlowPolicy` / `CoreTaskPolicy` / `MinimalFlowPolicy` (`force=True` still guarded) — `server/orchestration/core_policy.py`.
- Subprocess entrypoint: `python -m prefect.flow_engine <entrypoint>` reads `PREFECT__FLOW_RUN_ID` (`flow_engine.py:281`).
- Subflows: synthetic tracking task + `parent_task_run_id` (`flow_engine.py:981`, `:922`).

---

## 3. Task Execution — Local State Advancement

```mermaid
flowchart TB
    subgraph CALL["@task call/submit (tasks.py)"]
        T["Task.__call__ (tasks.py:1198)"]
        S2["Task.submit (tasks.py:1277)"]
        D["Task.delay → deferred Scheduled run<br/>tasks.py:1777"]
    end

    subgraph TRUNNERS["Task Runners (task_runners.py)"]
        TP["ThreadPoolTaskRunner (:242)<br/>default, in-proc threads"]
        PP["ProcessPoolTaskRunner (:749)<br/>pickle → subprocess"]
        PT["PrefectTaskRunner (:1337)<br/>submit to SERVER (deferred)"]
    end

    subgraph ENG["Task Engine (task_engine.py)"]
        SE["Sync/AsyncTaskRunEngine (:432/:1044)"]
        SS["set_state (:556)<br/>advances state LOCALLY<br/>+ emits prefect.task_run.* event"]
        BR["begin_run (:516)<br/>poll w/ clamped backoff"]
    end

    subgraph FUT["Futures (futures.py)"]
        PC["PrefectConcurrentFuture (:201)"]
        PD["PrefectDistributedFuture (:283)<br/>waits via TaskRunWaiter"]
    end

    subgraph SERVERSIDE["Server"]
        EVIN["WS /events/in"]
        REC["TaskRunRecorder service<br/>upserts TaskRun rows<br/>services/task_run_recorder.py"]
        TQ["TaskQueue → WS /task_runs/subscriptions/scheduled"]
        TWK["TaskWorker (task_worker.py)<br/>executes deferred runs"]
    end

    T --> SE
    S2 -->|FlowRunContext required| TP & PP
    D --> PT
    TP --> PC
    PT --> PD
    PP -->|"pickle State (not Future)"| SE
    SE --> SS --> BR
    SS -->|"prefect.task_run.* event"| EVIN --> REC
    PD -.->|observes events| EVIN
    PT -->|Scheduled run| TQ --> TWK
```

Key contrasts vs flow engine (from `src/prefect/AGENTS.md`):
- `Task.create_run` (tasks.py:876) makes the **only** server call (`client.create_task_run`); all later transitions are local.
- `ProcessPoolTaskRunner` requires picklable data — futures must be resolved to `State` in the parent (`resolve_to_final_result`).
- `PrefectTaskRunner`/`Task.delay()` = server-side execution; the `TaskWorker` reads deferred parameters from result storage (`task_worker.py:633`).

---

## 4. Remote Execution — Workers, Runner, Bundles

```mermaid
flowchart TB
    subgraph SCHED["Scheduling (server)"]
        DEP["Deployment (models/deployments.py)"]
        SCHEDS["Scheduler service<br/>services/scheduler.py<br/>(Docket perpetual)"]
        WQ["WorkQueue / WorkPool<br/>models/workers.py"]
        FRS["Scheduled FlowRun"]
    end

    subgraph WDISPATCH["BaseWorker (workers/base.py)"]
        POLL["_get_scheduled_flow_runs (:1477)<br/>REST + WS worker channel"]
        PROP["_propose_pending/submitting (:1741/:1790)<br/>Scheduled→Pending→Submitting"]
        SUB["_submit_run (:1566) → subclass run()"]
    end

    subgraph LAUNCH["Launch paths"]
        PW["ProcessWorker (workers/process.py:134)<br/>a) FlowRunExecutorContext direct<br/>b) WorkspaceResolvingEngineCommandStarter<br/>   → pull steps → python -m prefect.engine"]
        BN["Bundle (bundles/)<br/>create_bundle_for_flow_run<br/>cloudpickle+gzip standalone file<br/>→ python -m prefect.bundles.execute"]
        OTHER["Cloud/Container workers<br/>(worker subclasses)"]
    end

    subgraph RUNNER["Runner (runner/runner.py:182) — no work pool"]
        SP["_state_proposer, _scheduled_run_poller<br/>_process_manager (SIGTERM→SIGKILL)<br/>TCP control channel for cancel"]
        WEB["tiny webserver /health, /run_count, /shutdown<br/>runner/server.py"]
    end

    INF["flow run subprocess<br/>python -m prefect.flow_engine<br/>reads PREFECT__FLOW_RUN_ID"]

    DEP --> SCHEDS --> FRS --> WQ --> POLL --> PROP --> SUB
    SUB --> PW & BN & OTHER
    PW & BN --> INF
    RUNNER --> SP --> INF
    RUNNER --> WEB
```

- Worker↔server WebSocket: `/work_pools/{name}/workers/connect` (`server/api/workers.py:1088`), protocol schema `client/schemas/worker_channel.py`.
- Server→worker teardown messaging: pluggable `server/worker_communication/cleanup_queue/` (in-memory default; `PREFECT_SERVER_WORKER_CHANNEL_CLEANUP_QUEUE_STORAGE` for Redis/HA).
- `infrastructure/` is now only **provisioners** (ECS, Cloud Run, etc.); old Agent/Infrastructure classes were removed in Prefect 3 (`infrastructure/agent.py` = 9-line shim).

---

## 5. Server Internals — Request & Service Flow

```mermaid
flowchart TB
    subgraph APP["create_app (server/api/server.py:859)"]
        LS["Lifespan (:920):<br/>alembic migrations → register block types<br/>→ Docket + background worker → services"]
        MID["Middleware: CORS, GZip, CSRF,<br/>SQLite request limit, Prometheus"]
        ROUTERS["~25 routers (API_ROUTERS :102)<br/>flows, flow_runs, task_runs, deployments,<br/>work_pools, events, concurrency_limits_v2,<br/>blocks, artifacts, variables, automations, ui/*"]
        UIAPP["create_ui_app: V1 at / , V2 at /v2<br/>cookie-based redirect middleware"]
    end

    subgraph ORCHL["Orchestration (orchestration/)"]
        CTX["OrchestrationContext (:71)<br/>initial/proposed/validated state<br/>ACCEPT|REJECT|ABORT|WAIT"]
        RULES["BaseOrchestrationRule (:534)<br/>before/after_transition + cleanup<br/>nested via AsyncExitStack"]
        CORE["core_policy.py: concurrency slots,<br/>caching, retries, pausing, cancellation,<br/>PreventResultDataLoss"]
        UT["global_policy.py transforms:<br/>timestamps, run counters, subflow parent updates"]
    end

    subgraph SVCS["Services — Docket 'find-and-flood'"]
        BW["In-process Docket worker<br/>api/background_workers.py<br/>runs per-entity tasks, e.g.<br/>mark-flow-run-late:{id}"]
        PS["perpetual_services.py registry<br/>scheduler · late_runs · foreman<br/>pause_expirations · repossessor<br/>cancellation_cleanup · db_vacuum"]
    end

    subgraph EVT["Events (server/events/)"]
        IN["WS /events/in → message bus<br/>(in-memory | PG LISTEN/NOTIFY)"]
        PIPE["EventsPipeline"]
        PERS["EventPersister → DB"]
        TRG["ReactiveTriggers + proactive<br/>triggers.py"]
        ACT["Actions: run deployment,<br/>pause/resume, cancel, notify"]
    end

    MODELS["models/ — SQLAlchemy CRUD layer<br/>(also emits prefect.* lifecycle events<br/>via events/schemas/lifecycle.py)"]
    ORM["database/orm_models.py<br/>Alembic: _migrations/versions/{postgresql,sqlite}/"]
    DBCFG["database/interface.py PrefectDBInterface<br/>configurations.py: PG / SQLite configs"]

    LS --> MID --> ROUTERS
    ROUTERS --> CTX --> RULES --> CORE & UT
    ROUTERS --> MODELS --> ORM
    PS --> BW --> MODELS
    IN --> PIPE --> PERS & TRG --> ACT
    RULES -->|"EnqueueScheduledTasks"| TQ["task_queue.py TaskQueue"]
    ORM --- DBCFG
```

Adding an endpoint = the layering order **schema → model → route** (`server/AGENTS.md:15-23`); `variables` is the canonical example.

---

## 6. Client Architecture & Data Interfaces

```mermaid
flowchart LR
    subgraph SURF["Surfaces"]
        SDK["SDK engines"]
        CLI["CLI"]
        UIV["ui-v2 (openapi-fetch<br/>+ generated prefect.ts)"]
    end

    subgraph CLIENT["src/prefect/client/"]
        GC["get_client (:176)<br/>reuses context clients,<br/>may boot ephemeral server"]
        PC["PrefectClient (:281)<br/>SyncPrefectClient (:1228)"]
        SUBS["mixin domain subclients:<br/>_flow_runs, _deployments, _work_pools,<br/>_events, _automations, ..."]
        WS["subscriptions.py WebSocket client<br/>events/clients.py PrefectEventSubscriber (:693)<br/>logging/clients.py PrefectLogsSubscriber"]
        CS["client/schemas/ (kept separate<br/>from server/schemas/)"]
        CC["cloud.py CloudClient (:74)"]
    end

    subgraph PROTO["Protocols"]
        REST["REST: set_state, CRUD,<br/>get_scheduled_flow_runs_for_work_pool"]
        EVWS["WS /events/in + /events/out"]
        LOGWS["WS logs /out"]
        WORK["WS work_pool_worker_channel.v1"]
    end

    SERVER["Prefect server / Prefect Cloud"]
    EMIT["emit_event (events/utilities.py:30)<br/>→ EventsWorker queue (events/worker.py:70)"]
    EPHEM["ephemeral SubprocessASGIServer<br/>(no PREFECT_API_URL)"]

    SDK --> GC --> PC --> SUBS
    CLI --> PC
    UIV -->|REST| SERVER
    SDK --> EMIT
    SUBS --> REST & EVWS & WORK
    WS --> EVWS & LOGWS
    PC --> CS
    PC --> CC --> SERVER
    GC -.-> EPHEM -.-> SERVER
```

- **`prefect-client` package build** (repo-root `client/`): `build_client.sh` copies `src/prefect/`, strips `cli/`, `server/` (except `server/api/`), etc. Dep changes must mirror `pyproject.toml` ↔ `client/pyproject.toml`.

---

## 7. Cross-Cutting Concerns

```mermaid
flowchart TB
    subgraph CTX["Context DI (context.py)"]
        FRC["FlowRunContext (:528)<br/>flow, flow_run, task_runner,<br/>result_store, run_results"]
        TKC["TaskRunContext (:531)<br/>task, task_run, result_store"]
        SER["serialize_context (:115) → travels<br/>into subprocesses/bundles"]
    end

    subgraph RESULTS["Results (results.py)"]
        RS["ResultStore (:437)<br/>write → mark_persisted()"]
        SERZ["serializers.py dispatch:<br/>pickle/json/orjson/compression"]
        STOR["Storage blocks (WritableFileSystem)<br/>blocks/ + integrations (S3, GCS, ...)"]
        TIERS["default storage = 3 tiers:<br/>setting → server block (API) → local path"]
    end

    subgraph CONC["Concurrency"]
        V2["v2 lease-based (concurrency/)<br/>acquire → lease → renew thread<br/>→ release; events"]
        V1["v1 legacy (concurrency/v1/)"]
        SERVER["server models:<br/>concurrency_limits (v1)<br/>concurrency_limits_v2 + decay"]
    end

    subgraph CONF["Settings (settings/)"]
        SET["Settings(PrefectBaseSettings)<br/>models/root.py:94 nested PREFECT_*"]
        PROF["profiles.toml + sources.py<br/>env filter, dotenv, profiles"]
    end

    subgraph BLK["Blocks"]
        COREB["Block (blocks/core.py:316)<br/>register_type_and_schema (:1580)"]
        INTEG["integrations expose blocks via<br/>prefect.collections entry-points"]
    end

    CTX --> RS
    RS --> SERZ --> STOR
    RS --> TIERS
    V2 --> SERVER
    COREB --> SERVER
    INTEG --> COREB
    SET --> PROF
```

---

## 8. Reference Index

| Topic | Where |
|---|---|
| Server invariant contracts | `src/prefect/server/AGENTS.md` |
| SDK invariant contracts (engines, results, suspension) | `src/prefect/AGENTS.md` |
| Worker/runner/deployment contracts | `src/prefect/workers/AGENTS.md`, `runner/AGENTS.md`, `deployments/AGENTS.md` |
| State proposal (sync/async) | `src/prefect/utilities/engine/__init__.py:514,612` |
| set_state endpoint | `src/prefect/server/api/flow_runs.py:820` |
| Orchestration context/rules/transforms | `server/orchestration/rules.py:71,534,969` |
| Policy rule set | `server/orchestration/core_policy.py` |
| Worker polling → dispatch → submit | `workers/base.py:1477,1506,1566,1741,1790` |
| Perpetual services registry | `server/services/perpetual_services.py`, `server/api/background_workers.py` |
| TaskRunRecorder (event→DB upsert) | `server/services/task_run_recorder.py` |
| Event ingestion websockets | `server/api/events.py` |
| Worker WebSocket channel | `server/api/workers.py:1088`, `client/schemas/worker_channel.py` |
| Client composition | `client/orchestration/__init__.py:281,1228`, subclients in `client/orchestration/*/client.py` |
| Event emission / subscriber | `events/utilities.py:30`, `events/clients.py:275,693`, `events/worker.py:70` |
| ResultStore / default tiers | `results.py:437,163` |
| Contexts | `context.py:528,531,115` |
| Settings model / profiles | `settings/models/root.py:94`, `settings/profiles.py:182` |
| Blocks registration | `blocks/core.py:316,1580` |
| Concurrency v2 leases | `concurrency/_leases.py`, `concurrency/services.py` |
| DB interface / migrations | `server/database/interface.py`, `server/database/_migrations/versions/` |
| UI API layer | `ui-v2/src/api/service.ts`, generated `ui-v2/src/api/prefect.ts` |
| prefect-client build | `client/build_client.sh`, `client/pyproject.toml` |
| Deployments (prefect.yaml, steps) | `deployments/base.py`, `deployments/steps/{core,pull,utility}.py`, `deployments/runner.py:167` |
| Bundles | `bundles/__init__.py`, `bundles/execute.py` |
| Tests map | `tests/` mirrors src; e2e `integration-tests/`; cloud compat `compat-tests/` (submodule); UI `ui-v2/tests/`, `ui-v2/e2e/` |

## 9. Rules of Thumb (from AGENTS.md contracts)

1. **Flow states → server; task states → local + events.** Never bypass server orchestration for flow runs.
2. **Sync/async engine paths must stay in lockstep** — mirror every behavior change.
3. **SQLite and PostgreSQL migrations must stay in lockstep** — CI tests both.
4. **`force=True` routes through `MinimalFlowPolicy`** — not a true bypass (still `PreventResultDataLoss`).
5. **Perpetual services use `Perpetual(automatic=True)`** and Docket keys `"<service>:<entity-id>"` to prevent duplicate enqueues.
6. **Every result write must call `mark_persisted()`** or state metadata is silently dropped.
7. **Server vs client schemas stay separate** (`server/schemas/` vs `client/schemas/`).
8. **Auth token comparisons use `hmac.compare_digest`** (CSRF, basic auth, WS auth).
