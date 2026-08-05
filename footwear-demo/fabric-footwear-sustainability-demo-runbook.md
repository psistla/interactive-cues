# Footwear Sustainability — Fabric End-to-End Demo Runbook

**v2 · 2026-07-20** — every number below is measured against the actual CSVs, not estimated.

| | |
|---|---|
| **Dataset (raw)** | `C:\claude-projects\datasets\out\footwear-sustainability-star` — 7 CSVs, 38k rows, 4.4 MB |
| **Dataset (clean)** | `…\footwear-sustainability-star\clean\` — same star, all defects fixed, Power BI-ready |
| **Format** | Recorded / self-paced asset, chaptered |
| **Audience** | Mixed exec + technical ("art of the possible"), doubling as a portfolio piece |
| **Storyline** | Carbon per revenue dollar → supplier & factory accountability |
| **Fabric items** | Lakehouse · Notebook · Semantic model (Direct Lake) · Report · Ontology (preview) · Data agent |
| **Runtime** | ~22 min across 6 chapters |
| **Build effort** | 6–9 hrs first pass, then rebuild clean for the recording |

---

## 0. Pick your path first

Two versions of this demo exist, and the choice changes Chapter 2 only.

| | **Path A — raw CSVs + notebook** | **Path B — clean CSVs** |
|---|---|---|
| Source | `…\footwear-sustainability-star\*.csv` | `…\footwear-sustainability-star\clean\*.csv` |
| Chapter 2 | Notebook fixes the data on camera | Upload and go |
| Runtime | 22 min | ~18 min |
| Setup | ~2 hrs | ~20 min |
| You keep | The asserts going green, the flat-to-sloped reveal, "silver earns its keep" | Nothing extra |
| You lose | — | The single strongest technical beat |

**Record Path A.** The notebook chapter is what separates this from a Power BI tutorial that anyone could give. Keep the clean CSVs for rehearsal, for a Power BI Desktop-only cut, and as your fallback if Spark misbehaves on the day.

Everything below is written for Path A. If you take Path B, skip §3 entirely, load the 7 clean CSVs into the lakehouse, and start at §4 — the table names change from `gold_*` to plain `dim_*` / `fact_order_line`, and nothing else moves.

---

## 1. Planning the demo

### 1.1 Who you're actually talking to

You picked mixed exec+tech *and* portfolio. Those pull in opposite directions — execs want the story, architects want the plumbing, a portfolio asset wants to be re-watchable. Resolve it with **chapters, not compromise**: every chapter opens with 20 seconds of "why this matters", then goes hands-on. A viewer who only wants the story watches 1, 4, 6. Chapter titles carry that promise.

### 1.2 The through-line

One sentence, said at 00:30 and again at the close:

> **"The same governed model that answers a report also answers an agent — and it answers both with the same number."**

The carbon-intensity story is the vehicle; the punchline is *one semantic layer, three consumers* (report, ontology, agent). That's the differentiated part — anyone can demo a Power BI report.

### 1.3 The 22 minutes

| # | Chapter | Min | Items | The beat |
|---|---|---:|---|---|
| 1 | The question nobody can answer | 2 | — | Show the 7 CSVs. "Which suppliers are we paying to pollute?" Four tabs of Excel and no answer. |
| 2 | Land it and make it mean something | 4 | Lakehouse, Notebook | Upload → Delta. Silver fixes region; gold repairs the date FK, resolves SCD2, shapes the signal. End on the asserts going green. |
| 3 | One model, many answers | 4 | Semantic model | Direct Lake, relationships, 9 measures. Carbon intensity defined **once**. |
| 4 | The reveal | 5 | Report | 3 pages, ending on the supplier scorecard and the uncomfortable number. |
| 5 | Teaching the business its own vocabulary | 4 | Ontology (preview) | Generate from the semantic model, then say what a star schema can't: *OrderLine **suppliedBy** Supplier*. |
| 6 | Ask it anything | 3 | Data agent | 5 scripted questions. The last one has no report page. |

Plus a 30s cold open and a 60s close → **~23:30 total**.

### 1.4 Build order ≠ chapter order

Record in chapter order. **Build** in risk order, so anything that can kill the demo fails early and cheap:

1. **Confirm the four tenant settings** (§3.1). Written yes/no from your admin. 10 minutes, and it can end the project.
2. **Notebook through Cell 5**, asserts green. ~2 hrs.
3. **Semantic model + relationships** (§4.1–4.2). ~1 hr.
4. **Ontology spike** (§6) — 30 min. Needs #3, needs nothing from the report. ***This is the go/no-go.***
5. **Data agent** (§7), five questions rehearsed. ~1 hr.
6. **Report** (§5) last — most polish-hungry, least likely to fail. Don't spend four hours on visuals before you know the ontology works.

Then throw the workspace away and build it once more, clean, for the recording.

### 1.5 What to cut if you're over

Cut page 3 of the report, then Chapter 2's silver detail. **Never cut Chapter 5 or 6** — those are the only parts that aren't a 2018 Power BI demo.

---

## 2. What's wrong with the raw data

Seven defects, all measured. In Path A these *are* Chapter 2's material — you fix them on camera. In Path B they're already fixed.

*(The `§` column is the reference used by the notebook comments and the risk table below.)*

| § | Defect | Measured | Consequence if ignored |
|---|---|---|---|
| 2.1 | No signal in the fact table | Intensity by rating A→E: 0.1595 / 0.1543 / 0.1605 / 0.1605 / 0.1597 | Every chart is a flat bar chart |
| 2.2 | Ship mode & material flat too | Spread 1.04× and 1.03× | Page 2 has two dead visuals |
| 2.3 | `region` random vs `country` | All 19 countries appear in all 4 regions | Map visuals look broken |
| 2.4 | SCD2 dims not resolved | 2,400 product rows / 800 keys; only 10,011 of 30,000 fact rows hit a current row | Product count reads 2,400; trust evaporates |
| 2.5 | `dim_date_id` random vs `order_ts` | **29,940 of 30,000 rows disagree** | Report and agent give different trends |
| 2.6 | Shaping breaks the arithmetic | `(embodied+transport)×qty = line_co2e_kg` holds 30,000/30,000 in source, 0/30,000 after naive scaling | Model contradicts itself on screen |
| 2.7 | Country list incomplete in any hand-written map | `SWEDEN` missing from my first draft | Null regions |

Three deserve narration:

**2.1 — There is no signal at all.** Every FK was assigned uniformly at random, so "our A-rated suppliers aren't our cleanest" doesn't exist, and neither does its opposite. Four flat bars in a row reads as a broken demo, not a finding. The fix scales emissions by supplier rating, factory renewable %, material family and shipping mode — every one a column the dataset already carries. Say it out loud: *"synthetic data, shaped deliberately so the pattern is visible."* That costs you nothing. Shipping a flat demo costs you a lot.

**2.2 — The cargo bike problem.** `dim_ship_mode` carries a real `kg_co2e_per_tonne_km` factor. The fact table ignores it completely:

| mode | factor in the dim | avg transport CO2e/unit in the fact |
|---|---:|---:|
| AIR | 1.050 | 2.341 |
| SEA | 0.012 | 2.346 |
| LAST_MILE_CARGO_BIKE | 0.000 | 2.359 |

A cargo bike emitting like a 747, on screen, in front of a sustainability audience. Deriving transport properly (`rate × tonnes × lane_km`) fixes it and produces the best stat in the demo: **air freight is 17% of revenue and 32% of emissions.**

**2.4 — SCD2 is the best thing in the dataset.** `product_nk`/`supplier_nk`, `valid_from`, `valid_to`, `is_current` are all present and real. That supports the most credible question you can ask an architect audience: *"was this supplier A-rated at the time of the order, or today?"* — which a flat CSV extract cannot answer and a governed model can. Keep both views deliberately; make the difference a beat, not a defect.

### What's genuinely good

- Source derived columns reconcile to the cent (30,000/30,000) — worth a "trust but verify" moment.
- `dim_date` covers 2025-01-01 → 2026-07-20, 566 unique dates, no gaps. Enough for trend; **not** enough for YoY (2026 stops in July). Use rolling 3-month.
- 30k fact rows on Direct Lake — instant. No perf excuses on camera.
- Six clean dimensions, 800 current styles, 6 categories, 12 materials, 6 ship modes. Every visual has enough cardinality to look real.

---

## 3. Chapter 2 — Lakehouse + notebook

### 3.1 Prerequisites — check these before you upload anything

**Four tenant settings**, exact names from the ontology prerequisites. All need a Fabric administrator:

1. *Enable Ontology item (preview)*
2. *Users can use Copilot and other features powered by Azure OpenAI* (required for data agent)
3. *Data sent to Azure OpenAI can be **processed** outside your capacity's geographic region, compliance boundary, or national cloud instance*
4. *Data sent to Azure OpenAI can be **stored** outside your capacity's geographic region, compliance boundary, or national cloud instance*

Capacity must be **F2 or higher** (data agent minimum).

> #3 and #4 are the ones regulated tenants refuse. If your admin won't enable them, Chapters 5 and 6 are dead on this tenant — find out now, not in hour six.

### 3.2 Workspace and upload

Create workspace `Footwear-Sustainability-Demo`. New item → **Lakehouse**, name `lh_footwear`. Open it → `Files` → **Upload** → all 7 raw CSVs into a folder `bronze/`.

Manual upload is deliberate. A pipeline adds 8 minutes of demo and teaches nothing here — say that on camera; it's a credibility point, not a shortcut.

### 3.3 The notebook

New item → **Notebook**, name `nb_build`, attach `lh_footwear`.

```python
# Cell 1 — bronze: CSV -> Delta, as-is
TABLES = ["dim_date","dim_ship_mode","dim_factory","dim_supplier",
          "dim_product","dim_customer","fact_order_line"]

