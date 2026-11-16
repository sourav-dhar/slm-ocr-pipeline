# Milestone plan

| | |
|---|---|
| Version | v1 — 2026-11-16 |
| Order agreed | 2026-11-16 (decision D29 defines "done") |

The order keeps GPU spend late and produces measured results early. Each milestone is a shippable increment and ends with a tagged release.

**How milestones are broken down.** Only the current milestone has a task breakdown, in `plans/milestone-NN-<slug>.md`. The first task of every milestone is to write that breakdown and get the owner's sign-off, so that it reflects what the previous milestones actually taught us. Each task is one GitHub issue and one pull request, sized for a single working session.

Terms in **bold** are defined in `GLOSSARY.md`. "Ref. week" is the reference course week a milestone corresponds to. Defect IDs refer to `plans/00-reference-analysis.md`, section 5.

| # | Milestone | Ref. week | Real GPU |
|---|---|---|---|
| 1 | Foundation | — | No |
| 2 | Contracts | — | No |
| 3 | Pipeline | 2 | No |
| 4 | Inference on Modal | 3 | Yes, about $5–10 |
| 5 | Evaluation harness | — | Yes, about $10–20 |
| 6 | Modal target | 5 | About $2 |
| 7 | Rust API | 4 | No |
| 8 | Worker | 5 | No |
| 9 | Kubernetes target and autoscaling | 1, 5 | No |
| 10 | Benchmark campaign | — | Yes, about $40–60 |
| 11 | Efficiency and accuracy experiments | 3 | Yes, about $15–30 |
| 12 | Gateway and MCP tool | 6 | No |
| 13 | Showcase | — | No |

---

## 1. Foundation

**Goal.** A repository where nothing can reach `main` without passing checks.

**Scope.** Licence and credits; directory layout; Python and Rust workspaces with one trivial tested unit each; `just` recipes; CI for lint, type-check and tests in both languages; pre-commit, secret scanning, dependency updates; pull-request and issue templates; branch protection; release automation.

**Exit evidence.** CI green on `main`; a pull request with a failing check cannot merge; `just check` passes locally; `v0.1.0` tagged.

**Fixes.** Q1, Q2 (the reference has no tests, CI or lint).

## 2. Contracts

**Goal.** The interfaces everything else is built against, fixed and tested before any implementation exists.

**Scope.** The HTTP **contract** as an OpenAPI document (submit a **document**, read a **job**); the output schema for **elements** as JSON Schema, versioned; a typed client for an OpenAI-compatible **engine**; the **simulator** wired into tests; a contract test suite that can be pointed at any base URL.

**Exit evidence.** The contract test suite runs in CI against a minimal in-memory implementation; the engine client is tested against the simulator, including error and timeout paths.

**To verify first.** Whether `llm-d-inference-sim` exposes the vLLM queue metrics autoscaling needs and accepts image inputs. If not, write a small simulator of our own.

## 3. Pipeline

**Goal.** A document becomes Markdown and elements, with no GPU involved.

**Scope.** Rasterising; **page triage** as a step (initially sending every page onward); **layout detection** with PP-DocLayoutV3 running on CPU; cropping **regions**; concurrent **recognition** through the engine client; assembly into elements and Markdown; per-region failure reporting; a command-line entry point. Both **two-stage** and **single-stage** shapes are supported by the same interface.

**Exit evidence.** `parse <file>` produces schema-valid output against the simulator; a failed region is reported as failed, never as empty success; golden-file tests cover assembly.

**Fixes.** R7, R8, R9, P1, P2, P4, S3.

## 4. Inference on Modal

**Goal.** Real recognisers behind real engines, at zero cost when idle.

**Scope.** vLLM serving GLM-OCR and Qwen 3.5 4B on Modal; SGLang serving GLM-OCR; weights on a Modal Volume, downloaded once at pinned revisions with checksums; scale to zero; proxy authentication; a budget guard that refuses to start when the month's spend is near the cap.

**Exit evidence.** The pipeline's command line returns real Markdown through Modal; a recorded cold-start timeline (container start, weights load, first token) for each engine.

**Fixes.** S1, S6, P5.

## 5. Evaluation harness

**Goal.** Numbers we can defend.

