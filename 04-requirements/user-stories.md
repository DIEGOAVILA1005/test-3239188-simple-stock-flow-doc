# User Stories — Simple Stock Flow

> Reconstructed **from `spec/data-model.md`**: each story originates from an access pattern (Q1–Q10,
> §6.1), an invariant (§2), or a closed decision (§11). The *Source* column allows tracking each
> acceptance criterion back to the model. What does not come from the model is marked as an
> **assumption** (`S-nn`) and listed at the end.

**Actors** (§1, §2.5): **administrator** (`role = admin`) and **seller** (`role = seller`). There is no
*customer* or *buyer* actor (§1, §7).

---

## US-01 · Log in
**As an** operator **I want to** authenticate with my username and password **so that** I can use the system.

| # | Acceptance criterion | Source |
|---|---|---|
| 1 | The system looks up the user by exact username (high-frequency pattern, on every login) | Q10, §6.1 |
| 2 | Username is compared in lowercase and trimmed | §2.5 (`NormalizeUsername`) |
| 3 | Password is verified against `password_hash` **via the hash port**; the domain never sees the cleartext password | D-09, §2.5 |
| 4 | `password_hash` does not appear in any response, log, or error message | §7 |

## US-02 · Register sellers
**As an** administrator **I want to** create seller users **so that** they can register sales.

| # | Acceptance criterion | Source |
|---|---|---|
| 1 | Only an authenticated administrator can register users; anonymous registration is **not** valid | §11 H-3, DP-04, §9.2 (defect A-1) |
| 2 | The assignable role at runtime is `seller`; **nobody grants `admin` at runtime** (provisioned by deployment from the environment) | §11 H-3, DP-04, §9.2 |
| 3 | `username` is mandatory and **unique**; stored in lowercase and trimmed | §2.5, §4 |
| 4 | Password is stored only as a hash | D-09, §1 |

## US-03 · Create product
**As an** administrator **I want to** register a product **so that** I can sell it.

| # | Acceptance criterion | Source |
|---|---|---|
| 1 | A product has **name, price, stock, category, optional image, and nothing else** | §1, DP-03 |
| 2 | Name is mandatory, not empty, stored trimmed | §2.2 |
| 3 | `price > 0` | §2.2 |
| 4 | `stock >= 0` | §2.2, `ck_product_stock_non_negative` |
| 5 | Category is mandatory and must exist | §2.2, FK-1 |
| 6 | Image is optional; absence is represented with `NULL`, never an empty string | §1, §2.2 |

## US-04 · Search products
**As a** seller or administrator **I want to** search for products **so that** I can find them quickly.

| # | Acceptance criterion | Source |
|---|---|---|
| 1 | Filters by **partial text** in name and by **category** | Q1, §6.1 |
| 2 | Returns only **active** products (not soft-deleted) | Q1, §6.1, ADR-003 |
| 3 | Sorts by name and **paginates**, returning the total count of items | Q1, §6.1 |
| 4 | A product can be queried by identifier | Q2, §6.1 |

## US-05 · Modify a product
**As an** administrator **I want to** rename, reprice, restock, and recategorize **so that** I can maintain the catalog.

| # | Acceptance criterion | Source |
|---|---|---|
| 1 | The same invariants as US-03 apply | §2.2 |
| 2 | Restocking adds units; **withdrawing more stock than available fails** | §2.2 (`Restock`, `Withdraw`) |
| 3 | If two changes compete on the same product, the second is detected as a **conflict** and does not overwrite the first | D-04, ADR-002, §3 (`xmin`) |
| 4 | Changing name, price, or category **does not rewrite** already registered sales | §1 *Frozen name*, ADR-004 |

## US-06 · Soft-delete a product
**As an** administrator **I want to** remove a product from the catalog **so that** it is no longer sold.

| # | Acceptance criterion | Source |
|---|---|---|
| 1 | Deletion is **logical** (`deleted_at`); it is **never** physically deleted | §2.2, §7.1, ADR-003 |
| 2 | A soft-deleted product does not appear in searches nor can it be sold | Q1, Q3, §5 |
| 3 | Historical sales including it remain intact | §7.1, FK-3 |
| 4 | A manual physical `DELETE` **fails loudly** | FK-3 `RESTRICT`, §5 |

