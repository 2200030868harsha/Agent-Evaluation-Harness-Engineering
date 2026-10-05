# Perturbation Log

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):** In the retry test input, the required `endorsements` field was set to `None`, representing a source where the required field was absent.
- **Command I ran:** `.venv/bin/pytest tests/test_us01_retry.py -v`
- **What I predicted:** The validator should classify the problem as a missing source rather than retrying repeatedly, because the missing field cannot be recovered from another model call.
- **What actually happened (paste the key output line):** `test_ac_01_04_missing_source_halts_immediately PASSED`; the test asserts `RetryFutileEscalation` and exactly one API call.
- **How this differs from the unperturbed run:** Recoverable format/consistency failures can retry, but an absent required source field causes immediate escalation instead of another API call.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):** Used the deliberate edge-case input `fixtures/documents/income_sum_mismatch.txt`, where the stated monthly income total does not equal the sum of the extracted components.
- **Command I ran:** `.venv/bin/mortgage-extract fixtures/documents/income_sum_mismatch.txt --mode replay --verbose`
- **What I predicted:** The extraction should remain structurally valid, but the independent mathematical validator should flag the inconsistent total.
- **What actually happened (paste the key output line):** `"consistent": false` with `calculated: 9642.17`, `stated: 10892.17`, and `delta: -1250.0`.
- **How this differs from the unperturbed run:** The normal appraisal and missing-bonus runs were consistent; the mismatch input produced an explicit validation discrepancy instead of silently accepting the extracted total.

---

### System 3 — multi-source synthesis

- **Change I made (file + what I changed):** Enabled the coordinator's simulated source-timeout configuration with `--simulate-timeout`, causing the logistics source to become unavailable.
- **Command I ran:** `.venv/bin/supply-chain-investigate meridian --offline --simulate-timeout`
- **What I predicted:** The investigation should still finish, preserve evidence from available sources, and mark information dependent on the failed logistics source as incomplete.
- **What actually happened (paste the key output line):** `Sources unavailable: logistics unavailable (timeout)` and `late_shipment_count` was reported as `[missing source: timeout reading logistics]`.
- **How this differs from the unperturbed run:** The normal run used both logistics and other sources; after the timeout, logistics-dependent evidence became incomplete while the overall briefing still completed.

---
