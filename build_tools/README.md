# {{cookiecutter.project_name}}
<!-- vscode-markdown-toc -->
* 1. [Setting up Google Cloud project](#SettingupGoogleCloudproject)
	* 1.1. [Prerequisites](#Prerequisites)
	* 1.2. [GCP Organizational policies](#GCPOrganizationalpolicies)
	* 1.3. [GCP Foundation Setup - Terraform](#GCPFoundationSetup-Terraform)
	* 1.4. [Deploying Kubernetes Microservices to GKE](#DeployingKubernetesMicroservicestoGKE)
	* 1.5. [Deploying Microservices to CloudRun](#DeployingMicroservicestoCloudRun)
* 2. [Development](#Development)
* 3. [End-to-End API tests](#End-to-EndAPItests)
* 4. [CI/CD and Test Automation](#CICDandTestAutomation)
	* 4.1. [Github Actions](#GithubActions)
	* 4.2. [Test Github Action workflows locally](#TestGithubActionworkflowslocally)
* 5. [CloudBuild](#CloudBuild)

<!-- vscode-markdown-toc-config
	numbering=true
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->

> This solution skeleton is created from https://github.com/GoogleCloudPlatform/solutions-template

Please contact {{cookiecutter.admin_email}} for any questions.

> **_New Developers:_** Consult the [development guide](./DEVELOPMENT.md) for setup and contribution instructions

##  1. <a name='SettingupGoogleCloudproject'></a>Setting up Google Cloud project

This guide will detail how to set up your new solutions template project. See the [development guide](./DEVELOPMENT.md) for how to contribute to the project.

Project Requirements:

| Tool  | Current Version  | Documentation site |
|---|---|---|
| Skaffold   | v2.x    | https://skaffold.dev/ |
| Kustomize  | v4.3.1  | https://kustomize.io/ |
| gcloud CLI | Latest  | https://cloud.google.com/sdk/docs/install |

###  1.1. <a name='Prerequisites'></a>Prerequisites

Set up environmental variables
```
export PROJECT_ID={{cookiecutter.project_id}}
export ADMIN_EMAIL={{cookiecutter.admin_email}}
export REGION={{cookiecutter.gcp_region}}
export API_DOMAIN={{cookiecutter.api_domain}}
export BASE_DIR=$(pwd)
```

Login to Google Cloud (Optional in Cloud Shell)
```
gcloud auth application-default login
gcloud auth application-default set-quota-project $PROJECT_ID
gcloud config set project $PROJECT_ID
```
- NOTE: you will need to run ```gcloud auth application-default login``` instead of ```gcloud auth login``` if you have multiple projects in the gcloud config.

###  1.2. <a name='GCPOrganizationalpolicies'></a>GCP Organizational policies

Optionally, you may need to update Organization policies for CI/CD test automation.

Run the following commands to update Organization policies:
```
export ORGANIZATION_ID=$(gcloud organizations list --format="value(name)")
gcloud resource-manager org-policies disable-enforce constraints/compute.requireOsLogin --organization=$ORGANIZATION_ID
gcloud resource-manager org-policies delete constraints/compute.vmExternalIpAccess --organization=$ORGANIZATION_ID
gcloud resource-manager org-policies delete constraints/iam.allowedPolicyMemberDomains --organization=$ORGANIZATION_ID
```

Or, change the following Organization policy constraints in [GCP Console](https://console.cloud.google.com/iam-admin/orgpolicies)
- constraints/compute.requireOsLogin - Enforced Off
- constraints/compute.vmExternalIpAccess - Allow All

###  1.3. <a name='GCPFoundationSetup-Terraform'></a>GCP Foundation Setup - Terraform

Set up Terraform environment variables and GCS bucket for state file.
If the new project is just created recently, you may need to wait for 1-2 minutes
before running the Terraform command.

```
export TF_VAR_project_id=$PROJECT_ID
export TF_VAR_api_domain=$API_DOMAIN
export TF_VAR_web_app_domain=$API_DOMAIN
export TF_VAR_admin_email=$ADMIN_EMAIL
export TF_BUCKET_NAME="${PROJECT_ID}-tfstate"
export TF_BUCKET_LOCATION="us"

# Grant Storage admin to the current user IAM.
export CURRENT_USER=$(gcloud config list account --format "value(core.account)")
gcloud projects add-iam-policy-binding $PROJECT_ID --member="user:$CURRENT_USER" --role='roles/storage.admin'

# Create Terraform Statefile in GCS bucket.
bash setup/setup_terraform.sh
```

Run Terraform apply

```
# Init Terraform
cd terraform/environments/dev
terraform init -backend-config=bucket=$TF_BUCKET_NAME

# Enabling GCP services.
terraform apply -target=module.project_services -target=module.service_accounts -auto-approve

# If using GKE, create GKE cluster first. (This will take around 10 mins averagely)
terraform apply -target=module.vpc_network -target=module.gke -auto-approve

# Run the rest of Terraform
terraform apply -auto-approve
```

###  1.4. <a name='DeployingKubernetesMicroservicestoGKE'></a>Deploying Kubernetes Microservices to GKE

Install required packages:

- For MacOS:
  ```
  brew install --cask skaffold kustomize google-cloud-sdk
  ```

- For Windows:
  ```
  choco install -y skaffold kustomize gcloudsdk
  ```

- For Linux/Ubuntu:
  ```
  curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64 && \
  sudo install skaffold /usr/local/bin/
  ```

* Make sure to use __skaffold 2.0.4__ or later for development.

Build all microservices (including web app) and deploy to the cluster:
```
cd $BASE_DIR
export CLUSTER_NAME=main-cluster
gcloud container clusters get-credentials $CLUSTER_NAME --region $REGION --project $PROJECT_ID
skaffold run -p dev --default-repo=gcr.io/$PROJECT_ID
```

Test with API endpoint:
```
export API_DOMAIN=$(kubectl describe ingress | grep Address | awk '{print $2}')
echo "http://${API_DOMAIN}/sample_service/docs"
```

###  1.5. <a name='DeployingMicroservicestoCloudRun'></a>Deploying Microservices to CloudRun

Build common image
```
cd common
gcloud builds submit --config=cloudbuild.yaml --substitutions=\
_PROJECT_ID="$PROJECT_ID",\
_REGION="$REGION",\
_REPOSITORY="cloudrun",\
_IMAGE="common"
```

Set up endpoint permission:
```
export SERVICE_NAME=sample-service
gcloud run services add-iam-policy-binding $SERVICE_NAME \
--region="$REGION" \
--member="allUsers" \
--role="roles/run.invoker"
```

Build service image
```
gcloud builds submit --config=cloudbuild.yaml --substitutions=\
_CLOUD_RUN_SERVICE_NAME=$SERVICE_NAME,\
_PROJECT_ID="$PROJECT_ID",\
_REGION="$REGION",\
_REPOSITORY="cloudrun",\
_IMAGE="cloudrun-sample",\
_SERVICE_ACCOUNT="deployment-dev@$PROJECT_ID.iam.gserviceaccount.com",\
_ALLOW_UNAUTHENTICATED_FLAG="--allow-unauthenticated"
```

Manually deploy a microservice to CloudRun with public endpoint:
```
gcloud run services add-iam-policy-binding $SERVICE_NAME \
--region="$REGION" \
--member="allUsers" \
--role="roles/run.invoker"
```

##  2. <a name='Development'></a>Development


##  3. <a name='End-to-EndAPItests'></a>End-to-End API tests

TBD

##  4. <a name='CICDandTestAutomation'></a>CI/CD and Test Automation

###  4.1. <a name='GithubActions'></a>Github Actions

###  4.2. <a name='TestGithubActionworkflowslocally'></a>Test Github Action workflows locally

- Install Docker desktop: https://www.docker.com/products/docker-desktop/
- Install [Act](https://github.com/nektos/act)
  ```
  # Mac
  brew install act

  # Windows
  choco install act-cli
  ```

- Run a specific Workflow
  ```
  act --workflows .github/workflows/e2e_gke_api_test.yaml
  ```

##  5. <a name='CloudBuild'></a>CloudBuild

TBD

# Development Process & Best Practices

See the [developer guide](./DEVELOPMENT.md) for detailed development workflow