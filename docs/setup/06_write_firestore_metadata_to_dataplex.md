# Write Firestore Metadata to Dataplex Universal Catalog

This notebook reads Cloud SQL table metadata from Firestore (stored in notebook 05), creates custom Dataplex catalog entries for the Cloud SQL table, and applies governance aspects to enrich the metadata with provenance and licensing information.

## Prerequisites

### IAM Roles Required
- `roles/firestore.viewer` - Read metadata from Firestore
- `roles/dataplex.catalogAdmin` - Create entry groups, entry types, entries, and aspects

### APIs to Enable
- Firestore API
- Dataplex API

### Prior Notebooks
**Required:**
- [`04_setup_cloudsql_census.md`](04_setup_cloudsql_census.md) - Cloud SQL instance and census table
- [`05_setup_firestore.md`](05_setup_firestore.md) - Firestore metadata extraction and storage

**Recommended:**
- [`01_config_and_data_setup.md`](01_config_and_data_setup.md) - Creates the `data-governance-public` aspect type used in this notebook

## GCP Services Used

- **Firestore** - Read structured metadata documents
- **Dataplex Catalog** - Create custom entries and apply aspects
- **Dataplex Entry Groups** - Organize custom entries by system type
- **Dataplex Entry Types** - Define schema for custom entry types
- **Dataplex Aspects** - Attach governance metadata to entries

## Configuration

### Key Variables

| Variable | Example Value | Purpose |
|----------|---------------|---------|
| `PROJECT_ID` | `johnswain-1200-20260303091357` | Your GCP project ID |
| `REGION` | `us-central1` | GCP region |
| `CATALOG_LOCATION` | `us` | Dataplex catalog location (must match dataset locations) |
| `DATABASE_ID` | `(default)` | Firestore database ID |
| `DATASET_NAME` | `census_data` | Cloud SQL database name (from Firestore) |
| `TABLE_NAME` | `census_residence_type_copy` | Cloud SQL table name |
| `INSTANCE_NAME` | `census-demo-db` | Cloud SQL instance name |
| `DB_NAME` | `census_data` | Cloud SQL database name |
| `ENTRY_GROUP_ID` | `cloudsql-entries` | Custom entry group for Cloud SQL |
| `ENTRY_TYPE_ID` | `cloudsql-table` | Custom entry type for Cloud SQL tables |

## Step-by-Step Walkthrough

### 1. Configuration and Authentication (Cells 0-4)

**Purpose:** Set up environment variables and verify access

- Sets project configuration from previous notebooks
- Verifies Application Default Credentials
- Lists required IAM roles and permissions
- Prints console links for reference

**Important:** `CATALOG_LOCATION` is set to `us` to match BigQuery and other data assets created in previous notebooks.

### 2. Initialize Clients (Cells 5-9)

**Purpose:** Install libraries and create service clients

**Libraries:**
- `google-cloud-firestore` - Read metadata from Firestore
- `google-cloud-dataplex` - Create catalog entries and aspects
- `google.protobuf` - Construct structured data for Dataplex API

**Clients:**
- `firestore.Client()` - Access Firestore documents
- `dataplex_v1.CatalogServiceClient()` - Manage Dataplex catalog resources

### 3. Read Metadata from Firestore (Cells 10-13)

**Purpose:** Retrieve Cloud SQL table metadata stored in notebook 05

**Firestore Path:**
```
datasets/{DATASET_NAME}  ← Dataset-level metadata
  └─ tables/{TABLE_NAME} ← Table-level metadata
```

**Data Retrieved:**

1. **Dataset Document** (Cell 10)
   - Connection info: `instance_name`, `connection_name`, `region`
   - Source metadata: `source_type`, `description`, `labels`

2. **Table Document** (Cell 11)
   - Schema: `fields` array (column names, types, descriptions)
   - Statistics: `row_count`, `geographic_levels`
   - Structure: `indexes`, `partitioning`, `clustering_fields`

