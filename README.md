# GCP Dataplex Universal Catalog Demo

A hands-on demonstration of Google Cloud Dataplex capabilities using real census data. This repository showcases metadata management, data governance, business glossaries, and multi-source data integration across BigQuery, Cloud Storage, and CloudSQL PostgreSQL.

## 🎯 What This Demo Does

This collection of Jupyter notebooks demonstrates:

- **Dataplex Universal Catalog**: Unified metadata management across multiple data sources
- **Custom Aspect Types**: Enriching data assets with structured metadata
- **Business Glossaries**: ISO 11179-compliant terminology management
- **Multi-Source Integration**: Cataloging BigQuery, Cloud Storage, CloudSQL PostgreSQL, and Firestore
- **Automated Metadata Extraction**: Pattern-based metadata enrichment at scale
- **Cross-Platform Governance**: Unified view of data assets across GCP services
- **Firestore as Metadata Store**: Hierarchical metadata storage and retrieval patterns

## 📊 Datasets

- **US Census Bureau ACS** (~278 tables): American Community Survey data (2007-2018)
- **UK Census 2021 TS001** (232K rows): Usual residents across 7 geographic levels

## 🚀 Quick Start

### Prerequisites

- GCP project with billing enabled
- Python 3.9+ and Jupyter notebooks
- gcloud CLI installed and configured
- Required IAM roles:
  - `roles/bigquery.admin`
  - `roles/storage.admin`
  - `roles/dataplex.admin`
  - `roles/dataplex.catalogAdmin`
  - `roles/cloudsql.admin`
  - `roles/datastore.owner`
  - `roles/firestore.viewer`

### Setup Authentication

```bash
# Install and initialize gcloud
gcloud init

# Authenticate
gcloud auth login
gcloud auth application-default login

# Set your project
gcloud config set project YOUR_PROJECT_ID
```

### Run the Demo

```bash
# Clone the repository
git clone https://github.com/your-org/gcp-demo-dataplex.git
cd gcp-demo-dataplex

# Create virtual environment
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install Jupyter
pip install jupyter ipykernel

# Launch Jupyter
jupyter notebook
```

### Recommended Workflow

Run the notebooks in this order:

1. **`setup/01_config_and_data_setup.ipynb`** (10-30 min)
   - Copy US Census data to BigQuery
   - Create custom Dataplex aspect types
   - Apply metadata enrichment

2. **`setup/02_upload_census_to_gcs.ipynb`** (10-15 min)
   - Upload UK Census 2021 to Cloud Storage
   - Load data into BigQuery

3. **`setup/03_apply_glossary_terms.ipynb`** (15-25 min)
   - Create business glossary with ISO 11179 principles
   - Generate and link business terms to data assets

4. **`setup/04_setup_cloudsql_census.ipynb`** (20-30 min)
   - Create CloudSQL PostgreSQL instance
   - Load census data
   - Enable Dataplex integration

5. **`setup/05_setup_firestore.ipynb`** (10-15 min)
   - Create Firestore Native Mode database
   - Create a copy of the CloudSQL census table
   - Extract and store table metadata in Firestore

6. **`setup/06_write_firestore_metadata_to_dataplex.ipynb`** (5-8 min)
   - Read metadata from Firestore
   - Create custom Dataplex entry group and entry type
   - Register CloudSQL table in Dataplex Catalog with governance aspect

