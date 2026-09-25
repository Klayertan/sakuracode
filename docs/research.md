# Phase 0 Research — Sakura Code

Research date: **2026-09-25**. Scope: HANDOFF §19 Phase 0 and §25 A.

> **How to read this document**
>
> Each fact carries a confidence tag:
>
> - **[V] Verified.** Read directly in a primary source (source code, package registry metadata).
> - **[S] Search excerpt.** Taken from a web-search excerpt of the cited page. The page itself could
>   not be opened because this research environment's network policy blocks `*.sakura.ad.jp`,
>   `opencode.ai`, `huggingface.co`, `qiita.com` and `docs.vllm.ai`. Treat [S] facts as
>   likely but not confirmed. Re-check them against the official page before they drive code.
> - **[TODO]** Not confirmed. Anything marked this way must not be guessed in code (HANDOFF §28.6).
>
> Items in the §7 checklist at the end need someone with a browser and a Sakura account.

---

## 1. Sakura 高火力 DOK — API

| Item | Finding | Conf. | Source |
|---|---|---|---|
| API published | 2024-08-13. The API covers task registration and automation. | [S] | [Announcement 2024-08-13][dok-api-announce] |
| Spec | OpenAPI-style reference, "高火力 DOK APIドキュメント (1.0.0)" | [S] | [API spec][dok-api-spec] |
| Base URL | `https://secure.sakura.ad.jp/cloud/zone/{zone}/api/managed-container/1.0`. The example uses zone `is1a` (Ishikari). | [S] | [API spec][dok-api-spec] (search excerpt) |
| Known paths | `…/tasks/`, `…/tasks/{taskId}/containers/{containerIndex}/stream/` | [S] | same |
| Auth | HTTP Basic auth. User = Sakura Cloud **API access token**, password = **access token secret**. | [S] | same |
| Task cancel | A dedicated task-cancel API exists. It is separate from the artifact API. | [S] | [Qiita: DOK API usage][qiita-dok-api] (excerpt) |
| Status values | `status` field; one observed value is `waiting`. Full enum unknown. | [S]/[TODO] | same |
| Exact request/response fields | **Unknown.** Needed: plan identifier, image, registry credential reference, command/env, HTTP port exposure, execution-time limit, endpoint URL in the response, cancel path. | **[TODO]** | Needs [API spec][dok-api-spec] |

### 1.1 Execution time limit (critical for §8.1 / §8.4)

| Item | Finding | Conf. | Source |
|---|---|---|---|
| Time-limit setting | 2025-10-14: "タスク実行時間上限設定" launched, together with SSH access. | [S] | [Announcement 2025-10-14][dok-timelimit-2025] |
| Behaviour | When the limit is exceeded the task is **automatically cancelled**. Presets are selectable at task creation. | [S] | [FAQ][dok-faq] (excerpt) |
| **Default when unspecified** | **20 days.** That is the maximum guaranteed task runtime. | [S] | [FAQ][dok-faq], [Announcement 2026-05-21][dok-unlimited-2026] |
| **"Unlimited" option** | Since 2026-05-21 the max runtime can be set to **unlimited** (tasks and notebooks). | [S] | [Announcement 2026-05-21][dok-unlimited-2026] |
| Stop lag | Stopping after the limit may be delayed by **a few minutes**. | [S] | [Announcement 2026-05-21][dok-unlimited-2026] |
| Settable via API? Field name? Granularity (presets only, or any number of minutes)? | Unknown | **[TODO]** | [API spec][dok-api-spec] |

> ⚠️ **Safety implication.** A task created without a limit runs for up to **20 days**, and "unlimited"
> is now a selectable value. The controller must **always** send an explicit limit and must read it
> back from the created task. If the limit is not confirmed, cancel the task immediately. See
> [design-review.md](design-review.md) R1.

### 1.2 HTTP exposure (inference endpoint)

