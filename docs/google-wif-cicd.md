# ☁️ Google Cloud CI/CD Setup via Workload Identity Federation (GitHub → GCP)

This document describes how to set up **Workload Identity Federation (WIF)** between GitHub Actions and Google Cloud
for secure, keyless CI/CD deployments across KRV AI repositories.

---

## 🧭 Overview

Instead of using long-lived service account keys, we configure **GitHub OIDC** to issue short-lived credentials
that impersonate a GCP service account (`github-cicd@PROJECT_ID.iam.gserviceaccount.com`) during builds and deployments.

---

## Prequisites

Authenticate via gcloud + locate project for build + deployment.

```bash
gcloud auth login
gcloud projects list
```

This document serves as the setup template for the krv official website - please change `$PROJECT_ID` accordingly.

```bash
PROJECT_ID=krv-webbie
gcloud config set project $PROJECT_ID
```

> 📝 **Save for GitHub setup:** Note your `PROJECT_ID` value - you'll need this as `GCP_PROJECT_ID` in your GitHub workflow.

## ⚙️ Step 1 — Enable APIs

```bash
gcloud services enable \
  iam.googleapis.com \
  iamcredentials.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  run.googleapis.com \
  secretmanager.googleapis.com
```

**What each API does in CI/CD web deployment:**

- `iam.googleapis.com` — creates the `github-cicd` service account that GitHub Actions will impersonate to deploy
- `iamcredentials.googleapis.com` — allows GitHub Actions to get temporary credentials without storing permanent keys
- `cloudbuild.googleapis.com` — builds your web app into a container image from source code during CI
- `artifactregistry.googleapis.com` — stores the built container images that will be deployed to Cloud Run
- `run.googleapis.com` — deploys your containerized web app and manages traffic/scaling
- `secretmanager.googleapis.com` — stores environment variables, API keys, and database credentials your web app needs

---

## 👤 Step 2 — Create the CI/CD Service Account

```bash
PROJECT_ID="krv-webbie"
gcloud iam service-accounts create github-cicd \
  --display-name="GitHub CI/CD Service Account"
```

Grant project-level roles:

```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:github-cicd@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/run.admin"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:github-cicd@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:github-cicd@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:github-cicd@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/cloudbuild.builds.editor"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:github-cicd@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/serviceusage.serviceUsageConsumer"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:github-cicd@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/logging.logWriter"
```

**Why these roles are needed:** The `github-cicd` service account needs to perform the complete web deployment workflow. `run.admin` allows it to create and update Cloud Run services where your web app will run. `iam.serviceAccountUser` lets Cloud Run assume the default compute service account to access other GCP resources. `artifactregistry.writer` enables pushing the built container images to storage. `cloudbuild.builds.editor` allows triggering builds that convert your source code into deployable containers. `serviceusage.serviceUsageConsumer` ensures the service account can check API quotas and usage limits during deployment operations. `logging.logWriter` permits writing build logs, deployment logs, and application logs to Cloud Logging for monitoring and debugging.

---

## 🔐 Step 3 — Create Workload Identity Pool and Provider

```bash
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")

# Set your GitHub organization and repository
GITHUB_ORG="Krv-Analytics"
```

> 📝 **Save for GitHub setup:** The `PROJECT_NUMBER` from the first command will be needed as a GitHub secret later.

gcloud iam workload-identity-pools create github-pool \
 --project=$PROJECT_ID \
 --location="global" \
 --display-name="GitHub Pool"

gcloud iam workload-identity-pools providers create-oidc github-provider \
 --project=$PROJECT_ID \
  --location="global" \
  --workload-identity-pool="github-pool" \
  --display-name="GitHub Provider" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --attribute-condition="attribute.repository.startsWith('${GITHUB_ORG}/')"

````

