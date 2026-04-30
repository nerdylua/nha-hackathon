# PS-01 - Advanced NLP / OCR: Clinical Document Classification & STG Compliance

## Problem Statement

Automatically read mixed-quality healthcare claim documents, classify each page/document, extract required clinical and administrative evidence, detect mandatory visual elements where applicable, and evaluate each claim against National Health Authority (NHA) Standard Treatment Guidelines (STG).

The solution must produce a strict JSON output for every page of every document in all package folders under `Claims/`. Each JSON row should explain which mandatory documents, clinical findings, dates, and extra documents were detected for the package code, along with the page number and document timeline rank.

## Dataset Structure

The dataset is organized by package code, then by case folder:

```text
Claims/
  MG006A/
    <case_id>/
      <claim documents: .pdf, .jpg, .jpeg>
  MG064A/
    <case_id>/
      <claim documents: .pdf, .jpg, .jpeg>
  SB039A/
    <case_id>/
      <claim documents: .pdf, .jpg, .jpeg>
  SG039C/
    <case_id>/
      <claim documents: .pdf, .jpg, .jpeg>
```

Observed dataset summary:

| Package Code | Condition | Case Folders | Files |
|:-------------|:----------|-------------:|------:|
| MG006A | Enteric Fever | 10 | 139 |
| MG064A | Severe Anemia | 10 | 104 |
| SB039A | Total Knee Replacement | 10 | 98 |
| SG039C | Cholecystectomy | 10 | 93 |

File types observed: `245` PDFs, `120` JPGs, and `69` JPEGs.

## Core Requirements

- Robust OCR for scanned PDFs and images with varied quality, orientation, and document layouts.
- Page-level document classification for mandatory STG documents and extra/non-required documents.
- Field extraction for dates, clinical findings, diagnosis indicators, procedure/package evidence, and document metadata.
- Visual element detection where required, such as post-operative photos, implant visibility, signatures, stamps, stickers, invoices, QR/barcodes, and report images.
- Rule checks for STG eligibility, package-specific clinical criteria, diagnosis-to-procedure support, length/timeline consistency, and age/date validity.
- Explainability through confidence scores and provenance where the implementation supports it: page number, source document, bounding box, extracted text span, and detected visual region.
- Temporal validation by assigning `document_rank` to pages/documents according to the clinical episode sequence.
- Human-in-the-loop review support for uncertain or Conditional cases.

## Required Output Contract

`ps-1.txt` defines the evaluation output. The output must be a JSON array of objects, one object per page/document page analyzed.

Important rules:

- The solution must support all 4 package codes: `MG064A`, `SG039C`, `MG006A`, and `SB039A`.
- Output format must strictly match the package-specific keys. Solutions that do not follow the expected keys can be rejected without evaluation.
- Use `1` for present/true and `0` for absent/false for binary checks.
- Use `null` when a date field is not available.
- `case_id` comes from the case folder name.
- The document link key is shown inconsistently in the source examples as `link`, `S3_link`, `S3_link/DocumentName`, or `s3_link`; normalize to the evaluator's expected key if provided by the submission template.
- `procedure_code` must exactly match the parent package folder.
- `page_number` is `1` for a single-page image and sequential for pages in a multi-page PDF.
- `extra_document = 1` means the document/page is outside the STG-listed package requirements.
- `document_rank = 99` must be used for extra documents.
- If a multi-page document has one timeline rank, assign the same `document_rank` to all pages belonging to that document.

## Common Identification Fields

Every package output includes these identifiers:

| Key | Type | Description |
|:----|:-----|:------------|
| `case_id` | string | Unique case identifier, usually the case folder name. |
| `S3_link` / document link | string | Link or path of the document being analyzed. |
| `procedure_code` | string | Package code: one of `MG064A`, `SG039C`, `MG006A`, `SB039A`. |
| `page_number` | integer | Page number within the source document. |
| `extra_document` | integer | `1` if outside required STG document list, else `0`. |
| `document_rank` | integer | Chronological rank in the treatment timeline; `99` for extra documents. |

## Package A: MG064A - Severe Anemia

Expected JSON keys:

```json
{
  "case_id": "C1",
  "S3_link": "Bucket_name/document.pdf",
  "procedure_code": "MG064A",
  "page_number": 1,
  "clinical_notes": 0,
  "cbc_hb_report": 1,
  "indoor_case": 0,
  "treatment_details": 0,
  "post_hb_report": 0,
  "discharge_summary": 0,
  "severe_anemia": 1,
  "common_signs": 0,
  "significant_signs": 0,
  "life_threatening_signs": 0,
  "extra_document": 0,
  "document_rank": 2
}
```

