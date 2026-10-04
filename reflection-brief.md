# Project Evaluation and Observability — Reflection Brief

## 1. Evaluation Approach

I evaluated three AI extraction/synthesis systems using their provided test suites, static analysis checks, end-to-end executions, and controlled perturbations.

The evaluation focused on whether each system:
- validates outputs rather than trusting extraction blindly,
- represents uncertainty and missing information explicitly,
- routes risky cases for further review,
- detects consistency failures,
- preserves source provenance,
- and degrades gracefully when an upstream source becomes unavailable.

The environment used Python 3.13.0 on Linux x86_64.

---

## 2. System 1 — Validated, Routed Insurance Policy Extraction Pipeline

### Baseline evidence

The complete test suite produced 45 passed tests and 3 skipped tests.

Static checks were clean:
- mypy: no issues found in 11 source files.
- ruff: all checks passed.

The routing-focused fallback test suite passed 9 tests. This included tests covering low-confidence routing, integration failures, reviewer disagreement, stratified sampling, and calibration/reporting behavior.

### Validation and routing observations

The tests demonstrate that the pipeline does not treat every extraction as automatically trustworthy. Required-field nulls are treated as missing source information, formatting and consistency failures trigger retry behavior, and missing source information can halt processing.

The routing tests also demonstrate that low-confidence and integration-failure cases can be routed to human review. The reviewer-disagreement test shows that a high-confidence extraction can still require human review when an independent reviewer disagrees.

### Calibration evidence

The calibration report contained these observed slices:

- auto / premium_amount: n=3, confidence=0.95, accuracy=1.00, Brier=0.003
- home / deductible: n=1, confidence=0.90, accuracy=1.00, Brier=0.010
- umbrella / exclusions: n=2, confidence=0.93, accuracy=0.00, Brier=0.865
- overall Brier score: 0.291

The umbrella/exclusions slice is particularly informative because its observed accuracy was 0.00 despite a reported confidence of 0.93. This provides an independent signal that confidence should not automatically be interpreted as correctness.

### Perturbation

The retry/missing-source tests demonstrated that:
- a required null can become a `missing_source` outcome,
- format and consistency failures cause retry behavior,
- missing source information can halt immediately,
- and retry metadata is preserved.

### Live end-to-end routing evidence

The live policy pipeline was successfully executed against the course-provided Anthropic endpoint. The run produced repeated HTTP 200 responses and wrote 9 routing decisions.

The live routing summary was:

- 8 `human_review`
- 1 `spot_check`
- 0 `auto_approve`
- 1 escalation

The serialized `routing_decisions.json` contains concrete live human-review records. For example, `POL-2025-003` was routed to `human_review` because `exclusions` had confidence 0.85 and there was also reviewer disagreement on `coverage_limit`. `POL-2025-005` was routed to `human_review` because of reviewer disagreements on `coverage_limit`, `deductible`, and `endorsements`.

The live run also recorded:
- `POL-2025-009` with an `endorsements_absent` / `missing_source` pattern
- `POL-2025-010` with a `premium_does_not_match_components` / `consistency` pattern

This provides end-to-end evidence that the live model output reaches validation, reviewer disagreement handling, human routing, spot checking, and escalation rather than relying only on offline routing tests.

---

## 3. System 2 — Resilient Mortgage Document Extraction

### Baseline evidence

The complete test suite produced 25 passed tests.

Static checks were clean:
- mypy: no issues found in 11 source files.
- ruff: all checks passed.

The replay examples demonstrated both successful normalization and explicit consistency checking.

### Observed extraction behavior

The informal appraisal fixture produced:

- gross living area: 2400 square feet
- appraised value: $410,000
- validation: consistent
- discrepancies: none

The missing-bonus income fixture produced:
- base monthly income: 5673.08
- bonus monthly income: `null`
- bonus YTD: `null`
- validation: consistent
- discrepancies: none

This shows that missing information is represented explicitly as null rather than being silently invented.

### Mathematical consistency evidence

The sum-mismatch fixture produced:

- base monthly income: 5416.67
- bonus: 1250.00
- commission: 2140.00
- overtime: 385.50
- other: 450.00
- stated monthly total: 10892.17
- calculated total: 9642.17
- validation: inconsistent
- discrepancy delta: -1250.00

The system therefore preserves both the calculated and stated values and records the discrepancy instead of silently accepting the stated total.

### Perturbation

The controlled comparison between the normal appraisal fixture and the sum-mismatch income fixture showed the difference between a consistent extraction and an extraction containing a mathematical inconsistency.

This demonstrates that extraction quality is evaluated not only by whether fields can be populated, but also by whether the resulting data is internally consistent.

---

## 4. System 3 — Supply Chain Multi-Source Synthesis

### Baseline evidence

The complete test suite produced 34 passed tests.

Static checks were clean:
- mypy: no issues found in 8 source files.
- ruff: all checks passed.

The normal offline Meridian investigation completed successfully.

### Well-Established information

The baseline briefing classified several findings as Well-Established.

