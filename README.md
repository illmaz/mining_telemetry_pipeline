# Mining Telemetry Pipeline

A pipeline that ingests haul-truck telemetry from a simulated gold mine and keeps producing correct production numbers even when the incoming data is broken.

## Why this project

Before switching careers I worked in a gold mine, in the pit, surrounded by haul trucks running ore out of the pit and dumping it. I worked alongside the geologists and saw the data side of it up close. When I started building portfolio projects I wanted to build something I'd actually seen, rather than another taxi dataset.

Pit telemetry is messy in specific ways. Trucks lose signal in the pit and buffer readings. Scales drift out of calibration. Firmware updates roll out to half the fleet at a time. Instead of working with clean data I deliberately injected failures and solved them. It's built around the assumption that the data will be wrong, because on a real site it is.

## The idea

The generator writes two things. One is the dirty telemetry the pipeline ingests, with realistic faults injected. The other is a ground-truth ledger of what each truck actually hauled, which the pipeline is never allowed to read.

At the end I reconcile the pipeline's report against that ledger. Matching numbers mean the pipeline is provably correct. A gap tells me which truck is off and by how much. 

## The pipeline

Telemetry lands as raw JSON and is picked up incrementally by Auto Loader into a bronze table, which stays untouched so there's always a faithful record of what arrived. Silver does the cleaning: validating readings against each truck model's physical payload limits, deduplicating on the cycle key, and handling schema changes. Anything that fails validation goes to a quarantine table rather than being dropped or trusted. Gold aggregates to tonnage per truck per shift, which is the number a site actually cares about.

## What breaks, and how it's handled

Each failure is injected deliberately, with before-and-after reconciliation to prove the fix. Full evidence and screenshots are in [RESULTS.md](RESULTS.md).

A miscalibrated scale reporting 15,000 tonnes on a truck rated for 140. It's a valid number, so type checks pass. Validation catches it against the model's real limits.

Duplicate readings from network retries. Every copy is individually valid, so validation is useless here. They're deduplicated on the natural key instead.

Schema drift from a firmware rollout, which was the worst of them. When the update renamed the payload column on half the fleet, four of eight trucks silently disappeared from the report with no error anywhere. The reconciliation looked cleaner than reality because the missing trucks weren't there to mismatch. The fix resolves both column names before anything downstream runs.

Also handled: tonnage arriving as `"153.8 t"` instead of a number, and a new column appearing mid-stream, which is the one drift that's safe to just accept.

The direction of the reconciliation gap turned out to be diagnostic on its own. Reading higher than truth means double counting. Lower means something got dropped.

## Running it

The full chain runs as a Lakeflow job with task dependencies, so a failure stops the run rather than letting downstream steps work on bad data. The job is defined as code with a Databricks Asset Bundle and deploys from the command line.

## Stack

Databricks, PySpark, Delta Lake, Auto Loader, Lakeflow Jobs, Asset Bundles.
