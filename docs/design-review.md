# Design Review, MVP Scope and Repo Plan

Reviews [HANDOFF.md](HANDOFF.md) using the facts in [research.md](research.md), following HANDOFF
§25 B/C and §26. Date: 2026-09-25. **No code has been written. This document is awaiting approval.**

---

## 1. Good (keep as-is)

- **G1 — Reuse OpenCode, don't fork it** (§0, §5). OpenCode is MIT and already supports
  OpenAI-compatible custom providers, so the only integration needed is a config file.
- **G2 — CLI before GUI, GUI as a thin layer over a service layer** (§19 Phase 1/4, §28.7–8). This
  makes the safety logic testable without Qt.
- **G3 — Prices are configuration, with an effective date** (§6.6, §28.5). This has already
  proven necessary: the billing method changed on 2026-04-01, and the H100 plan went from β to GA
  on 2026-03-25.
- **G4 — Safety over automation, with a server-side time limit** (§8.1, §28.1). DOK supports a
  per-task limit with auto-cancel.
- **G5 — Explicit state machine** (§12), with button enablement derived from it.
- **G6 — Error-case list** (§13). It's a good test inventory. Most items map onto a fake DOK client.
- **G7 — Record blog numbers from day 1** (§20). Cold-start time and ¥/hour are also inputs to
  safety tuning.
- **G8 — Credentials in keyring, never in git or logs** (§17).

## 2. Risk

| # | Risk | Why it matters | Mitigation (see Change) |
|---|---|---|---|
| **R1** | **The server-side time limit is not automatic.** An unspecified limit means **20 days**, and "unlimited" is now selectable. | One forgotten field could cost about ¥480k at the β H100 rate. | C1: refuse to create a task without an explicit limit, read it back, and cancel if it doesn't match. |
| **R2** | **Idle shutdown runs on the PC** (§8.2). If the PC sleeps, crashes or loses network, nothing stops the GPU until the hard limit. | A laptop sleeping at lunch leaves the GPU billing for up to the full hard limit. | C2: move idle shutdown **into the container** as a self-exit watchdog. The PC-side timer becomes a UX layer only. |
| **R3** | **vLLM `--api-key` does not protect every generation route.** In 0.30.0, `POST /invocations` is registered unconditionally, and like `/tokenize` and `/metrics` it sits outside the guarded prefixes. The DOK URL is probably public. | Anyone who finds the URL can use your GPU. | C2: run vLLM on `127.0.0.1` behind a gateway that forwards only `/v1/*` with a bearer token. |
| **R4** | **Cold start is billed and could be long.** The model download runs inside the billed container. The 8GPU plan has 250 Mbps best-effort networking (single-H100 plan unknown). 60 GB at 250 Mbps ≈ 32 min ≈ ¥500 per start. | Each start costs real money, which pushes toward longer idle timeouts and conflicts with §8.2. | Measure first (Phase 1). Try weights-in-image from Sakura CR (pull may happen before billing starts) against a runtime download. Start with a smaller FP8 model. Show "cold start cost" in the UI. |
| **R5** | **Budget blind spots.** The local DB only knows about sessions this tool started. Tasks created in the control panel, or orphaned by a crash, are invisible. | The ¥95k guard can be bypassed without anyone noticing. | C3: compute month-to-date from the **DOK task list** (source of truth), with SQLite as a cache and history. |
| **R6** | **The limit fires late.** Sakura says the stop can lag by a few minutes. | Budget maths that assumes an exact stop overshoots. | Add a stop-lag margin (e.g. 10 min × rate) to every reservation. |
| **R7** | **Price and billing unknowns**: H100 GA price, minimum billing unit, and whether queued time or image pull is billed. | Estimated cost could be systematically wrong. | Keep pricing in config with an effective date. Label every figure "estimated". Reconcile against the monthly bill manually until a billing API is confirmed. |
| **R8** | **The ambassador credit may not cover DOK or H100.** | The whole premise (§1.1) depends on it. | Confirm before any live GPU test (research §8.4). |
| **R9** | **The endpoint URL changes every task** (`<uuid>.container.sakurausercontent.com`). | OpenCode config must be rewritten each session, and OpenCode's `{env:}` apiKey substitution is buggy for custom providers (#19946). | C5: local shim at a fixed `127.0.0.1` URL. |
| **R10** | **Capacity and queueing.** A task can sit in `waiting`. | The GUI waits forever and the user walks away. The task then starts later with nobody watching. | Add a `WAITING_CAPACITY` state with a queue timeout that cancels the task. The in-container watchdog still covers a late start. |
| **R11** | **Model memory fit and tool-calling quality.** Some candidates don't fit in 80 GB at FP8 (Qwen3-Coder-Next), and tool-call parsers vary. | Failed loads are billed. Agent loops can break in subtle ways. | Pin the model, quantisation, `--max-model-len` and parser in a model profile file. Smoke-test tool calls before real use. |
| **R12** | **Unknown edge behaviour for long streaming requests** (SSE and timeouts at the Sakura HTTP edge). | Long agent turns could be cut off mid-stream. | Test explicitly in Phase 1 with a long streamed completion. |
| **R13** | **API key blast radius.** A Sakura Cloud API token may control the whole account, not just DOK. | A leaked token could affect other resources. | Use a dedicated key with the narrowest permission Sakura allows (research §8.9). Store it only in keyring. |
| **R14** | **Month boundary and clock.** | Which month counts, and when? | Use Asia/Tokyo month boundaries. Rely on server timestamps from DOK. Local clock is a fallback only. |

