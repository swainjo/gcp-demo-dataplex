# Setup Firestore Database

This notebook creates a Firestore database in Native Mode, validates it with CRUD operations, connects to the Cloud SQL instance from notebook 04, extracts comprehensive table metadata (schema, indexes, statistics), and stores it in a hierarchical Firestore document structure for later synchronization to Dataplex.

## Prerequisites

### IAM Roles Required
- `roles/datastore.owner` - Create Firestore databases and manage documents
- `roles/serviceusage.serviceUsageAdmin` - Enable Firestore API
- `roles/cloudsql.client` - Connect to Cloud SQL instances (for metadata extraction section)

### APIs to Enable
- Firestore API (App Engine Firestore API)
- Cloud SQL Admin API (for metadata extraction)
- Service Usage API

### Prior Notebooks
**Required for Section 7-10:** [`04_setup_cloudsql_census.md`](04_setup_cloudsql_census.md) - The metadata extraction portion depends on:
- Cloud SQL instance `census-demo-db`
- Database `census_data`
- Table `census_residence_type`

**Note:** Sections 1-6 (Firestore setup and validation) can run independently.

## GCP Services Used

- **Firestore** - NoSQL document database for storing structured metadata
- **Service Usage API** - Enable/verify Firestore API
- **Cloud SQL Python Connector** - Connect to Cloud SQL for metadata extraction
- **PostgreSQL Information Schema** - Query table and column metadata
- **SQLAlchemy** - Database abstraction layer

## Configuration

### Key Variables

| Variable | Example Value | Purpose |
|----------|---------------|---------|
| `PROJECT_ID` | `johnswain-1200-20260303091357` | Your GCP project ID |
| `REGION` | `us-central1` | GCP region for resources |
| `DATABASE_ID` | `(default)` | Firestore database name |

### Cloud SQL Configuration (Sections 7-10)

| Variable | Example Value | Purpose |
|----------|---------------|---------|
| `INSTANCE_NAME` | `census-demo-db` | Cloud SQL instance name |
| `DATABASE_NAME` | `census_data` | PostgreSQL database name |
| `DB_USER` | `postgres` | Database user |
| `DB_PASSWORD` | `Census2021Demo!` | Database password |

### Firestore Database Settings

- **Mode:** Native Mode (Document/Collection model)
- **Location:** `us-central1` (matches region)
- **Type:** Multi-region capable with strong consistency

**Native Mode vs Datastore Mode:**
- **Native Mode:** Recommended for new applications, supports real-time listeners and offline sync
- **Datastore Mode:** Backwards compatible with older Datastore applications
- This demo uses Native Mode for modern Firestore features

## Step-by-Step Walkthrough

### 1. Configuration and Authentication (Cells 2-6)

**Purpose:** Set up environment and verify permissions

- Sets `PROJECT_ID`, `REGION`, and `DATABASE_ID`
- Verifies Application Default Credentials
- Lists required IAM roles
- Installs `google-cloud-firestore` library

### 2. Enable APIs (Cells 7-9)

**Purpose:** Ensure Firestore API is enabled programmatically

**Process:**
1. **Check if API is Enabled** (Cell 7)
   - Queries Service Usage API: `GET /v1/projects/{PROJECT_ID}/services/firestore.googleapis.com`
   - Checks state: `ENABLED` or `DISABLED`

2. **Enable API if Needed** (Cells 8-9)
   - POST to Service Usage API to enable
   - Waits for operation to complete (~30 seconds)

**Manual Alternative:**
```bash
gcloud services enable firestore.googleapis.com --project={PROJECT_ID}
```

### 3. Create Firestore Database (Cells 10-13)

**Purpose:** Create a Firestore database in Native Mode

**Creation Process:**

1. **Check for Existing Database** (Cell 10)
   - Uses `gcloud firestore databases list` command
   - Looks for `(default)` database

2. **Create Database** (Cells 11-12)
   - Command: `gcloud firestore databases create --location={REGION} --type=firestore-native`
   - Typical creation time: 30-60 seconds
   - Creates the default database in Native Mode

3. **Verify Creation** (Cell 13)
   - Confirms database exists and is in `ACTIVE` state

**Important:** Each project can have one `(default)` Firestore database. Additional databases require Firebase project setup.

### 4. Verify Firestore (Cells 14-20)

**Purpose:** Test basic Firestore operations (CRUD) to confirm setup

**Test Operations:**

1. **Initialize Client** (Cell 14)
   ```python
   db = firestore.Client(project=PROJECT_ID, database=DATABASE_ID)
   ```

