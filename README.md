### build.pkr.hcl

```bash
#!/bin/bash

# Script to download files from Azure Blob Storage
# Reads configuration from .env file

set -e  # Exit on error

# Function to print messages
log_info() {
    echo "[INFO] $1"
}

log_error() {
    echo "[ERROR] $1"
}

log_warn() {
    echo "[WARN] $1"
}

# Get the directory where the script is located
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ENV_FILE="$SCRIPT_DIR/.env"

# Check if .env file exists
if [ ! -f "$ENV_FILE" ]; then
    log_error ".env file not found at: $ENV_FILE"
    exit 1
fi

# Load environment variables from .env file
log_info "Loading environment variables from .env file..."
# Remove Windows line endings and source the file
sed -i 's/\r$//' "$ENV_FILE" 2>/dev/null || true
set -a
source "$ENV_FILE"
set +a

# Validate required environment variables
required_vars=("AZ_TENANT_ID" "AZ_CLIENT_ID" "AZ_CLIENT_SECRET" "AZ_SUBSCRIPTION_ID" "ACCOUNT" "CONTAINER")
for var in "${required_vars[@]}"; do
    if [ -z "${!var}" ]; then
        log_error "Required environment variable $var is not set in .env file"
        exit 1
    fi
done

# Configuration
DOWNLOAD_DIR="/oracle/systemctl_scripts"
STORAGE_ACCOUNT="${ACCOUNT}"
CONTAINER="${CONTAINER}"
SOURCE_PREFIX="${SOURCE_PREFIX:-}"  # Optional: download only files with this prefix

log_info "Configuration:"
echo "  Download Directory: $DOWNLOAD_DIR"
echo "  Storage Account: $STORAGE_ACCOUNT"
echo "  Container: $CONTAINER"
if [ -n "$SOURCE_PREFIX" ]; then
    echo "  Source Prefix: $SOURCE_PREFIX"
fi

# Create download directory if it doesn't exist
if [ ! -d "$DOWNLOAD_DIR" ]; then
    log_info "Creating download directory: $DOWNLOAD_DIR"
    mkdir -p "$DOWNLOAD_DIR"
fi

# Check if Azure CLI is installed
if ! command -v az &> /dev/null; then
    log_error "Azure CLI is not installed. Please install it first."
    exit 1
fi

# Login to Azure using service principal
log_info "Authenticating to Azure..."
az login --service-principal \
    --username "$AZ_CLIENT_ID" \
    --password "$AZ_CLIENT_SECRET" \
    --tenant "$AZ_TENANT_ID" > /dev/null 2>&1

if [ $? -ne 0 ]; then
    log_error "Failed to authenticate with Azure"
    exit 1
fi

log_info "Successfully authenticated to Azure"

# Set the subscription
az account set --subscription "$AZ_SUBSCRIPTION_ID"

# Verify storage account exists
log_info "Verifying storage account..."
if ! az storage account show --name "$STORAGE_ACCOUNT" &> /dev/null; then
    log_error "Storage account '$STORAGE_ACCOUNT' not found"
    exit 1
fi

# Verify container exists
log_info "Verifying container..."
if ! az storage container show --name "$CONTAINER" --account-name "$STORAGE_ACCOUNT" &> /dev/null; then
    log_error "Container '$CONTAINER' not found in storage account '$STORAGE_ACCOUNT'"
    exit 1
fi

# Counter for downloaded files
downloaded_count=0
failed_count=0
skipped_count=0

# List all blobs in container
log_info "Listing blobs in container..."
if [ -n "$SOURCE_PREFIX" ]; then
    blob_list=$(az storage blob list \
        --account-name "$STORAGE_ACCOUNT" \
        --container-name "$CONTAINER" \
        --prefix "$SOURCE_PREFIX" \
        --auth-mode login \
        --query "[].name" -o tsv)
else
    blob_list=$(az storage blob list \
        --account-name "$STORAGE_ACCOUNT" \
        --container-name "$CONTAINER" \
        --auth-mode login \
        --query "[].name" -o tsv)
fi

if [ -z "$blob_list" ]; then
    log_warn "No blobs found in container"
    exit 0
fi

total_blobs=$(echo "$blob_list" | wc -l)
log_info "Found $total_blobs blobs to download"
echo ""

# Download each blob
while IFS= read -r blob_name; do
    if [ -z "$blob_name" ]; then
        continue
    fi
    
    # Create local file path
    local_file="$DOWNLOAD_DIR/$blob_name"
    local_dir=$(dirname "$local_file")
    
    # Create directory structure if needed
    if [ ! -d "$local_dir" ]; then
        mkdir -p "$local_dir"
    fi
    
    echo "Downloading: $blob_name -> $local_file"
    
    if az storage blob download \
        --account-name "$STORAGE_ACCOUNT" \
        --container-name "$CONTAINER" \
        --name "$blob_name" \
        --file "$local_file" \
        --auth-mode login \
        --overwrite 2>&1 | grep -v "Percent complete"; then
        ((downloaded_count++))
    else
        log_error "Failed to download: $blob_name"
        ((failed_count++))
    fi
done <<< "$blob_list"

# Summary
echo ""
log_info "Download completed!"
echo "  Files downloaded: $downloaded_count"
if [ $failed_count -gt 0 ]; then
    log_warn "Files failed: $failed_count"
fi

# Logout from Azure
az logout > /dev/null 2>&1

exit 0



```