## 3. Change (proposed amendments to HANDOFF)

**C1 — Three-layer stop defence, with the hard limit mandatory.** Replaces the §8 intent that
"the GUI is not the only safeguard".

| Layer | Where | Survives PC death? | Purpose |
|---|---|---|---|
| L1 DOK time limit | Sakura (server) | ✅ | Absolute ceiling. Always set, never "unlimited", read back after create. |
| L2 In-container watchdog | Container | ✅ | Exits the process after `IDLE_MINUTES` with no `/v1` requests, or at `MAX_SESSION_MINUTES`. |
| L3 Controller | PC (CLI/GUI) | ❌ | Warnings, countdown, Keep Running, stop button, budget guard. |

The session's hard limit is:

```
max_minutes = min(requested,
                  floor((safety_limit - month_to_date - stop_lag_margin) / rate_per_minute))
```

`start` refuses if `max_minutes < minimum_useful_minutes` (e.g. 20). This is §8.3 and §8.4 made
concrete, and it effectively reserves the session's worst-case cost before the task starts.

**C2 — A small `gateway` process inside the container.** Stdlib or `aiohttp`, about 150 lines.

- It listens on the DOK HTTP port. vLLM listens on `127.0.0.1:8001` only.
- It forwards `/v1/*` only, and requires `Authorization: Bearer <token>`. The token is generated
  per session by the controller and passed as a task env var.
- It exposes `GET /healthz` (unauthenticated). This returns `loading` or `ready` from vLLM's
  `/health`, plus `last_request_at` and `idle_seconds`, so the controller can show idle status
  without scraping metrics.
- It implements the L2 watchdog: on idle or max lifetime it terminates vLLM and exits, which ends
  the task.
- Keep Running is `POST /v1/sakura/keepalive` (authenticated), which resets the idle clock.

**C3 — Month-to-date comes from DOK, not only from SQLite.** `cost` lists this month's tasks via
the API and sums (end − start) × the rate in effect. SQLite stores sessions, history and blog
metrics. If the two disagree, show both and use the higher one for guards.

**C4 — State machine additions.**

- Add `WAITING_CAPACITY` between STARTING_TASK and WAITING_CONTAINER, with a timeout that cancels.
- Add `UNKNOWN` for "lost contact with DOK or the container". The UI must still offer Stop.
- `STOPPING` means the cancel was sent and we're polling until a terminal status. It is not "done"
  until confirmed.
- Merge READY and RUNNING: "running" is just READY with a recent request. `IDLE_WARNING` is a
  sub-state.
- `ERROR` always offers, and by default performs, a cancel of any known task ID.

