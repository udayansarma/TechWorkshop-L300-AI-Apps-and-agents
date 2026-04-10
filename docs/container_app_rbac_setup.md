# Container App RBAC Setup

This document captures the RBAC (Role-Based Access Control) changes required for the container app's system-assigned managed identity to access all dependent Azure services.

## Problem

After deploying the container app, API calls to Azure AI Foundry agents/conversations fail with:

```
PermissionDenied: The principal <principal-id> lacks the required data action
Microsoft.CognitiveServices/accounts/AIServices/agents/write to perform
POST /api/projects/{projectName}/openai/* operation.
```

The Bicep template (`src/infra/DeployAzureResources.bicep`) only assigns `AcrPull` to the container app's managed identity. Additional roles are needed for AI Foundry, Cosmos DB, and Storage.

## Key Findings

- **Azure AI Developer** role does **NOT** include `Microsoft.CognitiveServices/accounts/AIServices/agents/write` — it only covers `OpenAI/*`, `SpeechServices/*`, `ContentSafety/*`, and `MaaS/*`.
- **Azure AI User** role has `Microsoft.CognitiveServices/*` (wildcard) which covers all data actions including the agents/conversations API.
- Roles assigned on a parent AI Services account do **NOT** always inherit to the project sub-resource for data plane operations. Roles must be assigned directly on the **project** scope.
- Cosmos DB uses its own RBAC system, separate from Azure RBAC.

## Roles Required