Required checks:

| Key | Meaning |
|:----|:--------|
| `clinical_notes` | Clinical note documenting patient condition. |
| `cbc_hb_report` | CBC/Hemoglobin investigation report. |
| `indoor_case` | Indoor case or admission documentation. |
| `treatment_details` | Treatment/transfusion details. |
| `post_hb_report` | Post-treatment hemoglobin report. |
| `discharge_summary` | Discharge summary after treatment. |
| `severe_anemia` | STG severe anemia criteria satisfied. |
| `common_signs` | Any common sign such as pallor, fatigue, or weakness is documented. |
| `significant_signs` | Any significant sign such as tachycardia or breathlessness is documented. |
| `life_threatening_signs` | Any life-threatening sign such as cardiac failure, severe hypoxia, or shock is documented. |

Typical rank order: clinical notes, CBC/Hb investigation, treatment details, post-Hb report, discharge summary.

## Package B: SG039C - Cholecystectomy

Expected JSON keys:

```json
{
  "case_id": "MAV/GJ/R3/2026032310030008",
  "S3_link": "Bucket_name/document.pdf",
  "procedure_code": "SG039C",
  "page_number": 1,
  "clinical_notes": 0,
  "usg_report": 0,
  "lft_report": 0,
  "operative_notes": 0,
  "pre_anesthesia": 0,
  "discharge_summary": 1,
  "photo_evidence": 0,
  "histopathology": 0,
  "clinical_condition": 1,
  "usg_calculi": 0,
  "pain_present": 1,
  "previous_surgery": 0,
  "extra_document": 0,
  "document_rank": 5
}
```

Required checks:

| Key | Meaning |
|:----|:--------|
| `clinical_notes` | Required clinical notes. |
| `usg_report` | Ultrasound report. |
| `lft_report` | Liver function test report. |
| `operative_notes` | Operative notes. |
| `pre_anesthesia` | Pre-anesthesia/pre-operative fitness record. |
| `discharge_summary` | Discharge summary. |
| `photo_evidence` | Required photo evidence. |
| `histopathology` | Histopathology report. |
| `clinical_condition` | Any qualifying STG clinical condition is present. |
| `usg_calculi` | USG evidence of calculi is present. |
| `pain_present` | Pain is documented. |
| `previous_surgery` | Previous surgery is documented where relevant. |

## Package C: MG006A - Enteric Fever

Expected JSON keys:

```json
{
  "case_id": "C1",
  "S3_link": "Bucket_name/document.pdf",
  "procedure_code": "MG006A",
  "page_number": 1,
  "clinical_notes": 0,
  "investigation_pre": 1,
  "pre_date": "06-02-2026",
  "vitals_treatment": 0,
  "investigation_post": 0,
  "post_date": null,
  "discharge_summary": 0,
  "poor_quality": 1,
  "fever": 0,
  "symptoms": 0,
  "extra_document": 0,
  "document_rank": 2
}
```

Required checks:

| Key | Meaning |
|:----|:--------|
| `clinical_notes` | Required clinical notes. |
| `investigation_pre` | Pre-treatment investigation report. |
| `pre_date` | Date extracted from pre-investigation report only; expected date format should be normalized, preferably `DD-MM-YYYY`. |
| `vitals_treatment` | Vitals and treatment documentation. |
| `investigation_post` | Post-treatment investigation report. |
| `post_date` | Date extracted from post-investigation report only; `null` if absent. |
| `discharge_summary` | Discharge summary. |
| `poor_quality` | Page/document quality is poor enough to affect OCR or interpretation. |
| `fever` | Fever criteria documented as per STG. |
| `symptoms` | Any qualifying symptom among the STG-listed symptoms is documented. |

## Package D: SB039A - Total Knee Replacement

Expected JSON keys:

```json
{
  "case_id": "C1",
  "S3_link": "Bucket_name/document.pdf",
  "procedure_code": "SB039A",
  "page_number": 1,
  "clinical_notes": 0,
  "xray_ct_knee": 0,
  "indoor_case": 0,
  "operative_notes": 0,
  "implant_invoice": 0,
  "post_op_photo": 0,
  "post_op_xray": 0,
  "discharge_summary": 0,
  "doa": null,
  "dod": null,
  "arthritis_type": 0,
  "post_op_implant_present": 0,
  "age_valid": 0,
  "extra_document": 1,
  "document_rank": 99
}
```

Required checks:

