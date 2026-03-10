# Apply Glossary Terms to Census Bureau ACS Dataset

This notebook creates an ISO 11179-compliant Business Glossary in Dataplex, organizes US Census ACS columns into semantic categories (Object Classes), generates human-readable glossary terms, and links them to BigQuery table columns for enhanced data discovery and governance.

## Prerequisites

### IAM Roles Required
- `roles/dataplex.catalogAdmin` - Create glossaries, categories, terms, and entry links
- `roles/bigquery.admin` - Query table schemas via INFORMATION_SCHEMA

### APIs to Enable
- Dataplex API
- BigQuery API

### Prior Notebooks
**Required:** [`01_config_and_data_setup.md`](01_config_and_data_setup.md) - This notebook depends on:
- The `census_bureau_acs` BigQuery dataset with ACS tables
- The `blockgroup_2018_5yr` table for schema analysis
- BigQuery entries indexed in Dataplex

## GCP Services Used

- **BigQuery** - Query `INFORMATION_SCHEMA.COLUMNS` to inspect table schemas
- **Dataplex Catalog** - Create and manage glossaries, categories, terms, and entry links
- **Dataplex REST API** - Direct API calls for glossary management (Python client library has limited glossary support)
- **Cloud Resource Manager** - Retrieve project number for entry references

## Configuration

### Key Variables

| Variable | Example Value | Purpose |
|----------|---------------|---------|
| `PROJECT_ID` | `johnswain-1200-20260303091357` | Your GCP project ID |
| `CATALOG_LOCATION` | `us` | Location for Dataplex catalog resources |
| `DATASET_ID` | `census_bureau_acs` | BigQuery dataset with census tables |
| `TABLE_ID` | `blockgroup_2018_5yr` | Sample table to analyze (155 columns) |
| `GLOSSARY_ID` | `acs-demographics-glossary` | Business glossary identifier |
| `MAX_TERMS_PER_CATEGORY` | `20` | Demo limit for terms per category |

**Note:** The notebook analyzes one representative table (`blockgroup_2018_5yr`) but the approach can be extended to all 278 ACS tables.

## Step-by-Step Walkthrough

### 1. Configuration and Authentication (Cells 1-5)

**Purpose:** Initialize project settings and verify credentials

- Imports `PROJECT_ID` and `DATASET_ID` from notebook 01 configuration
- Sets `CATALOG_LOCATION` to `us` (matches BigQuery dataset location)
- Selects `blockgroup_2018_5yr` as the sample table (representative ACS schema)
- Defines `GLOSSARY_ID` for the business glossary
- Verifies Application Default Credentials
- Lists required IAM roles

### 2. Analyze Table Schema (Cells 6-7)

**Purpose:** Extract column names and types from the BigQuery table

**Process:**
1. Queries `INFORMATION_SCHEMA.COLUMNS` for the target table
2. Retrieves 155 columns with their data types
3. Stores results in a pandas DataFrame for processing

**Sample Columns:**
- `male_35_to_39` (INT64) - Male population aged 35-39
- `workers_16_and_over` (INT64) - Workers aged 16+
- `bachelors_degree` (INT64) - People with bachelor's degrees
- `commute_15_to_19_mins` (INT64) - Commute time 15-19 minutes
- `black_including_hispanic` (INT64) - Race/ethnicity category

**Column Naming Pattern:** Columns use underscore-separated descriptive names that encode meaning (e.g., gender, age range, category).

### 3. Create Business Glossary (Cells 8-11)

**Purpose:** Create a glossary container following ISO 11179 Metadata Registry standards

**API Call:** Uses Dataplex REST API (POST)
```
POST https://dataplex.googleapis.com/v1/projects/{PROJECT_ID}/locations/{CATALOG_LOCATION}/glossaries?glossaryId={GLOSSARY_ID}
```

**Glossary Properties:**
- **ID:** `acs-demographics-glossary`
- **Display Name:** "ACS Demographics Glossary"
- **Description:** Follows ISO 11179 for metadata organization
- **Location:** `us` (matches catalog location)

**Error Handling:** Returns 409 if glossary already exists (safe to ignore on re-runs).

### 4. Create Categories (Object Classes) (Cells 12-14)