**Scope.** Runners for OmniDocBench and olmOCR-bench that call their official scorers; a throughput and cost runner (pages per second, latency percentiles, **cost per 1,000 pages**); result files in a stable format; chart generation from result files; the docs site skeleton with a benchmark page.

**Exit evidence.** The harness reproduces one published score for the GLM-OCR SDK **baseline** within a stated tolerance; our pipeline with the same recogniser matches that baseline's quality; the first accuracy-versus-cost chart is generated from committed result files.

**Fixes.** Q5.

## 6. Modal target

**Goal.** The whole submit-and-poll service, deployable with one command.

**Scope.** A small Python application exposing the contract on Modal, using Modal's function calls for job delivery (spawn a call, then poll its result) and Modal's autoscaler; result storage with expiry; proxy authentication.

**Exit evidence.** The contract test suite passes against the deployed Modal target; cold start measured end to end.

## 7. Rust API

**Goal.** The Kubernetes target's front door.

**Scope.** An Axum service implementing the contract; Redis Streams for **job** delivery; the storage interface with its Redis backend; pooled connections; input validation by file signature, size and page count; health checks that reflect Redis.

**Exit evidence.** The contract test suite passes locally against the API with a stand-in worker; a status read is shown not to touch the payload.

**Fixes.** R3, R4 (in part), R5, R6, R7.

## 8. Worker

**Goal.** No job is ever lost.

**Scope.** A Python service that consumes the stream through a consumer group, runs the pipeline, acknowledges on completion, reclaims jobs from dead workers, dead-letters after repeated failure, isolates failures per job, and shuts down gracefully.

**Exit evidence.** A test that kills a worker mid-job and shows the job completing on another worker; a poisoned job reaches the dead-letter stream without blocking others.

**Fixes.** R1, R2, R10.

## 9. Kubernetes target and autoscaling

**Goal.** Scale from zero and back, proven without a GPU.

**Scope.** Kustomize base and overlays; KEDA scaling the worker on consumer-group lag and the engine on a signal that exists at zero replicas; metrics scraping configured; fake GPUs on kind so scheduling, taints and tolerations are exercised; the simulator as engine; non-root containers, probes, pinned images, Redis with persistence and authentication.

**Exit evidence.** A CI job on kind that submits jobs to an idle cluster and asserts replicas go 0 → N → 0 with every job completed; dashboards committed.

**Fixes.** A1–A5, R4, S2 (in part), S6.

## 10. Benchmark campaign

**Goal.** The accuracy-versus-cost chart.

**Scope.** Tier-1 recognisers on vLLM; GLM-OCR and PaddleOCR-VL-1.6 on SGLang as well; the six open-source toolkits on the same GPU; the four hosted services on the 500-page subset; our own document set. Tier 2 if budget remains.

**Exit evidence.** A published benchmark page where every figure links to its result file, configuration and command; provider terms re-read and recorded before publication.

## 11. Efficiency and accuracy experiments

**Goal.** Move our point on the chart, and show how far.

**Scope.** The digital-page path; serving experiments reproduced from the reference's claims (speculative decoding, token budgets, concurrency), each with a before and after; then the training-free techniques chosen from milestone 10's error breakdown.

**Exit evidence.** One short write-up per experiment with its measured effect, including experiments that did not help.

**Fixes.** P3, P5.

## 12. Gateway and MCP tool

**Goal.** A safe perimeter and an agent-facing interface.

**Scope.** Envoy Gateway with API keys and rate limiting; default-deny network policies; the stdio MCP tool `parse_document`, confined to a workspace folder, with size and type checks and a timeout that tolerates a cold start.

**Exit evidence.** CI shows unauthenticated and over-limit calls rejected; an agent parses a local file through the tool against each target.

**Fixes.** S2, S4, S5.

## 13. Showcase

**Goal.** A front page that earns a second look.

**Scope.** The README with animated diagrams and charts generated from source; the docs site completed; defect reports to the reference repository; `v1.0.0`.

**Exit evidence.** Every figure in the README traces to a result file; every visual rebuilds from a command.

---

## Optional tracks

Decided after milestone 10: a niche fine-tune, and a real-GPU run of the Kubernetes target on a hyperscaler.
