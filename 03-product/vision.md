# Vision — Simple Stock Flow

> Reconstructed from `spec/data-model.md`. Reflects what the model **already commits to**; adds no
> features. What is not supported is marked as an **assumption**.

---

## Vision Statement

> **For** the operational team of a business with a catalog (sellers and administrator),
> **who need to** sell without exceeding stock and subsequently know what was sold,
> **Simple Stock Flow** is a **catalog, stock, and sales control system**
> **that** enforces rules in the data itself, leaves sales immutable, and delivers a stable
> report by product.
> **Unlike** a spreadsheet or rules living only in the application,
> **our system** guarantees in the engine what can be guaranteed and **declares in writing** what is
> not yet.

## Principles

| # | Principle | What it means | Source |
|---|---|---|---|
| P-1 | **Simple by design** | Five entities, five tables, "no surplus"; a product has name, price, stock, category, and image and nothing else | §2, DP-03 |
| P-2 | **The engine rules** | If the document contradicts the database, the document is broken | Intro, art. X |
| P-3 | **What is not guaranteed is declared** | Every rule: *engine*, *domain only*, or *pending*. Declared debt ≠ silent trap | § "How to read this", §13 |
| P-4 | **The past does not change** | Immutable sales and frozen values; the report for a closed range does not move | §1, §2.3, §11.1 |
| P-5 | **History is never lost** | Soft delete for products; no physical deletion | §7.1, ADR-003 |
| P-6 | **Minimal personal data** | No end customer; the report is not broken down by seller | §7, DP-02 |
| P-7 | **Single source of truth** | No `DEFAULT` in the engine; values are set by the domain | §3 |

## What the Product Delivers

1. **Catalog** with search, fixed categories, soft delete, and optional image (US-03 to US-08).
2. **Sales** that deduct stock atomically, freeze sold items, and are not edited (US-09 to US-11).
3. **Report** aggregated by product and range, calculated in the engine, and stable (US-12).
4. **Access** with two roles, no anonymous registration, and no granting of `admin` at runtime (US-01, US-02).

## What It Is NOT

No customers or buyers · no payments or multiple currencies · no category CRUD · no editing or
voiding of sales · no change auditing · no seller breakdown.
*(All decided and recorded: §1, §2.1, §2.3, §7, §8, D-05, DP-02, DP-03.)*

## How We Will Know It Works

The model **does not define numerical metrics** (see S-08 in `04-requirements/non-functional-requirements.md`). Instead, it sets **verifiable** criteria:

| Criterion | How it is checked | Source |
|---|---|---|
| No sale leaves stock negative | `ck_product_stock_non_negative` rejects the `INSERT`/`UPDATE` | §2.2 |
| The report for a closed range does not change | Same query, same result after new sales | §11.1 |
| The document does not lie | The queries in §10 return what §3, §4, and §6.2 affirm | §10 |
| Every debt is recorded | The log in §13 matches the document's markings | §13 |

## Assumptions

| # | Assumption |
|---|---|
| S-10 | The product is for **internal business use**; it has no online store or buyer access (consistent with the absence of a customer entity, §1) |
| S-09 | The line of business is not stated; see `problem-framing.md` |