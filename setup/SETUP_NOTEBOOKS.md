# Setup Notebooks Documentation

This directory contains four Jupyter notebooks for setting up, enhancing, and cleaning up GCP Dataplex demo resources.

## config_and_data_setup.ipynb

**Purpose**: Complete setup notebook for creating and configuring a GCP Dataplex demo environment with US Census data.

### What it does:

1. **Configuration & Authentication** (Section 1)
   - Configures GCP project ID and region settings
   - Verifies authentication credentials
   - Checks required IAM permissions (BigQuery Admin, Dataplex Admin, Dataplex Catalog Admin)

2. **BigQuery Dataset Creation** (Section 2)
   - Copies ~278 tables from the public `bigquery-public-data.census_bureau_acs` dataset
   - Creates a new `census_bureau_acs` dataset in your project
   - Includes US Census Bureau American Community Survey data (blockgroup, CBSA, census tract data)
   - Data spans multiple years (2007-2018) and survey periods (1-year, 3-year, 5-year)
   - Validates successful table copying with progress tracking

3. **Dataplex Aspect Types Creation** (Section 3)
   - Creates two custom aspect types for metadata enrichment:
     - **Census Survey Metadata**: Captures survey vintage (year), estimate period (1-year/5-year), and experimental data flag
     - **Public Data Governance**: Captures source agency, license type, and last cataloged date
   - Validates aspect type creation and structure

4. **Apply Aspects to BigQuery Tables** (Section 4)
   - Parses table names to extract metadata (year, estimate period)
   - Applies both aspect types to all 278 census tables automatically
   - Enriches the data catalog with searchable, structured metadata
   - Validates aspect application on sample tables

### Time Estimate
10-30 minutes depending on data transfer speed

### Prerequisites
- GCP project with billing enabled
- Required IAM roles:
  - `roles/bigquery.admin`
  - `roles/dataplex.admin`
  - `roles/dataplex.catalogAdmin`

---

## upload_census_to_gcs.ipynb

**Purpose**: Complete workflow to upload UK Census 2021 dataset (TS001) to Google Cloud Storage and load into BigQuery for analysis.

### What it does:

1. **Configuration & Authentication** (Section 1)
   - Configures GCP project ID, region, and bucket name
   - Verifies authentication credentials
   - Checks required IAM permissions (Cloud Storage Admin, BigQuery Admin)
   - Installs required libraries (google-cloud-storage, google-cloud-bigquery)

2. **Create GCS Bucket** (Section 2)
   - Initializes Cloud Storage client
   - Creates a new GCS bucket in the specified region
   - Handles cases where bucket already exists gracefully
   - Provides console link to view bucket

3. **Upload Census Files** (Section 3)
   - Walks through `source_data/census2021-ts001/` directory
   - Uploads 7 census CSV files at different geographic levels:
     - Country (England, Wales)
     - Region
     - Upper Tier Local Authority (UTLA)
     - Lower Tier Local Authority (LTLA)
     - Middle Layer Super Output Area (MSOA)
     - Lower Layer Super Output Area (LSOA)
     - Output Area (OA)
   - Uploads metadata documentation file
   - Excludes hidden files (.DS_Store)
   - Displays upload progress with file sizes

4. **Validation** (Section 4)
   - Lists all uploaded files from the bucket
   - Verifies file count matches expected (8 files)
   - Displays bucket statistics (file count, total size)
   - Cross-references uploaded files against expected files

5. **Summary & Next Steps** (Section 5)
   - Displays summary of resources created (both GCS and BigQuery)
   - Provides console links and command-line examples
   - Suggests next steps and cleanup instructions

6. **Create BigQuery Dataset** (Section 6)
   - Installs BigQuery client library
   - Creates dataset `census_uk_2021` in US location
   - Handles cases where dataset already exists gracefully
   - Provides BigQuery Console link

7. **Load CSV from GCS to BigQuery** (Section 7)
   - Defines table schema with proper data types
   - Renames CSV columns to be BigQuery-friendly (no spaces)
   - Loads UTLA census data from GCS to BigQuery table `ts001_utla`
   - Displays load statistics (rows, table size)

8. **Validation & Query Examples** (Section 8)
   - Validates row count (175 ULTAs expected)
   - Previews top 10 local authorities by population
   - Runs aggregate statistics query
   - Provides sample SQL queries for further analysis
   - Displays BigQuery Console links

### Dataset Information

**Census 2021 - TS001**: Number of usual residents in households and communal establishments
- Source: UK Office for National Statistics (ONS)
- Release Date: 2022-12-13
- Coverage: England and Wales
- Geographic Level Loaded: UTLA (Upper Tier Local Authorities) - 175 rows

### Time Estimate
10-15 minutes

### Prerequisites
- GCP project with billing enabled
- Required IAM roles: 
  - `roles/storage.admin` (for GCS operations)
  - `roles/bigquery.admin` (for BigQuery operations)
- Census data files in `source_data/census2021-ts001/`

---

## apply_glossary_terms.ipynb

**Purpose**: Creates a Dataplex Business Glossary following ISO 11179 principles for organizing census demographic data with business terminology.

### What it does:

1. **Configuration & Authentication** (Section 1)
   - Configures GCP project settings and validates authentication
   - Installs required libraries (Dataplex, BigQuery)
   - Initializes API clients