for t in TABLES:
    (spark.read.option("header", True).option("inferSchema", True)
        .csv(f"Files/bronze/{t}.csv")
        .write.mode("overwrite").saveAsTable(f"bronze_{t}"))

print("bronze loaded")
```

```python
# Cell 2 — silver: fix region, which the source gets wrong (§2.3)
from pyspark.sql.functions import col, create_map, lit
from itertools import chain

# verified complete against all 19 countries in dim_supplier + dim_factory + dim_customer
COUNTRY_REGION = {
    "CHINA":"ASIA","INDIA":"ASIA","VIETNAM":"ASIA","INDONESIA":"ASIA","JAPAN":"ASIA",
    "ETHIOPIA":"AFRICA",
    "ITALY":"EUROPE","PORTUGAL":"EUROPE","GERMANY":"EUROPE","SPAIN":"EUROPE",
    "FRANCE":"EUROPE","UNITED KINGDOM":"EUROPE","NETHERLANDS":"EUROPE","SWEDEN":"EUROPE",
    "BRAZIL":"AMERICA","MEXICO":"AMERICA","UNITED STATES":"AMERICA","CANADA":"AMERICA",
    "AUSTRALIA":"APAC",
}
region_map = create_map([lit(x) for x in chain(*COUNTRY_REGION.items())])

