# Architecture — Simple Stock Flow

> **First pass**, reconstructed **backwards** from `spec/data-model.md` (the only input of the challenge).
> Each statement has its source (`§n`, `FK-n`, `D-nn`). What does not come from the model is marked as
> an **assumption** (`S-nn`) and is listed in §10. This folder is **closed at the end** (step 6), contrasting it
> with `04`, `03`, `02`, and `01`.

---

## 1. Architectural Style

**Hexagonal Architecture (Ports and Adapters).** The domain does not know the database or the storage: it communicates with the outside only through ports.

| Evidence in the model | Source |
|---|---|
| "The domain never sees the cleartext password; the hash is produced by a port" | §1, §2.5, D-09 |
| The report is calculated in the engine by a **read port** | §1, D-06 |
| The translation between C# collections and tables is the responsibility of the **persistence adapter**, "where the mapping lives" | §0 |
| If `stock >= 0` trips, "something wrote outside the adapter" | §1, ADR-002 |
| The image binary lives in **external storage**, referenced by an opaque key | §1, D-08 |
| The database **does not** participate in the external storage transaction | §7.1 |

**It is not a distributed system.** There is a single database (`simple_stock_flow`, schema `sales`) and the model does not describe any other service with its own data (§0, §3). **S-01:** a single deployable backend is assumed.

---

## 2. System Components

| Component | Responsibility | Source |
|---|---|---|
| **Domain** (`src/domain/`) | Aggregates, value objects (`Money`, `Quantity`), business invariants | §2, §12 |
| **Application** | Use cases; custom query value objects (*Date Range*); orchestrates ports | §1 (Date Range) |
| **Outbound persistence adapter** (`src/adapters/outbound/persistence/Configurations/`) | EF Core ↔ tables mapping; shadow properties (`deleted_at`, `xmin`, FK); global soft delete filter | §0, §3, §12 |
| **Inbound adapter (API)** | Exposes the HTTP contract. The contract lives in `api-contract.md`, **outside this model** | §12 |
| **Database** | PostgreSQL 16, `sales` schema. DDL owner: **EF migrations and nothing else** | §3.2, ADR-001 |
| **Infrastructure** | Spins up the engine; **does not define the schema** | §6.2 |

The three cited project repositories (`simple-stock-flow-api`, `-infra`, `-docs`) confirm the API / infrastructure / documentation separation (§10, §12).

---

## 3. Aggregates and their boundaries

| Aggregate | Root | Contains | Lifecycle | Source |
|---|---|---|---|---|
| **Catalog** | `Product` | Name, price, stock, category, image (opaque key) | **Soft** delete, never physical | §2.2, ADR-003 |
| **Sales** | `Sale` | `SaleItem` (composition, `internal` constructor) | **Immutable** after registration | §2.3, §2.4 |
| **Identity** | `User` | `username`, `password_hash`, `role` | No deletion | §2.5, §7.1 |
| *(reference)* | `Category` | `name` | **Not an aggregate**; read-only repository; 5 seeded rows | §2.1, §9.1 |

**Boundary rule:** aggregates cross paths **only by root identity** (`category_id`, `product_id`, `sold_by_user_id`), never by object navigation (§5, *Nature* column). `SaleItem` does not exist outside its sale (FK-2 with `CASCADE`, §5).

---

## 4. Ports

The **names are proposed** (S-02); what the model fixes is the need for each one.

| Port | Type | Access pattern served | Source |
|---|---|---|---|
| Product repository | Outbound | Q1 search (text, category, active, paginated), Q2 by id, Q3 by batch | §6.1 |
| Category repository | Outbound, **read-only** | Q4 list, Q5 by id | §2.1, §6.1 |
| Sales repository | Outbound | Q6 sale with lines, Q7 by paginated range | §6.1 |
| **Report read port** | Outbound, **read model** | Q9 aggregation by product in the engine. Not persisted | §1, D-06, ADR-004 |
| User repository | Outbound | Q10 by exact name, on every login | §6.1 |
| **Password hashing port** | Outbound | Produce and verify `password_hash` | D-09, §7 |
| **Image storage port** | Outbound | Save/delete binary; the domain only sees `image_key` | D-08, §7.1 |

**Q8** (sales by unpaginated range) "has no consumer" and the model recommends **removing it from the port** instead of leaving it as a trap (§6.1).

---

## 5. Where each rule lives

This is the core of the architecture: the model classifies **each rule** into one of three marks (§ "How to read this"). An invariant that only lives in C# **protects the application, not the data**: a manual `psql` bypasses it silently.

### 5.1 In the engine (today)

| Rule | Object | Source |
|---|---|---|
| Primary keys of the 5 tables | `PK_*` | §4 |
| `category.name` unique · `user.username` unique | `IX_category_name`, `IX_user_username` (unique indexes) | §4 |
| `product.stock >= 0` | `ck_product_stock_non_negative` — "last barrier" | §2.2, ADR-002 |
| Category mandatory and existing | FK-1 `RESTRICT` | §5 |
| Line does not exist without a sale | FK-2 `CASCADE` | §5 |
| Product soft delete | `product.deleted_at` + global filter | §2.2, §13 D-1, ADR-003 |
| Line → product is not deleted | FK-3 `RESTRICT` (T-20) | §5, §13 D-2 |
| A product is not repeated in a sale | Unique index `(sale_id, product_id)` (T-20) | §2.3, §13 D-2 |

