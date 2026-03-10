# Configuration and Data Setup

This notebook sets up the foundational components for the Dataplex demo by copying US Census Bureau American Community Survey (ACS) data into your GCP project, creating custom Dataplex aspect types, and applying metadata enrichment to the census tables.

## Prerequisites

### IAM Roles Required
- `roles/bigquery.admin` - Create datasets, copy tables, and query data
- `roles/dataplex.admin` - Create and manage Dataplex resources
- `roles/dataplex.catalogAdmin` - Create aspect types and apply aspects to entries

### APIs to Enable
- BigQuery API
- BigQuery Data Transfer API
- Dataplex API

### Prior Notebooks
None - this is the first notebook in the setup sequence.

## GCP Services Used

- **BigQuery** - Data warehouse for storing and querying census tables
- **BigQuery Data Transfer Service** - Copying tables from public datasets
- **Dataplex Catalog** - Metadata management and custom aspect types
- **Application Default Credentials** - Authentication

## Configuration

### Key Variables

| Variable | Example Value | Purpose |
|----------|---------------|---------|
| `PROJECT_ID` | `johnswain-1200-20260303091357` | Your GCP project ID |
| `REGION` | `us-central1` | GCP region for resources |
| `DATASET_ID` | `census_bureau_acs` | BigQuery dataset to create |
| `DATASET_LOCATION` | `US` | Multi-region location for BigQuery dataset |
| `CATALOG_LOCATION` | `us` | Location for Dataplex catalog resources |
| `SOURCE_PROJECT` | `bigquery-public-data` | Source of census data |
| `SOURCE_DATASET` | `census_bureau_acs` | Source dataset to copy |

**Important Note:** The `CATALOG_LOCATION` must match the BigQuery dataset location (both set to `US`/`us`) for proper catalog integration.

## Step-by-Step Walkthrough

### 1. Configuration and Authentication (Cells 1-4)

**Purpose:** Verify environment and authentication setup

- Sets project ID and region variables
- Verifies Application Default Credentials using `google.auth.default()`
- Lists required IAM roles
- Provides instructions for granting roles via `gcloud` commands

**Key Command:**
```bash
gcloud auth application-default login
```

### 2. Create BigQuery Dataset and Copy Tables (Cells 5-13)

**Purpose:** Copy 278 census tables from public data into your project

**Steps:**

1. **Install Dependencies** - Installs `google-cloud-bigquery` and `google-cloud-bigquery-datatransfer`
2. **Initialize BigQuery Client** - Creates client with project credentials
3. **Create Dataset** - Creates `census_bureau_acs` dataset in `US` multi-region
4. **List Source Tables** - Queries `bigquery-public-data.census_bureau_acs` to enumerate all 278 tables
5. **Copy Tables** - Iterates through each table and copies it using `bigquery.CopyJobConfig`
   - Copies one table at a time
   - Uses `WRITE_TRUNCATE` disposition
   - Typical runtime: 10-30 minutes for all tables
6. **Validation** - Counts copied tables and lists a sample

**Table Naming Convention:** Tables follow the pattern `{geography}_{vintage}_{period}yr`
- Examples: `blockgroup_2018_5yr`, `cbsa_2010_1yr`, `county_2015_5yr`
- Geographies: blockgroup, cbsa, county, state, tract, etc.
- Vintages: 2007-2018
- Periods: 1yr or 5yr estimates

### 3. Create Dataplex Aspect Types (Cells 14-24)

**Purpose:** Define custom metadata schemas for census data governance

**Aspect Type 1: Census Survey Metadata** (`census-survey-metadata`)

Captures survey-specific metadata for each table:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `survey_vintage` | Integer | Yes | Year the survey was conducted (2007-2018) |
| `estimate_period` | Enum | Yes | Estimation period: `1_year` or `5_year` |
| `is_experimental` | Boolean | No | Whether the table contains experimental data |

**Aspect Type 2: Public Data Governance** (`data-governance-public`)

Captures governance and provenance metadata:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `source_agency` | String | No | Government agency providing the data |
| `data_license` | String | No | License or terms of use |
| `last_cataloged_date` | DateTime | No | When metadata was last updated |

**Location:** Both aspect types are created in location `us` to match the BigQuery dataset location.

### 4. Apply Aspects to BigQuery Tables (Cells 25-33)

**Purpose:** Enrich all 278 census tables with structured metadata

**Key Components:**

1. **Helper Function: `get_survey_metadata(table_name)`**
   - Parses table names to extract vintage and period
   - Returns tuple: `(vintage: int, period: "1_year" | "5_year")`
   - Example: `blockgroup_2018_5yr` → `(2018, "5_year")`

2. **Entry Discovery**
   - Searches for BigQuery entries under `@bigquery` system
   - Uses entry filter: `project={PROJECT_ID} dataset={DATASET_ID} table=*`
   - Entry ID format: `bigquery.googleapis.com/projects/{project}/datasets/{dataset}/tables/{table}`

