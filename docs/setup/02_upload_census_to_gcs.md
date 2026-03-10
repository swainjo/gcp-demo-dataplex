# Upload Census 2021 Data to Google Cloud Storage

This notebook ingests UK Census 2021 TS001 (Usual Resident Population) data into Google Cloud Storage, creates a BigQuery dataset, loads CSV files as materialized tables, and demonstrates BigLake external table capabilities for querying data directly from GCS.

## Prerequisites

### IAM Roles Required
- `roles/storage.admin` - Create buckets and upload files to Cloud Storage
- `roles/bigquery.admin` - Create datasets, load tables, and create external connections
- `roles/iam.serviceAccountAdmin` - Grant permissions to BigLake connection service account

### APIs to Enable
- Cloud Storage API
- BigQuery API
- BigQuery Connection API

### Prior Notebooks
None - this notebook is independent of the US Census ACS setup in notebook 01.

### Local Data Required
This notebook expects CSV files in `../source_data/census2021-ts001/`:
- `census2021-ts001-ctry.csv` (Country level - 3 rows)
- `census2021-ts001-rgn.csv` (Region level - 10 rows)
- `census2021-ts001-utla.csv` (Upper Tier Local Authority - 174 rows)
- `census2021-ts001-ltla.csv` (Lower Tier Local Authority - 331 rows)
- `census2021-ts001-msoa.csv` (Middle Layer Super Output Area - ~7k rows)
- `census2021-ts001-lsoa.csv` (Lower Layer Super Output Area - ~35k rows)
- `census2021-ts001-oa.csv` (Output Area - ~188k rows)
- `metadata.json` (Dataset metadata)

**Total size:** ~9.49 MB

## GCP Services Used

- **Cloud Storage** - Object storage for CSV files
- **BigQuery** - Data warehouse for querying census data
- **BigQuery Connection API** - Creates secure BigLake connections with fine-grained IAM
- **BigLake** - External tables with Dataplex integration and column-level security
- **Cloud Resource Manager** - Service account and IAM management

## Configuration

### Key Variables

| Variable | Example Value | Purpose |
|----------|---------------|---------|
| `PROJECT_ID` | `johnswain-1200-20260303091357` | Your GCP project ID |
| `REGION` | `us-central1` | GCP region for resources |
| `BUCKET_NAME` | `{PROJECT_ID}-census-data` | Cloud Storage bucket name |
| `DATASET_ID` | `census_uk_2021` | BigQuery dataset for UK census |
| `DATASET_LOCATION` | `US` | Multi-region location for BigQuery |
| `TABLE_ID` | `ts001_utla` | Materialized table (UTLA level) |
| `BIGLAKE_TABLE_ID` | `ts001_ctry_biglake` | External BigLake table (country level) |
| `CONNECTION_ID` | `census-biglake-connection` | BigQuery Connection for BigLake |

## Step-by-Step Walkthrough

### 1. Configuration and Authentication (Cells 1-6)

**Purpose:** Set up environment variables and verify access

- Sets `PROJECT_ID`, `REGION`, and `BUCKET_NAME`
- Verifies Application Default Credentials
- Checks for `roles/storage.admin` permission
- Prints bucket name for reference

### 2. Create GCS Bucket (Cells 7-9)

**Purpose:** Create a regional Cloud Storage bucket for census data

- Installs `google-cloud-storage` library
- Creates bucket with:
  - Name: `{PROJECT_ID}-census-data`
  - Location: Matches `REGION` (e.g., `us-central1`)
  - Storage class: Standard
- Handles case where bucket already exists
- Prints bucket details and console link

**Bucket Naming:** Uses project ID as prefix to ensure global uniqueness.

### 3. Upload Census Files (Cells 10-12)

**Purpose:** Upload all UK Census 2021 TS001 files from local directory to GCS

**Upload Process:**
1. Scans `../source_data/census2021-ts001/` directory
2. Iterates through all files (7 CSVs + metadata.json)
3. Uploads each file to `gs://{BUCKET_NAME}/census2021-ts001/`
4. Prints progress with file sizes

**Files Uploaded:**
- 7 geographic-level CSV files (OA, LSOA, MSOA, LTLA, UTLA, RGN, CTRY)
- `metadata.json` with dataset information

**CSV Schema:** All CSVs share the same structure:
- `date` - Census date (2021-03-21)
- `geography` - Area name
- `geography_code` - ONS code
- `total_residents` - Total usual residents
- `household_residents` - Residents in households
- `communal_residents` - Residents in communal establishments