### 5.2 Domain only (declared debt: T-20 moves them down to the engine)

`price > 0` · `quantity > 0` · `category.name` not empty · `user.role ∈ {admin, seller}` ·
`username` in lowercase and trimmed (§4).

### 5.3 Domain only **by nature** (not expressible in a CHECK)

- **Withdrawing more stock than available fails** — process rule (`Product.Withdraw`, §2.2).
- **A confirmed sale has at least one line** — would require a deferred trigger (§2.3).
- **Deducting stock and adding the line are a single operation** (`Sale.AddItem` → `Product.Withdraw`, §2.3).
- **Sale immutability** — guaranteed **by the absence** of an edit/delete operation (§2.3).

### 5.4 Pending

`sale.sold_by` → `sold_by_username` and `sold_by_user_id` with FK-4 `RESTRICT` (T-12) · partial and trigram indexes (T-13) · `category_name` frozen in `sale_item` (T-11) (§3, §5, §6.2).

---

## 6. Critical flow: registering a sale

1. The inbound adapter receives the request and translates it into a use case.
2. The use case **reads the products by batch** (Q3) — "the contention point of D-04" (§6.1).
3. `Sale.AddItem` calls `Product.Withdraw` (deducts stock) and **copies frozen name, price, and category** to the line (§1 *Frozen name*, §2.3, §2.4).
4. `Sale.EnsureConfirmable` demands at least one line (§2.3).
5. The adapter persists **in a single transaction**; EF detects conflict with `xmin` (D-04, §3).
6. If, despite everything, something wrote negative stock, the engine's `CHECK` fails loudly (ADR-002).

**What is guaranteed and what is not:** atomicity applies **within the database**. External storage **does not participate** in the transaction (§7.1), so no operation mixing both promises atomicity.

---

## 7. Concurrency

**Optimistic, with `xmin` as a witness** (D-04, T-10). `xmin` is a Postgres system column, exposed as a shadow property, and "does not appear in `information_schema`" (§3). The last barrier against a race condition escaping optimistic control is the `stock` `CHECK` (ADR-002).

---

## 8. Persistence and schema

- **Migrations own the DDL**, no exceptions (ADR-001). Includes extensions: `pg_trgm` would be installed **inside** the index migration (§6.2).
- **No column has a `DEFAULT`:** values are set by the domain (§3).
- **Timestamps `timestamptz`**, server in UTC (§3).
- **Single-currency:** no currency column (D-05).
- **No audit columns** `created_at` / `updated_at` (closed decision, §8 of the model).
- **Seed:** 5 categories with fixed id in the initial migration; **not the initial admin**: it is created by the application startup with environment credentials (§9.1, §9.2, D-09, D-10).
- **Names:** singular tables, intact `sales` schema (§0).

---

## 9. Security and privacy (decisions with architectural impact)

| Decision | Source |
|---|---|
| `password_hash` never in logs, responses, or projections, and **is never indexed** | §7 |
| The domain never sees the cleartext password | D-09, §2.5 |
| Closed roles: `admin`, `seller` | §1, §2.5 |
| **Nobody grants `admin` at runtime:** provisioned by the deployment from the environment; an admin registers sellers | §11 H-3, DP-04 |
| `username` and `sold_by` are **personal data**; restricted access | §7 |
| The report **is not broken down by seller** | DP-02 |
| There is no client or buyer entity | §1 |

---

## 10. Assumptions

| # | Assumption | Why it's needed |
|---|---|---|
| S-01 | A single deployable backend (not distributed) | The model describes a single database and no other service |
| S-02 | Port names are proposed | The model fixes the need, not the identifier |
| S-03 | Authentication delivers a token/session via the inbound adapter | The model mentions "login" (Q10) but not the mechanism |
| S-04 | There is no user interface within the scope of this documentation | The model does not mention a frontend |

---

## 11. Points to verify at closure (step 6)

The model has internal inconsistencies that the architecture **must not inherit without stating them**:

1. **Column count:** §3 says 22; the link and §10 say 21. A number is not fixed here.
2. **Old snapshot vs. new snapshot:** the output in §10 is from **2026-09-19** and §13 measures **2026-09-20**. This architecture takes §13 as the current state, because "the engine wins" and it is the later measurement.
3. **Index `(sale_id, product_id)`:** §6.2 assumes it's missing (T-13); §2.3, §4 and §13 assume it exists (T-20). Here it is treated as **existing**.
4. **`category_name`:** appears in the `product` table in §3 (error: belongs to `sale_item`), and §2.4 marks it as *engine* while §3 marks it as *pending (T-11)*. Here it is treated as **pending**.