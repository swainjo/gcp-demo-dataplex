# Setup CloudSQL Census Database

This notebook creates a Cloud SQL PostgreSQL instance, loads UK Census 2021 TS001 (Usual Resident Population) data across multiple geographic levels, and enables Dataplex Universal Catalog integration for automatic metadata synchronization.

## Prerequisites

### IAM Roles Required
- `roles/cloudsql.admin` - Create and manage Cloud SQL instances
- `roles/compute.networkUser` - Network access for Cloud SQL Connector
- `roles/iam.serviceAccountUser` - Use service accounts for connections

### APIs to Enable
- Cloud SQL Admin API
- Compute Engine API

### Prior Notebooks
**Recommended (but not required):** [`02_upload_census_to_gcs.md`](02_upload_census_to_gcs.md) - While this notebook can run independently, notebook 02 demonstrates the same UK Census data in BigQuery/GCS for comparison.

### Local Data Required
CSV files in `../source_data/census2021-ts001/`:
- `census2021-ts001-oa.csv` (~188,000 rows - Output Areas)
- `census2021-ts001-lsoa.csv` (~35,000 rows - Lower Layer SOA)
- `census2021-ts001-msoa.csv` (~7,000 rows - Middle Layer SOA)
- `census2021-ts001-ltla.csv` (~331 rows - Lower Tier Local Authority)
- `census2021-ts001-utla.csv` (~174 rows - Upper Tier Local Authority)
- `census2021-ts001-rgn.csv` (~10 rows - Regions)
- `census2021-ts001-ctry.csv` (~3 rows - Countries)

**Total expected rows:** ~232,000 across all geographic levels

## GCP Services Used

- **Cloud SQL** - Managed PostgreSQL 15 relational database
- **Cloud SQL Admin API** - Programmatic instance management
- **Cloud SQL Python Connector** - IAM-based authenticated connections without managing IPs
- **Dataplex Universal Catalog** - Automatic metadata discovery for Cloud SQL databases
- **Compute Engine** (implicit) - Networking for Cloud SQL

## Configuration

### Key Variables

| Variable | Example Value | Purpose |
|----------|---------------|---------|
| `PROJECT_ID` | `johnswain-1200-20260303091357` | Your GCP project ID |
| `REGION` | `us-central1` | GCP region for Cloud SQL instance |
| `INSTANCE_NAME` | `census-demo-db` | Cloud SQL instance identifier |
| `DATABASE_NAME` | `census_data` | PostgreSQL database name |
| `DB_USER` | `postgres` | Database user (default superuser) |
| `DB_PASSWORD` | `Census2021Demo!` | Database password (change for production!) |

**Security Note:** The password is hardcoded for demo purposes. In production:
- Use Secret Manager to store credentials
- Enable Cloud SQL IAM authentication
- Use Cloud SQL Proxy or Private IP
- Rotate passwords regularly

### Instance Specifications

- **Machine Type:** `db-f1-micro` (0.6 GB RAM, 1 shared vCPU)
- **Storage:** 10 GB SSD (`PD_SSD`)
- **PostgreSQL Version:** 15
- **Network:** Public IP with authorized networks (for demo; use Private IP in production)
- **Backups:** Enabled, daily at 03:00 UTC
- **High Availability:** Disabled (single zone for cost savings)

**Cost Estimate:** ~$10-15/month for a running db-f1-micro instance. **Remember to run cleanup notebook 04 to avoid ongoing charges.**

## Step-by-Step Walkthrough

### 1. Configuration and Authentication (Cells 2-5)

**Purpose:** Set up connection parameters and install dependencies

- Defines instance, database, and credential variables
- Installs required packages:
  - `cloud-sql-python-connector[pg8000]` - Cloud SQL Connector with pg8000 driver
  - `sqlalchemy` - ORM and database toolkit
  - `pandas` - Data manipulation for CSV loading
  - `google-cloud-resource-manager` - Project number lookup
  - `certifi` - SSL certificate bundle
