# Price Guide Redesign

## 1) Current data model

Your current system models pricing as a deep hierarchy.

A measure sheet item has options and upcharges. Options and upcharges each have prices by office, and an upcharge can either target a specific option or fall back to a default; some upcharges are a percentage of the option’s price. We also keep normalized price rows so that reads and edits are consistent. A “normalized price row” is just one flat record that says “for this option or upcharge (and optionally its parent option), in this office, during this time window, the price is X (flat or percent).”

Stated more technically: In both Leap One Server and Leap 360 (Leap CRM), this core shape is consistent: a measure sheet item (`SSMeasureSheetItem`) has child price guide items for options (`SSPriceGuideItem` with `isAccessory=false`) and for upcharges (`SSPriceGuideItem` with `isAccessory=true`); those items store per‑office totals by office (`Office`), and a normalized price row (`PriceObject`) captures office‑scoped rows (including percent‑of‑parent for upcharges). Leap One Server uses V1 embedded arrays alongside V2 normalized price rows (`PriceObject`) for reads/exports, while Leap 360 documents the same semantics with RxDB/Parse schemas, including per‑option upcharge mapping via `accessoryPrices` and a `'default'` line, plus a forward‑looking note about eventually “dropping MeasureSheetItem” from fetch paths. A “normalized price row” is one tuple on `PriceObject` that encodes the surface `(company, office, item, optional parentItem, isPercentage, amount, typeCode/percentageTypeCodes, effective_from/to)` so each price surface is represented by a single current row per effective window.

Compatibility between upcharges and options is captured on each upcharge via `disabledParents`, which lists the option ids that must not be combined with that upcharge. At runtime the system prices a line by traversing from the chosen measure sheet item to its selected option, then to the applicable upcharges, and finally to office‑specific totals—favoring any per‑option matches first, otherwise falling back to a default line, and applying percent‑based upcharges against the option price when flagged.

### Steve’s critique of the current data model

- **Incorrect relationship modeling for offices/markets**: The way entities are related for office/market assignment is wrong. It should be a true many-to-many via a join table (associative collection) with indexed IDs, not the current association approach.
- **Overly deep hierarchical pricing structure causing combinatorial explosion**: Pricing is modeled ~4 levels deep (measurement → manufacturer → options → market-specific prices). This structure grows exponentially and becomes impossible to set up and manage.
- **Lack of safe bulk-edit workflows**: Today, large updates require showing all rows or running fragile scripts that risk breaking or overloading servers. The model and supporting UX should enable filtering, previewing a change set, and committing batch updates.
- **Relationship cascade patterns need rework**: Current “cascades down” approach contributes to depth and management pain; the structure of relationships should be changed.
- **Performance risk at scale without proper indexing**: At large record counts, naive scans cause outages; indexing (including compound indexes) must be first-class in the design.

> Notes on scope: Steve explicitly wants to preserve the conceptual separation between measurement (what’s measured) and options/price guide items (what’s selected and priced). This is a design constraint for any proposed new model.

### Current fallback behavior (today)
* Upcharges: if no per‑option price exists for an office (or a per‑option office total is 0 in export), the exporter falls back to that upcharge’s `'default'` totals for the office.
* Options: option totals are per‑office only in V1 arrays; there is no embedded default row for options.
* V2 `PriceObject`: consumers read office/item rows (and parent option for upcharges) and apply percent‑of‑parent when flagged. We generate one row per office for each option or upcharge (and a `parentItem` link for per‑option upcharges). Reads just use the row that matches the office; if no row exists, the amount is effectively 0. Percentage upcharges are flagged on the row and multiply against the option price at read‑time. There is no general “look elsewhere” fallback in V2 today.
* Why this wasn’t formalized before: V1 embedded arrays encoded scope explicitly (office, per‑option) and the exporter hard‑coded a simple rule (per‑option office total > 0 wins, else default). Options had no default surface, and V2 lacks default‑row semantics, so a general precedence chain wasn’t needed or available.

## 2) Proposed new design: side‑by‑side ER diagram comparison

Current (nested) model
```
Company
├─ Offices
│    └─ used inside arrays and PriceObject
├─ Measure Sheet Item
│    ├─ items[] → Price Guide Option
│    │     ├─ itemPrices[officeId,total]
│    │     ├─ accessories[] → Price Guide Item (isAccessory=true)
│    │     │     ├─ accessoryPrices[{priceGuideItemId, itemTotals[]}]
│    │     │     ├─ disabledParents[optionIds]
│    │     │     └─ percentagePrice flag
│    │     └─ (sometimes) parentItem pointer
│    └─ includedOffices[]
└─ PriceObject (normalized totals; office, item, optional parentItem)
```
Flattened relational model
```
Company
├─ Offices
├─ Measure Sheet Items
│   ├─ Product Options
│   └─ Add‑Ons (accessories, upcharges)
├─ measure_sheet_item_offices (MSI ↔ Office join)
├─ Prices (flat rows; option_id XOR addon_id, optional parent_option_id, office_id, effectivity)
└─ Option Conflicts (addon_id ↔ option_id; exceptions only)

Partner catalogs (manufacturer‑owned)
├─ Price Catalogs (manufacturer MSRP; link to Prices via catalog_id)
│   └─ Price Catalog Versions (publishable snapshots)
├─ Catalog Subscriptions (Dealer ↔ Catalog; mode real_time/deployed; multiplier; optional office scope)
├─ Catalog Deployments (deployment audit; lineage)
├─ Pricing Tiers (dealer segmentation by manufacturer)
├─ Dealer Tier Assignments (Dealer ↔ Tier; effective‑dated)
├─ Catalog Version Slices (category/subcategory slices per version)
└─ Subscription Version Overrides (Dealer/category overrides → specific version; office/date scoped)
```

**Price Catalogs** (table: `price_catalogs`) are manufacturer‑owned lists of MSRP prices that group normalized rows in the `prices` table. **Price Catalog Versions** (table: `price_catalog_versions`) are publishable snapshots a manufacturer can deploy to dealers. **Catalog Subscriptions** (table: `catalog_subscriptions`) link a dealer to a catalog and define the field `mode` (`real_time` or `deployed`). The field `mode` determines whether pricing is pulled live from the manufacturer catalog (`real_time`) or applied via deployed change sets into the dealer’s `prices` (`deployed`). The subscription also defines the field `multiplier_numeric` (for example, 0.40 for 40% of MSRP) and an optional field `office_id` scope. **Catalog Deployments** (table: `catalog_deployments`) record pushing a published version to selected dealers and set lineage fields on cloned price rows (fields: `source_version_id`, `source_price_id`). Together these enable one authoritative manufacturer catalog (field: `prices.catalog_id`) with either live read‑through pricing (apply `multiplier_numeric` at read time) or scheduled change‑set deployments with full traceability.

**Key reduction.** Pricing data moves from nested arrays and per‑item objects into a single normalized `prices` table; at runtime you join `measure_sheet_items` → (`product_options` or `add_ons`) → `prices`.

In our new design, the `measure_sheet_item_offices` table replaces the embedded `includedOffices[]` array so you assign an MSI to offices by adding join rows. The `prices` table is the one place all numeric amounts live; each row says “for this option or add‑on (and optionally a specific parent option), in this office (or all offices when `office_id` is NULL), during this date window, the price is X.” The `option_conflicts` table takes the idea of disallowed parent/child combinations and makes it a small lookup you can query, so compatibility is explicit and easy to enforce.

### Q&A: Answering potential questions about the new design for key stakeholders
- How do we model the Duration Series of shingles in this new design?   
Create a product option (table: `product_options`) under the relevant measure sheet item (table: `measure_sheet_items`) with name “Duration Series” (field: `brand` = “Owens Corning”, set the optional field: `sku`). In the `prices` table, write one default row (keep the field `office_id`  = NULL) for the base amount (field: `amount`); add office‑specific overrides only where markets differ (set the field: `office_id` = that office).