### 4. Validation (Cells 13-15)

**Purpose:** Verify successful upload

- Lists all files in the `census2021-ts001/` prefix
- Checks file count (should be 8)
- Displays file sizes and total size (~9.49 MB)
- Provides GCS console link

### 5. BigLake Setup (Cells 16-28)

**Purpose:** Create a BigLake external table that queries CSV data directly from GCS with enhanced security

**BigLake Benefits:**
- Column-level security and row-level security
- Automatic Dataplex metadata integration
- Fine-grained access control via IAM
- Query acceleration and caching

**Setup Steps:**

1. **Install BigQuery Connection Client** (Cell 16)
   - `google-cloud-bigquery-connection` library

2. **Create BigQuery Connection** (Cells 17-20)
   - Connection ID: `census-biglake-connection`
   - Type: Cloud Resource connection
   - Location: `US` (must match dataset)
   - Generates a service account: `bqcx-{project-number}@gcp-sa-bigquery-condel.iam.gserviceaccount.com`

3. **Grant IAM Permissions** (Cell 21)
   - Grants `roles/storage.objectViewer` to the connection service account
   - Allows BigQuery to read files from the bucket
   - Uses Cloud Resource Manager API

4. **Create External Table** (Cells 22-26)
   - Table: `ts001_ctry_biglake`
   - Source: `gs://{BUCKET_NAME}/census2021-ts001/census2021-ts001-ctry.csv`
   - Uses `ExternalConfig` with:
     - `source_format`: CSV
     - `skip_leading_rows`: 1
     - `autodetect`: True
     - `connection_id`: Full connection resource name
   - Schema auto-detected from CSV

5. **Validate BigLake Table** (Cells 27-28)
   - Runs test queries on external table
   - Verifies data can be read from GCS
   - Example queries: count rows, show sample data

### 6. Create BigQuery Dataset (Cells 29-32)

**Purpose:** Create a dataset for materialized (non-external) tables

- Creates `census_uk_2021` dataset in `US` location
- Sets description: "UK Census 2021 TS001 - Usual Resident Population"
- Handles case where dataset already exists

### 7. Load CSV to BigQuery (Cells 33-35)

**Purpose:** Load UTLA-level data as a materialized table for comparison with BigLake

**Load Configuration:**
- Source: `../source_data/census2021-ts001/census2021-ts001-utla.csv`
- Destination: `{PROJECT_ID}.census_uk_2021.ts001_utla`
- Format: CSV
- Options:
  - `skip_leading_rows=1` (header row)
  - `autodetect=True` (infer schema)
  - `write_disposition="WRITE_TRUNCATE"` (overwrite if exists)

**Load Time:** Typically completes in seconds for 174 rows.

### 8. Validation and Query Examples (Cells 36-44)

**Purpose:** Demonstrate querying both materialized and external tables

**Example Queries:**

1. **Row Count**
   ```sql
   SELECT COUNT(*) as total_rows FROM `{PROJECT_ID}.census_uk_2021.ts001_utla`
   ```

2. **Top Areas by Population**
   ```sql
   SELECT geography, total_residents 
   FROM `{PROJECT_ID}.census_uk_2021.ts001_utla`
   ORDER BY total_residents DESC
   LIMIT 10
   ```

3. **Aggregation Statistics**
   ```sql
   SELECT 
     COUNT(*) as num_areas,
     SUM(total_residents) as total_population,
     AVG(total_residents) as avg_population,
     MAX(total_residents) as max_population
   FROM `{PROJECT_ID}.census_uk_2021.ts001_utla`
   ```

**Console Links:** Notebook generates direct links to:
- BigQuery tables in console
- GCS bucket
- SQL workspace with sample queries

## Data Flow

```
Local CSV Files (../source_data/census2021-ts001/)
          ↓
     [Upload to GCS]
          ↓
gs://{BUCKET_NAME}/census2021-ts001/*.csv
          ↓
          ├─→ [BigLake External Table]
          │   - ts001_ctry_biglake (reads directly from GCS)
          │   - Uses BigQuery Connection with service account
          │   - Column-level security enabled
          │
          └─→ [Load to BigQuery]
              - ts001_utla (materialized table)
              - Data stored in BigQuery storage
              - Standard BigQuery performance
```

## Key Functions

