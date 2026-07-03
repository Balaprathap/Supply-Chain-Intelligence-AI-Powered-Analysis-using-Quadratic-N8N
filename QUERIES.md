# SQL Queries Reference

Common PostgreSQL queries for supply chain analytics. Run in Quadratic or any PostgreSQL client.

---

## Table of Contents
1. [KPI Summary](#kpi-summary)
2. [Customer Analysis](#customer-analysis)
3. [Order Performance](#order-performance)
4. [Regional Comparison](#regional-comparison)
5. [Trend Analysis](#trend-analysis)
6. [Data Quality](#data-quality)

---

## KPI Summary

### Overall Supply Chain Metrics
```sql
SELECT 
  COUNT(DISTINCT fa.order_id) as total_orders,
  COUNT(DISTINCT fa.customer_id) as unique_customers,
  COUNT(CASE WHEN fa.otif = 1 THEN 1 END) as otif_orders,
  COUNT(CASE WHEN fa.in_full = 1 THEN 1 END) as in_full_orders,
  COUNT(CASE WHEN fa.on_time = 1 THEN 1 END) as on_time_orders,
  ROUND(100.0 * COUNT(CASE WHEN fa.otif = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as otif_percent,
  ROUND(100.0 * COUNT(CASE WHEN fa.in_full = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as in_full_percent,
  ROUND(100.0 * COUNT(CASE WHEN fa.on_time = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as on_time_percent
FROM fact_aggregate fa;
```

**Expected Output (May 17, 2025):**
| total_orders | unique_customers | otif_orders | otif_percent | in_full_percent | on_time_percent |
|--------------|-----------------|-------------|--------------|-----------------|-----------------|
| 422 | 44 | 101 | 23.93 | 42.42 | 52.13 |

---

### KPI Breakdown by Status
```sql
SELECT 
  'OTIF' as metric,
  COUNT(CASE WHEN otif = 1 THEN 1 END) as success_count,
  COUNT(*) as total_count,
  ROUND(100.0 * COUNT(CASE WHEN otif = 1 THEN 1 END) / COUNT(*), 2) as percent
FROM fact_aggregate
UNION ALL
SELECT 
  'In-Full' as metric,
  COUNT(CASE WHEN in_full = 1 THEN 1 END) as success_count,
  COUNT(*) as total_count,
  ROUND(100.0 * COUNT(CASE WHEN in_full = 1 THEN 1 END) / COUNT(*), 2) as percent
FROM fact_aggregate
UNION ALL
SELECT 
  'On-Time' as metric,
  COUNT(CASE WHEN on_time = 1 THEN 1 END) as success_count,
  COUNT(*) as total_count,
  ROUND(100.0 * COUNT(CASE WHEN on_time = 1 THEN 1 END) / COUNT(*), 2) as percent
FROM fact_aggregate
ORDER BY metric;
```

---

## Customer Analysis

### Top 5 Customers by Order Volume
```sql
SELECT 
  fa.customer_id,
  COUNT(DISTINCT fa.order_id) as total_orders,
  COUNT(CASE WHEN fa.otif = 1 THEN 1 END) as otif_orders,
  ROUND(100.0 * COUNT(CASE WHEN fa.otif = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as otif_percent,
  ROUND(100.0 * COUNT(CASE WHEN fa.in_full = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as in_full_percent,
  ROUND(100.0 * COUNT(CASE WHEN fa.on_time = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as on_time_percent
FROM fact_aggregate fa
GROUP BY fa.customer_id
ORDER BY total_orders DESC
LIMIT 5;
```

**Purpose:** Identify high-volume customers with OTIF performance
**Use Case:** Focus service improvements on top customers

---

### Top 5 Worst OTIF Customers (At-Risk)
```sql
SELECT 
  fa.customer_id,
  COUNT(DISTINCT fa.order_id) as total_orders,
  COUNT(CASE WHEN fa.otif = 1 THEN 1 END) as otif_orders,
  ROUND(100.0 * COUNT(CASE WHEN fa.otif = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as otif_percent
FROM fact_aggregate fa
GROUP BY fa.customer_id
HAVING COUNT(DISTINCT fa.order_id) >= 5  -- Only customers with 5+ orders
ORDER BY otif_percent ASC
LIMIT 5;
```

**Purpose:** Identify at-risk customers with consistently poor OTIF
**Use Case:** Prioritize for supply chain intervention

---

### Customer Performance Matrix
```sql
SELECT 
  fa.customer_id,
  COUNT(DISTINCT fa.order_id) as total_orders,
  COUNT(CASE WHEN fa.otif = 1 THEN 1 END) as otif_count,
  COUNT(CASE WHEN fa.in_full = 1 THEN 1 END) as in_full_count,
  COUNT(CASE WHEN fa.on_time = 1 THEN 1 END) as on_time_count,
  CASE 
    WHEN ROUND(100.0 * COUNT(CASE WHEN fa.otif = 1 THEN 1 END) / 
         COUNT(DISTINCT fa.order_id), 2) >= 80 THEN 'Excellent'
    WHEN ROUND(100.0 * COUNT(CASE WHEN fa.otif = 1 THEN 1 END) / 
         COUNT(DISTINCT fa.order_id), 2) >= 60 THEN 'Good'
    WHEN ROUND(100.0 * COUNT(CASE WHEN fa.otif = 1 THEN 1 END) / 
         COUNT(DISTINCT fa.order_id), 2) >= 40 THEN 'Fair'
    ELSE 'Poor'
  END as performance_tier
FROM fact_aggregate fa
GROUP BY fa.customer_id
ORDER BY fa.customer_id;
```

**Purpose:** Categorize customers by performance tier
**Use Case:** Segment for targeted service level agreements (SLAs)

---

## Order Performance

### Order Line Fill Rate
```sql
SELECT 
  COUNT(*) as total_line_items,
  COUNT(CASE WHEN delivery_qty = order_qty THEN 1 END) as full_items,
  COUNT(CASE WHEN delivery_qty < order_qty THEN 1 END) as partial_items,
  ROUND(100.0 * COUNT(CASE WHEN delivery_qty = order_qty THEN 1 END) / 
        COUNT(*), 2) as line_fill_rate
FROM fact_order_line;
```

**Expected Output:**
| total_line_items | full_items | partial_items | line_fill_rate |
|-----------------|-----------|---------------|----------------|
| 1,000+ | 623 | 377 | 62.30 |

---

### Volume vs Line Fill Gap Analysis
```sql
SELECT 
  'Volume Fill' as metric,
  ROUND(100.0 * SUM(CASE WHEN delivery_qty >= order_qty THEN 1 ELSE 0 END) / 
        COUNT(*), 2) as percent
FROM fact_order_line
UNION ALL
SELECT 
  'Line Fill' as metric,
  ROUND(100.0 * COUNT(CASE WHEN delivery_qty = order_qty THEN 1 END) / 
        COUNT(*), 2) as percent
FROM fact_order_line;
```

**Insight:** Large gap (96.54% - 62.30%) = Picking/packing errors, not capacity issues

---

### Late Orders (By Days Late)
```sql
SELECT 
  fa.order_id,
  fa.customer_id,
  fa.order_placement_date,
  DATE(fol.agreed_delivery_date) as promised_date,
  DATE(fol.actual_delivery_date) as actual_date,
  DATE(fol.actual_delivery_date) - DATE(fol.agreed_delivery_date) as days_late,
  fol.in_full,
  fol.on_time,
  fol.on_time_in_full
FROM fact_aggregate fa
JOIN fact_order_line fol ON fa.order_id = fol.order_id
WHERE DATE(fol.actual_delivery_date) > DATE(fol.agreed_delivery_date)
ORDER BY days_late DESC
LIMIT 20;
```

**Purpose:** Identify orders with delivery delays
**Use Case:** Root cause analysis on logistics delays

---

## Regional Comparison

### India vs USA OTIF Comparison
```sql
SELECT 
  'India' as region,
  COUNT(DISTINCT fa.order_id) as total_orders,
  ROUND(100.0 * COUNT(CASE WHEN fa.otif = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as otif_percent,
  ROUND(100.0 * COUNT(CASE WHEN fa.in_full = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as in_full_percent,
  ROUND(100.0 * COUNT(CASE WHEN fa.on_time = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as on_time_percent
FROM fact_aggregate fa
WHERE fa.order_placement_date >= '2025-05-01' AND fa.order_placement_date <= '2025-05-17'
UNION ALL
SELECT 
  'USA' as region,
  COUNT(DISTINCT fa.order_id) as total_orders,
  ROUND(100.0 * COUNT(CASE WHEN fa.otif = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as otif_percent,
  ROUND(100.0 * COUNT(CASE WHEN fa.in_full = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as in_full_percent,
  ROUND(100.0 * COUNT(CASE WHEN fa.on_time = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as on_time_percent
FROM fact_aggregate fa
WHERE fa.order_placement_date >= '2025-05-01' AND fa.order_placement_date <= '2025-05-17'
ORDER BY region;
```

**Insight:** India ~0% OTIF; USA mixed → localized improvement opportunity

---

### Top Customers by Region
```sql
-- India Top 5
SELECT 
  fa.customer_id,
  COUNT(DISTINCT fa.order_id) as total_orders,
  ROUND(100.0 * COUNT(CASE WHEN fa.otif = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as otif_percent
FROM fact_aggregate fa
WHERE fa.order_placement_date >= '2025-05-01'
GROUP BY fa.customer_id
ORDER BY total_orders DESC
LIMIT 5;

-- USA Top 5 (modify date filter as needed)
```

---

## Trend Analysis

### Orders by Date
```sql
SELECT 
  fa.order_placement_date,
  COUNT(DISTINCT fa.order_id) as order_count,
  COUNT(CASE WHEN fa.otif = 1 THEN 1 END) as otif_count,
  ROUND(100.0 * COUNT(CASE WHEN fa.otif = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as daily_otif_percent
FROM fact_aggregate fa
GROUP BY fa.order_placement_date
ORDER BY fa.order_placement_date DESC;
```

**Purpose:** Track OTIF trends over time
**Use Case:** Detect performance regressions or improvements

---

### Rolling 7-Day OTIF Average
```sql
SELECT 
  fa.order_placement_date,
  COUNT(DISTINCT fa.order_id) as orders_7d,
  ROUND(100.0 * COUNT(CASE WHEN fa.otif = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as otif_7d_avg
FROM fact_aggregate fa
WHERE fa.order_placement_date >= (CURRENT_DATE - INTERVAL '7 days')
GROUP BY fa.order_placement_date
ORDER BY fa.order_placement_date DESC;
```

---

## Data Quality

### Check for Missing/NULL Values
```sql
SELECT 
  'fact_aggregate' as table_name,
  COUNT(*) as total_rows,
  COUNT(CASE WHEN order_id IS NULL THEN 1 END) as null_order_id,
  COUNT(CASE WHEN customer_id IS NULL THEN 1 END) as null_customer_id,
  COUNT(CASE WHEN otif IS NULL THEN 1 END) as null_otif
FROM fact_aggregate
UNION ALL
SELECT 
  'fact_order_line' as table_name,
  COUNT(*) as total_rows,
  COUNT(CASE WHEN order_id IS NULL THEN 1 END) as null_order_id,
  COUNT(CASE WHEN customer_id IS NULL THEN 1 END) as null_customer_id,
  COUNT(CASE WHEN delivery_qty IS NULL THEN 1 END) as null_delivery_qty
FROM fact_order_line;
```

---

### Check for Duplicates
```sql
-- Duplicates in fact_aggregate
SELECT 
  order_id, 
  customer_id, 
  COUNT(*) as duplicate_count
FROM fact_aggregate
GROUP BY order_id, customer_id
HAVING COUNT(*) > 1;

-- Duplicates in fact_order_line
SELECT 
  order_id, 
  customer_id, 
  product_id,
  COUNT(*) as duplicate_count
FROM fact_order_line
GROUP BY order_id, customer_id, product_id
HAVING COUNT(*) > 1;
```

---

### Data Freshness Check
```sql
SELECT 
  'fact_aggregate' as table_name,
  MAX(created_at) as last_update,
  CURRENT_TIMESTAMP - MAX(created_at) as age,
  COUNT(*) as total_records
FROM fact_aggregate
UNION ALL
SELECT 
  'fact_order_line' as table_name,
  MAX(created_at) as last_update,
  CURRENT_TIMESTAMP - MAX(created_at) as age,
  COUNT(*) as total_records
FROM fact_order_line;
```

**Purpose:** Verify data is current
**Use Case:** Alert if data is older than 24 hours

---

### Order-Line Consistency Check
```sql
SELECT 
  fol.order_id,
  COUNT(*) as line_count,
  SUM(fol.order_qty) as total_qty_ordered,
  SUM(fol.delivery_qty) as total_qty_delivered,
  SUM(fol.order_qty) - SUM(fol.delivery_qty) as qty_shortfall
FROM fact_order_line fol
GROUP BY fol.order_id
HAVING SUM(fol.delivery_qty) > SUM(fol.order_qty)  -- Delivered more than ordered
ORDER BY qty_shortfall DESC;
```

**Purpose:** Find data anomalies (e.g., delivered > ordered)
**Use Case:** Data quality validation

---

## Common Use Cases

### Generate Daily Report
```sql
-- Run this query daily to send to stakeholders
SELECT 
  CURRENT_DATE as report_date,
  (SELECT COUNT(DISTINCT order_id) FROM fact_aggregate 
   WHERE order_placement_date = CURRENT_DATE) as today_orders,
  (SELECT ROUND(100.0 * COUNT(CASE WHEN otif = 1 THEN 1 END) / 
        COUNT(*), 2) FROM fact_aggregate 
   WHERE order_placement_date = CURRENT_DATE) as today_otif_percent,
  COUNT(DISTINCT fa.order_id) as total_orders_all_time,
  ROUND(100.0 * COUNT(CASE WHEN fa.otif = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as overall_otif_percent
FROM fact_aggregate fa;
```

---

### Export Customer Dashboard
```sql
-- Export this to CSV for dashboard
SELECT 
  fa.customer_id,
  COUNT(DISTINCT fa.order_id) as total_orders,
  ROUND(100.0 * COUNT(CASE WHEN fa.otif = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as otif_percent,
  ROUND(100.0 * COUNT(CASE WHEN fa.in_full = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as in_full_percent,
  ROUND(100.0 * COUNT(CASE WHEN fa.on_time = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as on_time_percent,
  CASE 
    WHEN ROUND(100.0 * COUNT(CASE WHEN fa.otif = 1 THEN 1 END) / 
         COUNT(DISTINCT fa.order_id), 2) >= 80 THEN '✓ Excellent'
    WHEN ROUND(100.0 * COUNT(CASE WHEN fa.otif = 1 THEN 1 END) / 
         COUNT(DISTINCT fa.order_id), 2) >= 60 THEN '⚠ Good'
    WHEN ROUND(100.0 * COUNT(CASE WHEN fa.otif = 1 THEN 1 END) / 
         COUNT(DISTINCT fa.order_id), 2) >= 40 THEN '⚠ Fair'
    ELSE '✗ Poor'
  END as status
FROM fact_aggregate fa
GROUP BY fa.customer_id
ORDER BY otif_percent DESC;
```

---

## Query Performance Tips

```
1. Always use WHERE clause on order_placement_date for large queries
   - Filters to specific date range (e.g., last 30 days)
   
2. Use indexes for GROUP BY and JOIN queries
   - Indexes on: customer_id, order_id, otif
   
3. For real-time dashboards, cache results every 1 hour
   - Supabase can do this automatically
   
4. Avoid SELECT * - specify only needed columns
   - Reduces data transfer, improves query speed
   
5. Use LIMIT when testing large result sets
   - Start with LIMIT 100, then remove when confident
```

---

## Export for Reporting

### As Quadratic Formula
```javascript
// In Quadratic cell:
=QUERY("SELECT * FROM fact_aggregate LIMIT 100")
```

### As CSV (From Quadratic)
```
1. Run query in Quadratic
2. Click Export → Download as CSV
3. Open in Excel or Google Sheets
```

### As PDF Report (From Quadratic)
```
1. Format table with headers
2. Click Export → Download as PDF
3. Email to stakeholders
```

---

## Further Reading

- [PostgreSQL Window Functions](https://www.postgresql.org/docs/current/functions-window.html)
- [PostgreSQL Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html)
- [Query Optimization Guide](https://www.postgresql.org/docs/current/performance-tips.html)
