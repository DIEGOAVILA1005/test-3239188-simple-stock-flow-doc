# Problem Framing — Simple Stock Flow

> Reconstructed **backwards** from `spec/data-model.md`: the problem is the one that model makes
> necessary to solve. What the model does not state is marked as an **assumption**.

---

## 1. The Problem

A business that sells catalog items needs to **know how many units it has**, **sell without exceeding what it has**, and **subsequently know what it sold and how much**. Without a tool enforcing these rules, stock gets out of sync, sales leave no reliable trace, and reporting depends on who is asked.

The data model reveals the problem on three fronts:

| Pain Point | How the Model Solves It | Source |
|---|---|---|
| **Selling what is not in stock** | `stock >= 0` guaranteed by the engine; sale fails if stock is insufficient | §2.2, ADR-002 |
| **The past changing** | The sale line **freezes** name, price, and category; the sale is immutable | §1 *Frozen name*, §2.3, §2.4 |
| **Not knowing what sells most** | Aggregated report by product over a date range, stable and calculated in the engine | §1, D-06, §11.1 |

## 2. Who Suffers From It

| Persona | What They Need | Source |
|---|---|---|
| **Seller** (`seller`) | Register sales quickly and with the correct catalog | §1, §2.5 |
| **Administrator** (`admin`) | Maintain catalog and users; view reports | §1, §11 H-3 |

**Who is not part of the problem:** the buyer. The system **has no customer entity**; it records the **internal operator** who made the sale (§1, §7).

> **S-09.** The type of business is not stated. The five seeded categories (*General, Herramientas, Electricidad, Fontanería, Pinturas*, §9.1) suggest a supply or hardware store, but it is an **inference**, not a datum from the model.

## 3. Why Obvious Fixes Are Not Enough

| Obvious Fix | Why It Fails | Source |
|---|---|---|
| Validating stock only in the application | A manual `psql` or future migration bypasses it **silently**; it protects the application, not the data | § "How to read this", §1 |
| Reading price and name from the catalog when querying a sale | Renaming or repricing rewrites history | §1 *Frozen name* |
| Deleting products no longer sold | Breaks sale lines and reports | §7.1, ADR-003, FK-3 |
| Choosing "the most recent label" in the report | A new sale would alter already read data from a closed range | §11.1 |

## 4. What Is NOT the Problem

Decided to leave out, and won't be reopened without a real requirement:

- **Customers and buyers** (§1) · **payments or cards** (§7) · **multiple currencies** (D-05)
- **Category maintenance** (§2.1, §4.1) · **editing or voiding sales** (§2.3)
- **Change audit** (§8) · **extra product attributes** (DP-03)
- **Analysis by seller** (DP-02)

## 5. Constraints Conditioning the Solution

- **Honesty about what is guaranteed:** every rule is declared *engine*, *domain only*, or *pending*; what does not exist is not promised (§ "How to read this", §13).
- **Small privacy surface** (§7).
- **The engine rules:** if the document contradicts the database, the document is broken (§ intro, art. X).

## 6. Question the Product Must Answer

> *Can I register a sale with certainty that stock is correct, that the record will never change, and that I can later know what was sold?*