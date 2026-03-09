# GCP Dataplex Demo Repository - Getting Started Guide

Welcome to the GCP Dataplex demo repository! This collection of Jupyter notebooks demonstrates Google Cloud Platform's Dataplex Universal Catalog capabilities using real Census data. The demos showcase metadata management, data governance, business glossaries, and multi-source data integration with BigQuery, Cloud Storage, and CloudSQL PostgreSQL.

## Prerequisites

### Required Software
- **Python 3.9 or higher** - [Download Python](https://www.python.org/downloads/)
- **Jupyter** - Install with `pip install jupyter` or use [Google Colab](https://colab.research.google.com)
- **gcloud CLI** - [Installation guide](https://cloud.google.com/sdk/docs/install)

### Google Cloud Platform
- **GCP Project** with billing enabled - [Create a project](https://console.cloud.google.com/projectcreate)
- **Sufficient IAM permissions** - You'll need admin-level access or specific roles for the services used in each demo
- **Understanding of GCP basics** - Familiarity with the Cloud Console and basic GCP concepts

### Cost Considerations
The demo setup creates:
- BigQuery dataset with ~278 tables (US Census data)
- CloudSQL PostgreSQL instance (db-f1-micro)
- Cloud Storage bucket with UK Census 2021 data
- Dataplex catalog resources (aspect types, business glossary)

**Estimated costs**: $10-20 to run the complete demo if cleaned up within 24 hours. However:
- **Always complete the cleanup notebook** to avoid ongoing charges (especially CloudSQL)
- Monitor your billing in the [GCP Console](https://console.cloud.google.com/billing)
- Set up [budget alerts](https://cloud.google.com/billing/docs/how-to/budgets) for your project
- CloudSQL instances incur charges even when idle - clean up promptly

## Initial Setup

### 1. Install and Configure gcloud CLI

```bash
# Install gcloud CLI (if not already installed)
# Follow instructions at: https://cloud.google.com/sdk/docs/install

# Initialize gcloud
gcloud init

# Authenticate with your Google account
gcloud auth login

# Set up application default credentials (required for notebooks)
gcloud auth application-default login

# Set your default project
gcloud config set project YOUR_PROJECT_ID

# Verify setup
gcloud config list
gcloud auth list
```

### 2. Clone or Download This Repository

```bash
git clone https://github.com/your-org/gcp-demo-dataplex.git
cd gcp-demo-dataplex
```

### 3. Set Up Python Environment (Recommended)

```bash
# Create a virtual environment
python3 -m venv venv

# Activate the environment
# On macOS/Linux:
source venv/bin/activate
# On Windows:
venv\Scripts\activate

# Install Jupyter
pip install jupyter ipykernel

# Launch Jupyter
jupyter notebook
```

## Running a Demo

### Datasets Used in This Demo

This repository uses two census datasets to demonstrate multi-source data governance:

**1. US Census Bureau - American Community Survey (ACS)**
- **Source:** BigQuery public dataset `bigquery-public-data.census_bureau_acs`
- **Tables:** ~278 tables covering 2007-2018
- **Geographic Levels:** Block groups, CBSAs, census tracts, counties, states
- **Survey Periods:** 1-year, 3-year, and 5-year estimates
- **Use Case:** Demonstrates Dataplex aspects and business glossaries at scale

**2. UK Census 2021 - TS001 (Usual Residents)**
- **Source:** UK Office for National Statistics (ONS)
- **Release Date:** 2022-12-13
- **Coverage:** England and Wales
- **Geographic Levels:** 7 levels (Country, Region, UTLA, LTLA, MSOA, LSOA, OA)
- **Total Records:** 232,157 rows
- **Files Location:** `source_data/census2021-ts001/`
- **Use Case:** Demonstrates multi-platform integration (GCS, BigQuery, CloudSQL)

### Step-by-Step Process

1. **Choose a demo** from the Available Demos section below (recommend running in order)
2. **Open the notebook** in Jupyter or Google Colab
3. **Read the introduction** to understand what the demo does
4. **Update configuration variables** (usually in the second cell):
   - `PROJECT_ID` - Your GCP project ID
   - `REGION` - Your preferred GCP region (e.g., "us-central1")
   - `INSTANCE_NAME` - CloudSQL instance name (for setup_cloudsql_census.ipynb)
   - `BUCKET_NAME` - GCS bucket name (for upload_census_to_gcs.ipynb)
   - Other demo-specific variables
5. **Run cells in order** from top to bottom
6. **Wait for long-running operations** (CloudSQL creation takes 10-15 minutes)
7. **Complete the cleanup notebook** when finished to delete all resources and avoid charges

### Configuration Tips

**Finding Your Project ID:**
```bash
gcloud config get-value project
```

**Choosing a Region:**
- Use `us-central1` for general US workloads (Iowa)
- Use `us-east1` for East Coast US (South Carolina)
- Use `europe-west1` for European workloads (Belgium)
- See [all regions](https://cloud.google.com/about/locations)

**Validation:**
Most notebooks include validation cells that check your configuration. Look for checkpoint messages like:
```
✅ Checkpoint passed: Configuration validated
```

## Common Issues and Troubleshooting

### Authentication Issues

**Problem:** `DefaultCredentialsError: Could not automatically determine credentials`

**Solution:**
```bash
gcloud auth application-default login
```

**Problem:** `Permission denied` or `403 Forbidden`

**Solution:** Verify you have the required IAM permissions. Check with:
```bash
gcloud projects get-iam-policy YOUR_PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:user:YOUR_EMAIL"
```

### API Not Enabled

**Problem:** `Service 'servicename.googleapis.com' is not enabled`

**Solution:** Enable the required API:
```bash
gcloud services enable servicename.googleapis.com --project=YOUR_PROJECT_ID
```

Most notebooks include a cell with the exact command to enable all required APIs.

### Permission Errors

**Problem:** `User does not have permission to access resource`

**Solution:** You need specific IAM roles. Each notebook lists required permissions in the prerequisites section. Common roles include:
- `roles/bigquery.admin` - For BigQuery demos
- `roles/storage.admin` - For Cloud Storage access
- `roles/dataplex.admin` - For Dataplex resource management
- `roles/dataplex.catalogAdmin` - For Dataplex catalog and glossary operations
- `roles/cloudsql.admin` - For CloudSQL instance management
- `roles/compute.networkUser` - For CloudSQL network configuration

Grant roles using:
```bash
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="user:YOUR_EMAIL" \
  --role="roles/ROLE_NAME"
```

### Quota and Resource Limits

**Problem:** `Quota exceeded` or `Resource limit reached`

**Solution:**
- Check your quotas in the [GCP Console](https://console.cloud.google.com/iam-admin/quotas)
- Request quota increases if needed
- Try a different region that may have more available resources
- Delete unused resources from previous demos

### Region or Location Issues

**Problem:** Resources must be in the same region but aren't

**Solution:**
- Use consistent region values throughout the notebook
- Some services use `region` (e.g., "us-central1")
- Others use `location` (e.g., "US", "EU", or specific region)
- Check the notebook's configuration section for guidance

### Network or Firewall Issues

**Problem:** `Connection refused` or `Network error`

**Solution:**
- Verify your internet connection
- Check if your organization has firewall rules blocking GCP
- Some demos (like Dataflow) may require VPC configuration
- Check the specific demo's troubleshooting section

### Long-Running Operations

**Problem:** Cell seems stuck with `[*]` indicator

**Solution:**
- This is normal for long operations (CloudSQL creation: 10-15 minutes, BigQuery dataset copy: 10-20 minutes)
- Check the notebook for time estimates
- Monitor progress in the [GCP Console](https://console.cloud.google.com)
- For CloudSQL operations, check: https://console.cloud.google.com/sql/instances
- Don't interrupt unless it exceeds the expected time significantly

### Cleanup Issues

**Problem:** Resources won't delete or show errors

**Solution:**
- Some resources must be deleted in a specific order
- Wait a few minutes and try again (eventual consistency)
- Delete manually in the GCP Console if needed
- Check for dependent resources that need deletion first

## Available Demos

### Overview

This repository contains a complete Dataplex Universal Catalog demo with five notebooks that should be run sequentially:

1. **config_and_data_setup.ipynb** - Core BigQuery and Dataplex setup
2. **upload_census_to_gcs.ipynb** - UK Census data to Cloud Storage and BigQuery
3. **apply_glossary_terms.ipynb** - Business glossary with ISO 11179 principles
4. **setup_cloudsql_census.ipynb** - CloudSQL PostgreSQL with Dataplex integration
5. **cleanup_all_resources.ipynb** - Complete resource cleanup

**Total Time:** 45-90 minutes for complete setup  
**Total Cleanup Time:** 15-20 minutes

---

### 1. Configuration and Data Setup
**File:** `setup/config_and_data_setup.ipynb`  
**Time:** 10-30 minutes  
**Services:** BigQuery, Dataplex Catalog  
**Difficulty:** Beginner

**Description:** Creates the foundation for the Dataplex demo by copying US Census Bureau American Community Survey data (~278 tables) from BigQuery public datasets into your project. Creates custom Dataplex aspect types for metadata enrichment and applies them automatically to all census tables.

**What You'll Learn:**
- BigQuery dataset creation and table copying
- Dataplex aspect types (custom metadata schemas)
- Automatic metadata enrichment at scale
- Pattern-based metadata extraction from table names

**Prerequisites:**
- Required IAM roles:
  - `roles/bigquery.admin`
  - `roles/dataplex.admin`
  - `roles/dataplex.catalogAdmin`

**Key Features:**
- Copies 278 tables (ACS data from 2007-2018)
- Creates 2 custom aspect types:
  - Census Survey Metadata (vintage, estimate period, experimental flag)
  - Public Data Governance (source agency, license, catalog date)
- Automatically enriches all tables with structured metadata

---

### 2. Upload Census to Cloud Storage
**File:** `setup/upload_census_to_gcs.ipynb`  
**Time:** 10-15 minutes  
**Services:** Cloud Storage, BigQuery  
**Difficulty:** Beginner

**Description:** Uploads UK Census 2021 dataset (TS001 - usual residents) to Google Cloud Storage and loads it into BigQuery. Demonstrates multi-source data integration by adding UK census data alongside US census data.

**What You'll Learn:**
- Cloud Storage bucket creation and file uploads
- Loading CSV data from GCS to BigQuery
- Schema definition and column mapping
- Multi-geography data handling (7 geographic levels)

**Prerequisites:**
- Required IAM roles:
  - `roles/storage.admin`
  - `roles/bigquery.admin`
- Census data files in `source_data/census2021-ts001/`

**Dataset Information:**
- **Source:** UK Office for National Statistics (ONS)
- **Release Date:** 2022-12-13
- **Coverage:** England and Wales
- **Geographic Levels:** Country, Region, UTLA, LTLA, MSOA, LSOA, OA
- **Total Rows:** 232,157 across all geographic levels

---

### 3. Apply Glossary Terms
**File:** `setup/apply_glossary_terms.ipynb`  
**Time:** 15-25 minutes  
**Services:** Dataplex Business Glossary, BigQuery  
**Difficulty:** Intermediate

**Description:** Creates a comprehensive business glossary following ISO 11179 data element naming principles. Automatically categorizes census columns, creates human-readable business terms, and links them to BigQuery table columns for enhanced discoverability.

**What You'll Learn:**
- ISO 11179 data governance principles
- Dataplex Business Glossary creation
- Hierarchical category structures (Object Classes)
- Automated term generation and linking
- Business-friendly metadata for technical data

**Prerequisites:**
- Census dataset created (via `config_and_data_setup.ipynb`)
- Required IAM roles:
  - `roles/dataplex.catalogAdmin`
  - `roles/bigquery.admin`

**Key Features:**
- Creates root glossary: "ACS Demographics Glossary"
- 5 ISO 11179 Object Class categories:
  - Age and Sex
  - Race and Ethnicity
  - Commuting Characteristics
  - Labor Force
  - Educational Attainment
- Generates human-readable term names (e.g., "Male Population Age 35 to 39")
- Links terms to BigQuery columns for semantic search

---

### 4. Setup CloudSQL Census Database
**File:** `setup/setup_cloudsql_census.ipynb`  
**Time:** 20-30 minutes (includes 10-15 min provisioning)  
**Services:** CloudSQL PostgreSQL, Dataplex Universal Catalog  
**Difficulty:** Intermediate

**Description:** Creates a CloudSQL PostgreSQL instance, loads UK Census 2021 data, and enables Dataplex Universal Catalog integration. Demonstrates how Dataplex can automatically catalog metadata from relational databases alongside cloud-native data sources.

**What You'll Learn:**
- CloudSQL PostgreSQL instance creation and configuration
- Cloud SQL Python Connector for secure connections
- Loading data from CSV to PostgreSQL
- Dataplex Universal Catalog integration with CloudSQL
- Multi-source metadata management (BigQuery + PostgreSQL)

**Prerequisites:**
- Required IAM roles:
  - `roles/cloudsql.admin`
  - `roles/compute.networkUser`
- Census data files in `source_data/census2021-ts001/`

**Key Features:**
- Creates `db-f1-micro` PostgreSQL 15 instance
- Loads 232,157 rows across 7 geographic levels
- Creates indexed tables for query performance
- Enables automatic Dataplex cataloging (metadata appears in 2-48 hours)
- Demonstrates cross-platform data governance

**Important Notes:**
- CloudSQL instance creation takes 10-15 minutes
- Instance incurs charges even when idle - clean up promptly
- Dataplex metadata sync happens automatically after 2-48 hours

---

### 5. Cleanup All Resources
**File:** `setup/cleanup_all_resources.ipynb`  
**Time:** 15-20 minutes  
**Services:** BigQuery, Dataplex, CloudSQL, Cloud Storage  
**Difficulty:** Beginner

**Description:** Complete cleanup notebook that removes all resources created by the setup notebooks. **Critical for cost management** - CloudSQL instances incur ongoing charges even when idle.

**What It Cleans Up:**
- Removes all Dataplex aspects from BigQuery tables
- Deletes custom aspect types
- Deletes BigQuery datasets and all tables (~278 tables)
- Deletes business glossary, categories, and terms
- Disables Dataplex integration and deletes CloudSQL instance
- Optionally deletes Cloud Storage bucket

**⚠️ Warning:**
- This operation **cannot be undone**
- All data will be **permanently deleted**
- All custom metadata will be removed
- Run this notebook as soon as you finish exploring the demo

**Prerequisites:**
- Same IAM roles as setup notebooks

---

## Best Practices

### Before Running the Demos
- [ ] Read the entire notebook introduction first
- [ ] Verify you have all prerequisites
- [ ] Ensure required APIs are enabled (notebooks include enable commands)
- [ ] Check that you have necessary IAM permissions
- [ ] Update all configuration variables (PROJECT_ID, REGION, etc.)
- [ ] Have `source_data/census2021-ts001/` directory with census files

### During the Demo
- [ ] Execute cells in order (top to bottom)
- [ ] Read explanatory markdown cells
- [ ] Wait for cells to complete before proceeding (especially CloudSQL creation)
- [ ] Review output messages and checkpoint confirmations
- [ ] Monitor the GCP Console for resource creation
- [ ] Note the time estimates for long-running operations

### After the Demo
- [ ] **CRITICAL: Run cleanup_all_resources.ipynb immediately**
- [ ] Verify resources are deleted in the GCP Console
- [ ] Check CloudSQL instances are removed (ongoing charges!)
- [ ] Review billing to understand costs incurred
- [ ] Export any interesting metadata or queries before cleanup

### Recommended Workflow
```bash
# 1. Core setup (required)
jupyter notebook setup/config_and_data_setup.ipynb

# 2. UK Census to GCS (required for CloudSQL demo)
jupyter notebook setup/upload_census_to_gcs.ipynb

# 3. Business glossary (optional but recommended)
jupyter notebook setup/apply_glossary_terms.ipynb

# 4. CloudSQL with Dataplex integration (optional)
jupyter notebook setup/setup_cloudsql_census.ipynb

# 5. Explore Dataplex Universal Catalog in GCP Console
# - Search for assets
# - View metadata and aspects
# - Browse business glossary
# - Check CloudSQL catalog entries (after 2-48 hours)

# 6. Cleanup (REQUIRED - avoid ongoing charges!)
jupyter notebook setup/cleanup_all_resources.ipynb
```

## Getting Help

### Documentation Resources
- [Google Cloud Documentation](https://cloud.google.com/docs)
- [Dataplex Documentation](https://cloud.google.com/dataplex/docs)
- [Dataplex Universal Catalog](https://cloud.google.com/dataplex/docs/catalog)
- [CloudSQL for PostgreSQL](https://cloud.google.com/sql/docs/postgres)
- [BigQuery Documentation](https://cloud.google.com/bigquery/docs)
- [Python Client Libraries](https://cloud.google.com/python/docs/reference)
- [ISO 11179 Metadata Registry](https://www.iso.org/standard/50340.html)

### Cost Management
- [GCP Pricing Calculator](https://cloud.google.com/products/calculator)
- [GCP Free Tier](https://cloud.google.com/free)
- [Understanding Billing](https://cloud.google.com/billing/docs)

### Support Channels
- [Stack Overflow - google-cloud-platform](https://stackoverflow.com/questions/tagged/google-cloud-platform)
- [GCP Community](https://www.googlecloudcommunity.com/)
- [GitHub Issues](https://github.com/your-org/gcp-demo-dataplex/issues) for demo-specific problems

## Contributing

If you find issues or have suggestions:
1. Check existing [GitHub Issues](https://github.com/your-org/gcp-demo-dataplex/issues)
2. Create a new issue with:
   - Demo name
   - Error message or unexpected behavior
   - Steps to reproduce
   - Your environment (OS, Python version, gcloud version)

## Next Steps

### Explore the Demo Environment
- **Dataplex Universal Catalog**: Search for assets across BigQuery and CloudSQL
- **Business Glossary**: Browse terms and see linked data assets
- **Custom Aspects**: View enriched metadata on census tables
- **Cross-Platform Governance**: See how Dataplex unifies metadata from multiple sources

### Extend the Demos
- Add your own datasets to the Dataplex catalog
- Create additional custom aspect types for your domain
- Expand the business glossary to more tables and columns
- Integrate other data sources (Cloud SQL MySQL, Cloud Spanner, etc.)
- Apply data quality rules and policies

### Console Links for Exploration
After setup, explore these resources:
- **BigQuery Datasets**: 
  - US Census: `https://console.cloud.google.com/bigquery?project=<PROJECT_ID>&d=census_bureau_acs`
  - UK Census: `https://console.cloud.google.com/bigquery?project=<PROJECT_ID>&d=census_uk_2021`
- **Cloud Storage Bucket**: `https://console.cloud.google.com/storage/browser/<BUCKET_NAME>?project=<PROJECT_ID>`
- **CloudSQL Instance**: `https://console.cloud.google.com/sql/instances?project=<PROJECT_ID>`
- **Dataplex Catalog Search**: `https://console.cloud.google.com/dataplex/search?project=<PROJECT_ID>`
- **Dataplex Aspect Types**: `https://console.cloud.google.com/dataplex/govern/aspect-types?project=<PROJECT_ID>`
- **Business Glossaries**: `https://console.cloud.google.com/dataplex/dp-glossaries?project=<PROJECT_ID>`

### Production Considerations
After learning from these demos, consider these topics for production Dataplex deployments:
- **Security**: IAM best practices, VPC Service Controls, private service connectivity
- **Data Governance**: Data classification, PII detection, compliance frameworks
- **Metadata Management**: Automated aspect application, metadata versioning, lineage tracking
- **Data Quality**: DQ rules, validation frameworks, monitoring and alerting
- **Multi-Source Integration**: Cataloging diverse data sources (databases, files, APIs)
- **Business Glossaries**: Enterprise-wide terminology standards, approval workflows
- **Cost Optimization**: Resource tagging, lifecycle policies, query optimization
- **Automation**: Infrastructure as Code (Terraform), CI/CD for metadata pipelines
- **High Availability**: Multi-region deployments, disaster recovery for CloudSQL

### Learning Resources
- [Google Cloud Skills Boost](https://www.cloudskillsboost.google/) - Hands-on labs and learning paths
- [Dataplex Fundamentals](https://cloud.google.com/dataplex/docs/introduction) - Official getting started guide
- [Data Catalog Best Practices](https://cloud.google.com/dataplex/docs/best-practices-catalog) - Governance patterns
- [ISO 11179 Information](https://www.iso.org/standard/50340.html) - Metadata registry standard
- [Google Cloud Architecture Center](https://cloud.google.com/architecture) - Reference architectures
- [Coursera Google Cloud Courses](https://www.coursera.org/googlecloud) - Structured learning
- [YouTube - Google Cloud Tech](https://www.youtube.com/user/googlecloudplatform) - Video tutorials

---

**Happy Learning! 🎓**

For detailed information about each setup notebook, see [`setup/SETUP_NOTEBOOKS.md`](setup/SETUP_NOTEBOOKS.md).

If you encounter any issues not covered in this guide, please check the specific notebook's troubleshooting section or open an issue.

**Remember:** Always run `cleanup_all_resources.ipynb` when finished to avoid ongoing charges!