- How do we model an upcharge for the gold color of shingles in this new design?  
Create an add‑on (table: `add_ons`) under the same measure sheet item (table: `measure_sheet_items`) named “Gold Color.” If the upcharge applies to all options (table: `product_options`), write a default row in the `prices` table (table: `prices`) — set the field `addon_id` to the add‑on id and keep the field `parent_option_id` = NULL; if it only applies when a specific option is chosen, set the field `parent_option_id` = that option’s id. Set the field `is_percentage` = false for a flat amount or set the field `is_percentage` = true for percent‑of‑parent. Use `Option Conflicts` (table: `option_conflicts`) if the color is not allowed with certain options.

- How do we handle price changes to a single option in this new design?  
End‑date the current row in the `prices` table (table: `prices`) for the surface (fields: `company_id`, `option_id`, `office_id`) by setting the field `effective_to`; then insert a new row and set the field `effective_from` to the new start date (or insert first, then set the old row’s `effective_to`). Ensure no overlap between the new row’s (`effective_from`, `effective_to`) and existing rows for that same surface. Reads always select the single current row via the specificity and effectivity rules.

- How do we keep “measurements stay the same; options change” behavior?  
The measure sheet item (table: `measure_sheet_items`) remains the measurement source (field: `measure_uom`). Packages (concept) still pick an active product option (table: `product_options`) per MSI; product options (table: `product_options`) and add‑ons (table: `add_ons`) determine price while measurements remain stable.

- How do we assign items to different markets/offices and share across markets?  
Use the MSI↔Office join (table: `measure_sheet_item_offices`) to assign measure sheet items (table: `measure_sheet_items`) to offices (table: `offices`). Start with default prices in the `prices` table (table: `prices`; keep the field `office_id` = NULL), then create overrides only where a market differs (set the field `office_id` = that office) to avoid duplication.

- How are per‑option add‑on prices represented vs default add‑on prices?  
Per‑option add‑on rows set the field `parent_option_id` = the chosen option’s id (table: `product_options`); default add‑on rows keep the field `parent_option_id` = NULL. The precedence chain prefers office‑specific + per‑option over office‑specific + default, then office‑default + per‑option, then office‑default + default (fields: `office_id`, `parent_option_id`).

- How do we avoid combinatorial explosion from the old model? 
Store one default row in the `prices` table (table: `prices`) and only the minimal overrides needed (per‑option via the field `parent_option_id`, or per‑office via the field `office_id`). Move compatibility exceptions to `Option Conflicts` (table: `option_conflicts`) instead of embedding long lists.

- How do we bulk‑edit prices safely across many items/offices?  
Filter → preview a change set of rows in the `prices` table (table: `prices`; operations: insert/update/end‑date) → commit in transactions with non‑overlap checks. Keep covering indexes on the fields (`company_id`, `option_id`/`addon_id`, `parent_option_id`, `office_id`, `effective_from`, `effective_to`) to make this fast.

- How does this perform at scale and how do we index it? 
Add compound indexes on the `prices` table (table: `prices`) matching read/write keys (fields above); selection uses windowing on indexed columns. Office assignment uses the join table (table: `measure_sheet_item_offices`) with a primary key on (fields: `measure_sheet_item_id`, `office_id`).

- How do we import external price lists or AI‑generated scaffolds?  
Map products to product options (table: `product_options`) and add‑ons (table: `add_ons`), then write rows in the `prices` table (table: `prices`) — default first (keep the field `office_id` = NULL), then overrides (set the fields `office_id` and/or `parent_option_id`). Validate uniqueness and temporal non‑overlap per surface during ingest.

## 3) Proposed new design: overview

Flatten pricing into **two operational levels**:

* Level 1: **Measure Sheet Items** (what is measured and sold variants of).
* Level 2: **Product Options** (base choices) and **Add‑Ons** (accessories, upcharges).
* Put **all numeric prices** into one flat table (**Prices**) with columns for office, option or add‑on, and optional parent option. Defaults are expressed by `NULL` in the office column (for example, office is NULL = default across offices).
* Keep **compatibility** as a small, queryable join (**Option Conflicts**) only for exceptions (when an add‑on is not allowed with a specific option). If you rarely need exceptions, this table stays tiny.
* **Office availability** for a Measure Sheet Item is a true many‑to‑many join (no embedded arrays).
* Pricing fallbacks (office‑specific → default; per‑option add‑on → default add‑on) are handled by row ordering rather than deep traversal.

Partner catalogs and dealer modes (manufacturer → dealer):
* Introduce manufacturer‑owned catalogs (table: `price_catalogs`) and versioned snapshots (table: `price_catalog_versions`). MSRP entries live as normalized rows in the Prices table (table: `prices`) and are linked to a catalog via the field `catalog_id`.
* Dealers subscribe to catalogs via subscriptions (table: `catalog_subscriptions`) with the field `mode` = `real_time` or `deployed` and the field `multiplier_numeric` (for example, 0.40 for “40% of MSRP”), optionally scoped by the field `office_id`.
* Real‑time mode: dealer reads select the most specific active MSRP row from the manufacturer’s catalog (tables: `price_catalogs`/`prices`) and apply the field `multiplier_numeric` on the fly (same specificity and effectivity rules).
* Deployed mode: manufacturers publish a catalog version (table: `price_catalog_versions`) and create deployments (table: `catalog_deployments`) to selected dealers; rows are cloned into the dealer’s Prices table (table: `prices`) with lineage fields `source_version_id` and `source_price_id` so dealers read their own company‑scoped rows.

Contractor pricing tiers. Add Pricing Tiers (table: `pricing_tiers`) to represent Tier 1 / Tier 2 / Tier 3 segmentation per manufacturer, and Dealer Tier Assignments (table: `dealer_tier_assignments`) to assign a dealer to a tier with dates. Extend Price Catalog Versions (table: `price_catalog_versions`) with an optional field `tier_id` or model a join table `catalog_version_tiers` if a version can serve multiple tiers. Reads resolve the active tier for a dealer and prefer a tiered version when present, falling back to the base version otherwise.

Category‑scoped versioning. Add Catalog Version Slices (table: `catalog_version_slices`) to represent category or subcategory slices (for example, Roofing V2 while Windows remain V1). Add Subscription Version Overrides (table: `subscription_version_overrides`) so a dealer can point a specific category/subcategory (and optionally `office_id`) to a chosen version with an `effective_from` date—allowing pre‑staged switches on a future date without affecting other categories.

### How our design addresses Steve’s critiques
* Offices/markets are a true many‑to‑many join: Replace embedded office arrays with `measure_sheet_item_offices` so each MSI can be assigned/unassigned to offices by adding/removing join rows. Provide indexed queries (company_id + office_id + measure_sheet_item_id) and simple APIs for bulk assignment.
* Depth and combinatorics are flattened: Keep two operational levels (MSI → product_options/add_ons). Move all numeric prices to normalized rows (PriceObject v3) with explicit defaults and targeted overrides (office, per‑option). This eliminates deep traversal and the exponential edit surface.
* Safe bulk‑edit workflow is first‑class: Build filter → preview → commit flows over normalized price rows (filter by category, subcategory/brand, office set, is_percentage). Show a dry‑run diff (rows to insert/update/end‑date), then apply in a single transactional batch with guardrails (limits, rate control, rollback on failure).
* Relationship cascades are removed: No implicit “cascade‑down” updates. Edits target only the intended price rows (e.g., add an office‑specific override instead of rewriting embedded arrays). Sanity hooks remain limited to things like keeping Package active options valid; integrity fixes (e.g., delete option → remove option_conflicts and price rows) are explicit.
* Indexing and performance are designed in: Add compound indexes that match selection and updates across company, office, option/add‑on, parent_option, effective_from/effective_to. Prefer covering indexes on frequent reads (office + option/add_on) and partial indexes for active rows (date windows) to avoid scans.
* Preserve Steve’s separation of concerns: Measurement (MSI) remains distinct from options/add‑ons; packages continue to select an active option per MSI without changing the measurement itself. Package‑driven selection and stable measurements are retained, while pricing edit fan‑out is minimized via normalization and defaults.