For example, average lead time was corroborated across two sources:
- 12.0 days — supplier_audit, as of 2026-04-10
- 12.0 days — logistics, as of 2026-04-05

Defect rate was also reported by two sources:
- 180.0 ppm — supplier_audit, as of 2026-04-10
- 190.0 ppm — internal_quality, as of 2026-04-08

### Contested information

The baseline briefing classified `on_time_delivery_rate` as Contested.

The two reported values were:
- 95.0% — supplier_audit, as of 2026-04-10
- 78.0% — logistics, as of 2026-04-05

The system explicitly marked this high-impact conflict for escalation rather than silently selecting one source as correct.

### Incomplete information

`production_capacity_utilization` was classified as Incomplete because no source reported the metric.

The system therefore distinguished absence of evidence from evidence of a particular value.

### Resilience perturbation

The simulated-timeout run reported:

`Sources unavailable: logistics unavailable (timeout)`

The briefing still completed.

The `late_shipment_count` metric was classified as Incomplete with the reason that reading logistics timed out.

The 78% logistics value for on-time delivery was consequently unavailable, while the available 95% supplier-audit value remained.

This demonstrates graceful degradation: source failure changed the evidence state rather than causing the coordinator to fabricate or preserve unavailable information as though it were current evidence.

---

## 5. Cross-System Reliability Lessons

Across the three systems, several reliability principles appeared repeatedly.

### Validation should be independent of extraction

System 1 validates required fields, formatting, and consistency. System 2 performs mathematical consistency checks after extraction. System 3 compares information across sources.

In each case, the extracted answer is treated as an input to further validation rather than as unquestioned truth.

### Uncertainty should be explicit

System 1 can route uncertain cases to human review and provides calibration evidence.

System 2 represents unavailable income components as `null`.

System 3 separates Well-Established, Contested, and Incomplete information and records source/date details.

These mechanisms make uncertainty visible to downstream users.

### Provenance matters

System 3 demonstrates the value of attaching source and date information to reported values. The 95% and 78% delivery figures are not merely conflicting numbers; the briefing preserves which source reported each value and when.

This makes the disagreement inspectable.

### Failure should change the evidence state, not disappear

The System 3 timeout experiment was especially useful. When logistics became unavailable, its affected information moved into an Incomplete state.

Similarly, System 1's missing-source and retry tests and System 2's discrepancy handling prevent failures from being silently converted into apparently valid outputs.

### Human review is a reliability mechanism

The insurance pipeline demonstrates that automation and human review can coexist. Low confidence, integration failure, and reviewer disagreement can trigger human review rather than forcing an automated decision.

The calibration result for the umbrella/exclusions slice also demonstrates why confidence should be evaluated against observed outcomes.

---

## 6. Perturbation Summary

| System | Perturbation | Observed effect |
|---|---|---|
| Insurance | Missing/invalid extraction conditions | Missing source, retry, or halt behavior was exercised by the tests |
| Mortgage | Sum-mismatch edge-case fixture | Mathematical inconsistency was detected and recorded with calculated/stated values and delta |
| Supply Chain | Simulated logistics timeout | Investigation completed while affected evidence became Incomplete |

The perturbations were useful because they evaluated behavior under imperfect conditions rather than only confirming successful baseline cases.

---

## 7. Overall Reflection

The main lesson from the evaluation is that reliable AI systems need observable controls around the model output.

The three systems use different controls for different failure modes:
- validation and human routing for insurance extraction,
- mathematical consistency checks for mortgage data,
- source corroboration, conflict classification, provenance, and graceful degradation for supply-chain investigation.

The evidence also shows why passing functional tests alone is insufficient. Static checks establish implementation quality, while end-to-end artifacts show how the system behaves in realistic workflows. Perturbations then test whether the system remains explicit and safe when assumptions are violated.

A useful reliability principle across all three systems is:

**Do not make uncertainty disappear. Detect it, represent it, preserve its provenance, and route it appropriately.**

---

## 8. Evidence Index

### System 1
- `01-policy-pipeline/tests.txt`
- `01-policy-pipeline/mypy.txt`
- `01-policy-pipeline/ruff.txt`
- `01-policy-pipeline/routing-fallback.txt`
- `01-policy-pipeline/calibration.txt`
- `01-policy-pipeline/perturbation.txt`
- `environment.txt`

### System 2
- `02-mortgage-extraction/tests.txt`
- `02-mortgage-extraction/mypy.txt`
- `02-mortgage-extraction/ruff.txt`
- `02-mortgage-extraction/appraisal-informal-sqft.txt`
- `02-mortgage-extraction/income-missing-bonus.txt`
- `02-mortgage-extraction/income-sum-mismatch.txt`
- `02-mortgage-extraction/perturbation.txt`

### System 3
- `03-supply-chain/tests.txt`
- `03-supply-chain/mypy.txt`
- `03-supply-chain/ruff.txt`
- `03-supply-chain/baseline-briefing.txt`
- `03-supply-chain/timeout-briefing.txt`
- `03-supply-chain/perturbation.txt`
