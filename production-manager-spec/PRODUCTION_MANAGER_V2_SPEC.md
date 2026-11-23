# Market POS — Production Manager v2.0
## Development Specification

| | |
|---|---|
| **Document status** | Draft for implementation |
| **Date** | 23 November 2026 |
| **Plugin** | `mp-production-manager` (current version 1.0.4 → target 2.0.0) |
| **Platform** | WordPress plugin inside Market POS (multi-tenant, scoped by `OrgID`) |
| **Audience** | Claude (as the implementing developer) and the GN Information Systems team |

---

## 0. How to use this document (instructions for Claude)

1. This spec is the **source of truth** for the v2 rebuild. The v1.0.4 code dump in the project is the **starting point**. Read the relevant v1 file before you change it.
2. Work **one phase at a time** (Section 14), and within a phase **one work item at a time** (IDs like `P1-03`). Do not start a later phase unless asked.
3. For every work item, deliver:
   - the complete contents of each changed or new file (no "…rest unchanged" fragments unless the file is over ~400 lines, in which case give exact before/after blocks),
   - any migration SQL inside the PHP migration runner (Section 6.9), never as a manual step only,
   - a short "How to test" list that maps to the acceptance criteria.
4. Follow the coding rules in Section 13 without exception (PHP 7.4 compatibility, `OrgID` scoping, capability checks, prepared statements).
5. **The module must stay industry-neutral.** Never hard-code bakery words (bread, flour, loaves, baker) in code, labels or messages. All user-facing nouns come from the terminology system (Section 3). Bakery, restaurant, carpentry and soap examples in this spec are **test data only**.
6. If something in this spec conflicts with how Market POS core really works, or an item is listed in Section 15 (Open questions), **stop and ask** instead of guessing.

---

## 1. Purpose and problem statement

The Production Manager lets any business that **turns materials into finished goods** do four things: buy and receive materials, define what goes into each product, record production, and hand finished goods to the shop's stock in Market POS.

Typical users:

| Business type | Materials (inputs) | Finished goods (outputs) |
|---|---|---|
| Bakery / food production | Flour, yeast, sugar, oil, packaging | Loaves, buns, cakes |
| Restaurant / kitchen | Meat, vegetables, oil, spices | Plates/portions, sauces, prepped items |
| Carpentry / furniture workshop | Timber (planks, sheets), nails, screws, glue, varnish | Tables, chairs, doors, cabinets |
| Soap / cosmetics / chemicals | Oils, caustic soda, fragrance, colourants, wrappers | Soap bars, liquid soap, lotions |
| Other light manufacturing | Any raw material, component or packaging | Any product sold through Market POS |

**Users are mostly not technical.** v1 is hard for them because:
- the work is spread across 10 pages organised around database tables rather than tasks,
- labels are technical and sometimes wrong ("purchase units" when the system actually counts packs),
- the screens never say what to do next or explain what a number means,
- handing finished goods to the shop is a manual "Sync" button with no receiving or approval step,
- several bugs give wrong costs and wrong stock figures (Section 12).

### 1.1 Goals
- **G1 One place:** four screens organised by task (Section 7).
- **G2 Straightforward material intake:** plain-language setup with a live explanation sentence and a two-step *submit → approve* receiving flow.
- **G3 Straightforward recipes:** define them the way people think, as "this makes N units from…", with live cost and "can make" figures.
- **G4 Guidance everywhere:** a to-do list, next-step suggestions, live calculations and plain explanations.
- **G5 Seamless handover:** a completed batch automatically creates a pending handover to shop stock, which a shop person accepts, adjusts or rejects. Nobody types finished goods into Goods Received by hand.
- **G6 Correct numbers:** correct unit conversion, weighted-average costing, a full stock ledger, and stock counts.
- **G7 Works for any industry:** configurable terminology, unit sets, waste/offcut allowance, labour and overhead cost, and ready-after (curing) time.

### 1.2 Non-goals for v2.0
- Accounting journal entries.
- Purchase orders sent to suppliers (receiving only).
- Lot/expiry tracking of materials (Phase 3 optional, Section 14).
- Deducting materials automatically when the POS sells an item ("make on sale", used by restaurants for made-to-order dishes). This is designed for in Section 9.8 but built only after the open question is answered.

---

## 2. Glossary (internal / code terms)

The code uses these neutral terms. UI labels come from Section 3.

| Code term | Meaning | v1 name |
|---|---|---|
| **Material** | Anything consumed to make a product (ingredient, timber, chemical, packaging) | Ingredient (`ing_ingredients`) |
| **Pack** | The physical thing you buy and count when receiving (bag, bottle, plank, box, drum, crate). It can also be a loose unit (kg). | "Pack Size" (confusing) |
| **Content unit** | The measuring unit of what's inside one pack (kg, l, m, unit…) | `purchase_unit` |
| **Pack size** | How many content units are in one pack (50 kg per bag) | `purchase_unit_qty` |
| **Base unit** | The smallest unit in a unit group, used for all storage and maths (g, ml, mm, mm², unit) | `base_unit` |
| **Recipe** | A list of materials and quantities that produces a stated **yield** of a product | `ing_recipe` (per single unit, no yield) |
| **Yield** | How many product units one recipe makes (400 loaves, 20 plates, 1 table, 100 bars) | n/a |
| **Batch** | One production run of a product | `ing_products` row |
| **Delivery** | One receipt of materials from a supplier (a GRN) | GRN in `ing_received` |
| **Handover** | Transfer of finished goods from production into POS shop stock, waiting for acceptance | "Sync" |
| **Ledger** | The signed history of every stock movement per material | n/a (calculated on the fly) |
| **Finished good / product** | An item in the Market POS `items` table | `items` |

---

## 3. Terminology and industry profiles

### 3.1 Industry profile setting
Each organisation chooses a profile during first-run setup (changeable later in Settings). The profile only sets **defaults**: labels, suggested pack names, example placeholders and which unit groups appear first. It **never** changes business logic.

| Key | Profile | material | materials | recipe | batch | producer | output placeholder |
|---|---|---|---|---|---|---|---|
| `food` | Bakery / food production | Ingredient | Ingredients | Recipe | Batch | Baker | e.g. 400 loaves |
| `kitchen` | Restaurant / kitchen | Ingredient | Ingredients | Recipe | Prep batch | Cook | e.g. 20 plates |
| `workshop` | Carpentry / workshop | Material | Materials | Materials list | Job | Craftsperson | e.g. 1 table |
| `chemical` | Soap / cosmetics / chemicals | Raw material | Raw materials | Formula | Batch | Maker | e.g. 100 bars |
| `general` | Other / general manufacturing | Material | Materials | Recipe | Production run | Producer | e.g. 50 units |

### 3.2 Overrides
Settings → Terminology lets the owner override each label (material, materials, recipe, recipes, batch, batches, producer). Overrides are stored per organisation.

### 3.3 Implementation
- Add a PHP helper `mppm_term( string $key, bool $plural = false ): string` that returns the org override, then the profile default, then the `general` default.
- Pass all terms to JS via `wp_localize_script` as `MPPM.terms`, and add a JS helper `term(key, plural)`.
- **Rule:** no user-facing string may contain a hard-coded material/recipe/batch noun. Use the helpers. Code review must grep for "ingredient", "recipe", "batch", "bread" and "flour" in UI strings.
- Storage: option `mppm_org_{OrgID}_profile` (string), option `mppm_org_{OrgID}_terms` (JSON object).

### 3.4 Currency
v1 hard-codes "ZMW". v2 adds `mppm_currency()`, which reads the org's currency if Market POS stores it (see Open question Q7), then the option `mppm_org_{OrgID}_currency`, then `'ZMW'`. All money is formatted through `mppm_format_money()` (PHP) or `money()` (JS).

---

## 4. Users, roles and permissions

### 4.1 Capabilities (module-level, per organisation)

| Capability | Allows |
|---|---|
| `view` | See all Production screens (read only) |
| `setup` | Create and edit materials and recipes; enter opening stock |
| `receive` | Create, edit and submit deliveries |
| `approve_delivery` | Approve or reject submitted deliveries |
| `produce` | Record batches |
| `accept_handover` | Accept, adjust or reject handovers into shop stock |
| `adjust_stock` | Create and approve stock counts and adjustments |
| `void_reverse` | Void approved deliveries, reverse batches, override shortage blocks |
| `settings` | Change module settings, roles and terminology |

### 4.2 Roles (preset bundles)

| Role | Capabilities |
|---|---|
| **Owner / Manager** | all |
| **Storekeeper** | view, receive, adjust_stock (create only) |
| **Producer** | view, produce |
| **Shop receiver** | view, accept_handover |
| **Viewer** | view |

### 4.3 Storage and defaults
- New table `ing_user_roles` (Section 6.7). A user can hold several roles.
- **Small-business default:** if an org has **no** rows in `ing_user_roles`, every logged-in user in that org is treated as Owner/Manager. This keeps v1 behaviour, so nothing breaks on upgrade.
- WordPress `administrator` always has all capabilities.
- Helper: `MPPM_Access::can( string $cap ): bool`. Every REST route's `permission_callback` must call it with the correct capability. Being logged in alone is **not** enough.

### 4.4 Separation of duties
- Setting **"Require a second person to approve deliveries"** (`require_second_approver`, default **on**).
- When it is on, the user who submitted a delivery cannot approve it, and the same applies to the producer accepting their own handover.
- **Exception:** if the org has only one active user, self-approval is allowed. The record is stamped `self_approved = 1` and the UI shows "Approved by the same person who entered it".

