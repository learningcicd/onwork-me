### build.pkr.hcl

```bash
#!/bin/bash

# Simplified test script
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ENV_FILE="$SCRIPT_DIR/.env"

set -a
source "$ENV_FILE"
set +a

DOWNLOAD_DIR="/oracle/systemctl_scripts"
mkdir -p "$DOWNLOAD_DIR"

echo "Logging in..."
az login --service-principal \
    --username "$AZ_CLIENT_ID" \
    --password "$AZ_CLIENT_SECRET" \
    --tenant "$AZ_TENANT_ID"

echo "Setting subscription..."
az account set --subscription "$AZ_SUBSCRIPTION_ID"

echo "Listing blobs..."
az storage blob list \
    --account-name "$ACCOUNT" \
    --container-name "$CONTAINER_DOWNLOAD" \
    --auth-mode login \
    --output table

echo ""
echo "Downloading ALL blobs using az storage blob download-batch..."
az storage blob download-batch \
    --account-name "$ACCOUNT" \
    --source "$CONTAINER_DOWNLOAD" \
    --destination "$DOWNLOAD_DIR" \
    --auth-mode login \
    --overwrite

echo ""
echo "Files in download directory:"
ls -lR "$DOWNLOAD_DIR"


```
