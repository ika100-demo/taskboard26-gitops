---
plan_id: 001-persistent-tasks-and-web-task-board
spec: 001-persistent-tasks-and-web-task-board
feature: Tasks kept in Postgres by the Task API, and a web task board that lists and creates them
gitops_app: ika100-demo/taskboard26-gitops
status: draft
repos:
  - id: taskboard26-gitops
    shape: gitops-app
    summary: Postgres database for the Task API and the Task API URL for the board
    acs: [AC-001.1, AC-001.4]
    order: before-merge
    gitops:
      - {resource: postgres, type: database}
      - {uses: postgres, component: taskboard26-api}
      - {env: {TASK_API_URL: 'http://taskboard26-api'}, component: taskboard26-web}
    depends_on: []
    done: false
  - id: taskboard26-api
    shape: service-python
    summary: Persistent tasks in Postgres
    acs: [AC-001.1, AC-001.2, AC-001.3, AC-001.6]
    criteria:
      - "Given tasks were created through `POST /tasks` and the database at `DATABASE_URL` keeps running, when the Task API process is restarted or replaced by a new version, then `GET /tasks` returns `200` with the same tasks, each with the same `id`, `title`, `due_date` and `created_at`, oldest first (system AC-001.1) (replaces AC-001.8)"
      - "Given an empty database at `DATABASE_URL`, when the Task API starts, then it creates its table itself and `GET /tasks` returns `200` with `[]`; given the table already holds tasks, when the Task API starts, then it keeps every stored task unchanged (system AC-001.1)"
      - "Given the database at `DATABASE_URL` is unreachable, when a client sends `GET /tasks` or a valid `POST /tasks`, then the response is `503` with `detail` `\"storage unavailable\"` within 5 seconds, no task is created, `GET /ready` answers `503` and `GET /health` keeps answering `200` (system AC-001.2)"
      - "Given the database is unreachable when the Task API starts, when it starts, then the process keeps running, `GET /health` answers `200` and `GET /ready` answers `503` (system AC-001.2)"
      - "Given the database was unreachable and becomes reachable again, when 60 seconds have passed, then without a restart `GET /ready` answers `200`, `GET /tasks` returns `200` with every task stored before the outage and a valid `POST /tasks` returns `201` (system AC-001.3)"
      - "Given 10,000 tasks are stored in the database, when a client sends a further valid `POST /tasks`, then the response is `409` with `detail` `\"task limit reached\"` and `GET /tasks` still lists 10,000 tasks, also after a restart (system AC-001.6)"
      - "Given the contract's rules for `POST /tasks`, when a client sends a title that is missing, empty or only whitespace after trimming, or longer than 200 characters after trimming, then the response is `422` with `detail` a list whose entry has `loc` ending in `title`, and no task is stored (system AC-001.6)"
    arguments: |
      Storage switches from the in-memory store to the Postgres database the gitops entry wires in (`DATABASE_URL`, `PG*`
      from the CloudNativePG Secret). The existing endpoints, fields, validation, 422/409 answers and the 10,000-task limit
      of API spec 001 stay as they are (system A1); API spec 001's A10 no longer holds (system A2). No carry-over of
      in-memory tasks (system A3). Keep `created_at` at microsecond precision and order by it with an insertion sequence
      as tie-breaker. Liveness never touches the database; readiness pings it with the timeouts of the contract.
    depends_on: []
    done: false
  - id: taskboard26-web
    shape: web-nextjs
    summary: Task board on the start page that lists and creates tasks
    acs: [AC-001.4, AC-001.5, AC-001.6, AC-001.7]
    criteria:
      - "Given the Task API at `TASK_API_URL` returns tasks A, B and C from `GET /tasks`, when a person opens `/`, then the page shows exactly these three tasks in that order (oldest first), each with its title and its `due_date` as `YYYY-MM-DD` or the text \"No due date\"; given it returns `[]`, the page shows \"No tasks yet\" (system AC-001.4)"
      - "Given the board is open, when a person enters the title \"Buy milk\" and optionally the due date 2026-12-24 and clicks \"Add task\", then the board sends `POST /tasks` to the Task API and on `201` shows the returned task at the end of the list without a full page reload, and the \"Title\" and \"Due date\" fields are emptied (system AC-001.5)"
      - "Given the board is open, when a person submits a title that is empty or only whitespace, then the board shows \"Enter a title\" next to the form, keeps the entered values and sends no request to the Task API; when the title is longer than 200 characters after trimming, it shows \"Title must be at most 200 characters\" in the same way (system AC-001.6)"
      - "Given the board is open, when a person submits a valid task and the Task API answers `409`, then the board shows \"Task limit reached\" next to the form, keeps the entered values and lists no new task (system AC-001.6)"
      - "Given the Task API is unreachable, does not answer within 5 seconds or answers `5xx` (including `503` \"storage unavailable\") or a status the contract does not list, when a person opens `/`, then the page loads with `200` and shows \"Tasks could not be loaded. Try again later.\" in place of the list (system AC-001.7)"
      - "Given the Task API is unreachable, does not answer within 5 seconds, answers `5xx`, a `422` or a status the contract does not list, when a person submits the form, then the board shows \"Task could not be saved. Try again.\", keeps the entered values and lists no new task (system AC-001.7)"
      - "Given the Task API is unreachable, when the platform probes the board, then `/api/health` and `/api/ready` keep answering `200`, so the board stays served (system AC-001.7)"
    arguments: |
      The start page `/` is the board (system A6): the list, then a form with "Title" (required), "Due date" (optional,
      a date input sending `YYYY-MM-DD`) and an "Add task" button. The Task API is called only from the Next.js server
      (Server Component for the list, a Server Action or route handler for the create) at `TASK_API_URL`, which is an
      in-cluster URL the browser cannot reach; never expose it as `NEXT_PUBLIC_*`. Add `TASK_API_URL` to
      `docs/env-vars.md` (local default `http://localhost:8080`). Dev only (system A10); no sign-in (system A9).
    depends_on: []
    done: false