- Applies SSL patch for aiohttp (see "Key Functions" section)

**Why Cloud SQL Connector?**
- Automatic IAM-based authentication
- Encrypted connections without managing certificates
- No need to allowlist IP addresses
- Automatic connection pooling

### 2. Create Cloud SQL Instance (Cells 6-9)

**Purpose:** Provision a managed PostgreSQL instance or verify existing one

**Process:**

1. **Check if Instance Exists** (Cell 6)
   - Queries Cloud SQL Admin API: `GET /sql/v1/projects/{PROJECT_ID}/instances/{INSTANCE_NAME}`
   - If found, skips creation

2. **Create Instance** (Cells 7-8)
   - POST request to Cloud SQL Admin API with instance configuration
   - Settings:
     - PostgreSQL 15
     - db-f1-micro tier
     - 10 GB SSD storage
     - Public IP (IPv4)
     - Automated backups enabled
     - Binary logging disabled (not needed for this demo)
   - Returns an operation ID

3. **Wait for Creation** (Cell 9)
   - Polls operation status every 10 seconds
   - Instance creation typically takes 3-8 minutes
   - Final state: `RUNNABLE`

**API Endpoint:**
```
POST https://sqladmin.googleapis.com/v1/projects/{PROJECT_ID}/instances
```

### 3. Connect and Create Schema (Cells 10-15)

**Purpose:** Establish connection and create database table with indexes

**Connection Flow:**

1. **Create SQLAlchemy Engine** (Cell 10)
   - Uses `cloud-sql-python-connector` with `getconn()` creator function
   - Connection string format: `postgresql+pg8000://`
   - Authenticates via Application Default Credentials (no password in connection string for IAM auth)

2. **Create Database** (Cell 11)
   - Connects to default `postgres` database
   - Executes: `CREATE DATABASE census_data`
   - Handles "database already exists" error gracefully

3. **Create Table Schema** (Cells 12-14)
   - Table name: `census_residence_type`
   - Schema:
     | Column | Type | Description |
     |--------|------|-------------|
     | `id` | SERIAL PRIMARY KEY | Auto-incrementing ID |
     | `date` | DATE | Census reference date (2021-03-21) |
     | `geography` | VARCHAR(255) | Area name (e.g., "Birmingham") |
     | `geography_code` | VARCHAR(20) | ONS code (e.g., "E08000025") |
     | `geography_level` | VARCHAR(20) | Level: OA, LSOA, MSOA, LTLA, UTLA, RGN, CTRY |
     | `total_residents` | INTEGER | Total usual residents |
     | `household_residents` | INTEGER | Residents in households |
     | `communal_residents` | INTEGER | Residents in communal establishments |

4. **Create Indexes** (Cell 15)
   - `idx_geography_level` - For filtering by level
   - `idx_total_residents` - For sorting/filtering by population
   - `idx_geography` - For text searches on area names
   - **Purpose:** Improve query performance on common access patterns

### 4. Load Census Data (Cells 16-19)

**Purpose:** Import CSV files from all seven geographic levels into PostgreSQL

**Loading Strategy:**

1. **Define CSV Files** (Cell 16)
   - List of 7 files with relative paths
   - Each represents a different geographic granularity

2. **Column Mapping** (Cell 17)
   - Maps CSV columns to table columns
   - Adds `geography_level` derived from filename (e.g., "UTLA" from `utla.csv`)

3. **Load with Pandas** (Cells 18-19)
   - Reads each CSV with `pd.read_csv()`
   - Adds `geography_level` column
   - Uses `df.to_sql()` with `if_exists='append'` to insert rows
   - `index=False` - Don't insert pandas index as a column

**Load Results:**
- **Expected:** ~232,000 total rows across 7 levels
- **Actual:** Varies based on CSV availability and transaction success
- **Common Issue:** UTLA and CTRY loads may fail with transaction errors (see Troubleshooting)