### How our design addresses Bill's requirements
- Long‑running; one price guide across Leap products
  - Single canonical `prices` surface and shared entities (MSIs/options/add‑ons) act as the platform source of truth for all apps; no per‑app forks.

- Platform service; extensible for CRM
  - Clear read/write boundaries (current‑price selection, price upsert) and extensibility via `type_code`, `percentage_type_codes`, and `attributes_json` keep the schema stable as features grow.

- Consider CRM needs without building CRM first
  - Preserves Leap 360 semantics (per‑option add‑on pricing, defaults, percent‑of‑parent). CRM can adopt the same selection precedence without schema changes.

- ACID and fast multi‑join reporting
  - Relational schema with explicit keys and recommended compound indexes supports transactional integrity and performant reporting joins.

- Managed DB service (no self‑host)
  - Intended for AWS RDS/Aurora or equivalent; no server management assumed.

- ANSI SQL compliance (minimize vendor lock‑in)
  - Core queries use ANSI CTEs and window functions. Postgres‑specific features were replaced with an ANSI‑first transactional overlap check; JSONB usage is avoided in favor of normalized tables.


## 4) Proposed new design: schema discussion in detail

**measure_sheet_items**

* Fields: id, company_id, name, measure_uom (for example, EA, SQ, LF), category, subcategory, active, created_at, updated_at
* Meaning: a thing reps measure and sell variants of

**product_options**

* Fields: id, measure_sheet_item_id, name, brand, sku, attributes_json, active, created_at, updated_at
* Meaning: a base choice under a Measure Sheet Item (for example, Acme Classic Series)

**add_ons** (accessories, upcharges)

* Fields: id, measure_sheet_item_id, name, sku, measure_uom (optional, defaults to parent Measure Sheet Item), attributes_json, active, created_at, updated_at
* Meaning: a priced modifier that adds to or multiplies the base price. Percent-vs-flat is defined per price row on `prices.is_percentage`.

**prices** (the price matrix; one row = one price surface)

* Fields: id, company_id, office_id (nullable), option_id (nullable), addon_id (nullable), parent_option_id (nullable, only for per‑option add‑on pricing), is_percentage (boolean), amount, currency, effective_from, effective_to, created_at, updated_at, type_code (nullable), percentage_type_codes (nullable array), catalog_id (nullable), source_version_id (nullable), source_price_id (nullable)
* Constraints (conceptual): exactly one of option_id or addon_id must be present; parent_option_id may be present only when addon_id is set
* Meaning: the numeric price for either an option or an add‑on, optionally scoped to office; defaults expressed via NULLs. The optional `type_code`/`percentage_type_codes` preserve per‑component and percent‑of‑parent semantics we already use in V2.

- Denormalization note: `measure_sheet_item_id` is derivable through `option_id` or `addon_id` → their `measure_sheet_item_id`. Prefer omitting it to avoid drift. If you denormalize it for convenience, enforce correctness with a trigger and an invariant constraint.

##### Fallback precedence for prices (new design)
* Purpose: pick one “winning” price when multiple scopes could apply (office‑specific, per‑option, or defaults).
* Rule of thumb: more specific beats less specific. Specificity order: office > parent option > default (NULL).

Deterministic selection and tie-breakers

- Option prices (addon_id IS NULL):
  - Candidate set: rows with `option_id = chosen_option` and `(office_id = :office_id OR office_id IS NULL)` that are active as of `:as_of_date`.
  - Selection order: prefer `office_id = :office_id` over `office_id IS NULL`; within the same specificity, pick the row with the greatest `effective_from`; ties break on greatest `updated_at`; final tie-breaker on greatest `id` to ensure deterministic reads.

- Add‑on prices (addon_id IS NOT NULL):
  - Candidate set: rows with `addon_id = chosen_addon`, `(parent_option_id = chosen_option OR parent_option_id IS NULL)`, and `(office_id = :office_id OR office_id IS NULL)` active as of `:as_of_date`.
  - Specificity chain (from most to least specific):
    1) `(office_id = :office_id, parent_option_id = chosen_option)`
    2) `(office_id = :office_id, parent_option_id IS NULL)`
    3) `(office_id IS NULL, parent_option_id = chosen_option)`
    4) `(office_id IS NULL, parent_option_id IS NULL)`
  - Within a tie, order by greatest `effective_from`, then greatest `updated_at`, then greatest `id`.

- Uniqueness and temporal correctness: at most one active row per `(company_id, option_id, office_id)` for options and per `(company_id, addon_id, parent_option_id, office_id)` for add‑ons over any time window. Enforce “no overlapping effective date ranges” with exclusion constraints (see constraints section below). `is_percentage` only changes how the amount is applied, not which row wins.

- Why this matters: it eliminates sentinel values (“0 means default”), avoids data duplication across offices/options, and makes reads/editing deterministic.

**measure_sheet_item_offices**

* Fields: measure_sheet_item_id, office_id (composite primary key)
* Meaning: which offices sell a given Measure Sheet Item

**option_conflicts** (optional; accessories, upcharges compatibility exceptions)

* Fields: addon_id, option_id (composite primary key)
* Meaning: an add‑on is not allowed with a specific option (store only exceptions)

**offices**

* Fields: id, company_id, name, region_id, active, created_at, updated_at
* Meaning: organizational pricing scope

(Optionally) **pricing_adjustments** (simple “micro‑rules” if needed)

* Fields: id, company_id, measure_sheet_item_id (nullable), office_id (nullable), key (for example, PITCH, LAYERS, STORIES, MIN_CHARGE, WASTE_PCT, FLAT_FEE), condition_json, value_numeric, precedence, effective_from, effective_to
* Meaning: small deterministic adjustments without a heavy rules engine

**pricing_tiers** (contractor segmentation)

* Fields: id, owner_company_id, name, criteria_json, active, created_at, updated_at
* Meaning: represents Tier 1 / Tier 2 / Tier 3 (or similar) defined by a manufacturer

**dealer_tier_assignments** (effective‑dated tier per dealer)

* Fields: dealer_company_id, tier_id, effective_from, effective_to
* Meaning: assigns a dealer to a pricing tier with validity dates

**catalog_version_slices** (category‑scoped version slices)

* Fields: id, version_id, category, subcategory (nullable), notes
* Meaning: allows publishing a version that only covers a category/subcategory slice (e.g., Roofing V2)

**subscription_version_overrides** (dealer/category version pinning)

* Fields: catalog_id, dealer_company_id, category, subcategory (nullable), version_id, office_id (nullable), effective_from, effective_to
* Meaning: lets a dealer point a specific category/subcategory (optionally per office) to a specific version with an activation window

**price_catalogs** (manufacturer‑owned)

* Fields: id, owner_company_id, name, status ENUM(draft, published, archived), created_at, updated_at
* Meaning: a manufacturer’s MSRP catalog that groups normalized price rows

**price_catalog_versions**

* Fields: id, catalog_id, version, published_at, notes
* Meaning: a snapshot for deployable change sets

**catalog_subscriptions** (dealer enrollment and multipliers)

* Fields: id, catalog_id, dealer_company_id, mode ENUM(real_time, deployed), multiplier_numeric, office_id (nullable), allow_overrides BOOLEAN, effective_from, effective_to, created_at, updated_at
* Meaning: attaches a dealer to a catalog with operating mode and MSRP multiplier

**catalog_deployments** (audit and coordination)

* Fields: id, catalog_id, version_id, dealer_company_id, status ENUM(queued, running, succeeded, failed), started_at, completed_at, notes
* Meaning: deployment runs that clone MSRP rows into dealer companies with lineage


## 5) How data entities in the old design map to our proposed new design

* **SSMeasureSheetItem → measure_sheet_items**

  * `measurementType` → `measure_uom`
  * `includedOffices[]` → `measure_sheet_item_offices` rows
  * Category, subcategory map 1:1
  * Images/notes map to attributes or direct columns

