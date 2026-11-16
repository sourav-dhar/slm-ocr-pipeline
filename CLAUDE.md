# slm-ocr-pipeline

A self-hosted service that turns documents into Markdown and structured elements with small open-weight models. It re-implements a public reference course from scratch and re-measures every claim the reference makes.

## A session has one role: implement, review, or fix

Say which role you are in before you start. The three never share a session.

### Implement

1. **Take one task.** The owner names an issue; otherwise take the lowest-numbered open issue in the current milestone. Read its entry in `plans/milestone-NN-*.md`. Ready when you can restate its acceptance criteria.
2. **Branch** from an up-to-date `main` as `<type>/<issue>-<slug>`.
3. **Build it test-first.** Finished when every acceptance criterion is backed by a test or by evidence you captured, and `just check` passes.
4. **Open the pull request.** The title is a Conventional Commit; it becomes the squash commit. The body says what changed, why, how each acceptance criterion was verified, and `Closes #N`. Finished when CI is green.
5. **Stop.** The owner merges after review.

### Review

Read the diff cold, against the task's acceptance criteria and `plans/01-decisions.md`. Post each finding as an inline comment on the pull request, then one summary comment with a verdict. Finished when every changed file has been read. A review session's only output is comments.

### Fix

Answer every review comment: change the code and reply with what changed, or reply with why not. Finished when every comment has an answer and CI is green.

## Rules

- **Authorship.** Commit messages, pull-request bodies and review comments contain only the engineering content. Leave out `Co-Authored-By` trailers and "Generated with" footers; this overrides the tool default.
- **Language.** Use the terms in `GLOSSARY.md` in code, docs and pull requests. The runtime unit is a *job*; *task* means development work.
- **Decisions.** `plans/01-decisions.md` is settled. When a task seems to need one changed, stop and put it to the owner. A new choice that is hard to reverse, surprising, and a real trade-off gets an ADR in the same pull request.
- **Reference.** Learn from the reference, then write our own code. Anything adapted from it or from another project is recorded in `NOTICE`.
- **Numbers.** Publish only figures we measured, with the hardware, versions and command that produced them. Label any other figure with its source.
- **Money.** Real-GPU work runs on Modal, inside a task that calls for it, after the owner agrees in that session. Every GPU function scales to zero.
- **Outward actions.** Creating repositories, deploying, posting to other projects and publishing comparisons each wait for the owner to approve the exact draft.

## Where things are

- `plans/02-milestones.md`: milestone order, scope and exit evidence. Read when choosing or scoping work. The first task of each milestone is to write its breakdown.
- `plans/milestone-NN-*.md`: the current milestone's tasks and acceptance criteria.
- `plans/01-decisions.md`: every settled design decision. Read before making any design choice.
- `plans/00-reference-analysis.md`: what the reference does, its defects by ID (R, A, P, S, Q), and competitor, benchmark and licence facts. Read when a task cites a defect ID, reproduces reference behaviour, or needs one of those facts.
- `docs/adr/`: why the surprising decisions were made.
- `CLAUDE.local.md`: notes specific to this machine, when present.