**Purpose:** Organize columns into semantic categories following ISO 11179 Object Classes

**ISO 11179 Object Classes:** Categories represent real-world concepts that data elements describe (e.g., "Person", "Geography", "Time").

**Five Categories Defined:**

1. **Age and Sex** (`age-and-sex`)
   - Columns: `male_*`, `female_*`, `*_to_*_years`, age ranges
   - Examples: `male_35_to_39`, `female_62_to_64`

2. **Race and Ethnicity** (`race-and-ethnicity`)
   - Columns: `white_*`, `black_*`, `asian_*`, `hispanic_*`, `*_alone_or_in_combination`
   - Examples: `white_including_hispanic`, `asian_alone`

3. **Commuting Characteristics** (`commuting-characteristics`)
   - Columns: `commute_*`, `commuters_*`, travel time/mode
   - Examples: `commute_15_to_19_mins`, `commuters_by_public_transportation`

4. **Labor Force** (`labor-force`)
   - Columns: `workers_*`, `employed_*`, `unemployed_*`, labor status
   - Examples: `workers_16_and_over`, `employed_pop`

5. **Educational Attainment** (`educational-attainment`)
   - Columns: `*_degree`, `*_school`, education levels
   - Examples: `bachelors_degree`, `high_school_graduate`

**Categorization Function: `categorize_column(column_name)`**

Uses regex pattern matching to classify columns:
- Returns category ID if matched (e.g., `age-and-sex`)
- Returns `None` for uncategorized columns (e.g., `geo_id`, `total_pop`)

**Results:** Out of 155 columns:
- 96 categorized
- 59 uncategorized (mostly IDs, totals, and granular subcategories)

**API Calls:** Creates each category via POST to glossary categories endpoint.

**Known Issue:** Category creation may return 400 errors depending on API version/format. If categories are created manually in console, the subsequent steps will succeed.

### 5. Create Glossary Terms (Cells 15-18)

**Purpose:** Generate human-readable business terms for each categorized column

**Term Generation Functions:**

**`column_to_display_name(column_name)`**
Converts snake_case column names to readable labels:
- Splits on underscores
- Capitalizes words
- Joins with spaces
- Example: `male_35_to_39` → "Male 35 To 39" (further formatted to "Male Population Age 35 to 39")

**`generate_term_description(column_name, column_type, category_id)`**
Builds detailed term descriptions:
- Includes column name, data type, and category context
- Follows ISO 11179 data element definition guidelines
- Example: "Represents the male_35_to_39 column (INT64) in the census dataset. This field is part of the Age and Sex demographic category."

**Term Creation Process:**
1. Groups columns by category
2. Limits to `MAX_TERMS_PER_CATEGORY` (20) for demo purposes
3. Creates terms via REST API POST to category terms endpoint
4. Tracks successful creations

**Results:** 53 terms created across 5 categories

**API Call Structure:**
```
POST .../glossaries/{GLOSSARY_ID}/categories/{CATEGORY_ID}/terms?termId={TERM_ID}
```

**Term Properties:**
- **ID:** Sanitized column name (e.g., `male-35-to-39`)
- **Display Name:** Human-readable label
- **Description:** Detailed definition with context
- **Related Category:** Parent ISO 11179 Object Class

**Error Handling:** Requires `created_categories` dict from step 4. If categories failed to create, this step will raise `KeyError`.

### 6. Link Terms to Data Assets (Cells 19-22)

**Purpose:** Create semantic connections between glossary terms and BigQuery table columns

**Entry Link Concept:** In Dataplex, an "Entry Link" connects a catalog entry (e.g., a BigQuery column) to a glossary term, providing business context for technical assets.

**Process:**

1. **Find BigQuery Table Entry** (Cell 19)
   - Searches `@bigquery` system entry group
   - Filters for the specific table
   - Retrieves entry ID and linked resource

2. **Build Entry Links** (Cells 20-21)
   - For each created term, constructs an entry link
   - **Link Type:** `definition` (term defines the column)
   - **Source:** BigQuery column via path `Schema.{column_name}`
   - **Target:** Glossary term entry

