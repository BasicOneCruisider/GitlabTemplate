# 🚀 GitLab CI/CD Pipeline for Azure Web App Deployment

## 🛡️ Professional & Secure Deployment Template

This document provides a comprehensive guide for using a reusable GitLab CI/CD template designed for continuous deployment of web applications to Azure Web Apps. The pipeline emphasizes security, artifact consistency, and controlled promotion across environments (Staging to Production).

### Key Features

* **Zero-Downtime Strategy**: Sequential deployment ensuring Staging validation before Production.

* **Immutable Artifacts**: The same build package (`deploy_package.zip`) is used for all deployment stages.

* **Security Gate**: Production deployment is **manual**, enforcing human verification of the Staging environment.

* **Secure Authentication**: Uses Azure Service Principal credentials stored exclusively in GitLab Variables (Git-excluded secrets).

## 1. Pipeline Stages Overview

The pipeline executes through four distinct, sequential stages, ensuring that code is built, tested, and validated before deployment:

| Stage | Job | Description | Security / Trigger | 
 | ----- | ----- | ----- | ----- | 
| **`build`** | `build_package` | Compiles the application (Node/npm example) and creates the artifact (`deploy_package.zip`). | Creates the immutable artifact. | 
| **`test`** | `unit_tests` | Executes automated unit tests. | Quality gate. | 
|  | `security_scan` | Simulates a static analysis (SAST) or dependency scan. | Security feedback (non-blocking by default). | 
| **`deploy_staging`** | `deploy_to_staging` | **Automated deployment** to the Staging Web App (`my-webapp-staging`). | Triggered only by commits to the `main` branch. | 
| **`deploy_prod`** | `deploy_to_production` | Deploys the identical artifact to Production (`my-webapp-prod`). | **CRITICAL:** Requires **Manual** action after Staging success. | 

## 2. Configuration & Customization

### A. Global Variables (In `.gitlab-ci-azure-template.yml`)

These non-sensitive variables define resource names and must be customized for your project:

| Variable Name | Default Value | Description | Action Required | 
 | ----- | ----- | ----- | ----- | 
| `STAGING_APP_NAME` | `my-webapp-staging` | Azure Web App name for the Staging slot. | **Modify** | 
| `PROD_APP_NAME` | `my-webapp-prod` | Azure Web App name for the Production slot. | **Modify** | 
| `RESOURCE_GROUP_NAME` | `rg-app-production` | Common Azure Resource Group hosting the Web Apps. | **Modify** | 
| `AZURE_CLI_IMAGE` | `mcr.microsoft.com/azure-cli:latest` | Docker image used for all Azure CLI operations. | (Usually kept default) | 

### B. Secrets and Credentials (GitLab Variables)

**🚨 CRITICAL SECURITY STEP:** These variables **must not** be committed to Git. They must be configured in your GitLab project settings (**Settings > CI/CD > Variables**).

The pipeline uses a **Service Principal (SPN)** to authenticate with Azure.

| Variable Name | Description | Configuration | **Security Best Practice** | 
 | ----- | ----- | ----- | ----- | 
| `AZURE_TENANT_ID` | Your Azure Entra ID (Tenant) ID. | Masked, Protected. | Required for Service Principal identification. | 
| `AZURE_CLIENT_ID` | The Application ID of the Service Principal. | Masked, Protected. | The username for the SPN. | 
| `AZURE_CLIENT_SECRET` | The secret key/password for the Service Principal. | **Masked, Protected, Highly Recommended as File Type.** | The password for the SPN. | 

> **Permissions required for the Service Principal:** The SPN must be granted the **Contributor** role (or a least-privilege custom role covering `Microsoft.Web/*` and `Microsoft.Resources/subscriptions/resourceGroups/read`) on the Resource Group specified by `$RESOURCE_GROUP_NAME`.

## 3. Understanding the Authentication Anchor (`.azure_cli_template`)

For security and efficiency, all Azure deployment jobs inherit authentication logic via a YAML Anchor: