# Data-Modeling-From-Chaos-to-Star-Schema
A Power BI data modeling project transforming a disorganized 17-table dataset into a governed star schema, covering dimension/fact design, data quality remediation, DAX measures, and row-level security. The dataset simulates a B2B sales/marketing/fulfillment system and was intentionally structured to reproduce common production data issues.

## Power Query Organization
Queries are organized into numbered groups, separating the raw source layer from the tables built on top of it as the model grows:

| Group | Contents |
|---|---|
| `01_Stage` | Raw queries, staged exactly as received from source systems — no transformations applied|
| `02_Dimensions` | Descriptive, mostly static tables that provide context for analysis |
| `03_Facts` | Transactional/event tables — records of something that happened, holding measures and dates, connected to dimensions via foreign keys |
| `04_Support` | Tables that are neither dim nor fact (e.g., security) |
| `Other Queries` | Newly connected tables/sources not yet triaged into a dimension, fact, or support role — working backlog |

## Data Source
Initial exploration pass over all 23 raw tables in dataset.xlsx — understanding *before* changing anything. 
Goal: identify grain, candidate role (dimension / fact / junk / support), and known quality issues.

| # | Table | Grain (1 row =) | Candidate role | Known issues |
|---|---|---|---|---|
| 1 | `Address` | 1 address | Merge into `dim_customer` | — |
| 2 | `CAMPAIGN_LOG` | 1 campaign, 1 day | Split: `dim_campaign` + `fact_campaign_spend` | Mixes static attrs (name, budget) with daily transactions (spend, clicks, impressions) |
| 3 | `campaign_skus` | 1 campaign | Source for `fact_campaign_sku` | Product list stored as delimited string in one cell; header row missing |
| 4 | `cities` | 1 city | Merge into `dim_customer` | Header row stored as data (needs promote-headers) |
| 5 | `CUST_MASTER` | 1 customer (company) | `dim_customer` core | Contains test row (CustomerID = 999) to filter out |
| 6 | `customer_contacts` | 1 contact (many per customer) | Merge into `dim_customer` | Grain mismatch vs. customer — must filter to primary contact before merge or it fans out |
| 7 | `dim_order` | unclear — single unlinked ID column | Junk, drop | No relatable context |
| 8 | `exchange_rates` | 1 currency, 1 date | Support | Not yet applied to facts |
| 9 | `inventory` | 1 product (wide, 1 column per month) | `fact_inventory` | Wide format — needs unpivot |
| 10 | `invoice_lines` | 1 invoice line | `fact_invoices` detail | — |
| 11 | `INVOICES` | 1 invoice | `fact_invoices` header | — |
| 12 | `order_line_items` | 1 order line | `fact_sales` detail | — |
| 13 | `ORDERS_2025` | 1 order | `fact_sales` header source | Has `LegacyRef` column not present in 2026 |
| 14 | `ORDERS_2026` | 1 order | `fact_sales` header source | Missing `LegacyRef`, `GiftMessage`, `OrderNotes` vs. 2025 |
| 15 | `payments` | 1 payment | `fact_payments` | — |
| 16 | `products` | 1 product | `dim_product` core | — |
| 17 | `regions` | 1 region | Redundant — info already present via `cities` | Do not merge separately (duplicate context) |
| 18 | `sales_targets` | 1 month | Support / comparison fact (target vs. actual) | Missing 2025-01 and 2025-08 |
| 19 | `security` | 1 employee | Support (RLS) | Header row stored as data |
| 20 | `sheet1` | 1 shipment | Duplicate of `shipments` | Identical to `shipments` — import artifact, drop |
| 21 | `shipments` | 1 shipment | `fact_shipments` | — |
| 22 | `subcategories` | 1 subcategory | Merge into `dim_product` | Combined category|subcategory column, needs split |
| 23 | `user_details` | 1 customer (despite "user" naming) | Merge into `dim_customer` | Naming inconsistency: `user_id` here vs. `customer_id` elsewhere — same entity |

## Data Model Architecture
All relationships are one-to-many with single-direction filters flowing from dimension to fact. No fact-to-fact relationships exist; related facts share a conformed dimension (customer, product, or date) instead.
### Dimension Tables
| Table | Role | Notes |
|---|---|---|
| `dim_customer` | Standard | Consolidated from 6 source tables |
| `dim_product` | Standard | Deduplicated on business key (product code) |
| `dim_campaign` | Standard | Static attributes only |
| `dim_geo` | Role-playing | Connected to fact_sales twice (ship-to active, bill-to inactive via `USERELATIONSHIP`) |
| `dim_order_flags` | Junk | order_channel, status, priority — extracted from `fact_sales` to avoid low-cardinality flag columns living in the fact |