2. **Create Document** (Cells 15-16)
   - Collection: `test_setup`
   - Document ID: `verification_doc`
   - Data: `{"status": "active", "created_at": datetime.now(), "test_type": "setup_validation"}`
   - Uses `.set()` to write document

3. **Read Document** (Cell 17)
   - Retrieves document by ID
   - Verifies data integrity

4. **Query Collection** (Cell 18)
   - Filters: `status == 'active'`
   - Demonstrates query capabilities

5. **Delete Document** (Cell 19-20)
   - Cleans up test data
   - Verifies deletion

**Result:** Confirms Firestore is fully operational

### 5. Summary and Next Steps (Cells 21-22)

**Purpose:** Provide links and usage guidance

- Console link to Firestore database
- Instructions for viewing data
- Cleanup reminder

### 6. Example Operations (Cells 23-28)

**Purpose:** Demonstrate common Firestore patterns for metadata storage

**Examples:**

1. **Batch Writes** (Cell 23-24)
   - Create multiple user documents
   - Use `.set()` for deterministic IDs

2. **Filtered Queries** (Cell 25)
   - Query by field: `role == 'analyst'`
   - Demonstrates indexes (automatically created for simple queries)

3. **Update Operations** (Cell 26)
   - Partial document updates with `.update()`
   - Increment age for a specific user

4. **Batch Operations** (Cell 27)
   - Create 5 documents in a single batch
   - Atomic operation (all or nothing)

5. **Cleanup** (Cell 28)
   - Delete all example documents
   - Iterates through query results

### 7. Cloud SQL Configuration and Connection (Cells 29-33)

**Purpose:** Connect to Cloud SQL instance to extract metadata

**Setup:**
1. **Define Cloud SQL Variables** (Cell 29)
2. **Install Packages** (Cell 30)
   - `cloud-sql-python-connector[pg8000]`
   - `sqlalchemy`
   - `certifi`

3. **Apply SSL Patch** (Cell 31)
   - Same `patched_tcp_connector_init()` as notebook 04
   - Ensures SSL certificate validation works

4. **Create Connection** (Cells 32-33)
   - SQLAlchemy engine with Cloud SQL Connector
   - Connects to `census_data` database
   - Tests connection with simple query

### 8. Table Copy and Metadata Extraction (Cells 34-40)

**Purpose:** Create a copy of the census table and extract comprehensive metadata

**Process:**

1. **Create Table Copy** (Cell 34)
   - Table: `census_residence_type_copy`
   - SQL: `CREATE TABLE ... AS SELECT * FROM census_residence_type`
   - Purpose: Demonstrate metadata extraction without modifying original

2. **Query Information Schema** (Cells 35-37)
   - **Columns:** Queries `information_schema.columns` for:
     - Column name
     - Data type
     - Nullability
     - Default values
     - Character max length
   - **Primary Keys:** Queries `information_schema.table_constraints` and `key_column_usage`
   - **Indexes:** Queries `pg_indexes` for index definitions
   - **Row Count:** `SELECT COUNT(*) FROM census_residence_type_copy`

3. **Build Field Metadata** (Cells 38-40)
   - Creates `field_descriptions` dict mapping column names to business descriptions
   - Examples:
     - `date`: "Census reference date (2021-03-21)"
     - `geography`: "Geographic area name (e.g., local authority, region)"
     - `total_residents`: "Total number of usual residents in the area"
   - Combines schema info with business context

### 9. Store Metadata in Firestore (Cells 41-45)

**Purpose:** Persist structured metadata in Firestore for later Dataplex sync

**Firestore Schema:**

```
datasets (collection)
  └─ census_data (document) ← Dataset-level metadata
       ├─ database_name: "census_data"
       ├─ source_type: "cloudsql"
       ├─ instance_name: "census-demo-db"
       ├─ connection_name: "{PROJECT_ID}:{REGION}:census-demo-db"
       ├─ region: "us-central1"
       ├─ description: "UK Census 2021 TS001 - Usual Resident Population by Geography"
       ├─ labels: {"source": "ons", "census_year": "2021", "dataset": "ts001"}
       └─ tables (subcollection) ← Table-level metadata
            └─ census_residence_type_copy (document)
                 ├─ tableName: "census_residence_type_copy"
                 ├─ description: "Census 2021 TS001 data..."
                 ├─ row_count: 232000
                 ├─ geographic_levels: ["OA", "LSOA", "MSOA", "LTLA", "UTLA", "RGN", "CTRY"]
                 ├─ indexes: [list of index definitions]
                 ├─ fields: [array of field metadata objects]
                 ├─ partitioning: None
                 ├─ clustering_fields: []
                 └─ last_sync: timestamp
```

