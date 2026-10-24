## 5. Failures found and improvements made

Each row is a failure a test layer caught during development, in the order it happened, and the change that fixed it.
Sample numbers are on all 32 public cases (status accuracy / exact sequence accuracy).

| # | Caught by | Failure | Fix | Effect |
|---|---|---|---|---|
| 1 | Sample regression | First end-to-end run: 94% status, 41% sequence. Absence inferred from *any* later snapshot that lacked a record overrode an explicit PRESENT row (case_05's profile left on a failover replica was reported deleted because a later, unrelated inventory had no rows). | Absence inferred from a full snapshot is now only a fallback when nothing explicit ever observed the record. | case_05, case_24 fixed |
| 2 | Sample regression | Workflow steps were attached to the wrong person: any deletion-worker step whose 30-second phase matched was accepted. | Learned *per step class* cadences ("Policy action completed" steps at phase 9–11 s, "final result"/check steps at 20–22 s) from workflows whose account and steps are both identifier-linked. | precision 0.81 → 0.90 |
| 3 | Sample regression | Profile deletions were assigned to cases arbitrarily inside overlapping request windows (case_21/25/27/29 swapped accounts). | Account inherits the owner of its chain's identifiable steps; learned "deletion precedes reported_at by 46–74 min" prior; minute-phase of the first identifiable step; global one-to-one assignment, most decisive first. | sequence 44% → 62% |
| 4 | Sample regression | Failed workflow steps (search entries of *other* people) linked by timing alone (case_18, case_32). | Only successful policy actions may be linked by timing alone: a failed step leaves identifiers behind and must link by identifier. | status 97% → 100% |
| 5 | Trace review (L6) | 10 retained audit records reported ABSENT: a `cache_lookup result="miss"` line in the same log trace overwrote the readback's `rows=1`. | Lines without a resource id contribute only lookup keys to the trace merge. | regression test `split_trace_log` scenario |
| 6 | Invariants (contract) | case_61 produced COMPLETE with no steps (invalid row): the subject's only footprint is an attachment shown in an object-storage screenshot by file name. | Second visual pass that transcribes file listings; data-catalog LINK activities map `object_19 (delivery_confirmation_19.pdf)`; defined fallback for subjects with no observable footprint. | 0 contract errors |
| 7 | Invariants (time travel) | 6/51 rows changed when later evidence was physically removed: the workflow cadence was re-learned from the truncated data, and workflow elimination used other cases' later evidence. | Workflow assignment is computed per audit cutoff with only evidence available then; the learned workflow cadence is a frozen artifact (`code/cache/workflow_model.json`) describing the deletion worker, not any person. | 0 time-travel failures |
| 8 | Sample regression | An attachment last seen 199 days before case_07's request (it belongs to the same person's earlier February request) was attached to the September request. | Records last seen more than 90 days before the request are treated as an earlier lifecycle. | precision 0.963 → 0.978 |
| 9 | Intent eval (L2) | The LLM labelled sample case_27 ("please audit the deletion … confirm the account was fully removed") NON_DELETE when batched with the own intent set. | Prompt states that requests to audit/confirm a removal reaffirm an erasure request. | LLM accuracy 98.9% → 100% on 94 texts |
| 10 | Unit test | The budget allowed one more zero-estimate call after the cap was reached exactly. | `reserve` refuses when committed spend already equals the cap. | budget tests green |
| 11 | External review (independent scorer) | 7 sample `VERIFY_RECORD_PRESENT` steps on retained security audit records were missing. Four of those records appear only in a records bundle *exported* two days after the audit, but its `security_audit_events` snapshot was *taken* at the audit instant; one appears in a later log export on a line timestamped before the audit. | A retained record that the subject's identifiers link to, seen only in a later export whose own snapshot/line time is at or before `audit_at`, adds an uncited post-remediation `VERIFY_RECORD_PRESENT` to an INCOMPLETE case. It never adds a finding or citation, never supports a COMPLETE verdict, and never changes a status. The time-travel invariant allows exactly this difference. | sequence 68.8% → 84.4% (holdout 60% → 100%); 116 eval rows gain the check |
| 12 | External review (probe) | A request with an unknown email and copied timestamps was answered COMPLETE, citing an account attached only by workflow timing. | A subject that resolves to no identity gets no timing-only workflow link, so no verdict can rest on timing alone. | e2e `unknown_email_gets_no_timing_link`; 0 eval rows change |
| 13 | External review (robustness) | Unreadable visual evidence (no cache or no vision backend) silently counted as "no records", which turns INCOMPLETE into COMPLETE. A requests CSV saved with a UTF-8 BOM crashed with `KeyError: 'case_id'`, and a wrong `--dataset-dir` printed a traceback. | Unreadable files are tracked; every COMPLETE verdict that could depend on one is marked `unverified_evidence` in its trace and counted on stderr, and `GDA_STRICT_EVIDENCE=1` turns this into exit code 4. CSVs are read as `utf-8-sig` with trimmed headers. Missing inputs give a one-line error and exit 1. | unit tests `test_byte_order_mark_and_padded_header`, `test_unreadable_visual_evidence_is_tracked` |
| 14 | External review (metrics) | The field-level table was degenerate: steps were aligned on `(phase, system_id, resource_id)`, so those fields matched by construction. | Steps are paired by best field agreement (exact first). | non-degenerate field table |
| 15 | External review (design) | The 90-day staleness cut-off of #8 was a tuned constant that could hide a real leftover on its own. | A record is now left to an earlier lifecycle only if it was *also* last seen no later than the audit of the same person's earlier deletion request. Both sample precedents (case_61 → case_07, case_46 → case_17) satisfy both conditions. Using the earlier-request condition alone turned case_33 into a false COMPLETE (its export was last seen PRESENT four weeks before), which a second independent review caught, so the 90-day age stays as a safety margin. | samples and eval unchanged; e2e `earlier_request_owns_its_leftovers` and `recent_leftover_of_earlier_request_is_reported` |
| 16 | Second independent review | Without the LLM cache the default run still exited 0 with 7 cases flipped to COMPLETE, and the unverified flag marked every COMPLETE row. | A flag now marks only COMPLETE cases during whose request (`requested_at`..`audit_at`) an unreadable file was captured; this catches all 8 offline flips. The run exits 4 by default after writing every row; `GDA_STRICT_EVIDENCE=0` accepts the verdicts. | offline: 120 instead of 158 flagged rows, 8/8 flips caught; replay unaffected |

