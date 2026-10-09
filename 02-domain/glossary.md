# Glossary — Simple Stock Flow Ubiquitous Language

> Reconstructed from `spec/data-model.md`. Defines the core terms used across domain models,
> architectural ports, requirements, and user stories. Each term references its model source.

---

| Term | Definition | Source |
|---|---|---|
| **Stock** | The current available physical quantity of a catalog item in the warehouse (`product.stock >= 0`). | §2.2, ADR-002 |
| **Frozen Label / Frozen Snapshot** | Historical preservation of product attributes (`product_name`, `category_name`, `unit_price`) copied directly into `SaleItem` at purchase time, ensuring catalog changes never modify historical sales records. | §1, §2.4, D-06 |
| **Soft Delete** | Logical deletion mechanism (`product.deleted_at IS NOT NULL`) that hides a product from active searches and sales while preserving foreign key integrity for historical reports. | §2.2, §7.1, ADR-003 |
| **Immutable Sale** | Business rule stating that once a sale transaction is committed, its header and line items can never be edited, updated, or removed. | §2.3, §7.1 |
| **Optimistic Concurrency (`xmin`)** | Concurrency strategy leveraging PostgreSQL system column `xmin` as a row-version shadow property to detect concurrent modifications without blocking locks. | D-04, §3 |
| **Opaque Key (`image_key`)** | Random string token referencing an external image binary in storage, keeping binary payloads outside the relational database. | §1, D-08, §7.1 |
| **Money** | Single-currency Value Object represented by `numeric(18,2)` rounding half away from zero. | D-05, §2.2 |
| **Quantity** | Non-negative integer Value Object representing product units or purchase line quantities. | §2.2, §2.4 |
| **Date Range** | Value Object encapsulating start and end timestamps (`timestamptz`) for querying sales and calculating aggregated sales reports. | §1, Q7, Q9 |
| **Declared Debt** | Known inconsistency or missing database constraint explicitly recorded in §13 rather than left as a silent defect. | §13 |
| **Admin** | System operator role authorized to manage catalog items, register sellers, and view aggregated reports. Admin accounts are provisioned via environment configuration. | §1, §2.5, DP-04 |
| **Seller** | System operator role authorized to search products, query categories, and register sales. | §1, §2.5 |