**Field Metadata Structure:**
```python
{
    "name": "total_residents",
    "type": "integer",
    "mode": "NULLABLE",
    "description": "Total number of usual residents in the area",
    "is_primary_key": False
}
```

**Document Creation:**

1. **Dataset Document** (Cells 41-42)
   - Path: `datasets/census_data`
   - Contains connection info, labels, and description

2. **Table Document** (Cells 43-45)
   - Path: `datasets/census_data/tables/census_residence_type_copy`
   - Contains schema, row count, indexes, and field details
   - Nested under dataset as a subcollection

**Why Firestore?**
- Flexible schema for evolving metadata
- Hierarchical structure mirrors database/table relationships
- Easy to query and update programmatically
- Can be used as a metadata cache for Dataplex

### 10. Query and Validate (Cells 46-53)

**Purpose:** Verify metadata was stored correctly and demonstrate query patterns

**Validation Queries:**

1. **Read Dataset Document** (Cell 46)
   - Retrieves `datasets/census_data`
   - Prints connection info and labels

2. **Read Table Document** (Cell 47)
   - Retrieves `datasets/census_data/tables/census_residence_type_copy`
   - Displays row count and description

3. **List All Fields** (Cell 48)
   - Iterates through `fields` array
   - Shows field names, types, and descriptions

4. **Collection Group Query** (Cell 49)
   - Queries ALL `tables` subcollections across all datasets
   - Demonstrates cross-dataset queries
   - Useful for: "Show me all tables with more than X rows"

5. **Query by Source Type** (Cell 50)
   - Filters datasets: `source_type == 'cloudsql'`
   - Demonstrates dataset-level filtering

6. **Close Connections** (Cells 51-53)
   - Closes SQLAlchemy engine and Cloud SQL connector
   - Cleans up resources

## Data Flow

```
Cloud SQL Instance (census-demo-db)
          ↓
   [Connect via Cloud SQL Connector]
          ↓
PostgreSQL Database (census_data)
  └─ Table: census_residence_type
          ↓
   [Query Information Schema]
          ↓
Metadata Extraction:
  - Columns (names, types, constraints)
  - Primary keys
  - Indexes
  - Row counts
  - Statistics
          ↓
   [Combine with Business Descriptions]
          ↓
Structured Metadata Documents
          ↓
   [Write to Firestore]
          ↓
Firestore (Native Mode)
  datasets (collection)
    └─ census_data (document)
         ├─ Dataset metadata
         └─ tables (subcollection)
              └─ census_residence_type_copy (document)
                   └─ Complete table metadata
```

## Key Functions

### `patched_tcp_connector_init()`

Patches `aiohttp.TCPConnector` to use certifi SSL certificates (same as notebook 04).

**Purpose:** Prevent SSL verification errors when using Cloud SQL Connector

### `field_descriptions` Dictionary

Maps column names to human-readable business descriptions.

**Example:**
```python
field_descriptions = {
    "date": "Census reference date (2021-03-21)",
    "geography": "Geographic area name (e.g., local authority, region)",
    "geography_code": "ONS area code (9-character alphanumeric identifier)",
    "geography_level": "Geographic hierarchy level (OA, LSOA, MSOA, LTLA, UTLA, RGN, CTRY)",
    "total_residents": "Total number of usual residents in the area",
    "household_residents": "Number of residents living in households",
    "communal_residents": "Number of residents in communal establishments (e.g., care homes, student halls)"
}
```

**Usage:** Enriches technical schema with business context for Dataplex

## Validation

### Verify Firestore Database Creation
```bash
gcloud firestore databases list --project={PROJECT_ID}
# Should show (default) database in ACTIVE state
```