3. **Validation** (Cells 12-13)
   - Checks that documents exist
   - Prints field metadata to verify completeness
   - Example output: `date (date): Census reference date (2021-03-21)`

**Error Handling:** If documents don't exist, notebook 05 must be run first.

### 4. Create Dataplex Entry for Cloud SQL (Cells 14-19)

**Purpose:** Create a custom catalog entry representing the Cloud SQL table

**Why Custom Entries?**
- Cloud SQL tables appear in Dataplex automatically via Universal Catalog integration (enabled in notebook 04)
- However, auto-discovery can take 2-48 hours
- Custom entries allow immediate cataloging with programmatic control
- Can add custom metadata not available in auto-discovered entries

**Step 4a: Create Entry Group** (Cells 14-15)

Entry groups organize related entries by system type.

**Configuration:**
- **Entry Group ID:** `cloudsql-entries`
- **Display Name:** "Cloud SQL Entries"
- **Description:** "Custom entries for Cloud SQL tables and databases"
- **Location:** `us` (matches catalog location)

**API Call:**
```python
catalog_service.create_entry_group(
    parent=f"projects/{PROJECT_ID}/locations/{CATALOG_LOCATION}",
    entry_group_id=ENTRY_GROUP_ID,
    entry_group=EntryGroup(...)
)
```

**Error Handling:** If entry group exists (409), continues without error.

**Step 4b: Create Entry Type** (Cells 16-17)

Entry types define the schema and metadata structure for custom entries.

**Configuration:**
- **Entry Type ID:** `cloudsql-table`
- **Display Name:** "Cloud SQL Table"
- **Description:** "Represents a table in Cloud SQL PostgreSQL"
- **Platform:** `GCP` (Google Cloud Platform)
- **System:** `CloudSQL` (identifies the source system)

**Entry Type Characteristics:**
- Platform and system fields enable filtering in Dataplex search
- Entry types are reusable across multiple entries
- Can define custom required aspects (not used in this demo)

**API Call:**
```python
catalog_service.create_entry_type(
    parent=f"projects/{PROJECT_ID}/locations/{CATALOG_LOCATION}",
    entry_type_id=ENTRY_TYPE_ID,
    entry_type=EntryType(platform="GCP", system="CloudSQL", ...)
)
```

**Step 4c: Create Entry** (Cells 18-19)

The entry represents the specific Cloud SQL table in the catalog.

**Entry Attributes:**

1. **Fully Qualified Name (FQN):**
   ```
   cloudsql_postgresql:{projectId}.{region}.{instanceId}.{databaseId}.{schemaId}.{tableId}
   ```
   Example:
   ```
   cloudsql_postgresql:johnswain-1200-20260303091357.us-central1.census-demo-db.census_data.public.census_residence_type_copy
   ```
   
   **Purpose:** Global unique identifier following CloudSQL resource hierarchy

2. **Entry ID:**
   - Format: `cloudsql-{instance}-{database}-{table}-manual`
   - Example: `cloudsql-census-demo-db-census-data-census-residence-type-copy-manual`
   - Sanitized: Underscores replaced with hyphens to meet ID requirements

3. **Entry Source:**
   - **System:** `CloudSQL`
   - **Platform:** `GCP`
   - **Display Name:** From Firestore table metadata
   - **Description:** From Firestore table metadata

4. **Entry Type Reference:**
   - Points to the `cloudsql-table` entry type created in step 4b

**API Call:**
```python
catalog_service.create_entry(
    parent=f"projects/{PROJECT_ID}/locations/{CATALOG_LOCATION}/entryGroups/{ENTRY_GROUP_ID}",
    entry_id=entry_id,
    entry=Entry(...)
)
```

**Validation:** Prints entry name and FQN after creation.

### 5. Apply Governance Aspect (Cells 20-23)

**Purpose:** Attach governance metadata to the Cloud SQL entry using the `data-governance-public` aspect type

