# Reference analysis: `neural-maze/production-ocr-course`

| | |
|---|---|
| Version | v1.2 — 2026-11-16 (v1 2026-11-15; v1.1 adds section 11; v1.2 adds section 12) |
| Reference commit | `3271469` on `main` (2026-09-24), identical to upstream HEAD on the analysis date |
| Purpose | Shared ground truth for the rebuild. Read this before planning or implementing any milestone. |
| Status | Analysis only. The questions listed in section 10 were settled on 2026-11-16; see `plans/01-decisions.md`. |

**How to read the evidence tags.** `[code]` = read directly in the reference repository. `[sdk]` = read in the `glmocr==0.1.4` wheel source. `[primary]` = read in a primary source (model card, vLLM source/docs, paper). `[vendor]` = a number published by whoever built the thing, not independently measured. `[inferred]` = reasoning from verified facts, not observed at runtime. Nothing in this analysis was executed against a GPU; every runtime behaviour is `[inferred]` until we measure it.

---

## 1. What the reference is

A six-week course repo (The Neural Maze: Miguel Otero Pedrido, Antonio Zarauz Moreno) that builds a self-hosted document-OCR service on Kubernetes. 487 stars, 145 forks, 21 commits, 3 merged community PRs, no issues. `LICENSE` is Apache-2.0; the README says MIT.

**Use case.** A client (framed as an AI agent using a tool) uploads a PDF or image and gets back structured Markdown plus layout JSON. The service is asynchronous (submit, then poll), runs a small open-weight vision-language model instead of a frontier API, and scales GPU capacity to zero when idle. The value proposition is privacy, cost control and throughput for bursty document workloads.

**The core idea: two-stage, layout-first OCR.** A small deterministic detector (PP-DocLayoutV3, 33M params) finds and labels regions on each page. Each region is cropped and sent to a generative VLM with a task-specific prompt (text, table, formula). The claim is that narrowing what the VLM sees reduces hallucination and token cost compared with feeding it whole pages.

**Size.** About 10K lines of real text. The final system is small: 215 lines of Rust, 141 lines of Python, a 301-line SDK config, one shell entrypoint, and about 450 lines of Kubernetes YAML. The rest is docs, Makefiles and per-week copies.

---

## 2. Final architecture (root of `main`)

```
Client ──POST /process (multipart, ≤10MB)──▶ Producer (Rust/Axum, CPU pool)
                                               │ HSET task:<id> {status,filename,extension,data=b64}
                                               │ LPUSH ocr_tasks <id>
                                               ▼
                                            Redis 7 (single pod, no persistence)
                                               ▲                         │ BRPOP + RPOP (collector)
Client ──GET /status/<id>──▶ Producer ─HGETALL─┘                         ▼
                                                     Worker (Python, T4 GPU pool)
                                                       ├ write /dev/shm/<id>.<ext>
                                                       ├ GLM-OCR SDK: rasterise → PP-DocLayoutV3 → crop
                                                       └ one chat-completion per region ──▶ vLLM (A100 pool)
                                                                                              Qwen/Qwen3.5-4B
KEDA: worker ← Redis list length | vLLM ← Prometheus vllm:num_requests_waiting | API ← CPU 70%
```

### 2.1 Producer — `client_rt_producer/src/main.rs` `[code]`

- Axum 0.6, tokio, `redis` 0.23, tower-http 0.4. Listens on `0.0.0.0:5000` (hardcoded).
- `POST /process`: reads multipart field `file`, generates a UUID v4, base64-encodes the bytes, then runs `HSET task:<id> status queued filename … extension … data <b64>` followed by `LPUSH ocr_tasks <id>`. Returns `202 {task_id, status:"queued"}`.
- `GET /status/:task_id`: `EXISTS`, then `HGETALL`, returns `{task_id, status, result, error}`. `result` is the worker's JSON string re-parsed.
- `GET /health`: always 200.
- Body limit 10 MB via `RequestBodyLimitLayer`.
- Image: `rust:1.88-alpine` builder with cargo-chef, `alpine:3.20` runtime.

### 2.2 Worker — `client_rt_consumer/worker.py` `[code]`

- Creates `GlmOcr(config_path, enable_layout=True)` at import time. The SDK blocks here until vLLM answers a "hello" chat completion, up to `connect_timeout` (600 s) `[sdk]`.
- Loop: `BRPOP ocr_tasks` (2 s timeout) for the first task, then `RPOP` in a 10 ms poll until the batch has `MAX_BATCH_SIZE` (4) tasks or `BATCH_WINDOW_MS` (100) elapses.
- Per task: `HGETALL`, set `status=processing`, base64-decode into `/dev/shm/<id>.<ext>`, `chmod 0644`.
- One call for the whole batch: `await asyncio.to_thread(ocr_engine.parse, paths)`. The loop does not pop more work until it returns.
- Per result: `HSET status=done result=<json{markdown,layout}> data=""`. On any exception the whole batch is marked `failed`.
- Image: `nvidia/cuda:12.1.1-cudnn8-devel-ubuntu22.04`, `glmocr[selfhosted,layout]==0.1.4`, `transformers==5.3.0`, other pip deps unpinned, runs as root.

### 2.3 What the GLM-OCR SDK actually does inside `parse()` `[sdk]`