for t in ["dim_factory","dim_supplier","dim_customer"]:
    (spark.table(f"bronze_{t}")
        .withColumn("region", region_map[col("country")])
        .write.mode("overwrite").option("overwriteSchema", True)
        .saveAsTable(f"silver_{t}"))

# should return zero rows - cheapest guard in the notebook
display(spark.table("silver_dim_supplier").filter(col("region").isNull()).select("country").distinct())
```

```python
# Cell 3 — silver: pass-through for the rest
for t in ["dim_date","dim_ship_mode","dim_product","fact_order_line"]:
    (spark.table(f"bronze_{t}")
        .write.mode("overwrite").option("overwriteSchema", True)
        .saveAsTable(f"silver_{t}"))
```

```python
# Cell 4 — gold: fix the date FK, resolve SCD2 to current, shape the carbon signal
from pyspark.sql.functions import (col, when, lit, to_date, create_map,
                                   year, quarter, date_format, concat, round as sround)
from itertools import chain

sup   = spark.table("silver_dim_supplier")
prod  = spark.table("silver_dim_product")
fact  = spark.table("silver_fact_order_line")
fac   = spark.table("silver_dim_factory")
date  = spark.table("silver_dim_date")
ship  = spark.table("silver_dim_ship_mode")

# --- §2.5: dim_date_id is random in the source. Rebuild it from order_ts. ---
fact = (fact.drop("dim_date_id")
            .withColumn("order_date", to_date(col("order_ts")))
            .join(date.select(to_date(col("date_actual")).alias("order_date"),
                              col("id").alias("dim_date_id")), "order_date")
            .drop("order_date"))

# --- §2.4: id -> natural key -> current id, for supplier and product ---
sup_cur  = sup.filter(col("is_current")).select("supplier_nk",
                col("id").alias("cur_supplier_id"), "sustainability_rating")
