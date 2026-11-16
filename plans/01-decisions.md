# Settled decisions

| | |
|---|---|
| Version | v1.1 — 2026-11-16 (adds D33) |
| Source | Three rounds of design review with the project owner, 2026-11-15 to 2026-11-16 |
| Status | Every row is agreed. Changing one is the owner's call: raise it, then update this file and bump the version. |

Terms in **bold** are defined in `GLOSSARY.md`. Defect IDs (R1, A2, …) refer to `plans/00-reference-analysis.md`, section 5.

## Identity and scope

| ID | Decision |
|---|---|
| D1 | A new repository, `slm-ocr-pipeline`, public under `sourav-dhar` from the first commit. Apache-2.0. The README and `NOTICE` credit the **reference** course. The reference clone stays read-only. |
| D2 | Follow the reference's architecture and incremental path, but build each piece correctly the first time. Each deliberate deviation is recorded. A few performance settings are reproduced first only so a before/after can be shown. |
| D3 | Headline claim and what we do not chase: see `docs/adr/0004-compete-on-accuracy-versus-cost.md`. |
| D4 | Licences and comparison terms: see `docs/adr/0005-licence-and-terms-constraints.md`. |

## Architecture

| ID | Decision |
|---|---|
| D5 | Two **targets** behind one **contract**: see `docs/adr/0001-two-targets-one-contract.md`. |
| D6 | We own the **pipeline**; the GLM-OCR SDK is a **baseline** only: see `docs/adr/0002-own-pipeline-not-glm-ocr-sdk.md`. |
| D7 | **Job** delivery on Kubernetes: see `docs/adr/0003-redis-streams-for-job-delivery.md`. Fixes R1, R2. |
| D8 | **Document** bytes sit behind a storage interface. First backend: raw binary in Redis with a 10 MB cap and an expiry on every key; reading a job's status never touches the payload. An S3-compatible backend comes later. Fixes R3, R5. |
| D9 | **Page triage** is a pipeline step from the first version. The path that reads **digital pages** directly is built right after baseline numbers exist. |
| D10 | Output is Markdown plus versioned **element**-level JSON: type, page, bounding box, reading order, text, and confidence where the **recogniser** provides it. |
| D11 | Languages: Rust for the Kubernetes target's API, Python for the pipeline, worker, Modal target and tooling. |

## Recognisers and engines

| ID | Decision |
|---|---|
| D12 | Tier 1, measured in full: Qwen 3.5 4B (the reference's choice, as **baseline**), GLM-OCR, PaddleOCR-VL-1.6, OvisOCR2. |
| D13 | Tier 2, only if budget remains: MiniCPM-OCR 1.0, LightOnOCR-3 0.8B, Unlimited-OCR-2, Qwen 3.5 2B. Plus one classical CPU method as a floor. |
| D14 | **Engines**: vLLM for every recogniser; SGLang additionally for GLM-OCR and PaddleOCR-VL-1.6 only. |
| D15 | GPU-free development and CI use a **simulator** in place of an engine. |

## Money

| ID | Decision |
|---|---|
| D16 | Modal is the only GPU cloud. Workspace budget $50 per month. No GPU container is ever kept warm. Expected total for the project: roughly $80–130 before free credits. |
| D17 | CI runs everything GPU-free automatically. GPU **benchmarks** run only on a manual trigger behind a budget check. Modal deploys happen only on release tags, with the owner's approval. |

## Evaluation

| ID | Decision |
|---|---|
| D18 | **Benchmarks**: OmniDocBench v1.6 and olmOCR-bench with their official scorers, plus 50–100 permissively licensed documents of our own that include **digital pages**. The harness is trusted only after it reproduces one published score. |
| D19 | Open-source comparison on the same GPU: Docling, MinerU, Marker, olmOCR, the PaddleOCR-VL pipeline, the GLM-OCR SDK. |
| D20 | Hosted comparison: Mistral OCR, Azure Document Intelligence, Google Document AI and AWS Textract on a 500-page subset, capped at $25. Each provider's terms are re-read before publishing. |
| D21 | Techniques that need no training are chosen after baseline results show where errors are. No model training in the first version. |

## Operations

| ID | Decision |
|---|---|
| D22 | Kubernetes perimeter: Envoy Gateway (Gateway API) for API keys and rate limiting, with default-deny network policies inside the cluster. Modal uses its built-in proxy authentication. Fixes S2, S4. |
| D23 | Agent access: one local (stdio) MCP tool, `parse_document`, confined to a workspace folder and able to point at either **target**. |
| D24 | Observability in the first version: Prometheus metrics per pipeline stage, Grafana dashboards kept in the repository, structured logs carrying the job ID. Fixes A1. |

## Process

| ID | Decision |
|---|---|
| D25 | Protected `main`; short-lived branches; every change through a pull request with required CI; Conventional Commits; squash merge. |
| D26 | Review happens in a different working session from the one that wrote the code, posted as inline pull-request comments. The owner merges. |
| D27 | Commits and pull requests carry no tool-attribution lines. |
| D28 | Work is tracked as GitHub issues grouped into milestones. Plans live in `plans/`, decision records in `docs/adr/`. |
| D29 | A milestone is done when tests are green, docs are updated, every deviation from the reference is recorded, evidence is captured, and a release is tagged. |
| D30 | Tooling: Python 3.12 with uv, ruff, pyright, pytest; Rust with clippy and rustfmt; `just` as the task runner. |
| D31 | Every README visual is generated from source kept in the repository. A GitHub Pages site (MkDocs Material) hosts a benchmark page generated from result files. |
| D32 | Verified defects are reported to the reference repository with reproductions. Each report is drafted for the owner's approval before posting. |
| D33 | `CLAUDE.md` is committed to the public repository alongside the plans, glossary and decision records. Machine-specific notes stay in the untracked `CLAUDE.local.md`. |

## Deliberately left open

| Item | Decide when |
|---|---|
| A niche fine-tune (which language, script or document type, if any) | After milestone 10 shows where recognisers are weak |
| A real-GPU Kubernetes run on a hyperscaler | After milestone 10, if budget remains |
| Distributed tracing | After milestone 9 |
| When the S3-compatible storage backend lands | When a job needs more than 10 MB |
| Which training-free techniques to build beyond the digital-page path | After milestone 10 |