**Load Time:** Typically 30-60 seconds for all files

### 5. Validate and Query (Cells 20-25)

**Purpose:** Verify data load and demonstrate query capabilities

**Validation Queries:**

1. **Row Count by Level** (Cell 20)
   ```sql
   SELECT geography_level, COUNT(*) as count
   FROM census_residence_type
   GROUP BY geography_level
   ORDER BY count DESC
   ```

2. **Sample Data** (Cell 21)
   - Shows first 10 rows to verify schema and data

3. **Top Areas by Population** (Cell 22)
   ```sql
   SELECT geography, geography_level, total_residents
   FROM census_residence_type
   ORDER BY total_residents DESC
   LIMIT 20
   ```

4. **Statistics by Geographic Level** (Cell 23)
   - Aggregates: `COUNT(*)`, `SUM(total_residents)`, `AVG(total_residents)`, `MAX(total_residents)`
   - Groups by `geography_level`

5. **Communal Residents Percentage** (Cell 24)
   ```sql
   SELECT 
     geography,
     geography_level,
     total_residents,
     communal_residents,
     ROUND((communal_residents::DECIMAL / total_residents * 100), 2) as pct_communal
   FROM census_residence_type
   WHERE total_residents > 0
   ORDER BY pct_communal DESC
   LIMIT 20
   ```

### 6. Enable Dataplex Integration (Cells 26-28)

**Purpose:** Sync Cloud SQL metadata to Dataplex Universal Catalog automatically

**What is Dataplex Integration?**
- Cloud SQL instances can be configured to export metadata to Dataplex
- Dataplex discovers databases, tables, columns, and schemas
- Enables unified catalog across BigQuery, GCS, and Cloud SQL
- Allows applying aspects, tags, and governance policies

**Process:**

1. **Enable Integration** (Cell 26)
   - PATCH request to Cloud SQL Admin API
   - Sets `settings.enableDataplexIntegration: true`
   - Returns an operation ID

2. **Wait for Operation** (Cell 27)
   - Polls operation status every 5 seconds
   - Configuration update typically takes 1-2 minutes

3. **Verify Configuration** (Cell 28)
   - GET request to confirm `enableDataplexIntegration` is `true`

**Metadata Sync Timeline:**
- Configuration: Immediate
- Initial metadata sync: 2-48 hours
- Subsequent syncs: Hourly (incremental updates)

**After Sync:** The Cloud SQL table will appear in Dataplex Catalog search and can be enriched with aspects (as demonstrated in notebook 06).

### 7. Connection Information (Cells 29-30)

**Purpose:** Provide connection strings and commands for external access

**Information Provided:**
- **Connection Name:** `{PROJECT_ID}:{REGION}:{INSTANCE_NAME}`
- **Public IP Address:** Instance's public IPv4
- **Connection String:** `postgresql://{USER}:{PASSWORD}@{IP}:5432/{DATABASE}`
- **psql Command:** `psql "host={IP} dbname={DATABASE} user={USER}"`
- **Cloud SQL Proxy Command:** `./cloud-sql-proxy {CONNECTION_NAME}`

**Security Warning:** These credentials provide full database access. Use IAM authentication and Private IP in production.

### 8. Dataplex Access Guide (Cells 31-32)

**Purpose:** Instructions for finding and enriching the table in Dataplex

