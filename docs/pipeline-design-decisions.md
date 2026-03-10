# Pipeline Design Decisions

## Document Information

| Field | Detail |
|-------|--------|
| **Project** | Microsoft Fabric End-to-End Analytics Pipeline |
| **Dataset** | Retail Analytics — 20,000 Orders, $29.3M Revenue |
| **Architecture** | Medallion (Bronze → Gold) on Microsoft Fabric |
| **Version** | 1.0 |

---

## 1. Architecture Pattern: Medallion (Bronze → Gold)

### Decision

Implemented a two-layer Medallion architecture: Bronze (raw) and Gold (clean), skipping the Silver layer.

### Rationale

- **Bronze layer** stores raw data exactly as ingested from source CSV files, converted to Delta format. No transformations applied. This preserves the original data for auditing and reprocessing.
- **Gold layer** stores cleaned, validated, and enriched data ready for the semantic model. Business logic (calculated columns, filtered records) is applied here.
- **Silver layer was intentionally skipped** because the source data is relatively clean (structured CSVs with consistent schemas). A Silver layer adds value when sources are messy (JSON APIs, unstructured logs, schema-on-read files). For structured retail data, Bronze-to-Gold is sufficient and reduces pipeline complexity.

### Trade-offs

| Approach | Pros | Cons |
|----------|------|------|
| Bronze → Gold (chosen) | Simpler pipeline, fewer failure points, faster development | Less granular lineage if complex transforms are needed later |
| Bronze → Silver → Gold | Better for messy/multi-source data, clearer separation of cleaning vs. enrichment | More tables to maintain, longer refresh chain, overkill for clean CSVs |

---

## 2. Storage Format: Delta Tables on OneLake

### Decision

All Lakehouse tables use Delta format stored on OneLake.

### Rationale

- **Delta format** provides ACID transactions, time travel (query previous versions), and schema enforcement — critical for enterprise data reliability.
- **OneLake** is Fabric's unified storage layer. All workloads (Pipelines, Dataflows, Notebooks, Semantic Models) read from the same physical storage with no data duplication.
- **Parquet underneath** — Delta tables are Parquet files with a transaction log. This means compatibility with Spark, SQL, and Power BI Direct Lake mode.

### Alternative Considered

| Option | Why Not Chosen |
|--------|---------------|
| CSV files in Lakehouse Files section | No schema enforcement, no transactions, poor query performance |
| SQL Database (Warehouse) | Adds complexity; Lakehouse with Delta provides sufficient query performance for this scale |
| External Azure Data Lake | Unnecessary for a Fabric-native pipeline; OneLake is built-in |

---

## 3. Ingestion: Data Pipeline with Copy Activity

### Decision

Used Fabric Data Pipeline with Copy Activity to ingest CSV files into Lakehouse Bronze tables.

### Rationale

- **Copy Activity** is the simplest ingestion method for file-to-table conversion. It handles CSV parsing, schema inference, and Delta table creation in one step.
- **No-code orchestration** — the pipeline designer is visual, making it maintainable by BI developers who may not write Spark/Python code.
- **Parallel execution** — multiple Copy Activities run simultaneously, ingesting all 7 files in approximately 2 minutes.

### Configuration Details

| Setting | Value | Why |
|---------|-------|-----|
| Source | Lakehouse Files (uploaded CSVs) | CSVs uploaded manually or via automated upload |
| Destination | Lakehouse Tables (Delta) | Automatic Delta conversion |
| Write mode | Overwrite | Full refresh — simpler than incremental for this data volume |
| Column mapping | Auto-detect | Source CSV headers match destination schema |
| Error handling | Fail pipeline on error | No partial loads — ensures data consistency |

### Alternative Considered

| Option | Why Not Chosen |
|--------|---------------|
| Spark Notebook ingestion | More flexible but requires Python/Spark skills; overkill for CSV-to-Delta |
| Dataflow Gen2 for ingestion | Possible, but Data Pipeline is purpose-built for data movement; Dataflows are better for transformation |
| Shortcut to external storage | Source is local CSV, not an external data lake |

---

## 4. Transformation: Dataflow Gen2

### Decision

Used Dataflow Gen2 (Power Query Online) for all data cleansing and enrichment between Bronze and Gold layers.

### Rationale

