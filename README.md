# E-commerce Profiles Project

RudderStack Profiles project for building unified customer profiles with identity resolution (ID stitching).

## Table of Contents

- [Setup](#setup)
- [Quick Start](#quick-start)
- [Project Structure](#project-structure)
- [Architecture](#architecture)
- [Features Computed](#features-computed)
- [Running the Project](#running-the-project)
- [QA Guide: Identity Stitcher](#qa-guide-identity-stitcher)
- [Troubleshooting Workflow](#troubleshooting-workflow)
- [Configuration & Tuning](#configuration--tuning)
- [Production Deployment](#production-deployment)

---

## Setup

Before running this project, you need to configure your own Snowflake connection. See the [Site Configuration docs](https://www.rudderstack.com/docs/profiles/get-started/site-configuration/) for detailed instructions.

### 1. Create a Connection

```bash
# Initialize a new profiles connection
pb init connection
```

Follow the prompts to configure your Snowflake credentials. This creates a connection file in `~/.pb/siteconfig.yaml`.

### 2. Update pb_project.yaml

Edit `pb_project.yaml` and set your connection name:

```yaml
connection: <your_connection_name>
```

### 3. Set Output Schema

In `pb_project.yaml`, update the entity configuration to use your output schema:

```yaml
entities:
  - name: user
    # ... other config ...
    feature_views:
      using_ids:
        - id: user_id
          name: with_customer_id
          # Output schema is derived from your connection's default database/schema
```

The output tables will be created in your connection's target schema. You can verify your output location after running with:

```bash
pb show models -p . --migrate_on_load
```

---

## Quick Start

```bash
# Activate virtual environment
source .venv/bin/activate

# Run full build
pb run -p . --concurrency 10 --migrate_on_load

# Run only ID stitcher (faster iteration when tuning)
pb run -p . -m models/user_id_stitcher --concurrency 10 --migrate_on_load
```

---

## Project Structure

```
profiles_project/
├── pb_project.yaml           # Project config, ID types, entity definitions
├── models/
│   ├── inputs.yaml           # Data source definitions (6 sources)
│   ├── profiles.yaml         # ID stitcher + 26 feature definitions
│   └── sql_models.yaml       # SQL templates for filtered identifies & enriched order items
├── output/                   # Generated SQL artifacts (gitignored)
├── logs/                     # Run logs (gitignored)
└── .venv/                    # Python virtual environment
```

---

## Architecture

For a conceptual overview, see [How Profiles Works](https://www.rudderstack.com/docs/profiles/how-profiles-works/).

### Data Flow

```
                    ┌─────────────────┐
                    │  f_orders       │──────┐
                    │  (transactions) │      │
                    └─────────────────┘      │
                    ┌─────────────────┐      │    ┌──────────────────┐     ┌─────────────────────┐
                    │  f_order_items  │──────┼───►│  ID STITCHER     │────►│  USER_ID_STITCHER   │
                    │  (line items)   │      │    │  (identity       │     │  (identity graph)   │
                    └─────────────────┘      │    │   resolution)    │     └─────────────────────┘
┌─────────────────┐ ┌─────────────────┐      │    └────────┬─────────┘
│  identifies_js  │►│ filtered_       │──────┘             │
│  identifies_ruby│►│ identifies      │                    ▼
└─────────────────┘ │ (super-connector│          ┌─────────────────────┐
                    │  filtering)     │          │  WITH_CUSTOMER_ID   │
                    └─────────────────┘          │  (26 features)      │
                                                 └─────────────────────┘
```

### Data Sources

| Source | Table | Description |
|--------|-------|-------------|
| **f_orders** | `WAREHOUSE.ANALYTICS.F_ORDERS_FILTERED_TP` | Order transactions with revenue, dates, coupons |
| **f_order_items** | `WAREHOUSE.ANALYTICS.F_ORDER_ITEMS_FILTERED_TP` | Line items with product/design details |
| **identifies_js** | `rudderstack.marketplace_javascript_production.identifies` | JS SDK identify events |
| **identifies_ruby** | `rudderstack.marketplace_ruby_production.identifies` | Ruby SDK identify events |
| **dim_designs** | `WAREHOUSE.ANALYTICS.DIM_DESIGNS` | Design theme/genre lookup |
| **dim_products** | `WAREHOUSE.ANALYTICS.DIM_PRODUCTS` | Product attributes lookup |

### ID Types

| ID Type | Description | Sources |
|---------|-------------|---------|
| **user_id** | Customer identifier (SHA1 hashed if email) | Orders (customer_id), Identifies |
| **anonymous_id** | Browser/device identifier from RudderStack | Identifies only |

### Output Tables

These tables are created in your connection's output schema (run `pb show models` to see exact locations):

| Table | Description |
|-------|-------------|
| **USER_ID_STITCHER** | Identity graph - maps all IDs to a canonical `user_main_id` |
| **WITH_CUSTOMER_ID** | Feature view with 26 computed features, keyed by `user_id` |

> **Note**: In the SQL examples below, replace `<your_output_schema>` with your actual output schema (e.g., `DEV.MY_PROFILES`).

---

## Features Computed

This project computes 26 features using [Entity Variables](https://www.rudderstack.com/docs/profiles/feature-development/entity-vars/). All features are output to [Feature Views](https://www.rudderstack.com/docs/profiles/feature-development/feature-views/).

### Order & Revenue
- `total_orders` - Count of completed orders
- `total_order_items` - Sum of items purchased
- `lifetime_revenue_usd` - Total revenue
- `lifetime_sales_subtotal_usd` - Subtotal before fees
- `avg_order_value_usd` - Average order value

### Timing
- `first_order_date` - First purchase date
- `last_order_date` - Most recent purchase
- `days_since_last_order` - Recency metric

### Behavior
- `is_repeat_buyer` - Has more than 1 order
- `total_refund_amount_usd` - Refund total
- `has_used_coupon` - Ever used a coupon code

### Product Preferences
- `total_unique_products`, `total_unique_designers`
- `favorite_theme`, `favorite_theme_genre`
- `favorite_canvas_group`, `favorite_product_style`, `favorite_product_color`
- `unique_themes_purchased`, `unique_theme_genres_purchased`, `unique_canvas_groups_purchased`
- `total_apparel_items`, `total_non_apparel_items`

### Identity Counts
- `count_user_ids` - Number of user_id values stitched into this profile
- `count_anonymous_ids` - Number of anonymous_id values stitched into this profile

---

## Running the Project

See the [Profile Builder Commands](https://www.rudderstack.com/docs/profiles/get-started/commands/) for full CLI reference.

```bash
# Activate environment first
source .venv/bin/activate

# Full run (ID stitcher + all features)
pb run -p . --concurrency 10 --migrate_on_load

# ID stitcher only (faster - use when tuning identity resolution)
pb run -p . -m models/user_id_stitcher --concurrency 10 --migrate_on_load

# Resume from a specific sequence number
pb run -p . --seq_no <N> --concurrency 10 --migrate_on_load

# Show what would be built (dry run / inspect models)
pb show models -p . --migrate_on_load
```

---

## QA Guide: Identity Stitcher

This section explains how to validate the identity stitcher is working correctly, investigate why IDs got merged, and identify potential issues. For background on how identity resolution works, see the [Identity Graph documentation](https://www.rudderstack.com/docs/profiles/identity-resolution/identity-graph/).

### Key Concepts

**Identity Graph Structure**: The `USER_ID_STITCHER` table maps every identifier to a canonical `user_main_id` (see [ID Stitcher Model](https://www.rudderstack.com/docs/profiles/identity-resolution/id-stitcher/)):

| Column | Description |
|--------|-------------|
| `user_main_id` | Canonical profile ID (the "cluster" identifier) |
| `other_id` | An ID that belongs to this profile |
| `other_id_type` | Type of the ID (`user_id` or `anonymous_id`) |

All IDs with the same `user_main_id` are considered to belong to the same person.

### Step 1: Check Overall Health

Run this first to understand the current state:

```sql
-- Cluster size distribution overview
WITH cluster_sizes AS (
  SELECT
    user_main_id,
    COUNT(DISTINCT other_id) as cluster_size,
    COUNT(DISTINCT CASE WHEN other_id_type = 'anonymous_id' THEN other_id END) as anon_ids,
    COUNT(DISTINCT CASE WHEN other_id_type = 'user_id' THEN other_id END) as user_ids
  FROM <your_output_schema>.USER_ID_STITCHER
  GROUP BY user_main_id
)
SELECT
  COUNT(*) as total_profiles,
  MAX(cluster_size) as largest_cluster,
  ROUND(AVG(cluster_size), 2) as avg_cluster_size,
  MEDIAN(cluster_size) as median_cluster_size,
  SUM(CASE WHEN cluster_size > 100 THEN 1 ELSE 0 END) as clusters_over_100,
  SUM(CASE WHEN cluster_size > 50 THEN 1 ELSE 0 END) as clusters_over_50,
  SUM(CASE WHEN cluster_size > 10 THEN 1 ELSE 0 END) as clusters_over_10,
  SUM(CASE WHEN user_ids > 1 THEN 1 ELSE 0 END) as profiles_with_multiple_user_ids
FROM cluster_sizes;
```

**What to look for:**
- **Largest cluster**: Should be reasonable (under 1000 ideally, definitely investigate if >10K)
- **Median cluster size**: Should be close to 1-2 (most people have 1-2 devices)
- **Profiles with multiple user_ids**: Not inherently bad, but large numbers warrant investigation

### Step 2: Identify Problematic Clusters

Find the largest clusters to investigate:

```sql
-- Top 20 largest clusters with breakdown
SELECT
  user_main_id,
  COUNT(DISTINCT CASE WHEN other_id_type = 'anonymous_id' THEN other_id END) as anonymous_ids,
  COUNT(DISTINCT CASE WHEN other_id_type = 'user_id' THEN other_id END) as user_ids,
  COUNT(DISTINCT other_id) as total_ids
FROM <your_output_schema>.USER_ID_STITCHER
GROUP BY user_main_id
ORDER BY total_ids DESC
LIMIT 20;
```

**Red flags:**
- Clusters with >50 anonymous_ids: Likely a shared/public device or bot
- Clusters with >5 user_ids: Multiple real people incorrectly merged
- Huge disparity (e.g., 10K anonymous_ids but 1 user_id): Shared device scenario

### Step 3: Investigate Why IDs Got Stitched

Once you identify a suspicious cluster, trace HOW the IDs got connected:

```sql
-- See all IDs in a specific cluster
SELECT
  user_main_id,
  other_id,
  other_id_type
FROM <your_output_schema>.USER_ID_STITCHER
WHERE user_main_id = '<suspicious_user_main_id>'
ORDER BY other_id_type, other_id;
```

Then check what connected them - IDs get stitched when they appear together in the same source row:

```sql
-- Check if anonymous_ids in the cluster share user_ids (this is how they get connected)
WITH cluster_ids AS (
  SELECT other_id, other_id_type
  FROM <your_output_schema>.USER_ID_STITCHER
  WHERE user_main_id = '<suspicious_user_main_id>'
),
cluster_anon_ids AS (
  SELECT other_id as anonymous_id FROM cluster_ids WHERE other_id_type = 'anonymous_id'
),
cluster_user_ids AS (
  SELECT other_id as user_id FROM cluster_ids WHERE other_id_type = 'user_id'
)
-- Find which source rows connected these IDs
SELECT
  i.anonymous_id,
  i.user_id,
  i.timestamp,
  'identifies' as source
FROM (
  SELECT anonymous_id, user_id, timestamp
  FROM rudderstack.marketplace_javascript_production.identifies
  UNION ALL
  SELECT anonymous_id, user_id, timestamp
  FROM rudderstack.marketplace_ruby_production.identifies
) i
WHERE i.anonymous_id IN (SELECT anonymous_id FROM cluster_anon_ids)
   OR i.user_id IN (SELECT user_id FROM cluster_user_ids)
ORDER BY i.timestamp
LIMIT 500;
```

### Step 4: Find Bridge IDs (Root Cause of Over-Stitching)

Bridge IDs are the specific identifiers causing clusters to merge. A "bridge" is an ID that connects multiple otherwise-separate groups:

```sql
-- Find anonymous_ids that are "bridges" (connect to multiple user_ids)
WITH identifies_combined AS (
  SELECT anonymous_id, user_id
  FROM rudderstack.marketplace_javascript_production.identifies
  WHERE anonymous_id IS NOT NULL AND anonymous_id != ''
    AND user_id IS NOT NULL AND user_id != ''
  UNION ALL
  SELECT anonymous_id, user_id
  FROM rudderstack.marketplace_ruby_production.identifies
  WHERE anonymous_id IS NOT NULL AND anonymous_id != ''
    AND user_id IS NOT NULL AND user_id != ''
)
SELECT
  anonymous_id,
  COUNT(DISTINCT user_id) as distinct_user_ids,
  LISTAGG(DISTINCT user_id, ', ') WITHIN GROUP (ORDER BY user_id) as user_ids_sample
FROM identifies_combined
GROUP BY anonymous_id
HAVING COUNT(DISTINCT user_id) > 1
ORDER BY distinct_user_ids DESC
LIMIT 50;
```

```sql
-- Find user_ids that connect to many anonymous_ids (potential shared accounts)
WITH identifies_combined AS (
  SELECT anonymous_id, user_id
  FROM rudderstack.marketplace_javascript_production.identifies
  WHERE anonymous_id IS NOT NULL AND anonymous_id != ''
    AND user_id IS NOT NULL AND user_id != ''
  UNION ALL
  SELECT anonymous_id, user_id
  FROM rudderstack.marketplace_ruby_production.identifies
  WHERE anonymous_id IS NOT NULL AND anonymous_id != ''
    AND user_id IS NOT NULL AND user_id != ''
)
SELECT
  user_id,
  COUNT(DISTINCT anonymous_id) as distinct_anonymous_ids
FROM identifies_combined
GROUP BY user_id
HAVING COUNT(DISTINCT anonymous_id) > 20
ORDER BY distinct_anonymous_ids DESC
LIMIT 50;
```

### Step 5: Validate Features Look Reasonable

Check if features make sense for profiles (bad stitching often shows in nonsensical feature values):

```sql
-- Find profiles with suspicious feature values
SELECT
  user_id,
  count_user_ids,
  count_anonymous_ids,
  total_orders,
  lifetime_revenue_usd,
  first_order_date,
  last_order_date
FROM <your_output_schema>.WITH_CUSTOMER_ID
WHERE count_user_ids > 5  -- Multiple user_ids merged
   OR count_anonymous_ids > 100  -- Too many devices
   OR lifetime_revenue_usd > 100000  -- Unusually high (adjust threshold)
ORDER BY count_user_ids DESC, count_anonymous_ids DESC
LIMIT 50;
```

### Step 6: Spot-Check Random Profiles

Don't just look at outliers - validate that normal profiles look correct:

```sql
-- Random sample of multi-id profiles
SELECT
  user_id,
  count_user_ids,
  count_anonymous_ids,
  total_orders,
  lifetime_revenue_usd,
  is_repeat_buyer,
  first_order_date,
  last_order_date
FROM <your_output_schema>.WITH_CUSTOMER_ID
WHERE count_anonymous_ids BETWEEN 2 AND 5  -- Reasonable multi-device users
  AND total_orders > 0
ORDER BY RANDOM()
LIMIT 20;
```

---

## Troubleshooting Workflow

### When to Inspect Profile Data vs Source Data

| Symptom | Where to Look | What to Do |
|---------|---------------|------------|
| Large clusters exist | Profile data (USER_ID_STITCHER) | Identify the cluster, then go to source |
| Feature values look wrong | Profile data (WITH_CUSTOMER_ID) | Check count_user_ids/count_anonymous_ids first |
| Need to understand WHY ids merged | **Source data** (identifies tables) | Trace the specific edges that connected IDs |
| Finding bridge IDs | **Source data** | Run bridge ID queries on raw identifies |
| Validating a fix | Profile data (after re-run) | Check cluster stats improved |

### Workflow: Investigating a Bad Cluster

1. **Start in Profile Data**: Find suspicious clusters using cluster size queries
2. **Get the IDs**: List all IDs in that cluster from USER_ID_STITCHER
3. **Move to Source Data**: Query identifies tables to see what rows connected those IDs
4. **Identify the Bridge**: Find which anonymous_id or user_id is the culprit
5. **Apply Fix**: Either add to exclusion list or adjust threshold
6. **Re-run and Validate**: Run ID stitcher only, check if cluster broke apart

### Common Issues and Fixes

| Issue | Symptom | Fix |
|-------|---------|-----|
| **Super-connector anonymous_id** | One anon_id connects 100+ user_ids | Lower threshold in sql_models.yaml or add to exclusion list |
| **Shared account** | One user_id has 500+ anonymous_ids | Add bidirectional filtering for user_ids |
| **Bot traffic** | Repetitive patterns, template-like IDs | Filter by ID pattern (e.g., exclude IDs starting with '#{') |
| **Internal/admin sessions** | Known internal user causing merges | Hardcode exclusion in pb_project.yaml |
| **Transitive over-stitching** | Small bridges create chains | Lower both thresholds, more aggressive filtering |

### Diagnostic Queries for Common Scenarios

**"Why is this customer's profile showing orders from another person?"**
```sql
-- 1. Get the user_main_id for the affected customer
SELECT user_main_id, other_id, other_id_type
FROM <your_output_schema>.USER_ID_STITCHER
WHERE other_id = '<customer_user_id>';

-- 2. See all IDs merged into that profile
SELECT * FROM <your_output_schema>.USER_ID_STITCHER
WHERE user_main_id = '<user_main_id_from_step_1>';

-- 3. Check source data for the bridge
-- (use the query from Step 3 above with this user_main_id)
```

**"Our largest cluster grew - what changed?"**
```sql
-- Compare cluster composition between runs (if you have historical snapshots)
-- Or check recent identify events for the cluster's IDs
WITH cluster_ids AS (
  SELECT other_id, other_id_type
  FROM <your_output_schema>.USER_ID_STITCHER
  WHERE user_main_id = '<largest_cluster_main_id>'
)
SELECT
  DATE_TRUNC('day', timestamp) as day,
  COUNT(*) as events,
  COUNT(DISTINCT anonymous_id) as unique_anon_ids,
  COUNT(DISTINCT user_id) as unique_user_ids
FROM (
  SELECT anonymous_id, user_id, timestamp
  FROM rudderstack.marketplace_javascript_production.identifies
  WHERE anonymous_id IN (SELECT other_id FROM cluster_ids WHERE other_id_type = 'anonymous_id')
)
GROUP BY 1 ORDER BY 1 DESC;
```

**"Is our filtering actually working?"**
```sql
-- Check how many anonymous_ids the filter is excluding
WITH super_connectors AS (
  SELECT anonymous_id, COUNT(DISTINCT user_id) as user_count
  FROM (
    SELECT anonymous_id, user_id FROM rudderstack.marketplace_javascript_production.identifies
    WHERE anonymous_id IS NOT NULL AND user_id IS NOT NULL
    UNION ALL
    SELECT anonymous_id, user_id FROM rudderstack.marketplace_ruby_production.identifies
    WHERE anonymous_id IS NOT NULL AND user_id IS NOT NULL
  )
  GROUP BY anonymous_id
  HAVING COUNT(DISTINCT user_id) > 5  -- Current threshold
)
SELECT
  COUNT(*) as excluded_anonymous_ids,
  SUM(user_count) as total_edges_prevented
FROM super_connectors;
```

---

## Configuration & Tuning

For detailed configuration reference, see the [Profiles documentation](https://www.rudderstack.com/docs/profiles/).

### Key Configuration Files

| File | What to Edit | Docs |
|------|--------------|------|
| `models/sql_models.yaml` | Super-connector threshold (`HAVING COUNT(...) > N`) | [SQL Models](https://www.rudderstack.com/docs/profiles/feature-development/sql-models/) |
| `pb_project.yaml` | Hardcoded ID exclusions, ID type definitions | [pb_project.yaml](https://www.rudderstack.com/docs/profiles/get-started/pb-project/) |
| `models/inputs.yaml` | Source table definitions, ID mappings | [inputs.yaml](https://www.rudderstack.com/docs/profiles/get-started/inputs-yaml/) |
| `models/profiles.yaml` | ID stitcher edge sources, feature definitions | [profiles.yaml](https://www.rudderstack.com/docs/profiles/get-started/profiles-yaml/) |

### Current Filtering Strategy

The project uses **pre-filtering** in `sql_models.yaml` to exclude problematic IDs BEFORE they enter the ID stitcher:

```yaml
# filtered_identifies model logic:
# 1. Combines JS and Ruby identify events
# 2. Identifies anonymous_ids connecting to >5 distinct user_ids
# 3. Excludes those "super-connectors" from stitcher input
```

**Threshold**: Anonymous IDs connecting to >5 distinct user_ids are excluded.

### Tuning the Threshold

| Threshold | Trade-off |
|-----------|-----------|
| **Lower (e.g., >3)** | More aggressive filtering, smaller clusters, but may exclude legitimate users with multiple accounts |
| **Higher (e.g., >10)** | Less filtering, potential for larger clusters, but preserves more legitimate connections |

### Hardcoded Exclusions

For known problematic IDs that don't fit threshold logic, add them to `pb_project.yaml`:

```yaml
id_types:
  - name: anonymous_id
    filters:
      - type: exclude
        regex: "^specific-bad-id-pattern.*"
```

---

## Production Deployment

This section covers how to deploy this profiles project to the RudderStack platform for scheduled, automated runs. See the [Profiles FAQ](https://www.rudderstack.com/docs/profiles/faq/) for common deployment questions.

### Prerequisites

Before deploying to production:

1. **QA the identity stitcher** - Follow the [QA Guide](#qa-guide-identity-stitcher) to validate cluster sizes and stitching quality
2. **Test locally** - Run `pb run` successfully with your connection
3. **Push to GitHub** - The project must be in a Git repository (GitHub, GitLab, or BitBucket supported)

### Step 1: Push Project to GitHub

```bash
# Initialize git if not already done
git init

# Add all files
git add .

# Commit
git commit -m "Initial profiles project"

# Add remote and push
git remote add origin git@github.com:<your-org>/<repo-name>.git
git push -u origin main
```

**Important**: Make sure sensitive files are excluded. The `.gitignore` should include:
- `.venv/`
- `output/`
- `logs/`
- Any local siteconfig files

### Step 2: Configure Warehouse Destination in RudderStack

Before importing the project, you need a warehouse destination configured in your RudderStack workspace:

1. Go to **RudderStack Dashboard** → **Destinations**
2. Add a **Snowflake** destination (or your warehouse type)
3. Configure with credentials that have:
   - **Read access** to source schemas (`WAREHOUSE.ANALYTICS`, `rudderstack`)
   - **Write access** to output schema for profiles tables

### Step 3: Create Profiles Project in RudderStack Dashboard

1. Go to **RudderStack Dashboard** → **Profiles**
2. Click **Create Profile**
3. Select **Import from Git**
4. Enter your repository URL:
   ```
   https://github.com/<your-org>/<repo-name>
   ```
   Or for a specific branch:
   ```
   https://github.com/<your-org>/<repo-name>/tree/<branch-name>
   ```
5. Authenticate with GitHub (RudderStack will provide an SSH key to add to your repo)
6. Select the **warehouse destination** configured in Step 2
7. Click **Validate** to verify the project compiles successfully

### Step 4: Configure Schedule

Once the project is imported:

1. Go to your Profiles project **Settings**
2. Set the **Run Schedule** (e.g., daily, hourly)
3. Configure **Concurrency** (Snowflake only) - recommend `10` for this project
4. Enable the schedule

### Step 5: Deploy and Monitor

1. Click **Run Now** to trigger an initial run
2. Monitor the run in the **Runs** tab
3. Check your warehouse for output tables once complete

### Git Branch Strategy for Dev/Prod

RudderStack supports different Git branches for different environments:

| Environment | Branch | URL Format |
|-------------|--------|------------|
| Development | `dev` | `https://github.com/org/repo/tree/dev` |
| Production | `main` | `https://github.com/org/repo/tree/main` |

You can create separate Profiles projects in RudderStack pointing to different branches, each with their own:
- Output schema (dev vs prod)
- Run schedule
- Warehouse credentials

### Keeping Projects in Sync

The RudderStack dashboard project syncs with your Git repository:

1. **Push changes to GitHub** → Changes are automatically picked up on next run
2. **Manual sync** → Go to project settings and click "Sync" to pull latest changes immediately

### Supported Git URL Formats

```
# Branch
https://github.com/<org>/<repo>/tree/<branch-name>

# Tag
https://github.com/<org>/<repo>/tag/<tag-name>

# Specific commit
https://github.com/<org>/<repo>/commit/<commit-hash>

# Subdirectory (if project is in a subfolder)
https://github.com/<org>/<repo>/tree/<branch>/path/to/project
```

### Production Checklist

Before enabling the production schedule:

- [ ] Identity stitcher QA complete (cluster sizes acceptable)
- [ ] All features validated and returning expected values
- [ ] Super-connector threshold tuned appropriately
- [ ] Warehouse credentials have correct read/write permissions
- [ ] Output schema is the correct production schema
- [ ] Schedule frequency matches data freshness requirements
- [ ] Alerting/monitoring configured for failed runs

---

## Connection Details

Configure your connection in `pb_project.yaml`. See [Setup](#setup) for instructions.

**Source Databases** (these should be accessible from your connection):
- `WAREHOUSE.ANALYTICS` - orders, products, designs tables
- `rudderstack` - identify events from JS/Ruby SDKs

---

## Additional Resources

### RudderStack Profiles Documentation

**Getting Started**
- [Profiles Overview](https://www.rudderstack.com/docs/profiles/overview/) - Introduction to RudderStack Profiles
- [How Profiles Works](https://www.rudderstack.com/docs/profiles/how-profiles-works/) - Identity graphs and feature computation
- [Quickstart Guide](https://www.rudderstack.com/docs/profiles/get-started/quickstart/) - Create your first profiles project

**Configuration**
- [pb_project.yaml](https://www.rudderstack.com/docs/profiles/get-started/pb-project/) - Project configuration reference
- [inputs.yaml](https://www.rudderstack.com/docs/profiles/get-started/inputs-yaml/) - Define data sources
- [profiles.yaml](https://www.rudderstack.com/docs/profiles/get-started/profiles-yaml/) - Define models and features

**Identity Resolution**
- [Identity Graph](https://www.rudderstack.com/docs/profiles/identity-resolution/identity-graph/) - Understanding identity graphs
- [ID Stitcher Model](https://www.rudderstack.com/docs/profiles/identity-resolution/id-stitcher/) - Configure identity stitching
- [ID Types](https://www.rudderstack.com/docs/profiles/identity-resolution/id-types/) - Define and configure ID types

**Features**
- [Entity Variables](https://www.rudderstack.com/docs/profiles/feature-development/entity-vars/) - Define computed features
- [Feature Views](https://www.rudderstack.com/docs/profiles/feature-development/feature-views/) - Output feature tables
- [SQL Models](https://www.rudderstack.com/docs/profiles/feature-development/sql-models/) - Custom SQL templates

**CLI Reference**
- [Profile Builder Commands](https://www.rudderstack.com/docs/profiles/get-started/commands/) - Full CLI command reference
- [Run Profiles Project](https://www.rudderstack.com/docs/profiles/get-started/run-profiles-project/) - Compile and run commands

**Operations**
- [Warehouse Output](https://www.rudderstack.com/docs/profiles/get-started/warehouse-output/) - Output tables and views
- [Site Configuration](https://www.rudderstack.com/docs/profiles/get-started/site-configuration/) - Connection and credentials setup
- [FAQ](https://www.rudderstack.com/docs/profiles/faq/) - Common questions and troubleshooting

### Local Resources
- Run `pb --help` for CLI options
- Run `pb help <command>` for command-specific help
- Check `logs/` directory for detailed run logs