prod_cur = prod.filter(col("is_current")).select("product_nk",
                col("id").alias("cur_product_id"), "material_family", "unit_weight_kg")

f = (fact
     .join(sup.select(col("id").alias("dim_supplier_id"), "supplier_nk"), "dim_supplier_id")
     .join(sup_cur, "supplier_nk")
     .join(prod.select(col("id").alias("dim_product_id"), "product_nk"), "dim_product_id")
     .join(prod_cur, "product_nk")
     .join(fac.select(col("id").alias("dim_factory_id"), "renewable_energy_pct"), "dim_factory_id")
     .join(ship.select(col("id").alias("dim_ship_mode_id"), "mode", "kg_co2e_per_tonne_km"),
           "dim_ship_mode_id"))

# --- §2.1 / §2.2 signal shaping: deterministic, documented, stated on camera ---

# EMBODIED: supplier rating drives it, renewable-heavy factories pull it down,
# material family scales it. All three are columns the dataset already carries.
sup_mult = (when(col("sustainability_rating") == "A", lit(0.55))
            .when(col("sustainability_rating") == "B", lit(0.80))
            .when(col("sustainability_rating") == "C", lit(1.00))
            .when(col("sustainability_rating") == "D", lit(1.45))
            .otherwise(lit(1.90)))
sup_mult = sup_mult * (lit(1.0) - (col("renewable_energy_pct") / lit(100.0)) * lit(0.30))

MATERIAL = {"Cork":0.60, "Algae Foam":0.65, "Recycled PET":0.70, "Hemp Canvas":0.70,
            "TENCEL Lyocell":0.75, "Recycled Rubber":0.80, "Organic Cotton":0.85,
            "Recycled Nylon":0.90, "Bio-EVA":0.95, "Natural Rubber":1.00,
            "Apple Leather":1.10, "Chrome-free Leather":1.55}
mat_mult = create_map([lit(x) for x in chain(*MATERIAL.items())])[col("material_family")]

# TRANSPORT: the source ignores dim_ship_mode.kg_co2e_per_tonne_km entirely, so a
# cargo bike emits as much as air freight. Derive it properly:
#   tonnes * km * kgCO2e-per-tonne-km. Long-haul lane 12,000 km; last mile 50 km.
lane_km = when(col("mode").startswith("LAST_MILE"), lit(50.0)).otherwise(lit(12000.0))

# §2.6: scale the PER-UNIT columns, then re-derive line totals FROM them, so
# (embodied + transport) * qty == line_co2e_kg still holds after shaping.
gold = (f
    .withColumn("embodied_co2e_kg_per_unit",
        sround(col("embodied_co2e_kg_per_unit") * sup_mult * mat_mult, 4))
    .withColumn("transport_co2e_kg_per_unit",
        sround(col("kg_co2e_per_tonne_km") * (col("unit_weight_kg") / lit(1000.0)) * lane_km, 4))
    .withColumn("line_co2e_kg",
        sround((col("embodied_co2e_kg_per_unit") + col("transport_co2e_kg_per_unit"))
               * col("quantity"), 3))
    .withColumn("co2e_per_revenue_dollar",
        sround(col("line_co2e_kg") / col("line_revenue"), 4))
    .drop("dim_supplier_id","dim_product_id","sustainability_rating","renewable_energy_pct",
          "supplier_nk","product_nk","material_family","unit_weight_kg",
          "mode","kg_co2e_per_tonne_km")
    .withColumnRenamed("cur_supplier_id","dim_supplier_id")
    .withColumnRenamed("cur_product_id","dim_product_id"))

gold.write.mode("overwrite").option("overwriteSchema", True).saveAsTable("gold_fact_order_line")

# current-only dims for the semantic model; the rest pass through
sup.filter(col("is_current")).write.mode("overwrite").option("overwriteSchema", True).saveAsTable("gold_dim_supplier")
prod.filter(col("is_current")).write.mode("overwrite").option("overwriteSchema", True).saveAsTable("gold_dim_product")
for t in ["dim_ship_mode","dim_factory","dim_customer"]:      # NOT dim_date - built below
    (spark.table("silver_" + t)
        .write.mode("overwrite").option("overwriteSchema", True).saveAsTable("gold_" + t))