**Discovery Steps:**
1. Navigate to [Dataplex Catalog Search](https://console.cloud.google.com/dataplex/catalog/search)
2. Wait 2-48 hours for initial sync (or check periodically)
3. Search for `census_residence_type`
4. Filter by:
   - **System:** Cloud SQL
   - **Asset Type:** Table
   - **Project:** Your project ID

**Enrichment Options:**
- Add business descriptions to tables and columns
- Apply custom aspect types (like `data-governance-public` from notebook 01)
- Link to glossary terms
- Tag for data classification

**Next Step:** See [`06_write_firestore_metadata_to_dataplex.md`](06_write_firestore_metadata_to_dataplex.md) for programmatic metadata enrichment.

## Data Flow

```
Local CSV Files (../source_data/census2021-ts001/)
          ↓
     [pandas read_csv]
          ↓
Pandas DataFrames (7 files, ~232K rows)
          ↓
     [Add geography_level column]
          ↓
     [df.to_sql() with SQLAlchemy]
          ↓
Cloud SQL PostgreSQL Instance (census-demo-db)
  └─ Database: census_data
      └─ Table: census_residence_type
          ├─ Indexes: geography_level, total_residents, geography
          └─ Constraints: PRIMARY KEY (id)
          ↓
     [Enable Dataplex Integration]
          ↓
Metadata Sync to Dataplex Universal Catalog (2-48 hours)
          ↓
Unified Catalog Entry with Schema, Stats, and Governance
```

## Key Functions

### `getconn()` and `getconn_census()` - Connection Creators

SQLAlchemy engine creator functions using Cloud SQL Connector.

**Purpose:**
- Establish authenticated connections via IAM
- Automatically handle SSL/TLS encryption
- Pool connections efficiently

**Implementation:**
```python
def getconn_census():
    conn = connector.connect(
        f"{PROJECT_ID}:{REGION}:{INSTANCE_NAME}",
        "pg8000",
        user=DB_USER,
        password=DB_PASSWORD,
        db=DATABASE_NAME
    )
    return conn

engine = sqlalchemy.create_engine(
    "postgresql+pg8000://",
    creator=getconn_census
)
```

### `patched_tcp_connector_init()` - SSL Context Patch

Patches `aiohttp.TCPConnector` to use certifi's SSL certificate bundle.

**Why Needed:**
- The Cloud SQL Connector uses aiohttp internally
- Default SSL context may not include all CA certificates
- Certifi provides an up-to-date certificate bundle
- Prevents SSL verification failures

**Implementation:**
```python
import ssl
import certifi
import aiohttp

original_init = aiohttp.TCPConnector.__init__

def patched_tcp_connector_init(self, *args, **kwargs):
    if 'ssl' not in kwargs:
        kwargs['ssl'] = ssl.create_default_context(cafile=certifi.where())
    original_init(self, *args, **kwargs)

aiohttp.TCPConnector.__init__ = patched_tcp_connector_init
```

**When to Use:** Apply this patch before creating the Cloud SQL Connector if you encounter SSL errors.

## Validation

### Verify Instance is Running
```bash
gcloud sql instances describe census-demo-db --project={PROJECT_ID}
# Check state: RUNNABLE
```

### Check Row Counts
```python
with engine.connect() as conn:
    result = conn.execute(text("SELECT COUNT(*) FROM census_residence_type"))
    print(f"Total rows: {result.scalar()}")  # Should be ~232,000
```

### Verify Indexes
```sql
SELECT indexname, indexdef 
FROM pg_indexes 
WHERE tablename = 'census_residence_type';
```

### Test Query Performance
```sql
EXPLAIN ANALYZE 
SELECT * FROM census_residence_type 
WHERE geography_level = 'UTLA' 
ORDER BY total_residents DESC 
LIMIT 10;
-- Should use idx_geography_level and idx_total_residents
```

### Check Dataplex Integration Status
```bash
gcloud sql instances describe census-demo-db \
  --format="value(settings.dataplexIntegration)"
# Should output: True
```

### Verify in Dataplex (after 2-48 hours)
Navigate to [Dataplex Catalog](https://console.cloud.google.com/dataplex/catalog) and search for `census_residence_type`.

## Troubleshooting

### Issue: Instance Creation Times Out
**Cause:** Instance creation can take 5-10 minutes in some regions.
**Solution:**
- Wait longer (up to 15 minutes)
- Check instance status in Cloud SQL console
- Verify billing is enabled on the project

### Issue: "Permission denied for database" Error
**Cause:** User doesn't have permissions on the database.
**Solution:**
```sql
GRANT ALL PRIVILEGES ON DATABASE census_data TO postgres;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO postgres;
```

### Issue: CSV Load Fails with "Transaction Error"
**Cause:** Known issue with some CSV files (UTLA, CTRY) causing transaction conflicts.
**Solution:**
- Retry the failing file individually
- Check for malformed data in CSV
- Reduce batch size in `to_sql()` using `chunksize` parameter:
  ```python
  df.to_sql('census_residence_type', engine, if_exists='append', 
            index=False, chunksize=1000)
  ```

### Issue: "SSL: CERTIFICATE_VERIFY_FAILED"
**Cause:** SSL certificate verification fails during Cloud SQL Connector connection.
**Solution:**
- Apply the `patched_tcp_connector_init()` function (Cell 5)
- Ensure `certifi` package is installed
- Update certifi: `pip install --upgrade certifi`

### Issue: Connection Refused or Times Out
**Cause:** Network or firewall blocking connection.
**Solution:**
- Verify Cloud SQL instance has public IP enabled
- Check that your IP is in authorized networks (or use Cloud SQL Proxy)
- Ensure Cloud SQL Admin API is enabled
- For production, use Private IP with VPC peering

### Issue: High Memory Usage During Load
**Cause:** Loading large CSV files (especially OA with 188K rows) into memory.
**Solution:**
- Use `chunksize` parameter in `pd.read_csv()` and `to_sql()`:
  ```python
  for chunk in pd.read_csv(csv_file, chunksize=5000):
      chunk['geography_level'] = level
      chunk.to_sql('census_residence_type', engine, 
                   if_exists='append', index=False)
  ```

### Issue: Dataplex Integration Not Showing Metadata
**Cause:** Initial sync can take 2-48 hours.
**Solution:**
- Wait longer (check after 24 hours)
- Verify integration is enabled: `settings.enableDataplexIntegration: true`
- Check Dataplex service account has necessary permissions
- Try triggering a manual sync (if available in console)

### Issue: Instance Costs Are High
**Cause:** Cloud SQL charges for running instances even when idle.
**Solution:**
- Use db-f1-micro for demos (cheapest tier)
- Stop the instance when not in use (still incurs storage costs)
- **Delete the instance after the demo using cleanup notebook 04**
- Consider Cloud SQL Serverless (when available)

## Next Steps

After completing this notebook, you have:
- ✓ A Cloud SQL PostgreSQL instance with UK Census 2021 data
- ✓ A structured table with ~232K rows across 7 geographic levels
- ✓ Indexes optimized for common query patterns
- ✓ Dataplex integration enabled for metadata discovery

**Continue to:** [`05_setup_firestore.md`](05_setup_firestore.md) to:
- Create a Firestore database
- Extract table metadata from this Cloud SQL instance
- Store structured metadata for later sync to Dataplex

**Alternative Path:** Use the Cloud SQL instance for:
- Analytics queries comparing UK vs US census data
- Building dashboards with Looker or Data Studio
- Demonstrating federated queries with BigQuery (using `EXTERNAL_QUERY`)

**Cost Management:** Remember to run [`cleanup/04_cleanup_cloudsql.ipynb`](../cleanup/04_cleanup_cloudsql.ipynb) to delete the instance and avoid ongoing charges (~$10-15/month).

## Additional Resources

- [Cloud SQL for PostgreSQL Documentation](https://cloud.google.com/sql/docs/postgres)
- [Cloud SQL Python Connector](https://github.com/GoogleCloudPlatform/cloud-sql-python-connector)
- [Dataplex Universal Catalog](https://cloud.google.com/dataplex/docs/universal-catalog)
- [UK Census 2021 TS001 Dataset](https://www.ons.gov.uk/datasets/TS001/editions/2021/versions/1)
- [Cloud SQL Pricing](https://cloud.google.com/sql/pricing)
- [Cloud SQL Best Practices](https://cloud.google.com/sql/docs/postgres/best-practices)
