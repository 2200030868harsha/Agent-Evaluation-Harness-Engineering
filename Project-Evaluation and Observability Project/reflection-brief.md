# Reflection Brief — Evaluation and Observability Capstone

Name: Harsha Tarun Bimini
Date: 2026-10-05

## 0. Environment

| Field | Value |
|---|---|
| OS & version | Linux 6.6.97+ x86_64 |
| Python version | 3.13.0 |
| Date run | 2026-10-05 |
| Ran any system live? (which) | Policy pipeline attempted; blocked because no Anthropic API key was configured. Offline routing tests were used as the permitted fallback. |

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 45 passed, 3 skipped |
| Routing output file | Not generated: the live pipeline stopped at API authentication before producing routing_decisions.json. Offline routing evidence is in `evidence/01-policy/routing.txt`. |
| auto_approve / human_review / spot_check counts | N/A for live output; routing test evidence includes a deterministic 10-decision case with 7 auto_approve and 3 human_review. |

### 1a Retry boundary

The perturbation test `evidence/01-policy/perturbation.txt` includes the required-field/missing-source behavior. A missing required field is classified as `missing_source`; the test for `endorsements_absent` returns `RetryFutileEscalation` after exactly one API call. Retrying is futile because the source field is absent rather than malformed. Another API call cannot recover information that is not present in the source, so escalation is safer than repeatedly asking the model to invent or guess the value.

### 1b Reading the router

The routing test `tests/test_us04_routing.py` demonstrates the safety boundary directly. The extraction gives every field a confidence of 0.99, but the reviewer disagrees on `premium_amount`. The result is still `human_review`, with `premium_amount` recorded under `reviewer_disagreements` and no fields below the confidence threshold.

The three routing signals are:
1. extractor confidence,
2. reviewer agreement/disagreement,
3. integration-check results.

The test explicitly demonstrates that trusted model confidence alone is insufficient. If confidence alone controlled routing, the 0.99-confidence extraction would have been auto-approved despite the reviewer disagreement.

### 1c Aggregate lies

The calibration artifact `evidence/01-policy/calibration.txt` contains:

`umbrella exclusions n=2 conf=0.93 acc=0.00 brier=0.865`

while the overall Brier score is `0.291`.

This is a clear example of aggregate metrics hiding a dangerous subgroup: the model is highly confident but completely inaccurate for the `umbrella × exclusions` slice. A slicing policy by `policy_type × field` exposes this failure much better than the overall score alone.

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 passed |
| Document run | `evidence/02-mortgage/appraisal.txt` |
| Classified type | Appraisal / property document |

### 2a Mathematical discrepancy

The discrepancy artifact `evidence/02-mortgage/sum_mismatch.txt` shows:

- calculated total monthly income: `9642.17`
- stated monthly total: `10892.17`
- delta: `-1250.0`
- validation: `consistent: false`

Schema/tool validation guarantees that the extracted structure is valid JSON with the expected shape and types. It cannot prove that the numbers are mathematically consistent. The independent deterministic validator catches the arithmetic discrepancy even though the extracted data has a valid structure.

Conversely, a mathematical validator cannot guarantee that malformed JSON or the wrong field types were produced. The two mechanisms therefore provide different guarantees.

### 2b Missing field

In `evidence/02-mortgage/missing_bonus.txt`, `bonus_monthly`, `bonus_ytd`, commission, overtime, other income, and the stated total are represented as `null` when the document does not provide those values.

The system should not fabricate a bonus amount simply because the schema contains a bonus field. `null` preserves the distinction between "not provided" and "zero". This is important because treating missing information as zero would silently change the financial meaning of the document.

### 2c Normalization

The source fixture `fixtures/documents/appraisal_informal_sqft.txt` contains:

`Gross Living Area: approximately 2,400 sq ft`

The extracted value in `evidence/02-mortgage/appraisal.txt` is:

`gross_living_area_sqft: 2400`