**Aspect Type:** `data-governance-public`

This aspect type was created in notebook 01 with fields:
- `source_agency` (String) - Government agency or organization
- `data_license` (String) - License or terms of use
- `last_cataloged_date` (DateTime) - Metadata refresh timestamp

**Aspect Data for UK Census:**
```python
{
    "source_agency": "Office for National Statistics (ONS)",
    "data_license": "Open Government Licence v3.0",
    "last_cataloged_date": "2026-03-10T12:00:00Z"
}
```

**Aspect Structure:**

Aspects are organized as a dictionary with:
- **Key:** Full aspect type resource name
  ```
  {PROJECT_ID}.{CATALOG_LOCATION}.data-governance-public
  ```
- **Value:** Aspect data as a Protobuf Struct

**Why Aspects?**
- Standardized metadata fields across different data sources
- Reusable schema (same aspect type used in notebook 01 for BigQuery)
- Searchable and filterable in Dataplex catalog
- Supports data governance and compliance requirements

**API Call:**
```python
catalog_service.update_entry(
    entry=Entry(
        name=entry_name,
        aspects={
            aspect_key: Aspect(
                aspect_type=aspect_type_name,
                data=struct_pb2.Struct(...)
            )
        }
    ),
    update_mask=field_mask_pb2.FieldMask(paths=["aspects"])
)
```

**Validation:** Prints aspect key and confirms application.

### 6. Validate and Display Results (Cells 24-26)

**Purpose:** Verify entry creation and aspect attachment

**Validation Steps:**

1. **Fetch Entry** (Cell 24)
   - Retrieves the entry from Dataplex
   - Confirms it exists and is accessible

2. **Check Aspects** (Cell 25)
   - Verifies `data-governance-public` aspect is attached
   - Prints aspect data (source agency, license, timestamp)

3. **Summary and Links** (Cell 26)
   - Displays entry details:
     - Entry ID
     - Fully Qualified Name
     - Display name
     - Number of aspects applied
   - Provides console links:
     - Dataplex Catalog search
     - Cloud SQL instance
     - Entry Group

**Success Indicators:**
- Entry created successfully with Cloud SQL FQN
- `data-governance-public` aspect attached
- Entry searchable in Dataplex Catalog (may take a few minutes to index)

## Data Flow

```
Firestore (from Notebook 05)
  └─ datasets/census_data (dataset metadata)
       └─ tables/census_residence_type_copy (table metadata)
          ↓
     [Read Metadata]
          ↓
Metadata in Memory
  - Connection info
  - Schema (fields, types, descriptions)
  - Statistics (row count, indexes)
          ↓
     [Create Dataplex Resources]
          ↓
Dataplex Catalog Entry Group (cloudsql-entries)
  └─ Entry Type (cloudsql-table)
       └─ Entry (census_residence_type_copy)
            ├─ FQN: cloudsql_postgresql:{project}.{region}.{instance}.{db}.public.{table}
            ├─ Entry Source: CloudSQL / GCP
            ├─ Display Name & Description (from Firestore)
            └─ Aspects:
                 └─ data-governance-public
                      ├─ source_agency: ONS
                      ├─ data_license: OGL v3.0
                      └─ last_cataloged_date: timestamp
          ↓
     [Searchable in Dataplex Catalog]
          ↓
Unified Data Catalog
  - BigQuery tables (from notebook 01)
  - GCS objects (from notebook 02)
  - Cloud SQL tables (from this notebook + auto-discovery)
  - S3 objects (from notebook 07)
```

## Key Functions

This notebook uses procedural code without custom helper functions, relying on GCP client libraries:

- **`firestore.Client().collection().document().get()`** - Read Firestore documents
- **`catalog_service.create_entry_group()`** - Create entry group
- **`catalog_service.create_entry_type()`** - Define custom entry type
- **`catalog_service.create_entry()`** - Create catalog entry
- **`catalog_service.update_entry()`** - Apply aspects to entry
- **`struct_pb2.Struct()`** - Build Protobuf structures for aspect data