| Item | Finding | Conf. | Source |
|---|---|---|---|
| HTTP toggle | When creating a task you can enable HTTP and choose a port. Sakura's own vLLM example uses port **8000**. | [S] | [さくらのナレッジ: gpt-oss-120b on DOK][knowledge-gptoss] |
| URL format | Each task gets its own URL, like `https://<UUID>.container.sakurausercontent.com`. **The URL changes every task.** | [S] | same |
| Access control on that URL | Unknown. Assume it is **public** unless the spec says otherwise. | **[TODO]** | — |
| SSE/streaming, request timeout, body size limits at the Sakura edge | Unknown. This matters for long agent turns. | **[TODO]** | — |
| Known-good precedent | Sakura ran `vllm/vllm-openai` (a gpt-oss tag) on DOK behind the HTTP option. | [S] | [さくらのナレッジ][knowledge-gptoss] |

### 1.3 Other DOK facts

| Item | Finding | Conf. | Source |
|---|---|---|---|
| Private registry | Registry credentials must be registered in DOK first. Using Sakura Container Registry is recommended. | [S] | [Private registry manual][dok-private-registry] |
| Artifacts | Outputs are downloadable after the task ends. The 8GPU plan has **20 GB** of artifact storage. | [S] | [8GPU plan release][dok-8gpu-release] |
| Persistent model cache between tasks | Unknown. Assume none, so every task re-downloads the model or pulls it with the image. | **[TODO]** | — |
| Does the container's main process exiting end the task (and billing)? | Very likely: DOK is a batch-task model. **Still unconfirmed.** | **[TODO]** | — |

---

## 2. Sakura 高火力 DOK — plans and pricing

> HANDOFF §6.6: prices must live in config with an effective date and never be hard-coded. The
> table below is **reference data for writing `pricing.toml`**. It is not a constant for code.

