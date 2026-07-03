# Architecture & Design Documentation

Technical deep-dive into the Supply Chain Intelligence pipeline architecture, design decisions, and data flows.

---

## System Architecture

### High-Level Overview
```
Google Sheets (Master Data)
       ↓
Email + CSV Attachments (Gmail)
       ↓
N8N Workflow (localhost:5678)
├─ Gmail Trigger → Polls every minute for "daily sales" emails
├─ Extract CSV (Aggregate) → 57 rows per run
├─ Extract CSV (Order Lines) → 109 rows per run
├─ Validate & Transform Data → Date format conversion
└─ Parallel Insert to Supabase
       ↓
Supabase PostgreSQL (Session Pooler: port 6543)
├─ fact_aggregate table (57 rows)
└─ fact_order_line table (109 rows)
       ↓
Quadratic AI Spreadsheet
├─ PostgreSQL query connector
├─ Real-time KPI calculations
├─ Customer rankings (Top 5 global + India)
└─ Export reports (CSV, PDF)
```

---

## Component Architecture

### 1. Data Source Layer (Google Sheets)

**Sheets in ecosystem:**
- `Fact_Order_Summary` – Master order details (24+ columns)
- `dim_products` – Product reference data
- `dim_customers` – Customer master data
- `Open_Exchange_Rates_Historical_INR` – Daily FX rates

**Why Google Sheets?**
- Single source of truth for all master data
- Real-time collaboration (multiple users can update)
- Easy backup & version history
- Integrates with Gmail export workflow

**Data Flow:**
```
Google Sheets → Email Export → Gmail Inbox → N8N Trigger
```

---

### 2. ETL Pipeline (N8N)

**N8N Local Server Architecture:**
```
http://localhost:5678
├─ Editor UI (workflow design & testing)
├─ Webhook Server (handles Gmail triggers)
├─ Task Runner (executes nodes)
└─ Database (workflow storage & execution logs)
```

**Workflow Topology:**

**Node 1: Gmail Trigger** (gmailTrigger v1.4)
```json
{
  "type": "gmailTrigger",
  "pollInterval": "1 minute",
  "filters": {
    "labelIds": ["INBOX"],
    "q": "subject:daily sales"
  },
  "downloadAttachments": true
}
```
- **Purpose:** Listen for emails matching filter
- **Output:** Email object with attachments
- **Frequency:** Every 60 seconds
- **Credentials:** Gmail OAuth2

**Nodes 2-3: Extract CSV** (extractFromFile v1.1)
```
Node 2: attachment_1 → fact_aggregate_*.csv → 57 rows
Node 3: attachment_0 → fact_order_line_*.csv → 109 rows
```
- **Purpose:** Parse CSV into structured data
- **Output:** Array of objects (one per row)
- **Type Conversion:** Automatic for numbers, dates
- **Error Handling:** Skip malformed rows, log errors

**Nodes 4-5: Insert to PostgreSQL** (postgres v2.6)

**Node 4 Configuration:**
```json
{
  "type": "postgres",
  "operation": "insertOrUpdate",
  "table": "fact_aggregate",
  "columnMappings": {
    "order_id": "$json.order_id",
    "customer_id": "$json.customer_id",
    "order_placement_date": "$json.order_placement_date.toDateTime('dd-MM-yyyy').format('yyyy-MM-dd')",
    "on_time": "$json.on_time",
    "in_full": "$json.in_full",
    "otif": "$json.otif"
  }
}
```

**Node 5 Configuration:**
```json
{
  "type": "postgres",
  "operation": "insertOrUpdate",
  "table": "fact_order_line",
  "columnMappings": {
    "order_id": "$json.order_id",
    "order_placement_date": "$json.order_placement_date.toDateTime('dd-MM-yyyy').format('yyyy-MM-dd')",
    "customer_id": "$json.customer_id",
    "product_id": "$json.product_id",
    "order_qty": "$json.order_qty",
    "agreed_delivery_date": "$json.agreed_delivery_date.toDateTime('dd-MM-yyyy').format('yyyy-MM-dd')",
    "actual_delivery_date": "$json.actual_delivery_date.toDateTime('dd-MM-yyyy').format('yyyy-MM-dd')",
    "delivery_qty": "$json.delivery_qty",
    "in_full": "$json['In Full']",
    "on_time": "$json['On Time']",
    "on_time_in_full": "$json['On Time In Full']"
  }
}
```

**Design Decisions:**

| Decision | Rationale |
|----------|-----------|
| **Local N8N** | Low latency, full control, no subscription costs |
| **Gmail Trigger** | Email-based workflow, easy scheduling, audit trail |
| **CSV Extraction** | Excel exports natively as CSV, simple parsing |
| **Parallel Extraction** | Both files processed simultaneously (faster) |
| **Parallel Insert** | Two tables loaded independently (no blocking) |
| **Date Transformation** | Standardize input format (dd-MM-yyyy) → DB format (yyyy-MM-dd) |
| **insertOrUpdate** | Idempotent: safe to re-run without duplicates |