3. **Create Links via API** (Cell 22)
   - Uses project number (not project ID) for entry references
   - Entry path format:
     - Source: `projects/{PROJECT_NUMBER}/locations/us/entryGroups/@bigquery/entries/{ENTRY_ID}` with `aspect_path: Schema.{column_name}`
     - Target: `projects/{PROJECT_NUMBER}/locations/us/entryGroups/@dataplex/entries/{TERM_ENTRY_ID}`
   - Link type resource name: `projects/dataplex-types/locations/global/entryLinkTypes/definition`

**Results:** 53 entry links created, connecting columns to their business term definitions

**Validation:** After creation, navigate to Dataplex Catalog in console and view the BigQuery table entry to see linked glossary terms on each column.

## Data Flow

```
BigQuery Table (blockgroup_2018_5yr)
          ↓
   [Query INFORMATION_SCHEMA]
          ↓
Column Names & Types (155 columns)
          ↓
   [Regex Categorization]
          ↓
Categorized Columns (96 matched, 59 uncategorized)
          ↓
   [Create Glossary Structure]
          ↓
Glossary → Categories (5) → Terms (53, max 20 per category)
          ↓
   [Create Entry Links]
          ↓
BigQuery Columns ←→ Glossary Terms (definition links)
          ↓
   [Enhanced Data Discovery]
```

## Key Functions

### `categorize_column(column_name: str) -> str | None`

Classifies columns into ISO 11179 Object Classes using regex patterns.

**Pattern Examples:**
- Age/Sex: `r"(male|female|_\d+_to_\d+|years)"`
- Race/Ethnicity: `r"(white|black|asian|hispanic|race|ethnic)"`
- Commuting: `r"(commute|transport|travel|vehicle)"`
- Labor Force: `r"(worker|employed|unemployed|labor)"`
- Education: `r"(degree|school|education|graduate)"`

**Returns:** Category ID string or `None`

### `column_to_display_name(column_name: str) -> str`

Converts technical column names to business-friendly labels.

**Transformations:**
- Replace underscores with spaces
- Capitalize each word
- Apply domain-specific formatting (e.g., age ranges)

**Examples:**
```python
column_to_display_name("male_35_to_39")
# "Male Population Age 35 to 39"

column_to_display_name("bachelors_degree")
# "Bachelors Degree"

column_to_display_name("commute_15_to_19_mins")
# "Commute 15 To 19 Minutes"
```

### `generate_term_description(column_name: str, column_type: str, category_id: str) -> str`

Builds comprehensive term descriptions following ISO 11179 definition guidelines.

**Format:**
```
"Represents the {column_name} column ({column_type}) in the census dataset. 
This field is part of the {category_display_name} category."
```

**Example:**
```python
generate_term_description("male_35_to_39", "INT64", "age-and-sex")
# "Represents the male_35_to_39 column (INT64) in the census dataset. 
#  This field is part of the Age and Sex demographic category."
```

## Validation

