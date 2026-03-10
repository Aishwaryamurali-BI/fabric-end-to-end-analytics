# DAX Measures Catalog

## Document Information

| Field | Detail |
|-------|--------|
| **Project** | Microsoft Fabric End-to-End Analytics Pipeline |
| **Total Measures** | 19 |
| **Measure Table** | All measures stored on the orders table (or create a dedicated _Measures table) |
| **Naming Convention** | Plain business English, no abbreviations — optimized for Copilot NL queries |

---

## Quick Reference

| # | Measure | Returns | Expected Value | Used On |
|---|---------|---------|----------------|---------|
| 1 | Total Revenue | Currency | $29,258,166 | All pages |
| 2 | Total Cost | Currency | $13,611,690 | Page 1 |
| 3 | Gross Profit | Currency | $15,646,476 | Page 1 |
| 4 | Profit Margin % | Percentage | 53.5% | Pages 1, 3 |
| 5 | Total Orders | Integer | 20,000 | All pages |
| 6 | Average Order Value | Currency | $1,578.77 | Pages 1, 2 |
| 7 | Total Items Sold | Integer | 82,916 | Page 3 |
| 8 | Total Tax Collected | Currency | $2,194,363 | Reference |
| 9 | Total Shipping Revenue | Currency | $122,902 | Reference |
| 10 | Completed Orders | Integer | 10,015 | Reference |
| 11 | Return Rate | Percentage | 12.7% | Page 3 |
| 12 | Online vs InStore Revenue | Currency | ~$22.4M | Reference |
| 13 | Revenue YoY % | Percentage | 2.7% | Page 1 (internal) |
| 14 | Revenue YoY Label | Text | "▲ 2.7% YoY" | Page 1 KPI |
| 15 | Channel Count Label | Text | "across 5 channels" | Page 1 KPI |
| 16 | Gross Profit Label | Text | "$15.6M gross profit" | Page 1 KPI |
| 17 | Prior Year Label | Text | "vs $14.4M prior year" | Page 1 KPI |
| 18 | AOV Subtitle | Text | "per transaction" | Page 1 KPI |
| 19 | Total Customers | Integer | 3,000 | Page 2 |

---

## Core Financial Measures

### 1. Total Revenue

```dax
Total Revenue = SUM(order_items[line_total])
```

| Property | Value |
|----------|-------|
| **Returns** | Currency |
| **Expected Value** | $29,258,166 |
| **Format** | Currency ($), 0 decimal places |
| **Used On** | All pages — KPI cards, bar charts, tables |
| **Responds to Filters** | Yes — department, region, tier, year, channel, promotion |

**Design Note:** This measure uses `order_items[line_total]` instead of `orders[subtotal]`. The order_items table is directly connected to the products table, so department and category filters work correctly. Using `orders[subtotal]` would cause every department to show the same $29.3M total because the product filter cannot reach the orders table through a single-direction relationship.

---

### 2. Total Cost

```dax
Total Cost = SUM(order_items[line_cost])
```

| Property | Value |
|----------|-------|
| **Returns** | Currency |
| **Expected Value** | $13,611,690 |
| **Format** | Currency ($), 0 decimal places |
| **Used On** | Internal — feeds Gross Profit and Profit Margin calculations |

**Design Note:** Same table placement rationale as Total Revenue. Uses order_items for correct filter propagation.

---

### 3. Gross Profit

```dax
Gross Profit = [Total Revenue] - [Total Cost]
```

| Property | Value |
|----------|-------|
| **Returns** | Currency |
| **Expected Value** | $15,646,476 |
| **Format** | Currency ($), 0 decimal places |
| **Used On** | Internal — feeds Profit Margin; also used in Gross Profit Label |

**Design Note:** References other measures rather than repeating SUM formulas. This means if Total Revenue or Total Cost logic changes, Gross Profit automatically updates.

---

### 4. Profit Margin %

```dax
Profit Margin % = DIVIDE([Gross Profit], [Total Revenue], 0)
```

| Property | Value |
|----------|-------|
| **Returns** | Percentage |
| **Expected Value** | 53.5% |
| **Format** | Percentage, 1 decimal place |
| **Used On** | Page 1 KPI card, Page 3 product table |

**Design Note:** DIVIDE handles division-by-zero safely (returns 0 instead of error). Always use DIVIDE instead of the / operator in DAX.

**Formatting:** If this measure shows as 0.535 instead of 53.5%, click the measure → Measure tools ribbon → Format → Percentage.

---

### 5. Total Orders

```dax
Total Orders = COUNTROWS(orders)
```

| Property | Value |
|----------|-------|
| **Returns** | Integer |
| **Expected Value** | 20,000 |
| **Format** | Whole number, thousands separator |
| **Used On** | Page 1 KPI card, AOV denominator |

**Note:** This counts rows in the orders table. It responds to date, customer, channel, and promotion filters. It does NOT respond to product/department filters unless bidirectional cross-filtering is enabled (by design — order count is an order-level metric, not a product-level one).

