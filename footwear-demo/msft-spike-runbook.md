# Blocker spike — evidence for the Microsoft meeting

**Thursday 2026-08-06, afternoon. Run this Monday or Tuesday, not Wednesday night.**

The goal is **not** a working ontology. Microsoft can build one of those. The goal is
seven captured refusals, each with the exact error text, because that is the one asset
in the room they cannot produce themselves.

Prerequisite confirmed: *Enable Ontology item (preview)* is on. Semantic model with
relationships already exists (`sm_footwear_sustainability`, Path B, tables `dim_*` /
`fact_order_line`).

**Budget: ~2 hours.** Phase 0 is 20 min, Phase 1 is 45, Phase 2 is 30, capture is the rest.

---

## Capture discipline — do this before you start

Open one document. For every step below record four things:

| | |
|---|---|
| **What I tried** | one sentence, in product terms |
| **What happened** | screenshot, plus the error text typed out verbatim |
| **Where it appeared** | which surface — picker, save, refresh, query |
| **Deal consequence** | one line: what this stops a customer from doing |

The fourth column is the whole point. A screenshot of an error is a bug report. A
screenshot with a deal consequence attached is field intelligence, and field
intelligence is what gets you invited back.

---

## ⚠️ Read this before Phase 2

**Do not enable OneLake security on `lh_footwear`.** It may be one-way, and it can break
the demo you already built through Chapter 4. Create a throwaway lakehouse
(`lh_spike_secured`) with two small tables copied from the demo, and do test 8 there.

Same logic for test 7 — the rename breaks bindings by design, so it goes last in
Phase 2, after everything that needs a working ontology is captured.

---

## Phase 0 · Baseline (20 min)

You need a working ontology as the control before you break anything. This is also
your slide 1 opening image, and it exercises the semantic-model import path, which is
a genuine Fabric differentiator nobody else offers.

1. `+ New item` → **Ontology (preview)** → `FootwearSustainabilityOntology`
   *(letters, numbers, underscores only)*
2. Import from semantic model → `sm_footwear_sustainability`
3. Rename to business language: `dim_supplier` → **Supplier**, `dim_factory` →
   **Factory**, `dim_product` → **Product**, `dim_customer` → **Customer**,
   `fact_order_line` → **OrderLine**
4. Set key property `id` on each
5. Relationships, all bound with mapping table `fact_order_line`:
   *suppliedBy* · *producedAt* · *sold* · *placedBy*

**Capture:** how long the import took, how much it got right unaided, and what it got
wrong. "The generator did X of Y correctly" is a compliment with a number attached,
and it buys you the right to criticise for the next twenty minutes.

**Also capture:** run one NL2Ontology query. You should be able to speak to the agent
grounding surface first-hand, since that is the thing they are actually building toward.

---

## Phase 1 · The non-destructive refusals (45 min)

None of these damage the working ontology. Run them in order.

### 1 · Second static binding
Try to add a second static data binding to **Supplier** from any other table.
> Documented: *"Each entity type supports one **static** data binding. You can't combine
> static data from multiple sources for a single entity type."*

**Deal consequence:** Customer cannot be assembled from CRM + POS + loyalty. Conformance
stays in gold. This is the one that reshapes a retail roadmap.

### 2 · Shortcut / external table
Create a OneLake shortcut in `lh_footwear` to any table outside it. Try to bind.
> Documented: *"Ontology only supports **managed** lakehouse tables… not **external**
> tables."*

**Deal consequence:** the largest one. Shortcuts are how Fabric tells customers to avoid
copying data — and the binding page's own opening claim is *"integrate data into a
semantic layer without copying source data."* Capture the picker with the shortcut
absent, next to that sentence. **This screenshot is slide 4.**

### 3 · Column mapping via a space in a column name
Create a small delta table with a column named `supplier name` (space included). Column
mapping auto-enables. Try to bind.

**Deal consequence:** real retail schemas carry spaces and punctuation constantly. This
fires without anyone choosing it, and the error will not mention column mapping.

### 4 · Import-mode semantic model table
Docs say column mapping is enabled automatically *"on the delta tables that store data
for import mode semantic model tables"* — while the overview lists Power BI semantic
models as a binding source. Test an import-mode model.

**Deal consequence:** if this fails, a documented data source silently doesn't work for
the most common semantic-model storage mode in the install base. Highest-value single
finding if it reproduces. Test it carefully and describe it precisely.

### 5 · Versioning
Look for version history, diff, or rollback anywhere in the item. Change a definition
and try to get the prior state back.
> Documented: *"No, versioning isn't currently available."*

**Deal consequence:** no change control on the layer that defines Revenue. Every other
production data asset in the estate has it.

### 6 · Manual refresh
Insert a row upstream. Query the ontology — it should be stale. Refresh the graph
model. Time the gap.

**Deal consequence:** anyone who assumes an agent reads live data is wrong, and nothing
in the product tells them so at query time.

---

## Phase 2 · Destructive, in this order (30 min)

### 7 · Rename a bound table
Rename a bound lakehouse table and see what breaks and how loudly.
> Documented: *"Changing the lakehouse table name after mappings are created may result
> in problems accessing data."*

**Deal consequence:** "may result in problems" is doing a lot of work. Find out whether
it fails loudly or silently. Silent is a much worse finding, and a much better slide.

### 8 · OneLake security — on `lh_spike_secured`, not the demo
Enable OneLake security on the throwaway lakehouse. Try to use it as a binding source.
> Documented: *"You can't use lakehouses with OneLake security enabled as data sources
> for bindings."*

**Deal consequence:** the headline. To give an agent a governed definition of Customer,
you must first turn off the storage-layer security. Not reconfigure — turn off.

**The sentence this earns you:** *"Your differentiation against Palantir and Databricks
is governance, and the ontology can't read your governed storage."*

---

## What you walk out of the spike holding

- One working ontology, generated from an existing semantic model — the differentiator
- Seven refusals with verbatim error text and a deal consequence each
- One screenshot pairing the "without copying source data" claim with the managed-tables-only
  limitation — the contradiction, on one page, in their own documentation
- First-hand experience of NL2Ontology, so "my experience with the feature" is true

If a test doesn't reproduce, **say so in the room.** Reporting six of seven honestly is
worth more than seven asserted. They will check.

---

## If Phase 0 fails

If the import path itself is broken, stop and tell me. That inverts the meeting — a
broken generator is a bigger finding than all seven limits combined, and the deck would
lead with it instead.