| Plan | GPU | Price (tax incl.) | Status / date | Conf. | Source |
|---|---|---|---|---|---|
| NVIDIA V100 | V100 32GB? | "from ¥57.6/h" after a ~70% price cut on 2025-11-18 | GA | [S] | [V100 price revision][dok-v100-price] |
| NVIDIA H100 (1 GPU) | H100 80GB | **¥0.28/s = ¥1,008/h** (β price, from 2024-08) | β → **GA on 2026-03-25**. The **GA price is unconfirmed.** | [S]/**[TODO]** | [H100 β launch][dok-h100-beta], [GA notice][dok-billing-2026] |
| NVIDIA H100 8GPU 専有 (β) | 8× H100 80GB, 160 vCPU, 1,760 GB RAM, **250 Mbps best-effort network**, 4 TB NVMe + 20 GB artifacts | **¥0.83/s = ¥2,988/h**, **60-second minimum** | β since 2026-01-13 | [S] | [8GPU plan release][dok-8gpu-release] |
| Campaign (expired) | H100 at ¥0.11/s until 2025-07-31 | — | expired | [S] | [Sakura PR on X][x-campaign] |

**Billing method.** It changed for tasks that **start on or after 2026-04-01** [S]
([notice][dok-billing-2026]):

- The fee is calculated **daily**, from actual usage during the task's execution period, and **billed monthly**.
- Execution time is counted **per container, per second**. Billing runs from container execution
  start to execution end. Stopped or queued time is reportedly not billed [S]
  ([zero2one overview][zero2one]).
- **[TODO]** Does the billable time include image pull and model download? A download runs inside
  the container, so it is almost certainly billed. An image pull probably happens before
  "execution start".
- **[TODO]** Is there a 60 s minimum for the 1-GPU H100 plan too?
- **[TODO]** Is there a billing/usage API for reconciling actual cost? Sakura Cloud has billing APIs,
  but DOK coverage is unconfirmed.

**Credits.**

- New DOK users get a **¥3,000** free credit, usable on all plans including H100 [S]
  ([trial notice][dok-trial]).
- **Ambassador ¥100k/month GPU credit.** No public information. Whether it applies to DOK, whether
  unused credit carries over, and whether H100 is included are all **[TODO — ask Sakura ambassador
  contact]**. This is a project-level assumption (HANDOFF §1.1).

**Reference arithmetic.** Uses the β H100 price. Replace it once the GA price is known.

- ¥95,000 safety limit ÷ ¥1,008/h ≈ **94 GPU-hours/month** (≈ 4.3 h per weekday).
- A task left at the default 20-day limit: 480 h × ¥1,008 ≈ **¥483,840**. That is roughly 5× the
  monthly budget.
- 30 minutes of cold start (model download and load) ≈ **¥504 per start**.

---

## 3. Sakura Container Registry

| Item | Finding | Conf. | Source |
|---|---|---|---|
| Image naming | `<registry-name>.sakuracr.jp/<image>:<tag>` | [S] | [Container Registry manual][cr-manual] |
| Auth | Per-registry **users** (username, password and permission), managed in the control panel's user tab, up to 1,000 users. `docker login <name>.sakuracr.jp` | [S] | [Container Registry manual][cr-manual], [DevelopersIO][cr-classmethod] |
| DOK integration | Register the registry credential in DOK before creating tasks that use it. | [S] | [DOK private registry][dok-private-registry] |
| Pull bandwidth / storage limits / pricing | Unknown | **[TODO]** | — |

Recommendation: create a **pull-only** registry user for DOK and a separate push user for your PC.

---

## 4. OpenCode

| Item | Finding | Conf. | Source |
|---|---|---|---|
| Package | npm `opencode-ai`, latest **1.18.32** (published 2026-09-21) | [V] | npm registry metadata |
| License | **MIT** | [V] | npm registry metadata |
| Upstream repo | `github.com/anomalyco/opencode` (issues are filed there) | [S] | [issue #5674][oc-5674] |
| Custom provider | Add a `provider.<id>` entry with `"npm": "@ai-sdk/openai-compatible"`, `options.baseURL` (ending in `/v1`) and a `models` map. Pick the model with `"model": "<provider>/<model-id>"`. Model keys must exactly match the server's `model` name. | [S] | [OpenCode providers docs][oc-providers], [haimaker guide][oc-haimaker] |
| Variable substitution | Config supports `{env:VAR}` and `{file:path}`. | [S] | [OpenCode config docs][oc-config] |
| **Known bug** | `{env:VAR}` **does not work for `apiKey`** in a custom `@ai-sdk/openai-compatible` provider. `{file:…}` works. | [S] | [issue #19946][oc-19946] |
| Older bug | Custom provider `options` (baseURL/apiKey) were not passed to API calls. | [S] | [issue #5674][oc-5674] |
| Credential storage | `/connect` stores keys outside the JSON config (recommended over inline keys). | [S] | [haimaker guide][oc-haimaker] |
| Permission / approval mode (HANDOFF §17) | Config schema for "ask before bash/edit" | **[TODO]** — verify in [config docs][oc-config] | — |

Example shape, **unverified against the current schema**:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "sakura": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Sakura DOK",
      "options": {
        "baseURL": "http://127.0.0.1:8765/v1",          // see design-review C5 (local shim)
        "apiKey": "{file:~/.config/sakura-code/opencode-token}"
      },
      "models": { "<served-model-name>": { "name": "Sakura coding model" } }
    }
  },
  "model": "sakura/<served-model-name>"
}
```

---

## 5. vLLM

Checked against the **vLLM 0.30.0** source (PyPI sdist, released 2026-09-22, Apache-2.0,
Python ≥3.10 <3.15). All findings are [V] (read in source), not runtime-tested.

| Item | Finding | Where |
|---|---|---|
| OpenAI server | `vllm serve <model>`. The official image is `vllm/vllm-openai`. | docs |
| API key | `--api-key` (repeatable) or `VLLM_API_KEY`. The CLI flag takes precedence. | `vllm/entrypoints/serve/middleware/register.py` |
| **Auth scope** | The middleware only guards paths starting with **`/v1`, `/v2`, `/inference`, `/cohere`**. Every other path is **unauthenticated**. | `vllm/entrypoints/serve/middleware/authenticate.py` (`GUARDED_PREFIX`) |
| **Unauthenticated generation path** | `POST /invocations` (SageMaker-compatible) is **registered unconditionally** and routes to the generate/chat handlers. Because it is outside `GUARDED_PREFIX`, **`--api-key` does not protect it**. `/tokenize`, `/metrics`, `/load`, `/ping` and `/health` are also open. | `vllm/entrypoints/launchers/api_server/routers.py`, `vllm/entrypoints/serve/sagemaker/api_router.py` |
| Dev endpoints | `/sleep`, `/pause`, `/collective_rpc` and others are only registered when `VLLM_SERVER_DEV_MODE=1`. Never set it. | `routers.py` |
| Health | `GET /health` returns 200 once the engine is up. It does not require auth. | `serve/instrumentator/basic.py` |
| Metrics (idle detection) | Prometheus `/metrics` exposes `vllm:num_requests_running`, `vllm:num_requests_waiting` and request-success counters. It is **unauthenticated**. | `vllm/v1/metrics/loggers.py` |
| Tool calling | `--enable-auto-tool-choice --tool-call-parser <name>`. Parsers include `qwen3_coder`/`qwen3_xml` (Qwen3-Coder), `hermes` (Qwen2.5/3), `openai` (gpt-oss), `mistral`, `glm45`/`glm47`, `kimi_k2`, `deepseek_v31`, and others. | `docs/features/tool_calling.md`, `vllm/tool_parsers/__init__.py` |

> ⚠️ **Security implication.** `--api-key` is not enough on a public DOK URL. Bind vLLM to
> `127.0.0.1` inside the container, and put a small gateway in front of it that only forwards
> `/v1/*` with a valid bearer token and exposes an unauthenticated `/healthz`. See
> design-review R3 and C2.

---

## 6. Candidate models (single H100 80GB)

All figures are **[S]** from search excerpts. Check weight sizes, context and licence on each
Hugging Face model card before choosing.

| Model | Arch | License | Weights on H100 | Context | vLLM tool parser | Notes |
|---|---|---|---|---|---|---|
| **Qwen3.6-35B-A3B** (FP8 variant published) | MoE, ~3B active | Apache-2.0 | FP8 ≈ 35–40 GB (estimate), leaving ~35 GB for KV cache | 262,144 native | `qwen3_coder` | Released spring 2026. Recommended for high throughput on H100. [HF][hf-q36-moe], [thundercompute][tc-best] |
| **Qwen3.6-27B** (FP8 variant) | Dense | Apache-2.0 | FP8 ≈ 28–30 GB (estimate) | 262,144 | `qwen3_coder` | Released 2026-04-22. Dense, so it's slower per token than the A3B MoE. [HF][hf-q36-27b] |
| **gpt-oss-120b** | MoE, 5.1B active | Apache-2.0 | MXFP4 ≈ 60–65 GB. **Officially fits one 80 GB GPU.** | 128K (verify) | `openai` | Sakura has a DOK walkthrough, so this is the lowest-risk option for "does DOK work at all". Less KV headroom. [thundercompute][tc-best], [さくらのナレッジ][knowledge-gptoss] |
| Qwen3-Coder-Next (80B-A3B) | MoE, 3B active | Apache-2.0 | FP8 ≈ 85 GB, which **does not fit** with KV cache. 4-bit ≈ 46 GB. | 256K | `qwen3_coder` | Strong agentic coder, but a single H100 needs 4-bit quantisation. Needs vLLM ≥0.15. [HF FP8][hf-qcn], [Unsloth][unsloth-qcn] |
| Devstral Small 2 (24B) | Dense | Apache-2.0 | BF16 ≈ 48 GB, FP8 ≈ 24 GB | 256K | `mistral` | SWE-bench Verified 68.0% (vendor figure). [HF][hf-devstral] |

Preliminary view (to be confirmed by a real coding benchmark in Phase 6):

- **Lifecycle bring-up:** a tiny model, so each debug start costs minutes, not a 60 GB download.
- **First real model:** **Qwen3.6-35B-A3B-FP8**. It's Apache-2.0, has headroom for long context,
  and uses the `qwen3_coder` parser that OpenCode-style agents rely on.
- **Comparison run:** **gpt-oss-120b**. Sakura's own documentation covers it, which makes it good
  blog material.

---

## 7. Local stack (package registry check)

| Package | Latest | License | Use |
|---|---|---|---|
| PySide6 | 6.11.2 | LGPL-3.0 / GPL | GUI (Phase 4). LGPL is fine for personal use and dynamic linking. |
| httpx | 0.28.1 | BSD-3 | DOK + health client |
| pydantic | 2.13.5 | MIT | wire models, settings |
| keyring | 25.7.0 | MIT | Windows Credential Manager |
| typer | 0.27.2 | MIT | CLI (Phase 1) |
| platformdirs | 4.11.13 | MIT | config/data dirs on Windows |
| qasync | 0.28.0 | BSD-2 | asyncio + Qt (Phase 4) |

All [V] from PyPI metadata, 2026-09-25.

---

## 8. Confirmation checklist (確認事項)

Please check these in the control panel, the manual, or with your Sakura contact. Items 1–4
block the DOK client code.

1. **DOK API spec.** Save the OpenAPI/spec page ([link][dok-api-spec]) into `docs/vendor/`, or add
   `manual.sakura.ad.jp` to this environment's allowed domains. We need the exact fields for task
   create (plan ID, image, registry credential, command/env, **HTTP port**, **time limit**), the
   task object (status enum, **endpoint URL**, start/end timestamps) and the **cancel** endpoint.
2. **Time limit via API.** Is it settable per task, and at what granularity? Does the created task
   echo it back?
3. **Current H100 (1 GPU) GA price** after 2026-03-25, minimum billing unit, and whether image pull
   or queue time is billed.
4. **Ambassador credit.** Does it cover DOK, and H100 in particular? Is it monthly ¥100k, and does
   it expire?
5. Single-H100 plan **network bandwidth**, vCPU/RAM and local disk (these affect model download time).
6. Does the **task end when the container's main process exits**, and does billing stop then?
7. HTTP endpoint behaviour: public or not, SSE streaming support, and the edge request timeout.
8. Is there a usage/billing API to fetch actual DOK cost per task or month?
9. Can a Sakura Cloud API key be **scoped** to DOK only (least privilege)?
10. Zone: is `is1a` the only DOK zone?

---

[dok-api-announce]: https://www.sakura.ad.jp/corporate/information/announcements/2024/08/13/1968216548/
[dok-api-spec]: https://manual.sakura.ad.jp/koukaryoku-dok-api/spec.html
[dok-faq]: https://manual.sakura.ad.jp/cloud/koukaryoku-container/faqs.html
[dok-timelimit-2025]: https://www.sakura.ad.jp/corporate/information/announcements/2025/10/14/1968221227/
[dok-unlimited-2026]: https://www.sakura.ad.jp/corporate/information/announcements/2026/05/21/1968224613/
[dok-billing-2026]: https://www.sakura.ad.jp/corporate/information/announcements/2026/03/25/1968223993/
[dok-8gpu-release]: https://www.sakura.ad.jp/corporate/information/newsreleases/2026/01/13/1968222777/
[dok-h100-beta]: https://cloud.watch.impress.co.jp/docs/news/1619017.html
[dok-v100-price]: https://www.sakura.ad.jp/corporate/information/newsreleases/2025/11/18/1968221878/
[dok-trial]: https://www.sakura.ad.jp/corporate/information/newsreleases/2024/10/15/1968217344/
[dok-private-registry]: https://manual.sakura.ad.jp/cloud/koukaryoku-container/using-private-registries.html
[x-campaign]: https://x.com/sakura_pr/status/1920297778082951179
[zero2one]: https://zero2one.jp/sakura-ai/high-power-dok-overview/
[qiita-dok-api]: https://qiita.com/salmon111/items/bf0fe00e58c36f81a0b2
[knowledge-gptoss]: https://knowledge.sakura.ad.jp/46179/
[cr-manual]: https://manual.sakura.ad.jp/cloud/appliance/container-registry/index.html
[cr-classmethod]: https://dev.classmethod.jp/articles/sakura-container-registry-beta/
[oc-providers]: https://opencode.ai/docs/providers/
[oc-config]: https://opencode.ai/docs/config/
[oc-haimaker]: https://haimaker.ai/blog/opencode-custom-provider-setup/
[oc-5674]: https://github.com/anomalyco/opencode/issues/5674
[oc-19946]: https://github.com/anomalyco/opencode/issues/19946
[hf-q36-moe]: https://huggingface.co/Qwen/Qwen3.6-35B-A3B
[hf-q36-27b]: https://huggingface.co/Qwen/Qwen3.6-27B
[hf-qcn]: https://huggingface.co/Qwen/Qwen3-Coder-Next-FP8
[unsloth-qcn]: https://unsloth.ai/docs/models/qwen3-coder-next
[hf-devstral]: https://huggingface.co/mistralai/Devstral-Small-2-24B-Instruct-2512
[tc-best]: https://www.thundercompute.com/blog/best-open-source-llms
