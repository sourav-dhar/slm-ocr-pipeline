# Milestone 1: Foundation — task breakdown

| | |
|---|---|
| Version | v1 — 2026-11-16 |
| Goal | A repository where nothing can reach `main` without passing checks |
| Exit evidence | CI green on `main`; a pull request with a failing check cannot merge; `just check` passes locally; `v0.1.0` tagged |
| Real GPU | None |

Tasks run in order. Each is one GitHub issue and one pull request, except 1.1, which creates the repository. Decision IDs (D25, …) refer to `plans/01-decisions.md`.

| Task | Issue | Status |
|---|---|---|
| 1.1 Create the repository | — | Done 2026-11-16 (commit `ce59428`) |
| 1.2 Founding documents | #1 | Open |
| 1.3 Python workspace | #2 | Open |
| 1.4 Rust workspace | #3 | Open |
| 1.5 Continuous integration | #4 | Open |
| 1.6 Hygiene and automation | #5 | Open |
| 1.7 Protection, tracking and the first release | #6 | Open |

GitHub issues are the live record of status; this table is a convenience and may lag.

## Before any task: tools on the development machine

Installed by the owner, once. No pull request.

- Rust via `rustup` (stable toolchain, with `rustfmt` and `clippy`). On Windows the installer offers to add the Visual Studio C++ build tools it needs; accept.
- `just` (task runner).
- `pre-commit`, installed with `uv tool install pre-commit`.

Done when `cargo --version`, `just --version` and `pre-commit --version` each print a version.

---

## 1.1 Create the repository

The only change ever pushed straight to `main`.

**Do**
- Initialise git with `main` as the default branch.
- Add `LICENSE` (Apache-2.0, full text), `NOTICE` (credits the reference course by name, URL and pinned commit `3271469`, and states this is an independent re-implementation), a short `README.md` (name, one paragraph, "under construction", the same credit), `.gitignore` (Python, Rust, editors, `.env`, local-only notes), `.gitattributes` (`* text=auto eol=lf`), `.editorconfig`.
- Create the public GitHub repository `sourav-dhar/slm-ocr-pipeline` with a description and topics, and push.
- Turn on "require a pull request before merging" for `main`.
- Create the GitHub milestone "1 Foundation" and one issue per task 1.2–1.7, each with that task's acceptance criteria.

**Stop and ask** before creating the public repository: show the owner the exact files and the repository description first.

**Accept when**
- The repository is public and GitHub detects the licence as Apache-2.0.
- `main` has exactly one commit.
- A direct push to `main` is rejected.
- Issues for 1.2–1.7 exist under the milestone.

## 1.2 Add the project's founding documents

**Do**
- Add `plans/` (reference analysis, decisions, milestone plan, this file), `GLOSSARY.md`, `CLAUDE.md` and `docs/adr/0001`–`0005`.
- Add `CONTRIBUTING.md` describing, for a human reader, the workflow in D25–D29: branching, commit style, review, what "done" means.
- Link them from the README.

**Accept when**
- Every relative link in the added documents resolves.
- The pull request is squash-merged and closes its issue.

## 1.3 Python workspace

**Do**
- A `uv` workspace at the root with one member, `core/`, holding the package `slm_ocr` in a `src` layout. Python pinned to 3.12.
- One small typed function with a test, so every tool has something real to check.
- `ruff` for lint and format, `pyright` in strict mode for `core/`, `pytest` with coverage reporting.
- A `justfile` with `fmt`, `lint`, `typecheck`, `test`, and `check` (runs the other three without modifying files).

**Accept when**
- `just check` passes on a clean checkout after `uv sync`.
- `uv.lock` is committed.
- Introducing a type error or a lint violation makes `just check` fail (state what you tried in the pull request).

## 1.4 Rust workspace

**Do**
- A Cargo workspace at the root with one member, `services/api`, a binary crate that serves `GET /healthz` with Axum (current stable release).
- One test that calls the router in-process and asserts 200.
- `rust-toolchain.toml` pinning the stable version with `rustfmt` and `clippy`. Clippy runs with warnings denied.
- Extend the `justfile` so `fmt`, `lint`, `test` and `check` cover Rust as well as Python.

**Accept when**
- `just check` passes and now exercises both languages.
- `Cargo.lock` is committed.
- A clippy warning makes `just check` fail (state what you tried).

## 1.5 Continuous integration

**Do**
- One workflow, triggered on pull requests and pushes to `main`, with a Python job and a Rust job that run the same commands as `just check`.
- Every third-party action pinned to a commit SHA. Workflow permissions default to read-only. In-progress runs for the same branch are cancelled by a newer push. Dependency caches for `uv` and Cargo.
- A final job named `ci-ok` that succeeds only if every other job succeeded, so branch protection needs to require just one check.

**Accept when**
- The workflow is green on the pull request.
- A deliberately failing commit turns it red, and the fix turns it green (link both runs in the pull request).

## 1.6 Hygiene and automation

**Do**
- `pre-commit` configuration: ruff, ruff-format, `cargo fmt`, secret scanning (gitleaks), whitespace and end-of-file fixers, YAML check, workflow lint (actionlint). A CI job runs the same hooks.
- Dependabot for GitHub Actions, Cargo and Python, weekly and grouped.
- A pull-request template (what changed, why, how it was verified, issue closed) and issue templates for a task, a bug and an experiment.
- A check that the pull-request title is a valid Conventional Commit, because squash merging turns the title into the commit message.

**Accept when**
- `pre-commit run --all-files` is clean.
- A pull request with a non-conforming title fails the title check (link the run).
- A test secret committed on a throwaway branch is caught by the secret scan (do not push a real one).

## 1.7 Protection, tracking and the first release

**Do**
- A ruleset on `main`: pull request required; `ci-ok` and the title check required; conversations must be resolved before merging; squash merge only; linear history; no force pushes or deletion. Required approvals stay at zero, because GitHub does not let an author approve their own pull request.
- Repository settings: squash merge only, squash message taken from the pull-request title and description, branches deleted on merge.
- Labels for type and area. All thirteen milestones created with their goal from `plans/02-milestones.md`. One issue under milestone 2: "Write the milestone 2 task breakdown".
- Release automation that builds the changelog from Conventional Commits and tags a release.
- Tag `v0.1.0`.

**Accept when**
- A pull request with a failing check, or an unresolved review comment, cannot be merged (show the blocked state).
- The `v0.1.0` release exists with generated notes.
- Milestones 1–13 exist on GitHub and milestone 1 has no open issues.
