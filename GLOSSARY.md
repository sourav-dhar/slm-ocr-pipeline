# slm-ocr-pipeline

A self-hosted service that turns documents into Markdown and structured elements using small open-weight models, built to be measured: every quality, speed and cost figure it reports is reproducible.

## Language

### What flows through the system

**Document**:
A file submitted for parsing: a PDF or a single image.
_Avoid_: file, upload, input

**Job**:
One request to parse one document, identified by a job ID and carrying a status until it finishes or fails.
_Avoid_: task (the reference's word; here "task" means a unit of development work), request, ticket

**Page**:
One page of a document.
_Avoid_: sheet, image (a page is not always handled as an image)

**Digital page**:
A page whose text can be read directly from the file.
_Avoid_: born-digital page, native page, text page

**Scanned page**:
A page whose content exists only as pixels.
_Avoid_: image page, raster page

**Region**:
An area of a page found by layout detection, with a label and a bounding box.
_Avoid_: crop, block, box, zone

**Element**:
One unit of parsed output: a typed piece of content with its page, bounding box and position in reading order.
_Avoid_: chunk, segment, result item

### What the system does

**Pipeline**:
The portable core that turns a document into elements.
_Avoid_: SDK, engine, processor

**Page triage**:
The per-page decision of whether recognition is needed at all.
_Avoid_: routing (reserved for choosing between recognisers), classification

**Layout detection**:
Finding and labelling the regions of a page.
_Avoid_: segmentation, layout analysis, layout parsing

**Recognition**:
Turning the pixels of a region or page into text or markup.
_Avoid_: OCR (names the whole field, not this step), inference, transcription

**Two-stage**:
A pipeline shape in which layout detection runs first and recognition runs once per region.
_Avoid_: layout-first, hybrid

**Single-stage**:
A pipeline shape in which one model performs recognition on a whole page with no separate layout detection.
_Avoid_: end-to-end, page-level

### What runs it

**Recogniser**:
A model that performs recognition.
_Avoid_: VLM, SLM, the model, OCR model

**Engine**:
An inference server that hosts a recogniser behind an OpenAI-compatible API.
_Avoid_: server, backend, runtime

**Simulator**:
An engine stand-in that answers like an engine without loading a recogniser.
_Avoid_: mock, stub, fake engine

**Target**:
A place the whole system is deployed. There are two: the Kubernetes target and the Modal target.
_Avoid_: environment, platform, adapter, deployment

**Contract**:
The HTTP interface and output schema that every target must satisfy.
_Avoid_: API spec, interface

### How we judge it

**Reference**:
The public course repository this project re-implements, at its pinned commit.
_Avoid_: upstream, the original, the course

**Baseline**:
A configuration reproduced from someone else's published setup so that ours can be compared against it.
_Avoid_: control, benchmark (a benchmark is a dataset with a scorer)

**Benchmark**:
A public dataset together with its official scorer.
_Avoid_: eval, test set

**Cold start**:
The time from submitting a job while nothing is running to receiving its result.
_Avoid_: warm-up, spin-up time

**Cost per 1,000 pages**:
The price of the compute consumed to parse 1,000 pages at a stated hardware, concurrency and warm state.
_Avoid_: price per page, unit cost
