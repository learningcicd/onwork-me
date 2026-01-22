### build.pkr.hcl

```bash
#!/bin/bash

# Script to upload files recursively to Azure Blob Storage
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

# Check if .env file exists
if [ ! -f .env ]; then
    log_error ".env file not found in current directory"
    exit 1
fi

# Load environment variables from .env file
log_info "Loading environment variables from .env file..."
export $(grep -v '^#' .env | xargs)

# Validate required environment variables
required_vars=("AZ_TENANT_ID" "AZ_CLIENT_ID" "AZ_CLIENT_SECRET" "AZ_SUBSCRIPTION_ID" "ACCOUNT" "CONTAINER")
for var in "${required_vars[@]}"; do
    if [ -z "${!var}" ]; then
        log_error "Required environment variable $var is not set in .env file"
        exit 1
    fi
done

# Configuration
SOURCE_DIR="/home/guava/wls_ftAdminpacker_Rakesh/recepies/wlstscripts_onboot"
STORAGE_ACCOUNT="${ACCOUNT}"
CONTAINER="${CONTAINER}"
TARGET_DIR="tn_dev"

log_info "Configuration:"
echo "  Source Directory: $SOURCE_DIR"
echo "  Storage Account: $STORAGE_ACCOUNT"
echo "  Container: $CONTAINER"
echo "  Target Directory: $TARGET_DIR"

# Check if source directory exists
if [ ! -d "$SOURCE_DIR" ]; then
    log_error "Source directory does not exist: $SOURCE_DIR"
    exit 1
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

# Counter for uploaded files
uploaded_count=0
failed_count=0

# Function to upload a single file
upload_file() {
    local file_path="$1"
    local relative_path="${file_path#$SOURCE_DIR/}"
    local blob_path="$TARGET_DIR/$relative_path"
    
    log_info "Uploading: $relative_path"
    
    if az storage blob upload \
        --account-name "$STORAGE_ACCOUNT" \
        --container-name "$CONTAINER" \
        --name "$blob_path" \
        --file "$file_path" \
        --overwrite \
        --auth-mode login > /dev/null 2>&1; then
        ((uploaded_count++))
    else
        log_error "Failed to upload: $relative_path"
        ((failed_count++))
    fi
}

# Export function so it can be used with find
export -f upload_file
export -f log_info
export -f log_error
export SOURCE_DIR
export TARGET_DIR
export STORAGE_ACCOUNT
export CONTAINER
export uploaded_count
export failed_count

# Find and upload all files recursively
log_info "Starting recursive upload from $SOURCE_DIR..."
echo ""

# Use find to get all files and upload them
while IFS= read -r -d '' file; do
    if [ -f "$file" ]; then
        upload_file "$file"
    fi
done < <(find "$SOURCE_DIR" -type f -print0)

# Summary
echo ""
log_info "Upload completed!"
echo "  Files uploaded: $uploaded_count"
if [ $failed_count -gt 0 ]; then
    log_warn "Files failed: $failed_count"
fi

# Logout from Azure
az logout > /dev/null 2>&1

exit 0
"
```