* **SSPriceGuideItem (isAccessory=false) → product_options**

  * `displayTitle` → `name`
  * `subCategory2` (brand) → `brand`
  * `customRefId` (SKU) → `sku`
  * `itemPrices[{ officeId, total }]` → **prices** rows with `option_id` and `office_id` (default with `office_id=NULL`)

* **SSPriceGuideItem (isAccessory=true) → add_ons** (accessories, upcharges)

  * `percentagePrice` → removed at entity level; percent semantics live on `prices.is_percentage`
  * `accessoryPrices[{ priceGuideItemId (option), itemTotals[] }]` → **prices** rows with `addon_id` and either `parent_option_id` (per‑option) or `parent_option_id=NULL` (default)
  * `disabledParents[]` → **option_conflicts** rows (store only the exceptions).

* **PriceObject → prices (implemented as PriceObject v3)**

  * `office` → `office_id`
  * `item` (option or upcharge) → `option_id` or `addon_id`
  * `parentItem` (for upcharge) → `parent_option_id`
  * `isPercentage` → `is_percentage`
  * `amount` → `amount`
  * `typeCode` → `type_code`; `percentageTypeCodes` → `percentage_type_codes`

* **“Default upcharge line” semantics**

  * Old: special `'default'` record in `accessoryPrices` arrays
  * New: simple **prices** row with `addon_id=…`, `parent_option_id=NULL` (no magic identifiers)

* **Brand/SKU**

  * Old: fields on Price Guide Item
  * New: `brand`, `sku` on product_options (and optionally on add_ons)

* **Additional details**

  * Old: nested configs on items or Measure Sheet Items
  * New: keep as structured JSON on product_options/add_ons/measure_sheet_items or normalize if you need relational reporting

## 6) Case studies: how old Mongo queries in leap_one_server would be performed in our new data model

### 6.1) Sync check: Aggregate latest updates across MSIs, options, upcharges, and PriceObjects
When a salesperson opens the app, we quickly check if anything in the price guide changed for their office: every measure sheet item (MSI), its price guide options, and its upcharges. We turn that into a simple “last updated” map so the app only downloads what actually changed.

The query below builds a company-and-office scoped “what changed” snapshot across MSIs, options, upcharges, and office price totals. For context: it joins MSIs to their options and upcharges, converts arrays into id→timestamp maps with `$arrayToObject`, and pulls `PriceObject` update times for the selected office using `$lookup`, `$group`, and `$project`.

```js
// MongoDB (Aggregation Pipeline)
db.collection('SSMeasureSheetItem').aggregate([
  { $match: { _p_company: `Company$${companyId}`, "includedOffices.objectId": officeId } },
  { $lookup: { from: 'SSPriceGuideItem', localField: 'items.objectId', foreignField: '_id', as: 'items',
      pipeline: [ { $match: { _p_company: `Company$${companyId}` } }, { $project: { _updated_at: 1 } } ] } },
  { $lookup: { from: 'SSPriceGuideItem', localField: 'accessories.objectId', foreignField: '_id', as: 'accessories',
      pipeline: [ { $match: { _p_company: `Company$${companyId}` } }, { $project: { _updated_at: 1 } } ] } },
  { $project: { _id: 1, _updated_at: 1,
      items: { $arrayToObject: { $map: { input: { $ifNull: [ '$items', [] ] }, as: 'item', in: { k: { $toString: '$$item._id' }, v: { $toLong: '$$item._updated_at' } } } } },
      accessories: { $arrayToObject: { $map: { input: { $ifNull: [ '$accessories', [] ] }, as: 'accessory', in: { k: { $toString: '$$accessory._id' }, v: { $toLong: '$$accessory._updated_at' } } } } } } },
  { $group: { _id: null, measureSheetItems: { $push: { _id: '$_id', _updated_at: '$_updated_at' } }, items: { $push: '$items' }, accessories: { $push: '$accessories' } } },
  { $lookup: { from: 'PriceObject', as: 'priceObjects', pipeline: [
      { $match: { _p_company: `Company$${companyId}`, _p_office: `Office$${officeId}` } },
      { $addFields: { id: { $concat: [ '$_id', '-', { $substr: [ '$_p_item', 17, 10 ] }, '-', { $substr: [ { $ifNull: [ '$_p_parentItem', 'SSPriceGuideItem$' ] }, 17, 10 ] } ] } } } ] } },
  { $project: {
      measureSheetItems: { $arrayToObject: { $map: { input: '$measureSheetItems', as: 'item', in: { k: { $toString: '$$item._id' }, v: { $toLong: '$$item._updated_at' } } } } },
      options: { $mergeObjects: '$items' },
      upCharges: { $mergeObjects: '$accessories' },
      priceObjects: { $arrayToObject: { $map: { input: '$priceObjects', as: 'item', in: { k: { $toString: '$$item.id' }, v: { $toLong: '$$item._updated_at' } } } } } } }
])
```

**Citation**: `cloud/classes/MeasureSheetItem/PriceGuideDownload.cjs#checkPriceGuideSync()`

#### How we would do this in the new data model

* No array unwinds. Just grab `updated_at` per entity and the **most recent** applicable `prices` row for the user’s office (or the office-agnostic fallback).
* Because `prices` has rows for both `office_id = this office` and `office_id IS NULL` (default), we pick the **most specific** current row per option / add-on.

#### SQL (returns compact maps you can diff on the client)

```sql
-- Inputs
-- :company_id, :office_id, :as_of_date (e.g., CURRENT_DATE)

WITH current_price AS (
  SELECT p.*,
         ROW_NUMBER() OVER (
           PARTITION BY COALESCE(p.option_id, -1), COALESCE(p.addon_id, -1), COALESCE(p.parent_option_id, -1)
            ORDER BY (p.office_id IS NOT NULL) DESC,
                     p.effective_from DESC
         ) AS rn
  FROM prices p
  WHERE p.company_id = :company_id
    AND (p.effective_from <= :as_of_date AND COALESCE(p.effective_to, DATE '9999-12-31') > :as_of_date)
    AND (p.office_id = :office_id OR p.office_id IS NULL)
),
m AS (
  SELECT jsonb_object_agg(id::text, EXTRACT(EPOCH FROM updated_at)::bigint) AS msis_map
  FROM measure_sheet_items
  WHERE company_id = :company_id
    AND EXISTS (
      SELECT 1 FROM measure_sheet_item_offices mo
      WHERE mo.measure_sheet_item_id = measure_sheet_items.id
        AND mo.office_id = :office_id
    )
),
o AS (
  SELECT jsonb_object_agg(id::text, EXTRACT(EPOCH FROM updated_at)::bigint) AS options_map
  FROM product_options
  WHERE measure_sheet_item_id IN (
    SELECT id FROM measure_sheet_items msi
    JOIN measure_sheet_item_offices mo ON mo.measure_sheet_item_id = msi.id
    WHERE msi.company_id = :company_id AND mo.office_id = :office_id
  )
),
a AS (
  SELECT jsonb_object_agg(id::text, EXTRACT(EPOCH FROM updated_at)::bigint) AS addons_map
  FROM add_ons
  WHERE measure_sheet_item_id IN (
    SELECT id FROM measure_sheet_items msi
    JOIN measure_sheet_item_offices mo ON mo.measure_sheet_item_id = msi.id
    WHERE msi.company_id = :company_id AND mo.office_id = :office_id
  )
),
pmap AS (
  SELECT jsonb_object_agg(
           (COALESCE(option_id, -1)::text || '-' ||
            COALESCE(addon_id,  -1)::text || '-' ||
            COALESCE(parent_option_id, -1)::text || '-' ||
            COALESCE(office_id, -1)::text),
           EXTRACT(EPOCH FROM updated_at)::bigint
         ) AS prices_map
  FROM current_price
  WHERE rn = 1
)
SELECT m.msis_map, o.options_map, a.addons_map, pmap.prices_map
FROM m, o, a, pmap;
```

