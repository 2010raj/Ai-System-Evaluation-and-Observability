# Perturbation Log

For each system, one deliberate change to an input or configuration was used to
predict and observe how the system behaved under an edge case. The observations
below are grounded in the captured run artifacts.

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):** Used the existing missing-source case in the policy dataset where endorsements were absent for `POL-2025-009`. This deliberately exercised the missing-source path rather than changing the extraction code.
- **Command I ran:** `.venv/bin/policy-extractor pipeline data/policies/ --routing-out routing_decisions.json --seed 42`
- **What I predicted:** The system would not invent the missing endorsements and would represent the missing-source condition explicitly, with escalation or review rather than treating the record as a normal successful extraction.
- **What actually happened (paste the key output line):** `endorsements_absent` was reported for `POL-2025-009` with category `missing_source`, and the live run summary reported `escalations: 1`.
- **How this differs from the unperturbed run:** Normal records were routed through `human_review` or `spot_check`. The missing-source case produced an explicit escalation signal instead of being treated as ordinary complete evidence.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):** Used the inconsistent-income fixture represented in `02-mortgage-extraction/discrepancy-run.txt`, where the stated monthly income total does not equal the sum of the extracted components.
- **Command I ran:** `.venv/bin/mortgage-extract fixtures/documents/income_sum_mismatch.txt`
- **What I predicted:** The extraction would preserve the extracted values, but the mathematical validator would detect that the stated total was inconsistent with the calculated component sum.
- **What actually happened (paste the key output line):** `calculated: 9642.17`, `stated: 10892.17`, `delta: -1250.0`, and `consistent: false`.
- **How this differs from the unperturbed run:** The normal extraction run in `extract-run.txt` produced `consistent: true` with no discrepancies. The perturbed income fixture instead produced an explicit mathematical discrepancy while preserving the extracted values.

---

### System 3 — multi-source synthesis

- **Change I made (file + what I changed):** Simulated a timeout while reading the logistics source using the coordinator's timeout simulation option.
- **Command I ran:** `.venv/bin/supply-chain-investigate meridian --offline --simulate-timeout`
- **What I predicted:** The coordinator would degrade gracefully when logistics became unavailable, marking affected evidence as incomplete rather than failing or treating unavailable information as confirmed.
- **What actually happened (paste the key output line):** `Sources unavailable: logistics unavailable (timeout)`. The briefing still completed; `late_shipment_count` became `Incomplete`, and the logistics 78.0% on-time delivery value became unavailable while the available supplier-audit value of 95.0% remained.
- **How this differs from the unperturbed run:** In the baseline run, `on_time_delivery_rate` was `Contested` because both 95.0% from supplier_audit and 78.0% from logistics were available. After the simulated timeout, the logistics value was unavailable, so the conflict disappeared and the available 95.0% supplier-audit value remained. The system completed with an explicit `Incomplete` state instead of failing silently or fabricating evidence.
