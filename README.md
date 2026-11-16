# slm-ocr-pipeline

A self-hosted service that turns PDFs and images into Markdown and structured elements, using small open-weight models. Built to be measured: every quality, speed and cost figure it reports will be reproducible.

**Status:** under construction. There is nothing to run yet. The [milestone plan](plans/02-milestones.md) shows what is being built and in what order.

## Project documents

The plans, decisions and working rules are kept in the repository.

| Document | What it holds |
|---|---|
| [Milestone plan](plans/02-milestones.md) | The thirteen milestones: their order, scope and the evidence that ends each one |
| [Milestone 1 task breakdown](plans/milestone-01-foundation.md) | The current milestone's tasks and their acceptance criteria |
| [Settled decisions](plans/01-decisions.md) | Every design decision agreed so far, by ID |
| [Decision records](docs/adr/) | Why the surprising decisions were made |
| [Reference analysis](plans/00-reference-analysis.md) | A study of the reference (the course repository credited below): what it does, its defects by ID, and the recognisers, benchmarks and licences around it |
| [Glossary](GLOSSARY.md) | The project's vocabulary |
| [Contributing](CONTRIBUTING.md) | How a change reaches `main` |
| [`CLAUDE.md`](CLAUDE.md) | The same workflow as instructions for AI coding agents |

## Credit

This project is an independent, from-scratch re-implementation of the system taught in the [Production-Grade SLM-Powered OCR Course](https://github.com/neural-maze/production-ocr-course) by The Neural Maze (Miguel Otero Pedrido and Antonio Zarauz Moreno). It follows that course's architecture and learning path. Where it departs from the course's design, the reason is recorded.

## Licence

Apache-2.0. See [`LICENSE`](LICENSE) and [`NOTICE`](NOTICE).