### Verify Glossary Creation
Navigate to [Dataplex Catalog](https://console.cloud.google.com/dataplex/glossaries) and check for "ACS Demographics Glossary".

### Count Categories and Terms
```python
# Via REST API GET
glossary_url = f"{base_url}/projects/{PROJECT_ID}/locations/{CATALOG_LOCATION}/glossaries/{GLOSSARY_ID}"
response = requests.get(glossary_url, headers=headers)
print(response.json())
# Check categoryCount and termCount
```

### View Terms in Console
1. Navigate to Dataplex > Catalog > Glossaries
2. Click "ACS Demographics Glossary"
3. Browse categories (Age and Sex, Race and Ethnicity, etc.)
4. Verify ~20 terms per category

### Check Entry Links on BigQuery Table
1. Navigate to Dataplex > Catalog > Search
2. Search for `blockgroup_2018_5yr`
3. Open the table entry
4. Click "Schema" tab
5. Verify columns show linked glossary terms (e.g., `male_35_to_39` linked to "Male Population Age 35 to 39")

### Verify Links via API
```python
# List entry links for the BigQuery table
entry_links_url = f"{base_url}/{bigquery_entry_name}/entryLinks"
response = requests.get(entry_links_url, headers=headers)
print(f"Entry links: {len(response.json().get('entryLinks', []))}")  # Should be 53
```

## Troubleshooting

### Issue: "Glossary already exists" (409 Error)
**Cause:** Glossary was created in a previous run.
**Solution:** This is expected on re-runs. The notebook handles this gracefully. To start fresh:
```bash
# Delete glossary via console or gcloud (when supported)
# For now, use a different GLOSSARY_ID or delete via REST API
```

### Issue: Category Creation Returns 400 Errors
**Cause:** API format or version mismatch; the Dataplex glossary API is evolving.
**Solution:**
1. Create categories manually in Dataplex console:
   - Navigate to your glossary
   - Click "Add Category"
   - Enter category ID and display name
2. Modify the `created_categories` dict in cell 14 with manual category resource names
3. Continue with term creation

### Issue: `KeyError` When Creating Terms (Cell 15-18)
**Cause:** Categories weren't created successfully in step 4.
**Solution:**
1. Check `created_categories` dictionary for entries
2. If empty, manually create categories (see above)
3. Populate `created_categories` manually:
   ```python
   created_categories = {
       "age-and-sex": "projects/{PROJECT_ID}/locations/us/glossaries/{GLOSSARY_ID}/categories/age-and-sex",
       # ... add others
   }
   ```

### Issue: "Entry not found" When Creating Links
**Cause:** BigQuery table entry hasn't been indexed by Dataplex yet.
**Solution:**
1. Wait 2-5 minutes after dataset creation
2. Verify entry exists via search in Dataplex console
3. Re-run the entry discovery cell (19)

### Issue: Project Number vs Project ID Confusion
**Cause:** Entry links require project number, not project ID.
**Solution:** The notebook retrieves project number via Cloud Resource Manager API. If this fails:
```python
from google.cloud import resourcemanager_v3
client = resourcemanager_v3.ProjectsClient()
project = client.get_project(name=f"projects/{PROJECT_ID}")
PROJECT_NUMBER = project.name.split("/")[-1]
```

### Issue: Only Analyzing One Table
**Cause:** The notebook focuses on `blockgroup_2018_5yr` as a representative sample.
**Solution:** To extend to all 278 tables:
1. Loop through all tables in `INFORMATION_SCHEMA.COLUMNS`
2. Aggregate unique columns across all tables
3. Generate terms for the union of all columns
4. Link terms to multiple tables

### Issue: Limited Terms Per Category (20)
**Cause:** `MAX_TERMS_PER_CATEGORY = 20` is set for demo purposes.
**Solution:** Increase or remove the limit:
```python
MAX_TERMS_PER_CATEGORY = None  # No limit
# or
MAX_TERMS_PER_CATEGORY = 100  # Higher limit
```

## Next Steps

After completing this notebook, you have:
- ✓ An ISO 11179-compliant Business Glossary in Dataplex
- ✓ 5 semantic categories (Object Classes)
- ✓ 53 business terms with descriptions
- ✓ Columns in `blockgroup_2018_5yr` linked to glossary terms

**Extend the Glossary:**
- Apply terms to all 278 ACS tables (modify the notebook to loop through all tables)
- Add more categories for uncategorized columns
- Create cross-references between related terms

**Continue to:**
- [`04_setup_cloudsql_census.md`](04_setup_cloudsql_census.md) to set up CloudSQL with UK Census data
- [`06_write_firestore_metadata_to_dataplex.md`](06_write_firestore_metadata_to_dataplex.md) (after completing notebook 05) to sync CloudSQL metadata to Dataplex

**Use the Glossary:**
- Search for terms in Dataplex Catalog to find related datasets
- View linked columns when exploring BigQuery tables
- Use terms in data quality rules and policies

## Additional Resources

- [ISO 11179 Metadata Registry Standard](https://www.iso.org/standard/50340.html)
- [Dataplex Glossaries Documentation](https://cloud.google.com/dataplex/docs/glossaries)
- [Dataplex Catalog Overview](https://cloud.google.com/dataplex/docs/catalog-overview)
- [US Census Bureau ACS Data Dictionary](https://www.census.gov/programs-surveys/acs/technical-documentation/code-lists.html)
- [Dataplex REST API Reference](https://cloud.google.com/dataplex/docs/reference/rest)
