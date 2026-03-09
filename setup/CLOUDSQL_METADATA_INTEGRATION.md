# CloudSQL Metadata Integration in Firestore

## Overview

The `setup_firestore.ipynb` notebook has been extended with CloudSQL metadata storage functionality. This integration extracts table and column metadata from CloudSQL PostgreSQL and stores it in Firestore using a hierarchical schema that mirrors the BigQuery structure.

## What Was Added

Four new sections were added to the notebook (Sections 6-9), containing 24 new cells:

### Section 6: CloudSQL Metadata Storage (Cells 29-33)

**Purpose**: Configure and connect to CloudSQL PostgreSQL instance

**Key Features**:
- CloudSQL connection configuration (instance, database, credentials)
- Install required libraries: `cloud-sql-python-connector`, `pg8000`, `sqlalchemy`, `certifi`
- SSL certificate configuration for secure connections
- Connection verification to the `census_data` database

### Section 7: Extract CloudSQL Table Metadata (Cells 34-39)

**Purpose**: Query PostgreSQL system tables to extract comprehensive metadata

**Metadata Extracted**:
- **Column Information**: Name, data type, length, nullability from `information_schema.columns`
- **Primary Keys**: Constraint information from `information_schema.table_constraints`
- **Indexes**: Index definitions from `pg_indexes`
- **Table Statistics**: Row counts and geographic level distribution
- **Field Descriptions**: Business-friendly descriptions for each column

**Example Output**:
```
✅ Found 8 columns in table 'census_residence_type'
✅ Primary key columns: geography_code
✅ Found 4 indexes
   - census_residence_type_pkey
   - idx_geography
   - idx_geography_level
   - idx_total_residents
✅ Table statistics:
   Total rows: 232,157
   Geographic levels: OA, LSOA, MSOA, LTLA, RGN
```

### Section 8: Store Metadata in Firestore (Cells 40-44)

**Purpose**: Write extracted metadata to Firestore in hierarchical structure

**Firestore Schema**:

```
Collection: datasets
  Document: census_data
    Fields:
      - database_name: "census_data"
      - source_type: "cloudsql_postgresql"
      - instance_name: "census-demo-db"
      - connection_name: "project:region:instance"
      - region: "us-central1"
      - description: "UK Census 2021 demographic data"
      - labels: { data_source, year, survey }
      - last_sync: SERVER_TIMESTAMP
    
    Subcollection: tables
      Document: census_residence_type
        Fields:
          - tableName: "census_residence_type"
          - description: "Census 2021 TS001 - ..."
          - row_count: 232157
          - geographic_levels: ["OA", "LSOA", "MSOA", "LTLA", "RGN"]
          - indexes: [{ name, definition }, ...]
          - fields: [
              {
                name: "geography_code",
                type: "VARCHAR(20)",
                mode: "REQUIRED",
                description: "Unique geographic identifier code",
                is_primary_key: true,
                is_sensitive: false
              },
              ... (8 fields total)
            ]
          - partitioning: null
          - clustering_fields: null
          - last_sync: SERVER_TIMESTAMP
```

**Key Features**:
- Uses `merge=True` to preserve existing metadata
- Stores complete schema with all 8 columns
- Includes business descriptions for each field
- Tracks sync timestamps

### Section 9: Query and Validate Stored Metadata (Cells 45-52)

**Purpose**: Demonstrate querying capabilities and validate storage

**Query Examples**:

1. **Read Dataset Document**: Retrieve and display dataset-level metadata
2. **Read Table Document**: Retrieve and display table schema
3. **Display Field Metadata**: Show all columns with descriptions
4. **Query 1 - Get All Tables in Dataset**: List all tables in `census_data`
5. **Query 2 - Collection Group Query**: Find all tables containing `geography_code` field across ALL datasets
6. **Query 3 - Filter by Source Type**: Find all CloudSQL PostgreSQL datasets

**Example Output**:
```
✅ Dataset document retrieved:
   Database: census_data
   Source Type: cloudsql_postgresql
   Instance: census-demo-db
   Region: us-central1

✅ Table document retrieved:
   Table Name: census_residence_type
   Row Count: 232,157
   Geographic Levels: OA, LSOA, MSOA, LTLA, RGN
   Number of Fields: 8
   Number of Indexes: 4
```

## Hierarchical Schema Benefits

### 1. Searchability
Collection Group queries enable finding tables by field names across all datasets:
```python
# Find all tables with 'geography_code' field
tables_ref = db.collection_group("tables")
```

### 2. Hierarchical Navigation
Mirrors the CloudSQL structure:
- Instance → Database (dataset document)
- Database → Tables (tables subcollection)
- Tables → Fields (fields array)

### 3. Extensibility
Easy to add new metadata fields without schema migrations:
- Data quality metrics
- PII classification
- Data lineage
- Usage statistics