| Key | Meaning |
|:----|:--------|
| `clinical_notes` | Required clinical notes. |
| `xray_ct_knee` | X-ray or CT knee evidence. |
| `indoor_case` | Indoor case/admission document. |
| `operative_notes` | Operative notes. |
| `implant_invoice` | Implant invoice. |
| `post_op_photo` | Post-operative photo. |
| `post_op_xray` | Post-operative X-ray. |
| `discharge_summary` | Discharge summary. |
| `doa` | Date of admission from discharge summary, normalized to `DD-MM-YYYY`; `null` if unavailable. |
| `dod` | Date of discharge from discharge summary, normalized to `DD-MM-YYYY`; `null` if unavailable. |
| `arthritis_type` | Clinical condition/arthritis type justifies the package. |
| `post_op_implant_present` | Post-operative report/image shows implant present. |
| `age_valid` | Patient age satisfies STG requirement. |

## Document Ranking Guidance

`document_rank` represents the clinical episode order. Lower ranks are earlier in the treatment journey; higher ranks are later. Use `99` for extra documents.

General sequence:

| Rank | Event Type | Examples |
|-----:|:-----------|:---------|
| 1 | Clinical/admission evidence | Clinical notes, indoor case, admission notes |
| 2 | Pre-treatment investigations | CBC/Hb, USG, LFT, X-ray/CT, pre-investigation |
| 3 | Procedure/treatment evidence | Treatment details, vitals/treatment, operative notes |
| 4 | Post-treatment evidence | Post-Hb, post-op photo, post-op X-ray, implant evidence |
| 5 | Discharge | Discharge summary |
| 99 | Extra document | Consent-only pages, unrelated labs, unrelated images, duplicate/non-STG documents |

Package-specific ranking may vary slightly, but the rank must remain chronologically consistent and stable across all pages of the same multi-page document.

## Human-Readable Summary - Sample Table

| Claim ID | File | Page | Document Classification | Claim Clinical Rules Checks |
|---------:|:-----|:----:|:------------------------|:----------------------------|
| CN001 | D1.pdf | 1 | Discharge summary | Admission date is after surgery date; clinical rules as per STG are not matching |
| CN001 | D1.pdf | 2 | Clinical note | |
| CN001 | D1.pdf | 3 | Angioplasty report | |
| CN001 | D2.pdf | 1 | X-ray | |
| CN001 | D3.pdf | 1 | CT Scan | |
| CN001 | d4.png | 1 | Patient Photo | |
| CN001 | d5.png | 1 | Non-required document | Not required for clinical or claim validation |

## Sequence - Sample Episode Timeline

| Sequence | Event Type | Date | Source Document | Temporal Validity |
|---------:|:-----------|:-----|:----------------|:------------------|
| 1 | Admission | 02-Feb-2026 | Discharge Summary / Indoor Case | Valid |
| 2 | Diagnostic Investigation | 02-Feb-2026 | CT Scan / X-ray / Lab Report | Before procedure |
| 3 | Procedure or Treatment | 03-Feb-2026 | Operative Notes / Treatment Details | Valid |
| 4 | Post-Procedure Monitoring | 04-05 Feb 2026 | Clinical Notes / Post-op Evidence | Valid |
| 5 | Discharge | 06-Feb-2026 | Discharge Summary | After treatment |

## Suggested Solution Components

- **Document intake**: Recursively ingest all files under `Claims/<procedure_code>/<case_id>/`.
- **OCR and layout understanding**: Extract text while preserving page structure, tables, stamps, headers, dates, and report sections.
- **Document classification**: Classify each page/document into package-specific mandatory document categories or `extra_document`.
- **Visual detection**: Detect visual evidence needed by package, especially photos, post-op X-rays, implant evidence, invoices, stamps, signatures, and stickers.
- **Data structuring**: Normalize extracted values into the package-specific JSON schema.
- **Rules engine**: Encode STG/policy checks for each package code.
- **Timeline builder**: Assign `document_rank`, propagate rank across pages of the same document, and flag temporal inconsistencies.
- **Decisioning and explainability**: Produce Pass / Conditional / Fail internally or in a companion report, with reasons and evidence pointers.

## Notebook Implementation Context

The starter notebook `nha_ps1_skeletal_notebook_main.ipynb` is the intended coding scaffold. It is not a complete implementation; most functions are stubs that participants must fill in.

Notebook configuration:

| Name | Purpose |
|:-----|:--------|
| `DATA_ROOT = Path("./Claims")` | Input dataset root. In this local dataset, `Claims` contains package folders first, then case folders. |
| `OUTPUT_ROOT = Path("./outputs")` | Suggested destination for JSON/CSV/HTML outputs. |
| `SUPPORTED_EXTENSIONS` | Includes PDF and common image formats: `.pdf`, `.png`, `.jpg`, `.jpeg`, `.tif`, `.tiff`, `.bmp`. |
| `PACKAGE_CODES` | `["MG064A", "SG039C", "MG006A", "SB039A"]`. |
| Decision labels | `PASS`, `CONDITIONAL`, `FAIL`. |