# gold_dim_date needs Year/Month/Quarter attributes, and you CANNOT add them later as
# DAX calculated columns - unsupported on Direct Lake over a SQL endpoint (see §4.1).
(date.withColumn("Year",      year("date_actual"))
     .withColumn("Month",     date_format("date_actual", "yyyy-MM"))
     .withColumn("MonthName", date_format("date_actual", "MMM yyyy"))
     .withColumn("Quarter",   concat(lit("Q"), quarter("date_actual"), lit(" "), year("date_actual")))
     .write.mode("overwrite").option("overwriteSchema", True).saveAsTable("gold_dim_date"))
```

```python
# Cell 5 — the check. Run it on camera.
import pyspark.sql.functions as F

g = spark.table("gold_fact_order_line").join(
        spark.table("gold_dim_supplier").select(F.col("id").alias("dim_supplier_id"),
                                                "sustainability_rating"), "dim_supplier_id")
display(g.groupBy("sustainability_rating")
         .agg(F.round(F.sum("line_co2e_kg")/F.sum("line_revenue"), 4).alias("kgCO2e_per_dollar"),
              F.count("*").alias("lines"))
         .orderBy("sustainability_rating"))

gf = spark.table("gold_fact_order_line")
assert gf.count() == 30000, "fact row count changed - joins fanned out"
assert spark.table("gold_dim_product").count() == 800,  "product should be 800 current rows"
assert spark.table("gold_dim_supplier").count() == 150, "supplier should be 150 current rows"

# §2.6: line totals still reconcile to the per-unit columns
bad = gf.filter(F.abs((F.col("embodied_co2e_kg_per_unit") + F.col("transport_co2e_kg_per_unit"))
                      * F.col("quantity") - F.col("line_co2e_kg")) > 0.01).count()
assert bad == 0, f"{bad} rows where per-unit columns don't reconcile to line_co2e_kg"

# §2.5: the date FK now agrees with the row's own timestamp
bad_dt = (gf.join(spark.table("gold_dim_date").select(F.col("id").alias("dim_date_id"), "date_actual"),
                  "dim_date_id")
            .filter(F.to_date("order_ts") != F.to_date("date_actual")).count())
assert bad_dt == 0, f"{bad_dt} rows where dim_date_id disagrees with order_ts"
print("all checks passed")
```

**All five asserts caught something real when I validated this against the CSVs:**

- `count == 30000` — the SCD2 joins are exactly where a silent fan-out happens, and a fan-out inflates every number without breaking anything visibly. (Verified: no fan-out.)
- the reconciliation assert — fails on all 30,000 rows without the per-unit fix.
- the date assert — fails on 29,940 rows without the FK rebuild.

**On camera, spend 45 seconds on:** `print("all checks passed")`, then the before/after of the rating table. Flat-to-sloped, with a green assert above it, is the moment an architect decides you know what you're doing.

---

## 4. Chapter 3 — Direct Lake semantic model

From the lakehouse ribbon → **New semantic model** → name `sm_footwear_sustainability`, select the **7 `gold_` tables only**.

### 4.1 Relationships

All many-to-one, single cross-filter, active:

| From (`gold_fact_order_line`) | To | Column |
|---|---|---|
| `dim_date_id` | `gold_dim_date` | `id` |
| `dim_customer_id` | `gold_dim_customer` | `id` |
| `dim_product_id` | `gold_dim_product` | `id` |
| `dim_supplier_id` | `gold_dim_supplier` | `id` |
| `dim_factory_id` | `gold_dim_factory` | `id` |
| `dim_ship_mode_id` | `gold_dim_ship_mode` | `id` |

Then **mark `gold_dim_date` as a date table** on `date_actual`, and **sort `MonthName` by `Month`** — otherwise your time axis reads Apr, Aug, Dec.

> ⚠️ **Do not try to add DAX calculated columns here.** Calculated columns are **not supported on Direct Lake over a SQL analytics endpoint** — which is exactly what "New semantic model" from the lakehouse ribbon creates. (Preview-supported on *Direct Lake on OneLake* only.) `Year = YEAR(...)` will fail, on camera, in Chapter 3. That's why Cell 4 builds the date attributes in Spark.

### 4.2 Measures

Put them in a dedicated `_Measures` table, then hide its placeholder column:

```dax
_Measures = { BLANK() }
```

> This one *is* legal on Direct Lake on SQL — the docs allow calculated tables **that don't reference Direct Lake columns or tables**, and `{ BLANK() }` references nothing. A calculated table built from `gold_*` would not be.

```dax
Revenue = SUM('gold_fact_order_line'[line_revenue])