### Fact Tables
| Table | Grain | Type | Connects to |
|---|---|---|---|
| `fact_sales` | 1 order line | Transactional | dim_customer, dim_product, dim_date, dim_geo (ship/bill), dim_order_flags |
| `fact_order_process` | 1 order | Accumulating snapshot | dim_customer, dim_date |
| `fact_inventory` | 1 product, 1 month | Transactional | dim_product, dim_date |
| `fact_campaign_spend` | 1 campaign, 1 day | Transactional | dim_campaign, dim_date |
| `fact_promotion_coverage` | 1 campaign–product pair | Factless | dim_campaign, dim_product |
| `fact_sales_targets` | 1 month | Standalone | dim_date |

## Key Design Decisions

Points in the build where more than one valid approach existed, and the reasoning behind the option chosen.

**Entity naming resolved as "customer," not "user," despite the source system using both terms.**
`user_details` and `customer_contacts` referred to the same real-world entity under different names — a common symptom of multiple departments naming the same thing differently. Resolved by majority usage: most source tables referenced "customer," so the model standardized on `dim_customer` and treated "user" as a legacy synonym rather than building two parallel entities or defaulting to whichever term appeared first.

**RLS applied on `dim_customer`, not `dim_geo`.**
Both dimensions carry regional context and both connect to `fact_sales`. `dim_customer` also connects to `fact_order_process`, so filtering there secures two fact tables through a single role instead of one. Extending `dim_geo` into every fact just to create a security surface would have altered fact grain with no reporting benefit.

**`fact_order_process` built as an accumulating snapshot instead of separate order/shipment/invoice/payment facts.**
`orders`, `shipments`, `invoices`, and `payments` describe stages of the same business process. Keeping them as separate facts would require repeated fact-to-fact joins (via shared dimensions) to compute any cross-stage metric. Consolidating into a single row per order — one date column per stage — reduces cycle-time calculations to a single `DATEDIFF` instead of a multi-hop query.

**`dim_geo` implemented as a role-playing dimension rather than two separate tables.**
`fact_sales` requires both a ship-to and a bill-to location. Duplicating the geography table would mean maintaining two physically separate but logically identical dimensions. Instead, `dim_geo` connects to `fact_sales` twice — one active relationship (ship-to), one inactive (bill-to) — with the inactive path invoked via `USERELATIONSHIP()` in DAX measures that need billing geography.

**Low-cardinality order flags extracted into a junk dimension instead of left in the fact table.**
`order_channel`, `status`, and `priority` are non-numeric attributes with no natural home in `dim_customer` or `dim_product`. Leaving them in `fact_sales` would add row weight without adding measure value. Bundling them into `dim_order_flags` keeps the fact table numeric while preserving filter capability on these attributes.

**`fact_promotion_coverage` modeled as a factless fact, not as an attribute of `dim_campaign` or `dim_product`.**
The campaign-to-product relationship is many-to-many and carries no numeric value of its own — only the fact that an association exists. A factless fact (foreign keys only, no measures) answers the actual business question — "was this product part of this campaign" — without forcing that relationship onto either dimension as a false one-to-many attribute.

**`dim_date` built with `CALENDARAUTO()` instead of a static, hardcoded date table.**
A hardcoded date table requires manual maintenance as the business's date range grows, and is a common source of stale-dimension bugs in production models. `CALENDARAUTO()` derives the date range directly from the model's existing fact tables, so the dimension expands automatically as new data is loaded.

**Products matched on business key (product code), not on product name.**
The source `products` table had no surrogate ID, only a business key. Joining on name seemed simpler but two rows shared a product name under different codes with inconsistent attribute completeness — joining `fact_sales` to `dim_product` on name caused row fan-out and silently inflated total sales. Switching the join key to the business key, after isolating and resolving the duplicate, removed the ambiguity.

**Garbage source tables deleted outright; tables with content absorbed elsewhere deactivated instead.**
`dim_order` (a single, unlinked ID column) had no salvageable information and was removed from the model entirely. `invoices`, `invoice_lines`, `payments`, and `shipments`, by contrast, had their reportable content already captured in `fact_order_process` — these were deactivated rather than deleted, preserving the option to re-enable a source table if a future requirement needs stage-level detail not carried into the accumulating snapshot.

**Core measures collected in a disconnected `_Measures` table rather than built individually inside each fact table.**
With multiple people building reports on top of the same model, measures scattered across fact tables invite duplicate, slightly different versions of the same calculation with no single source of truth. A disconnected table holding all core DAX measures gives report builders one place to find, reuse, and maintain them.



## Tools Used
+ Power BI Desktop
+ Excel (source data)