The notebook provides `PACKAGE_SCHEMAS` and a `validate_output_rows(package_code, rows)` helper. The validator checks **exact key names and exact key order**, so row dictionaries must be initialized from the relevant schema and preserved through post-processing.

Schema link-key caveat:

- `MG064A` uses `link` in the notebook schema.
- `SG039C` uses `S3_link/DocumentName` in the notebook schema.
- `MG006A` uses `S3_link/DocumentName` in the notebook schema.
- `SB039A` uses `link` in the notebook schema.
- The notebook also defines `KEY_ALIASES` to map `S3_link`, `s3_link`, and `S3_link/DocumentName` to `link` internally. For final evaluation, emit the exact key expected by `PACKAGE_SCHEMAS` or the final organizer template.

Primary scaffold functions to implement:

| Stage | Function(s) |
|:------|:------------|
| Ingest files | `iter_case_files`, `discover_cases` |
| Split files into pages | `extract_pages` |
| OCR | `run_ocr` |
| Quality assessment | `estimate_page_quality` |
| Visual cue detection | `detect_visual_elements` |
| Document classification | `classify_document_type` |
| Entity helpers | `find_dates`, `find_age`, `contains_any`, `normalize_date` |
| Row creation | `initialize_output_row`, `populate_row_for_package`, `infer_document_rank` |
| Timeline | `build_episode_timeline`, `build_timeline_df` |
| Rules and decisions | `aggregate_case_rows`, `run_rules_engine`, `build_human_readable_summary`, `render_decision_report` |
| Export | `export_case_outputs` |

Important implementation notes from the notebook:

- `discover_cases` must account for the real dataset layout: `Claims/<package_code>/<case_id>/<files>`. The final runner in the notebook attempts to infer package code from `PACKAGE_CODE_LOOKUP` or `Claims/<case_id>/package_code.txt`, so it may need adjustment to iterate package folders directly.
- The final runner expects each processed page to produce a `PageResult`, a strict output row, timeline entries, a decision object, and reviewer-facing summary/timeline DataFrames.
- Final notebook objects are `BATCH_RESULTS`, `FINAL_STRICT_OUTPUTS`, `FINAL_PAGE_RESULTS`, `FINAL_DECISIONS`, `FINAL_SUMMARIES`, and `FINAL_TIMELINES`.
- `FINAL_STRICT_OUTPUTS` is the important evaluation-facing structure: a mapping from case ID to the list of strict per-page JSON rows.
- `FINAL_STRICT_OUTPUTS_DF` is only a preview/analysis table; JSON rows remain the required submission artifact.
- Date normalization should target `DD-MM-YYYY` for `MG006A.pre_date`, `MG006A.post_date`, `SB039A.doa`, and `SB039A.dod`.
- Be careful with duplicate/stub cells in the notebook: later cells redefine `normalize_date` and `normalize_dates_in_rows` as `pass`, overriding earlier working examples if executed in order. Keep the implemented versions in the final executed notebook.
- Some scaffold function signatures are inconsistent between definitions and the final runner. Align them before running end-to-end, especially `run_ocr`, `detect_visual_elements`, and `classify_document_type`.

Suggested output files from `export_case_outputs`:

- Strict package-specific JSON rows for evaluation.
- Human-readable case summary table.
- Episode timeline table.
- Decision record containing `PASS`, `CONDITIONAL`, or `FAIL`, confidence, missing documents, rule flags, and timeline flags.

## Scoring & Evaluation

Awards to the top 3 teams by overall score. Minimum qualification score: **>= 70%**.

| Category | Weightage | Rank 1 | Rank 2 | Rank 3 |
|:---------|:---------:|:-------|:-------|:-------|
| Document Classification | 40% | F1 >= 0.95 | 0.90 <= F1 < 0.95 | 0.85 <= F1 < 0.90 |
| Rule creation logic and provenance detection | 40% | F1 >= 0.96 | 0.90 <= F1 < 0.96 | 0.85 <= F1 < 0.90 |
| Design of the solution | 20% | F1 >= 0.93 | 0.88 <= F1 < 0.93 | 0.80 <= F1 < 0.88 |

## Notes

- Treat `ps-1.txt` as the source of truth for the strict output schema.
- Keep package-specific keys exactly as expected by the evaluator.
- Preserve provenance for each field and rule check internally, even if the submitted JSON only requires binary/document-rank fields.
- Conditional cases should be surfaced for human review with the uncertain field, confidence, source page, and extracted evidence.
