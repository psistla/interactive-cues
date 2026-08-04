# Automated Testing Options: Fabric Medallion Implementation

Scoped to the stated landscape: bronze shortcuts, thin silver, gold facts and dimensions built by MERGE, notebook-based transformations with inline logic, two Dataflow Gen2 tables, manual DAX validation.

---

## 0. Constraints this document is built on

| Input | Stated | Consequence for design |
|---|---|---|
| Git integration | Not done, manual artifact promotion | No CI gate exists. Everything in Phase 1 must run inside Fabric. Unit tests are not available until Git lands. |
| Code structure | Purely inline notebook cells | Unit testing requires refactor first. Runtime assertions are the only near-term option. |
| PyPI packages | Allowed | Third-party assertion libraries and `sempy_labs` are viable. |
| Failure behavior | Block | Orchestration must halt on blocking severity, and you need a quarantine path so a single bad partition does not stop the whole load. |
| Environments | Dev and test exist, validation happens in prod | Highest-severity process risk in the list. Section 7. |
| Results | Delta results table plus quality report | Single results schema, one report, all layers. Section 5. |
| Scale | Smaller than enterprise default | Assumed 50 to 200 tables, under 10TB, F2 to F16. Design holds to roughly 500 tables; past that, assertion runtime and CU cost need partition-scoped checks rather than full-table scans. Tell me if this assumption is wrong. |
| Dataflows Gen2 | Migration negotiable | Recommended. Section 4, Option I. |

