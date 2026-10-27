# Evaluation Log - Triage Agent

Project: Salem CivicAgent
Last updated: 2026-10-27

Purpose: record every evaluation run with the exact setup that produced it, so each number can be traced and reproduced. Never compare numbers from different row sets or different setups without saying so.

## 1. Setup

| Item | Value |
|---|---|
| Task | Classify a citizen complaint into category, priority and location |
| Categories | water_supply, drainage_sewage, garbage, streetlight, road_damage, other |
| Priorities | low, medium, high, emergency |
| Dev set | `data/complaints.csv`, 100 rows, hand-reviewed labels (see section 3) |
| Test set | `data/test_set.csv`, 50 rows. Run once on 2026-10-27 after the prompt was frozen |
| Labeling rules | `data/labeling_guide.md` |
| LLM | gemini-3.5-flash-lite(from `.env`; **confirm**, because result files made before the guard do not store the model name) |
| Temperature | 0 |
| Output | Structured JSON validated with Pydantic |
| Eval script | `scripts/eval_triage.py`, rows shuffled with seed 42, results cached per run in `results/` |

## 2. Metrics

- **Category accuracy:** predicted category equals label.
- **Priority exact accuracy:** predicted priority equals label.
- **Priority within one level:** predicted priority is at most one step from the label (low < medium < high < emergency).
- **Emergency recall:** of the rows labelled emergency, how many were predicted emergency. This is the safety-critical number.
- **Confusion matrix:** shows which labels are mistaken for which.

## 3. Dataset history and label changes

| Date | Change | Reason |
|---|---|---|
| 2026-10-24 | Row C032 category changed from drainage_sewage to water_supply | The label contradicted our own rule: contaminated drinking water is water_supply |
| 2026-10-24 | Rows C087, C090, C093, C098 (category other) were accidentally removed from `complaints.csv`, then restored with `scripts/restore_rows.py` | Rows had no priority rule. A run scored on 96 rows (73% exact) was **discarded** because removing hard rows inflated the score |

Rule going forward: a label may be changed when the guide shows it is wrong. Rows are never deleted because the model gets them wrong.

## 4. Runs on the dev set (100 rows)

| Run | Prompt | Category | Priority exact | Within one | Emergency recall | Low rated medium |
|---|---|---|---|---|---|---|
| prompt1_full | v1 (short rules) | 99% | 71% | 100% | 14/15 | 14 |
| prompt2_run1 | v2 (full guide rules) | 98% | 72% | 100% | 14/15 | 11 |
| prompt2_run2 | v2, same prompt rerun | 98% | 72% | 99% | 13/15 | 10 |

Earlier partial run, not comparable: 23 rows, water and drainage only, a different model (gemini-3.8-flash), category 100%, priority exact 74%. Ordering bias made the sample unrepresentative, so it is not used for any conclusion.

### Confusion matrices (label -> predicted)

**prompt1_full**
```
      low: low=17  medium=14  high=0  emergency=0
   medium: low=3   medium=20  high=2  emergency=0
     high: low=0   medium=5   high=20 emergency=4
emergency: low=0   medium=0   high=1  emergency=14
```

**prompt2_run1**
```
      low: low=20  medium=11  high=0  emergency=0
   medium: low=4   medium=17  high=4  emergency=0
     high: low=0   medium=6   high=21 emergency=2
emergency: low=0   medium=0   high=1  emergency=14
```

**prompt2_run2**
```
      low: low=21  medium=10  high=0  emergency=0
   medium: low=4   medium=17  high=4  emergency=0
     high: low=0   medium=6   high=21 emergency=2
emergency: low=0   medium=1   high=1  emergency=13
```

## 5. Findings

1. **Prompt v2 is not measurably better than v1.** Exact priority moved by one point, which is inside the run-to-run noise. With n=100, a 72% score has a 95% margin of roughly +/- 9 points.
2. **Temperature 0 is not deterministic.** The same prompt run twice changed 5 of 100 rows (C086, C054, C094, C098, C055). One flip rated a child attacked by dogs as `high` in one run and `medium` in the next.
3. **Real movement under the same total:**
   - Fixed: no-water-for-3+-days rows (C001, C005), dark-street rows (C053, C062), the fallen tree category (C091), and some low rated medium.
   - Broke: C040 (dead animal) and C054 (leaning pole) moved to `other` because the v2 `other` rule was too broad. Garbage rows with pests (C036, C044, C051) dropped from high to medium.