---

## 5. Units of measure

### 5.1 Unit groups (base unit in bold)

| Group | Units (factor to base) |
|---|---|
| Weight | **g** 1 · mg 0.001 · kg 1000 · t 1,000,000 · oz 28.3495 · lb 453.592 |
| Volume | **ml** 1 · cl 10 · l 1000 · m3 1,000,000 · fl_oz 29.5735 · cup 240 · gallon 3785.41 · board_ft 2359.737 |
| Length | **mm** 1 · cm 10 · m 1000 · inch 25.4 · ft 304.8 · yd 914.4 |
| Area *(new)* | **mm2** 1 · cm2 100 · m2 1,000,000 · in2 645.16 · ft2 92,903.04 |
| Count | **unit** 1 · pair 2 · dozen 12 · gross 144 |

### 5.2 Changes from v1
- **Remove `bag25kg` and `bag50kg`** as units. A bag is a *pack* (Section 5.3). The migration converts them (Section 6.9, step M4).
- Add the Area group (sheets of plywood, fabric, glass, sandpaper).
- Add `t`, `m3`, `board_ft`, `yd`.
- Store unit keys in lower case. Display labels are friendly ("kg (kilograms)").

### 5.3 The pack model (the core of "straightforward intake")
Every material has:

| Field | Example: flour | Example: nails | Example: plank | Example: beef bought loose |
|---|---|---|---|---|
| `pack_name` (free text, singular) | bag | box | plank | kg |
| `pack_size` (decimal > 0) | 50 | 1000 | 2.4 | 1 |
| `content_unit` (= legacy `purchase_unit`) | kg | unit | m | kg |
| `base_unit` (derived) | g | unit | mm | g |

- **Base units per pack** = `pack_size × factor(content_unit)`. Flour: 50 × 1000 = 50,000 g.
- Receiving is always counted in **packs** (decimals allowed: 2.5 bags).
- Recipes may use **any unit in the same group** as the content unit.
- When the material is bought loose, the pack name equals the content unit and the pack size is 1. The UI offers a tick box: "I buy this loose (by the kg / litre / metre…)".

### 5.4 Display rules (`mppm_format_qty()` / JS `qty()`)
- Never show raw base units to users unless they are the natural unit (unit, g for small amounts).
- Material stock is shown as **packs + content**: `12.4 bags (620 kg)`. Loose materials show content only: `85 kg`.
- Auto-scale within a group for readability: ≥1000 g → kg, ≥1000 ml → l, ≥1000 mm → m, ≥1,000,000 mm² → m².
- Decimals: up to 2 for values ≥ 1, up to 3 for values < 1. Strip trailing zeros.
- Negative stock (only possible after an override) is shown in red with "(short)".

---

## 6. Data model

The DB name stays in the constant `MPPM_DB_NAME`. Allow overriding it from `wp-config.php` with `if ( ! defined('MPPM_DB_NAME') ) define(...)`.
**All new and changed tables must include `OrgID int NOT NULL` with an index, and every query must filter by it.**

### 6.1 Existing tables: columns added or changed

#### `ing_ingredients` (materials). Keep the table name.
Existing: `ingredient_id, ingredient_name, unit_measure, discription, purchase_unit, purchase_unit_qty, base_unit, OrgID`

| Add / change | Definition | Notes |
|---|---|---|
| `pack_name` | varchar(30) NULL | singular, e.g. "bag" |
| `purchase_unit_qty` | stays decimal(12,4) | meaning is now **pack_size**, NOT NULL default 1 after migration |
| `reorder_level_packs` | decimal(12,3) NULL | per-material low-stock warning, counted in packs |
| `avg_cost_base` | decimal(18,8) NOT NULL DEFAULT 0 | weighted-average cost per base unit (Section 10.2) |
| `is_active` | tinyint(1) NOT NULL DEFAULT 1 | archive instead of delete when history exists |
| `created_at`, `updated_at` | datetime | |

Legacy `unit_measure` keeps being written (= content unit) for compatibility.

#### `ing_recipe` (recipe lines). Keep the table name.
Existing: `recipe_id, product_id, ingredient_id, recipe_unit, qty decimal(12,4), OrgID`

| Add / change | Definition | Notes |
|---|---|---|
| `qty_per_yield` | decimal(14,4) NOT NULL | quantity **for the whole recipe yield**, in `recipe_unit` (what the user types) |
| `waste_pct` | decimal(5,2) NOT NULL DEFAULT 0 | offcut / trim / spill allowance, 0–90 |
| `qty` | keep | **derived** = `qty_per_yield ÷ yield_qty` (per single unit, excluding waste). Kept for legacy readers. |
| `sort_order` | int NOT NULL DEFAULT 0 | |

#### `ing_products` (batches). Keep the table name.
Existing: `batch_number (PK, AI), item_id, batch_total, date_delivered, production_rep, collected_by, synced_to_inventory, OrgID`

| Add / change | Definition | Notes |
|---|---|---|
| `batch_ref` | varchar(20) NOT NULL | human number, e.g. `B-2026-00014` (Section 6.8) |
| `planned_qty` | decimal(14,3) NOT NULL | |
| `good_qty` | decimal(14,3) NOT NULL | goods that go to the shop. `batch_total` keeps = good_qty (legacy) |
| `rejected_qty` | decimal(14,3) NOT NULL DEFAULT 0 | rejects / waste output |
| `material_cost` | decimal(14,2) NOT NULL DEFAULT 0 | sum of ledger costs for this batch |
| `labour_cost` | decimal(14,2) NOT NULL DEFAULT 0 | |
| `overhead_cost` | decimal(14,2) NOT NULL DEFAULT 0 | |
| `unit_cost` | decimal(14,4) NOT NULL DEFAULT 0 | (material + labour + overhead) ÷ good_qty |
| `status` | varchar(20) NOT NULL | `handover_pending` · `received` · `partly_received` · `handover_rejected` · `written_off` · `reversed` |
| `ready_date` | date NULL | production date + recipe `ready_after_days` |
| `reference` | varchar(100) NULL | optional customer/order/job reference |
| `notes` | text NULL | |
| `override_reason` | varchar(500) NULL | set when a shortage block was overridden |
| `created_by` | int NOT NULL | |
| `created_at` | datetime NOT NULL | |
| `reversed_by`, `reversed_at`, `reverse_reason` | int / datetime / varchar(500) NULL | |

Relabel in UI: `date_delivered` → "Date made". `production_rep` → "{producer}" (e.g. "Baker", "Craftsperson"). `collected_by` → "Handed over to". It is pre-filled from the handover acceptor and can be empty when the batch is created.

#### `ing_received` (approved delivery lines). Keep it, but it now holds **approved lines only**.
Existing: `received_id, ingredient_id, GRN, total_received int, price, date_received, received_by, verified_by, OrgID, voided, void_reason`

| Add / change | Definition | Notes |
|---|---|---|
| `grn_id` | int NULL | FK → `ing_grn.grn_id` (NULL for legacy rows until backfilled) |
| `qty_packs` | decimal(14,3) NOT NULL | replaces the integer `total_received`. Keep writing `total_received = ROUND(qty_packs)` for legacy readers |
| `pack_size_snapshot` | decimal(12,4) NOT NULL | pack size at the time of receipt, **so history never changes** |
| `content_unit_snapshot` | varchar(20) NOT NULL | |
| `qty_base` | decimal(18,4) NOT NULL | qty_packs × pack_size_snapshot × factor |
| `price` | keep | **price per pack** (Section 10.1). Widen to decimal(14,4) |
| `unit_cost_base` | decimal(18,8) NOT NULL | price ÷ base per pack |

#### `ing_stock_deductions`: keep for legacy, plus a ledger link
Add `planned_qty_base decimal(18,4)`, `unit_cost_base decimal(18,8)`, `cost_total decimal(14,4)`, `ledger_id int`. The ledger (6.2) is now the source of truth. This table is kept in sync for compatibility.