**Error Handling:**
```
Input Validation
   ↓
CSV Parsing Error → Log to N8N error queue
   ↓
Type Conversion Error → Skip row, continue
   ↓
Database Insert Error → Rollback, log error message
   ↓
Success → Move to next batch
```

---

### 3. Data Warehouse (Supabase PostgreSQL)

**Database Schema:**

```sql
fact_aggregate
├─ id (SERIAL PRIMARY KEY)
├─ order_id (TEXT, NOT NULL)
├─ customer_id (INT, NOT NULL)
├─ order_placement_date (DATE)
├─ on_time (INT: 0/1)
├─ in_full (INT: 0/1)
├─ otif (INT: 0/1)
├─ created_at (TIMESTAMP)
└─ UNIQUE(order_id, customer_id)

fact_order_line
├─ id (SERIAL PRIMARY KEY)
├─ order_id (TEXT, NOT NULL, FK)
├─ order_placement_date (DATE)
├─ customer_id (INT, NOT NULL)
├─ product_id (INT)
├─ order_qty (INT)
├─ agreed_delivery_date (DATE)
├─ actual_delivery_date (DATE)
├─ delivery_qty (INT)
├─ in_full (INT: 0/1)
├─ on_time (INT: 0/1)
├─ on_time_in_full (INT: 0/1)
└─ created_at (TIMESTAMP)
```

**Indexes:**
```sql
CREATE INDEX idx_customer_id ON fact_aggregate(customer_id);
CREATE INDEX idx_order_id ON fact_order_line(order_id);
CREATE INDEX idx_customer_otif ON fact_aggregate(customer_id, otif);
```

**Connection Pool:**
```
Standard Connection (Port 5432)
├─ Direct PostgreSQL
├─ Max 20 connections
├─ Best for: Interactive queries
└─ Use: Local development

Session Pooler (Port 6543) ← Used by N8N & Quadratic
├─ PgBouncer pool management
├─ Unlimited connections (shared pool)
├─ Best for: High-frequency inserts/queries
└─ Why: Prevents connection exhaustion on 166+ row inserts
```

**Design Decisions:**

| Decision | Rationale |
|----------|-----------|
| **Supabase** | Managed PostgreSQL, easy setup, built-in OAuth, free tier |
| **Session Pooler** | N8N makes 2 parallel INSERT connections; pooler prevents timeouts |
| **UNIQUE Constraint** | Prevent duplicate orders if workflow re-runs |
| **Foreign Key** | Ensure order_line rows reference valid orders |
| **INT for Flags** | 0/1 is simpler than BOOLEAN for CSV parsing |
| **created_at** | Audit trail for when data was loaded |

---

### 4. Analytics Layer (Quadratic)

**Query Architecture:**

**Query 1: Top 5 Global Customers by Volume & OTIF**
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

**Query 2: Top 5 India Customers**
```sql
... (same as above)
WHERE fa.order_placement_date >= '2025-05-01' 
  AND fa.order_placement_date <= '2025-05-17'
```

**Calculation Methods:**

```
OTIF % = COUNT(otif=1) / COUNT(*) × 100
         (Orders that are BOTH on-time AND in-full)

In-Full % = COUNT(in_full=1) / COUNT(*) × 100
            (Orders delivered 100% of requested items)

On-Time % = COUNT(on_time=1) / COUNT(*) × 100
            (Orders delivered by promised date)
```

**Quadratic Features Used:**
1. **PostgreSQL Connector** – Direct data access (no export needed)
2. **AI Formulas** – Auto-generate insights, summaries
3. **Interactive Charts** – Visualize OTIF trends by customer
4. **Export API** – Programmatic CSV/PDF generation

**Design Decisions:**

| Decision | Rationale |
|----------|-----------|
| **Direct PostgreSQL** | Real-time data (no ETL delay) |
| **GROUP BY customer** | Customer-level insights for intervention |
| **LIMIT 5** | Focus on high-impact customers |
| **Percentages** | Easier to interpret than raw counts |
| **CASE WHEN** | Flexible filtering (OTIF, IF, OT independently) |

---

## Data Transformation Flow

### Input CSV Format
```
fact_aggregate:
order_id, customer_id, order_placement_date, on_time, in_full, otif
FMY53010, 789103, 17-05-2025, 1, 0, 0
```

### Transformation Steps (N8N)
```
1. CSV Parse
   ↓
2. Type Detection
   order_id: string
   customer_id: number
   order_placement_date: date (dd-MM-yyyy)
   on_time/in_full/otif: number (0 or 1)
   ↓
3. Date Conversion
   17-05-2025 → 2025-05-17
   ↓
4. Validation
   - customer_id not null
   - order_id unique (UNIQUE constraint)
   - date format valid
   ↓
5. Insert to PostgreSQL
   INSERT INTO fact_aggregate (order_id, customer_id, ...)
   VALUES ('FMY53010', 789103, ...)
```

### Output in PostgreSQL
```
id | order_id | customer_id | order_placement_date | on_time | in_full | otif | created_at
1  | FMY53010 | 789103      | 2025-05-17           | 1       | 0       | 0    | 2025-07-03T16:30:00Z
```

---

## Performance & Scalability