3. **Aspect Application**
   - Applies `census-survey-metadata` aspect with parsed vintage and period
   - Applies `data-governance-public` aspect with:
     - `source_agency`: "US Census Bureau"
     - `data_license`: "Public Domain"
     - `last_cataloged_date`: Current timestamp

4. **Validation**
   - Retrieves a sample table entry
   - Verifies both aspects are attached
   - Prints aspect values for confirmation

## Data Flow

```
bigquery-public-data.census_bureau_acs (278 tables)
          ↓
     [BigQuery Copy Job]
          ↓
{PROJECT_ID}.census_bureau_acs (278 tables)
          ↓
     [Dataplex Catalog Discovery]
          ↓
BigQuery Entries in Dataplex (location: us)
          ↓
     [Apply Aspects]
          ↓
Enriched Catalog Entries with Custom Metadata
```

## Key Functions

### `get_survey_metadata(table_name: str) -> tuple[int, str]`

Extracts survey metadata from table names following the naming convention.

**Logic:**
- Splits table name by underscore
- Second-to-last part is the vintage year (e.g., "2018")
- Last part contains the period (e.g., "5yr" → "5_year")
- Returns `(vintage, period)` tuple

**Examples:**
```python
get_survey_metadata("blockgroup_2018_5yr")  # Returns (2018, "5_year")
get_survey_metadata("cbsa_2010_1yr")        # Returns (2010, "1_year")
```

## Validation

### Verify Dataset Creation
```python
dataset = bigquery_client.get_dataset(f"{PROJECT_ID}.{DATASET_ID}")
print(f"Dataset {dataset.dataset_id} exists in {dataset.location}")
```

### Count Copied Tables
```python
tables = list(bigquery_client.list_tables(dataset))
print(f"Copied {len(tables)} tables")  # Should show 278
```

### Verify Aspects Applied
```python
sample_entry = catalog_client.get_entry(
    name=f"projects/{PROJECT_ID}/locations/{CATALOG_LOCATION}/entryGroups/@bigquery/entries/{entry_id}"
)
print(f"Aspects applied: {list(sample_entry.aspects.keys())}")
# Should show: ['{PROJECT_ID}.us.census-survey-metadata', '{PROJECT_ID}.us.data-governance-public']
```

### Check in Console
Navigate to [Dataplex Catalog](https://console.cloud.google.com/dataplex/catalog) and search for one of your census tables to view the applied aspects.

## Troubleshooting

### Issue: "Dataset already exists" Error
**Cause:** The dataset was created in a previous run.
**Solution:** Either delete the existing dataset or set `exists_ok=True` in the dataset creation config.

### Issue: "Permission denied" on BigQuery Copy
**Cause:** Missing `roles/bigquery.admin` or `roles/bigquery.jobUser`.
**Solution:** Grant the required roles:
```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=user:YOUR_EMAIL \
  --role=roles/bigquery.admin
```

### Issue: "Aspect type not found" When Applying Aspects
**Cause:** Aspect types weren't created or were created in a different location.
**Solution:** 
1. Verify aspect types exist: Check Dataplex console under "Aspect Types"
2. Ensure `CATALOG_LOCATION` matches dataset location (both `us`)
3. Re-run the aspect type creation cells

### Issue: Copy Jobs Take Too Long
**Cause:** Copying 278 tables can take 10-30 minutes.
**Solution:** This is normal. Consider:
- Running the notebook during off-hours
- Copying only a subset of tables for testing (modify the table list)
- Using a batch job with BigQuery Data Transfer Service

### Issue: "Entry not found" During Aspect Application
**Cause:** BigQuery entries haven't been indexed by Dataplex yet.
**Solution:** Wait 2-5 minutes after dataset creation, then re-run the aspect application cells. Dataplex automatically discovers BigQuery resources.

## Next Steps

After completing this notebook, you have:
- ✓ 278 US Census ACS tables in BigQuery
- ✓ Two custom Dataplex aspect types
- ✓ All tables enriched with survey metadata and governance aspects

**Continue to:** [`02_upload_census_to_gcs.md`](02_upload_census_to_gcs.md) to set up UK Census 2021 data in Google Cloud Storage and BigLake.

**Alternative Path:** Jump to [`03_apply_glossary_terms.md`](03_apply_glossary_terms.md) to create a business glossary for the ACS data you just set up.

## Additional Resources

- [Dataplex Aspect Types Documentation](https://cloud.google.com/dataplex/docs/catalog-aspect-types)
- [BigQuery Public Datasets - Census ACS](https://console.cloud.google.com/marketplace/product/united-states-census-bureau/acs)
- [US Census Bureau ACS Documentation](https://www.census.gov/programs-surveys/acs)
- [ISO 11179 Metadata Registry Standard](https://www.iso.org/standard/50340.html)