**Why we create the workload identity pool and provider:** This establishes the secure bridge between GitHub Actions and Google Cloud. The pool acts as a trust boundary that only accepts tokens from GitHub's OIDC provider. The provider validates that incoming tokens are legitimate GitHub Actions tokens and maps GitHub repository information to Google Cloud attributes. The attribute condition ensures only repositories from your organization can use this setup, preventing unauthorized access from other GitHub repos. This replaces the need for storing long-lived service account keys in GitHub secrets.

---

## 🔑 Step 4 — Allow GitHub Repos to Impersonate the SA

```bash
# Set the specific repository that can deploy
GITHUB_REPO="krv-web"

gcloud iam service-accounts add-iam-policy-binding \
  github-cicd@$PROJECT_ID.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/attribute.repository/${GITHUB_ORG}/${GITHUB_REPO}"
````

> Repeat this for each repository (e.g., change `GITHUB_REPO` to `sentinel-ui`, `compass-ui`, etc.).

**Why this step is required:** This grants the specific GitHub repository permission to impersonate the `github-cicd` service account. Without this binding, GitHub Actions from your repository would be able to authenticate through the workload identity pool but wouldn't have permission to actually assume the service account's identity and deploy resources. Each repository needs its own binding, creating a precise security model where only authorized repos can deploy to your GCP project.

---

## 🪣 Step 5 — Ensure Cloud Build bucket access

```bash
BUCKET="${PROJECT_ID}_cloudbuild"
gcloud storage buckets create gs://$BUCKET --project=$PROJECT_ID --location=us-central1

gcloud storage buckets add-iam-policy-binding gs://$BUCKET \
  --member="serviceAccount:github-cicd@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.admin"
```

**Why Cloud Build bucket access is needed:** Cloud Build requires a storage bucket to store logs, and intermediate files during the container build process. Without proper access to this bucket, your GitHub Actions builds will fail with "forbidden from accessing bucket" errors. The `github-cicd` service account needs admin access to create, read, and write build artifacts to this bucket throughout the CI/CD pipeline.

---

## 📦 Step 6 — Create Artifact Registry Repository

Create a repository to store your container images:

```bash
REGION="us-central1"
REPO_NAME="web-apps"

gcloud artifacts repositories create $REPO_NAME \
  --repository-format=docker \
  --location=$REGION \
  --project=$PROJECT_ID
```

> 📝 **Save for GitHub setup:** Note your `REGION` and `REPO_NAME` values - you'll use `REGION` as the `region` input and `REPO_NAME` as the `repo` input in your GitHub workflow.

**Why Artifact Registry is needed:** This creates a secure, private Docker registry to store your built container images. GitHub Actions will push images here, and Cloud Run will pull from here for deployments. This keeps your container images within your GCP project and provides vulnerability scanning and access controls.

---

## 🧰 Step 7 — Verify

**To fully test the WIF setup:** You need to create a GitHub Actions workflow in your repository that uses the workload identity pool. The WIF can only be tested from within GitHub Actions, not locally. If this verification step runs without `forbidden from accessing bucket` errors, your GCP permissions are correct and the WIF should work when triggered from GitHub.

---

## 📋 Summary: Values for GitHub Workflow Setup

After completing this Google Cloud setup, you'll need these values for your GitHub workflow:

| GitHub Workflow Input     | Value from this setup                                 | Example        |
| ------------------------- | ----------------------------------------------------- | -------------- |
| `GCP_PROJECT_ID` (secret) | Your `PROJECT_ID` from Prerequisites                  | `krv-webbie`   |
| `PROJECT_NUMBER` (secret) | Output from `gcloud projects describe` in Step 3      | `106793116153` |
| `region` (input)          | Your `REGION` from Step 6                             | `us-central1`  |
| `repo` (input)            | Your `REPO_NAME` from Step 6                          | `web-apps`     |
| `service` (input)         | **You choose this** - name for your Cloud Run service | `krv-web`      |

**Next step:** Go to [GitHub WIF setup](./github-wif-cicd.md) to create your deployment workflow using these values.

---

✅ **You are now ready to deploy via GitHub Actions without keys. See [here](./github-wif-cicd.md)**