7. **Explore the Catalog** in [GCP Console](https://console.cloud.google.com/dataplex/search)

8. **`setup/99_cleanup_all_resources.ipynb`** (15-20 min) ⚠️ **REQUIRED**
   - Delete all resources to avoid ongoing charges

**Total Time:** 55-110 minutes for complete setup

## 📁 Repository Structure

```
gcp-demo-dataplex/
├── README.md                                        # This file
├── instructions.md                                  # Detailed getting started guide
├── setup/
│   ├── SETUP_NOTEBOOKS.md                          # Detailed notebook documentation
│   ├── 01_config_and_data_setup.ipynb              # Core BigQuery + Dataplex setup
│   ├── 02_upload_census_to_gcs.ipynb               # UK Census to GCS & BigQuery
│   ├── 03_apply_glossary_terms.ipynb               # Business glossary creation
│   ├── 04_setup_cloudsql_census.ipynb              # CloudSQL + Dataplex integration
│   ├── 05_setup_firestore.ipynb                    # Firestore setup + CloudSQL metadata storage
│   ├── 06_write_firestore_metadata_to_dataplex.ipynb  # Write Firestore metadata to Dataplex Catalog
│   └── 99_cleanup_all_resources.ipynb              # Complete resource cleanup
└── source_data/
    └── census2021-ts001/                           # UK Census 2021 data files
        ├── census2021-ts001-*.csv                  # CSV files (7 geographic levels)
        └── metadata/                               # Dataset documentation
```

## 💰 Cost Considerations

**Estimated Cost:** $10-20 if cleaned up within 24 hours

The demo creates:
- BigQuery dataset (~278 tables)
- CloudSQL PostgreSQL instance (db-f1-micro)
- Cloud Storage bucket
- Firestore Native Mode database (free tier)
- Dataplex catalog resources (aspect types, business glossary, custom entry group/type)

⚠️ **Important:** CloudSQL instances incur charges even when idle. Always run the cleanup notebook when finished!

## 🔗 Key Console Links

After running the setup, explore these resources:

- [Dataplex Universal Catalog Search](https://console.cloud.google.com/dataplex/search)
- [Business Glossaries](https://console.cloud.google.com/dataplex/dp-glossaries)
- [Aspect Types](https://console.cloud.google.com/dataplex/govern/aspect-types)
- [BigQuery Datasets](https://console.cloud.google.com/bigquery)
- [CloudSQL Instances](https://console.cloud.google.com/sql/instances)
- [Cloud Storage Buckets](https://console.cloud.google.com/storage/browser)
- [Firestore Database](https://console.cloud.google.com/firestore/data)

## 📚 What You'll Learn

### Dataplex Concepts
- Universal Catalog for multi-source metadata
- Custom aspect types for domain-specific metadata
- Business glossaries and terminology management
- Automatic vs. manual metadata enrichment
- Cross-platform data governance

### Technical Skills
- BigQuery dataset management
- Cloud Storage operations
- CloudSQL PostgreSQL setup and integration
- Firestore Native Mode database and document modeling
- Python API clients for GCP services
- Metadata extraction and pattern matching

### Best Practices
- ISO 11179 data element naming standards
- Metadata modeling and enrichment strategies
- Multi-source data governance patterns
- Cost-effective resource management

## 🛠️ Troubleshooting

### Common Issues

**Authentication Error:**
```bash
gcloud auth application-default login
```

**API Not Enabled:**
```bash
gcloud services enable dataplex.googleapis.com
gcloud services enable bigquery.googleapis.com
gcloud services enable storage.googleapis.com
gcloud services enable sqladmin.googleapis.com
gcloud services enable firestore.googleapis.com
```

**Permission Denied:**
```bash
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="user:YOUR_EMAIL" \
  --role="roles/dataplex.admin"
```

For detailed troubleshooting, see [instructions.md](instructions.md#common-issues-and-troubleshooting).

## 📖 Documentation

- **[Getting Started Guide](instructions.md)** - Comprehensive setup and usage instructions
- **[Setup Notebooks Documentation](setup/SETUP_NOTEBOOKS.md)** - Detailed notebook descriptions
- [Dataplex Documentation](https://cloud.google.com/dataplex/docs)
- [Dataplex Universal Catalog](https://cloud.google.com/dataplex/docs/catalog)
- [Firestore Documentation](https://cloud.google.com/firestore/docs)
- [ISO 11179 Standard](https://www.iso.org/standard/50340.html)

## 🤝 Contributing

Found an issue or have a suggestion?

1. Check existing [GitHub Issues](https://github.com/your-org/gcp-demo-dataplex/issues)
2. Create a new issue with:
   - Notebook name
   - Error message or unexpected behavior
   - Steps to reproduce
   - Your environment (OS, Python version, gcloud version)

## ⚠️ Important Reminders

- **Run notebooks in order** - They build on each other
- **Complete the cleanup notebook** - `99_cleanup_all_resources.ipynb` avoids unnecessary charges
- **Monitor CloudSQL costs** - Most expensive resource in the demo
- **Wait for long operations** - CloudSQL creation takes 10-15 minutes
- **Check billing alerts** - Set up budget alerts in your project
- **Firestore is free tier** - Stays within free limits for this demo

## 📝 License

This demo repository is provided as-is for educational purposes.

Census data sources:
- US Census Bureau ACS: Public domain
- UK Census 2021 (ONS): [Open Government Licence v3.0](https://www.nationalarchives.gov.uk/doc/open-government-licence/version/3/)

---

**Happy Learning!** 🎓

For questions or issues, see [instructions.md](instructions.md) or open an issue.