- **Power Query Online** is familiar to Power BI developers — same M language used in Power BI Desktop's Power Query Editor.
- **Visual, step-by-step transformations** — each step is documented in the Applied Steps pane, creating built-in data lineage.
- **Direct Lakehouse integration** — Dataflow Gen2 reads from and writes back to Lakehouse tables natively.

### Transformations Applied

| Table | Transformation | Business Reason |
|-------|---------------|-----------------|
| orders | Change order_date to Date type | Enables proper date filtering and time intelligence |
| orders | Remove rows where order_status = "Cancelled" | Cancelled orders should not appear in revenue calculations |
| customers | Create Full Name column (first_name + " " + last_name) | Cleaner display in reports and tables |
| customers | Filter out is_active = 0 | Inactive customers excluded from active customer counts |
| products | Filter out is_active = 0 | Discontinued products excluded from current product analysis |
| products | Create profit_per_unit = unit_price - unit_cost | Pre-calculated for performance; avoids row-level DAX |
| order_items | Create line_profit = line_total - line_cost | Pre-calculated profit per line item |
| All tables | Verify data types (dates, integers, decimals) | Prevents type mismatch errors in the semantic model |

### Alternative Considered

| Option | Why Not Chosen |
|--------|---------------|
| Spark Notebook transformations | More powerful for complex logic, but Power Query is sufficient and more accessible |
| T-SQL stored procedures (Warehouse) | Requires a separate Warehouse item; adds architectural complexity |
| Transform in Power BI Desktop (Power Query) | Moves transformation to the reporting layer, violating separation of concerns |

---

## 5. Data Model: Snowflake Schema (Star Schema Variant)

### Decision

Implemented a snowflake schema with two fact tables (orders, order_items) and five dimension tables.

### Schema Design

```
                    calendar
                       |
promotions --- orders --- customers
                |
           order_items --- products
                |
              stores
```

### Rationale

- **Two fact tables** — orders holds order-level data (totals, status, channel), while order_items holds line-level data (product, quantity, price). This is the standard retail data model pattern because one order can contain multiple products.
- **Star schema principles maintained** — each dimension connects to a fact table via a single foreign key. No dimension-to-dimension joins.
- **Calendar as a dedicated date dimension** — enables time intelligence DAX functions (YoY, MTD, QTD) and proper date sorting.

### Relationship Configuration

| From → To | Cardinality | Cross-Filter | Notes |
|-----------|------------|--------------|-------|
| order_items → orders | Many-to-One | Single | Multiple line items per order |
| order_items → products | Many-to-One | Single | Multiple items can reference same product |
| orders → customers | Many-to-One | Single | Multiple orders per customer |
| orders → stores | Many-to-One | Single | Multiple orders per store; nullable for online orders |
| orders → promotions | Many-to-One | Single | Multiple orders per promotion |
| orders → calendar | Many-to-One | Single | Multiple orders per date |

### Key Design Decision: Measure Table Placement

Revenue and cost measures use `SUM(order_items[line_total])` instead of `SUM(orders[subtotal])`. This is because filters from the products table flow through order_items but do not reach the orders table in a single-direction filter model. Placing revenue measures on order_items ensures they respond correctly to product and department filters.

---

## 6. Semantic Model: 19 DAX Measures

### Decision

Created 19 centralized DAX measures covering revenue, profitability, growth, customer metrics, and display labels.

### Rationale

- **Centralized measures** ensure every report page uses the same calculation logic. If the revenue formula changes, it is updated in one place.
- **Descriptive naming** — measure names use plain business language (e.g., "On-Time Delivery Rate" not "OTD_PCT") to optimize for Power BI Copilot natural language queries.
- **Display label measures** (e.g., "Revenue YoY Label", "Gross Profit Label") are separated from numeric measures so conditional formatting can reference the numeric version while the card displays the formatted text.

### Measure Categories

| Category | Count | Examples |
|----------|-------|---------|
| Core financial | 5 | Total Revenue, Total Cost, Gross Profit, Profit Margin %, AOV |
| Volume metrics | 3 | Total Orders, Total Items Sold, Completed Orders |
| Growth & comparison | 2 | Revenue YoY %, Revenue YoY Label |
| Operational | 3 | Return Rate, Return Rate Status, Promo Revenue |
| Display labels | 4 | Channel Count Label, Gross Profit Label, Prior Year Label, AOV Subtitle |
| Customer metrics | 2 | Total Customers, Revenue per Customer |

---

