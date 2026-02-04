### build.pkr.hcl

```bash
#!/bin/bash
set -e

# Determine the directory where the script is located
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# Look for .env file (prioritize /tmp for Packer)
if [ -f "/tmp/.env" ]; then
    # Packer provisioner location
    ENV_FILE="/tmp/.env"
elif [ -n "$ENV_FILE_PATH" ] && [ -f "$ENV_FILE_PATH" ]; then
    # Use explicitly provided path
    ENV_FILE="$ENV_FILE_PATH"
elif [ -f ".env" ]; then
    # Current working directory
    ENV_FILE=".env"
elif [ -f "$SCRIPT_DIR/.env" ]; then
    # Same directory as script
    ENV_FILE="$SCRIPT_DIR/.env"
else
    echo "Error: .env file not found"
    echo "Searched locations:"
    echo "  - /tmp/.env (Packer default)"
    echo "  - ENV_FILE_PATH environment variable"
    echo "  - Current directory: $(pwd)/.env"
    echo "  - Script directory: $SCRIPT_DIR/.env"
    exit 1
fi

echo "Using .env file: $ENV_FILE"
source "$ENV_FILE"

# Validate required environment variables
if [ -z "$AZURE_STORAGE_ACCOUNT" ]; then
    echo "Error: AZURE_STORAGE_ACCOUNT not set in .env file"
    exit 1
fi

if [ -z "$AZURE_STORAGE_CONTAINER" ]; then
    echo "Error: AZURE_STORAGE_CONTAINER not set in .env file"
    exit 1
fi

# Validate Service Principal credentials
if [ -z "$AZURE_CLIENT_ID" ]; then
    echo "Error: AZURE_CLIENT_ID not set in .env file"
    exit 1
fi

if [ -z "$AZURE_CLIENT_SECRET" ]; then
    echo "Error: AZURE_CLIENT_SECRET not set in .env file"
    exit 1
fi

if [ -z "$AZURE_TENANT_ID" ]; then
    echo "Error: AZURE_TENANT_ID not set in .env file"
    exit 1
fi

# Check if Azure CLI is installed
if ! command -v az &> /dev/null; then
    echo "Error: Azure CLI is not installed"
    echo "Please install Azure CLI: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli"
    exit 1
fi

# Set blob path prefix if provided
BLOB_PATH_PREFIX="${AZURE_BLOB_PATH_PREFIX:-}"

# Login with Service Principal
echo "Authenticating with Azure Service Principal..."
az login --service-principal \
    --username "$AZURE_CLIENT_ID" \
    --password "$AZURE_CLIENT_SECRET" \
    --tenant "$AZURE_TENANT_ID" \
    --output none

if [ $? -ne 0 ]; then
    echo "Error: Failed to authenticate with Azure Service Principal"
    exit 1
fi

echo "Successfully authenticated with Azure"

# Function to download file from Azure Blob Storage
download_blob() {
    local blob_name=$1
    local output_file=$2
    
    echo "Downloading ${blob_name}..."
    
    az storage blob download \
        --account-name "$AZURE_STORAGE_ACCOUNT" \
        --container-name "$AZURE_STORAGE_CONTAINER" \
        --name "${BLOB_PATH_PREFIX}${blob_name}" \
        --file "$output_file" \
        --auth-mode login \
        --no-progress
    
    if [ $? -ne 0 ] || [ ! -f "$output_file" ]; then
        echo "Error: Failed to download ${blob_name}"
        exit 1
    fi
}

# Create .ssh directory if it doesn't exist
SSH_DIR="/home/oracle/.ssh"
sudo mkdir -p "$SSH_DIR"

# Download my_azure_key
download_blob "my_azure_key" "/tmp/my_azure_key"

# Download authorized_key
download_blob "authorized_key" "/tmp/authorized_key"

# Copy files to destination
echo "Copying files to $SSH_DIR..."
sudo cp /tmp/my_azure_key "$SSH_DIR/my_azure_key"
sudo cp /tmp/authorized_key "$SSH_DIR/authorized_key"

# Set ownership
echo "Setting ownership to oracle:oracle..."
sudo chown oracle:oracle "$SSH_DIR/my_azure_key"
sudo chown oracle:oracle "$SSH_DIR/authorized_key"

# Set permissions
echo "Setting permissions..."
sudo chmod 400 "$SSH_DIR/my_azure_key"      # -r--------
sudo chmod 644 "$SSH_DIR/authorized_key"    # -rw-r--r--

# Set ownership and permissions for .ssh directory
sudo chown oracle:oracle "$SSH_DIR"
sudo chmod 700 "$SSH_DIR"

# Clean up temporary files
rm -f /tmp/my_azure_key /tmp/authorized_key

# Logout from Azure
echo "Logging out from Azure..."
az logout --output none

echo "SSH keys successfully configured!"
echo "Private key: $SSH_DIR/my_azure_key (400)"
echo "Authorized key: $SSH_DIR/authorized_key (644)"

```

# GitLab Configuration
GITLAB_TOKEN=your_gitlab_personal_access_token_here
GITLAB_PROJECT_ID=12345678
GITLAB_URL=https://gitlab.com
GITLAB_BRANCH=main