---

### 6. Average Order Value

```dax
Average Order Value = DIVIDE([Total Revenue], [Total Orders], 0)
```

| Property | Value |
|----------|-------|
| **Returns** | Currency |
| **Expected Value** | $1,578.77 |
| **Format** | Currency ($), 0 decimal places |
| **Used On** | Page 1 KPI card, Page 2 customer table (AOV column) |

---

## Volume Measures

### 7. Total Items Sold

```dax
Total Items Sold = SUM(order_items[quantity])
```

| Property | Value |
|----------|-------|
| **Returns** | Integer |
| **Expected Value** | 82,916 |
| **Format** | Whole number, thousands separator |
| **Used On** | Page 3 KPI card, product table |

**Critical Note:** This returns 82,916 (sum of the quantity column), NOT 44,152 (row count of order_items). Each order line can have quantity 1–5, so the sum of quantities exceeds the row count. If you need the number of line items processed, create a separate measure: `Total Line Items = COUNTROWS(order_items)`.

---

### 8. Total Tax Collected

```dax
Total Tax Collected = SUM(orders[tax_amount])
```

| Property | Value |
|----------|-------|
| **Returns** | Currency |
| **Expected Value** | $2,194,363 |
| **Format** | Currency ($), 0 decimal places |
| **Used On** | Reference — available for financial detail reports |

---

### 9. Total Shipping Revenue

```dax
Total Shipping Revenue = SUM(orders[shipping_amount])
```

| Property | Value |
|----------|-------|
| **Returns** | Currency |
| **Expected Value** | $122,902 |
| **Format** | Currency ($), 0 decimal places |
| **Used On** | Reference — available for operations reports |

---

### 10. Completed Orders

```dax
Completed Orders = 
    COUNTROWS(
        FILTER(orders, orders[order_status] = "Completed")
    )
```

| Property | Value |
|----------|-------|
| **Returns** | Integer |
| **Expected Value** | 10,015 (50.1% of total) |
| **Format** | Whole number |
| **Used On** | Reference — available for operations KPIs |

---

## Quality Measures

### 11. Return Rate

```dax
Return Rate = 
    DIVIDE(
        COUNTROWS(FILTER(orders, orders[order_status] = "Returned")),
        [Total Orders],
        0
    )
```

| Property | Value |
|----------|-------|
| **Returns** | Percentage |
| **Expected Value** | 12.7% (2,539 returns out of 20,000 orders) |
| **Format** | Percentage, 1 decimal place |
| **Used On** | Page 3 KPI card (with red accent when above 10% threshold) |

---

### 12. Online vs InStore Revenue

```dax
Online vs InStore Revenue = 
    CALCULATE(
        [Total Revenue],
        FILTER(orders, orders[channel] <> "In-Store")
    )
```

| Property | Value |
|----------|-------|
| **Returns** | Currency |
| **Expected Value** | ~$22.4M (all non-In-Store channels combined) |
| **Format** | Currency ($) |
| **Used On** | Reference — available for channel strategy analysis |

---

## Year-over-Year Measures

### 13. Revenue YoY %

```dax
Revenue YoY % = 
    VAR CurrentYear = CALCULATE([Total Revenue], calendar[year] = 2024)
    VAR PriorYear = CALCULATE([Total Revenue], calendar[year] = 2023)
    RETURN
        DIVIDE(CurrentYear - PriorYear, PriorYear, 0)
```

| Property | Value |
|----------|-------|
| **Returns** | Decimal (0.027) |
| **Expected Value** | 0.027 (2.7% growth) |
| **Format** | Percentage, 1 decimal place |
| **Used On** | Internal — feeds Revenue YoY Label |

**Design Note:** This uses hardcoded year values (2024, 2023) for simplicity. For a dynamic version that always compares the latest year to the prior year, use time intelligence functions like SAMEPERIODLASTYEAR. The hardcoded version is appropriate for a static portfolio dataset.

---

### 14. Revenue YoY Label

```dax
Revenue YoY Label = 
    VAR Pct = [Revenue YoY %]
    RETURN
        IF(
            Pct >= 0,
            "▲ " & FORMAT(Pct, "0.0%") & " YoY",
            "▼ " & FORMAT(ABS(Pct), "0.0%") & " YoY"
        )
```

| Property | Value |
|----------|-------|
| **Returns** | Text |
| **Expected Value** | "▲ 2.7% YoY" |
| **Format** | N/A (text) |
| **Used On** | Page 1 Total Revenue KPI card — Reference label |