2. **Analyze Table Schema** (Section 2)
   - Queries BigQuery INFORMATION_SCHEMA to get all columns from `blockgroup_2018_5yr`
   - Categorizes columns using pattern matching (age/sex, race/ethnicity, commuting, labor force, education)
   - Displays categorization results with statistics

3. **Create Business Glossary** (Section 3)
   - Creates root glossary: "ACS Demographics Glossary"
   - Implements ISO 11179 organizational structure
   - Sets up glossary in US multi-region location

4. **Create Categories (Object Classes)** (Section 4)
   - Creates 5 ISO 11179 Object Class categories:
     - Age and Sex
     - Race and Ethnicity
     - Commuting Characteristics
     - Labor Force
     - Educational Attainment
   - Each category includes descriptions aligned with ISO 11179 principles

5. **Create Glossary Terms** (Section 5)
   - Generates business terms for categorized columns
   - Creates human-readable display names (e.g., "Male Population Age 35 to 39")
   - Adds contextual descriptions for each term
   - Configurable term limits (default: 20 terms per category)

6. **Link Terms to Data Assets** (Section 6)
   - Links glossary terms to BigQuery table columns
   - Creates `related_entries` relationships
   - Enables discovery of data through business terminology

7. **Validation & Exploration** (Section 7)
   - Lists all created terms by category
   - Validates term linkages with sample data
   - Provides console links for manual exploration

8. **Summary & Next Steps** (Section 8)
   - Displays comprehensive statistics (categories, terms, linkages, coverage)
   - Provides guidance for extending to more tables
   - Offers best practices for glossary maintenance
   - Exports glossary structure to JSON

### Time Estimate
15-25 minutes depending on number of terms created

### Prerequisites
- Census dataset already created (via `config_and_data_setup.ipynb`)
- Required IAM roles:
  - `roles/dataplex.catalogAdmin` (to create glossary, categories, and terms)
  - `roles/bigquery.admin` (to query table schemas)

### ISO 11179 Alignment
- **Object Classes**: Implemented as Dataplex Glossary Categories
- **Data Elements**: Implemented as Glossary Terms linked to table columns
- **Classification Scheme**: Hierarchical category structure for demographic concepts

---

## cleanup_all_resources.ipynb

**Purpose**: Complete cleanup notebook to remove all resources created by the setup notebook.

### What it does:

1. **Configuration & Authentication** (Section 1)
   - Sets up GCP project and region configuration
   - Verifies authentication
   - Installs required libraries (BigQuery, Dataplex)

2. **Remove Aspects from BigQuery Tables** (Section 2)
   - Lists all tables in the `census_bureau_acs` dataset
   - Removes both aspect types (census-survey-metadata and data-governance-public) from all table entries
   - Processes ~278 tables with progress tracking

3. **Delete Dataplex Aspect Types** (Section 3)
   - Deletes the `census-survey-metadata` aspect type
   - Deletes the `data-governance-public` aspect type
   - Handles cases where aspect types are already deleted

4. **Delete BigQuery Dataset** (Section 4)
   - Permanently deletes the entire `census_bureau_acs` dataset
   - Removes all ~278 tables and their data
   - Uses `delete_contents=True` to ensure complete cleanup

5. **Cleanup Summary** (Section 5)
   - Provides summary of deleted resources
   - Includes console links to verify cleanup
   - Optional instructions for removing IAM permissions

### ⚠️ Warning
- This operation **cannot be undone**
- All data in the `census_bureau_acs` dataset will be **permanently deleted**
- All custom metadata (aspects) will be removed

### Time Estimate
5-10 minutes

---

## Usage Workflow

```bash
# 1. Run setup to create demo environment
jupyter notebook setup/config_and_data_setup.ipynb

# 2. (Optional) Upload UK Census 2021 data to GCS
jupyter notebook setup/upload_census_to_gcs.ipynb

# 3. (Optional) Apply business glossary for ISO 11179-compliant metadata
jupyter notebook setup/apply_glossary_terms.ipynb

# 4. Explore Dataplex features and test functionality
# ... your demo/testing activities ...

# 5. Run cleanup to remove all resources
jupyter notebook setup/cleanup_all_resources.ipynb
```

## Console Links

After setup, you can view resources at:
- **BigQuery Dataset**: `https://console.cloud.google.com/bigquery?project=<PROJECT_ID>&d=census_bureau_acs`
- **Cloud Storage Bucket**: `https://console.cloud.google.com/storage/browser/<BUCKET_NAME>?project=<PROJECT_ID>`
- **Dataplex Aspect Types**: `https://console.cloud.google.com/dataplex/govern/aspect-types?project=<PROJECT_ID>`
- **Dataplex Catalog**: `https://console.cloud.google.com/dataplex/search?project=<PROJECT_ID>`
- **Business Glossaries**: `https://console.cloud.google.com/dataplex/dp-glossaries?project=<PROJECT_ID>`

## Notes

- All notebooks include robust error handling and progress tracking
- Setup notebook is idempotent - can be rerun safely if interrupted
- Glossary notebook can be extended to cover additional tables and columns
- Cleanup notebook handles missing resources gracefully
- All operations include detailed console output with status indicators