### Current Performance (Observed)
```
N8N Workflow Execution Time: ~45 seconds
├─ Gmail trigger: 1 sec (1 email)
├─ Extract CSV (Aggregate): 5 sec (57 rows)
├─ Extract CSV (Order Lines): 5 sec (109 rows)
├─ Insert Aggregate: 10 sec (57 rows insert)
├─ Insert Order Lines: 15 sec (109 rows insert)
└─ Total: ~45 seconds (includes network latency)

Quadratic Query Execution: ~2 seconds
├─ PostgreSQL query: 1 sec (GROUP BY + aggregations)
├─ Result formatting: 0.5 sec
└─ Chart rendering: 0.5 sec
```

### Scalability Analysis
```
Current Load:
├─ Emails: 1 per day (86,400 seconds apart)
├─ Orders per run: 422 (aggregated)
├─ Order lines per run: 1,000+ 
└─ Total records stored: ~10K+ (rolling 90-day window)

Bottlenecks:
├─ N8N single-threaded for one workflow
├─ Supabase free tier: 500MB disk (sufficient for 5+ years)
├─ Quadratic: Limited to 1000 rows per query result

Scaling Options:
├─ Move to N8N Cloud (managed service)
├─ Use Supabase Pro tier (more connections)
├─ Implement data archival (move old data to S3)
└─ Split queries (paginate 1000+ row results)
```

---

## Security & Credentials

### Credential Management
```
N8N Credentials (Encrypted in local DB)
├─ Gmail OAuth2 (access token, refresh token)
├─ PostgreSQL (username, password)
└─ Never stored in code or logs

Environment Variables (.env)
├─ SUPABASE_URL
├─ SUPABASE_PASSWORD
├─ GMAIL_CLIENT_ID
└─ GMAIL_CLIENT_SECRET

Best Practices:
├─ Use .gitignore to exclude .env
├─ Rotate credentials every 90 days
├─ Use OAuth (no passwords in URLs)
├─ Enable Supabase Row Level Security (RLS) for production
```

### Data Privacy
```
GDPR/CCPA Compliance:
├─ No PII in fact_aggregate (only customer_id)
├─ No sensitive data in fact_order_line
├─ created_at timestamp for audit trails
├─ Data retention: 90-day rolling window (configurable)
└─ Deletion: can be automated with cron job
```

---

## Monitoring & Maintenance

### Monitoring Checklist
```
Daily:
└─ Check N8N execution logs
   ├─ Did workflow execute? (should be 1x/day)
   ├─ Any errors in CSV extraction?
   └─ Row insert counts (57 + 109 expected)

Weekly:
├─ Verify Quadratic reports are fresh
├─ Check Supabase disk usage
└─ Review customer OTIF trends

Monthly:
├─ Analyze failure rate (should be <1%)
├─ Update N8N & Supabase credentials if rotating
└─ Archive old data if disk >80% full
```

### Maintenance Tasks
```
N8N:
├─ Backup workflow JSON (export from UI)
├─ Update N8N version (npm update -g n8n)
└─ Restart service if memory leak detected

PostgreSQL:
├─ VACUUM ANALYZE (maintenance)
├─ Check for slow queries (explain analyze)
└─ Backup database (Supabase auto-backups)

Quadratic:
├─ Update queries if schema changes
└─ Re-authorize OAuth if token expires
```

---

## Design Decisions Summary

| Component | Choice | Alternative | Why |
|-----------|--------|-------------|-----|
| **Pipeline Tool** | N8N | Zapier, Pipedream | Local control, no subscription |
| **Database** | Supabase | AWS RDS, Firebase | Managed PostgreSQL, free tier |
| **Analytics** | Quadratic | Tableau, Power BI | AI spreadsheet, real-time, easy |
| **Email Trigger** | Gmail | Outlook, Slack | Universal, built-in API |
| **Data Format** | CSV | JSON, Parquet | Excel-native, simple |
| **Connection Pool** | Session Pooler | Direct | Handle parallel inserts |
| **KPI Metric** | OTIF % | Only On-Time % | Combined metric more meaningful |

---

## Future Architecture Improvements

### Phase 2: Real-Time Streaming
```
Current: Batch (1x daily email)
Future: Real-time event streaming
  ├─ Kafka topic for order events
  ├─ Stream processor (Flink/Spark)
  └─ Real-time Quadratic dashboard
```

### Phase 3: Predictive Analytics
```
Current: Descriptive (what happened?)
Future: Predictive (what will happen?)
  ├─ ML model: Forecast OTIF by customer
  ├─ Anomaly detection: Flag unusual patterns
  └─ Recommendation engine: Suggest actions
```

### Phase 4: Multi-Region Support
```
Current: India + USA aggregated
Future: Separate pipelines per region
  ├─ Regional N8N instances
  ├─ Federated PostgreSQL (Postgres-XL)
  └─ Unified Quadratic dashboard
```

---

## References

- [N8N Documentation](https://docs.n8n.io)
- [Supabase PostgreSQL Guide](https://supabase.com/docs)
- [Quadratic SQL Queries](https://quadratic.app/docs)
- [PostgreSQL Best Practices](https://www.postgresql.org/docs)