**Difficulty:** Easy. Simpler than the aggregation you had; the “fallback” is handled by the window function ordering.

---

### 6.2) Fetch normalized price rows and group by item and office
To price a chosen option across offices, we read the V2 price totals that were split out into `PriceObject`s and group them so each option shows its office-specific breakdowns. This powers consistent option and upcharge math.

The query below gathers each option’s price totals per office and groups them into per-item office breakdown arrays. For context: it filters by company, item pointers, and office pointers, then uses two `$group` stages to bucket by office and then by item, projecting the final nested shape.

```js
// MongoDB (Aggregation Pipeline)
db.collection('PriceObject').aggregate([
  { $match: { _p_company: `Company$${companyId}`, _p_item: { $in: itemRelIds }, _p_office: { $in: officeRelIds } } },
  { $project: { amount: 1, isPercentage: 1, typeCode: 1, _p_item: 1, _p_office: 1 } },
  { $group: { _id: { itemId: '$_p_item', officeId: '$_p_office' }, breakdowns: { $push: '$$ROOT' } } },
  { $group: { _id: '$_id.itemId', officePrices: { $push: { officeId: '$_id.officeId', breakdowns: '$breakdowns' } } } },
  { $project: { _id: 0, itemId: '$_id', officePrices: 1 } },
])
```

**Citation**: `cloud/classes/PriceObject/fetchPriceObjects.cjs#fetchPriceObjects()`

#### How we would do this in the new data model

* `prices` is already normalized. We just select the **current** price per `(option, office)` (with fallback) and group/shape it for the client.

#### SQL (per set of options and offices)

```sql
-- Inputs: :company_id, :option_ids[], :office_ids[], :as_of_date

WITH requested_offices AS (
  SELECT unnest(:office_ids::int[]) AS office_id
),
candidates AS (
  SELECT p.option_id,
         ro.office_id AS requested_office_id,
         p.office_id,
         p.is_percentage,
         p.amount,
         p.effective_from,
         p.updated_at,
         p.id
  FROM prices p
  CROSS JOIN requested_offices ro
  WHERE p.company_id = :company_id
    AND p.option_id = ANY(:option_ids)
    AND (p.effective_from <= :as_of_date AND COALESCE(p.effective_to, DATE '9999-12-31') > :as_of_date)
    AND (p.office_id = ro.office_id OR p.office_id IS NULL)
),
ranked AS (
  SELECT *,
         ROW_NUMBER() OVER (
           PARTITION BY option_id, requested_office_id
           ORDER BY (office_id IS NOT NULL) DESC,
                    effective_from DESC,
                    updated_at DESC,
                    id DESC
         ) AS rn
  FROM candidates
)
SELECT option_id,
       jsonb_agg(
         jsonb_build_object(
           'officeId', requested_office_id,
           'amount', amount,
           'isPercentage', is_percentage
         ) ORDER BY requested_office_id
       ) AS office_prices
FROM ranked
WHERE rn = 1
GROUP BY option_id;
```

**Difficulty:** Easy. This is what the table was designed for.

---

### 6.2.a) Real‑time dealer pricing from manufacturer catalog (MSRP × multiplier)
Dealers in `real_time` mode fetch MSRP rows from the manufacturer’s catalog and apply their subscription multiplier on read.

```sql
-- Inputs: :catalog_id, :dealer_company_id, :office_id, :as_of_date

WITH sub AS (
  SELECT multiplier_numeric
  FROM catalog_subscriptions
  WHERE catalog_id = :catalog_id
    AND dealer_company_id = :dealer_company_id
    AND mode = 'real_time'
    AND effective_from <= :as_of_date
    AND COALESCE(effective_to, DATE '9999-12-31') > :as_of_date
  ORDER BY effective_from DESC
  LIMIT 1
), ranked AS (
  SELECT p.option_id, p.addon_id, p.parent_option_id, p.is_percentage, p.amount,
         ROW_NUMBER() OVER (
           PARTITION BY COALESCE(p.option_id,-1), COALESCE(p.addon_id,-1), COALESCE(p.parent_option_id,-1)
           ORDER BY (p.office_id IS NOT NULL) DESC, p.effective_from DESC, p.updated_at DESC, p.id DESC
         ) rn
  FROM prices p
  WHERE p.catalog_id = :catalog_id
    AND (p.office_id = :office_id OR p.office_id IS NULL)
    AND p.effective_from <= :as_of_date
    AND COALESCE(p.effective_to, DATE '9999-12-31') > :as_of_date
)
SELECT option_id, addon_id, parent_option_id, is_percentage,
       amount * (SELECT multiplier_numeric FROM sub) AS dealer_amount
FROM ranked
WHERE rn = 1;
```

**Difficulty:** Easy. Reuses the same specificity selection; applies one multiplier.

---

### 6.2.b) Deployed change set: cloning MSRP rows to dealer with lineage
Manufacturers publish a version and deploy; deploys clone MSRP rows into dealer `prices` with lineage fields populated.

Pseudo-steps inside a transaction:

1. Resolve `version_id` for the catalog version to deploy.
2. Diff manufacturer rows vs latest deployed lineage for the dealer.
3. End‑date superseded dealer rows; insert new rows with `company_id = dealer_company_id`, `catalog_id = :catalog_id`, `source_version_id = :version_id`, `source_price_id = p.id`.

**Difficulty:** Moderate transactionally; straightforward with the existing normalized shape.

### 6.3) Query MSIs with multi-dimensional filters and optional ID-only mode
Admins can list measure sheet items for certain offices and product categories, optionally narrowing by subcategories, specific MSIs, or “items that have more than one option.” In some screens we only need the IDs for bulk actions.

The query below filters MSIs by office, category/subcategory, IDs, and item counts; it can return either paginated MSI documents or ID-only results. For context: it builds a `$match` with an office-aware branch (including a “no office” case), then conditionally applies `$project`, `$sort`, `$skip`, and `$limit` or an ID-only projection.

```js
// MongoDB (Aggregation Pipeline)
const matchStage = { _p_company: `Company$${companyId}` };
// Office filter (supports a special "**no-office**" token)
if (includedOffices.includes('**no-office**')) {
  matchStage.$or = [
    { includedOffices: { $size: 0 } },
    { 'includedOffices.objectId': { $in: includedOffices.filter((o) => o !== '**no-office**') } },
  ];
} else if (includedOffices.length) {
  matchStage['includedOffices.objectId'] = { $in: includedOffices };
}
if (categories.length) matchStage.category = { $in: categories };
if (subCategories.length) matchStage.subCategory = { $in: subCategories };
if (msItemIds.length) matchStage._id = { $in: msItemIds };
if (filterItemCountGreaterThanOne) matchStage.itemCount = { $gt: 1 };

const pipeline = [ { $match: matchStage }, /* optionally add $project, $sort, $skip, $limit or ID-only projection */ ];
```

**Citation**: `cloud/classes/MeasureSheetItem/fetchMeasureSheetItems.cjs#fetchMeasureSheetItems()`

#### How we would do this in the new data model

* Office availability is a **true M:N join** (`measure_sheet_item_offices`).
* All filters are plain `WHERE` clauses; “ID-only” is just a different projection.

#### SQL (IDs only and full rows variants)

**IDs only**

```sql
SELECT msi.id
FROM measure_sheet_items msi
LEFT JOIN measure_sheet_item_offices mo
       ON mo.measure_sheet_item_id = msi.id
WHERE msi.company_id = :company_id
  AND (:office_mode = 'ANY' OR mo.office_id = :office_id)
  AND (:categories_is_null OR msi.category = ANY(:categories))
  AND (:subcategories_is_null OR msi.subcategory = ANY(:subcategories))
  AND (:ids_is_null OR msi.id = ANY(:msi_ids))
  AND (
    :gt_one_option IS NOT TRUE OR (
      SELECT COUNT(*) FROM product_options po WHERE po.measure_sheet_item_id = msi.id
    ) > 1
  )
GROUP BY msi.id
ORDER BY msi.id
OFFSET :offset LIMIT :limit;
```