**C5 — Local OpenCode shim (Phase 3).** A tiny local proxy on `127.0.0.1:8765/v1`. It forwards to
the current DOK URL and injects the session token. The OpenCode config stays static, the token
never goes into `opencode.json` (sidestepping bug #19946), and when the GPU is offline or over
budget OpenCode gets a clear error message. Phase 3 can start with plain config generation using
`{file:}` and add the shim later.

**C6 — Stack simplifications.**

- `sqlite3` from the standard library instead of SQLModel. The schema is tiny, so skip the
  dependency.
- **Typer** for the CLI (Phase 1).
- `platformdirs` for paths.
- `respx` or `httpx.MockTransport` for API fakes.
- Python ≥3.11 (for `tomllib`).
- Settings in TOML under the user config dir.
- `pricing.toml` holds `[[plans]] id, gpu, yen_per_second, min_billable_seconds, effective_from,
  source_url`.

**C7 — Model profiles.** Add `models/*.toml` with HF repo, revision, quantisation,
`max_model_len`, tool parser, extra vLLM args and expected VRAM. The container reads the profile
through env vars. This is §14 without hard-coding.

**C8 — Rename `docker/` to `container/`** and add `gateway.py`. Also add a `doctor` CLI command
that checks credentials, registry, pricing freshness and keyring.

## 4. Unknown (blocking items, from research §8)

1. **Exact DOK API schema.** Create-task fields, time-limit field, HTTP port field, endpoint URL
   field, status enum and cancel path. **This blocks `sakura/client.py`.**
2. H100 (1 GPU) GA price and minimum billing unit. Is pull or queue time billed?
3. Ambassador credit applicability to DOK and H100.
4. Does container exit end the task and stop billing? **Needed for C2.**
5. Single-H100 plan network, disk and RAM, and whether any model cache persists between tasks.
6. HTTP edge: public or not, SSE, timeouts.
7. Billing or usage API availability.
8. Whether API keys can be scoped to DOK only.
9. OpenCode's current permission (approval) config keys.

## 5. MVP — "day 1" scope

**Goal:** the §19 Phase 1 success path (PC → DOK task → container ready → endpoint → manual
request → stop), with L1 and L2 safety already in place. No GUI, no OpenCode yet.

Can be built now, offline, in this cloud session:

1. Repo bootstrap: `pyproject.toml`, `src/sakura_code/`, pytest, ruff, and a README with the safety
   notes.
2. `core/`: `pricing.py`, `budget.py` (month-to-date, `max_minutes`, reservation, stop-lag margin,
   JST month boundary) and `state.py` (C4). Pure functions, heavily unit-tested (§19 Phase 2 list:
   month boundary, rounding, stopped/failed/duplicate tasks, clock skew).
3. `security/credentials.py`: keyring wrapper and redacting logger filter.
4. `container/gateway.py` and `Dockerfile`/`entrypoint.sh`: auth, `/healthz`, idle and
   max-lifetime watchdog, with tests against a fake upstream.

Needs the DOK spec (unknown 1), then your PC and account:

5. `sakura/client.py`: list, get, create and cancel against the real schema, with recorded
   fixtures. **Create refuses without a time limit.**
6. CLI: `doctor`, `start --max-minutes`, `status`, `stop` (idempotent, polls to terminal), `cost`.
7. Live smoke test on your PC with a **tiny model**. Record cold-start seconds and ¥, then stop.
   Expected cost is a few hundred yen.

Not in day 1: GUI, OpenCode auto-config or shim, SQLite history (a JSONL log is enough at first),
idle-warning UI, and model benchmarking.

## 6. Implementation order

| Step | Content | Needs |
|---|---|---|
| 1 | Bootstrap, `core/` (pricing, budget, state) and tests | nothing |
| 2 | Container gateway, watchdog, Dockerfile and tests | nothing |
| 3 | DOK client, fixtures, CLI (`doctor/start/status/stop/cost`) | **DOK spec** |
| 4 | Build and push the image to Sakura CR, then run the live smoke test with a tiny model | your PC, credentials, **credit confirmation** |
| 5 | Budget engine and SQLite history, reconciled against the DOK task list (C3) | step 3 |
| 6 | Real model profile (Qwen3.6-35B-A3B-FP8), plus tool-call and long-stream tests | step 4 |
| 7 | OpenCode integration (config generation, then the C5 shim) | step 6 |
| 8 | PySide6 GUI over the service layer | step 5 |
| 9 | Idle-warning UX (L3) and Keep Running | steps 2 and 8 |
| 10 | Real coding tasks A/B/C, then the blog | all |

## 7. Recommended repo structure

```text
sakuracode/
├─ README.md
├─ pyproject.toml              # hatchling, python>=3.11, console script `sakura-code`
├─ .gitignore
├─ config/
│  ├─ pricing.example.toml     # rates + effective_from + source_url, never used as a silent default
│  └─ models/
│     ├─ tiny-smoke.toml
│     └─ qwen3.6-35b-a3b-fp8.toml
├─ docs/
│  ├─ HANDOFF.md  research.md  design-review.md
│  ├─ architecture.md  security.md  blog-notes.md
│  └─ vendor/                  # saved DOK API spec (once provided)
├─ src/sakura_code/
│  ├─ __init__.py
│  ├─ cli.py                   # Typer; thin, calls services only
│  ├─ settings.py              # TOML settings in platformdirs user config dir
│  ├─ core/                    # pure logic, no I/O
│  │  ├─ pricing.py
│  │  ├─ budget.py
│  │  ├─ guard.py
│  │  └─ state.py
│  ├─ sakura/                  # the only code that knows the DOK wire format
│  │  ├─ client.py
│  │  └─ models.py
│  ├─ services/
│  │  ├─ session.py            # start/stop/status orchestration, L3 safety
│  │  └─ ledger.py             # month-to-date from DOK + SQLite (C3)
│  ├─ inference/health.py      # polls gateway /healthz
│  ├─ opencode/                # Phase 3
│  │  ├─ config.py
│  │  └─ shim.py
│  ├─ storage/db.py            # stdlib sqlite3
│  ├─ security/credentials.py  # keyring + log redaction
│  └─ gui/                     # Phase 4 (PySide6)
├─ container/
│  ├─ Dockerfile               # FROM vllm/vllm-openai:<pinned>
│  ├─ entrypoint.sh
│  ├─ gateway.py               # auth proxy + idle/max-lifetime watchdog (C2)
│  └─ README.md
└─ tests/
   ├─ core/ (test_pricing, test_budget, test_guard, test_state)
   ├─ sakura/ (test_client with recorded fixtures)
   ├─ services/ (test_session with fake client)
   └─ container/ (test_gateway)
```

## 8. Decisions requested

1. Approve C1–C8, in particular the in-container watchdog (C2) and treating DOK as the source of
   truth for spend (C3).
2. Provide the DOK API spec: save the page, or allow `manual.sakura.ad.jp` in this environment.
   Also answer research §8 items 2–4 when you can.
3. Approve day-1 scope (§5) and the go-ahead to bootstrap steps 1–2, which need no credentials and
   no GPU spend.