## US-07 · Manage product image
**As an** administrator **I want to** attach, replace, or remove the image **so that** I can identify the product.

| # | Acceptance criterion | Source |
|---|---|---|
| 1 | The system stores only an **opaque key** (`image_key`), never the path nor bytes | §1, D-08 |
| 2 | When replacing or removing, `image_key` is nullified first and the transaction **committed**; **afterwards** the binary is deleted | §7.1 |
| 3 | Atomicity is not promised with external storage: an orphaned binary is acceptable, a broken key is not | §7.1, H-2 |

## US-08 · Query categories
**As an** operator **I want to** view categories **so that** I can classify and filter products.

| # | Acceptance criterion | Source |
|---|---|---|
| 1 | There are **five** fixed categories, seeded in the initial migration | §1, §9.1, D-10 |
| 2 | Listed sorted by name | Q4, §6.1 |
| 3 | Categories **cannot** be created, renamed, or deleted | §2.1 |

## US-09 · Register a sale
**As a** seller **I want to** register a sale of multiple products **so that** I can deduct stock and record the transaction.

| # | Acceptance criterion | Source |
|---|---|---|
| 1 | The sale records **who** made it and **when** (`sold_at`) | §2.3, §3 |
| 2 | Has **at least one line** to be confirmed | §2.3 (`EnsureConfirmable`) |
| 3 | **A product is not repeated** within the same sale | §2.3, §13 D-2 |
| 4 | Each line has `quantity > 0` | §2.4 (`Quantity`) |
| 5 | Deducting stock and adding the line are **a single operation**; if stock is insufficient, the sale fails | §2.3, §2.2 |
| 6 | The line **freezes** product name, unit price, and category name at that moment | §1, §2.4, D-06 |
| 7 | Only existing and **non-deleted** products can be sold | §5 |
| 8 | Total and subtotals **are calculated, not stored** | §1, art. VII |

## US-10 · Query a sale
**As an** operator **I want to** view a sale with its lines **so that** I can review what was sold.

| # | Acceptance criterion | Source |
|---|---|---|
| 1 | Displays the sale with all its lines | Q6, §6.1 |
| 2 | Values shown are those **frozen** in the sale, not current catalog values | §1 |
| 3 | The sale **is not edited or deleted** | §2.3, §7.1 |

## US-11 · List sales by date range
**As an** administrator **I want to** view sales for a period **so that** I can review them.

| # | Acceptance criterion | Source |
|---|---|---|
| 1 | Filters by **date range**; the end date cannot precede the start date | §1 (*Date range*), Q7 |
| 2 | Sorts by descending date and **paginates**, with total item count | Q7, §6.1 |

## US-12 · Sales report by product
**As an** administrator **I want an** aggregated report by product in a range **so that** I know what sells most.

| # | Acceptance criterion | Source |
|---|---|---|
| 1 | Aggregates **by product** over a date range, sorted by total amount descending | §1, Q9, §6.1 |
| 2 | Calculated **in the engine** via a read port; **not persisted** | §1, D-06 |
| 3 | If a product was recategorized within the range, it **groups by the frozen label**: **two rows** appear, not one | §11.1 (closed H-1) |
| 4 | A report for a closed range **does not change** even if new sales are registered with other labels | §11.1 |
| 5 | The report **is not broken down by seller** | DP-02, §7.1 |

---

## Out of Scope (decided in the model)

Customer or buyer (§1) · category maintenance (§2.1, §4.1) · editing or voiding sales (§2.3) · multiple currencies (D-05) · audit `created_at/updated_at` (§8) · extra product attributes, such as description, SKU, or code (DP-03) · breakdown of report by seller (DP-02).

## Assumptions

| # | Assumption | Reason |
|---|---|---|
| S-05 | **Permissions by role:** administrator manages catalog, users, and reports; seller registers sales and queries | The model fixes *seller registration* by admin (DP-04), but not the remaining permissions |
| S-06 | **Report columns:** product, frozen category label, total quantity, and total amount | The model indicates aggregation by product sorted by amount; index includes `quantity` and `unit_price` (§6.2) |
| S-07 | **Login error messages** do not distinguish non-existent user from incorrect password | The model does not specify it; standard practice |