Note: With a true many-to-many join in `measure_sheet_item_offices`, there is no "no-office" token. Absence of a join row means the MSI is not available in that office.

**Full rows**

```sql
SELECT msi.*
FROM measure_sheet_items msi
JOIN measure_sheet_item_offices mo ON mo.measure_sheet_item_id = msi.id
WHERE msi.company_id = :company_id
  AND mo.office_id = :office_id   -- or adapt as above
  -- + same filters as needed
ORDER BY msi.updated_at DESC
OFFSET :offset LIMIT :limit;
```

**Difficulty:** Easy. No pipelines; standard indexed filters.

---

### 6.4) Discover existing linked fields tied to measure sheet via attribute IDs
When syncing measure-sheet-linked fields from an external system, we need to avoid creating duplicates. We figure out which attribute IDs we’ve already brought in so new ones can be added safely.

The query below finds existing linked fields for measure sheet by extracting and de-duplicating their attribute IDs. For context: it filters company records, unwinds the nested `destinationSettings.jobProgressMeasureSheet` array, `$group`s to accumulate distinct IDs, and projects a compact list for comparison.

```js
// Parse Aggregate (Mongo pipeline under the hood)
const pipeline = [
  { $match: { _p_company: `Company$${companyId}`, destinationSettings: { $exists: true }, 'destinationSettings.jobProgressMeasureSheet.attributeId': { $in: linkedFieldIds } } },
  { $limit: limit },
  { $unwind: '$destinationSettings.jobProgressMeasureSheet' },
  { $group: { _id: null, attributeIds: { $addToSet: '$destinationSettings.jobProgressMeasureSheet.attributeId' } } },
  { $project: { _id: 0, attributeIds: 1 } },
];
await new Parse.Query('LinkedValueObject').aggregate(pipeline);
```

**Citation**: `cloud/functions/JobProgress/LinkedValues.cjs#startSyncLeapCrmLinkedFields()`

#### How we would do this in the new data model

Two options:

**A) Keep JSONB (minimal change).**
Store integration settings as JSONB on a `linked_value_objects` table and query with JSONB operators.

```sql
-- Inputs: :company_id, :attribute_ids[], :limit

SELECT DISTINCT x->>'attributeId' AS attribute_id
FROM linked_value_objects lvo,
     LATERAL jsonb_path_query_array(lvo.destination_settings, '$.jobProgressMeasureSheet[*]') AS x
WHERE lvo.company_id = :company_id
  AND x->>'attributeId' = ANY(:attribute_ids)
LIMIT :limit;
```

**B) Normalize (preferred; makes this query unnecessary).**
Break out `linked_fields(company_id, measure_sheet_item_id, attribute_id, source_system, …)` and lookups become trivial:

```sql
SELECT DISTINCT attribute_id
FROM linked_fields
WHERE company_id = :company_id
  AND attribute_id = ANY(:attribute_ids)
LIMIT :limit;
```

**Difficulty:**

* (A) Easy with JSONB; similar complexity to the old pipeline.
* (B) Trivial if you normalize—arguably the old query becomes **unnecessary**.

Portability note: To maintain ANSI SQL portability and avoid vendor‑specific JSON functions, prefer approach (B) and keep integration structures normalized where feasible. If JSON storage is necessary, restrict to features available across common RDS engines and isolate JSON queries behind repository methods.

---

### 6.5) Compact map of option/upcharge last-updated timestamps for fast client diffs
To make the app’s “did this change?” check instant, we turn the options and upcharges under each MSI into a simple id→last-updated map that the client can diff quickly.

The query below converts the arrays of options and upcharges into compact id→timestamp maps. For context: inside the sync aggregation, a `$project` uses `$map` and `$arrayToObject` to emit `{ [id]: lastUpdated }` for options and upcharges.

```js
// Excerpt from the sync pipeline’s projection stage
{ $project: {
  items: { $arrayToObject: { $map: { input: { $ifNull: [ '$items', [] ] }, as: 'item', in: { k: { $toString: '$$item._id' }, v: { $toLong: '$$item._updated_at' } } } } },
  accessories: { $arrayToObject: { $map: { input: { $ifNull: [ '$accessories', [] ] }, as: 'accessory', in: { k: { $toString: '$$accessory._id' }, v: { $toLong: '$$accessory._updated_at' } } } } },
} }
```

**Citation**: `cloud/classes/MeasureSheetItem/PriceGuideDownload.cjs#checkPriceGuideSync()`

#### How we would do this in the new data model

* Use `jsonb_object_agg` over `product_options` and `add_ons`.
* Scope by MSIs that are available in the office.

#### SQL

```sql
-- Inputs: :company_id, :office_id

WITH msis AS (
  SELECT msi.id
  FROM measure_sheet_items msi
  JOIN measure_sheet_item_offices mo ON mo.measure_sheet_item_id = msi.id
  WHERE msi.company_id = :company_id AND mo.office_id = :office_id
),
opt AS (
  SELECT jsonb_object_agg(id::text, EXTRACT(EPOCH FROM updated_at)::bigint) AS options_map
  FROM product_options
  WHERE measure_sheet_item_id IN (SELECT id FROM msis)
),
addon AS (
  SELECT jsonb_object_agg(id::text, EXTRACT(EPOCH FROM updated_at)::bigint) AS add_ons_map
  FROM add_ons
  WHERE measure_sheet_item_id IN (SELECT id FROM msis)
)
SELECT opt.options_map, addon.add_ons_map
FROM opt, addon;
```

**Difficulty:** Easy. No array transforms needed.

---

#### Where the new model is simpler / harder / unnecessary

| Query                                      | New model status                                                                       |
| ------------------------------------------ | -------------------------------------------------------------------------------------- |
| 1) Sync check across entities & prices     | **Simpler.** Window function replaces complex lookups; fewer joins.                    |
| 2) Group normalized prices by item/office  | **Simpler.** Already normalized; grouping is straightforward.                          |
| 3) MSI filtering + ID-only                 | **Simpler.** Proper M:N join and indexed filters; no special “no-office” token needed. |
| 4) Discover linked fields by attribute IDs | **Unnecessary if normalized.** If kept in JSONB, still **easy** with JSONB functions.  |
| 5) Compact id→timestamp maps               | **Simpler.** Pure `jsonb_object_agg` from flat tables.                                 |

---

#### A note on performance & indexes

To keep these snappy at scale, add:

* `prices(company_id, option_id, office_id, effective_from, effective_to)` (partial index where `option_id IS NOT NULL`)
* `prices(company_id, addon_id, parent_option_id, office_id, effective_from, effective_to)` (partial index where `addon_id IS NOT NULL`)
* `prices(catalog_id, option_id, addon_id, parent_option_id, office_id, effective_from, effective_to)` (for manufacturer MSRP lookups)
* `catalog_subscriptions(catalog_id, dealer_company_id, mode, office_id, effective_from, effective_to)`
* `catalog_deployments(catalog_id, version_id, dealer_company_id, status)`
* `pricing_tiers(owner_company_id, active)`
* `dealer_tier_assignments(dealer_company_id, effective_from, effective_to)`
* `price_catalog_versions(catalog_id, tier_id, published_at)`
* `catalog_version_slices(version_id, category, subcategory)`
* `subscription_version_overrides(dealer_company_id, catalog_id, category, subcategory, office_id, effective_from, effective_to)`
* `product_options(measure_sheet_item_id, updated_at)`
* `add_ons(measure_sheet_item_id, updated_at)`
* `measure_sheet_item_offices(measure_sheet_item_id, office_id)` (PK on both)

These make the “specificity” windowing and office scoping cheap.

---

#### Temporal correctness and overlap guarantees (ANSI‑first)