gitops_pin:
  - service: taskboard26-api
    overlay: staging
    apply_after: taskboard26-api
    note: persistence holds wherever the Task API runs (system A4, A10); staging gets the database from the gitops entry and the persistent API through the usual after-dev-smoke promotion once the API's PR merged and dev smoke is green. The board stays in dev, so taskboard26-web gets no staging pin.
---

## taskboard26-gitops — Postgres database for the Task API and the Task API URL for the board

**Shape:** `gitops-app` · **Criteria:** AC-001.1, AC-001.4 · **Depends on:** []

AC-001.1 needs durable storage that outlives the Task API's pod: one Postgres database `postgres` (type `database`, CloudNativePG) is declared for the system and used only by `taskboard26-api` (system A4). Because the API runs in dev and staging, `uses: postgres` gives it `DATABASE_URL` and `PG*` in both. AC-001.4 needs the board to reach the Task API *of the same environment*: `TASK_API_URL: http://taskboard26-api` is the API's Service in the environment's namespace (port 80). The board has no instance outside dev, so the variable only takes effect there. This entry is applied before the component PRs merge: the in-memory API image ignores `DATABASE_URL`, and the template board ignores `TASK_API_URL`, so nothing breaks in between.

## taskboard26-api — Persistent tasks in Postgres

**Shape:** `service-python` · **Criteria:** AC-001.1, AC-001.2, AC-001.3, AC-001.6 · **Depends on:** []

The API owns the storage behaviour: tasks survive restarts and redeploys (AC-001.1), outages of the database answer `503` with readiness down and liveness up (AC-001.2), and it recovers by itself within 60 seconds (AC-001.3). It keeps its whole existing contract (system A1), so the board can be built in parallel against the contract below. It takes AC-001.6 for the server side: the `409` at 10,000 stored tasks (now counted in the database) and the `422` for the title rules the board also checks. Its spec's AC-001.8 ("after a restart `GET /tasks` returns `[]`") is superseded by the first criterion (system A2). Schema creation happens in the API on start, idempotent, never dropping data; a start with the database down must not crash the process.

## taskboard26-web — Task board on the start page that lists and creates tasks

**Shape:** `web-nextjs` · **Criteria:** AC-001.4, AC-001.5, AC-001.6, AC-001.7 · **Depends on:** []