Units Sold = SUM('gold_fact_order_line'[quantity])

Total CO2e (kg) = SUM('gold_fact_order_line'[line_co2e_kg])

Total Waste (kg) = SUM('gold_fact_order_line'[line_waste_kg])

-- the hero measure. defined once, here, forever.
Carbon Intensity =
DIVIDE( [Total CO2e (kg)], [Revenue] )

-- 0.09 is chosen, not arbitrary. Measured overall intensity is 0.102, so the company
-- sits 13.6% OVER target, 41% of revenue is above the line, and 92 of 150 suppliers
-- miss it. A target of 0.10 leaves you 2% over and the urgency evaporates; 0.12 puts
-- you UNDER target and there's no demo left.
Carbon Intensity Target = 0.09

Intensity vs Target =
DIVIDE( [Carbon Intensity] - [Carbon Intensity Target], [Carbon Intensity Target] )

-- revenue we cannot defend: lines above target intensity
Revenue at Risk =
CALCULATE(
    [Revenue],
    FILTER( 'gold_fact_order_line',
            'gold_fact_order_line'[co2e_per_revenue_dollar] > [Carbon Intensity Target] )
)

Revenue at Risk % = DIVIDE( [Revenue at Risk], [Revenue] )

-- rolling 3 months: the data stops mid-2026, so YoY would lie
CO2e Rolling 3M =
CALCULATE(
    [Total CO2e (kg)],
    DATESINPERIOD( 'gold_dim_date'[date_actual], MAX('gold_dim_date'[date_actual]), -3, MONTH )
)

