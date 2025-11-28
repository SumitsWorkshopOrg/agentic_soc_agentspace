# Agentic SOC Deployment Guide

This guide provides a streamlined workflow to deploy a custom Agentic SOC using Google Gemini Enterprise and Google SecOps.

## 📋 Prerequisites
* **Google Cloud Project** (Owner permissions)
* **Google SecOps** instance
* **Google SecOps SOAR API Key**
* **Google Threat Intelligence (GTI) API Key**

---

## 🚀 Part 1: Initial Setup

### 1. Configure Environment
[cite_start]Open Cloud Shell and run the following "all-in-one" block to enable APIs and setup the repository [cite: 12-27].

```bash
# 1. Enable APIs
gcloud services enable aiplatform.googleapis.com \
    storage.googleapis.com \
    cloudbuild.googleapis.com \
    compute.googleapis.com \
    discoveryengine.googleapis.com \
    securitycenter.googleapis.com \
    iam.googleapis.com

# 2. Clone Repository & Setup Python
# (Replace with your actual Project ID if not set)
export PROJECT_ID=$(gcloud config get-value project)
git clone --recurse-submodules [https://github.com/SumitsWorkshop0rg/agentic_soc_agentspace.git](https://github.com/SumitsWorkshop0rg/agentic_soc_agentspace.git)
cd agentic_soc_agentspace

cp .env.example .env
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

---

## 🏗️ Part 2: Infrastructure Deployment

### 1. Create Knowledge Base (RAG)
[cite_start]Create the RAG corpus for your security runbooks [cite: 28-33].

**⚠️ IMPORTANT:** Copy the `RAG_CORPUS_ID` from the output (e.g., `projects/.../ragCorpora/...`). You will need this for the `.env` file.

```bash
make rag-create NAME="Security Runbooks"
```

### 2. Create Staging Bucket & Service Account
[cite_start]Run this script to automatically create your bucket, service account, and download the required JSON key [cite: 35-61].

```bash
# Set Variables
export LOCATION="us-central1"
export BUCKET_NAME="gs://${PROJECT_ID}-staging-bucket"
export SA_NAME="chronicle-svc-acct"
export SA_EMAIL="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
export KEY_FILE="$(pwd)/chronicle-key.json"

# Disable Policy & Create Resources
gcloud resource-manager org-policies disable-enforce iam.disableServiceAccountKeyCreation --project=$PROJECT_ID
gcloud storage buckets create $BUCKET_NAME --location=$LOCATION
gcloud iam service-accounts create $SA_NAME --display-name="Chronicle Service Account for Agentic SOC"

# Bind Permissions & Generate Key
gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:${SA_EMAIL}" --role="roles/chronicle.editor" --condition=None
gcloud iam service-accounts keys create $KEY_FILE --iam-account=$SA_EMAIL

echo "------------------------------------------------"
echo "✅ Setup Complete"
echo "Bucket: $BUCKET_NAME"
echo "Key Path: $KEY_FILE"
echo "------------------------------------------------"
```

---

## ⚙️ Part 3: Configuration

### 1. Create Gemini App (CLI Method)
Instead of using the GUI, run this command to create the app and get your ID instantly.

```bash
python manage.py agentspace create-app --name "Google Security Agent" --type SOLUTION_TYPE_CHAT
```
*Copy the `AGENTSPACE_APP_ID` from the output.*

### 2. Update Configuration
[cite_start]Open your `.env` file (`nano .env`) and update the following values [cite: 229-244]:

* `GCP_STAGING_BUCKET`: (From Part 2 output)
* `CHRONICLE_SERVICE_ACCOUNT_PATH`: (From Part 2 output)
* `RAG_CORPUS_ID`: (From Part 2, Step 1)
* `AGENTSPACE_APP_ID`: (From Part 3, Step 1)
* `CHRONICLE_CUSTOMER_ID`: Your SecOps UUID
* `SOAR_API_KEY`: Your Siemplify Key
* `GTI_API_KEY`: Your VirusTotal Key

### 3. Grant IAM Permissions
[cite_start]Grant the AI agents access to your data [cite: 247-264]:

```bash
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
export VERTEX_SA_EMAIL="service-${PROJECT_NUMBER}@gcp-sa-aiplatform.iam.gserviceaccount.com"

# Grant Reasoning Engine Access
gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-aiplatform-re.iam.gserviceaccount.com" --role="roles/aiplatform.user"

# Grant Discovery Engine Access
gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-discoveryengine.iam.gserviceaccount.com" --role="roles/aiplatform.user"
gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-discoveryengine.iam.gserviceaccount.com" --role="roles/aiplatform.viewer"

# Grant Bucket Access
gcloud storage buckets add-iam-policy-binding $BUCKET_NAME --member="serviceAccount:${VERTEX_SA_EMAIL}" --role="roles/storage.objectViewer"
```

---

## 🚀 Part 4: Deployment

### 1. Deploy Agent Engine (Automated)
[cite_start]Run this command to deploy the agent and **automatically update your .env file** with the result [cite: 267-272].

```bash
make agent-engine-deploy 2>&1 | tee deploy.log

# Auto-update .env with the new Resource Name
RESOURCE_NAME=$(grep -o "projects/.*/reasoningEngines/[0-9]*" deploy.log | tail -n 1)
AGENT_ID=$(basename "$RESOURCE_NAME")

if [ ! -z "$RESOURCE_NAME" ]; then
    sed -i '/AGENT_ENGINE_RESOURCE_NAME/d' .env
    sed -i '/AGENT_ENGINE_ID/d' .env
    echo "AGENT_ENGINE_RESOURCE_NAME=$RESOURCE_NAME" >> .env
    echo "AGENT_ENGINE_ID=$AGENT_ID" >> .env
    echo "✅ .env updated successfully!"
else
    echo "❌ Could not find Resource Name. Check logs."
fi
```

### 2. Register Agent
Finalize the link between the Agent Engine and Gemini Enterprise:

```bash
make agentspace-register
```

---

## 🏁 Part 5: Finalize & Test

1.  Open **Gemini Enterprise** in the Google Cloud Console.
2.  Go to **Settings** > **Set up identity** > **Use Google Identity**.
3.  Navigate to the Chat Interface.
4.  In the "From your organization" section, pin the **Google Security Agent**.
5.  **Test Query:** *"Find logon events from the last 3 days"*.
