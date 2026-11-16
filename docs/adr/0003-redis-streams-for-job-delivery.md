# Redis Streams with consumer groups for job delivery on Kubernetes

The reference pushes job IDs onto a Redis list and pops them, so a worker that crashes or is scaled down mid-job loses that job for good. On the Kubernetes target we use Redis Streams with a consumer group: a job is acknowledged only when it finishes, jobs held by a dead worker are reclaimed, and a job that keeps failing moves to a dead-letter stream.

## Considered options

- **Redis list with a processing list (`LMOVE`).** Fixes loss but needs hand-built reclaim and retry counting, which Streams provide.
- **A dedicated broker (NATS, RabbitMQ, a cloud queue).** Stronger guarantees, but adds a component the reference does not have and that must also run in CI on kind.

## Consequences

- Delivery is at-least-once, so a worker may see a job twice; processing a job must be safe to repeat.
- Autoscaling reads consumer-group lag rather than list length.