4. **Systematic bias:** about a third of `low` complaints are rated `medium`. The model prefers medium for any real but minor service problem.
5. **Unresolved label rules:** rows in category `other` (C087, C090, C093, C098) are rated `low` while labelled `medium`, because no priority rule for `other` exists yet. No prompt can fix a missing rule.
6. **Subjective boundaries:** high vs emergency on rows such as C006, C013, C023 and C033 is arguable. Some of the remaining 28% error is label subjectivity, not model failure.
7. **No dangerous under-triage by more than one level**, except one run-2 flip of C086 (emergency labelled, `medium` predicted).

## 6. Rule-based safety floor (`app/agents/safety.py`)

A regex layer that only ever raises priority to `emergency` when the text matches the emergency definition in the guide (person attacked or bitten, electric shock, sparks, dangerous or hanging wires, open manhole, collapse or sinkhole, medical waste).

Evaluated on `results/triage_prompt2_run2.jsonl` (no API calls):

| Metric | Before floor | After floor |
|---|---|---|
| Priority exact | 72% | 74% |
| Emergency recall | 13/15 | 15/15 |

Rows changed: C086 (person attacked) and C054 (dangerous wires). Both changes matched the labels, and the floor raised no non-emergency complaint, so no false alarms appeared on the dev set.

**Caveat:** the patterns were written from the guide, but after the dev errors C086 and C054 had been seen. The dev improvement is therefore optimistic. The test set is the honest check, including a count of false alarms.

## 7. Decisions

- Route all category `other` complaints to manual review. Their priority only orders the review queue.
- Use deterministic code for rules that can be written down (department lookup, safety floor, escalation timers). Use the LLM only for understanding text.
- Stop tuning the prompt on the dev set. Remaining prompt edits are narrow rule fixes only (see section 8).

## 8. Open items

- [ ] Write priority rules for category `other` (bribery, cattle or blocked public space, noise, outbreak risk, attacks) and update the labels in the guide
- [ ] Narrow v3 prompt edits: dead animal on a road is garbage; street-lighting poles and wires are streetlight; trees, stray animals, taxes and electricity bills are other; pests, burning waste or rats around garbage mean high; a person injured or attacked is emergency
- [ ] Add prompt hash and model to every result file (guard in `eval_triage.py`) and confirm the model name above
- [x] Freeze the prompt, then run `data/test_set.csv` **once**; report category, priority exact, within-one, emergency recall, confusion matrix, and safety-floor false alarms (see section 10)
- [ ] Review 15 to 20 test-set labels independently and record the agreement rate

## 9. Reproduce

```
python -m scripts.eval_triage <run_name>              # calls the LLM, caches results
python -m scripts.eval_triage <run_name> --score-only # re-score saved results, no API calls
python -m scripts.eval_safety <run_name>              # evaluate the safety floor on saved results
```

```
| prompt3_run1 | flash-lite (3.5) | v3 | 99% | 75% | 99% | 14/15 (15/15 with floor) |
| prompt4_run1 | flash-lite (3.5) | v4 | 99% | 76% | 100% | 12/15 (floor result pending) |

```

## 10. Frozen test-set run (2026-10-27)

- Run: `test_final`
- Prompt hash: `d054177e`
- Model: `gemini-3.5-flash-lite`
- Dataset: `data/test_set.csv` (50 rows)

**The test labels were not independently checked before this run.** Treat these
metrics as provisional until the independent label review in section 8 is complete.

| Metric | Result |
|---|---:|
| Category accuracy | 100% (50/50) |
| Priority exact accuracy | 62% (31/50) |
| Priority within one level | 100% (50/50) |
| Emergency recall | 8/10 |
| Priority exact after safety floor | 64% (32/50) |
| Emergency recall after safety floor | 9/10 |
| Safety-floor false alarms | 0 |

### Priority confusion matrix (label -> predicted)

| Label \ Predicted | Low | Medium | High | Emergency |
|---|---:|---:|---:|---:|
| Low | 8 | 5 | 0 | 0 |
| Medium | 2 | 7 | 3 | 0 |
| High | 0 | 4 | 8 | 3 |
| Emergency | 0 | 0 | 2 | 8 |

The safety floor raised T011 (broken manhole lid with exposed iron rod) from
high to emergency, matching its test label. It raised no non-emergency row.