### 4. Automation Ready
Can be triggered automatically:
- Cloud Functions on CloudSQL schema changes
- Scheduled Cloud Run jobs for periodic sync
- Event-driven updates via Pub/Sub

### 5. Unified Catalog
Same pattern can be used for:
- BigQuery tables
- Cloud Storage buckets
- Cloud Spanner databases
- External data sources

## Usage

### Prerequisites
1. Complete `setup_cloudsql_census.ipynb` to create the CloudSQL instance and table
2. Complete sections 1-5 of `setup_firestore.ipynb` to create the Firestore database

### Running the Integration
Execute sections 6-9 in order:
1. **Section 6**: Configure and connect to CloudSQL (cells 29-33)
2. **Section 7**: Extract metadata (cells 34-39)
3. **Section 8**: Write to Firestore (cells 40-44)
4. **Section 9**: Query and validate (cells 45-52)

### Estimated Time
- 2-3 minutes to run all new sections
- Instant if CloudSQL instance is already running

## Accessing Stored Metadata

### Firestore Console
View the stored metadata at:
https://console.cloud.google.com/firestore/data?project=YOUR_PROJECT_ID

Navigate to:
- `datasets` collection → `census_data` document
- `datasets/census_data/tables` subcollection → `census_residence_type` document

### Python API
```python
from google.cloud import firestore

db = firestore.Client(project=PROJECT_ID)

# Get dataset metadata
dataset = db.collection("datasets").document("census_data").get()
print(dataset.to_dict())

# Get table metadata
table = db.collection("datasets").document("census_data") \
          .collection("tables").document("census_residence_type").get()
print(table.to_dict())

# Find all tables with a specific field
tables = db.collection_group("tables").stream()
for table in tables:
    fields = table.to_dict().get('fields', [])
    if any(f['name'] == 'geography_code' for f in fields):
        print(f"Found: {table.reference.path}")
```

## Next Steps

### 1. Add More Tables
Modify the extraction queries to process multiple tables:
```python
tables_to_extract = ["census_residence_type", "other_table_1", "other_table_2"]
for table_name in tables_to_extract:
    # Extract and store metadata
    pass
```

### 2. Add BigQuery Integration
Use the same hierarchical pattern for BigQuery:
```
Collection: datasets
  Document: bigquery_dataset_name
    Fields: { source_type: "bigquery", ... }
    Subcollection: tables
      Document: table_name
        Fields: { fields: [...] }
```

### 3. Build Search Interface
Create a web UI to search across all data sources:
- Search by field name
- Filter by data type
- Find tables by description keywords
- View lineage and relationships

### 4. Add Custom Metadata
Extend the schema with domain-specific fields:
- Data quality scores
- PII classification tags
- Business glossary terms
- Data owner and steward information
- Refresh schedules

### 5. Automate Sync
Set up automated metadata synchronization:
```python
# Cloud Function triggered on CloudSQL schema changes
def sync_cloudsql_metadata(event, context):
    # Extract metadata
    # Write to Firestore
    pass
```

## Cost Considerations

### Firestore Free Tier
- 1 GB storage
- 50,000 document reads/day
- 20,000 document writes/day
- 20,000 document deletes/day

### This Integration Usage
- **Storage**: < 1 MB per table
- **Writes**: 2 documents per table (1 dataset + 1 table)
- **Reads**: Depends on query frequency

**Conclusion**: Well within free tier limits for typical usage.

## Troubleshooting

### Connection Failed
```
❌ Failed to connect to CloudSQL: ...
```
**Solution**: 
1. Verify CloudSQL instance is running
2. Check `setup_cloudsql_census.ipynb` was completed successfully
3. Verify connection details (instance name, database, password)

### SSL Certificate Error
```
❌ SSL certificate verification failed
```
**Solution**: The notebook includes SSL configuration with certifi. If issues persist:
```python
import certifi
print(certifi.where())  # Verify certificate bundle path
```

### Firestore Write Failed
```
❌ Failed to write dataset document: ...
```
**Solution**:
1. Verify Firestore database exists (section 3)
2. Check IAM permissions: `roles/datastore.owner`
3. Ensure Firestore API is enabled

## Files Modified

- `setup/setup_firestore.ipynb`: Added 24 cells (sections 6-9)

## Related Documentation

- [Firestore Data Model](https://firebase.google.com/docs/firestore/data-model)
- [Firestore Collection Group Queries](https://firebase.google.com/docs/firestore/query-data/queries#collection-group-query)
- [Cloud SQL Python Connector](https://github.com/GoogleCloudPlatform/cloud-sql-python-connector)
- [PostgreSQL Information Schema](https://www.postgresql.org/docs/current/information-schema.html)
