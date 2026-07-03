# Setup Guide: Supply Chain Intelligence Pipeline

Complete step-by-step guide to deploy the N8N → Supabase → Quadratic pipeline.

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [N8N Setup (Local)](#n8n-setup-local)
3. [Supabase PostgreSQL Setup](#supabase-postgresql-setup)
4. [Gmail OAuth Configuration](#gmail-oauth-configuration)
5. [N8N Workflow Import](#n8n-workflow-import)
6. [Quadratic Configuration](#quadratic-configuration)
7. [Testing & Validation](#testing--validation)
8. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Software
- **Node.js** v18+ (check: `node -v`)
- **npm** v9+ (check: `npm -v`)
- **Git** (for cloning repo)
- **PostgreSQL** client tools (optional, for manual DB queries)

### Required Accounts
- **Supabase** (free tier available)
- **Gmail** account (for email trigger)
- **Google Cloud Console** (for OAuth credentials)
- **Quadratic** (free or pro)
- **Open Exchange Rates** (API key for currency conversion)

---

## N8N Setup (Local)

### Step 1: Install N8N Globally
```bash
npm install -g n8n
```

**Verify installation:**
```bash
n8n --version
```

### Step 2: Start N8N Server
```bash
n8n
```

**Expected output:**
```
Editor is now accessible via:
http://localhost:5678
```

### Step 3: Access N8N Editor
Open browser → `http://localhost:5678`

**First time setup:**
- Create admin user (email + password)
- Accept terms of service
- You're ready to import workflows

### Step 4: Keep N8N Running
- Do NOT close the terminal window
- N8N must stay running for workflows to execute
- For production: Use PM2 or Docker

---

## Supabase PostgreSQL Setup

### Step 1: Create Supabase Project
1. Go to [supabase.com](https://supabase.com)
2. Sign up / Log in
3. Click **New Project**
4. Fill in:
   - **Project name:** `supply-chain-intelligence`
   - **Password:** (save this securely)
   - **Region:** Choose closest to your location
5. Click **Create new project** (wait 2-3 min)

### Step 2: Find Connection Details
1. Go to **Project Settings** → **Database**
2. Copy **Connection String** (standard)
3. Note the **Connection Pooler** details:
   - **Host:** `aws-0-[region].pooling.supabase.com`
   - **Port:** `6543` (NOT 5432)
   - **Database:** `postgres`
   - **User:** `postgres.[project-id]`
   - **Password:** [from Step 1]

### Step 3: Create Tables
In Supabase SQL Editor, run:

```sql
-- Create fact_aggregate table
CREATE TABLE fact_aggregate (
  id SERIAL PRIMARY KEY,
  order_id TEXT NOT NULL,
  customer_id INT NOT NULL,
  order_placement_date DATE,
  on_time INT,
  in_full INT,
  otif INT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(order_id, customer_id)
);

-- Create fact_order_line table
CREATE TABLE fact_order_line (
  id SERIAL PRIMARY KEY,
  order_id TEXT NOT NULL,
  order_placement_date DATE,
  customer_id INT NOT NULL,
  product_id INT,
  order_qty INT,
  agreed_delivery_date DATE,
  actual_delivery_date DATE,
  delivery_qty INT,
  in_full INT,
  on_time INT,
  on_time_in_full INT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY(order_id) REFERENCES fact_aggregate(order_id)
);

-- Create indexes
CREATE INDEX idx_customer_id ON fact_aggregate(customer_id);
CREATE INDEX idx_order_id ON fact_order_line(order_id);
CREATE INDEX idx_customer_otif ON fact_aggregate(customer_id, otif);
```

### Step 4: Verify Tables Created
```sql
SELECT table_name FROM information_schema.tables 
WHERE table_schema = 'public';
```

Should return: `fact_aggregate`, `fact_order_line`

---

## Gmail OAuth Configuration

### Step 1: Create Google Cloud Project
1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Click **Create Project**
3. Name: `N8N Supply Chain` → **Create**
4. Wait for project to load

### Step 2: Enable Gmail API
1. Search for **Gmail API** in search bar
2. Click **Gmail API**
3. Click **Enable**

### Step 3: Create OAuth 2.0 Credentials
1. Go to **APIs & Services** → **Credentials**
2. Click **Create Credentials** → **OAuth 2.0 Client IDs**
3. Application type: **Web application**
4. Name: `N8N Gmail Automation`
5. **Authorized JavaScript origins:**
   ```
   http://localhost:5678
   ```
6. **Authorized redirect URIs:**
   ```
   http://localhost:5678/rest/oauth2-credential/callback
   ```
7. Click **Create** → Copy **Client ID** and **Client Secret**

### Step 4: Test OAuth Connection
- Keep these credentials safe
- You'll use them when configuring Gmail trigger in N8N

---

## N8N Workflow Import

### Step 1: Import Workflow JSON
1. In N8N editor, click **Import from file** (top menu)
2. Select `My_workflow.json` from repo
3. Review nodes and connections
4. Click **Import**

### Step 2: Add Credentials
The workflow will show **missing credentials** warnings. Add them:

**Gmail OAuth2:**
1. Click Gmail Trigger node
2. Click **Create new credential**
3. Paste **Client ID** and **Client Secret** from Step 3 above
4. Click **Sign in with Google**
5. Approve access

**PostgreSQL (Supabase):**
1. Click "Insert rows in a table" node
2. Click **Create new credential**
3. Fill in:
   - **Host:** `aws-0-[region].pooling.supabase.com`
   - **Port:** `6543`
   - **Database:** `postgres`
   - **User:** `postgres.[project-id]`
   - **Password:** [from Supabase Step 1]
4. Click **Test connection** → Should show ✓
5. Save credential
6. Repeat for "Insert rows in a table1" node (use same credential)

### Step 3: Configure Gmail Trigger Filter
1. Click **Gmail Trigger** node
2. Update filter:
   - **Label:** INBOX
   - **Query:** `subject:daily sales`
3. **Download Attachments:** ✓ Enabled

### Step 4: Activate Workflow
1. Click **Activate** button (top-right)
2. Confirm → Workflow is now **ACTIVE**
3. N8N will poll Gmail every minute for matching emails

### Step 5: Test Workflow
Send a test email:
- **To:** your.email@gmail.com
- **Subject:** `daily sales`
- **Attachments:** 
  - `fact_aggregate_india_2025-05-17.csv`
  - `fact_order_line_india_2025-05-17.csv`

Expected result: Both CSV files extracted → 57 + 109 rows inserted to Supabase

---

## Quadratic Configuration

### Step 1: Create Quadratic Sheet
1. Go to [quadratic.app](https://quadratic.app)
2. Click **New Sheet** → Name: `supply-chain-analysis`
3. Save

### Step 2: Connect to Supabase PostgreSQL
1. Click **Connect** (top-right)
2. Select **PostgreSQL**
3. Enter connection details (Session Pooler):
   - **Host:** `aws-0-[region].pooling.supabase.com`
   - **Port:** `6543`
   - **Database:** `postgres`
   - **User:** `postgres.[project-id]`
   - **Password:** [Supabase password]
4. Click **Connect** → Should show ✓

### Step 3: Create Analysis Cells

**Cell A1: Top 5 Global Customers**
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

**Cell A20: Top 5 India Customers**
```sql
SELECT 
  fa.customer_id,
  COUNT(DISTINCT fa.order_id) as total_orders,
  ROUND(100.0 * COUNT(CASE WHEN fa.otif = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as otif_percent,
  ROUND(100.0 * COUNT(CASE WHEN fa.in_full = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as in_full_percent,
  ROUND(100.0 * COUNT(CASE WHEN fa.on_time = 1 THEN 1 END) / 
        COUNT(DISTINCT fa.order_id), 2) as on_time_percent
FROM fact_aggregate fa
WHERE fa.order_placement_date >= '2025-05-01' AND fa.order_placement_date <= '2025-05-17'
GROUP BY fa.customer_id
ORDER BY total_orders DESC
LIMIT 5;
```

### Step 4: Use AI Formulas (Optional)
Quadratic has built-in AI:
1. Type `=` in a cell
2. Ask: "Calculate OTIF percentage from columns"
3. AI generates formula

### Step 5: Export & Share
- Click **Export** → CSV or PDF
- Share with stakeholders via email

---

## Testing & Validation

### Test 1: N8N Workflow
```bash
# Send test email with CSV attachment
# Subject: "daily sales"
# Wait 1-2 minutes for N8N to process
# Check N8N execution logs (N8N Dashboard)
```

**Expected:**
- Gmail Trigger: ✓ Received 1 email
- Extract CSV: ✓ Extracted 57 + 109 rows
- Insert to Supabase: ✓ Inserted successfully

### Test 2: Verify Data in Supabase
```sql
-- Check fact_aggregate
SELECT COUNT(*) FROM fact_aggregate;
-- Should return: 57 (or more if ran multiple times)

-- Check fact_order_line
SELECT COUNT(*) FROM fact_order_line;
-- Should return: 109 (or more)

-- Check sample data
SELECT * FROM fact_aggregate LIMIT 5;
SELECT * FROM fact_order_line LIMIT 5;
```

### Test 3: Query in Quadratic
1. In Quadratic, run the SQL query (Cell A1)
2. Should return: Top 5 customers with OTIF percentages
3. If error: Check PostgreSQL credentials

### Test 4: Check KPI Metrics
Run in Quadratic:
```sql
SELECT 
  COUNT(*) as total_orders,
  ROUND(100.0 * COUNT(CASE WHEN otif = 1 THEN 1 END) / COUNT(*), 2) as otif_percent,
  ROUND(100.0 * COUNT(CASE WHEN in_full = 1 THEN 1 END) / COUNT(*), 2) as in_full_percent,
  ROUND(100.0 * COUNT(CASE WHEN on_time = 1 THEN 1 END) / COUNT(*), 2) as on_time_percent
FROM fact_aggregate;
```

**Expected Results (from May 17, 2025 data):**
- Total Orders: 422
- OTIF %: 23.93%
- In-Full %: 42.42%
- On-Time %: 52.13%

---

## Troubleshooting

### N8N Issues

**Problem: "Port 5678 already in use"**
```bash
# Kill existing process
lsof -i :5678
kill -9 <PID>

# Or use different port
n8n -p 5679
```

**Problem: Gmail trigger not firing**
- Check N8N logs for errors
- Verify OAuth credentials are valid
- Ensure Gmail filter matches email subject

**Problem: CSV extraction fails**
- Check file format (must be CSV)
- Verify column names match expectations
- Check for special characters in filenames

### Supabase Issues

**Problem: "Connection refused" on port 6543**
- Use Session Pooler (port 6543), NOT standard port (5432)
- Check Supabase is online (check dashboard)
- Verify credentials are correct

**Problem: "Duplicate key value" error**
- Check UNIQUE constraint: `order_id + customer_id`
- Run `DELETE FROM fact_aggregate WHERE 1=1;` to clear test data
- Ensure no duplicate order IDs in CSV

**Problem: Foreign key constraint violation**
- Ensure `fact_aggregate` rows exist before `fact_order_line` inserts
- Check N8N node order (Aggregate → Order Lines)

### Quadratic Issues

**Problem: PostgreSQL connection fails**
- Verify connection string format
- Check user has `postgres` role
- Test connection in Supabase first

**Problem: Query returns no results**
- Check table names are lowercase (`fact_aggregate`, not `Fact_Aggregate`)
- Verify data was inserted (use test SQL above)
- Check date filters in WHERE clause

### Gmail Issues

**Problem: OAuth shows "Invalid Client"**
- Regenerate OAuth credentials in Google Cloud
- Ensure redirect URI matches exactly: `http://localhost:5678/rest/oauth2-credential/callback`

**Problem: "Permission denied" for Gmail**
- Revoke access in Google Account → Manage your Google Account → Security
- Re-authorize N8N with Gmail
- Ensure "Gmail API" is enabled in Google Cloud Console

---

## Production Deployment

### For Cloud N8N:
```bash
# Use Docker or N8N Cloud (n8n.io)
docker run -it -p 5678:5678 n8nio/n8n
```

### For Monitoring:
- Set up N8N execution alerts
- Monitor Supabase logs
- Schedule daily Quadratic report exports

### For Scaling:
- Use Supabase Connection Pooler (already configured)
- Add error handling in N8N workflows
- Implement data retention policies (90-day rolling window)

---

## Support & Next Steps

- Check **README.md** for architecture overview
- Review **My_workflow.json** for workflow structure
- Explore Quadratic AI features for advanced analytics
- Set up Open Exchange Rates API for currency conversion

Happy analyzing! 🚀
