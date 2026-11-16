# Compete on measured accuracy versus cost, not leaderboard rank

The headline claim this project works toward is the lowest measured cost per accurately parsed page among open self-hosted pipelines, shown as an accuracy-versus-cost chart. We do not try to top a public leaderboard, and the first version trains no model.

Without training, quality is capped by the recogniser we serve, and the leading recognisers on the main benchmark differ by less than that benchmark's own label noise. Serving and system design, on the other hand, are things we can change and measure.

## Consequences

- The evaluation harness is an early milestone, and every published figure states hardware, versions and the command that produced it.
- Techniques that need no training (page triage, routing between recognisers, re-checking uncertain regions) are chosen after baseline results show where errors are.
- Fine-tuning for a niche remains an optional later track, decided once baseline results exist.
