# Context & Scope — Simple Stock Flow

> Reconstructed **backwards** from `spec/data-model.md`. Explicitly defines system boundaries,
> operational context, actors, and what is strictly **In-Scope** versus **Out-of-Scope**.

---

## 1. System Description

**Simple Stock Flow** is an internal catalog, inventory, and sales control system built for business operations. Its core responsibility is guaranteeing strict inventory consistency (`stock >= 0`), preserving immutable historical sales records via frozen catalog snapshots, and generating reliable aggregated sales reports.

---

## 2. Actors & System Boundaries