## 7. Reporting: Three-Page Dashboard

### Decision

Built a three-page Power BI report with page-level navigation and synced slicers.

### Page Architecture

| Page | Audience | Purpose | Key Visuals |
|------|----------|---------|-------------|
| Executive Overview | C-suite, VPs | High-level KPIs and trends | Smart Narrative, Area Chart, KPI cards with YoY |
| Customer & Regional | Sales managers, Regional leads | Customer segments and geographic performance | Tile Slicer, Top N table, Tier-colored bars |
| Product & Promotion | Merchandising, Marketing | Product performance and promotion ROI | Heatmap matrix, Data bars, Promotion bar chart |

### Modern BI Features Used

| Feature | Where | Why |
|---------|-------|-----|
| Smart Narrative | Page 1 | AI-generated text summary updates with filters |
| Area Chart (not Line) | Page 1 | Gradient fill shows volume, not just direction |
| Tile Slicer | Page 2 | Modern button-style filter replaces outdated dropdown |
| Synced Slicers | All pages | Year filter on Page 2 applies globally |
| Top N Filtering | Pages 2, 3 | Dynamic top 10 that updates with page filters |
| Conditional Bar Colors (fx) | All pages | Per-bar colors based on value using DAX rules |
| Data Bars in Tables | Page 3 | Mini progress bars for margin visualization |
| Heatmap Matrix (Gradient) | Page 3 | White-to-blue gradient shows revenue concentration |
| Drillthrough | Page 2 → Hidden detail page | Right-click customer for full detail view |
| Page Navigator | All pages | Tab buttons in header for seamless navigation |

---

## 8. Refresh Strategy

### Decision

Full refresh (overwrite) for all tables on a daily schedule.

### Rationale

- **Data volume is manageable** — 20,000 orders and 44,000 line items refresh in under 5 minutes. Incremental refresh adds complexity without meaningful performance gain at this scale.
- **Full refresh ensures consistency** — no risk of partial updates or orphaned records.
- **Simple error recovery** — if a refresh fails, re-run the full pipeline. No need to identify and replay only the failed increment.

### Refresh Chain

```
5:00 AM — Data Pipeline: Copy Activity (CSV → Bronze Delta tables)
5:05 AM — Dataflow Gen2: Transform (Bronze → Gold)
5:15 AM — Semantic Model: Process (Gold tables → DAX measures recalculate)
5:20 AM — Reports available with fresh data
```

### When to Consider Incremental Refresh

Incremental refresh becomes valuable when:
- Source data exceeds 1 million rows
- Refresh window is constrained (must complete in under 30 minutes)
- Source system supports change detection (timestamps or change data capture)
- Historical data is static and only recent periods change

---

## 9. Security: Row-Level Security Plan

### Decision

Row-Level Security (RLS) is designed but not fully implemented in the portfolio version. The architecture supports it.

### Planned RLS Roles

| Role | Filter Table | Expression | Use Case |
|------|-------------|------------|----------|
| Regional Manager | customers | [region] = USERPRINCIPALNAME() lookup | Sees only their region's data |
| Store Manager | stores | [store_id] = USERPRINCIPALNAME() lookup | Sees only their store's data |
| Department Lead | products | [department] = "Electronics" | Sees only their department's products |
| Executive | (no filter) | Full access | Sees all data |

### Why Not Fully Implemented

- RLS requires Azure AD security groups mapped to roles, which requires an organizational Power BI Pro/Premium environment.
- The portfolio version demonstrates the architecture and DAX filter expressions. Full implementation would occur during deployment to a production Fabric workspace.

---

## 10. Performance Optimization

### Decisions Made

| Optimization | Implementation | Impact |
|-------------|---------------|--------|
| Pre-calculated columns | profit_per_unit and line_profit created in Dataflow Gen2 | Avoids row-level DAX iteration; faster measure evaluation |
| Star schema design | Single join path from any dimension to fact | Query engine uses optimal hash joins |
| Hidden columns | All key columns hidden from report view | Reduces field list clutter; prevents accidental misuse |
| Measure format strings | Currency, percentage formats set on measures | Avoids FORMAT() function in display measures where possible |
| Import mode | All tables imported (not DirectQuery) | Fastest query performance for dashboard interactivity |
| Calendar table marked | calendar marked as official date table | Enables optimized time intelligence engine |

---

*This document should be updated as the pipeline evolves or scales to production workloads.*
