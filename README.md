```bash
#!/bin/bash

# Configuration
SOURCE_DIR="/domains"
CONTAINER_NAME="domain"
ENV_FILE="/tmp/.env"

# Generate timestamp for unique filename
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
ARCHIVE_NAME="domains_backup_${TIMESTAMP}.tar.gz"
TEMP_ARCHIVE="/tmp/${ARCHIVE_NAME}"

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo -e "${YELLOW}Starting backup process...${NC}"

# Load environment variables from .env file
if [ -f "$ENV_FILE" ]; then
    echo -e "${YELLOW}Loading credentials from ${ENV_FILE}${NC}"
    export $(grep -v '^#' "$ENV_FILE" | xargs)
else
    echo -e "${RED}Error: .env file not found at ${ENV_FILE}${NC}"
    exit 1
fi

# Extract storage account name from ACCOUNT_ENDPOINT if not set
if [ -z "$AZURE_STORAGE_ACCOUNT" ]; then
    if [ -n "$ACCOUNT_ENDPOINT" ]; then
        # Extract account name from URL (e.g., https://wlsartifact.blob.core.windows.net -> wlsartifact)
        AZURE_STORAGE_ACCOUNT=$(echo "$ACCOUNT_ENDPOINT" | sed -n 's/https:\/\/\([^.]*\).*/\1/p')
        echo -e "${YELLOW}Extracted storage account: ${AZURE_STORAGE_ACCOUNT}${NC}"
    else
        echo -e "${RED}Error: Neither AZURE_STORAGE_ACCOUNT nor ACCOUNT_ENDPOINT found in .env${NC}"
        exit 1
    fi
fi

# Set BLOB_URL from ACCOUNT_ENDPOINT
if [ -n "$ACCOUNT_ENDPOINT" ]; then
    BLOB_URL="$ACCOUNT_ENDPOINT"
else
    BLOB_URL="https://${AZURE_STORAGE_ACCOUNT}.blob.core.windows.net"
fi

# Verify required variables for Service Principal authentication
if [ -z "$AZ_TENANT_ID" ] || [ -z "$AZ_CLIENT_ID" ] || [ -z "$AZ_CLIENT_SECRET" ] || [ -z "$AZ_SUBSCRIPTION_ID" ]; then
    echo -e "${RED}Error: Missing required Service Principal credentials in .env${NC}"
    echo -e "${RED}Required: AZ_TENANT_ID, AZ_CLIENT_ID, AZ_CLIENT_SECRET, AZ_SUBSCRIPTION_ID${NC}"
    exit 1
fi

echo -e "${YELLOW}Authenticating with Azure using Service Principal...${NC}"

# Check if source directory exists
if [ ! -d "$SOURCE_DIR" ]; then
    echo -e "${RED}Error: Source directory $SOURCE_DIR does not exist${NC}"
    exit 1
fi

# Create tar.gz archive
echo -e "${YELLOW}Creating archive: ${ARCHIVE_NAME}${NC}"
if tar -czf "$TEMP_ARCHIVE" -C / --exclude='domains/lost+found' --warning=no-file-changed domains 2>/dev/null || [ $? -eq 1 ]; then
    echo -e "${GREEN}Archive created successfully${NC}"
    ARCHIVE_SIZE=$(du -h "$TEMP_ARCHIVE" | cut -f1)
    echo -e "Archive size: ${ARCHIVE_SIZE}"
else
    echo -e "${RED}Error: Failed to create archive${NC}"
    exit 1
fi

# Upload to Azure Blob Storage using az CLI
echo -e "${YELLOW}Uploading to Azure Blob Storage...${NC}"
echo -e "Storage Account: ${AZURE_STORAGE_ACCOUNT}"
echo -e "Endpoint: ${BLOB_URL}"

if command -v az &> /dev/null; then
    # Login with Service Principal
    if az login --service-principal \
        --username "$AZ_CLIENT_ID" \
        --password "$AZ_CLIENT_SECRET" \
        --tenant "$AZ_TENANT_ID" > /dev/null 2>&1; then
        echo -e "${GREEN}Successfully authenticated with Azure${NC}"
        
        # Set the subscription
        az account set --subscription "$AZ_SUBSCRIPTION_ID"
        
        # Upload the blob
        if az storage blob upload \
            --account-name "${AZURE_STORAGE_ACCOUNT}" \
            --container-name "$CONTAINER_NAME" \
            --name "$ARCHIVE_NAME" \
            --file "$TEMP_ARCHIVE" \
            --auth-mode login \
            --overwrite; then
            echo -e "${GREEN}Upload successful!${NC}"
            echo -e "Blob URL: ${BLOB_URL}/${CONTAINER_NAME}/${ARCHIVE_NAME}"
            
            # Logout after upload
            az logout > /dev/null 2>&1
        else
            echo -e "${RED}Error: Upload failed${NC}"
            az logout > /dev/null 2>&1
            rm -f "$TEMP_ARCHIVE"
            exit 1
        fi
    else
        echo -e "${RED}Error: Azure authentication failed${NC}"
        echo -e "${RED}Please verify your Service Principal credentials${NC}"
        rm -f "$TEMP_ARCHIVE"
        exit 1
    fi
else
    echo -e "${RED}Error: Azure CLI (az) is not installed${NC}"
    echo "Please install it from: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli"
    rm -f "$TEMP_ARCHIVE"
    exit 1
fi

# Clean up temporary archive
echo -e "${YELLOW}Cleaning up temporary files...${NC}"
rm -f "$TEMP_ARCHIVE"

echo -e "${GREEN}Backup process completed successfully!${NC}"
```
