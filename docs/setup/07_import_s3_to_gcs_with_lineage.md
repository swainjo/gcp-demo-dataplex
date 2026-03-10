# Import Data from AWS S3 to GCS with Dataplex Custom Lineage

This notebook demonstrates cross-cloud data integration by importing American Community Survey (ACS) data from a public AWS S3 bucket into Google Cloud Storage, loading it into BigQuery, and recording comprehensive data lineage using the Dataplex Catalog and Data Lineage API. It showcases custom Dataplex entries for S3 objects and two-hop lineage tracking (S3 → GCS → BigQuery).

## Prerequisites

### IAM Roles Required
- `roles/storage.admin` - Create GCS buckets and upload objects
- `roles/bigquery.admin` - Create datasets and load tables
- `roles/dataplex.catalogAdmin` - Create custom entries for S3 objects
- `roles/datacatalog.admin` - Create lineage processes, runs, and events

### APIs to Enable
- Cloud Storage API
- BigQuery API
- Dataplex API
- Data Catalog Lineage API

### Prior Notebooks
None required - this notebook is independent and demonstrates S3-to-GCP integration.

**Note:** This notebook creates new lineage resources separate from previous notebooks.

### External Requirements
- Access to public AWS S3 bucket (no credentials needed for public data)
- `boto3` Python library for S3 access
- Internet connectivity for S3 downloads

## GCP Services Used

- **Cloud Storage** - Destination for S3 data transfer
- **BigQuery** - Data warehouse for query and analysis
- **Dataplex Catalog** - Custom entries for AWS S3 objects
- **Data Lineage API** - Record lineage processes, runs, and events
- **OpenLineage** - Standard lineage event format (attempted for GCS→BigQuery hop)
- **boto3** - AWS SDK for anonymous S3 access

## Configuration

### Key Variables

| Variable | Example Value | Purpose |
|----------|---------------|---------|
| `PROJECT_ID` | `johnswain-1200-20260303091357` | Your GCP project ID |
| `REGION` | `us-central1` | GCP region for resources |
| `CATALOG_LOCATION` | `us` | Dataplex catalog location |
| `BUCKET_NAME` | `{PROJECT_ID}-census-s3-import` | GCS bucket for imported data |
| `BQ_DATASET_ID` | `census_bureau_acs` | BigQuery dataset (matches notebook 01) |
| `BQ_TABLE_ID` | `s3_acs_data` | BigQuery table for S3 import |

### AWS S3 Configuration

| Variable | Value | Purpose |
|----------|-------|---------|
| `S3_BUCKET` | `dataworld-linked-acs` | Public S3 bucket with ACS data |
| `S3_PREFIX` | `TabulationQueries/` | Folder prefix in S3 |
| `S3_REGION` | `us-east-1` | AWS region for S3 bucket |
| `S3_FILE_KEY` | `TabulationQueries/pums_estimates_14.csv` | Specific file to import |