Each price row is active on the date interval `[effective_from, effective_to)`, where `effective_to` is treated as `9999-12-31` when NULL. Two rows overlap if their active intervals intersect by at least one day.

Goal: at most one active row per key/time window for each surface:
- Options: `(company_id, option_id, office_id)`
- Add‑ons: `(company_id, addon_id, parent_option_id, office_id)`

ANSI‑portable enforcement strategy:
- Add UNIQUE indexes that prevent exact duplicates (e.g., include `effective_from` in the key) and perform a transactional overlap check before insert/update.
- Within a transaction, lock potentially overlapping rows and reject on any intersection using the De Morgan form of range non‑overlap.

```sql
-- Example (options surface): transactional check before insert/update
BEGIN;

-- Parameters: :company_id, :option_id, :office_id, :new_from, :new_to
SELECT id
FROM prices
WHERE company_id = :company_id
  AND option_id = :option_id
  AND addon_id IS NULL
  AND COALESCE(office_id, -1) = COALESCE(:office_id, -1)
  AND NOT (
    COALESCE(:new_to, DATE '9999-12-31') <= effective_from OR
    COALESCE(effective_to, DATE '9999-12-31') <= :new_from
  )
FOR UPDATE;

-- if any row returns, reject; otherwise proceed to insert/update
INSERT INTO prices (...)
VALUES (...);

COMMIT;
```

Shape and integrity checks (ANSI‑standard):

```sql
ALTER TABLE prices
  ADD CONSTRAINT price_surface_xor
  CHECK (
    (option_id IS NOT NULL AND addon_id IS NULL) OR
    (option_id IS NULL AND addon_id IS NOT NULL)
  );

ALTER TABLE prices
  ADD CONSTRAINT parent_only_for_addon
  CHECK (
    parent_option_id IS NULL OR addon_id IS NOT NULL
  );

-- Optional: ensure MSRP labeling and lineage shape
ALTER TABLE prices
  ADD CONSTRAINT catalog_linkage_consistency
  CHECK (
    (catalog_id IS NULL AND source_version_id IS NULL AND source_price_id IS NULL) OR
    (catalog_id IS NOT NULL)
  );
```

Note: If you deploy on Postgres, you may optionally replace the transactional check with exclusion constraints for convenience; the ANSI approach above keeps us portable across AWS RDS engines and minimizes vendor lock‑in.

These guarantees ensure selection logic has a single winner per key/time window and make writes fail fast on accidental overlaps.

Semantics: Effectivity uses half-open date intervals `[effective_from, effective_to)`. Reads should use `effective_from <= :as_of_date AND COALESCE(effective_to, DATE '9999-12-31') > :as_of_date` to include the start date and exclude the end date. If you later require time-of-day precision or multi-time-zone handling, migrate to `timestamptz` columns with the same half-open convention.


## 7) Price calculation: current design vs proposed design

**Current design**

1. Select Measure Sheet Item and an option.
2. Option price = lookup in `itemPrices` for the office (or `PriceObject`).
3. Determine applicable accessories (upcharges), exclude those with `disabledParents`.
4. For each accessory: choose per‑option price in `accessoryPrices` else `'default'`; pick office total; if `percentagePrice`, multiply by base option price.
5. Sum base + accessories; multiply by measured quantity.

**Proposed new design**

1. Select Measure Sheet Item and a product option.
2. Base price = most specific active row in **prices** (implemented via PriceObject v3) for (option, office) using the formal precedence chain above.
3. Determine selected add‑ons; if `option_conflicts` says blocked, skip.
4. For each add‑on: prefer per‑option row in **prices** (parent_option_id=option), else default add‑on row; if `is_percentage`, multiply amount by base option price.
5. Sum base + add‑ons; multiply by measured quantity.

Note: Always gate price reads by `measure_sheet_item_offices`. If a Measure Sheet Item is not assigned to the office, treat it as unavailable regardless of default prices.

## 8) Maintainability: old vs new (situation‑by‑situation)

1. Changing included offices on an MSI
- Old: Add office to MSI’s embedded `includedOffices`. Then, for every option and upcharge under that MSI, add per‑office totals (V1 arrays) or create `PriceObject` rows (V2). Often multiplicative by items × offices; needs copy utilities.
- New: Add a single join row in `measure_sheet_item_offices`. Prices use defaults unless you need a specific override for the new office; create only those override rows. Bulk tools can filter by MSI/category and create overrides in one batch.

2. Changing an option’s price across offices
- Old: Per office edit in `itemPrices` (V1) or one `PriceObject` per office (V2). Ten offices = ten edits/rows.
- New: Write one default option price row (office = NULL). Only create per‑office overrides where a market truly differs. Ten offices with the same price = one row; only outliers get overrides.

3. Changing an upcharge (default vs per‑option)
- Old: Two surfaces to manage (per‑option lines and a `'default'` line). Per‑option × office can explode combinatorially; exporter has the 0→default rule.
- New: One default add‑on row (office default with NULL), plus per‑option overrides only where needed. No sentinel “0 means default”; selection is deterministic via “more specific wins.”

4. Percent‑of‑parent upcharges
- Old: V1 uses `percentagePrice` with per‑option/default arrays; V2 flags `isPercentage` on rows and multiplies at read‑time.
- New: Same semantics, scoped cleanly: a percent row can be default (applies to all options) or per‑option (parent_option set). Edits are localized to those rows; no array surgery.

5. Deleting an option or upcharge
- Old: Option delete requires cleaning `disabledParents` on upcharges and removing `PriceObject` rows; upcharge delete requires removing its `PriceObject` rows. Embedded references and arrays make this cleanup error‑prone.
- New: Foreign‑key‑like behavior: remove `option_conflicts` rows and price rows via targeted deletes. No embedded arrays to sweep; hooks remain simple and bounded.

6. Adding a new office and copying pricing
- Old: Copy MSI office assignment and duplicate option/upcharge office totals (V1 arrays) or create many `PriceObject` rows (V2). Touches every MSI’s children; multiplicative.
- New: Add MSI↔Office join rows. Create default price rows once; generate office overrides only where the new market differs. Copy utilities can operate as “create overrides where source≠target,” not bulk duplicate‑all.

7. Package consistency after item changes
- Old: After MSI changes, ensure package items still point to valid active options; update runs through MSIs in a category.
- New: Same guard still applies, but fewer invalidations because option identity remains while prices move to normalized rows; fewer cross‑object writes reduce churn.

8. How many rows change (examples)
- Option price change, 10 offices: Old = 10 edits/rows; New = 1 default row + overrides only for outliers.
- Upcharge default change, 10 offices: Old = 10 totals in `'default'` line; New = 1 default row (office = NULL) for the add‑on.
- Upcharge per‑option across 25 options × 10 offices: Old = up to 250 totals; New = write only the 25 needed per‑option rows (default covers the rest), add office overrides only if markets differ.

Net effect: The new design replaces breadth‑first edits (every office/option cell) with targeted writes (defaults + minimal overrides), and provides filter/preview/commit tooling over normalized rows for safe bulk operations.

## 9) Advantages and potential cons

**Advantages**

* Dramatically simpler runtime path: two levels (Measure Sheet Item → Option/Add‑On → Prices) instead of deep nested traversal.
* One source of numeric truth: **prices** enables fast, indexed lookups and safe bulk edits (insert, update, preview, diff).
* Deterministic fallbacks via row ordering (office‑specific, per‑option add‑on) rather than ad‑hoc code paths.
* Correct many‑to‑many modeling for office availability.
* Cleaner analytics: straightforward grouping by office, option, add‑on, and date ranges.
* Compatibility exceptions are explicit, queryable rows (or omitted entirely if you do not need them).

**Potential cons / what you might lose**

* Some users are accustomed to seeing all pricing embedded under each item; moving totals into a central Prices table changes how admins think about edits.
* If you currently rely on special sentinel values (for example, “0 means use default”), those should be replaced by the explicit default row pattern (parent_option_id=NULL) and may require retraining.
* Migration effort: converting embedded arrays and PriceObject rows into flat Prices rows; backfilling `measure_sheet_item_offices`; optionally denormalizing or normalizing additional details.
* If you skip `option_conflicts` entirely, you lose fine‑grained per‑pair compatibility and must rely on category‑level or naming rules.

