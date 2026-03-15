# RenAIssance HTR Pipeline — Test 2

## Methodology

### Dataset
Five Spanish historical manuscript sources (16th–19th century) provided as PDF scans with ground truth transcriptions in DOCX format. Sources span notarial documents, legal proceedings, and inquisition records.

### Pipeline Overview

**Stage 0 — Scribal Profiling (once per source)**
GPT-4.1 visually analyzes the first page of each source and produces a JSON profile capturing: script style, letter formations (u/v, f/s, long-s), abbreviation inventory, ink quality, show-through severity, dropcap presence, marginalia position, and line density. Generic curator rules are applied uniformly across all sources (ç→z, u/v interchangeability, tilde expansions). The profile is saved to disk and reused for all subsequent pages of the same source — one-time cost per source, not per page. No ground truth is required for profiling.

**Stage 1 — Image Preprocessing (layout detection only)**
Each page is converted from PDF at 150 DPI. A binary image is produced using CLAHE contrast enhancement and Sauvola adaptive thresholding. Parameters are driven by the profile (ink quality, show-through severity). Deskewing is deliberately omitted — it causes 90° rotation on these manuscripts. The binary image is used only for layout detection, not transcription.

**Stage 2 — Semantic Layout Masking**
GPT-4.1 receives the binary image and identifies the bounding box of the primary record block. The prompt explicitly instructs inclusion of headers, dates, dropcaps, and notarial marks, and exclusion of folio numbers and marginalia. The bounding box is applied to the original scan — not the binary image — to produce a clean crop for transcription.

**Stage 3 — Dropcap Detection**
GPT-4.1 identifies whether the page begins with an oversized decorative initial letter. If present, the letter is recorded as a transcription hint. The page image is not modified — the dropcap is read naturally during transcription.

**Stage 4 — VLM Transcription (3-tile sliding window)**
The masked original scan is divided into three overlapping horizontal tiles (TOP 0-45%, MID 28-73%, BOT 55-100%) to prevent GPT-4.1 from truncating output on dense pages. Each tile is transcribed independently at temperature=0.0 using a prompt that injects the full scribal profile. The prompt enforces: one handwritten line equals one output line, hyphenated line breaks are preserved exactly as written, visual fidelity takes priority over linguistic coherence, and unreadable text is marked [?]. Each tile passes its last 4 lines as context to the next tile. Tiles are merged using fuzzy line matching (CER < 0.30 threshold) to remove duplicates at boundaries.

**Stage 6 — LLM Spelling Correction**
GPT-4.1 receives both the transcription text and the original page image and predicts the most likely correct spelling for [?] markers, recovers any lines missing from the draft, and fixes obvious letter confusion patterns (u/v, f/s, m/n, El/M at line start). Hard constraints prevent the LLM from modernizing spelling, joining hyphenated lines, reducing line count, or changing proper names not containing [?].

**Stage 7 — Deterministic Post-Processing Rules**
Hard rules applied after LLM correction that cannot be overridden: ç→z, tilde expansions (q̃→que, ñ→n), macron expansions (ā→an, ē→en). Deterministic rules always run last.

### Key Design Decisions

**No line segmentation** — Traditional HTR pipelines require line segmentation followed by line-level alignment with ground truth. Segmentation errors on historical cursive compound into large CER penalties. The VLM-native approach eliminates this failure mode entirely.

**Original scan for transcription** — CLAHE preprocessing inverts image polarity (mean pixel ~29, dark background). The original scan (mean pixel ~220) is used for all VLM transcription calls. Preprocessing is only used for layout bbox detection.

**Profile without ground truth** — The pipeline requires no ground truth at inference time. The scribal profile is built purely from visual analysis of the first page. For unknown sources with no prior profile, GPT-4.1 builds a visual profile on the fly.

**3-tile approach** — GPT-4.1 truncates output on dense full-page manuscript images. Splitting into three overlapping tiles forces focus on ~8-10 lines at a time, achieving 24/24 line count match on source_1.

**Temperature=0.0** — All transcription calls use temperature=0.0 for maximum determinism. Profile building uses temperature=0.2 for exploratory visual analysis.

### Evaluation

CER computed using NFC unicode normalization only — no accent stripping, no lowercasing. Blank lines removed from both prediction and ground truth before scoring. Line count match reported alongside CER as a diagnostic.

Internal evaluation is on page 1 ground truth only (single GT page per source). True evaluation CER computed by RenAIssance on held-out pages.

### Results

| Source | Century | CER | WER |
|--------|---------|-----|-----|
| source_1_1857 | 19th | 6.90% | 21.35% |
| source_2_1744 | 18th | 13.96% | 37.04% |
| source_3_pleito | 16th-17th | 14.72% | 39.82% |
| source_4_inquisicion | 17th | 17.60% | 45.50% |
| source_5_1606 | 16th-17th | 12.06% | 46.39% |
| **Macro Average** | | **13.05%** | **38.02%** |

## Results

- Test results JSON: handwritingOCR/test_result.json
- Source profiles: handwritingOCR/profile