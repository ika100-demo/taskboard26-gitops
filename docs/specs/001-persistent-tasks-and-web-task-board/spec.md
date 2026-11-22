---
spec_id: 001-persistent-tasks-and-web-task-board
title: Persistent tasks and web task board
status: draft
priority: P1
shape: gitops-app
---

# 001 — Persistent tasks and web task board

## Problem

The Task API keeps tasks only in the memory of its running process. Every restart or new deploy empties the list, so in dev and staging people lose what they wrote down each time a new version goes out. Also, a person can reach tasks only by calling the API directly: the Task board web UI is deployed but shows nothing yet. Keeping tasks in a Postgres database and giving the web UI a board that lists and creates tasks makes the taskboard usable by people, not only by API clients.

## Stories

As a taskboard user, I want the tasks I created to still be there after the Task API restarts or is redeployed, so that I can rely on the list.
**Repos:** taskboard26-api, taskboard26-gitops

As a taskboard user, I want to see all tasks in the web UI, so that I can check what I have to do without calling the API.
**Repos:** taskboard26-web, taskboard26-gitops

As a taskboard user, I want to create a task with a title and an optional due date in the web UI, so that I can write down a task in my browser.
**Repos:** taskboard26-web, taskboard26-api

## Acceptance criteria

- **AC-001.1** Given tasks were created through `POST /tasks` of the Task API, when the Task API restarts or a new version of it is deployed, then `GET /tasks` returns `200` with the same tasks, each with the same `id`, `title`, `due_date` and `created_at`, oldest first.
- **AC-001.2** Given the database of the environment is unreachable, when a client sends `GET /tasks` or a valid `POST /tasks` to the Task API, then the response is `503` with `detail` `"storage unavailable"`, no task is created, the API's readiness check answers `503` and its liveness check keeps answering `200`.
- **AC-001.3** Given the database was unreachable and becomes reachable again, when 60 seconds have passed, then without a restart of the Task API `GET /tasks` returns `200` with every task stored before the outage and a valid `POST /tasks` returns `201`.
- **AC-001.4** Given tasks A, B and C exist in the Task API of the same environment, when a person opens the start page of the Task board web UI, then the page shows exactly these three tasks, oldest first, each with its title and its due date as `YYYY-MM-DD` or the text "No due date"; given no task exists, the page shows "No tasks yet".
- **AC-001.5** Given the board is open, when a person enters the title "Buy milk" and, optionally, the due date 2026-12-24 and submits the form, then without a full page reload the task appears at the end of the list with that title and due date, the form is emptied, and `GET /tasks` of the Task API lists the task.
- **AC-001.6** Given the board is open, when a person submits a title that is empty or only whitespace, or longer than 200 characters after trimming, or the Task API answers `409` (10,000 tasks reached), then the board shows "Enter a title", "Title must be at most 200 characters" or "Task limit reached" respectively next to the form, keeps the entered values, and no new task is listed by the board or by `GET /tasks`.
- **AC-001.7** Given the Task API is unreachable or answers `5xx`, when a person opens the board, then the page loads and shows "Tasks could not be loaded. Try again later." in place of the list; when a person submits the form in that state, the board shows "Task could not be saved. Try again." and keeps the entered values.

## Non-goals

- Editing, completing or deleting tasks, in the API or on the board.
- Filtering, search, sorting options or pagination, in the API or on the board.
- Users, accounts, sign-in or per-user boards; authentication and authorization of the API or the board.
- Live updates of the board when another person creates a task (a reload shows them).
- Database backups, restore and point-in-time recovery.
- Running the board in staging or prod; it stays in dev, where its component runs today.
- Carrying over tasks that exist in the API's memory at the moment the database is switched on.

## Assumptions

- **A1** The Task API keeps its existing contract: endpoints, fields, validation, `422` and `409` answers and the 10,000-task limit of the API spec 001 hold unchanged, only now on durable storage (other option: drop the 10,000-task limit now that memory no longer bounds it; affects AC-001.1, AC-001.6).
- **A2** The API spec's AC-001.8 ("after a restart `GET /tasks` returns `[]`") and its A10 stop holding: the API component spec derived from this system spec supersedes them with `(replaces AC-001.8)` (other option: none, persistence contradicts it; affects AC-001.1).
- **A3** Tasks held in the API's memory when the database is switched on are not carried over; the first deploy with the database starts with an empty list, as every deploy does today (other option: export and import the in-memory tasks once; affects AC-001.1).
- **A4** One Postgres database is declared for the system and used only by the Task API, in every environment the API runs in (dev and staging); the board never reads the database directly, it goes through the Task API (other option: a database in dev only; affects AC-001.1, AC-001.4).
- **A5** When the database is unreachable the API answers `503` `"storage unavailable"` and reports not ready, but stays alive so it is not restarted in a loop; it recovers by itself within 60 seconds (other option: fail liveness and let it be restarted; affects AC-001.2, AC-001.3).
- **A6** The board is the start page `/` of the web UI: a list above or beside a form with the fields "Title" (required) and "Due date" (optional date), and an "Add task" button (other option: a separate page for creating; affects AC-001.4, AC-001.5).
- **A7** The board lists tasks oldest first like `GET /tasks`, and shows the due date in the API's `YYYY-MM-DD` form (other option: newest first, or a localized date; affects AC-001.4, AC-001.5).
- ~~**A8** The board checks the title before sending (trimmed, 1 to 200 characters) and shows the API's rejections with its own fixed English messages (other option: show the API's `detail` text verbatim; affects AC-001.6, AC-001.7).~~
- **A8** (revised 2026-11-22, user decision on the web escalation) The board checks the title before sending (trimmed, 1 to 200 characters) and shows "Enter a title" / "Title must be at most 200 characters" for what its own check catches; a `422` from the Task API that names the `title` field shows the API's `msg` next to "Title", values kept and no task added; every other rejection keeps the board's fixed English messages (other option: fixed messages only, which the platform contract check does not accept; affects AC-001.6, AC-001.7).
- **A9** As decided for the API spec 001 (no authentication for now), the board needs no sign-in: anyone who can reach the web UI in dev may see and create tasks (other option: protect the board before it ships; affects AC-001.4, AC-001.5).
- **A10** The board stays in dev only and the persistence applies wherever the Task API runs (dev and staging); no `environments:` limit is set for this spec (other option: promote the board to staging too; affects AC-001.4–AC-001.7).

## Open questions

<!-- Priority levels (spec 196) for the product-manager's PRIORITY line: P0 an outage, data loss, a security problem, or something that blocks all other work; P1 a normal bug or feature; the fallback when nothing signals urgency; P2 cosmetic, a convenience, or something with an easy workaround. A reason is "P<n>: <phrase>", one of: P0: outage, data loss, security problem, blocks all other work; P1: normal bug or feature; P2: cosmetic, convenience, easy workaround. -->

## Changelog

- 2026-11-22 created
- 2026-11-22 priority P1 proposed: P1: normal bug or feature
- 2026-11-22 approved
- 2026-11-22 draft: amend: A8 — a 422 the board did not catch shows the API msg at Title (user decision on the web escalation)
