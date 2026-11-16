# Contributing

This page describes how a change reaches `main`. The same path applies to every change, whoever writes it. Decisions D25–D29 in [`plans/01-decisions.md`](plans/01-decisions.md) are the settled core of this workflow, and that file wins wherever this page disagrees with it. The step-level detail (branch names, what a description holds, how a review is answered) mirrors [`CLAUDE.md`](CLAUDE.md). A change to the workflow updates both files in the same pull request.

Some of the checks and repository settings named here are still being put in place during milestone 1 (see the [task breakdown](plans/milestone-01-foundation.md)). Until a check exists, the pull request says how the change was verified by hand. Until the repository allows only squash merges and takes the commit message from the pull request, the owner chooses "Squash and merge" and sets the commit title and body by hand.

## The path of a change

### 1. Start from an issue

Work is tracked as GitHub issues grouped into milestones. One issue is one task: a single pull request, sized for one working session, with its acceptance criteria listed under "Accept when".

The [milestone plan](plans/02-milestones.md) gives the order of the milestones. The current milestone's tasks are in its breakdown in [`plans/`](plans/). You are ready to start a task when you can restate its acceptance criteria.

### 2. Branch

`main` is protected, so nothing is pushed to it directly. Branch from an up-to-date `main` and name the branch `<type>/<issue>-<slug>`, for example `feat/3-rust-workspace`. A branch lives for one task.

### 3. Make the change

Write the test first wherever the change can be tested. The change is ready for a pull request when:

- every acceptance criterion is backed by a test or by evidence you captured, such as command output or a link to a CI run; and
- `just check` passes locally.

### 4. Open a pull request

The title is a [Conventional Commit](https://www.conventionalcommits.org/en/v1.0.0/): `<type>(<optional scope>): <summary>`, for example `feat(api): serve GET /healthz`. Common types are `feat`, `fix`, `docs`, `test`, `refactor`, `perf`, `build`, `ci` and `chore`. Pull requests are squash-merged, so the title becomes the one commit that lands on `main`. Commit messages on the branch are read only by the reviewer.

The description says:

- what changed;
- why;
- how each acceptance criterion was verified; and
- which issue it closes, as `Closes #N`.

Commit messages and pull-request descriptions carry only engineering content. Leave out lines that credit a tool, such as `Co-Authored-By` trailers for an assistant or "Generated with" footers.

CI must pass before the pull request can merge.

### 5. Review

Review happens in a different working session from the one that wrote the change. A working session is one sitting at the code, by a person or by an AI coding agent. The session that wrote a change does not review it.

The reviewer reads the diff cold, against the task's acceptance criteria and the [settled decisions](plans/01-decisions.md). Each finding is posted as an inline comment on the pull request, followed by one summary comment with a verdict. The reviewer does not change the code.

### 6. Answer the review

Every review comment gets an answer. Either change the code and reply with what changed, or reply with why not. This step is finished when every comment has an answer and CI is green.

### 7. Merge

The owner merges, by squash.

## What "done" means

A **task** is done when its pull request is squash-merged and its issue is closed. That requires every acceptance criterion to be backed by a test or by captured evidence, the checks to pass, and every review comment to have an answer.

A **milestone** is done when all of the following are true:

- its tests are green;
- the documentation is updated;
- every deviation from the reference is recorded;
- its exit evidence, listed in the [milestone plan](plans/02-milestones.md), is captured; and
- a release is tagged.

## Where things are written down

- **GitHub issues and milestones** are the live record of what is open and what is done.
- **[`plans/`](plans/)** holds the reference analysis, the settled decisions, the milestone plan and the current milestone's task breakdown.
- **[`docs/adr/`](docs/adr/)** holds the decision records. A new choice that is hard to reverse, surprising and a real trade-off gets a record in the same pull request. Changing a settled decision is the owner's call, so raise it before building around it.
- **[`GLOSSARY.md`](GLOSSARY.md)** defines the project's vocabulary. Use its terms in code, documentation and pull requests. In particular, a *job* is the runtime unit and a *task* is a unit of development work.
- **[`CLAUDE.md`](CLAUDE.md)** gives this same workflow as instructions for AI coding agents. Its Rules section also holds project rules that apply to every contributor: material adapted from another project is recorded in [`NOTICE`](NOTICE), only measured figures are published, and GPU spend and outward actions wait for the owner; [`docs/adr/0005-licence-and-terms-constraints.md`](docs/adr/0005-licence-and-terms-constraints.md) adds a licence check for each new dependency.