## Validation

### Verify Firestore Metadata Exists
```python
db = firestore.Client(project=PROJECT_ID)
table_ref = db.collection('datasets').document(DATASET_NAME).collection('tables').document(TABLE_NAME)
table = table_ref.get()
assert table.exists, "Table metadata not found in Firestore"
```

### Check Entry Group Creation
Navigate to [Dataplex Catalog](https://console.cloud.google.com/dataplex/catalog) and verify:
- Entry group `cloudsql-entries` exists
- Entry type `cloudsql-table` is defined

### Search for Entry in Catalog
1. Go to Dataplex > Catalog > Search
2. Enter: `census_residence_type_copy`
3. Filter by:
   - System: CloudSQL
   - Entry Group: cloudsql-entries
4. Open the entry and verify:
   - Display name and description are correct
   - FQN follows Cloud SQL format
   - `data-governance-public` aspect is attached

### Verify Aspect Data
```python
entry = catalog_service.get_entry(name=entry_name)
aspect_key = f"{PROJECT_ID}.{CATALOG_LOCATION}.data-governance-public"
aspect = entry.aspects.get(aspect_key)
print(f"Source Agency: {aspect.data['source_agency']}")
print(f"License: {aspect.data['data_license']}")
```

### Check Entry via API
```python
from google.cloud import dataplex_v1

catalog_service = dataplex_v1.CatalogServiceClient()
entry_name = f"projects/{PROJECT_ID}/locations/{CATALOG_LOCATION}/entryGroups/{ENTRY_GROUP_ID}/entries/{entry_id}"
entry = catalog_service.get_entry(name=entry_name)
print(f"Entry FQN: {entry.fully_qualified_name}")
print(f"Aspects: {list(entry.aspects.keys())}")
```

## Troubleshooting

### Issue: Firestore Documents Not Found
**Cause:** Notebook 05 wasn't run, or metadata wasn't stored correctly.
**Solution:**
1. Verify Firestore database exists: `gcloud firestore databases list`
2. Check documents in Firestore console
3. Re-run notebook 05 to regenerate metadata
4. Verify document path: `datasets/{DATASET_NAME}/tables/{TABLE_NAME}`

### Issue: "Entry Group already exists" (409 Error)
**Cause:** Entry group was created in a previous run.
**Solution:** This is expected and safe to ignore. The notebook continues with the existing entry group. To start fresh:
```python
# Delete entry group (deletes all entries within it)
catalog_service.delete_entry_group(
    name=f"projects/{PROJECT_ID}/locations/{CATALOG_LOCATION}/entryGroups/{ENTRY_GROUP_ID}"
)
```

### Issue: "Entry Type already exists" (409 Error)
**Cause:** Entry type was created in a previous run.
**Solution:** Safe to ignore. Entry types are reusable. To delete:
```python
catalog_service.delete_entry_type(
    name=f"projects/{PROJECT_ID}/locations/{CATALOG_LOCATION}/entryTypes/{ENTRY_TYPE_ID}"
)
```

### Issue: "Aspect type not found"
**Cause:** The `data-governance-public` aspect type doesn't exist.
**Solution:**
1. Run notebook 01 to create the aspect type
2. Verify aspect type exists:
   ```python
   aspect_type_name = f"projects/{PROJECT_ID}/locations/{CATALOG_LOCATION}/aspectTypes/data-governance-public"
   aspect_type = catalog_service.get_aspect_type(name=aspect_type_name)
   ```
3. If missing, create it manually in Dataplex console or via API

### Issue: Entry Not Searchable in Catalog
**Cause:** Dataplex indexing can take 2-5 minutes.
**Solution:**
- Wait a few minutes and refresh the search
- Verify entry exists via API (see Validation section)
- Check that `CATALOG_LOCATION` matches across notebooks (should be `us`)

### Issue: Invalid Entry ID or FQN
**Cause:** Special characters or incorrect format.
**Solution:**
- Entry IDs must be lowercase, alphanumeric with hyphens
- The notebook sanitizes IDs: `entry_id = entry_id.replace('_', '-')`
- FQN must follow Cloud SQL format: `cloudsql_postgresql:{project}.{region}.{instance}.{db}.{schema}.{table}`
- Verify no typos in configuration variables

### Issue: Permission Denied When Creating Entry
**Cause:** Missing `roles/dataplex.catalogAdmin` role.
**Solution:**
```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=user:YOUR_EMAIL \
  --role=roles/dataplex.catalogAdmin
```

### Issue: Aspect Data Not Displayed Correctly
**Cause:** Protobuf Struct construction error or wrong field types.
**Solution:**
1. Verify aspect type schema matches the data:
   - `source_agency`: String
   - `data_license`: String
   - `last_cataloged_date`: DateTime (RFC3339 format)
2. Check Protobuf conversion:
   ```python
   from google.protobuf import struct_pb2
   aspect_data = {
       "source_agency": "ONS",
       "data_license": "OGL v3.0",
       "last_cataloged_date": "2026-03-10T12:00:00Z"
   }
   struct = struct_pb2.Struct()
   struct.update(aspect_data)
   ```

### Issue: Cloud SQL Table Shows Twice in Catalog
**Cause:** Both custom entry (this notebook) and auto-discovered entry (Universal Catalog) exist.
**Solution:** This is expected behavior:
- Custom entry: `cloudsql-entries` entry group, immediate, manually managed
- Auto-discovered entry: `@cloudsql` system entry group, takes 2-48 hours, automatic updates
- Use filters in search to distinguish them
- Custom entries allow richer metadata (like aspects) before auto-discovery completes

## Next Steps

After completing this notebook, you have:
- ✓ Read Cloud SQL metadata from Firestore
- ✓ Created a custom Dataplex entry for the Cloud SQL table
- ✓ Applied governance aspects with ONS/UK Census provenance
- ✓ Unified metadata across BigQuery, GCS, and Cloud SQL in Dataplex

**Extend the Workflow:**
- Add more aspects: data quality, SLA, ownership, sensitivity classification
- Link to glossary terms (similar to notebook 03)
- Create entries for additional Cloud SQL tables or databases
- Automate metadata sync with Cloud Functions triggered on Firestore changes

**Continue to:** [`07_import_s3_to_gcs_with_lineage.md`](07_import_s3_to_gcs_with_lineage.md) to:
- Import data from AWS S3 to GCS
- Create custom Dataplex entries for S3 objects
- Record full data lineage (S3 → GCS → BigQuery)

**Compare with Auto-Discovery:**
- Wait 2-48 hours after enabling Dataplex integration in notebook 04
- Search Dataplex for Cloud SQL tables under `@cloudsql` system entry group
- Compare auto-discovered metadata with custom entries
- Note: Auto-discovered entries sync automatically but have limited customization

**Use the Unified Catalog:**
- Search across all data sources in Dataplex
- Filter by aspects (e.g., "Show all public datasets")
- Build data lineage graphs
- Create data quality rules and monitoring

## Additional Resources

- [Dataplex Universal Catalog Documentation](https://cloud.google.com/dataplex/docs/universal-catalog)
- [Dataplex Custom Entries](https://cloud.google.com/dataplex/docs/catalog-custom-entries)
- [Dataplex Aspect Types](https://cloud.google.com/dataplex/docs/catalog-aspect-types)
- [Cloud SQL Dataplex Integration](https://cloud.google.com/sql/docs/postgres/dataplex)
- [Firestore Python Client Library](https://googleapis.dev/python/firestore/latest/index.html)
- [Dataplex Python Client Library](https://googleapis.dev/python/dataplex/latest/index.html)
- [Protobuf Struct Documentation](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#struct)
