# Two deployment targets behind one contract

The reference runs everything on Kubernetes with GPU node pools. We cannot afford standing GPU clusters, and the only real-GPU venue we have is Modal, which is serverless and has no node pools, KEDA or shared volumes. We therefore ship one portable pipeline with two targets that satisfy the same contract: a Kubernetes target (Rust API, Redis, worker, KEDA) proven in CI on kind with a simulator and fake GPUs, and a Modal target (a small Python app on Modal's own job queue and autoscaler) that is the only place real GPUs run.

## Considered options

- **Modal only.** Cheapest and simplest, but drops Kubernetes autoscaling, which is the reference's core subject.
- **Kubernetes only, on a hyperscaler.** Faithful to the reference, but every experiment costs cluster-hours and needs GPU quota we do not have.
- **One code path on both, with Redis-driven workers on Modal.** A worker polling Redis must stay running, which defeats scale-to-zero on Modal.

## Consequences

- Queueing and scaling are deliberately different per target; only the contract and the pipeline are shared. One contract test suite runs against both.
- The Kubernetes target is never exercised with a real recogniser unless the optional hyperscaler run happens. Its claims are limited to scheduling, scaling and delivery behaviour.