### Check Firestore Data in Console
1. Navigate to [Firestore Console](https://console.cloud.google.com/firestore)
2. Select `(default)` database
3. Browse `datasets` collection
4. Open `census_data` document
5. Expand `tables` subcollection

### Query Dataset Document
```python
db = firestore.Client(project=PROJECT_ID)
dataset_ref = db.collection('datasets').document('census_data')
dataset = dataset_ref.get()
print(dataset.to_dict())
```

### Query Table Document
```python
table_ref = dataset_ref.collection('tables').document('census_residence_type_copy')
table = table_ref.get()
print(f"Row count: {table.get('row_count')}")
print(f"Fields: {len(table.get('fields'))}")
```

### Verify Metadata Completeness
Check that the table document contains:
- ✓ `tableName`
- ✓ `description`
- ✓ `row_count` (should be ~232,000)
- ✓ `fields` array (should have 8 fields)
- ✓ `indexes` array (should list indexes on the table)
- ✓ `geographic_levels` (should be 7 levels)

### Test Collection Group Query
```python
tables = db.collection_group('tables').stream()
for table in tables:
    print(f"Table: {table.id}, Rows: {table.get('row_count')}")
```

## Troubleshooting

### Issue: "App Engine application not found"
**Cause:** Firestore requires an App Engine application (implicit in newer projects).
**Solution:**
```bash
gcloud app create --region={REGION}
# Then retry Firestore database creation
```

### Issue: "Database already exists" Error
**Cause:** The `(default)` Firestore database was already created.
**Solution:** This is fine; skip to the verification step (Section 4). You can only have one `(default)` database per project.

### Issue: Cannot Create Native Mode Database
**Cause:** Project already has a Datastore Mode database.
**Solution:** Firestore mode cannot be changed once set. Create a new project or use the existing Datastore Mode database (requires code changes).

### Issue: Firestore Client Connection Error
**Cause:** API not enabled or credentials not configured.
**Solution:**
1. Ensure Firestore API is enabled (Section 2)
2. Verify Application Default Credentials: `gcloud auth application-default login`
3. Check IAM role: `roles/datastore.owner`

### Issue: Cloud SQL Connection Fails in Section 7
**Cause:** Cloud SQL instance not created, stopped, or credentials incorrect.
**Solution:**
1. Verify instance exists: `gcloud sql instances list`
2. Check instance is `RUNNABLE` state
3. Verify credentials match notebook 04
4. Re-run notebook 04 if instance was deleted

### Issue: "Table does not exist" When Extracting Metadata
**Cause:** `census_residence_type` table wasn't created in notebook 04.
**Solution:**
1. Run notebook 04 completely before section 7 of this notebook
2. Verify table exists: `SELECT * FROM census_residence_type LIMIT 1;`

### Issue: Empty or Incomplete Metadata in Firestore
**Cause:** Metadata extraction queries failed or returned empty results.
**Solution:**
1. Check Cloud SQL connection is active
2. Verify table has data: `SELECT COUNT(*) FROM census_residence_type_copy`
3. Check for SQL errors in notebook output
4. Re-run cells 34-45 (metadata extraction and storage)

### Issue: SSL Certificate Verification Fails
**Cause:** Same as notebook 04 - missing or outdated SSL certificates.
**Solution:**
- Apply `patched_tcp_connector_init()` (Cell 31)
- Update certifi: `pip install --upgrade certifi`

### Issue: Firestore Write Quota Exceeded
**Cause:** Too many writes in a short period (unlikely in this notebook).
**Solution:**
- Wait a few seconds and retry
- Batch writes where possible (already implemented)
- Check [Firestore Quotas](https://cloud.google.com/firestore/quotas)

## Next Steps

After completing this notebook, you have:
- ✓ A Firestore database in Native Mode
- ✓ Validated CRUD operations
- ✓ Extracted comprehensive Cloud SQL table metadata
- ✓ Stored metadata in a hierarchical Firestore structure

**Continue to:** [`06_write_firestore_metadata_to_dataplex.md`](06_write_firestore_metadata_to_dataplex.md) to:
- Read this Firestore metadata
- Create Dataplex catalog entries for Cloud SQL tables
- Apply governance aspects
- Complete the metadata synchronization loop

**Alternative Uses:**
- Store metadata for other data sources (BigQuery, GCS)
- Build a metadata management API on top of Firestore
- Create dashboards showing data inventory from Firestore
- Implement metadata change tracking with Firestore timestamps

**Cost Note:** Firestore has a generous free tier:
- 1 GB storage free per project
- 50K reads, 20K writes, 20K deletes per day free
- This demo uses ~1 MB and minimal operations
- No cleanup needed unless you want to remove test data

## Additional Resources

- [Firestore Documentation](https://cloud.google.com/firestore/docs)
- [Native Mode vs Datastore Mode](https://cloud.google.com/firestore/docs/firestore-or-datastore)
- [Firestore Data Model](https://cloud.google.com/firestore/docs/data-model)
- [Firestore Python Client Library](https://googleapis.dev/python/firestore/latest/index.html)
- [PostgreSQL Information Schema](https://www.postgresql.org/docs/current/information-schema.html)
- [Firestore Security Rules](https://cloud.google.com/firestore/docs/security/get-started)
- [Firestore Best Practices](https://cloud.google.com/firestore/docs/best-practices)
