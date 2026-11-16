# Own the pipeline instead of depending on the GLM-OCR SDK

The reference delegates rasterising, layout detection, cropping, concurrent recognition and formatting to the GLM-OCR SDK. We write that pipeline ourselves. The SDK cannot drive single-stage recognisers, formats output for GLM-OCR only, always rasterises (so page triage is impossible), caps concurrency at 128 regardless of configuration, reports success when regions fail, and switches to the vendor's cloud API if a particular API key is present in the environment.

## Consequences

- Page triage is a first-class step of the pipeline from its first version, even while it sends every page to recognition.
- We give up the SDK's accumulated formatting heuristics. The SDK therefore stays in the benchmark as a baseline: with the same recogniser our pipeline must match its quality score and exceed its throughput before we claim to be better.
- The SDK is Apache-2.0; anything adapted from it is recorded in `NOTICE`.