The board is what a person sees: the list oldest first with "No due date" / "No tasks yet" (AC-001.4), the create form that appends without a full reload (AC-001.5), its own title checks plus the mapping of the API's `409` (AC-001.6), and the error states when the API is down, slow or failing (AC-001.7). It needs no dependency on the API repo: the contract below is the existing API contract plus the `503`, and the board's tests stub the Task API. Its readiness must not depend on the Task API, otherwise an API outage would take the page away instead of showing the AC-001.7 message.

Decomposition: one repo per concern the spec names (storage in the API, presentation in the web, infrastructure as data in the gitops entry). No shared library is needed, the board and the API share only the HTTP contract. No repo has to be created.

## Contract

Provider: `taskboard26-api`. Consumer: `taskboard26-web`, server-side only, at the base URL `TASK_API_URL` (in the cluster `http://taskboard26-api`). All bodies are JSON (`Content-Type: application/json`). No authentication (system A9). Everything except the `503` answers is the API's existing contract from API spec 001, unchanged (system A1).

**Task object**

| Field | Type | Notes |
|---|---|---|
| `id` | string, UUID | server-generated, never reused |
| `title` | string | trimmed, 1 to 200 characters |
| `due_date` | string `YYYY-MM-DD` or `null` | calendar date, no time; past dates allowed |
| `created_at` | string, UTC ISO 8601 with microseconds and `Z`, e.g. `2026-11-22T10:15:30.123456Z` | server-set, unchanged by restarts |

**`GET /tasks`** — list all tasks.

| Status | Body | Meaning |
|---|---|---|
| `200` | JSON array of task objects, oldest first (by `created_at`), no pagination; `[]` when none | stored tasks, the same across restarts and redeploys |
| `503` | `{"detail": "storage unavailable"}` | the database is unreachable |

**`POST /tasks`** — create a task.

Request: `{"title": "<string>", "due_date": "<YYYY-MM-DD>" | null}`; `due_date` may be omitted (same as `null`); unknown fields are ignored. The board sends the title as entered (the API trims it) and omits `due_date` when the field is empty.

| Status | Body | Meaning |
|---|---|---|
| `201` | the created task object | stored durably before the answer |
| `422` | `{"detail": [{"loc": ["body", "<field>"], "msg": "<text>", "type": "<rule>"}, ...]}` | invalid `title` or `due_date`; nothing stored |
| `409` | `{"detail": "task limit reached"}` | 10,000 tasks are stored; nothing stored |
| `503` | `{"detail": "storage unavailable"}` | the database is unreachable; nothing stored |

**Probes of the API:** `GET /health` answers `200` `{"status": "ok"}` whenever the process runs, independent of the database. `GET /ready` answers `200` `{"status": "ready"}` when a database ping succeeds, else `503`. The board's probes `/api/health` and `/api/ready` never call the Task API.

**Events:** none.

### Rules

```yaml
rules:
  - id: title-required
    endpoint: POST /tasks
    field: title
    kind: required
    value: true
    provider: taskboard26-api
    consumers: [taskboard26-web]
  - id: title-min
    endpoint: POST /tasks
    field: title
    kind: min_length
    value: 1
    provider: taskboard26-api
    consumers: [taskboard26-web]
  - id: title-max
    endpoint: POST /tasks
    field: title
    kind: max_length
    value: 200
    provider: taskboard26-api
    consumers: [taskboard26-web]
```

Rules no tool checks:

- Both length rules apply to the title **after trimming** leading and trailing whitespace: a title of only whitespace counts as empty. The board checks them before sending and shows "Enter a title" for `title-required` / `title-min` and "Title must be at most 200 characters" for `title-max`; the API enforces them again with `422`.
- `due_date` must be a calendar date in `YYYY-MM-DD` form or `null`/absent; anything else is `422` naming `due_date`. The board uses a date input, so it only ever sends a valid date or omits the field.
- At most 10,000 tasks are stored; a further valid create is `409` `"task limit reached"`. The board does not count tasks itself, it maps the `409`.
- The board shows its own fixed English messages, except that a `422` whose `detail` names `title` shows that entry's `msg` next to "Title" (system A8, revised 2026-11-22).