**Runtime target:** Fabric Runtime 1.3 (Spark 3.5, Delta Lake 3.2, Python 3.11). *Verified [2026-06-02, https://learn.microsoft.com/en-us/fabric/data-engineering/lifecycle]*: Runtime 1.3 is the current GA runtime and enters LTS on 2026-10-01 through March 2027. Runtime 2.0 (Spark 4.1, Delta Lake 4.2) is public preview only, so all code below targets 1.3. *Verified [2026-05-20, https://learn.microsoft.com/en-us/fabric/data-engineering/runtime-2-0]*: Delta Lake 4.x features under Runtime 2.0 are experimental and only work in Spark experiences, which breaks cross-workload table reads. Do not put test infrastructure there.

---

## 1. The gate model

Testing splits along one axis: what can be evaluated without data, and what can only be evaluated against data. Your current setup has zero of the first and ad hoc instances of the second.

| Gate | What it catches | Where it runs | Available to you today |
|---|---|---|---|
| G0 Static analysis | Syntax, style, dead code, hardcoded secrets | Build agent | No (needs Git) |
| G1 Unit tests | Transformation logic errors, edge case handling | Build agent, local Spark | No (needs Git plus refactor) |
| G2 Source contracts | Upstream schema drift, freshness breaks, volume anomalies on shortcut-backed bronze | Fabric notebook, pre-load | **Yes** |
| G3 Transformation integration | Silver and gold output correctness, row-level reconciliation | Fabric notebook, post-write | **Yes** |
| G4 Merge invariants | Key cardinality, idempotency, SCD correctness, orphan foreign keys | Fabric notebook, pre and post MERGE | **Yes** |
| G5 Semantic model | DAX regressions, RLS breaks, model rule violations, measure-to-source drift | Fabric notebook, pre-publish | **Yes** |
| G6 Post-publish monitoring | Drift, silent freshness failures | Scheduled notebook plus alerting | Yes |

G2, G3, G4, G5 need no Git and no refactor. That is where the entire near-term return is.

**The correction to make first:** your existing duplicate checks are inline assertions, not tests. No persisted result, no history, no severity classification, no gate. Converting them into G3/G4 checks with a results table costs almost nothing and is the prerequisite for everything else.

---

## 2. Option catalog

### A. Hand-rolled assertion engine in notebooks
Config-driven checks (SQL expressions plus a few Python callables) writing to a Delta results table, with severity-based blocking.

- Build cost: 2 to 4 days for the engine, then roughly 15 minutes per table to configure checks.
- Prerequisites: none beyond what you have.
- Strengths: zero dependency risk, full control of severity and gating semantics, results schema exactly matches your reporting need, runs on Runtime 1.3 with no package pinning.
- Weaknesses: you own it. No community expectation library, no profiling, no anomaly detection out of the box.
- **Verdict: start here.** At your scale the engine is small and the alternatives all impose either a config language or a governance surface you do not need yet.

### B. Great Expectations
Expectation suites, data docs, a large built-in expectation library.

- Build cost: 1 to 2 weeks including version pinning pain.
- Strengths: broad expectation library, profiling, HTML docs.
- Weaknesses: heavy dependency tree, Spark integration has historically been the most brittle part of the stack, and the 0.x to 1.x API break invalidates most published Fabric examples. *Unverified working hypothesis*: current GX Core on Spark 3.5 under Fabric Runtime 1.3 installs cleanly; test in an Environment item before committing.
- **Verdict: overkill at your scale.** The expectation library is the only real gain and you would use maybe fifteen of them.

### C. Soda Core
YAML-based checks (SodaCL), lighter than GX.

- Build cost: 3 to 5 days.
- Strengths: readable check definitions that a non-engineer can review, reasonable Spark support, much lighter than GX.
- Weaknesses: another config language to learn, and the open source tier keeps losing surface to the hosted product. Gating still has to be wired by hand.
- **Verdict: viable alternative to A** if you want checks defined by analysts rather than engineers. Otherwise A is simpler.

### D. Purview Unified Catalog data quality
No-code and low-code rules, quality scoring, catalog integration.

- <cite index="4-1">Setup requires registering Fabric as a source in Data Map and running a scan; service principal is currently the only supported authentication method for the scan, with MSI support still in backlog.</cite> <cite index="5-1">Guidance is to use a service principal for Data Map scans and a managed identity for the data quality scans themselves.</cite>
- <cite index="4-1">New tables added via shortcut or mirroring require a fresh Data Map scope scan before they can be added to a data product and evaluated.</cite> That is a real operational tax on a shortcut-heavy bronze layer.
- Feature currency: *Unverified, single vendor-blog source* that custom SQL expression rules went GA in March 2026 and incremental scanning entered preview in February 2026. Confirm on Microsoft Learn before planning around either.
- **Verdict: not a gate.** Purview DQ is asynchronous, catalog-oriented observability. It cannot block a pipeline. Adopt it later for governance-level scoring if SOC2-class reporting demands it, not as your testing framework.

### E. Delta CHECK constraints and table properties
Structural invariants enforced by the writer itself: nullability, value ranges, `delta.appendOnly`.

- Build cost: hours.
- Strengths: cannot be bypassed, zero runtime cost, fails the write rather than detecting after the fact.
- Weaknesses: *Training-era, last known Delta 3.2*: adding CHECK constraints raises the table's `minWriterVersion` (writer protocol 3). Readers are unaffected, but validate that your Dataflow Gen2 writes and any non-Spark writer still succeed against a constrained table before rolling this out broadly. Constraints also cannot express cross-row or cross-table conditions, so they complement rather than replace A.
- **Verdict: adopt for gold dimension and fact key columns.** Cheapest correctness win available.

### F. pytest plus local Spark on a build agent
Unit tests against extracted transformation functions.

- Build cost: the refactor dominates. Engine setup is a day; extracting inline notebook code into a testable module is the real project.
- Prerequisites: Git integration, a build agent, and the refactor.
- <cite index="17-1">The pattern relies on Fabric syncing notebooks to Git as near-interoperable Python files, then running Ruff and pytest against a local single-node Spark session on the build agent, which validates transformation and schema logic without consuming Fabric capacity.</cite>
- Pin the agent to the Fabric runtime: `pyspark==3.5.*`, `delta-spark==3.2.*`, Python 3.11. Version skew between agent and capacity produces tests that pass locally and fail in Fabric.
- **Verdict: Phase 3.** Highest long-term value, blocked on two prerequisites you do not have.

### G. pytestmsfabric
Community pytest plugin for Fabric notebooks. <cite index="18-1">Supports running tests in CI via a Docker container or via a Fabric workspace.</cite>

- **Verdict: evaluate, do not depend on.** Version 0.3.x, single maintainer, no support model. Read the source before it becomes load-bearing.

### H. Semantic model test harness (sempy plus semantic-link-labs)
Replaces your manual DAX validation.

- <cite index="10-1">sempy ships preinstalled in every Fabric notebook runtime; semantic-link-labs must be installed explicitly.</cite> <cite index="11-1">Scheduled DAX queries against a published model are a supported automated testing pattern.</cite>
- Three test classes worth building: (1) measure result assertions against known-good expected values, (2) reconciliation of key measures back to the gold Delta tables via SQL analytics endpoint, (3) Best Practice Analyzer as a non-blocking quality score.
- RLS testing: <cite index="9-1">a current pattern stores DAX Query View test functions as TMDL in a .pbip project and runs them with impersonation via semantic-link-labs, which allows validating RLS rules.</cite> That approach depends on Power BI user-defined functions, *which were still a preview feature as of March 2026 per that same source and which I have not re-verified*. Treat UDF-based test authoring as optional; plain `evaluate_dax` calls cover most of the value without the preview dependency.
- **Verdict: highest immediate return per hour spent,** because it replaces recurring manual labor rather than adding a new activity.

### I. Dataflow Gen2 handling
No unit test surface, no local execution, opaque M. Testing is limited to asserting on the output table after the fact, which means you detect failures but cannot localize them.

- **Verdict: migrate both tables to notebooks.** You already said negotiable. Two tables is a small enough surface that keeping a second, untestable execution engine is not worth the coverage hole. Do it before Phase 2 rather than after, so the Git migration only has to handle one artifact type.

### J. data-factory-testing-framework
Pipeline expression and activity-flow testing. <cite index="21-1">Installed alongside pytest and driven from an Azure DevOps pipeline stage.</cite>

- **Verdict: only if you have Data Factory pipelines doing conditional logic.** You did not mention pipelines. If orchestration is notebook-driven, skip.

---

## 3. Recommended sequence

### Phase 1: Runtime gates, no prerequisites (weeks 1 to 3)
1. Results table plus assertion engine (Option A). Section 5.
2. Port existing inline duplicate checks into it. No new logic, just structure.
3. G4 merge invariants on every gold table. Section 6. This is where your real exposure is.
4. Delta CHECK constraints on gold keys (Option E).
5. Blocking wired into orchestration with a quarantine path.

**Validation gate:** every gold table has at least a key uniqueness check, a null check on business keys, and a post-merge row count reconciliation. A deliberately corrupted source in dev produces a blocked run and a results row.

### Phase 2: Source contracts and semantic model (weeks 3 to 6)
6. G2 contracts on bronze shortcuts: schema fingerprint comparison against a stored baseline, freshness, row count delta bands.
7. G5 semantic model harness (Option H) replacing manual DAX checks.
8. Quality report over the results table.

**Validation gate:** a schema change in an upstream source is detected before silver runs, not after gold breaks. A deliberately broken measure is caught by the harness before publish.

### Phase 3: Migrate Dataflows and land Git (weeks 6 to 10)
9. Rebuild the two Dataflow Gen2 tables as notebooks (Option I).
10. Git integration, then deployment pipelines or `fab deploy` to replace manual promotion.

**Validation gate:** promotion of an artifact from dev to test happens without a human clicking through the portal.

### Phase 4: Refactor and unit tests (weeks 10+)
11. Extract inline transformation logic into a custom library, packaged as a wheel on an Environment item.
12. pytest plus local Spark in CI (Option F), plus Ruff for G0.

**Validation gate:** a pull request with a logic regression fails the build before any Fabric capacity is consumed.

---

## 4. Blocking semantics

You chose block. Three details determine whether that is safe or whether it turns into a 3am incident.

**Severity tiers.** Not every failure should halt.

| Severity | Example | Behavior |
|---|---|---|
| `blocking` | Duplicate business keys in a MERGE source, null in a dimension key, gold row count deviating more than the configured band | Halt downstream, do not publish |
| `quarantine` | Individual rows failing a business rule | Route rows to a reject table, continue with the valid remainder, record the count |
| `warning` | Freshness late but within tolerance, distribution shift | Log, alert, continue |

Hard-blocking on row-level rules is what turns testing frameworks into things teams disable. Quarantine is the release valve.

**Where the block happens.** Because promotion is manual today, the block must be enforced by orchestration, not by process. `notebookutils.notebook.exit()` returns a value the caller inspects; raising an unhandled exception fails the notebook activity outright. Use the exception for blocking and the exit value for structured results, so the failure surfaces in run history rather than only in your results table. *Training-era, last known: `notebookutils` on Runtime 1.3; `mssparkutils` is the deprecated predecessor.*

**Results are written before the block.** Persist first, then raise. A blocking failure that leaves no results row is a failure you have to reproduce manually.

---

## 5. Results schema and engine

### Results table

```python
# Run once. Fabric Runtime 1.3, Spark 3.5, Delta 3.2.
spark.sql("""
CREATE TABLE IF NOT EXISTS quality.check_results (
    run_id            STRING  NOT NULL,
    check_id          STRING  NOT NULL,
    check_name        STRING  NOT NULL,
    layer             STRING  NOT NULL,   -- bronze | silver | gold | semantic
    target_table      STRING  NOT NULL,
    check_type        STRING  NOT NULL,   -- uniqueness | completeness | referential | volume | freshness | reconciliation | custom
    severity          STRING  NOT NULL,   -- blocking | quarantine | warning
    status            STRING  NOT NULL,   -- pass | fail | error | skipped
    observed_value    DOUBLE,
    threshold_value   DOUBLE,
    failed_row_count  BIGINT,
    detail            STRING,
    duration_seconds  DOUBLE,
    executed_at       TIMESTAMP NOT NULL,
    executed_by       STRING
)
USING DELTA
PARTITIONED BY (layer)
""")
```

Partitioning by `layer` rather than date keeps partition count low at your scale while still giving the report a cheap filter. If run volume grows past roughly a million rows per month, repartition by `executed_at` month instead.

### Check configuration

Externalized, not embedded. One JSON file per layer in the lakehouse `Files` area.

```json
{
  "layer": "gold",
  "checks": [
    {
      "check_id": "gold.fact_sales.key_uniqueness",
      "check_name": "fact_sales business key is unique",
      "target_table": "gold.fact_sales",
      "check_type": "uniqueness",
      "severity": "blocking",
      "sql": "SELECT COUNT(*) AS failed FROM (SELECT order_id, line_no FROM gold.fact_sales GROUP BY order_id, line_no HAVING COUNT(*) > 1)",
      "threshold": 0
    },
    {
      "check_id": "gold.fact_sales.dim_customer_fk",
      "check_name": "fact_sales customer key resolves",
      "target_table": "gold.fact_sales",
      "check_type": "referential",
      "severity": "blocking",
      "sql": "SELECT COUNT(*) AS failed FROM gold.fact_sales f LEFT ANTI JOIN gold.dim_customer d ON f.customer_sk = d.customer_sk",
      "threshold": 0
    },
    {
      "check_id": "gold.fact_sales.volume_band",
      "check_name": "fact_sales daily volume within band",
      "target_table": "gold.fact_sales",
      "check_type": "volume",
      "severity": "warning",
      "sql": "SELECT COUNT(*) AS failed FROM gold.fact_sales WHERE load_date = CURRENT_DATE()",
      "min_value": 1000,
      "max_value": 500000
    }
  ]
}
```

Every check returns a single numeric column. `threshold` means "fail if greater than", `min_value`/`max_value` mean "fail if outside the band". Two evaluation modes, no expression language to maintain.

### Engine

Deploy as a notebook named `nb_quality_engine`, consumed by other notebooks with `%run nb_quality_engine`. When Phase 4 lands, this becomes a module in the wheel with no logic change.

```python
# nb_quality_engine
# Fabric Runtime 1.3 (Spark 3.5, Python 3.11, Delta 3.2)
import json
import time
import uuid
from datetime import datetime, timezone
from typing import Any

from pyspark.sql import Row
from pyspark.errors import AnalysisException

RESULTS_TABLE = "quality.check_results"
BLOCKING = "blocking"


class QualityGateFailure(Exception):
    """Raised when one or more blocking checks fail. Fails the notebook activity."""


def _load_config(config_path: str) -> list[dict[str, Any]]:
    # config_path is an abfss:// or Files/ path passed in by the caller, never hardcoded
    raw = spark.read.text(config_path, wholetext=True).collect()[0][0]
    return json.loads(raw)["checks"]


def _evaluate(check: dict[str, Any]) -> tuple[str, float, float | None, str]:
    """Returns (status, observed, threshold, detail)."""
    observed = float(spark.sql(check["sql"]).collect()[0][0])

    if "threshold" in check:
        threshold = float(check["threshold"])
        failed = observed > threshold
        detail = f"observed={observed} threshold={threshold}"
    else:
        lo, hi = float(check["min_value"]), float(check["max_value"])
        threshold = None
        failed = not (lo <= observed <= hi)
        detail = f"observed={observed} band=[{lo}, {hi}]"

    return ("fail" if failed else "pass"), observed, threshold, detail


def run_checks(config_path: str, executed_by: str) -> str:
    """Run every check in the config, persist results, then raise if any blocking check failed."""
    checks = _load_config(config_path)
    run_id = str(uuid.uuid4())
    now = datetime.now(timezone.utc)
    rows: list[Row] = []
    blocking_failures: list[str] = []

    for check in checks:
        started = time.perf_counter()
        status, observed, threshold, detail = "error", None, None, ""
        try:
            status, observed, threshold, detail = _evaluate(check)
        except AnalysisException as exc:
            detail = f"sql resolution failed: {exc}"
        except (ValueError, TypeError, IndexError) as exc:
            detail = f"result shape invalid, expected one numeric column: {exc}"

        if status in ("fail", "error") and check["severity"] == BLOCKING:
            blocking_failures.append(f'{check["check_id"]}: {detail}')

        rows.append(Row(
            run_id=run_id,
            check_id=check["check_id"],
            check_name=check["check_name"],
            layer=check.get("layer", "unknown"),
            target_table=check["target_table"],
            check_type=check["check_type"],
            severity=check["severity"],
            status=status,
            observed_value=observed,
            threshold_value=threshold,
            failed_row_count=int(observed) if observed is not None and check["check_type"] != "volume" else None,
            detail=detail,
            duration_seconds=round(time.perf_counter() - started, 3),
            executed_at=now,
            executed_by=executed_by,
        ))

    # Persist before blocking. A gate that leaves no evidence is not a gate.
    spark.createDataFrame(rows).write.mode("append").saveAsTable(RESULTS_TABLE)

    if blocking_failures:
        raise QualityGateFailure(
            f"{len(blocking_failures)} blocking check(s) failed in run {run_id}:\n"
            + "\n".join(blocking_failures)
        )
    return run_id
```

Note the `except` clauses are typed. An unexpected exception class propagates and fails the run loudly, which is correct: a broken engine must not report green.

---

## 6. Merge invariants

This is the gap your existing duplicate checks do not cover, and it is where gold silently corrupts.

```python
# nb_merge_guards
from delta.tables import DeltaTable
from pyspark.sql import DataFrame, functions as F


def assert_merge_key_unique(source: DataFrame, keys: list[str]) -> None:
    """MERGE fails outright when multiple source rows match one target row.
    Upstream dedupe on all columns does not guarantee uniqueness on the merge key."""
    dupes = source.groupBy(*keys).count().filter(F.col("count") > 1)
    offending = dupes.limit(5).collect()
    if offending:
        raise QualityGateFailure(
            f"merge key {keys} not unique in source. Sample: {[r.asDict() for r in offending]}"
        )


def assert_no_null_keys(source: DataFrame, keys: list[str]) -> None:
    """NULL merge keys never match and silently insert duplicates on every run."""
    condition = F.lit(False)
    for key in keys:
        condition = condition | F.col(key).isNull()
    null_count = source.filter(condition).count()
    if null_count:
        raise QualityGateFailure(f"{null_count} source rows have NULL in merge key {keys}")


def assert_idempotent(table_name: str, merge_fn, source: DataFrame) -> None:
    """Run the merge twice against the same source. State must be identical.
    Dev and test only: this writes twice."""
    merge_fn(source)
    first = spark.table(table_name)
    first_count = first.count()
    first_hash = first.selectExpr("SUM(xxhash64(*)) AS h").collect()[0]["h"]

    merge_fn(source)
    second = spark.table(table_name)
    if second.count() != first_count:
        raise QualityGateFailure(
            f"{table_name} not idempotent: row count {first_count} then {second.count()}"
        )
    second_hash = second.selectExpr("SUM(xxhash64(*)) AS h").collect()[0]["h"]
    if second_hash != first_hash:
        raise QualityGateFailure(f"{table_name} not idempotent: content changed on replay")


def assert_history_advanced(table_name: str, expected_ops: set[str]) -> None:
    """Confirms the last Delta operation was the one you intended.
    Catches a silently skipped merge that left the table stale."""
    history = DeltaTable.forName(spark, table_name).history(1).collect()
    if not history:
        raise QualityGateFailure(f"{table_name} has no Delta history")
    op = history[0]["operation"]
    if op not in expected_ops:
        raise QualityGateFailure(f"{table_name} last operation was {op}, expected one of {expected_ops}")
```

Two gotchas worth stating explicitly because they are not obvious from the MERGE failure message:

- `dropDuplicates()` treats NULL as equal to NULL, so a dedupe pass can leave rows that then behave as distinct keys in the MERGE join, where NULL never matches. Deduplicate and null-check separately.
- Idempotency testing writes twice. Never run `assert_idempotent` against production. This is one of several reasons Section 7 matters.

---

## 7. Risks in the current setup

**Ranked blocking, degrading, technical debt.**

**Blocking: validation happens in production.** You have dev and test environments and are not using them. Every consequence in this document changes under that condition. Idempotency tests cannot run. A blocking gate that fires in production is an outage rather than a caught defect, which is exactly backwards; the point of a gate is to fail somewhere cheap. Before Phase 1 completes, move validation to test with a representative data subset. This is a process change, not a tooling one, and no framework compensates for it.

**Blocking: blocking gates without Git means no rollback.** Manual artifact promotion means that when a gate correctly blocks a bad load, your recovery path is a human reconstructing the previous notebook version from memory. Delta time travel gets the data back; nothing gets the code back. Either accept warning-only severity until Phase 3 lands, or move Git integration ahead of the blocking gates in the sequence. Given you chose block, I would move Git to Phase 2.

**Degrading: inline code means tests duplicate logic.** Any assertion you write against inline transformation code restates the logic in a second place. When the notebook changes, the check silently becomes wrong rather than failing. This is tolerable for structural checks (uniqueness, nullability, referential integrity) because those assert on invariants rather than logic. It is not tolerable for business rule checks. Keep Phase 1 and 2 checks structural, and defer business logic assertions until the refactor.

**Degrading: bronze shortcuts are unowned.** Schema drift arrives without notice and without a deploy. Store a schema fingerprint per bronze table and compare on every run:

```python
import hashlib

def schema_fingerprint(table_name: str) -> str:
    fields = sorted(f"{f.name}:{f.dataType.simpleString()}" for f in spark.table(table_name).schema.fields)
    return hashlib.sha256("|".join(fields).encode()).hexdigest()
```

Compare against a stored baseline table. Additive changes are warnings; type changes and removals are blocking.

**Technical debt: CU cost of checks.** Full-table `COUNT(*)` on every gold table on every run is cheap at under 10TB and expensive above it. Scope volume and uniqueness checks to the loaded partition where possible. Budget roughly one to three percent of pipeline CU for assertions at your scale; measure it in the Capacity Metrics app after Phase 1 rather than trusting that estimate.

**Technical debt: no test data management.** Representative test data is the unglamorous prerequisite that stalls most of these programs. A sampled, key-preserving subset of production, refreshed monthly, is enough. Plan for it in Phase 1 rather than discovering it in Phase 4.

---

## 8. Semantic model harness

Replaces the manual DAX pass. Runs before publish, blocks on failure.

```python
# nb_semantic_tests
%pip install semantic-link-labs

import json
import sempy.fabric as fabric
import sempy_labs as labs


def run_dax_tests(dataset: str, workspace: str, test_config_path: str) -> None:
    """Each test: a DAX query returning one row, one column, plus an expected value and tolerance."""
    with open(test_config_path, "r", encoding="utf-8") as handle:
        tests = json.load(handle)["tests"]

    failures: list[str] = []
    for test in tests:
        try:
            result = fabric.evaluate_dax(
                dataset=dataset,
                dax_string=test["dax"],
                workspace=workspace,
            )
            actual = float(result.iloc[0, 0])
        except Exception as exc:  # sempy surfaces XMLA errors as generic exceptions
            failures.append(f'{test["name"]}: execution failed: {exc}')
            continue

        expected = float(test["expected"])
        tolerance = float(test.get("tolerance", 0))
        if abs(actual - expected) > tolerance:
            failures.append(f'{test["name"]}: expected {expected} +/- {tolerance}, got {actual}')

    if failures:
        raise QualityGateFailure("Semantic model tests failed:\n" + "\n".join(failures))
```

The `except Exception` here is deliberate and is the one place I would allow it: sempy wraps XMLA and .NET errors without a stable public exception hierarchy. *Training-era, last known: sempy exception surface as of early 2026.* Narrow it if you find the concrete types in your runtime.

Three test categories to populate it with:

1. **Reconciliation.** Same measure computed in DAX and in SQL against the gold table. Expected value is the SQL result, computed at runtime rather than hardcoded. Catches DirectLake fallback issues, relationship errors, and filter direction mistakes.
2. **Known-good anchors.** Hardcoded expected values for a frozen historical period that should never change. Catches silent restatement of history by an upstream merge.
3. **Model rules.** `labs.run_model_bpa(dataset=..., workspace=...)` as a warning-severity score, not a gate. *Training-era, last known API surface; verify against the installed `semantic-link-labs` version.*

RLS is the coverage gap this does not close. If RLS exists on these models, the impersonation-based approach referenced in Section 2H is the current option, with the caveat that it depends on a preview feature. Otherwise RLS stays manual for now, which is worth naming explicitly in your test coverage documentation rather than leaving implicit.

---

## 9. Verification status of claims in this document

| Claim | Status |
|---|---|
| Runtime 1.3 is current GA; 2.0 is public preview | Verified 2026-06-02, learn.microsoft.com/en-us/fabric/data-engineering/lifecycle |
| Delta 4.x features under Runtime 2.0 are Spark-experience only | Verified 2026-05-20, learn.microsoft.com/en-us/fabric/data-engineering/runtime-2-0 |
| Purview DQ on Fabric Lakehouse requires SPN for Data Map scan, MSI in backlog | Verified 2026-05-27, learn.microsoft.com/en-us/purview/unified-catalog-data-quality-fabric-lakehouse |
| Purview custom SQL rules GA March 2026, incremental scanning preview Feb 2026 | Unverified, single vendor blog. Confirm on Learn. |
| sempy preinstalled, semantic-link-labs requires install | Verified via community and vendor sources, April 2026 |
| Power BI UDF-based test functions still preview | Unverified as of this writing; source stated preview in March 2026 |
| CHECK constraints raise minWriterVersion and may affect non-Spark writers | Unverified working hypothesis. Test before rollout. |
| `notebookutils` API surface, `labs.run_model_bpa` signature | Training-era, last known early 2026 |
| Assertion CU overhead of one to three percent | Unverified estimate. Measure in Capacity Metrics after Phase 1. |
| GX Core installs cleanly on Runtime 1.3 | Unverified working hypothesis |

All code targets Fabric Runtime 1.3 (Spark 3.5, Delta Lake 3.2, Python 3.11) and has not been executed against a live capacity.