## 10) Migration feasibility for Leap CRM (Leap 360)

**Accuracy and alignment**: The Leap 360 research confirms the same primitives we use in Leap One Server: MSIs with `items` and `accessories`, per‑office totals in arrays, and a normalized `PriceObject` layer for office‑scoped rows and percent‑of‑parent. The proposed `prices` table is a direct consolidation of V2 `PriceObject` semantics with explicit defaults (office_id NULL) and per‑option add‑on scoping (parent_option_id).

**Feasible path**: Migration is straightforward with a staged rollout:
- Create `measure_sheet_item_offices`, `product_options`, `add_ons`, and `prices` alongside existing collections.
- Backfill joins from `includedOffices`; split `SSPriceGuideItem` into options vs add‑ons; convert `PriceObject` rows 1:1 into `prices` and expand any remaining V1 arrays (`itemPrices`, `accessoryPrices`) as rows. Choose `effective_from` policy (use source `updatedAt` or a fixed “now”) and leave `effective_to` NULL.
- Implement read selection as “most‑specific wins” (office > parent option > default) with deterministic tie‑breakers; then switch write paths to target single `prices` rows rather than embedded arrays.

Add partner catalogs and subscriptions:
- Create `price_catalogs`, `price_catalog_versions`, `catalog_subscriptions`, and `catalog_deployments`.
- For manufacturers, import MSRP rows into `prices` with `catalog_id` set and `type_code='MSRP'` (optional label).
- For dealers in `real_time` mode, read manufacturer catalog rows and apply multipliers at read time; for `deployed` mode, run deployments to clone MSRP rows into dealer `prices` with lineage fields populated.

Add tiers and category overrides:
- Create `pricing_tiers` and `dealer_tier_assignments`; populate dealer assignments based on business rules (volume bands, contracts) with dates.
- If tiered versions are preferred, add `tier_id` to `price_catalog_versions` (or `catalog_version_tiers` join) and publish per‑tier versions; otherwise consider `tier_adjustments` overlays if you want deltas.
- Create `catalog_version_slices` for categories/subcategories and `subscription_version_overrides` so dealers can pre‑stage category‑specific switches by `effective_from` (optionally per `office_id`).

**Potential gaps and decisions**:
- V1 arrays lack effective‑dating; set defaults as above. Upcharge exporter’s “0 means use default” should be retired in favor of explicit default rows.
- Leap 360’s note on “dropping MeasureSheetItem” affects fetch shape, not storage; the normalized `prices` surface remains valid either way.
- If staying on Parse/Mongo short‑term, enforce uniqueness and no‑overlap constraints at the app layer (compound keys + validation) to emulate the SQL constraints described earlier.

Outcome: You keep current business semantics from Leap 360 and Leap One Server, gain simpler edits and deterministic reads, and can phase the cutover behind flags without breaking existing exports/UI flows.

## 11) Migration feasibility for SalesPro (Leap One Server)

Leap One Server already encodes much of the “normalized prices” idea via `PriceObject`, so migration to the new `prices` surface is straightforward and low‑risk with a phased rollout. The steps below map directly from the existing code paths documented in `price-guide-research-leap-one-server.md`.

- Create new tables/collections alongside existing ones: `measure_sheet_item_offices`, `product_options`, `add_ons`, `prices`, (optionally) `option_conflicts`, and partner catalog tables (`price_catalogs`, `price_catalog_versions`, `catalog_subscriptions`, `catalog_deployments`).
- Backfill MSI↔Office join: expand `SSMeasureSheetItem.includedOffices[]` into rows in `measure_sheet_item_offices`.
- Split items by role: map `SSPriceGuideItem` with `isAccessory=false` to `product_options`; map `isAccessory=true` to `add_ons`. Preserve original ids for stable references in downstream systems.
- Migrate prices in two passes:
  - V2 first: copy `PriceObject` rows 1:1 into `prices` (item → `option_id`/`addon_id`, `parentItem` → `parent_option_id`, `office` → `office_id`, `isPercentage` → `is_percentage`, `amount`, `typeCode`/`percentageTypeCodes`). Set `effective_from` to the source `_updated_at` or a chosen cutover date; keep `effective_to` NULL.
  - V1 arrays next: expand `itemPrices[{ officeId, total }]` into option rows; expand `accessoryPrices[{ priceGuideItemId, itemTotals[] }]` into add‑on rows, using `parent_option_id` for per‑option entries and NULL for the old `'default'` line. Remove the “0 means default” exporter quirk in favor of explicit default rows.

- Introduce catalogs:
  - If acting as manufacturer, seed MSRP catalog rows (`catalog_id` set) and publish a first `price_catalog_version`.
  - If acting as dealer, enroll in `catalog_subscriptions` with mode and multiplier per dealer (and optionally per office).
  - For deployments, clone MSRP rows into dealer `prices` with `source_version_id` and `source_price_id` filled for lineage.

- Introduce tiers and category overrides:
  - Create `pricing_tiers`; assign dealers with `dealer_tier_assignments`.
  - Choose one strategy for tiered pricing: (A) tiered versions via `price_catalog_versions.tier_id` (or `catalog_version_tiers`), or (B) delta overlays via `tier_adjustments`.
  - For category slices, create `catalog_version_slices` and dealer‑specific `subscription_version_overrides` to schedule category‑level cutovers without changing other categories.
- Move compatibility to a join: convert `disabledParents[]` on upcharges to `option_conflicts` rows `(addon_id, option_id)` (store only exceptions).
- Update read paths behind a flag: replace `optionPrice()`/`accessoryPrice()` selection with the new precedence (office‑specific > parent option > default; tie‑break on `effective_from`, then `updated_at`, then id). Preserve percent‑of‑parent semantics (`is_percentage`) and package behavior (measurements remain stable, options/upcharges determine price).
- Update write paths: change save flows (e.g., `savePriceGuide`, `saveMeasureSheetItems`) to write targeted rows in `prices` instead of rewriting embedded arrays; keep dual‑write to old arrays temporarily if needed for backward compatibility.
- Sync/export surfaces: adapt `api/v1/PriceGuide.cjs` and `cloud/classes/MeasureSheetItem/PriceGuideDownload.cjs` to read from `prices` using “most‑specific wins.” Keep OpenAPI response shapes unchanged.
- Indexes and constraints: add compound indexes on `(company_id, option_id/addon_id, parent_option_id, office_id, effective_from, effective_to)`. If remaining on Parse/Mongo during transition, emulate temporal non‑overlap with application‑level checks and compound indexes; when moving to RDS, use the ANSI transactional overlap check.
  - Add indexes for catalogs/subscriptions as listed earlier; keep real‑time reads bounded by covering indexes on `prices(catalog_id, ...)`.
- Rollout strategy: enable by company/office via feature flags; run backfill per company, dual‑write for a short window, then flip reads; validate via diff tools (old exporter vs new selection) before removing legacy arrays.

Known nuances from the current system and how they are handled:
- Exporter fallback “per‑option office total 0 → use default”: replaced by explicit default rows (`parent_option_id=NULL`). Keep a compatibility shim if needed during cutover.
- Company guard on saves (`saveMeasureSheetItems`): preserved; `prices` is always scoped by `company_id`.
- Package/measurement separation: preserved; `measure_sheet_items` continues to anchor measurements while `product_options`/`add_ons` + `prices` handle pricing.
- Item reuse across MSIs is rare today; the new model treats options/add‑ons as children of an MSI as before, so no behavioral change is required.

Net: Because Leap One Server already uses `PriceObject` and has clear DTOs/exports for price guide, the backfill to `prices` plus the MSI↔Office join and `option_conflicts` can be delivered incrementally with minimal disruption and clear validation points.
