# Compete on measured accuracy versus cost, not leaderboard rank

The headline claim this project works toward is the lowest measured cost per accurately parsed page among open self-hosted pipelines, shown as an accuracy-versus-cost chart. We do not try to top a public leaderboard, and the first version trains no model.

Without training, quality is capped by the recogniser we serve, and the leading recognisers on the main benchmark are reported to differ by less than the label-error rate one audit of that benchmark found. Both the scores and the audit are other people's figures that we have not verified; see `plans/00-reference-analysis.md`, section 11.4. Serving and system design, on the other hand, are things we can change and measure.

## Consequences

- The evaluation harness is an early milestone, and every published figure states hardware, versions and the command that produced it.
- Techniques that need no training beyond page triage and the digital-page path, such as routing between recognisers and re-checking uncertain regions, are chosen after baseline results show where errors are.
- Fine-tuning for a niche remains an optional later track, decided once baseline results exist.