## 6. Known limitations

- Two sample verification steps (case_18 `audit_record_6645`, case_20 `audit_record_5569`) name security audit records
  that appear in no participant file at all, so they cannot be derived.
- Five hand-written sample cases (case_02/04/07/08/10) have accounts whose deletion activity sits outside the reported
  window or has no activity at all; the generic workflow linker sometimes picks a neighbouring account for them.
- Accuracy of timing (workflow) links on the samples: **18/18** on the generated cases case_11–32 and 6/10 on the
  hand-written cases case_01–10 (all 4 wrong links are case_04/08/10). Identifier links are 76/76 and late-export
  links 5/5. The hand-written deletions use a different activity wording ("reported the requested operation as
  completed") and are logged within ±10 min of `reported_at`. All 6 such events are sample cases; none appears among
  the 518 evaluation requests.
- When several requests share one reporting window and no identifier links any of their accounts, the assignment of
  deletion workflows to requests is resolved by elimination and can swap two accounts. Status is unaffected; only the
  account `resource_id` of a verification step changes.

## 7. How to reproduce

```bash
python3 code/main.py                       # 518 rows -> output.csv (replay cache, no API key)
python3 code/evaluation/main.py            # regenerates this report and usage_report.md
python3 code/evaluation/check_all.py       # gate: unit + e2e tests, sample regression, invariants
```
