# Permissive licences only, and comparisons only where terms allow

Every recogniser, layout model and library this project ships or defaults to carries a permissive licence (Apache-2.0, MIT or equivalent) with no revenue cap, non-commercial clause or copyleft. Hosted services are included in published comparisons only where their terms permit publishing results.

This is a constraint the code cannot show, and it is one of the few advantages a small project can hold over better-funded tools whose model weights carry revenue caps or custom licences.

## Consequences

- Excluded as shipped recognisers regardless of score: OXR-1.0, Chandra, HunyuanOCR, Nanonets-OCR2 and any model with a non-commercial or revenue-capped licence. They may still be run for comparison where their licence allows research use.
- Excluded from published comparisons: Reducto and Datalab's hosted API (terms forbid it for a competing project) and LlamaParse (requires written consent).
- Benchmark datasets with research-only licences are downloaded at run time and never redistributed.
- A new dependency's licence, and a new comparison target's terms, are checked before it is added.