### Errors

```yaml
errors:
  status: 422
  body:
    list: detail
    field: loc
    rule: type
    message: msg
  unlisted:
    message: "Task could not be saved. Try again."
```

A validation error is `422` with `detail` a list; each entry's `loc` is a list whose last element is the field name (`title` or `due_date`), `type` the rule that failed and `msg` a text the board shows at the field for `title` (system A8, revised). Other answers carry `detail` as one string: `409` `"task limit reached"`, `503` `"storage unavailable"`.

How the board treats answers:

| Call | Answer | Board shows |
|---|---|---|
| `GET /tasks` | `200` | the list, or "No tasks yet" for `[]` |
| `GET /tasks` | `503`, any other `5xx`, any status not listed above, a body that is not a task array, connection error, timeout | "Tasks could not be loaded. Try again later." in place of the list; the page itself answers `200` |
| `POST /tasks` | `201` | the returned task appended at the end of the list, form emptied |
| `POST /tasks` | `409` | "Task limit reached" next to the form, values kept |
| `POST /tasks` | `422` naming `title` (its own checks passed, so the API disagrees) | that entry's `msg` next to "Title", values kept, no task added to the list |
| `POST /tasks` | `422` naming only other fields, `503`, any other `5xx`, any status not listed above, connection error, timeout | "Task could not be saved. Try again." next to the form, values kept, no task added to the list |

### Timeouts

- **Board → Task API, `GET /tasks`:** 5 seconds for the whole request (connect and response). On timeout or connection error the board renders the page with "Tasks could not be loaded. Try again later."; no retry while rendering, a reload tries again.
- **Board → Task API, `POST /tasks`:** 5 seconds. On timeout or connection error the board shows "Task could not be saved. Try again." and keeps the values; it never retries by itself, because a create that timed out may have been stored (a reload shows it).
- **Task API → Postgres:** connect timeout 2 seconds, statement timeout 3 seconds, so a request answers `503` `"storage unavailable"` within 5 seconds of the database becoming unreachable or slow and the board's 5 seconds are not exceeded. A broken pooled connection is discarded and a new one is opened on the next request; no request waits for the database to come back.
- **Task API readiness:** `GET /ready` pings the database with a 2-second timeout and answers `503` when it fails. Kubernetes probes readiness every 5 seconds, so after the database returns the API is ready again within seconds, well inside the 60 seconds of AC-001.3. Liveness never touches the database, so an outage never restarts the API (system A5).
- **When the API is down or unready in the cluster:** an unready API pod is removed from its Service, so the board may see a connection error instead of the API's `503`; it treats both alike (the rows above). A client going through the Gateway may likewise get the Gateway's own `503` instead of the API's body during an outage; AC-001.2's body is checked by the API's own tests against the process.

## gitops-app PR

The gitops entry lands first, in the system ship's `compose/001-persistent-tasks-and-web-task-board` PR, before any component merges: it declares `postgres` (type `database`), adds `uses: [postgres]` to `taskboard26-api` (dev and staging) and `TASK_API_URL: http://taskboard26-api` to `taskboard26-web`, then renders. On the first database of the system the CloudNativePG operator is rendered into `operators/` and must be applied once per cluster (`/system:ship` does it on the local cluster; on any other cluster a person runs `KUBE_CONTEXT=<ctx> devbox run bootstrap`).

Dev pins of both components land in the same compose PR (spec 144). Staging: the persistent API reaches staging through the after-dev-smoke promotion once its PR merged and dev smoke is green; the staging database comes from the gitops entry. The board stays in dev.

Verify after Argo reconciles:

- dev and staging: the CloudNativePG `Cluster` for `postgres` is healthy and the `taskboard26-api` pod has `DATABASE_URL`; `GET /ready` of the API answers `200`.
- dev: create a task through the board at `http://taskboard26-web.taskboard26-dev.localhost:8088/`, delete the API pod (`/instance:verify dev` or a rollout restart through a new pin), reload: the task is still listed (AC-001.1, AC-001.4).
- dev: the board shows "No tasks yet" or the list, never the template page, and the dev smoke of both components passes.
