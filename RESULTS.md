# Mining Telemetry Pipeline: Production Failure Handling

A data engineering pipeline that ingests haul-truck telemetry from a simulated mine site and stays correct under real-world production failures. Built on Databricks with Auto Loader and Delta, using a medallion architecture (bronze, silver, gold).

**The core idea:** a data generator produces dirty telemetry and, separately, a "ground-truth" ledger recording what each truck actually hauled. The pipeline never sees the ledger; it's used only to reconcile the final report against reality. Correctness is proven with a number, not assumed.

**Fleet:** 8 haul trucks (4 Komatsu 777 at ~90t, 4 Komatsu 785 at ~140t), 35 cycles per shift, 5 shifts. Tonnage is validated against model-specific physical limits.

---

## Scenario 1: Impossible tonnage (sensor fault)

**Failure:** an onboard scale miscalibrates and reports a physically impossible load, such as 15,000t on a truck that carries 140t. A units or decimal glitch.

**Why it's dangerous:** the value is a valid number and passes type checks, so a naive pipeline ingests it and silently inflates production totals.

**Fix:** model-aware validation in silver. Each reading is checked against its truck model's physical bounds (777: 15 to 130t, 785: 25 to 195t). Impossible readings are routed to a quarantine table, never dropped and never trusted.

**Proof:** 21 impossible readings caught and isolated in quarantine; none reached the production report.

![Impossible tonnages caught in quarantine](portfolioproof/scenario1_impossible_tonnage.png)

**Insight:** validation has to be domain-aware. A valid number is not a valid tonnage. The quarantine keeps bad data visible for investigation instead of silently dropping or trusting it.

---

## Scenario 2: Duplicate records (network retry)

**Failure:** a truck transmits a reading, the network drops the acknowledgment, and the device re-sends. The same haul lands 2 to 3 times.

**Why it's dangerous:** every duplicate is individually valid, so validation can't catch it. Duplicates silently inflate totals with phantom production.

**Fix:** deduplicate in silver on the natural key `cycle_id`, keeping one row per haul. Bronze retains the duplicates as a faithful raw record.

**Before:** gold overstated by up to 600+ tonnes per truck-shift (gold over truth). **After:** the inflation is gone; remaining gaps are only the honest hauls quarantined by Scenario 1 (gold slightly under truth).

![Before: duplicates inflate totals](portfolioproof/scenario2_duplicates_before.png)
![After: dedup removes the inflation](portfolioproof/scenario2_duplicates_after.png)

**Insight:** deduplication is a different operation from validation. It looks across rows, not at a row in isolation. The direction of the reconciliation gap diagnoses the failure: over means duplicates, under means dropped data.

---

## Scenario 3: Schema drift (firmware rollout)

A firmware update rolls out to a pilot group of trucks (the 785s), so old-format and new-format telemetry arrive in the same batch. Three variants.

### 3A: Additive (new column)

**Failure:** updated trucks emit an extra `firmware_version` field.

**Fix:** none needed. Auto Loader's `addNewColumns` and `mergeSchema` auto-evolve the schema. Old rows get null, new rows carry the value.

**Insight:** additive change is the safe drift, and auto-evolution is the correct stance. The right response is to let it happen.

### 3B: Rename (column renamed)

**Failure:** updated trucks send `payload_tonnes` instead of `reported_tonnage`.

**Why it's dangerous:** the pipeline reads the old name, finds null for the pilot trucks, and quarantines them all. Four of eight trucks silently vanish, with no error. Worse, they don't even appear in the reconciliation mismatch, so the report looks cleaner than reality.

**Fix:** coalesce the old and new field names into one unified column before validation.

**Before:** silver contains only T01 to T04. **After:** all 8 trucks recovered.

![Before: 4 of 8 trucks vanish](portfolioproof/scenario3b_rename_before.png)
![After: all 8 trucks recovered](portfolioproof/scenario3b_rename_after.png)

**Insight:** a rename must never auto-evolve. That just adds the new column while your logic still reads the dead one, producing silent nulls. Defend by mapping names explicitly.

### 3C: Type change (string with unit)

**Failure:** updated trucks send tonnage as `"153.8 t"` (a string with a unit) instead of a number.

**Why it's dangerous:** the safe cast returns null on the unparseable string, so pilot trucks are quarantined and vanish. Same result as the rename, different cause.

**Fix:** strip non-numeric characters with `regexp_replace` before casting.

**Before:** T01 to T04 only. **After:** all 8 trucks recovered.

![Before: 4 of 8 trucks vanish](portfolioproof/scenario3c_typechange_before.png)
![After: all 8 trucks recovered](portfolioproof/scenario3c_typechange_after.png)

**Insight:** type and format drift need defensive parsing before casting. Any unhandled schema drift silently erases data.

---

## What this demonstrates

- Correctness proven by reconciliation against ground truth, not assumed.
- Bronze stays raw as a faithful record; silver cleans; gold serves the business.
- Each failure is handled with the right stance (quarantine, dedup, coalesce, parse, or deliberately allow), not a blanket rule.
- The reconciliation gap is itself a diagnostic signal: its direction and size point to which trucks failed and how.
