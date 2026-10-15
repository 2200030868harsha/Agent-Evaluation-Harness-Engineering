# Perturbation Log

---

### System 1 — validated, routed pipeline

- **Learner-created change:** Modified the actual policy input `data/policies/POL-2025-001.txt` by removing the `ENDORSEMENTS` section and its three endorsement entries.
- **Baseline:** The original policy input contained the `ENDORSEMENTS` section with Rental Reimbursement, Roadside Assistance, and Additional Insured entries.
- **Perturbed input:** The `ENDORSEMENTS` section was removed while the surrounding policy content was left unchanged.
- **Evidence of the change:** `perturbation-system1-diff.txt` records the exact learner-created input diff.
- **Predicted behavior:** Because `endorsements` is a required extracted field, removing its source information should make the extraction unable to recover that field from the document and should lead to the missing-source/escalation boundary rather than inventing a value.
- **Execution limitation:** A live `policy-extractor extract` run was attempted against the modified policy, but the configured Anthropic/Vocareum endpoint rejected the project's default model (`claude-haiku-4-5`) because it was not available for the organization. Therefore, no live extraction result is claimed here.
- **Supporting system behavior:** The existing retry test suite documents the corresponding missing-source boundary: `endorsements_absent` produces `RetryFutileEscalation` after exactly one API call, while recoverable format/consistency failures are retryable.
- **Evidence:** `01-policy-pipeline/perturbation-system1-diff.txt`

---

### System 2 — schema-enforced two-pass extraction

- **Learner-created change:** Created `perturbations/income_total_perturbation.txt` rather than modifying or reusing the bundled `income_sum_mismatch.txt` fixture.
- **Baseline input:** Base monthly income `$4,000.00`, bonus monthly income `$500.00`, and stated monthly total `$4,500.00`.
- **Baseline result:** `consistent=True` with no discrepancies.
- **Perturbation:** Changed only the learner-created input's stated monthly total from `$4,500.00` to `$5,000.00`.
- **Observed result:** `consistent=False`; calculated monthly total remained `$4,500.00`, stated total became `$5,000.00`, and the validator reported a `total_monthly_income` discrepancy with delta `-500.0`.
- **Evidence:**
  - `02-mortgage-extraction/perturbations/income_total_perturbation.txt`
  - `02-mortgage-extraction/perturbations/income_total_perturbation_run.txt`

---

### System 3 — multi-source synthesis

- **Learner-created perturbation:** Enabled the coordinator's simulated source-timeout configuration with `--simulate-timeout`, causing the logistics source to become unavailable.
- **Command:** `.venv/bin/supply-chain-investigate meridian --offline --simulate-timeout`
- **Predicted behavior:** The investigation should still finish, preserve evidence from available sources, and mark information dependent on the failed logistics source as incomplete.
- **Observed behavior:** `Sources unavailable: logistics unavailable (timeout)` was reported; logistics-dependent information such as `late_shipment_count` was marked `[missing source: timeout reading logistics]`, while the overall briefing still completed.
- **Contrast with normal run:** The normal offline investigation completed with logistics evidence available. Under the timeout perturbation, logistics-dependent evidence became incomplete without preventing the overall investigation from completing.
- **Evidence:** `03-supply-chain/timeout-run.txt`