Normalizing the measurement during extraction produces a canonical numeric value that downstream validation and analysis can use consistently, while preserving the meaning of the source text.

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 passed |
| Briefing file | `evidence/03-supply-chain/briefing.txt` |
| Section conflict landed in | Contested |

### 3a Conflicting metric

The briefing reports two different on-time delivery rates for Meridian:

- `95.0 percent` — `supplier_audit`, as of `2026-04-10`
- `78.0 percent` — `logistics`, as of `2026-04-05`

The system places the metric under `Contested` and marks it for escalation. Preserving both values is better than silently selecting one because the disagreement itself is important risk information. A decision-maker can then investigate the source definitions, dates, and evidence instead of receiving a falsely precise single number.

### 3b Simulated timeout

The timeout artifact `evidence/03-supply-chain/timeout.txt` begins with:

`Sources unavailable: logistics unavailable (timeout)`

The run still completes. `late_shipment_count` is placed under `Incomplete` with `missing source: timeout reading logistics`.

This distinguishes an unreachable source from a source that simply has nothing to report. The coordinator remains useful because it preserves the available evidence and explicitly identifies what could not be retrieved instead of failing the entire investigation or silently treating missing data as negative evidence.

### 3c Dates as a guardrail

The briefing contains claims about the same supplier with different dates. For example, on-time delivery is reported as 95.0% by `supplier_audit` as of 2026-04-10 and 78.0% by `logistics` as of 2026-04-05.

Requiring an "as of" date prevents a change over time from automatically being interpreted as a contradiction. A later measurement can legitimately differ from an earlier measurement. The date is therefore part of the evidence, not optional metadata.

## 4. Synthesis

### 4a One principle

The clearest example was the mortgage income mismatch. The extracted structure is valid, but the independent validator calculated `9642.17` while the document stated `10892.17`. The artifact `evidence/02-mortgage/sum_mismatch.txt` records `consistent: false` and the `-1250.0` delta.

Without evaluation, a structurally valid LLM extraction could have been accepted as correct. The deterministic check caught a problem that trusting the extraction alone would have allowed to ship.

### 4b Confidence ≠ correctness

The policy system demonstrates this most strongly. In the calibration artifact, `umbrella × exclusions` has confidence `0.93` but accuracy `0.00`, with Brier score `0.865`.

This mattered because a high confidence score did not correspond to reliable output. The routing tests reinforce the same principle: even 0.99 model confidence cannot override a reviewer disagreement. Confidence is therefore one signal for evaluation, not proof of correctness.

### 4c Apply it

For a real workflow where an LLM converts messy documents into structured records, I would combine **validated retry/escalation**, **independent deterministic validation**, and **provenance/conflict annotation**.

The key instruments would be:
- schema validation for structure and types,
- deterministic business-rule checks for arithmetic and consistency,
- confidence/reviewer/integration signals for routing,
- source and date provenance for conflicting external evidence,
- explicit `null`, `Incomplete`, or escalation states instead of invented values.

The overall lesson from these runs is that a reliable system should not only produce an answer. It should produce observable evidence about why the answer can or cannot be trusted.

If confidence alone controlled routing, the 0.99-confidence extraction would have been auto-approved despite the reviewer disagreement.

### 1c Aggregate lies

The calibration artifact `evidence/01-policy/calibration.txt` contains:

`umbrella exclusions n=2 conf=0.93 acc=0.00 brier=0.865`

while the overall Brier score is `0.291`.

This shows that an aggregate metric can hide a dangerous subgroup. The model is highly confident but completely inaccurate for the `umbrella × exclusions` slice. A slicing policy by `policy_type × field` exposes this failure much better than the overall score alone.

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 passed |
| Document run | `evidence/02-mortgage/appraisal.txt` |
| Classified type | Appraisal / property document |

### 2a Mathematical discrepancy

The discrepancy artifact `evidence/02-mortgage/sum_mismatch.txt` shows:

- calculated total monthly income: `9642.17`
- stated monthly total: `10892.17`
- delta: `-1250.0`
- validation: `consistent: false`

Schema/tool validation guarantees that the extracted structure is valid JSON with the expected shape and types. It cannot prove that the numbers are mathematically consistent. The independent deterministic validator catches the arithmetic discrepancy even though the extracted data has a valid structure.

Conversely, a mathematical validator cannot guarantee that malformed JSON or the wrong field types were produced. The two mechanisms therefore provide different guarantees.

### 2b Missing field

In `evidence/02-mortgage/missing_bonus.txt`, `bonus_monthly` and `bonus_ytd` are represented as `null` when the document does not provide those values.

The system should not fabricate a bonus amount simply because the schema contains a bonus field. `null` preserves the distinction between "not provided" and "zero". Treating missing information as zero would silently change the financial meaning of the document.

### 2c Normalization

The source fixture `fixtures/documents/appraisal_informal_sqft.txt` contains:

`Gross Living Area: approximately 2,400 sq ft`

The extracted value in `evidence/02-mortgage/appraisal.txt` is:

`gross_living_area_sqft: 2400`

Normalizing the measurement during extraction produces a canonical numeric value that downstream validation and analysis can use consistently.

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 passed |
| Briefing file | `evidence/03-supply-chain/briefing.txt` |
| Section conflict landed in | Contested |

### 3a Conflicting metric

The briefing reports two different on-time delivery rates for Meridian:

- `95.0 percent` — `supplier_audit`, as of `2026-04-10`
- `78.0 percent` — `logistics`, as of `2026-04-05`

The system places the metric under `Contested` and marks it for escalation. Preserving both values is better than silently selecting one because the disagreement itself is important risk information.

### 3b Simulated timeout

The timeout artifact `evidence/03-supply-chain/timeout.txt` begins with:

`Sources unavailable: logistics unavailable (timeout)`

The run still completes. `late_shipment_count` is placed under `Incomplete` with `missing source: timeout reading logistics`.

This distinguishes an unreachable source from a source that simply has nothing to report. The coordinator preserves available evidence and explicitly identifies what could not be retrieved instead of silently treating missing data as negative evidence.

### 3c Dates as a guardrail

The briefing contains the on-time delivery values of 95.0% from `supplier_audit` as of `2026-04-10` and 78.0% from `logistics` as of `2026-04-05`.

Requiring an "as of" date prevents a change over time from automatically being interpreted as a contradiction. A later measurement can legitimately differ from an earlier measurement, so the date is part of the evidence.

## 4. Synthesis

### 4a One principle

The clearest example was the mortgage income mismatch. The extracted structure is valid, but the independent validator calculated `9642.17` while the document stated `10892.17`. The artifact `evidence/02-mortgage/sum_mismatch.txt` records `consistent: false` and the `-1250.0` delta.

Without evaluation, a structurally valid LLM extraction could have been accepted as correct. The deterministic check caught a problem that trusting the extraction alone would have allowed to ship.

### 4b Confidence ≠ correctness

The policy system demonstrates this most strongly. In the calibration artifact, `umbrella × exclusions` has confidence `0.93` but accuracy `0.00`, with Brier score `0.865`.

This mattered because a high confidence score did not correspond to reliable output. The routing tests reinforce the same principle: even 0.99 model confidence cannot override a reviewer disagreement. Confidence is one signal for evaluation, not proof of correctness.

### 4c Apply it

For a real workflow where an LLM converts messy documents into structured records, I would combine validated retry/escalation, independent deterministic validation, and provenance/conflict annotation.

The key instruments would be:

- schema validation for structure and types,
- deterministic business-rule checks for arithmetic and consistency,
- confidence/reviewer/integration signals for routing,
- source and date provenance for conflicting external evidence,
- explicit `null`, `Incomplete`, or escalation states instead of invented values.

The overall lesson from these runs is that a reliable system should not only produce an answer. It should produce observable evidence about why the answer can or cannot be trusted.