**Design Note:** Returns formatted text with an arrow indicator. Positive growth shows ▲ (up arrow), negative shows ▼ (down arrow). The KPI card reference label color should be set to green (#2AA876) for positive, red (#D94B4B) for negative.

---

## Display Label Measures

These measures return formatted text strings for use in KPI card reference labels. They are intentionally separated from the numeric measures so conditional formatting can reference the numeric version.

### 15. Channel Count Label

```dax
Channel Count Label = 
    "across " & FORMAT(DISTINCTCOUNT(orders[channel]), "#") & " channels"
```

| Property | Value |
|----------|-------|
| **Returns** | Text |
| **Expected Value** | "across 5 channels" |
| **Used On** | Page 1 Total Orders KPI card — Reference label |

---

### 16. Gross Profit Label

```dax
Gross Profit Label = 
    FORMAT([Gross Profit] / 1000000, "$#,##0.0") & "M gross profit"
```

| Property | Value |
|----------|-------|
| **Returns** | Text |
| **Expected Value** | "$15.6M gross profit" |
| **Used On** | Page 1 Profit Margin KPI card — Reference label |

---

### 17. Prior Year Label

```dax
Prior Year Label = 
    VAR PY = CALCULATE([Total Revenue], calendar[year] = 2023)
    RETURN
        "vs " & FORMAT(PY / 1000000, "$#,##0.0") & "M prior year"
```

| Property | Value |
|----------|-------|
| **Returns** | Text |
| **Expected Value** | "vs $14.4M prior year" |
| **Used On** | Page 1 Total Revenue KPI card — additional Reference label |

---

### 18. AOV Subtitle

```dax
AOV Subtitle = "per transaction"
```

| Property | Value |
|----------|-------|
| **Returns** | Text (static) |
| **Expected Value** | "per transaction" |
| **Used On** | Page 1 Average Order Value KPI card — Reference label |

**Design Note:** This is a static text measure. It could be made dynamic (e.g., showing "per transaction | top 5% spend $X") but static is sufficient for the portfolio.

---

## Customer Measures

### 19. Total Customers

```dax
Total Customers = DISTINCTCOUNT(orders[customer_id])
```

| Property | Value |
|----------|-------|
| **Returns** | Integer |
| **Expected Value** | 3,000 |
| **Format** | Whole number, thousands separator |
| **Used On** | Page 2 KPI card |

**Design Note:** Uses DISTINCTCOUNT rather than COUNTROWS(customers) because it counts only customers who actually placed orders. If the customer table includes inactive customers who never ordered, COUNTROWS would overcount. DISTINCTCOUNT on orders[customer_id] gives the true active customer count within any filter context.

---

## Additional Page 2 & 3 Measures

These measures are created for specific KPI cards on Pages 2 and 3. They may use static text for simplicity.

### Revenue per Customer

```dax
Revenue per Customer = DIVIDE([Total Revenue], [Total Customers], 0)
```

| Expected Value | $9,753 | Page 2 KPI card |

### Return Rate Status

```dax
Return Rate Status = 
    IF([Return Rate] > 0.10, "Above 10% threshold", "Within target")
```

| Expected Value | "Above 10% threshold" | Page 3 KPI card — Reference label (red color) |

### Promo Revenue

```dax
Promo Revenue = 
    CALCULATE(
        [Total Revenue],
        FILTER(promotions, promotions[promotion_name] <> "No Promotion")
    )
```

| Expected Value | $2,724,221 | Page 3 KPI card |

### Promo Revenue Label

```dax
Promo Revenue Label = 
    FORMAT(DIVIDE([Promo Revenue], [Total Revenue], 0), "0.0%") & " of total revenue"
```

| Expected Value | "9.3% of total revenue" | Page 3 KPI card — Reference label |

### Items Sold Label

```dax
Items Sold Label = 
    "across " & FORMAT(DISTINCTCOUNT(order_items[product_id]), "#,##0") & " products"
```

| Expected Value | "across 294 products" | Page 3 KPI card — Reference label |

---

## Measure Formatting Reference

| Measure Type | Measure Tools Format | Display Units | Decimals |
|-------------|---------------------|---------------|----------|
| Currency (large) | Currency ($) | Millions | 1 |
| Currency (medium) | Currency ($) | Thousands | 0 |
| Currency (exact) | Currency ($) | None | 0 |
| Percentage | Percentage | None | 1 |
| Count | Whole Number | None or Thousands | 0 |
| Text labels | N/A | N/A | N/A |

**How to set format:** Click the measure in the Fields pane → Measure tools ribbon tab → Format dropdown → select the type. Then set decimal places in the same ribbon section.

---

## Measure Dependency Map

Some measures reference other measures. If you rename or delete a measure, these dependencies will break:

```
Total Revenue ← Gross Profit ← Profit Margin %
                             ← Gross Profit Label
             ← Average Order Value
             ← Revenue YoY % ← Revenue YoY Label
             ← Prior Year Label
             ← Promo Revenue ← Promo Revenue Label
             ← Online vs InStore Revenue
             ← Revenue per Customer

Total Orders ← Average Order Value
             ← Return Rate ← Return Rate Status

Total Customers ← Revenue per Customer
```

**Rule:** Never rename or delete Total Revenue, Total Cost, or Total Orders without checking all dependent measures first.

---

*This catalog should be updated whenever new measures are added to the semantic model.*