This notebook uses procedural code without custom helper functions, relying on GCP client libraries:

- **`storage.Bucket.blob()`** - Creates blob reference for upload
- **`blob.upload_from_filename()`** - Uploads local file to GCS
- **`bq_connection_client.create_connection()`** - Creates BigLake connection
- **`bigquery.ExternalConfig`** - Configures external table source
- **`bigquery.LoadJobConfig`** - Configures CSV load job

## Validation

### Verify GCS Upload
```python
blobs = list(storage_client.list_blobs(bucket_name, prefix="census2021-ts001/"))
print(f"Uploaded {len(blobs)} files")  # Should be 8
```

### Verify BigLake Connection
```python
connection = bq_connection_client.get_connection(name=connection_name)
print(f"Connection service account: {connection.cloud_resource.service_account_id}")
```

### Query BigLake External Table
```sql
SELECT COUNT(*) FROM `{PROJECT_ID}.census_uk_2021.ts001_ctry_biglake`
-- Should return 3 (England, Wales, Scotland)
```

### Check Materialized Table
```sql
SELECT COUNT(*) FROM `{PROJECT_ID}.census_uk_2021.ts001_utla`
-- Should return 174 (UTLA areas)
```

### Console Verification
1. Navigate to [Cloud Storage Browser](https://console.cloud.google.com/storage/browser)
2. Find your bucket and verify files are present
3. Navigate to [BigQuery](https://console.cloud.google.com/bigquery)
4. Expand `census_uk_2021` dataset and see both tables

## Troubleshooting

### Issue: "Bucket name already taken"
**Cause:** Bucket names are globally unique across all GCP projects.
**Solution:** The notebook uses `{PROJECT_ID}-census-data` which should be unique. If it still conflicts, add a suffix like `-v2`.

### Issue: Local CSV files not found
**Cause:** Files are expected in `../source_data/census2021-ts001/` relative to notebook location.
**Solution:** 
1. Verify the files exist in the correct directory
2. Download UK Census 2021 TS001 data from ONS if missing
3. Adjust the path in the notebook if files are in a different location

### Issue: "Permission denied" when granting IAM to connection service account
**Cause:** Your user lacks `roles/iam.serviceAccountAdmin` or similar.
**Solution:**
```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=user:YOUR_EMAIL \
  --role=roles/iam.serviceAccountAdmin
```

### Issue: BigLake table shows "Access Denied" error
**Cause:** Connection service account doesn't have `storage.objectViewer` on the bucket.
**Solution:** Re-run cell 21 to grant permissions, or manually grant via console:
1. Go to GCS bucket → Permissions
2. Add `bqcx-{project-number}@gcp-sa-bigquery-condel.iam.gserviceaccount.com`
3. Grant role: `Storage Object Viewer`

### Issue: External table query is slow
**Cause:** BigLake tables query data from GCS, which is slower than native BigQuery storage.
**Solution:** 
- Use BigLake for data that needs fine-grained security or changes frequently
- For better query performance, materialize the data (like `ts001_utla`)
- Consider partitioning and clustering for large datasets

### Issue: "Dataset already exists" error
**Cause:** Dataset was created in a previous run.
**Solution:** Either:
- Delete the existing dataset: `bq rm -r -d census_uk_2021`
- Or skip the dataset creation step

## Next Steps

After completing this notebook, you have:
- ✓ UK Census 2021 data in Cloud Storage
- ✓ A BigLake external table (`ts001_ctry_biglake`) with enhanced security
- ✓ A materialized BigQuery table (`ts001_utla`) for fast queries
- ✓ A BigQuery Connection configured for BigLake access

**Continue to:** 
- [`04_setup_cloudsql_census.md`](04_setup_cloudsql_census.md) to load UK Census data into CloudSQL PostgreSQL
- [`03_apply_glossary_terms.md`](03_apply_glossary_terms.md) to create business glossaries (focuses on US Census ACS data from notebook 01)

## Additional Resources

- [BigLake Documentation](https://cloud.google.com/bigquery/docs/biglake-intro)
- [BigQuery External Tables](https://cloud.google.com/bigquery/docs/external-tables)
- [UK Office for National Statistics - Census 2021](https://www.ons.gov.uk/census)
- [BigQuery Connection API](https://cloud.google.com/bigquery/docs/create-cloud-resource-connection)
- [Cloud Storage Best Practices](https://cloud.google.com/storage/docs/best-practices)