| Role | Scope | Purpose |
|------|-------|---------|
| **Azure AI User** | AI Services account | Wildcard data plane access for Cognitive Services |
| **Azure AI User** | AI Foundry project | Conversations API and agents API calls |
| **Cognitive Services OpenAI User** | AI Services account | OpenAI model inference |
| **Storage Blob Data Contributor** | Storage account | Blob read/write |
| **Cosmos DB Built-in Data Contributor** | Cosmos DB account | Document read/write (Cosmos DB's own RBAC) |

## Ready-to-Execute Script

Update the placeholder values before running.

```bash
#!/bin/bash
set -euo pipefail

# ============================================================
# CONFIGURATION — Update these values for your environment
# ============================================================
RESOURCE_GROUP="rg-udyaiapps"
AI_SERVICES_ACCOUNT="aif-6f2kxcdhlez5q"
AI_PROJECT="proj-6f2kxcdhlez5q"
STORAGE_ACCOUNT="6f2kxcdhlez5qsa"
COSMOS_DB_ACCOUNT="6f2kxcdhlez5q-cosmosdb"
CONTAINER_APP_NAME="ca-6f2kxcdhlez5q-app"

# ============================================================
# Resolve resource IDs and principal ID
# ============================================================
echo "Resolving resource IDs..."

CONTAINER_APP_PRINCIPAL_ID=$(az containerapp show \
  --name "$CONTAINER_APP_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --query "identity.principalId" -o tsv)

AI_SERVICES_SCOPE=$(az cognitiveservices account show \
  --name "$AI_SERVICES_ACCOUNT" \
  --resource-group "$RESOURCE_GROUP" \
  --query id -o tsv)

AI_PROJECT_SCOPE="${AI_SERVICES_SCOPE}/projects/${AI_PROJECT}"

STORAGE_SCOPE=$(az storage account show \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RESOURCE_GROUP" \
  --query id -o tsv)

echo "Container App Principal ID: $CONTAINER_APP_PRINCIPAL_ID"
echo ""

# ============================================================
# 1. Azure AI User — on AI Services account
# ============================================================
echo "Assigning Azure AI User on AI Services account..."
az role assignment create \
  --assignee "$CONTAINER_APP_PRINCIPAL_ID" \
  --role "Azure AI User" \
  --scope "$AI_SERVICES_SCOPE" \
  --output none

# ============================================================
# 2. Azure AI User — on AI Foundry project (CRITICAL)
# ============================================================
echo "Assigning Azure AI User on AI Foundry project..."
az role assignment create \
  --assignee "$CONTAINER_APP_PRINCIPAL_ID" \
  --role "Azure AI User" \
  --scope "$AI_PROJECT_SCOPE" \
  --output none

# ============================================================
# 3. Cognitive Services OpenAI User — on AI Services account
# ============================================================
echo "Assigning Cognitive Services OpenAI User on AI Services account..."
az role assignment create \
  --assignee "$CONTAINER_APP_PRINCIPAL_ID" \
  --role "Cognitive Services OpenAI User" \
  --scope "$AI_SERVICES_SCOPE" \
  --output none

# ============================================================
# 4. Storage Blob Data Contributor — on Storage account
# ============================================================
echo "Assigning Storage Blob Data Contributor on Storage account..."
az role assignment create \
  --assignee "$CONTAINER_APP_PRINCIPAL_ID" \
  --role "Storage Blob Data Contributor" \
  --scope "$STORAGE_SCOPE" \
  --output none

# ============================================================
# 5. Cosmos DB Built-in Data Contributor (Cosmos DB own RBAC)
# ============================================================
echo "Assigning Cosmos DB Built-in Data Contributor..."
az cosmosdb sql role assignment create \
  --account-name "$COSMOS_DB_ACCOUNT" \
  --resource-group "$RESOURCE_GROUP" \
  --role-definition-id "00000000-0000-0000-0000-000000000002" \
  --principal-id "$CONTAINER_APP_PRINCIPAL_ID" \
  --scope "/"

# ============================================================
# 6. Restart the container app to pick up new tokens
# ============================================================
echo ""
echo "Restarting container app to refresh tokens..."
REVISION=$(az containerapp revision list \
  --name "$CONTAINER_APP_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --query "[0].name" -o tsv)

az containerapp revision restart \
  --name "$CONTAINER_APP_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --revision "$REVISION" \
  --output none

echo ""
echo "Done! RBAC propagation may take 1-5 minutes."
echo "Wait a few minutes, then test the application."
```

## Bicep Fix — Container App Name Prefix

The container app resource names were also updated to start with alphabetic prefixes since `uniqueString()` can produce hashes starting with a digit, which violates Azure naming rules:

```bicep
// Before (could start with a digit):
var containerAppName = '${uniqueString(resourceGroup().id)}-app'
var containerAppEnvName = '${uniqueString(resourceGroup().id)}-cosu-cae'

// After (always starts with a letter):
var containerAppName = 'ca-${uniqueString(resourceGroup().id)}-app'
var containerAppEnvName = 'cae-${uniqueString(resourceGroup().id)}-cosu-cae'
```

cd src
az acr build --registry 6f2kxcdhlez5qcosureg --image chat-app:latest --platform linux/amd64 --file Dockerfile .
az containerapp update --name ca-6f2kxcdhlez5q-app --resource-group rg-udyaiapps --image 6f2kxcdhlez5qcosureg.azurecr.io/chat-app:latest

az ad sp create-for-rbac --name "TechWorkshopL300AzureAIUdy" --json-auth --role contributor --scopes /subscriptions/82292e9d-a706-480d-9341-63e0bd5489a9/resourceGroups/rg-udyaiapps

az ad sp create-for-rbac \
  --name "github-actions-sp" \
  --role contributor \
  --scopes /subscriptions/<YOUR_SUBSCRIPTION_ID>/resourceGroups/<YOUR_RESOURCE_GROUP> \
  --json-auth

az ad sp create-for-rbac \
  --name "TechWorkshopL300AzureAIUdy" \
  --role contributor \
  --scopes /subscriptions/82292e9d-a706-480d-9341-63e0bd5489a9/resourceGroups/rg-udyaiapps \
  --create-password false

# Step 2: Get the app ID
APP_ID=$(az ad sp list --display-name "TechWorkshopL300AzureAIUdy" --query "[0].appId" -o tsv)

# Step 3: Add a 29-day credential
az ad app credential reset --id "$APP_ID" --end-date "$(date -u -d '+29 days' '+%Y-%m-%dT%H:%M:%SZ')" --append
Creating 'contributor' role assignment under scope '/subscriptions/82292e9d-a706-480d-9341-63e0bd5489a9/