Supplier Intensity Rank =
RANKX( ALLSELECTED('gold_dim_supplier'[name]), [Carbon Intensity], , DESC )
```

Format `Carbon Intensity` to 3 decimals, `Intensity vs Target` and `Revenue at Risk %` as percentage, `Revenue` as currency.

### 4.3 Prep for AI — do not skip this

`Semantic model → Prep data for AI`. This is what makes Chapter 6 work rather than embarrass you:

- Every measure description reads like a sentence a human would say: *"Kilograms of CO2 equivalent emitted per dollar of revenue."*
- Synonyms: `Carbon Intensity` → *carbon per dollar, emissions intensity, carbon efficiency*; `gold_dim_supplier` → *vendor, partner, mill*.
- Hide every `id` and `*_id` column — the agent will happily try to aggregate them.
- Add **verified answers** for the questions you'll ask on camera.

Ten minutes here removes most of the risk from the agent chapter. An agent that answers wrong on a recording is worse than no agent chapter.

---

## 5. Chapter 4 — The report

Three pages. No more. Every extra page is scrub-bar time.

### Page 1 — "Where we stand"
- 4 KPI cards: `Revenue`, `Total CO2e (kg)`, `Carbon Intensity`, `Revenue at Risk %`
- Line: `CO2e Rolling 3M` by `MonthName`
- Stacked column: `Total CO2e (kg)` by factory `region`, split by `category`
- Slicers: `Year`, `region`, `category`

### Page 2 — "Carbon per dollar" (the story page)
- **Scatter**: X = `Revenue`, Y = `Carbon Intensity`, size = `Units Sold`, legend = `category`, detail = `style_name`. Constant line at 0.09 on Y. The above-and-right quadrant is the punchline: *high revenue, high intensity — these SKUs are the problem and we can't just drop them.*
- **Matrix**: `material_family` × `Carbon Intensity`, conditional-formatted red-to-green. Range is **Cork 0.075 → Chrome-free Leather 0.163**, a clean 2.2× gradient across 12 materials — the colour ramp actually means something.
- **Bar**: `Carbon Intensity` by `mode`, plus a second bar of `Total CO2e (kg)` share by `mode`.
  - **Strongest single stat in the demo: air freight is 17% of revenue and 32% of total emissions.** Say those two numbers back to back and stop talking.
  - Cargo bike and EV last-mile land near zero — now physically correct rather than accidental.

### Page 3 — "Supplier scorecard" (the reveal)
- Table: supplier `name`, `country`, `sustainability_rating`, `Revenue`, `Carbon Intensity`, `Intensity vs Target`, `Supplier Intensity Rank`, sorted by intensity descending, conditional formatting on vs-target
- Bar: `Carbon Intensity` by `sustainability_rating` — sloped, thanks to Chapter 2
- Card: suppliers above target (**92 of 150**)
- Drillthrough from page 2's scatter into this page filtered to one supplier

**The line to land on:** *"Rating tracks intensity — which means the rating is doing its job. Now look at where our money actually goes."* Then point at the revenue concentrated in C and D. That's the uncomfortable number, and it survives contact with an exec.

---

## 6. Chapter 5 — Ontology (preview)

The point is **not** "here is another modeling tool." It's: *a semantic model describes a star schema; an ontology describes the business.*

> ⚠️ **Order matters.** The generator reads the semantic model's **relationships**, not just its tables — Microsoft's own tutorial defines them before importing for exactly this reason. Import a model without relationships and you get a pile of unconnected entity types to wire by hand. Finish §4.1 completely first.

### 6.1 Generate from the semantic model

1. `+ New item` → **Ontology (preview)** → name `FootwearSustainabilityOntology`
   *(letters, numbers, underscores only — no spaces or dashes)*
2. Import from semantic model → `sm_footwear_sustainability`
3. Rename the generated entity types to business language, because that is the entire point:
   `gold_dim_supplier` → **Supplier** · `gold_dim_factory` → **Factory** · `gold_dim_product` → **Product** · `gold_dim_customer` → **Customer** · `gold_fact_order_line` → **OrderLine**

Set key properties: `id` on each.

### 6.2 Relationships — say what the star schema can't

All bound with mapping table `gold_fact_order_line`:

| Relationship | Origin | Target | Matched origin | Matched target |
|---|---|---|---|---|
| *suppliedBy* | OrderLine | Supplier | `id` | `dim_supplier_id` |
| *producedAt* | OrderLine | Factory | `id` | `dim_factory_id` |
| *sold* | OrderLine | Product | `id` | `dim_product_id` |
| *placedBy* | OrderLine | Customer | `id` | `dim_customer_id` |

### 6.3 The 40 seconds that justify the chapter

Read the names back out loud — *"OrderLine **suppliedBy** Supplier, **producedAt** Factory"* — then say the quiet part:

> "A semantic model knows `dim_supplier_id` joins to `id`. It does not know that means *supplied by*. An agent reasoning over your business needs the second thing. That's the whole difference."

One clean sentence beats a diagram. Don't over-explain it.

---

## 7. Chapter 6 — Data agent

`+ New item` → **Fabric data agent** → name `agent_sustainability`. Add **two** sources, deliberately, so you can show they agree:

1. `FootwearSustainabilityOntology`
2. `sm_footwear_sustainability`

### 7.1 Agent instructions

```
You answer questions about footwear supply chain sustainability.
Carbon intensity means kilograms of CO2 equivalent per dollar of revenue.
The company target for carbon intensity is 0.09 kg CO2e per dollar.
Always state the number and the unit. If a supplier is above target, say so explicitly.
Prefer the semantic model for aggregate metrics; use the ontology when the question
is about how entities relate to each other.
```

### 7.2 The five questions — in this order

1. *"What is our total revenue and carbon intensity for 2025?"* — warm-up; the number matches the report card exactly. **Pause here. The matching number is the whole thesis.**
2. *"Which five suppliers have the highest carbon intensity, and what are their sustainability ratings?"* — reproduces report page 3 without opening it.
3. *"How much revenue comes from suppliers above our carbon intensity target?"* — a measure, answered conversationally.
4. *"Which factories produce for our highest-intensity suppliers?"* — the ontology question. A star schema has no direct factory↔supplier path; the ontology traverses it through OrderLine. The one that shouldn't work but does.
5. *"How much of our total CO2e comes from air freight, and what share of revenue does air freight represent?"* — expected: **32% of emissions, 17% of revenue.** The report has no page for this, and that's the closing line: *"Nobody built a page for that question. They didn't have to."*

> ⚠️ **Don't phrase Q5 as a hypothetical.** Microsoft's data agent docs put counterfactual and causal questions explicitly out of scope — it does retrieval and aggregation, not what-if. The version above asks for two aggregates it can actually compute and lands the same point.
>
> If you want the what-if on screen, do the arithmetic in the voiceover: air→sea on the same lanes removes **~160,000 kg, about 18% of the total footprint** — measured from this dataset, not a guess.

Expand the "steps" panel on question 2 to show the generated DAX. Transparency lands well with the architect half and costs 15 seconds.

**Rehearse all five; pin verified answers for 1–3.** Agents are non-deterministic; a recording is not. If one stays shaky after three tries, cut it and close on four — never ship a wrong number to defend a nice line.

---

## 8. Recording notes

- **Pre-build everything, then re-run only what's visual.** Nobody watches a Spark session start. Cut to a completed cell.
- **Two browser profiles**: one with the finished workspace, one empty for the "create item" moments. Splice.
- **Chapter markers with timestamps** in the description. Half your viewers jump straight to Ch. 5.
- **Cold open (30s):** the finished supplier scorecard, one sentence — *"This took four hours and it answers questions in plain English. Here's how."* Then cut to the CSVs.
- **Close (60s):** the workspace lineage view — lakehouse → model → report → ontology → agent — with the one-liner: *one governed model, three consumers, one number.*
- **Answer "is this real data?" before anyone asks**, in Chapter 2. Synthetic, deliberately shaped, method on screen.

---

## 9. Risks, ranked

| Risk | Mitigation |
|---|---|
| Tenant settings #3/#4 refused | §3.1. Get a written yes/no **before** you upload a CSV. Kills Ch. 5 and 6. |
| Ontology preview unavailable or flaky | Build it right after the semantic model, before the report (§1.4). If blocked, Ch. 5 becomes "here's what it will look like" over the relationship view; you drop to 19 min. |
| SCD2 fan-out inflates every number silently | `assert count == 30000` in Cell 5. Non-negotiable. |
| Agent and report disagree on a time question | The `dim_date_id` rebuild (§2.5). Without it they *will* — 29,940 of 30,000 rows. |
| A DAX calculated column fails in Chapter 3 | Not supported on Direct Lake / SQL endpoint. Date attributes are built in Spark — don't "just add a quick calculated column" on camera. |
| Ship-mode or material chart is flat | §2.2. Skip that shaping and page 2 has two dead visuals and a cargo bike that emits like a 747. |
| Someone checks the carbon arithmetic on screen | The per-unit derivation (§2.6) keeps `(embodied + transport) × qty = line_co2e_kg` true on every row. |
| Agent answers differently on take 3 | Verified answers, pinned examples, rehearsal; accept four questions. |
| Direct Lake falls back to DirectQuery | 30k rows won't trigger it, but check the storage-mode indicator once before recording. |

---

## 10. Known-good numbers

Check these before you build the report. If they're off, Cell 4 didn't do what you think.

| measure | expected |
|---|---:|
| `Carbon Intensity`, no filters | **0.102** |
| `Revenue at Risk %` @ target 0.09 | **40.8%** |
| Suppliers above target | **92 of 150** |
| Carbon intensity, rating A → E | **0.064 · 0.081 · 0.097 · 0.135 · 0.170** |
| Carbon intensity, Cork → Chrome-free Leather | **0.075 → 0.163** |
| AIR share of revenue / of emissions | **17% / 32%** |
| Air→sea saving on the same lanes | **~160,000 kg, 18% of total** |
| Supplier / Product / Fact rows | **150 / 800 / 30,000** |

---

## Appendix — what changed in v2

Three validation passes against the real CSVs found 13 defects. All are fixed above; the clean CSVs have them applied.

**In the data (5):** no signal in the fact table · ship mode and material equally flat, with a cargo bike emitting like a 747 · `region` random vs `country` · `dim_date_id` random vs `order_ts` (29,940/30,000) · SCD2 dims unresolved.

**In the v1 notebook (5):** `SWEDEN` missing from the region map · a copy-paste bug writing product data to a factory-named table · shaping that broke the `(embodied+transport)×qty` reconciliation on all 30,000 rows · a pass-through loop that overwrote the enriched `gold_dim_date` · only two asserts, neither catching the above.

**In the Fabric assumptions (3):** calculated columns unsupported on Direct Lake / SQL endpoint (would have failed live in Ch. 3) · ontology generation reads the semantic model's *relationships*, so §4.1 must be complete first · the closing agent question was a counterfactual, which the docs put out of scope.

Also retuned: the carbon target from 0.12 → 0.09, because at 0.12 the company beat its own target and the demo had no tension.