**Data Source:** [American Community Survey (ACS) on AWS Open Data](https://registry.opendata.aws/census-dataworld-pums/)
- **Provider:** Data.world
- **License:** CC BY 4.0
- **Content:** Public Use Microdata Sample (PUMS) estimates

### Dataplex Resources

| Variable | Value | Purpose |
|----------|-------|---------|
| `ENTRY_GROUP_ID` | `aws-storage-entries` | Entry group for S3 objects |
| `ENTRY_TYPE_ID` | `s3-object` | Entry type for S3 object schema |
| `LINEAGE_PROCESS_ID` | `s3-to-gcs-import-process` | Lineage process for S3→GCS transfer |

## Step-by-Step Walkthrough

### 1. Configuration and Authentication (Cells 0-4)

**Purpose:** Set up environment and verify permissions

- Sets GCP project and region variables
- Configures AWS S3 bucket details
- Verifies Application Default Credentials
- Lists required IAM roles
- Installs boto3 for S3 access

**Anonymous S3 Access:** Public S3 buckets don't require AWS credentials. The notebook configures boto3 with `signature_version=UNSIGNED` for anonymous access.

### 2. Install Libraries (Cells 5-7)

**Purpose:** Install and import required Python packages

**Packages:**
- `boto3` - AWS SDK for S3 operations
- `google-cloud-storage` - GCS upload
- `google-cloud-bigquery` - BigQuery load
- `google-cloud-dataplex` - Custom catalog entries
- `google-cloud-datacatalog-lineage` - Lineage API

**Import Notes:**
- Uses `datacatalog_lineage_v1` for lineage client
- Attempts to import `OpenLineage` types (may fail - see Known Issues)

### 3. Connect to S3 (Cells 8-11)

**Purpose:** Access AWS S3 bucket anonymously and select file for transfer

**S3 Client Configuration:**
```python
s3_client = boto3.client(
    's3',
    region_name=S3_REGION,
    config=Config(signature_version=UNSIGNED)
)
```

**File Selection Process:**
1. **List Objects** (Cell 9)
   - Lists all objects under `TabulationQueries/` prefix
   - Shows available ACS estimate CSV files

2. **Select File** (Cells 10-11)
   - Chooses `pums_estimates_14.csv` (~90 KB)
   - Alternative: `pums_estimates_15.csv`
   - Prints file metadata (size, last modified)

**S3 URI Format:** `s3://dataworld-linked-acs/TabulationQueries/pums_estimates_14.csv`

### 4. Create GCS Bucket (Cells 12-14)

**Purpose:** Create a dedicated GCS bucket for S3 imports

**Bucket Configuration:**
- Name: `{PROJECT_ID}-census-s3-import`
- Location: Matches `REGION`
- Storage class: Standard
- Labels:
  - `source: aws-s3`
  - `demo: dataplex-lineage`
  - `dataset: census-acs`

**Labels Purpose:**
- Track data provenance
- Enable cost analysis by source
- Filter resources in GCP console

**Error Handling:** If bucket exists, continues without error.

### 5. Transfer S3 → GCS (Cells 15-16)

**Purpose:** Stream file from S3 to GCS without local disk storage

**Transfer Method:**
1. **Download from S3** - Streams object to memory via `io.BytesIO()`
2. **Upload to GCS** - Uploads from memory buffer to `data/` prefix

**Code Pattern:**
```python
import io

# Download from S3 to memory
obj = s3_client.get_object(Bucket=S3_BUCKET, Key=S3_FILE_KEY)
file_content = obj['Body'].read()

# Upload to GCS from memory
blob = bucket.blob(f"data/{os.path.basename(S3_FILE_KEY)}")
blob.upload_from_file(io.BytesIO(file_content))
```

**Advantages:**
- No local disk usage
- Single process
- Memory-efficient for small files (<100 MB)

**For Large Files:** Use streaming with chunked transfer or Google's Storage Transfer Service.

**Transfer Metadata:**
- Source: S3 URI
- Destination: `gs://{BUCKET_NAME}/data/pums_estimates_14.csv`
- Duration: Logged for lineage
- File size: Captured in MB

### 6. Load into BigQuery (Cells 17-21)

**Purpose:** Create BigQuery dataset and load CSV from GCS

**Dataset Creation:**
- ID: `census_bureau_acs` (reuses dataset from notebook 01 if it exists)
- Location: `US` (multi-region)
- Description: "US Census Bureau ACS Data"

**Load Job Configuration:**
```python
LoadJobConfig(
    source_format=SourceFormat.CSV,
    skip_leading_rows=1,
    autodetect=True,
    write_disposition=WriteDisposition.WRITE_TRUNCATE
)
```

**Schema Discovery:**
- `autodetect=True` - BigQuery infers schema from CSV
- Automatically detects: column names, data types, modes (NULLABLE/REQUIRED)

**Load Process:**
1. Creates dataset if it doesn't exist
2. Loads CSV from GCS: `gs://{BUCKET_NAME}/data/pums_estimates_14.csv`
3. Destination: `{PROJECT_ID}.census_bureau_acs.s3_acs_data`
4. Overwrites table if it exists (`WRITE_TRUNCATE`)

**Optional Preview:** Cell 21 queries the loaded table to verify data (10 rows).

### 7. Create Dataplex S3 Entry (Cells 22-26)

**Purpose:** Create a custom Dataplex catalog entry representing the S3 object

**Why Catalog S3 Objects?**
- Unified catalog across AWS and GCP
- Track cross-cloud data provenance
- Enable discovery of external data sources
- Document data lineage from origin

**Step 7a: Create Entry Group** (Cells 22-23)

**Configuration:**
- Entry Group ID: `aws-storage-entries`
- Display Name: "AWS Storage Entries"
- Description: "Custom entries for AWS S3 objects"

**Step 7b: Create Entry Type** (Cell 24)

**Configuration:**
- Entry Type ID: `s3-object`
- Display Name: "S3 Object"
- Platform: `AWS S3` (identifies external cloud provider)
- System: `S3` (Amazon Simple Storage Service)

**Entry Type Purpose:**
- Defines schema for S3 object entries
- Enables filtering by platform and system in Dataplex search
- Distinguishes from GCP-native resources

**Step 7c: Create S3 Entry** (Cells 25-26)

**Entry Attributes:**

1. **Fully Qualified Name (FQN):**
   ```
   s3://dataworld-linked-acs/TabulationQueries/pums_estimates_14.csv
   ```
   - Standard S3 URI format
   - Global identifier for the object

2. **Entry ID:**
   - Format: `s3-{sanitized-filename}`
   - Example: `s3-pums-estimates-14`
   - Alphanumeric with hyphens only

3. **Entry Metadata:**
   - Display name: Filename
   - Description: "American Community Survey PUMS estimates from Data.world on AWS"
   - Entry source: AWS S3 platform

**Validation:** Prints entry name and S3 URI after creation.

### 8. Create Lineage Resources (Cells 27-31)

**Purpose:** Record the S3 → GCS transfer as a lineage event

**Lineage Architecture:**

```
Process (s3-to-gcs-import-process)
  └─ Run (timestamped execution)
       └─ LineageEvent (source → target relationship)
            ├─ Source: S3 object (FQN from Dataplex entry)
            └─ Target: GCS object (gs:// URI)
```

**Step 8a: Create Lineage Process** (Cells 27-28)

A process represents a repeatable data transformation or movement.

**Configuration:**
- Process ID: `s3-to-gcs-import-process`
- Display Name: "S3 to GCS Import"
- Description: "Imports ACS data from AWS S3 to Google Cloud Storage"
- Location: `us` (matches catalog location)

**Process Lifecycle:** Once created, the process can have multiple runs (executions) over time.

**Step 8b: Create Lineage Run** (Cell 29)

A run represents a single execution of a process.

**Run Metadata:**
- Start time: Transfer start timestamp
- End time: Transfer completion timestamp
- State: `COMPLETED`
- Attributes (custom key-value pairs):
  - `source_file`: S3 URI
  - `destination_file`: GCS URI
  - `file_size_mb`: Size in MB
  - `transfer_duration_seconds`: Elapsed time

**Attributes Purpose:**
- Capture execution context
- Debug failures
- Track performance metrics
- Audit data movements

**Step 8c: Create Lineage Event** (Cells 30-31)

A lineage event connects source and target data assets.

**Event Links:**
```python
EventLink(
    source=EntryReference(fully_qualified_name="s3://..."),
    target=EntryReference(fully_qualified_name="gs://...")
)
```

**Link Types:**
- **Source:** S3 object FQN (from Dataplex custom entry)
- **Target:** GCS object URI (native GCP resource)

**API Call:**
```python
lineage_client.create_lineage_event(
    parent=run_name,
    lineage_event=LineageEvent(
        start_time=start_time,
        end_time=end_time,
        links=[EventLink(...)]
    )
)
```

**Result:** Lineage graph shows S3 → GCS relationship with timing and metadata.

### 9. GCS → BigQuery Lineage (Cells 32-35)

**Purpose:** Record the GCS → BigQuery load as a lineage event using OpenLineage format

**Lineage Hop 2:**

```
GCS Object (gs://.../data/pums_estimates_14.csv)
          ↓
     [BigQuery Load Job]
          ↓
BigQuery Table (census_bureau_acs.s3_acs_data)
```

**Step 9a: Create Process and Run** (Cells 32-34)

Similar to S3→GCS lineage:
- Process: `gcs-to-bigquery-load-process`
- Run: Timestamped execution with load job metadata
- Attributes: `source_file`, `destination_table`, `row_count`, `load_duration_seconds`

**Step 9b: Create OpenLineage Event** (Cell 35)

**OpenLineage:** Industry-standard format for lineage events (Linux Foundation project).

**Attempted Code:**
```python
from google.cloud.datacatalog_lineage_v1.types import OpenLineage

lineage_client.process_open_lineage_run_event(
    parent=process_name,
    request_body=OpenLineage.RunEvent(...)
)
```

**Known Issue:** The `OpenLineage` type may not be available in `google-cloud-datacatalog-lineage` at the time of this demo. Cell 35 may fail with `ImportError` or `AttributeError`.

**Workaround:** Use standard `LineageEvent` format (same as S3→GCS):
```python
lineage_client.create_lineage_event(
    parent=run_name,
    lineage_event=LineageEvent(
        links=[
            EventLink(
                source=EntryReference(fully_qualified_name="gs://..."),
                target=EntryReference(fully_qualified_name="bigquery://...")
            )
        ]
    )
)
```

### 10. Validation and Links (Cells 36-40)

**Purpose:** Verify resources and provide console access

**Summary Output:**
- ✓ S3 source file and size
- ✓ GCS destination and size
- ✓ BigQuery table and row count
- ✓ Dataplex S3 entry name
- ✓ Lineage processes (S3→GCS, GCS→BigQuery)
- ✓ Lineage runs and events

**Console Links:**
1. **Dataplex Catalog** - Search for S3 entry
2. **Data Lineage** - View lineage graphs
3. **GCS Bucket** - Browse uploaded files
4. **BigQuery Table** - Query and explore data
5. **Lineage Process** - View process details and runs

**Usage Notes:**
- Lineage graphs may take 2-5 minutes to appear in console
- S3 entry appears immediately in Dataplex search
- Filter by entry group `aws-storage-entries` to find S3 objects

## Data Flow

```
AWS S3 (Public Bucket)
  dataworld-linked-acs
    └─ TabulationQueries/pums_estimates_14.csv
          ↓
     [boto3 S3 Client - Anonymous Access]
          ↓
Memory Buffer (io.BytesIO)
          ↓
     [GCS Upload]
          ↓
Google Cloud Storage
  {PROJECT_ID}-census-s3-import
    └─ data/pums_estimates_14.csv
          ↓
     [BigQuery Load Job - CSV Autodetect]
          ↓
BigQuery
  {PROJECT_ID}.census_bureau_acs.s3_acs_data
          ↓
     [Query and Analysis]

Parallel: Metadata and Lineage Flow
          ↓
Dataplex Catalog
  ├─ Custom Entry: S3 object (FQN: s3://...)
  └─ Native Entry: GCS object (auto-discovered)
  └─ Native Entry: BigQuery table (auto-discovered)
          ↓
Data Lineage API
  ├─ Process: s3-to-gcs-import-process
  │    └─ Run with S3→GCS event
  └─ Process: gcs-to-bigquery-load-process
       └─ Run with GCS→BigQuery event
          ↓
Lineage Graph
  S3 Object → GCS Object → BigQuery Table
  (full provenance chain)
```

## Key Functions

This notebook uses procedural code without custom helper functions:

- **`boto3.client('s3', config=Config(signature_version=UNSIGNED))`** - Anonymous S3 access
- **`s3_client.get_object()`** - Download S3 object
- **`blob.upload_from_file(io.BytesIO(...))`** - Upload to GCS from memory
- **`bq_client.load_table_from_uri()`** - Load CSV from GCS to BigQuery
- **`catalog_service.create_entry()`** - Create Dataplex S3 entry
- **`lineage_client.create_process()`** - Define lineage process
- **`lineage_client.create_process_run()`** - Record execution
- **`lineage_client.create_lineage_event()`** - Link data assets

## Validation

### Verify S3 File Access
```python
response = s3_client.head_object(Bucket=S3_BUCKET, Key=S3_FILE_KEY)
print(f"S3 file size: {response['ContentLength']} bytes")
```

### Check GCS Upload
```bash
gsutil ls -lh gs://{PROJECT_ID}-census-s3-import/data/
# Should show pums_estimates_14.csv
```

### Verify BigQuery Load
```sql
SELECT COUNT(*) FROM `{PROJECT_ID}.census_bureau_acs.s3_acs_data`
-- Should return row count matching source CSV
```

### Find S3 Entry in Dataplex
1. Navigate to [Dataplex Catalog Search](https://console.cloud.google.com/dataplex/catalog/search)
2. Search: `pums_estimates_14`
3. Filter by Entry Group: `aws-storage-entries`
4. Verify FQN: `s3://dataworld-linked-acs/TabulationQueries/pums_estimates_14.csv`

### View Lineage Graph
1. Go to [Data Lineage](https://console.cloud.google.com/dataplex/lineage)
2. Search for `s3_acs_data` (BigQuery table)
3. View upstream lineage:
   - Should show GCS object as source
   - May show S3 object if lineage links are indexed (2-5 minutes)

### Check Lineage Process Runs
```python
parent = f"projects/{PROJECT_ID}/locations/{CATALOG_LOCATION}/processes/{LINEAGE_PROCESS_ID}"
runs = lineage_client.list_runs(parent=parent)
for run in runs:
    print(f"Run: {run.name}, State: {run.state}")
```

## Troubleshooting

### Issue: "Access Denied" on S3
**Cause:** S3 bucket is not public or region is incorrect.
**Solution:**
- Verify bucket is public: [AWS Open Data Registry](https://registry.opendata.aws/census-dataworld-pums/)
- Check `S3_REGION` matches bucket location (`us-east-1`)
- Test access manually: `aws s3 ls s3://dataworld-linked-acs/TabulationQueries/ --no-sign-request`

### Issue: boto3 Install Fails
**Cause:** Package not available or network issues.
**Solution:**
```bash
pip install --upgrade boto3
# or
pip install boto3 --no-cache-dir
```

### Issue: GCS Upload is Slow
**Cause:** Large file or network latency.
**Solution:**
- The demo file is small (~90 KB)
- For files >100 MB, use Google's Storage Transfer Service:
  ```bash
  gcloud transfer jobs create s3://source-bucket gs://dest-bucket \
    --source-creds-file=aws-creds.json
  ```

### Issue: BigQuery Load Fails with Schema Error
**Cause:** CSV format unexpected or autodetect issue.
**Solution:**
1. Download the CSV and inspect manually
2. Explicitly define schema instead of autodetect:
   ```python
   schema = [
       bigquery.SchemaField("column1", "STRING"),
       bigquery.SchemaField("column2", "INTEGER"),
       # ... define all columns
   ]
   job_config = LoadJobConfig(schema=schema, skip_leading_rows=1)
   ```

### Issue: S3 Entry Creation Returns 409
**Cause:** Entry already exists from previous run.
**Solution:** Safe to ignore or delete and recreate:
```python
catalog_service.delete_entry(
    name=f"projects/{PROJECT_ID}/locations/{CATALOG_LOCATION}/entryGroups/{ENTRY_GROUP_ID}/entries/{entry_id}"
)
```

### Issue: OpenLineage Import Error (Cell 35)
**Cause:** `OpenLineage` types not available in current library version.
**Solution:**
- **This is a known issue** mentioned in the subagent exploration
- Use standard `LineageEvent` format instead (see Step 9b workaround)
- Update library: `pip install --upgrade google-cloud-datacatalog-lineage`
- Check library docs for OpenLineage support

### Issue: Lineage Graph Doesn't Show S3→GCS Link
**Cause:** Lineage indexing delay (2-5 minutes) or S3 FQN format issue.
**Solution:**
1. Wait 5-10 minutes for indexing
2. Verify S3 entry FQN matches exactly what's used in lineage event
3. Check lineage event was created: `lineage_client.list_lineage_events(parent=run_name)`
4. Verify both source and target entries exist in Dataplex

### Issue: BigQuery Table Shows No Upstream Lineage
**Cause:** GCS→BigQuery lineage event failed (Cell 35 error).
**Solution:**
- Implement the workaround in Step 9b
- Manually create lineage event with correct FQNs:
  - Source: `gs://{BUCKET_NAME}/data/pums_estimates_14.csv`
  - Target: `bigquery:{PROJECT_ID}.census_bureau_acs.s3_acs_data`

### Issue: Memory Error During S3→GCS Transfer
**Cause:** File too large for in-memory buffer.
**Solution:**
```python
# Use streaming with chunked transfer
with s3_client.get_object(Bucket=S3_BUCKET, Key=S3_FILE_KEY)['Body'] as s3_stream:
    blob.upload_from_file(s3_stream, content_type='text/csv')
```

## Next Steps

After completing this notebook, you have:
- ✓ Imported ACS data from AWS S3 to GCP
- ✓ Created a custom Dataplex entry for the S3 object
- ✓ Loaded data into BigQuery for querying
- ✓ Recorded two-hop lineage: S3 → GCS → BigQuery

**Extend the Workflow:**
- Import additional S3 files in a loop
- Set up scheduled transfers with Cloud Functions or Cloud Composer
- Add data quality checks after BigQuery load
- Create data transformation lineage (e.g., BigQuery → Dataflow → BigQuery)
- Link S3 entries to glossary terms

**Cleanup:** Run [`cleanup/07_cleanup_s3_lineage.ipynb`](../cleanup/07_cleanup_s3_lineage.ipynb) to:
- Delete GCS bucket and files
- Delete BigQuery table
- Delete Dataplex S3 entry
- Delete lineage processes and runs

**Compare with Other Notebooks:**
- **Notebook 01:** BigQuery-to-BigQuery lineage (internal GCP)
- **Notebook 02:** GCS-to-BigQuery lineage (BigLake)
- **This Notebook:** S3-to-GCS-to-BigQuery lineage (cross-cloud)

**Production Considerations:**
- Use Storage Transfer Service for large/recurring S3 imports
- Implement AWS credential management for private buckets
- Add error handling and retry logic
- Monitor lineage graph completeness
- Automate lineage creation with event-driven Cloud Functions

## Additional Resources

- [Data Lineage API Documentation](https://cloud.google.com/data-catalog/docs/concepts/about-data-lineage)
- [Dataplex Custom Entries](https://cloud.google.com/dataplex/docs/catalog-custom-entries)
- [OpenLineage Project](https://openlineage.io/)
- [AWS Open Data Registry - ACS](https://registry.opendata.aws/census-dataworld-pums/)
- [Storage Transfer Service](https://cloud.google.com/storage-transfer-service/docs)
- [boto3 Documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html)
- [BigQuery Load Jobs](https://cloud.google.com/bigquery/docs/loading-data-cloud-storage-csv)
- [Cross-Cloud Data Lineage Best Practices](https://cloud.google.com/architecture/data-lineage-patterns)