1. Rasterise PDFs with `pypdfium2` at `pdf_dpi` (200), long side capped at 3500 px. The Dockerfile also installs PyMuPDF, pdf2image and poppler, which 0.1.4 does not use.
2. Run `PPDocLayoutV3ForObjectDetection` through Hugging Face `transformers` in PyTorch, in-process, on the T4. No PaddlePaddle, no ONNX. Pages are batched up to `layout.batch_size`.
3. Map each of 25 layout labels to a task: `text`, `table`, `formula`, `skip` (kept, not sent to the VLM) or `abandon` (dropped: headers, footers, page numbers, footnotes).
4. Crop each region by bounding box, resize to multiples of 28 px (GLM-OCR's patch geometry), JPEG-encode, base64.
5. Send one OpenAI-style `/v1/chat/completions` request per region (image + prompt, no system prompt, no streaming) through a `ThreadPoolExecutor(max_workers=min(max_workers, 128))` — `pipeline.py:437`.
6. Retry on 429/5xx with exponential backoff. A region that still fails gets `content=""` and a warning; the document is still returned as successful.
7. Format: sort regions, clean repetition, turn titles into `#`/`##`, wrap formulas in `$$`, merge split blocks. Returns `markdown_result` and `json_result` (pages → regions with `label`, `content`, `bbox_2d` normalised to 0–1000).

SDK maturity: PyPI classifier Alpha, latest release 0.1.5 (2026-04-08), one commit since April `[primary]`.

### 2.4 Inference server — `server/` `[code]`

`vllm/vllm-openai:v0.21.0` (released 2026-05-15; v0.33.0 is current) serving weights from a shared volume:

```
vllm serve /mnt/models/Qwen/Qwen3.5-4B --served-model-name Qwen/Qwen3.5-4B
  --gpu-memory-utilization 0.9 --max-model-len 16384
  --max-num-batched-tokens 262144 --max-num-seqs 512        # 512 from the ConfigMap; script default 256
  --limit-mm-per-prompt '{"image":1,"video":0}' --mm-encoder-tp-mode data
  --trust-remote-code --load-format instanttensor
  --reasoning-parser qwen3 --default-chat-template-kwargs '{"enable_thinking": false}'
  --enable-chunked-prefill
```

Model facts `[primary]`: `Qwen/Qwen3.5-4B` is a general multimodal model, 4.66B params, about 9.3 GB in BF16, Apache-2.0, native context 262,144, one image token per 32×32 px, thinking on by default. Its own card reports 86.2 on OmniDocBench v1.5.

### 2.5 Kubernetes — `k8s/aks`, `k8s/gke` `[code]`

- Four Deployments (`ocr-api`, `ocr-worker-rt`, `ocr-vlm`, `ocr-redis`) in namespace `default`, one kustomize `ConfigMap` shared by all, images tagged `:latest` with `imagePullPolicy: Always`.
- Weights live on a ReadWriteMany PVC (`azureblob-fuse-premium` 300Gi on AKS; Filestore 1Ti on GKE) mounted read-only at `/mnt/models`. A Job downloads `PaddlePaddle/PP-DocLayoutV3_safetensors` and `Qwen/Qwen3.5-4B` from Hugging Face with plain `urllib`.
- Worker mounts an in-memory `emptyDir` (4Gi) at `/dev/shm` because the Kubernetes default is 64 MB.
- AKS and GKE manifests are two full copies that differ only in image prefix, node selector keys, taint keys, storage class and one load-balancer annotation.

| Pool | AKS SKU | GKE machine | Autoscale | Workload |
|---|---|---|---|---|
| system | 1 unnamed default node | `e2-standard-4` | AKS none; GKE 1–3 | KEDA, Prometheus, GPU operator, ingest Job |
| `apinp` | `Standard_DS2_v2` | `n2-standard-2` | 1–5 | Producer |
| `redisnp` | `Standard_E4ds_v5` | `n2-highmem-4` | 1–3 | Redis |
| `gpunpt4` | `Standard_NC16as_T4_v3` (1×T4) | `n1-standard-4` + T4 | 0–4 | Worker |
| `gpunpa100` | `Standard_NC24ads_A100_v4` (1×A100 80GB) | `a2-ultragpu-1g` | 0–4 | vLLM |

### 2.6 Autoscaling — `keda-scaler.yml` `[code]`

| ScaledObject | Replicas | Triggers |
|---|---|---|
| worker | 0–10, poll 15 s, cooldown 300 s | cron (Mon–Fri 08:00–18:00 America/New_York → 1 replica); Redis `ocr_tasks` list length 1 |
| vLLM | 0–4, poll 10 s, cooldown 300 s | same cron; Prometheus `sum(vllm:num_requests_waiting{kubernetes_namespace="default"})` ≥ 1 |
| API | 1–5 | CPU 70% |

Platform add-ons are installed by hand with unpinned Helm charts: KEDA, kube-prometheus-stack, and (AKS only) the NVIDIA GPU Operator.

### 2.7 Perimeter and agent access

- `ocr-api-service` is a `LoadBalancer` with an internal-LB annotation, so it only gets a private IP `[code]`.
- Azure API Management policy (`apim-policy.xml`): subscription key, or JWT if an `Authorization` header is present, plus 100 calls / 60 s rate limit `[code]`. GCP has no equivalent artefact, only prose.
- MCP server (only in `week6_apim_mcp_deployment/mcp_server/`, not at root): FastMCP, one tool `parse_document(file_path, include_layout=False)`, stdio or SSE, submits to the Producer and polls every 0.5 s for up to 120 s.

### 2.8 Infrastructure as code

- GKE: Terraform in `infra-gke/` (VPC, cluster, 5 node pools, Artifact Registry, service account). Local state, no add-ons, no Kubernetes objects.
- AKS: none. Manual `az` commands in docs, wrapped by per-week Makefiles.
- No CI, no tests, no linter configuration anywhere on `main`.

---

## 3. How the system evolved

The weekly folders are not ancestors of the root code; they are later, separately tuned forks, and fixes do not propagate between them.

| Stage | Model | Shape | Notes |
|---|---|---|---|
| Week 3, first drop | `rednote-hilab/dots.mocr` | vLLM only | Replaced a week later (`717a3d1`) |
| Week 3 | `baidu/Unlimited-OCR` (3B, single-stage) | vLLM on one A100 | `MAX_MODEL_LEN=32768`, `MAX_NUM_BATCHED_TOKENS=65536`, `MAX_NUM_SEQS=64`, 10 images per prompt |
| Week 4 | same | **Synchronous** Rust gateway (axum 0.7, 1,430 lines) in front of vLLM | `POST /process` (JSON base64) and `/process-stream` (SSE); rasterises with `pdftoppm` at 150 DPI; up to 10 pages per VLM call; layout comes from the model's own `<|det|>` tags. No queue, no Redis, no layout model |
| Week 5 | `Qwen/Qwen3.5-2B` | Async: Producer → Redis → Worker (SDK + PP-DocLayoutV3 on T4) → vLLM | Batch 8 / 150 ms; charts and images **are** described |
| Week 6 | same | + MCP server, APIM walkthrough | `worker.py` here still has the LPOP bug |
| Root `main` | `Qwen/Qwen3.5-4B` | Async, 4 pools | Batch 4 / 100 ms; charts and images **skipped** |

Why the week-4 → week-5 redesign happened is never argued in the repo. The likely drivers `[inferred]`: GPU cold starts take minutes while a synchronous HTTP request has a 300 s budget; a queue length is a scaling signal that exists when replicas are at zero; and a two-stage pipeline needs a Python GPU worker for the layout model.

### 3.1 Two unmerged upstream branches (where the authors are heading)

- **`origin/restructure-repo`** (2026-10-14): one final system under `services/` + `deploy/`, a `course/` folder with a page per week (week 0–6, with checkable deliverables), `docs/api.md` and `docs/architecture.md` split out of the README, a single root `Makefile` (`make up / test / down`), `scripts/test_submit.py`. AKS only; GKE docs deleted; model back to Qwen 3.5 2B. Its week-3 page lists enabling MTP as a *stretch exercise*.
- **`origin/feat/local_dpl`** (2026-10-28): a different, local-first system. FastAPI + Inngest durable workflows + PostgreSQL + a web UI, with an `OcrEngine` abstract base class and two engines (Apple MLX; llama.cpp server via Docker). Model is single-stage `Unlimited-OCR-2`. Records per-batch telemetry (latency, prompt/generation tokens per second, pages per second). First appearance of `pyproject.toml`, ruff, mypy and three unit tests.

Useful signals from these: the authors are consolidating copies into one source of truth, moving toward a pluggable engine interface, and adding durable execution and telemetry — all gaps identified independently below.

---

## 4. Claims versus what the code does

| README claim | Finding | Evidence |
|---|---|---|
| "1.86 pages/second" | GLM-OCR's own published figure for its 0.9B model at single concurrency. No measurement of this Qwen stack exists in the repo or the course articles. | `[primary]` GLM-OCR report arXiv 2603.10910, Table 6 |
| "~50% throughput increase" from Multi-Token Prediction | Also GLM-OCR's figure. MTP is **not enabled**: no `--speculative-config` anywhere. vLLM's Qwen3.5 recipe says MTP *lowers* throughput under high concurrency. | `[code]` grep; `[primary]` vLLM recipe |
| "Fine-grained chart & image description" | Root config maps `chart` and `image` to `skip`; the `image_desc` prompt is unreachable. Weekly snapshots do map them. | `[code]` `config.yaml:263-265` |
| `max_workers: 512` "matches `MAX_NUM_SEQS`" | SDK caps the pool at 128 per worker pod. | `[sdk]` `pipeline.py:437` |
| `connection_pool_size`, `layout.workers`, `image_expect_length` tuning | `layout.workers` and `image_expect_length` are never read by the SDK. | `[sdk]` |
| "Each crop ~6K tokens, so 262K batched tokens prefills ~42 images" | At 32 px per token a full A4 page at 200 DPI is about 3.8K tokens; crops are hundreds. 6,144 is the unused `image_expect_length`. | `[primary]` model config; `[inferred]` arithmetic |
| `MAX_MODEL_LEN` "reduced from 32K, frees gigabytes" | Native context is 262,144, and vLLM's KV cache is a shared pool, not reserved per sequence. | `[primary]` |
| "Zero-copy `/dev/shm` handoff between layout and inference engines" | `/dev/shm` holds one temp file local to the worker. The real path is Redis base64 → file → image → JPEG → base64 → HTTP to another node. | `[code]`, `[sdk]` |
| "Layout pod pre-processes batch N+1 while vLLM handles batch N" | The worker awaits each batch before popping more. | `[code]` `worker.py:134` |
| "State-of-the-art on OmniDocBench" | Qwen3.5-4B scores 86.2 (v1.5, page-level). Specialised 0.9–1.3B OCR models score 95–97 (v1.6). | `[primary]`/`[vendor]` |
| "Atomic `HSET`" | The hash write is one command, but `HSET` then `LPUSH` are two round-trips with no transaction. | `[code]` `main.rs:138-152` |
| License: MIT | `LICENSE` is Apache-2.0. | `[code]` |

The architecture is sound in outline. The performance narrative is inherited from an earlier GLM-OCR-based version of the project (the kustomize label is still `system: aks-ocr-glmocr`) and was not re-measured after the model swap.

---

## 5. Defect and gap register

IDs are stable so plans and PRs can reference them.

### Reliability and correctness

| ID | Issue | Evidence |
|---|---|---|
| R1 | At-most-once delivery. `BRPOP` removes the task; a worker crash, OOM or KEDA scale-down mid-batch leaves tasks in `processing` forever. No ack, requeue, visibility timeout or dead-letter queue. | `[code]` |
| R2 | Batch blast radius. Any exception fails every task in the batch; an error while preparing task *k* leaves task *k* stuck in `processing` and later-popped tasks stuck in `queued`. | `[code]` `worker.py:32-51,86-90` |
| R3 | No TTL on `task:*` keys and no Redis `maxmemory` policy. Results accumulate until the pod is OOM-killed. | `[code]` |
| R4 | Redis is a single Deployment with no volume, auth or probes. A restart loses every queued task and every result. | `[code]` |
| R5 | Document bytes are stored base64 in Redis (33% inflation), and `GET /status` does `HGETALL`, pulling the whole payload on every poll while the task is queued. | `[code]` `main.rs:188` |
| R6 | A new Redis connection per HTTP request, opened before the body is read. | `[code]` `main.rs:103,174` |
| R7 | File type is taken from the filename (`split('.').last()`), unsanitised and unvalidated; a name with no dot becomes the "extension". No magic-byte check, no page limit (`pdf_max_pages: null`). | `[code]` |
| R8 | Failed regions return empty text and the document is still `done`. No partial-failure signal. | `[sdk]` |
| R9 | `parse()` defaults to `save_layout_visualization=True`, writing a temp directory per call that the worker never cleans. | `[sdk]` `api.py:192`; leak `[inferred]` |
| R10 | Synchronous Redis client inside `async` code; no SIGTERM handling. | `[code]` |

### Autoscaling

| ID | Issue | Evidence |
|---|---|---|
| A1 | No `ServiceMonitor`, `PodMonitor` or scrape annotation exists, so Prometheus never scrapes vLLM and the vLLM trigger has no data. The `kubernetes_namespace` label also is not what kube-prometheus-stack emits. | `[code]` grep; effect `[inferred]` |
| A2 | Scale-from-zero deadlock. vLLM has `minReplicaCount: 0` and its only non-cron trigger is a metric emitted by vLLM itself. Outside the cron window nothing can wake it, and workers block for up to 600 s on `connect()`. | `[code]`; effect `[inferred]` |
| A3 | Worker `maxReplicaCount` 10 but the T4 pool maxes at 4 nodes. | `[code]` |
| A4 | Deployments set `replicas:` while KEDA owns scaling, so every `kubectl apply -k` fights the HPA. | `[inferred]` |
| A5 | `listLength: 1` requests one T4 node per queued task, although each worker drains 4 per batch. | `[code]` |

### Performance and configuration

| ID | Issue |
|---|---|
| P1 | Effective VLM concurrency is 128 per worker, not 512 (`[sdk]`). |
| P2 | Dead configuration: `layout.workers`, `image_expect_length`, `VLLM_URL`, and the `GLMOCR_PIPELINE_OCR_API_API_*` env vars (the SDK's real names are `GLMOCR_OCR_API_HOST/PORT`) (`[sdk]`). |
| P3 | `MAX_NUM_BATCHED_TOKENS=262144` is 128× vLLM's A100-specific default and is unbenchmarked (`[primary]`). |
| P4 | Sampling parameters and the 28 px resize are GLM-OCR tunings applied to a different model; the formula post-processor can double-wrap Qwen's `$…$` output (`[sdk]`, `[inferred]`). |
| P5 | A full-page crop plus `max_tokens: 8192` can exceed `max-model-len 16384` (`[inferred]`). |
| P6 | Worker image is a CUDA `-devel` base; `logging.level: DEBUG` in production config. |

### Security

| ID | Issue |
|---|---|
| S1 | The ingest Job disables TLS verification, pulls unpinned `resolve/main`, verifies no checksums, and treats any existing file as complete; vLLM then runs with `--trust-remote-code`. |
| S2 | No authentication between any in-cluster components; Redis and vLLM are open to every pod. No NetworkPolicies, no namespaces, no securityContext, containers run as root. |
| S3 | If a `ZHIPU_API_KEY` or `GLMOCR_API_KEY` env var is ever present in the worker pod, the SDK silently switches to cloud mode and sends documents to Zhipu, overriding `maas.enabled: false` (`[sdk]` `api.py:104-111`). |
| S4 | APIM policy validates no audience or issuer, has an unfilled `{tenant-id}`, and the documented setup never binds the API to its backend. |
| S5 | MCP server over SSE has no authentication, and in-cluster it reads paths from the pod's filesystem, so it cannot parse a developer's local files. |
| S6 | `:latest` tags with `imagePullPolicy: Always`; unpinned Helm charts and pip dependencies. |

### Engineering process

| ID | Issue |
|---|---|
| Q1 | No tests of any kind on `main`. Both community bug fixes (queue order; MCP path traversal) are things a basic test would have caught. |
| Q2 | No CI/CD, no lint or format configuration, no pre-commit. |
| Q3 | Copy-paste snapshots drift: the queue-order fix never reached the week-6 copy. |
| Q4 | Docs contradict manifests in many places (regions, taints, paths, pool names, GPU driver strategy), and the manual GKE path cannot work as written. |
| Q5 | No load test, no quality evaluation, no cost model. |

---

## 6. External landscape (as of 2026-11-15)

Scores and throughput here are `[vendor]` unless stated, gathered through a web summariser. Re-verify any figure before publishing it.

### 6.1 Models

The top of OmniDocBench v1.6 is now 0.8–1.3B specialised OCR models scoring 96–97. A general 4B VLM is no longer the quality leader, and it is 4–5× larger.

| Candidate | Size | Licence | Architecture | Quality | Why it matters here |
|---|---|---|---|---|---|
| `Qwen/Qwen3.5-4B` (reference) | 4.66B | Apache-2.0 | General VLM, used on crops | 86.2 ODB v1.5 (page-level) | Baseline to reproduce |
| GLM-OCR | 0.9B | MIT | Two-stage with PP-DocLayoutV3 | 95.22 ODB v1.6 | The SDK's native model; prompts, resize and post-processing are tuned for it; supports MTP in vLLM |
| PaddleOCR-VL-1.6 | 0.9B | Apache-2.0 | Two-stage with PP-DocLayoutV3 | 96.34 | Same layout model; vLLM v0.33 adds a batched encoder |
| MiniCPM-OCR 1.0 | 1.3B | Apache-2.0 | Two-stage with PP-DocLayoutV3 | 96.44 | Official quantised builds, including GGUF for CPU |
| OvisOCR2 | 0.8B | Apache-2.0 | End-to-end page-level | 96.47 | Tests the "no layout stage" alternative |
| LightOnOCR-3 (0.8B / 4B) | 0.9B / 5B | Apache-2.0 | End-to-end | 85.5 / 86.3 olmOCR-bench | The 4B is a like-for-like size comparison |
| `baidu/Unlimited-OCR(-2)` | 3B | MIT (unconfirmed) | End-to-end, multi-page | 94.00 | What weeks 3–4 and the local branch use |

Avoid for an open-source project without reading the licence: OXR-1.0 (custom), Chandra OCR 2 (modified OpenRAIL-M), HunyuanOCR (Tencent community licence), Nanonets-OCR2 (unstated).

### 6.2 Hardware

- An A100 80GB is not needed for memory. A 4B model is about 10 GB of weights; 0.9–1.3B models are about 2–3 GB. A single 24 GB card can hold layout detection and the VLM together `[inferred]`.
- The T4 is a poor VLM host (no bfloat16) but fine for layout detection.
- AWS has no single-A100 instance. Upstream's own AWS port used an L40S 48 GB (`g6e`) and noted that the A100 token budget must be retuned or the card OOMs at startup.

| GPU | Indicative on-demand price (USD/h) |
|---|---|
| T4 16GB | AWS `g4dn.xlarge` 0.53; Azure 0.53 |
| L4 24GB | AWS `g6.xlarge` 0.81; GCP about 0.70; RunPod 0.44–0.69; Modal 0.80 |
| A10G 24GB | AWS `g5.xlarge` 1.01 |
| L40S 48GB | AWS `g6e.xlarge` 1.86; RunPod 0.79–1.09 |
| A100 80GB | Azure 3.67 (spot 0.68); GCP about 5.07; RunPod 1.19–2.19; Modal 2.50 |
| RTX 4090 / 5090 | RunPod 0.34 / 0.69 (not on hyperscalers) |

Control planes: EKS $0.10/h; GKE $0.10/h with one free zonal cluster; AKS free tier $0. New accounts on all three clouds start with zero GPU quota and approval takes hours to days.

### 6.3 Tooling worth knowing

- **Serving:** vLLM remains the default. SGLang is the credible A/B. TGI is archived.
- **Developing without a GPU:** `llm-d-inference-sim` is an OpenAI-compatible vLLM stand-in with load-dependent latency and vLLM-style metrics; `fake-gpu-operator` advertises fake `nvidia.com/gpu` so scheduling and taints can be tested on kind; PP-DocLayoutV3 runs on CPU (`layout.device: cpu`); GGUF builds of small OCR models run under llama.cpp.
- **Quality benchmarks:** OmniDocBench v1.6 (1,651 pages; dataset is research-only, non-commercial) and olmOCR-bench (1,403 PDFs, 7,010 unit tests; ODC-BY). They rank models differently, so use both.
- **Load testing:** GuideLLM and NVIDIA AIPerf for the inference server; k6 or Locust for the end-to-end submit → poll path.

---

## 7. Constraints on our side

- **Development machine:** a laptop with no GPU and limited memory. No model inference and no realistic local Kubernetes cluster with the full stack.
- **Consequence:** unit tests and small component tests run locally; multi-service and Kubernetes tests belong in CI (GitHub-hosted runners are free for public repositories and have more memory); anything touching a real GPU is a deliberate, time-boxed, paid cloud session.
- **No standing cloud GPU quota** on any hyperscaler, which is why real-GPU work runs on a pay-per-second GPU cloud.

---

## 8. What is worth keeping from the reference

- The decomposition: thin ingest API, queue as shock absorber, layout on a cheap GPU, generation on a fast GPU, each scaled by its own signal.
- Scale-to-zero GPU pools with a cron warm window.
- Weights on shared storage, downloaded once, instead of baked into images or pulled at pod start.
- Layout-first cropping as a hallucination and token-cost control.
- An MCP tool as the agent-facing interface, with workspace confinement.
- The incremental teaching path: inference alone → synchronous gateway → asynchronous pipeline → perimeter and agents.

## 9. Where a rebuild can be demonstrably better

1. **Measure everything the reference asserts.** A reproducible quality + throughput + cost harness, and a model comparison, would be original results rather than repeated claims.
2. **Make delivery reliable** (R1–R4): acknowledged queue semantics, TTLs, per-task isolation, persistence.
3. **Make autoscaling actually work** (A1–A5), with a recorded cold-start timeline as proof.
4. **Keep large payloads out of Redis** (R5).
5. **A pluggable engine boundary** so models and serving backends can be swapped and compared.
6. **Real SDLC:** tests, CI, pinned and scanned images, IaC for whichever cloud is chosen, one source of truth instead of copies.
7. **Security defaults** (S1–S6).

---

## 10. Questions this analysis raised

All ten were settled on 2026-11-16. The answers are in `plans/01-decisions.md`; this list is kept to show what the analysis left open.

1. Repository identity: name, where it lives on disk relative to the reference clone, licence, and how the reference is credited.
2. Fidelity: reproduce the reference's design first and then fix it in visible steps, or build the corrected design from the start.
3. Cloud and budget: which provider hosts the real deployment, how much GPU spend is acceptable, and whether model benchmarking happens on a cheaper GPU cloud than the Kubernetes demo.
4. Default model and the shortlist for comparison.
5. Whether to keep the GLM-OCR SDK as the orchestration layer (alpha, capped at 128 threads, several no-op knobs) or own that logic.
6. Queue technology and payload storage.
7. Milestone order and what "done" means for each.
8. Local and CI development loop given the hardware above.
9. Perimeter: the reference's API gateway is Azure-specific.
10. Workflow conventions: branching, commit style, review process, release cadence, and where plans and decision records live.

---

## 11. Competitors, benchmarking practice and no-training techniques (added in v1.1, 2026-11-16)

Gathered by web research on 2026-11-15/16. `[primary]` here means the raw primary content was read; `[summarised]` means a primary page was read only in summary form, so exact wording and figures need a second look before publication; `UNVERIFIED` means secondary source only. Terms-of-service readings are not legal advice — re-read each clause before publishing any comparison.

### 11.1 Open-source toolkits

| Toolkit (version) | Code licence | Model/weights licence | Uses a digital PDF's own text layer? | Batch command |
|---|---|---|---|---|
| Docling 2.150.1 | MIT | Apache-2.0 / CDLA-Permissive-2.0 | Yes (default backend; OCR only on bitmap content) | `docling ./pdfs --from pdf --to md --output ./out` |
| MinerU 4.2.2 | Custom "MinerU Open Source License" (Apache-2.0 plus conditions: commercial licence above 100M MAU or USD 20M monthly revenue; services must state MinerU is used) `[summarised]` | MinerU2.5-2509 tagged AGPL-3.0; MinerU2.5-Pro-2604 tagged Apache-2.0 | `flash` tier yes; other tiers rasterise but can reuse native text | `mineru-kit parse ./documents -o ./output --format zip` |
| Marker 2.2.0 / Surya 0.24.0 / Chandra 0.2.0 | Apache-2.0 | **Restricted**: modified OpenRAIL-M, free for research, personal use and startups under $5M (Marker/Surya) or $2M (Chandra) funding/revenue; Chandra may not be used to compete with Datalab's API | Yes, by mode: `no_ocr`, `fast`, `balanced` (VLM only for equations, complex tables, non-Latin scripts, scans), `vllm` (every page) | `marker /path/to/folder --output_dir out --mode balanced` |
| Unstructured 0.27.37 | Apache-2.0 | not checked | Yes (`auto` picks `fast` when text is extractable) | no documented folder-to-Markdown CLI found |
| olmOCR 0.4.27 | Apache-2.0 | Apache-2.0 (7B) | No, always renders pages | `olmocr ./workspace --markdown --pdfs ./pdfs/*.pdf` (pass `--model` explicitly) |
| PaddleOCR 3.7.0 | Apache-2.0 | Apache-2.0 (PaddleOCR-VL-1.6, PP-DocLayoutV3) | Not documented; treat as always-rasterise | per file: `paddleocr doc_parser -i "$f" --pipeline_version v1.6 --save_path out/` (directories of PDFs unsupported) |
| GLM-OCR SDK 0.1.5 | Apache-2.0 | MIT (model), Apache-2.0 (layout) | No | `glmocr parse <dir> --output ./results/` |

Published scores (all vendor- or competitor-run; not comparable across benchmarks or versions): OmniDocBench v1.6 — PaddleOCR-VL-1.6 96.34, MinerU2.5-Pro 95.75, GLM-OCR 95.22, olmOCR 85.74, Marker (old) 78.44. olmOCR-bench — Chandra 2 85.8, Marker `vllm` 83.7, olmOCR-2 82.4, GLM-OCR 75.2, MinerU 2.5.4 75.2. Datalab's own OmniParseBench ranks GLM-OCR 81.5 and PaddleOCR-VL-1.6 78.7, well below their OmniDocBench standing, because it scores output format (layout, HTML tables) as well as text.

Licence observation: PP-DocLayoutV3 + GLM-OCR or PaddleOCR-VL-1.6 is the only top-tier stack found with no revenue cap, AGPL or non-commercial clause anywhere.

### 11.2 Commercial APIs

| Service | List price per 1,000 pages | May we publish a comparison? |
|---|---|---|
| Mistral OCR 4.1 | $4 ($2 batch) | No restriction found `[summarised]` |
| Azure Document Intelligence | Read $1.50; Layout $10 `[primary]` | Yes; reciprocal clause, must supply replication information on request |
| Google Document AI | Enterprise OCR $1.50; Layout Parser $10 `[primary]` | Yes, only if the disclosure includes everything needed to replicate; reciprocal rights |
| AWS Textract | Text $1.50; Tables $15; Forms $50 | Yes, with replication information; reciprocal rights |
| LlamaParse | $1.50 (fast) to $37.50 (agentic plus) | **Needs written consent** (ToS s.3.2) |
| Reducto | $10 (r-1), $1 (mini) | **No**, if benchmarking for a competing product (Terms 5.2(c)) |
| Datalab hosted | $1 / $3 / $7.50 | **No**; benchmark results are defined as confidential |
| Unstructured hosted | $15 | No benchmark clause found; may not be used to build a competing product |
| General multimodal APIs | roughly $0.55 (Gemini 2.5 Flash-Lite) to $26 (largest models), by arithmetic at 1,500 input + 1,000 output tokens per page | Terms not fully checked |

### 11.3 Running the official scorers

- **OmniDocBench** (code Apache-2.0; dataset research-only, non-commercial; 1,651 pages; pin the exact commit, since v1.5 and v1.6 numbers are not comparable). Input: one `.md` per page, named after the page image. Run `python pdf_validation.py --config configs/end2end.yaml`, or the Docker image `ghcr.io/zeng-weijun/omnidocbench-eval:repro-ubuntu2204`. The formula metric (CDM) needs TeX Live, ImageMagick and Ghostscript. Overall = ((1 − text edit distance) × 100 + table TEDS + formula CDM) / 3; reading order is reported but excluded.
- **olmOCR-bench** (code Apache-2.0; dataset ODC-BY-1.0; 1,403 PDFs, 7,010 pass/fail tests; English only). Put Markdown under `bench_data/<tool>/`, then `python -m olmocr.bench.benchmark --dir ./olmOCR-bench/bench_data`. Row/column-span table tests need HTML tables.

### 11.4 Known pitfalls

- The top of OmniDocBench is saturated and noisy: the top seven span 95.2–97.0, and one audit reports label errors in about 12% of scored blocks (UNVERIFIED). Sub-point differences are not meaningful.
- Edit distance penalises harmless formatting differences; table scoring wants HTML; formula scoring depends on LaTeX normalisation.
- olmOCR-bench's "absent" tests reward dropping headers, footers and watermarks, so a faithful transcriber loses points.
- Both test sets are public and labs train on tens of millions of pages; contamination is unmeasured.
- Scores do not transfer: PureDocBench reports an 11.6-point average drop from clean to real-world captures; TableParseMap finds parsers above 93 TEDS on OmniDocBench reach 85 on real tables.
- Vendor and independent numbers disagree (Mistral claims 93.07; the official table lists 85.66, versions unclear).
- Good practice: pin tool, model and benchmark commit; report per-category and overall; state hardware, DPI and concurrency; publish raw outputs; add 50–100 documents of our own.

### 11.5 Gains available without training a model

| Technique | Reported effect | Shipped by |
|---|---|---|
| Digital-first hybrid parsing (use the PDF's own text; send only equations, complex tables and scans to the VLM) | Marker `balanced` vs full-VLM: 16.9 vs 4.5 pages/s for 80.5 vs 85.9 on its own benchmark (self-reported). Needs a garbled-text detector with OCR fallback | Marker (restricted weights), Docling, Unstructured, MinerU `flash` |
| Layout-then-region with per-type prompts and parallel decoding | Already the reference's design. No published result found for sending tables or formulas to a *different* specialist model than text | GLM-OCR SDK, PaddleOCR-VL, MinerU |
| Multi-model agreement ("Consensus Entropy", arXiv 2504.11101) | Training-free error detection, F1 +42% over VLM-as-judge; a voting study lifted word accuracy 96.28% → 98.27% | No mainstream toolkit found |
| Verifier/corrector on large tables (DEC, arXiv 2608.09842) | +1.57 TEDS on average, +5.66 on large tables, across three frozen parsers | Not confirmed |
| Confidence from token log-probabilities | Useful for error detection but calibration varies widely by model; no published gain for selective re-recognition in page parsing | Mistral OCR 4.1 exposes block confidence |

Fine-tuning a roughly 1B OCR model for a niche: PaddleOCR-VL's official guide cites about 2 hours on one 80 GB GPU for a Bengali example; GLM-OCR's guide says LoRA fits in 8 GB VRAM; one write-up reports a 1B model fine-tuned on about 700 samples beating a commercial API on Sinhala (UNVERIFIED). Expect a few hundred to a few thousand labelled pages and single-digit GPU-hours.

### 11.6 Where a solo pipeline can and cannot win

Plausible: licence cleanliness; cost against mid and high API tiers at sustained volume; a permissively licensed digital-first hybrid; one narrow niche via fine-tuning; targeted long-tail fixes; full benchmark transparency (which several vendors' terms prevent others from offering).

Unrealistic: topping OmniDocBench overall (the ceiling is the base model, and leaders differ by less than label noise); matching frontier general models on photographed or degraded documents; out-running purpose-built serving stacks on raw throughput; beating the cheapest API tiers (about $0.55–$1 per 1,000 pages) on price alone; broad multilingual coverage, compliance and support.

---

## 12. Platform and engine facts (added in v1.2, 2026-11-16)

Facts gathered while settling the design that later milestones depend on. Same evidence caveats as section 11; anything marked "to verify" must be checked before a task relies on it.

### 12.1 Modal

| GPU | Per second | Per hour |
|---|---|---|
| T4 16GB | $0.000164 | $0.59 |
| L4 24GB | $0.000222 | $0.80 |
| A10 24GB | $0.000306 | $1.10 |
| L40S 48GB | $0.000542 | $1.95 |
| A100 40GB | $0.000583 | $2.10 |
| A100 80GB | $0.000694 | $2.50 |
| H100 | $0.001097 | $3.95 |

- These are base rates for preemptible containers. Non-preemptible is 3×; pinning a region is 1.15–1.75×.
- CPU $0.0000131 per core-second (about $0.047 per core-hour); memory $0.00000222 per GiB-second.
- Starter plan: $0 base with $30 of free compute per month. Volumes: $0.09 per GiB-month with 1 TiB free. Workspace budgets and spend limits exist; environment-level budgets need the Team plan.
- **Web endpoints:** `@modal.web_server(port)` serves any process that speaks HTTP on a port (it must bind `0.0.0.0`). Request bodies up to 4 GiB. Proxy authentication uses `Modal-Key` and `Modal-Secret` headers. Default rate limit is 200 requests per second per function. Request timeout limits: to verify.
- **Autoscaling:** idle containers stop after `scaledown_window` (default 60 s, settable from 2 s to 20 min). `min_containers` keeps containers warm and therefore must stay at zero for GPU functions (decision D16). Memory and GPU snapshots can cut cold starts sharply (Modal reported about two minutes down to about ten seconds for some models, with GPU snapshots in alpha in December 2025): current status to verify.
- **`modal.Queue` is not a durable job queue:** items up to 1 MiB, 5,000 items and 64 MiB per partition, a partition is cleared 24 hours after its last write, and `get()` removes an item with no acknowledgement or redelivery. Modal's docs say not to rely on it for persistent storage. The Modal target therefore builds on Modal's function-call mechanism (spawn a call, poll its result) rather than on `modal.Queue`; the retention period for results is to verify in milestone 6.
- Modal publishes worked examples for an OpenAI-compatible vLLM server and for SGLang, including a vision-language example.

### 12.2 Engines and recognisers

- **vLLM versions:** the reference pins v0.21.0 (2026-05-15). v0.33.0 (2026-11-02) is current and adds a batched encoder for PaddleOCR-VL.
- **Multi-token prediction** is speculative decoding driven by the model's own prediction head: `--speculative-config '{"method":"mtp","num_speculative_tokens":N}'`. It is off unless that flag is passed. Supported for GLM-OCR from v0.16.0 and Qwen 3.5 from v0.17.0. vLLM's Qwen 3.5 recipe warns that it lowers per-token latency but reduces throughput under high concurrency. One user report on the GLM-OCR tracker measured 1.53× on the eager path and 0.62× with CUDA graphs (unverified).
- **Throughput-relevant vLLM settings:** the A100-specific default for `--max-num-batched-tokens` is 2048. OCR recipes disable the multimodal processor cache and prefix caching (`--mm-processor-cache-gb 0 --no-enable-prefix-caching`) because every page is unique. `--api-server-count N` parallelises request preprocessing.
- **vLLM metrics available for autoscaling and dashboards (v0.21.0):** `vllm:num_requests_running`, `vllm:num_requests_waiting`, `vllm:kv_cache_usage_perc`, `vllm:prompt_tokens`, `vllm:generation_tokens`, `vllm:request_success`, and the histograms `vllm:time_to_first_token_seconds`, `vllm:inter_token_latency_seconds`, `vllm:e2e_request_latency_seconds`, `vllm:request_queue_time_seconds`, `vllm:request_prefill_time_seconds`, `vllm:request_decode_time_seconds`. With speculative decoding: `vllm:spec_decode_num_draft_tokens`, `vllm:spec_decode_num_accepted_tokens`.
- **Per-recogniser notes:**
  - Qwen 3.5 4B: needs vLLM ≥ 0.17.0. Thinking is on by default and is disabled with `--default-chat-template-kwargs '{"enable_thinking": false}'`. One image token per 32×32 pixels, 64 to 16,384 tokens per image. Its card recommends temperature 0.7, top-p 0.8, top-k 20 without thinking; the reference instead sends greedy decoding with a repetition penalty of 1.1.
  - GLM-OCR: vLLM ≥ 0.16.0 (its SDK README says ≥ 0.19.0 with multi-token prediction). The card says 0.9B parameters; the weights file totals 1.33B. Its published 1.86 pages per second was measured at one replica and concurrency one.
  - PaddleOCR-VL-1.6: vLLM ≥ 0.11.1; its docs require GPU compute capability ≥ 8.0 and CUDA ≥ 12.6 for the vLLM backend and call the T4 "not recommended".
  - OvisOCR2: its card pins `vllm==0.22.1`.
  - LightOnOCR-3: needs `--trust-remote-code`, with thinking disabled.
- **SGLang:** documents GLM-OCR (a cookbook page, with its own speculative-decoding option from v0.5.10) and PaddleOCR-VL (0.9B, 1.5, 1.6). Support for MiniCPM-OCR, OvisOCR2 and Qwen 3.5 4B was not confirmed.
- **T4:** has no bfloat16 support, so it suits layout detection but not recognition.

### 12.3 Working without a GPU

- `llm-d-inference-sim` v0.9.0 (Apache-2.0, image `ghcr.io/llm-d/llm-d-inference-sim:v0.9.0`): an OpenAI-compatible server over HTTP and gRPC with `--mode echo` or `--mode random`, simulated time-to-first-token and inter-token latency that rise under load, and error injection. It exposes "a subset of standard vLLM Prometheus metrics"; which ones, and whether it accepts image inputs, are to verify in milestone 2.
- `fake-gpu-operator` advertises fake `nvidia.com/gpu` resources on a kind cluster, so GPU scheduling, taints and tolerations can be tested.
- PP-DocLayoutV3 (33M parameters, about 133 MB) runs on CPU.

### 12.4 Cost model

Cost per 1,000 pages = GPU price per hour ÷ (3.6 × pages per second).

Planning estimate for the whole project on Modal, before free credits: about $80–130. A full pass over both public benchmarks (about 3,000 pages) costs roughly $0.70–2.30 per recogniser-and-engine combination on an L4, assuming 0.3–1 page per second. The largest and least predictable item is GPU time spent getting each recogniser to start and debugging it, estimated at 15–30 GPU-hours. A container left warm is the main risk: an A100 80GB for a day is $60.

---

## 13. Sources

- The reference repository at `3271469`, plus `origin/restructure-repo` (`a9d1c3d`) and `origin/feat/local_dpl` (`02a27e1`).
- `glmocr==0.1.4` wheel source (PyPI) and `zai-org/GLM-OCR` on GitHub.
- vLLM v0.21.0 source and docs; vLLM recipes for Qwen3.5 and GLM-OCR; vLLM release notes.
- Hugging Face model cards: `Qwen/Qwen3.5-4B`, `PaddlePaddle/PP-DocLayoutV3_safetensors`, `zai-org/GLM-OCR`, and those listed in section 6.1.
- GLM-OCR technical report, arXiv 2603.10910.
- The Neural Maze course articles (several are partly paywalled; only visible text was used).
- OmniDocBench and olmOCR-bench leaderboards; provider pricing pages as of 2026-11-12 to 2026-11-15.
