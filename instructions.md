# GCP Demo Repository - Getting Started Guide

Welcome to the GCP demo repository! This collection of Jupyter notebooks demonstrates various Google Cloud Platform services and patterns. Each demo is designed to be educational, easy to follow, and executable in 15-25 minutes.

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
Most demos use minimal resources and should cost less than $5 to run. However:
- Always complete the cleanup section to avoid ongoing charges
- Monitor your billing in the [GCP Console](https://console.cloud.google.com/billing)
- Set up [budget alerts](https://cloud.google.com/billing/docs/how-to/budgets) for your project

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

### Step-by-Step Process

1. **Choose a demo** from the Available Demos section below
2. **Open the notebook** in Jupyter or Google Colab
3. **Read the introduction** to understand what the demo does
4. **Update configuration variables** (usually in the second cell):
   - `PROJECT_ID` - Your GCP project ID
   - `REGION` - Your preferred GCP region (e.g., "us-central1")
   - Other demo-specific variables
5. **Run cells in order** from top to bottom
6. **Complete the cleanup section** to delete resources and avoid charges

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
- `roles/dataflow.admin` - For Dataflow demos
- `roles/storage.admin` - For Cloud Storage access
- `roles/dataplex.admin` - For Dataplex and lineage demos

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
- This is normal for long operations (creating clusters, deploying pipelines)
- Check the notebook for time estimates
- Monitor progress in the [GCP Console](https://console.cloud.google.com)
- Don't interrupt unless it exceeds the expected time significantly

### Cleanup Issues

**Problem:** Resources won't delete or show errors

**Solution:**
- Some resources must be deleted in a specific order
- Wait a few minutes and try again (eventual consistency)
- Delete manually in the GCP Console if needed
- Check for dependent resources that need deletion first

## Available Demos

### Streaming Pipeline Lineage Demo
**File:** `streaming_lineage_demo.ipynb`  
**Time:** 15-20 minutes  
**Services:** Pub/Sub, Dataflow, BigQuery, Dataplex  
**Difficulty:** Intermediate

**Description:** Demonstrates automatic data lineage capture for streaming pipelines. Ingests real-time NYC taxi data from a public Pub/Sub topic, processes it through Dataflow, stores it in BigQuery, and shows how Dataplex automatically captures the lineage without any manual API calls.

**What You'll Learn:**
- Automatic vs. manual lineage recording
- Streaming pipeline patterns in GCP
- Dataflow template usage
- Real-time data processing with Pub/Sub
- BigQuery for streaming analytics

**Prerequisites:**
- Basic understanding of streaming concepts
- Familiarity with BigQuery
- No prior Dataflow or Dataplex experience needed

---

_More demos will be added to this repository over time. Each follows the same structure and conventions for consistency._

## Best Practices

### Before Running a Demo
- [ ] Read the entire notebook introduction first
- [ ] Verify you have all prerequisites
- [ ] Ensure required APIs are enabled
- [ ] Check that you have necessary IAM permissions
- [ ] Update all configuration variables

### During the Demo
- [ ] Execute cells in order (top to bottom)
- [ ] Read explanatory markdown cells
- [ ] Wait for cells to complete before proceeding
- [ ] Review output messages and checkpoint confirmations
- [ ] Monitor the GCP Console for resource creation

### After the Demo
- [ ] Complete the cleanup section
- [ ] Verify resources are deleted in the GCP Console
- [ ] Review billing to understand costs incurred
- [ ] Explore "next steps" suggestions if interested

## Getting Help

### Documentation Resources
- [Google Cloud Documentation](https://cloud.google.com/docs)
- [Python Client Libraries](https://cloud.google.com/python/docs/reference)
- [GCP Samples on GitHub](https://github.com/GoogleCloudPlatform/python-docs-samples)

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

### Extend the Demos
- Modify configuration to use your own data
- Combine multiple demos into a complete pipeline
- Add additional processing or transformation steps
- Integrate with other GCP services

### Production Considerations
After learning from these demos, consider these topics for production deployments:
- **Security**: IAM best practices, VPC configuration, encryption
- **Monitoring**: Cloud Logging, Cloud Monitoring, alerting
- **Automation**: Infrastructure as Code (Terraform), CI/CD
- **Cost Optimization**: Committed use discounts, autoscaling, resource scheduling
- **High Availability**: Multi-region deployments, disaster recovery
- **Data Governance**: Data classification, compliance, audit logging

### Learning Resources
- [Google Cloud Skills Boost](https://www.cloudskillsboost.google/) - Hands-on labs
- [Google Cloud Architecture Center](https://cloud.google.com/architecture) - Best practices
- [Coursera Google Cloud Courses](https://www.coursera.org/googlecloud)
- [YouTube - Google Cloud Tech](https://www.youtube.com/user/googlecloudplatform)

---

**Happy Learning! 🎓**

If you encounter any issues not covered in this guide, please check the specific demo's troubleshooting section or open an issue.