### 6.2 New table `ing_stock_ledger` (source of truth for balances)
```sql
CREATE TABLE IF NOT EXISTS `ing_stock_ledger` (
  `ledger_id`       bigint NOT NULL AUTO_INCREMENT,
  `OrgID`           int NOT NULL,
  `ingredient_id`   int NOT NULL,
  `movement_type`   varchar(20) NOT NULL,   -- OPENING | RECEIPT | PRODUCTION_USE | ADJUSTMENT | RECEIPT_VOID | BATCH_REVERSAL | MIGRATION
  `qty_base`        decimal(18,4) NOT NULL, -- signed: + in, − out
  `unit_cost_base`  decimal(18,8) NOT NULL DEFAULT 0,
  `value`           decimal(14,4) NOT NULL DEFAULT 0, -- qty_base × unit_cost_base (signed)
  `balance_after`   decimal(18,4) NOT NULL, -- running balance for this material
  `avg_cost_after`  decimal(18,8) NOT NULL,
  `source_type`     varchar(20) NOT NULL,   -- grn | batch | count | opening | migration
  `source_id`       int NOT NULL,
  `note`            varchar(255) NULL,
  `created_by`      int NOT NULL,
  `created_at`      datetime NOT NULL,
  PRIMARY KEY (`ledger_id`),
  KEY `idx_org_ing` (`OrgID`,`ingredient_id`,`ledger_id`),
  KEY `idx_source` (`source_type`,`source_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```
- **Balance** = `SUM(qty_base)` per material (or the latest `balance_after`; both must agree, and a nightly or on-demand integrity check compares them).
- Ledger rows are **never updated or deleted**. Corrections are new rows.
- All writes go through `MPPM_Ledger::post( array $movement )` inside the caller's transaction. It locks the material row (`SELECT … FROM ing_ingredients WHERE ingredient_id=%d AND OrgID=%d FOR UPDATE`), computes `balance_after` and `avg_cost_after`, inserts the row and updates `ing_ingredients.avg_cost_base`.

### 6.3 New table `ing_grn` (delivery header)
```sql
CREATE TABLE IF NOT EXISTS `ing_grn` (
  `grn_id`        int NOT NULL AUTO_INCREMENT,
  `OrgID`         int NOT NULL,
  `grn_no`        varchar(20) NULL,          -- assigned on approval (Section 6.8)
  `status`        varchar(12) NOT NULL,      -- draft | submitted | approved | rejected | voided
  `supplier_id`   int NULL,                  -- Market POS supplier, if available (Q3)
  `supplier_name` varchar(150) NULL,         -- free text fallback
  `invoice_no`    varchar(60) NULL,
  `date_received` date NOT NULL,
  `notes`         text NULL,
  `total_value`   decimal(14,2) NOT NULL DEFAULT 0,
  `created_by`    int NOT NULL,  `created_at`   datetime NOT NULL,
  `submitted_by`  int NULL,      `submitted_at` datetime NULL,
  `decided_by`    int NULL,      `decided_at`   datetime NULL,
  `self_approved` tinyint(1) NOT NULL DEFAULT 0,
  `reject_reason` varchar(500) NULL,
  `voided_by`     int NULL, `voided_at` datetime NULL, `void_reason` varchar(500) NULL,
  PRIMARY KEY (`grn_id`),
  KEY `idx_org_status` (`OrgID`,`status`),
  UNIQUE KEY `uq_org_grn` (`OrgID`,`grn_no`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 6.4 New table `ing_grn_lines` (draft and submitted lines; replaces `ing_received_draft`)
```sql
CREATE TABLE IF NOT EXISTS `ing_grn_lines` (
  `line_id`       int NOT NULL AUTO_INCREMENT,
  `grn_id`        int NOT NULL,
  `OrgID`         int NOT NULL,
  `ingredient_id` int NOT NULL,
  `qty_packs`     decimal(14,3) NOT NULL,
  `price_per_pack` decimal(14,4) NOT NULL,
  `line_total`    decimal(14,2) NOT NULL,
  `sort_order`    int NOT NULL DEFAULT 0,
  PRIMARY KEY (`line_id`),
  KEY `idx_grn` (`grn_id`), KEY `idx_org` (`OrgID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```
On **approval**, each line is copied into `ing_received` (with snapshots) and posted to the ledger as `RECEIPT`.
Drafts persist in the DB, so a page reload never loses work. The "Stock In" screen lists the user's open drafts. Drafts untouched for 30 days are auto-deleted (configurable).

### 6.5 New table `ing_recipe_header`
```sql
CREATE TABLE IF NOT EXISTS `ing_recipe_header` (
  `product_id`        int NOT NULL,
  `OrgID`             int NOT NULL,
  `yield_qty`         decimal(14,3) NOT NULL DEFAULT 1,
  `yield_label`       varchar(30) NULL,     -- e.g. "loaves", "plates", "tables", "bars"
  `labour_cost`       decimal(14,2) NOT NULL DEFAULT 0,  -- per yield
  `overhead_cost`     decimal(14,2) NOT NULL DEFAULT 0,  -- per yield
  `ready_after_days`  int NOT NULL DEFAULT 0,            -- curing / drying / setting time
  `instructions`      text NULL,
  `version`           int NOT NULL DEFAULT 1,
  `updated_by`        int NOT NULL, `updated_at` datetime NOT NULL,
  PRIMARY KEY (`OrgID`,`product_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 6.6 New tables for history, handovers and counts
```sql
CREATE TABLE IF NOT EXISTS `ing_recipe_history` (
  `history_id` int NOT NULL AUTO_INCREMENT, `OrgID` int NOT NULL, `product_id` int NOT NULL,
  `version` int NOT NULL, `snapshot_json` longtext NOT NULL,
  `changed_by` int NOT NULL, `changed_at` datetime NOT NULL,
  PRIMARY KEY (`history_id`), KEY `idx_org_prod` (`OrgID`,`product_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `ing_handover` (
  `handover_id`     int NOT NULL AUTO_INCREMENT,
  `OrgID`           int NOT NULL,
  `batch_number`    int NOT NULL,
  `item_id`         int NOT NULL,
  `qty_sent`        decimal(14,3) NOT NULL,
  `qty_accepted`    decimal(14,3) NULL,
  `unit_cost`       decimal(14,4) NOT NULL,
  `status`          varchar(20) NOT NULL,  -- pending | accepted | partly_accepted | rejected | cancelled | written_off
  `available_from`  date NOT NULL,         -- = batch ready_date (curing), or date made
  `difference_reason` varchar(500) NULL,
  `reject_reason`   varchar(500) NULL,
  `loss_value`      decimal(14,2) NOT NULL DEFAULT 0, -- (qty_sent − qty_accepted) × unit_cost
  `itemspurchase_id` int NULL,             -- POS record created on acceptance
  `created_by` int NOT NULL, `created_at` datetime NOT NULL,
  `decided_by` int NULL,     `decided_at` datetime NULL,
  PRIMARY KEY (`handover_id`),
  KEY `idx_org_status` (`OrgID`,`status`), UNIQUE KEY `uq_batch` (`batch_number`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `ing_stock_count` (
  `count_id` int NOT NULL AUTO_INCREMENT, `OrgID` int NOT NULL,
  `status` varchar(12) NOT NULL,          -- draft | submitted | approved | rejected
  `count_date` date NOT NULL, `notes` text NULL,
  `created_by` int NOT NULL, `created_at` datetime NOT NULL,
  `decided_by` int NULL, `decided_at` datetime NULL,
  PRIMARY KEY (`count_id`), KEY `idx_org` (`OrgID`,`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `ing_stock_count_lines` (
  `line_id` int NOT NULL AUTO_INCREMENT, `count_id` int NOT NULL, `OrgID` int NOT NULL,
  `ingredient_id` int NOT NULL,
  `system_qty_base` decimal(18,4) NOT NULL,   -- snapshot when the line is added
  `counted_qty_base` decimal(18,4) NOT NULL,
  `difference_base` decimal(18,4) NOT NULL,
  `reason` varchar(20) NOT NULL,              -- spoilage | spillage | damage | theft | count_error | sample_use | other
  `note` varchar(255) NULL,
  PRIMARY KEY (`line_id`), KEY `idx_count` (`count_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 6.7 Roles and sequences
```sql
CREATE TABLE IF NOT EXISTS `ing_user_roles` (
  `OrgID` int NOT NULL, `user_id` int NOT NULL, `role` varchar(20) NOT NULL,
  PRIMARY KEY (`OrgID`,`user_id`,`role`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `ing_sequences` (
  `OrgID` int NOT NULL, `seq_name` varchar(20) NOT NULL, `period` varchar(8) NOT NULL,
  `last_value` int NOT NULL DEFAULT 0,
  PRIMARY KEY (`OrgID`,`seq_name`,`period`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `ing_audit_log` (
  `audit_id` bigint NOT NULL AUTO_INCREMENT, `OrgID` int NOT NULL, `user_id` int NOT NULL,
  `action` varchar(40) NOT NULL, `object_type` varchar(20) NOT NULL, `object_id` int NOT NULL,
  `details_json` text NULL, `created_at` datetime NOT NULL,
  PRIMARY KEY (`audit_id`), KEY `idx_org_obj` (`OrgID`,`object_type`,`object_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 6.8 Human-readable numbers
- `MPPM_Sequence::next( string $name, int $org_id ): int` uses `INSERT … ON DUPLICATE KEY UPDATE last_value = LAST_INSERT_ID(last_value + 1)` and then `SELECT LAST_INSERT_ID()`. This is atomic and safe under concurrency. Period = year (`2026`).
- Delivery: `GRN-2026-00001`, assigned **on approval** (drafts have no number; the UI shows "Draft #grn_id").
- Batch: `B-2026-00001`, assigned on creation.
- Stock count: `SC-2026-00001`.
- Legacy GRNs (`3-1-290526`) are kept unchanged.
- Formats are filterable (`apply_filters('mppm_number_format', …)`).

### 6.9 Migration runner (v1.0.4 → v2.0.0)
`MPPM_Migrator::maybe_run()` runs on `admin_init` **and** on activation whenever `get_option('mppm_db_version') < MPPM_DB_VERSION`. Each step is idempotent, logged to `ing_audit_log`, and skipped if already applied (track completed steps in option `mppm_migrations_done`).

| Step | Action |
|---|---|
| M1 | Create all new tables (6.2–6.7). Add the new columns (6.1) with `add_column_if_missing`. |
| M2 | `DROP TRIGGER IF EXISTS trg_validate_grn_format` (kept from v1). |
| M3 | Fix `OrgID = 0` rows **only** if exactly one positive OrgID exists in `ing_ingredients`. Otherwise log a warning and show an admin notice listing the counts. **Never assign them blindly.** |
| M4 | Materials: if `purchase_unit` is `bag25kg` or `bag50kg`, set `pack_name='bag'`, `purchase_unit='kg'`, and `purchase_unit_qty` = 25 or 50 **only if** the existing value is NULL or 1. If it is something else, keep it, flag the material "needs review" and show it on the Today screen. Otherwise, NULL `purchase_unit_qty` → 1 and `pack_name` NULL → the content unit (treated as bought loose). Recompute `base_unit`. |
| M5 | Receipts: for each `ing_received` row, set `qty_packs = total_received`, snapshot the pack size and content unit from the material **as it is now** (this matches what v1 currently displays), and compute `qty_base` and `unit_cost_base = price ÷ base_per_pack`. Group rows by (OrgID, GRN) into `ing_grn` headers with status `approved` (or `voided` if voided = 1), `grn_no = GRN`, and set `grn_id`. |
| M6 | Recipes: for each product with lines, create `ing_recipe_header` with `yield_qty = 1` and `yield_label = NULL`, and set `qty_per_yield = qty`. Show the Today card "Review your {recipes}: they now support yields". |
| M7 | Batches: `batch_ref = 'B-LEGACY-' + batch_number`, `planned_qty = good_qty = batch_total`, `status = synced_to_inventory ? 'received' : 'handover_pending'`. Create `ing_handover` rows (status `pending`) for unsynced batches, with unit cost calculated by the **v2** cost formula from their deductions. |
| M8 | Ledger backfill per org and material in date order: non-voided receipts → `MIGRATION` (+qty_base, at the receipt's cost); deductions → `MIGRATION` (−qty_deducted_base, at the running average). Then set `avg_cost_base`. Verify that the ledger balance equals the v1-formula balance for every material and report any differences in an admin notice. |
| M9 | Copy any remaining `ing_received_draft` rows into `ing_grn` (status `draft`) and `ing_grn_lines`. After a successful migration, the old draft table is left in place but unused (dropped in v2.1). |
| M10 | Store `mppm_db_version = 2.0.0`. |

---

## 7. Information architecture (screens)

Replace the 10 v1 pages with **4 screens**. They are identical in WP Admin and in the `[mp_production]` shortcode: one renderer, one JS app (Section 11).

| # | Screen | Purpose | Replaces (v1) |
|---|---|---|---|
| 1 | **Today** | To-do list, alerts, "what can we make", recent activity | Dashboard, Feasibility |
| 2 | **Stock In** | Record deliveries, approve them, delivery history | Receive Stock, GRN Register |
| 3 | **Make** | Record batches, handovers waiting, batch history | Log Production, Batches |
| 4 | **{Materials} & {Recipes}** | Tabs: {Materials} (with stock), {Recipes}, Stock counts, Settings | Ingredients, Recipes, Stock Balance, Settings |

Navigation shows a **badge count** of items needing the current user's action, e.g. "Stock In (2)".

### 7.1 Global UX rules (apply to every screen)
1. **Plain language.** Never show these words in the UI: GRN (use "delivery"; show the GRN number only as a reference), sync, base unit, feasibility, partial, OrgID, draft token.
2. **Explain every number.** Each calculated value has a one-line grey helper sentence beneath it, e.g. "= 10 bags × 50 kg".
3. **Empty states teach.** Every empty list shows what it is for, plus a primary button to start.
4. **Confirm with a summary of effects**, not "Are you sure?". Example: "This adds 500 kg of Flour to stock (value ZMW 4,500) and sends it for approval."
5. **After every save, show next steps.** A success panel with 1–3 buttons for the logical next action.
6. **No browser `alert/confirm/prompt`.** Use the module's modal component.
7. **Validation inline**, next to the field, in plain words, before submit.
8. **Mobile first.** Works at 360 px width (workshop floors, kitchens). Touch targets ≥ 44 px.
9. **Searchable selects** (Select2 or an equivalent) for materials, products and people.
10. **Loading and error states** on every async action. Errors explain what to do ("Could not save. Check your connection and press Save again."). Never show raw SQL errors to users; log them and show a reference code.
11. **Accessibility:** labels on all inputs, colour never the only signal (icons + text), WCAG AA contrast.

---

## 8. Screen specifications

### 8.1 First-run setup guide
Shown on Today when the org has no materials, or when the user clicks "Setup guide".

| Step | Content | Done when |
|---|---|---|
| 1. What do you make? | Choose an industry profile (Section 3.1) with icons, then check the terminology preview | profile saved |
| 2. Add your {materials} | Inline material form (8.5.1), "Add another" | ≥ 1 material |
| 3. Create your first {recipe} | Recipe editor (8.5.2) | ≥ 1 recipe |
| 4. Enter opening stock | Table of materials with "How many {packs} do you have now?" and "What did one cost?" → posts `OPENING` ledger rows | completed or skipped |
| 5. Who does what? *(optional)* | Assign roles (Section 4) | completed or skipped |

A progress bar shows "Step 2 of 5". It can be dismissed and resumed.

### 8.2 Today
Sections, top to bottom:

**A. Needs your attention** (cards, filtered by the user's capabilities, most urgent first)

| Card | Condition | Button |
|---|---|---|
| "{n} deliveries waiting for approval" | submitted GRNs and user can approve | Review |
| "{product}: {qty} waiting to be received into the shop" | pending handover, `available_from ≤ today`, user can accept | Receive |
| "{product}: {qty} ready to hand over on {date}" | pending handover, `available_from > today` (curing) | View |
| "Your delivery draft from {date} is not submitted" | user's own drafts | Continue |
| "{material} is low: about {n} {pack_name}s left (enough for ~{x} {product})" | balance < reorder level | Record delivery |
| "{material} has run out" | balance ≤ 0 | Record delivery |
| "{product} has no {recipe}" | POS item flagged as produced (Q5) or recently sold, with no recipe | Create {recipe} |
| "{material} needs review after the upgrade" | migration flag | Fix |
| "Stock count submitted by {name}" | submitted count, user can adjust | Review |

**B. What you can make now:** a table of products with recipes: Product · Can make now · Runs out first · Cost per unit. Each row has a "Make" button that opens Make with the product pre-selected. The row is red when the answer is 0, with the line "Needs {qty} more {material}".

**C. Recent activity:** the last 10 audit events in plain sentences ("Mary approved delivery GRN-2026-00012 · 2 h ago").

**D. Quick actions:** Record a delivery · Make something · Count stock.

### 8.3 Stock In

Tabs: **New delivery** · **Waiting for approval ({n})** · **History**

#### 8.3.1 New delivery (create / edit draft)
**Header:** Date received* (default today, cannot be in the future) · Supplier (search existing, or type a new name) · Invoice / delivery note no. · Notes.
"Received by" = the logged-in user automatically (no dropdown).

**Lines:** a repeating row:

| Field | Behaviour |
|---|---|
| {Material}* | Searchable. The last option is "+ New {material}…", which opens the material form in a modal and returns with it selected. |
| How many {pack_name}s?* | Decimal ≥ 0.001. The label updates to the chosen material's pack name ("How many bags?"). |
| Price per {pack_name}* | ≥ 0. Pre-filled with the last approved price for this material. |
| (read-only) | Line total |

Below each line, a **live helper sentence** (all parts required):
> **10 bags × 50 kg = 500 kg** · ZMW 450.00 per bag = **ZMW 9.00 per kg** · Last price ZMW 430.00 (**▲ 4.7%**)
> After this delivery: **12.4 bags (620 kg)**, enough for about **4,960 loaves** of White Bread

Rules:
- Price change ≥ 20% from the last price → amber warning "Price is 25% higher than last time. Please check."
- The "enough for" part uses the product that consumes this material most (highest total usage in the last 30 days, otherwise the first recipe that uses it). It is omitted if no recipe uses the material.
- The same material twice in one delivery → warning "Flour is already on line 1. Combine them?" with a "Combine" button.

**Footer:** Delivery total · **Save draft** · **Submit for approval** (or **Save & approve** when the user can approve and either `require_second_approver` is off or they are the org's only user).

**Submit summary modal:** "You are sending 3 lines worth ZMW 12,450 for approval. Stock will increase only after approval." → Submit.

**After submit, success panel:** "Delivery sent for approval. {Approver names} will see it on their Today screen." Buttons: Record another delivery · Go to Today.

#### 8.3.2 Waiting for approval
A list of submitted deliveries: date · supplier · invoice · lines · total · submitted by. Opening one shows the read-only lines with the same helper sentences, plus a "Stock after approval" column.
Actions: **Approve** (summary: "This adds the following to stock: …") · **Reject** (reason required, returned to the submitter as an editable draft with the reason shown at the top) · **Edit** (approvers may correct quantities or prices before approving; changes are logged).

**On approve (one transaction):** assign the GRN number → copy lines to `ing_received` with snapshots → `MPPM_Ledger::post(RECEIPT)` per line (updates the average cost) → header `status=approved` → audit log → success panel: "Approved. Stock updated. You can now make up to: White Bread 4,960 · Buns 1,200." Buttons: Make something · Back to approvals.

#### 8.3.3 History
Filters: date range, supplier, material, status. Columns: GRN no. · Date · Supplier · Invoice · Lines · Total · Status · Approved by.
Row actions: **View** · **Print / PDF** · **Void** (requires `void_reverse`).

**Void rules:**
- Reason required (modal textarea).
- The system computes the effect: for each line, the material balance after reversal. If any balance would go **below zero**, show: "{material} from this delivery has already been used in batches B-2026-00031, B-2026-00033. Voiding will make the stock show −120 kg." Then require an extra "I understand" tick and the `void_reverse` capability.
- Posts `RECEIPT_VOID` ledger rows (negative, at the original line's unit cost). Sets `ing_received.voided=1` for compatibility.
- **"Correct this delivery"** button = void + open a new draft pre-filled with the same lines.

### 8.4 Make

Tabs: **New {batch}** · **Handovers ({n})** · **History**

#### 8.4.1 New batch (step by step on one page)

**Step 1: What are you making?** A searchable product select (only products with a recipe). Under the select, as soon as a product is chosen:
> You can make **up to 1,600 White Bread** right now. **Premix** runs out first.
> To make more, you need more Premix (each extra 400 needs 2.5 kg).

If there are no products with recipes, show the empty state "Create a {recipe} first" with a button.

**Step 2: How many?**
- "How many will you make?"* (planned qty).
- Shortcut buttons: "One {recipe} lot ({yield_qty})", "×2", "Maximum ({max})".
- Live requirements table:

| {Material} | Needed | You have | After | Status |
|---|---|---|---|---|
| Flour | 50 kg (1 bag) | 620 kg (12.4 bags) | 570 kg | ✔ Enough |
| Premix | 2.5 kg | 1.0 kg | — | ✖ Short by 1.5 kg |

"Needed" includes the waste allowance. A hover/tap shows "6 m + 15% offcuts = 6.9 m".

**Shortage rule (blocking):** if any line is short, the Save button is disabled and a panel shows:
> You don't have enough Premix to make 400. You can make **160** with what you have.
> [Make 160 instead] [Record a delivery] [I have the stock, it isn't recorded yet (manager)]

The third option needs `void_reverse` plus a reason (stored in `override_reason`). Stock may then go negative and a Today card will ask for a delivery or stock count to fix it.

**Step 3: Details**
- Date made* (default today, not in the future).
- {Producer}* (default: current user; searchable list of org users).
- Reference (optional: customer, order or job number). The placeholder depends on the profile ("Job / customer" for workshop).
- "Did everything go to plan?" toggle. When on, it reveals:
  - **Good quantity** (default = planned) and **Rejected / spoiled** (default 0),
  - **Actual material used**: an editable "Used" column, pre-filled with Needed, so the user can record that more or less was used.
- Labour cost and overhead cost: pre-filled from the recipe, scaled to the quantity, editable. **Hidden** unless the recipe has non-zero values or Settings → "Track labour & overhead" is on.
- Notes.

**Step 4: Review & save.** Summary modal:
> **Make 400 White Bread** (B-2026-00014)
> Uses: Flour 50 kg · Yeast 1 kg · Premix 2.5 kg · Water 30 l
> Cost: materials ZMW 810.00 + labour ZMW 0.00 = **ZMW 810.00 → ZMW 2.03 each**
> **400 White Bread will be sent to the shop to be received.** *(Ready from 21 Dec 2026, if curing applies)*
> [Save] [Back]

**On save (one transaction):**
1. Insert the batch (`ing_products`) with `batch_ref` and status `handover_pending`.
2. For each line: `MPPM_Ledger::post(PRODUCTION_USE, −actual_qty_base)` at the current `avg_cost_base`, and mirror the row into `ing_stock_deductions`.
3. Compute material, labour and overhead cost and unit_cost = total ÷ good_qty (Section 10.3).
4. If good_qty > 0: create `ing_handover` (pending, `available_from = ready_date or date made`). If good_qty = 0: status `written_off`.
5. Audit log.

**After save, success panel:**
> ✔ B-2026-00014 saved. 400 White Bread are waiting to be received by the shop.
> [Make another] [Receive it now] *(only if the user has `accept_handover` and second-person approval allows it)* [Go to Today]

#### 8.4.2 Handovers (the seamless link to Goods Received)
A list of pending handovers: product · qty · {batch} ref · made by · date made · available from · unit cost.
Curing items (available_from > today) appear greyed out with "Ready on {date}". Accepting them early requires a confirmation ("This item is meant to cure until … Accept anyway?").

For each pending handover:
- **Accept all**: summary "Add 400 White Bread to shop stock at ZMW 2.03 each (total ZMW 810.00)."
- **Accept a different number**: enter the counted quantity (0 < n < qty_sent) and a reason (required: damaged in transit / miscounted / samples / other) → status `partly_accepted`, `loss_value` recorded.
- **Reject**: reason required → status `rejected`, batch status `handover_rejected`. The producer sees a Today card offering **Send again** (new pending handover with the same quantity) or **Write off** (status `written_off`; the materials stay consumed and the loss value is recorded).

**On accept (one transaction):**
1. `INSERT INTO itemspurchase` with these fields only: `ItemID, QtyPurchased, PchsUnitPrice, DatePurchased, SupplierID, ReceivedBy, VerifierA, PaymentDate, workstation_id, OrgID`.
   - `QtyPurchased = qty_accepted`
   - `PchsUnitPrice = handover.unit_cost` (the cost of good units; losses are recorded separately as `loss_value`)
   - `DatePurchased` = acceptance date
   - `ReceivedBy` = accepting user; `VerifierA` = batch creator (**confirm with Q2**)
   - `SupplierID`, `PaymentDate`, `workstation_id`: same as the v1 behaviour (**confirm with Q2/Q4**)
   - **Do not** insert `GRN` (set by the core trigger `before_insert_itemspurchase_grn`), `TotalCost` or `AmountPaid` (generated columns).
   - Check `$result === false || $wpdb->last_error || ! $wpdb->insert_id` and roll back on failure (keep the v1 robustness check).
2. Update or insert `buyingprice` as in v1 (Section 10.4).
3. Store `itemspurchase_id` on the handover. Set handover and batch status. Set `ing_products.synced_to_inventory = 1` and `collected_by` = acceptor (legacy).
4. Audit log. Success panel: "400 White Bread added to shop stock."

#### 8.4.3 History
Filters: date range, product, status, producer. Columns: {Batch} ref · Date made · Product · Good qty · Rejected · Unit cost · Status · Made by · Received by.
Row actions: **View** (full detail including materials used and cost breakdown) · **Reverse** (`void_reverse`):
- **Allowed** while the handover is pending, rejected or written off: cancel the handover, post `BATCH_REVERSAL` ledger rows (+qty at the original cost), status `reversed`, reason required.
- **Blocked** once the goods are accepted into shop stock. The message says: "These goods are already in shop stock. Adjust them in Market POS first, then contact your manager." (v2.1 may automate this, see Q6.)
- Hard delete is removed from the UI.

### 8.5 {Materials} & {Recipes}

Tabs: **{Materials}** · **{Recipes}** · **Stock counts** · **Settings**

#### 8.5.1 {Materials} tab
**List** (the merged stock view). Columns: {Material} · In stock (packs + content) · Enough for (best product) · Avg cost (per content unit) · Value · Status. Search plus filter (All / Low / Out / Archived).
Status: ✔ OK · ⚠ Low (below reorder level) · ✖ Out · 🔍 Needs review.
Click a row to open the **material detail drawer**: summary, edit button, and a **movement history** (ledger) table: Date · What happened (plain sentence + link: "Delivery GRN-2026-00012", "Used in B-2026-00014 (White Bread)", "Stock count SC-2026-00002: spoilage") · In · Out · Balance after · Cost.

**Material form** (used here, in the setup guide and in the Stock In modal):

| Field | Notes |
|---|---|
| Name* | Unique per org (case-insensitive). Warn on near-duplicates ("Flour" vs "flour "). |
| How do you buy it?* | Pack name, free text with suggestions from the profile (food: bag, sack, bottle, tin, crate, tray, carton · workshop: plank, sheet, box, roll, tin, bag · chemical: drum, bag, bottle, jerrycan, carton). Tick box: "I buy it loose (by weight, volume, length…)". |
| How much is in one {pack}?* | Number + content unit (grouped select). Hidden when "loose" is ticked. |
| Warn me when below | Number of packs (optional) |
| Description | optional |

**Live explanation box** under the form (required):
> ✔ **1 bag = 50 kg.** You'll count bags when stock arrives. {Recipes} can use kg or g.

Workshop example: "✔ **1 plank = 2.4 m.** You'll count planks when stock arrives. Materials lists can use m, cm or mm."

**Edit rules:**
- Pack size and content unit can be changed freely **only if** the material has no ledger rows. Otherwise they are locked, with the note "Locked because stock has been recorded. Past deliveries keep the size they were received with." Changing them anyway is allowed for users with `setup`, through a confirm dialog explaining that **future** deliveries use the new size and past ones stay as received (snapshots guarantee this). Changing the content unit to a different unit group is never allowed once there are ledger rows.
- Delete: allowed only with no ledger rows and no recipe usage. Otherwise offer **Archive** (hidden from selects, history kept).

#### 8.5.2 {Recipes} tab
**List:** every POS product, with filters "Has {recipe}" / "No {recipe}". Columns: Product · Makes · Cost per unit · Selling price (from POS, Q8) · Margin % · Can make now · Last changed.

**Recipe editor:**
- Header sentence built from fields: **"This {recipe} makes [ 400 ] [ loaves ]"** (yield qty* + yield label, with the placeholder taken from the profile).
- Lines: {Material}* · Quantity* · Unit* (**only units compatible with the material's group**, defaulting to its content unit) · Waste % (hidden behind "+ add waste / offcut allowance"; shown by default for the workshop profile) · remove.
- Below each line, a helper: "= 0.125 kg per loaf · costs ZMW 1.13 per loaf".
- Extra costs (collapsible): Labour per {recipe} lot · Overhead per {recipe} lot · Ready after (days), with the helper "e.g. curing, drying, setting time. Goods can be handed over only after this".
- Instructions (optional free text).
- **Live cost panel** (side panel on desktop, bottom on mobile):
  > Cost per lot: **ZMW 810.00** · Per loaf: **ZMW 2.03**
  > Selling price: ZMW 3.50 · Margin: **42%** · Can make now: **1,600**
  > Biggest cost: Flour (56%)
  If the margin is < 0, it is shown in red with the line "You lose money on each unit at the current selling price."
- Validation: at least 1 line, no empty material, no duplicate material (offer to combine), qty > 0, unit compatible.
- On save: increment the version, write a snapshot to `ing_recipe_history`, and recompute `ing_recipe.qty`. Success panel: "Saved. You can make up to 1,600 now." Buttons: Make it now · Next product without a {recipe}.
- "Copy from another {recipe}" button (useful for product variants).
- "History" link shows previous versions (read-only, with who and when).

#### 8.5.3 Stock counts tab
- **New count:** choose "All {materials}" or select some. The table shows Material · System says · You counted (in packs, with an optional content-unit entry) · Difference · Reason (required when the difference ≠ 0).
- **Submit** → approver (`adjust_stock`; second person if required) → **Approve** posts `ADJUSTMENT` ledger rows at the current average cost. A summary shows the total value gained or lost.
- History list with the same View / Print actions.

#### 8.5.4 Settings tab (`settings` capability)
- Industry profile and terminology overrides (Section 3).
- Currency (if not taken from POS).
- Require second person to approve (deliveries, handovers, counts): on by default.
- Track labour & overhead: off by default (on by default for the workshop profile).
- Allow manager override of shortages: on by default.
- Price-change warning threshold: 20% by default.
- Draft auto-delete after N days: 30 by default.
- Roles: a table of org users × role check boxes.
- Plugin info (version, DB version, "Run integrity check" button → compares ledger sums with stored balances and reports per material).

**Settings must be saved per organisation** (option keys include OrgID) through the plugin's REST API, not via admin-ajax.

---

## 9. Business rules (consolidated)

1. **R1 Stock only moves through the ledger.** Every movement is a ledger row. There is no other way to change stock.
2. **R2 Deliveries change stock only on approval.** Drafts, submitted and rejected deliveries have no stock effect.
3. **R3 History is immutable.** Approved receipts store snapshots (pack size, unit, cost). Editing a material never changes past quantities or values.
4. **R4 Production cannot overdraw stock** unless a manager override with a reason is used.
5. **R5 Finished goods enter shop stock only through an accepted handover.** There is no manual sync button.
6. **R6 Separation of duties** where configured (Section 4.4).
7. **R7 Accepted goods cannot be reversed from this module** (v2.0).
8. **R8 Every write is scoped to the user's OrgID and logged to the audit log.**
9. **R9 Restaurants and "make on sale" (design only, Phase 3):** a product can be flagged `production_mode = on_sale`. Instead of batches, materials are deducted when Market POS records a sale of that product (`PRODUCTION_USE` with `source_type = 'sale'`). This needs the POS sales table hook (Q9). Prep items (sauces, dough, soap base) use normal batches; see 9.10.
10. **R10 Intermediate products (Phase 3):** a recipe can produce an intermediate (e.g. dough, sauce, soap base, an assembled frame) that is flagged `is_intermediate` and linked to a material record. Making it posts `PRODUCTION_USE` for its inputs and a `RECEIPT`-type ledger row (`source_type = 'batch'`) for the intermediate material at the batch unit cost, with **no handover**. Other recipes can then use it as a material.
11. **R11 Quantities:** material quantities are decimal. Product quantities are decimal in module tables, but **must be whole numbers when sent to `itemspurchase.QtyPurchased`** (int) until Q1 is answered. The UI enforces whole numbers for good_qty when the handover destination is POS.

---

## 10. Calculations

### 10.1 Conversions
```
factor(u)              = GROUPS[group(u)][u]                  // to base unit
base_per_pack          = pack_size × factor(content_unit)
receipt qty_base       = qty_packs × base_per_pack
unit_cost_base (receipt) = price_per_pack ÷ base_per_pack
recipe line base / unit  = (qty_per_yield × factor(recipe_unit) ÷ yield_qty) × (1 + waste_pct/100)
batch needs (base)       = recipe line base per unit × planned_qty
display in packs         = qty_base ÷ base_per_pack           // uses CURRENT pack size for display only
```

### 10.2 Weighted average cost (per material, per base unit)
On every **positive** movement (RECEIPT, OPENING, positive ADJUSTMENT, BATCH_REVERSAL):
```
if balance_before ≤ 0:  avg_after = unit_cost_in
else:                   avg_after = (balance_before × avg_before + qty_in × unit_cost_in) ÷ (balance_before + qty_in)
```
- Negative movements (PRODUCTION_USE, negative ADJUSTMENT) leave the average unchanged and are valued at `avg_before`.
- RECEIPT_VOID is valued at the voided line's original `unit_cost_base`. Recompute: if `balance_after > 0`, `avg_after = (balance_before × avg_before − qty_out × line_cost) ÷ balance_after`; otherwise keep `avg_before`. Clamp at ≥ 0.
- Store 8 decimals for cost per base unit.

### 10.3 Batch cost
```
material_cost = Σ |ledger.value| for this batch's PRODUCTION_USE rows   // actual used × avg at that moment
labour_cost   = recipe.labour_cost   × planned_qty ÷ yield_qty   (editable)
overhead_cost = recipe.overhead_cost × planned_qty ÷ yield_qty   (editable)
unit_cost     = (material_cost + labour_cost + overhead_cost) ÷ good_qty     // rejects raise the cost of good units
```
If good_qty = 0, then unit_cost = 0 and the whole cost is recorded as the loss value.

### 10.4 POS buying price
Same as v1: insert a `buyingprice` row for (item, org) if none exists. Otherwise update it only when `round(existing,2) ≠ round(unit_cost,2)`. (Whether this should instead be a weighted average with existing shop stock is Q10.)

### 10.5 Can make now / limiting material
```
for each recipe line: max_i = floor( balance_base_i ÷ per_unit_base_i )   (lines with per_unit_base = 0 are ignored)
can_make = min(max_i), limiting = argmin
"need more" for the next lot: (per_unit_base_limiting × yield_qty) shown in friendly units
```
A negative balance means `max_i = 0`.

### 10.6 Low stock
`status = out` if balance_base ≤ 0. `low` if reorder_level_packs set and balance_packs < reorder_level_packs. Otherwise `ok`. (The v1 global threshold is removed; the migration copies it into `reorder_level_packs` only where it makes sense: v1 compared it against content units, so convert it as `threshold ÷ pack_size` and round up to 1 decimal.)

### 10.7 Worked test cases (these MUST pass as automated tests)

**T1 Bakery: "White Bread", makes 400 loaves**

| Material | Pack | Price/pack | Cost per content unit | Recipe (per 400) |
|---|---|---|---|---|
| Flour | bag of 50 kg | 450.00 | 9.00 /kg | 50 kg |
| Yeast | pack of 20 kg | 1,200.00 | 60.00 /kg | 1 kg |
| Premix | pack of 2.5 kg | 300.00 | 120.00 /kg | 2.5 kg |
| Water | loose, l | 0.00 | 0 | 30 l |

- Batch cost for 400 = 450 + 60 + 300 + 0 = **ZMW 810.00** → **ZMW 2.025 → 2.03 per loaf** (store 2.0250).
- Stock: Flour 12 bags (600 kg), Yeast 1 pack (20 kg), Premix 4 packs (10 kg), Water 1,000 l.
  max: Flour 600 ÷ 0.125 = 4,800 · Yeast 20 ÷ 0.0025 = 8,000 · Premix 10 ÷ 0.00625 = **1,600** · Water 1,000 ÷ 0.075 = 13,333 → **can make 1,600, limiting Premix**.
- Display: Flour shows "12 bags (600 kg)".

**T2 Weighted average:** Flour balance 200 kg at 8.00/kg; approve a delivery of 10 bags × 50 kg at 450/bag (9.00/kg).
avg = (200 × 8 + 500 × 9) ÷ 700 = 6,100 ÷ 700 = **8.7143 per kg** (0.00871429 per g). Balance = 700 kg = 14 bags.

**T3 Restaurant: "Beef Stew", makes 20 plates**

| Material | Pack | Price/pack | Per content unit | Recipe (per 20) | Waste |
|---|---|---|---|---|---|
| Beef | loose, kg | 120.00 /kg | 120.00 | 3 kg | 0% |
| Onions | bag of 10 kg | 150.00 | 15.00 /kg | 1 kg | 10% |
| Tomatoes | crate of 25 kg | 500.00 | 20.00 /kg | 2 kg | 0% |
| Cooking oil | bottle of 5 l | 250.00 | 50.00 /l | 500 ml | 0% |

Cost for 20 = 360.00 + (1.1 × 15 = 16.50) + 40.00 + 25.00 = **441.50** → **22.075 per plate** (displayed ZMW 22.08). Tests the waste %, a ml recipe unit against l content, and a loose material.

**T4 Carpentry: "Coffee Table", makes 1 table; labour 200.00 per table**

| Material | Pack | Price/pack | Per content unit | Recipe (per 1) | Waste |
|---|---|---|---|---|---|
| Pine plank | plank of 2.4 m | 96.00 | 40.00 /m | 6 m | 15% |
| Nails | box of 1,000 unit | 150.00 | 0.15 /unit | 40 unit | 0% |
| Wood glue | bottle of 1 l | 80.00 | 0.08 /ml | 100 ml | 0% |
| Varnish | tin of 4 l | 480.00 | 0.12 /ml | 250 ml | 0% |

Material = (6.9 m × 40 = 276.00) + 6.00 + 8.00 + 30.00 = **320.00**; + labour 200.00 = **520.00 per table**.
Stock 10 planks (24 m): max from planks = floor(24 ÷ 6.9) = **3 tables**. "Needed" for 1 table shows "6.9 m (2.9 planks)". Tests length units, count units, waste and labour.

**T5 Soap: "Bar Soap", makes 100 bars, ready after 28 days**

| Material | Pack | Price/pack | Per content unit | Recipe (per 100) |
|---|---|---|---|---|
| Oil blend | drum of 25 kg | 1,250.00 | 50.00 /kg | 10 kg |
| Caustic soda | bag of 25 kg | 750.00 | 30.00 /kg | 1.4 kg |
| Water | loose, l | 0 | 0 | 3.8 l |
| Fragrance | bottle of 500 ml | 400.00 | 0.80 /ml | 200 ml |
| Wrappers | pack of 500 unit | 250.00 | 0.50 /unit | 100 unit |

Cost = 500.00 + 42.00 + 0 + 160.00 + 50.00 = **752.00** → **7.52 per bar**. A batch made 23 Nov 2026 gives a handover `available_from = 21 Dec 2026`; Today shows "ready to hand over on 21 Dec 2026"; accepting early asks for confirmation.

**T6 Rejects:** T1 planned 400, good 380, rejected 20 → unit_cost = 810 ÷ 380 = **2.1316**. The handover sends 380.

**T7 Partial acceptance:** a T1 handover of 400 at 2.025; shop counts 395 → `QtyPurchased = 395`, `PchsUnitPrice = 2.025` (rounded per the POS column), `loss_value = 5 × 2.025 = 10.13`.

**T8 Shortage block:** T1 stock, plan 2,000 loaves → blocked; the panel offers "Make 1,600 instead".

**T9 Void after use:** approve 1 bag of flour (50 kg) into zero stock, make 200 loaves (uses 25 kg), then void the delivery → warning that the balance becomes −25 kg; requires the `void_reverse` capability + tick; the ledger shows −50 kg; the balance is −25 kg (shown red).

**T10 Pack size change:** after receiving 10 bags of 50 kg (500 kg), change the pack size to 25 kg → the balance is still 500 kg (snapshot) and now displays as "20 bags (500 kg)".

---

## 11. Architecture and code structure

### 11.1 Target file layout
```
mp-production-manager/
  mp-production-manager.php          // bootstrap, constants, autoload, hooks only
  includes/
    core/      class-access.php  class-terms.php  class-format.php  class-uom.php
               class-sequence.php  class-audit.php  class-migrator.php  helpers.php
    domain/    class-materials.php  class-ledger.php  class-deliveries.php
               class-recipes.php  class-production.php  class-handover.php
               class-stock-count.php  class-costing.php  class-insights.php   // Today cards, can-make
    integration/ class-pos-bridge.php   // ONLY place that reads/writes Market POS core tables
    api/       class-rest-controller.php (+ one controller per domain)
    ui/        class-app-renderer.php    // single renderer used by admin page AND shortcode
  templates/   app-shell.php  today.php  stock-in.php  make.php  setup.php  partials/*.php
  assets/      js/app.js (modules: api, ui, today, stockin, make, setup)   css/app.css
  readme.txt   CHANGELOG.md
```
- **One app shell.** The admin menu registers a **single** page ("Production") that renders `MPPM_App_Renderer::render()`. The shortcode renders the same thing. The old 10 submenu pages are removed (keep hidden redirects from the old slugs to the right tab).
- Tabs are client-side (hash routing `#stock-in/approvals`) so links in Today cards deep-link.
- The JS keeps jQuery and Select2 (already used), split into modules inside one file or several files. No build step is required (Q11).

### 11.2 REST API, namespace `mp-production/v2`
All routes: `permission_callback` → `MPPM_Access::can(<cap>)`, a nonce via the `X-WP-Nonce` header, input validation through `args` schemas, and responses `{ success, data, message }`. Errors use `WP_Error` with a user-safe message and an `mppm_ref` code (details logged).

| Method | Route | Cap | Purpose |
|---|---|---|---|
| GET | `/bootstrap` | view | terms, currency, user caps, units, settings, counts for badges |
| GET | `/today` | view | attention cards, can-make table, activity |
| GET/POST | `/materials` | view / setup | list (with balances) / create |
| GET/PUT/DELETE | `/materials/{id}` | view / setup | detail / update / delete-or-archive |
| GET | `/materials/{id}/ledger` | view | movement history (paginated) |
| POST | `/materials/opening-stock` | setup | bulk OPENING rows |
| GET/POST | `/deliveries` | view / receive | list (filters) / create draft |
| GET/PUT/DELETE | `/deliveries/{id}` | view / receive | detail / update draft / delete draft |
| POST | `/deliveries/{id}/submit` | receive | |
| POST | `/deliveries/{id}/approve` | approve_delivery | |
| POST | `/deliveries/{id}/reject` | approve_delivery | `{reason}` |
| POST | `/deliveries/{id}/void` | void_reverse | `{reason, acknowledge_negative}` |
| GET | `/deliveries/{id}/void-preview` | void_reverse | balances after void |
| GET | `/deliveries/line-preview` | receive | helper sentence data `{ingredient_id, qty_packs, price}` |
| GET/POST | `/recipes` , `/recipes/{product_id}` | view / setup | list / get / save |
| GET | `/recipes/{product_id}/history` | view | |
| GET | `/production/plan?product_id&qty` | produce | requirements, can-make, shortages |
| POST | `/production` | produce | create batch (+ `override_reason`) |
| GET | `/production` , `/production/{batch}` | view | |
| POST | `/production/{batch}/reverse` | void_reverse | |
| GET | `/handovers` | view | pending/all |
| POST | `/handovers/{id}/accept` | accept_handover | `{qty_accepted?, difference_reason?, accept_early?}` |
| POST | `/handovers/{id}/reject` | accept_handover | `{reason}` |
| POST | `/handovers/{id}/resend` , `/writeoff` | produce | |
| GET/POST | `/counts` , `/counts/{id}` , `/counts/{id}/submit|approve|reject` | adjust_stock | |
| GET/PUT | `/settings` | settings | per-org settings, terms, roles |
| GET | `/integrity` | settings | ledger check |
| GET | `/export/{module}?format=csv|pdf` | view | stock, deliveries, batches, ledger |

The v1 routes (`mp-production/v1`) stay registered for **one release**, read-only (GET only, returning v2-computed data) and marked deprecated. All v1 write routes return 410 with the message "Please refresh the page — Production Manager has been updated."

### 11.3 POS bridge
`MPPM_POS_Bridge` is the **only** class allowed to touch Market POS core tables (`items`, `itemspurchase`, `buyingprice`, `org`, users and, later, sales). Methods: `get_products()`, `get_selling_price($item_id)`, `insert_purchase(array)`, `upsert_buying_price($item_id,$cost)`, `get_org_users()`, `get_suppliers()`. This isolates the open questions in one place.

---

## 12. v1 defects that v2 must fix (regression checklist)

| ID | v1 location | Defect | v2 fix |
|---|---|---|---|
| D1 | `class-inventory-sync.php::calculate_unit_cost` | Divides the price by the pack size only, not by pack size × unit factor, while the quantity is in base units. Unit cost is ~1000× too high when a pack size is set. It also uses unweighted `AVG(price)`. | Ledger-based weighted average (10.2, 10.3) |
| D2 | `class-stock-calculator.php::compute_balance` | Recalculates all history using the current pack size | Snapshots + ledger (R3) |
| D3 | `admin.js` settings handler | Posts to the unregistered admin-ajax action `mppm_save_settings` and blocks the PHP form, so settings never save | REST `/settings` per org |
| D4 | `class-feasibility-checker.php::get_all` | Uses the low-stock threshold instead of the feasibility threshold; the setting is unused | Setting removed; can-make logic per 10.5 |
| D5 | settings | `mppm_default_low_stock_threshold` is global across tenants | Per-material reorder level |
| D6 | `class-production-manager.php::log_batch` | Allows "short" batches, so stock goes negative | R4 blocking + override |
| D7 | `delete_batch` | Deletes synced batches without reversing POS stock | Reverse rules (8.4.3) |
| D8 | `class-shortcode.php::render` | Falls back to listing **all users of all orgs** | `POS_Bridge::get_org_users()` only, never unscoped |
| D9 | `mppm_get_org_id()` fallback 4 | Assigns an unresolved user to the first org found | Remove fallback 4. Return 0 and show "Your account is not linked to a business. Contact support." Block all actions. |
| D10 | draft endpoints | No capability checks, no validation; `remove_line` deletes any org's line by ID | Drafts are `ing_grn` + lines, scoped by OrgID and owner, with capability checks |
| D11 | `class-grn-generator.php` | Non-atomic sequence (duplicates under concurrency); uses server date, not received date | `MPPM_Sequence` (6.8) |
| D12 | integer columns | `total_received`, `batch_total` are int, so no decimals | Decimal columns (6.1) |
| D13 | `class-access-control.php` | Everyone logged in can do everything except void/delete | Roles and capabilities (Section 4) |
| D14 | `class-recipe-manager.php::save_recipe` | Accepts empty ingredient (0), duplicates, units only checked when there is a base unit; the transaction can be left open on early return | Full validation; always commit or roll back |
| D15 | `ing_received_draft` | A new token per page load means drafts are lost on refresh | DB drafts per user (6.4) |
| D16 | void GRN | Allowed with no check against consumed stock; browser `prompt()` | Void preview and rules (8.3.3) |
| D17 | `class-exporter.php` | Excel/PDF depend on missing libraries; `export_pdf` only for stock | CSV for all modules; PDF via a print-friendly HTML view (browser print) unless the libraries are bundled (Q12) |
| D18 | `admin.js` GRN list | Empty-state colspan 6 on a 7-column table | Shared table component |
| D19 | front/admin duplication | Two diverging UIs | Single renderer (11.1) |
| D20 | currency | Hard-coded "ZMW" | `mppm_currency()` (3.4) |

---

## 13. Coding rules (mandatory)

1. **PHP 7.4 compatible:** no `match`, enums, `readonly`, named arguments, constructor promotion, `str_contains`, nullsafe `?->` or union types. Typed properties and arrow functions are fine.
2. **WordPress 5.8+**. Use `$wpdb->prepare()` for every query with variables, including `LIMIT/OFFSET` (`%d`). Never interpolate request data.
3. **Every query on module tables filters `OrgID = %d`** using `mppm_get_org_id()`. If it returns 0, abort with 403.
4. **Transactions** (`START TRANSACTION` / `COMMIT` / `ROLLBACK`) around every multi-row write. Use `FOR UPDATE` row locks for ledger postings. Never return from inside a transaction without committing or rolling back. Wrap this in a helper `MPPM_DB::transaction(callable $fn)`.
5. **Check every write:** `$result === false || $wpdb->last_error` → throw → roll back.
6. **Capabilities** checked in `permission_callback`, never only in the UI.
7. **Escape all output** (`esc_html`, `esc_attr`, `esc_url`; JS `esc()` for every interpolated string). Sanitize all input (`sanitize_text_field`, `absint`, a decimal validator for quantities and money).
8. **Money:** store with decimal columns, compute in PHP with floats rounded only at presentation and storage boundaries (round to 2 for money, 4 for unit cost, 8 for cost per base unit). Never use JS numbers as the source of truth; the server recalculates everything.
9. **No hard-coded industry words** in strings (Section 3.3). **No hard-coded currency.**
10. **Text strings** wrapped in `__()` with text domain `mp-production-manager` (to allow future translation).
11. **Do not change Market POS core table schemas.** Only insert into `itemspurchase` and `buyingprice`, as specified. Never write `GRN`, `TotalCost` or `AmountPaid` in `itemspurchase`.
12. **Do not create DB triggers** (the hosting may lack the TRIGGER privilege; see v1 history).
13. **Migrations are idempotent** and run from the migrator only.
14. **Audit log** every create, approve, reject, void, reverse, accept, override and settings change.
15. Keep functions small. One class per domain concept. Document every public method with a docblock (purpose, params, return, side effects).
16. **Tests:** add a `tests/` folder with PHPUnit tests for `MPPM_UOM`, `MPPM_Costing`, `MPPM_Ledger` average-cost maths and can-make logic, covering T1–T10 (pure functions must not need WordPress). Provide a WP-CLI command `wp mppm selftest` that runs T1–T10 against a temporary org in a transaction and rolls back.

---

## 14. Delivery phases and work items

Each item lists its acceptance criteria (AC). An item is **done** when its AC pass, D-items in scope are fixed, and the coding rules are followed.

### Phase 1: Foundations and correctness
| ID | Work item | AC |
|---|---|---|
| P1-01 | Restructure the file layout (11.1), bootstrap, autoloader, `MPPM_DB` transaction helper | Plugin activates on PHP 7.4 and 8.x; v1 pages still load during the transition |
| P1-02 | Migrator + M1–M10 | Running twice changes nothing; ledger balances equal v1 balances for the sample data; admin notice reports results |
| P1-03 | `MPPM_UOM` v2 (5.1–5.2) + `MPPM_Format` (5.4) | Unit tests for conversions and display rules pass |
| P1-04 | `MPPM_Ledger` + `MPPM_Costing` (10.2–10.3) | T2, T1 cost, T3, T4, T5, T6 pass as unit tests |
| P1-05 | Terminology + profiles + currency (Section 3) | Changing the profile changes all labels; grep finds no hard-coded industry nouns in UI strings |
| P1-06 | Access control + roles (Section 4) | Each route rejects users without the capability (403); a no-roles org behaves as v1 |
| P1-07 | `MPPM_Sequence` + `MPPM_Audit` | 50 parallel `next()` calls give 50 unique numbers |
| P1-08 | Security fixes D8, D9, D10 | No route or screen can return another org's data or users |

### Phase 2: The four screens and the workflows
| ID | Work item | AC |
|---|---|---|
| P2-01 | App shell, single renderer, routing, modal/toast/table components, hidden redirects from v1 slugs | Same UI in admin and shortcode; works at 360 px |
| P2-02 | Materials tab + material form + live explanation + ledger drawer (8.5.1) | T10 passes; locked-edit rules enforced |
| P2-03 | Recipes tab + editor + live cost panel + history (8.5.2) | T1/T3/T4/T5 costs and can-make shown correctly; incompatible units not selectable |
| P2-04 | Stock In: drafts, line helper sentences, submit/approve/reject, history, void with preview (8.3) | T2 and T9 pass end to end; drafts survive reload; second-person rule enforced |
| P2-05 | Make: planner, shortage block + override, details, rejects/actual usage, save (8.4.1) | T1, T6 and T8 pass end to end |
| P2-06 | Handovers: accept / partial / reject / resend / write-off, POS bridge insert (8.4.2) | T7 passes; `itemspurchase` row correct; failure rolls everything back |
| P2-07 | Batch history + reverse rules (8.4.3) | Reversal blocked after acceptance; allowed before |
| P2-08 | Today screen: cards, can-make, activity, badges (8.2) | Each card appears or disappears with its condition; deep links open the right tab |
| P2-09 | First-run setup guide incl. opening stock (8.1) | New org can go from nothing to first batch without leaving the guide |
| P2-10 | Stock counts (8.5.3) | Approved count posts adjustments at the average cost; value shown |
| P2-11 | Settings tab via REST (8.5.4), integrity check | D3 fixed; settings are per org |
| P2-12 | Exports (CSV for stock, deliveries, batches, ledger; printable views) | Files open correctly in Excel (UTF-8 BOM) |
| P2-13 | Deprecate v1 routes (11.2), remove old pages | v1 write routes return 410 with a friendly message |

### Phase 3: Industry extensions (only on request)
| ID | Work item |
|---|---|
| P3-01 | Intermediate products / sub-recipes (R10) |
| P3-02 | Make-on-sale deduction for restaurants (R9), after Q9 |
| P3-03 | Material lots and expiry dates (food, chemicals), FEFO suggestions |
| P3-04 | Customer job orders for workshops: batch linked to a customer order, handover to "customer" instead of shop stock |
| P3-05 | Email / WhatsApp notifications for approvals and handovers |
| P3-06 | Reorder suggestions (average daily usage × lead time) |

---

## 15. Open questions (must be answered before the related work item)

| ID | Question | Blocks |
|---|---|---|
| Q1 | Is `itemspurchase.QtyPurchased` strictly integer? Can products sold by weight or volume (e.g. liquid soap by litre) be received with decimals? | R11, P2-06 |
| Q2 | Exact meaning of `itemspurchase.ReceivedBy`, `VerifierA` and `PaymentDate` in Market POS. Who should they be for production handovers? | P2-06 |
| Q3 | Does Market POS have a suppliers table (name, ID)? Should production use the "org as its own supplier" convention from v1 (`SupplierID = OrgID`)? | P2-04, P2-06 |
| Q4 | Should `workstation_id` be the user's OutletID (v1 behaviour)? Do multi-outlet orgs need production per outlet (separate material stock per outlet)? | P2-06, data model |
| Q5 | Is there a flag in `items` that marks products as "made in-house" vs "bought for resale"? If not, may we add a module-side table to mark them? | P2-03, Today cards |
| Q6 | Can the module post a negative or reversing entry into Market POS stock (to automate reversal after acceptance)? | v2.1 |
| Q7 | Where does Market POS store the org currency? | 3.4 |
| Q8 | Where is the product selling price stored (table/column)? | 8.5.2 margin |
| Q9 | Which table records POS sales lines, and is there a hook/trigger point for make-on-sale? | P3-02 |
| Q10 | Should `buyingprice` keep "last cost" (v1) or become a weighted average with existing shop stock? | 10.4 |
| Q11 | Is a JS build step (npm) acceptable, or must assets stay plain JS? (Default: plain JS + jQuery.) | P2-01 |
| Q12 | Should mPDF/PhpSpreadsheet be bundled, or is browser-print PDF acceptable? | P2-12 |
| Q13 | Are there other Market POS components that read `ing_received`, `ing_recipe.qty` or `ing_products.batch_total` directly? (v2 keeps those columns populated for safety.) | Migration |

---

## 16. Definition of done (whole v2.0)
- All Phase 1 and Phase 2 acceptance criteria pass; T1–T10 pass in `wp mppm selftest`.
- D1–D20 are verified fixed.
- A non-technical tester from each of the four example industries can, **without help and without leaving the four screens**: set up 3 materials, create a recipe, record and approve a delivery, make a batch, and receive it into shop stock.
- No user-facing technical jargon (7.1 rule 1); no hard-coded industry nouns or currency.
- Upgrade from v1.0.4 on a copy of production data completes with no balance differences, or with every difference explained in the migration report.
- `readme.txt` and `CHANGELOG.md` are updated (Stable tag 2.0.0).
