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
blob_json=$(az storage blob list \
    --account-name "$ACCOUNT" \
    --container-name "$CONTAINER_DOWNLOAD" \
    --auth-mode login \
    --output json)

echo "$blob_json" | jq -r '.[] | .name'

echo ""
echo "Downloading blobs individually..."
echo "$blob_json" | jq -r '.[] | .name' | while read -r blob_name; do
    echo "Downloading: $blob_name"
    az storage blob download \
        --account-name "$ACCOUNT" \
        --container-name "$CONTAINER_DOWNLOAD" \
        --name "$blob_name" \
        --file "$DOWNLOAD_DIR/$blob_name" \
        --auth-mode login \
        --overwrite
done

echo ""
echo "Files in download directory:"
ls -lR "$DOWNLOAD_DIR"

```
