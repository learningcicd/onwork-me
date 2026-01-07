### build.pkr.hcl

```bash
#!/bin/bash
set -e

# Parse command line arguments
AUTO_FSTAB=true  # Default to true for automation
for arg in "$@"; do
    case $arg in
        --auto-fstab)
            AUTO_FSTAB=true
            shift
            ;;
        --interactive)
            AUTO_FSTAB=false
            shift
            ;;
    esac
done

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'  # No Color

# Log function
log() {
    echo -e "${GREEN}[$(date +'%Y-%m-%d %H:%M:%S')]${NC} $1"
}

error() {
    echo -e "${RED}[ERROR]${NC} $1" >&2
    exit 1
}

warn() {
    echo -e "${YELLOW}[WARNING]${NC} $1"
}

# Check if running as root or with sudo
if [[ $EUID -ne 0 ]]; then
    error "This script must be run as root or with sudo"
fi

# Load environment variables from .env file
ENV_FILE="/tmp/.env"
if [[ ! -f "$ENV_FILE" ]]; then
    error "Environment file not found: $ENV_FILE"
fi

log "Loading environment variables from $ENV_FILE"
source "$ENV_FILE"

# Validate required environment variables
REQUIRED_VARS=("AZURE_FS_ENDPOINT" "AZ_TENANT_ID" "AZ_CLIENT_ID" "AZ_CLIENT_SECRET")
for var in "${REQUIRED_VARS[@]}"; do
    if [[ -z "${!var}" ]]; then
        error "Required environment variable $var is not set in $ENV_FILE"
    fi
done

# Parse Azure File Share endpoint
# Expected format: https://<storage-account>.file.core.windows.net/<share-name>
STORAGE_ACCOUNT=$(echo "$AZURE_FS_ENDPOINT" | sed -E 's|https://([^.]+)\..*|\1|')
SHARE_NAME=$(echo "$AZURE_FS_ENDPOINT" | sed -E 's|.*/([^/]+)$|\1|')
APPLOGS_SHARE_NAME=$(echo "$AZURE_FS_ENDPOINT_APPLOGS" | sed -n 's|.*/\([^/]*\)|\1|p')

log "Storage Account: $STORAGE_ACCOUNT"
log "Share Name: $SHARE_NAME"
log "Applogs Share Name: $APPLOGS_SHARE_NAME"

# Destination directory
DEST_DIR="/domains"
MOUNT_POINT="/shareddomainconfig"
MOUNT_POINT1="/Application_logs"

# Create directories if they don't exist
mkdir -p "$DEST_DIR"
mkdir -p "$MOUNT_POINT"
mkdir -p "$MOUNT_POINT1"

# Check if cifs-utils is installed
if ! command -v mount.cifs &> /dev/null; then
    log "Installing cifs-utils..."
    if command -v apt-get &> /dev/null; then
        apt-get update && apt-get install -y cifs-utils
    elif command -v yum &> /dev/null; then
        yum install -y cifs-utils
    else
        error "Cannot install cifs-utils. Please install it manually."
    fi
fi

# Check if Azure CLI is installed
if ! command -v az &> /dev/null; then
    log "Installing Azure CLI..."
    curl -sL https://aka.ms/InstallAzureCLIDeb | bash
fi

# Login to Azure using service principal
log "Authenticating with Azure..."
if ! az account show &>/dev/null; then
    az login --service-principal \
        --username "$AZ_CLIENT_ID" \
        --password "$AZ_CLIENT_SECRET" \
        --tenant "$AZ_TENANT_ID" &> /dev/null || error "Azure authentication failed"
else
    log "Already authenticated with Azure"
fi

# Set subscription if provided
if [[ -n "${AZ_SUBSCRIPTION_ID:-}" ]]; then
    az account set --subscription "$AZ_SUBSCRIPTION_ID" || warn "Failed to set subscription, continuing..."
fi

# Get storage account key
log "Retrieving storage account key..."
STORAGE_KEY=$(az storage account keys list \
    --account-name "$STORAGE_ACCOUNT" \
    --query "[0].value" \
    --output tsv) || error "Failed to retrieve storage account key"

# Check if already mounted and unmount if necessary
if mountpoint -q "$MOUNT_POINT"; then
    log "Unmounting existing mount at $MOUNT_POINT"
    umount "$MOUNT_POINT" || warn "Failed to unmount $MOUNT_POINT"
fi

if mountpoint -q "$MOUNT_POINT1"; then
    log "Unmounting existing mount at $MOUNT_POINT1"
    umount "$MOUNT_POINT1" || warn "Failed to unmount $MOUNT_POINT1"
fi

# Mount Azure File Share
log "Mounting Azure File Share..."
mount -t cifs \
    "//${STORAGE_ACCOUNT}.file.core.windows.net/${SHARE_NAME}" \
    "$MOUNT_POINT" \
    -o username="$STORAGE_ACCOUNT",password="$STORAGE_KEY",dir_mode=0755,file_mode=0755,serverino || \
    error "Failed to mount Azure File Share at $MOUNT_POINT"

mount -t cifs \
    "//${STORAGE_ACCOUNT}.file.core.windows.net/${APPLOGS_SHARE_NAME}" \
    "$MOUNT_POINT1" \
    -o username="$STORAGE_ACCOUNT",password="$STORAGE_KEY",dir_mode=0755,file_mode=0755,serverino || \
    error "Failed to mount Azure File Share at $MOUNT_POINT1"

if [[ $? -eq 0 ]]; then
    log "Azure file share mounted successfully at $MOUNT_POINT1"
    
    # Add to /etc/fstab for persistent mount
    if [[ "$AUTO_FSTAB" = true ]]; then
        FSTAB_ENTRY="//${STORAGE_ACCOUNT}.file.core.windows.net/$APPLOGS_SHARE_NAME $MOUNT_POINT1 cifs nofail,vers=3.0,username=$STORAGE_ACCOUNT,password=$STORAGE_KEY,dir_mode=0777,file_mode=0777,serverino,uid=$(id -u oracle),gid=$(id -g oracle) 0 0"
        
        # Check if entry already exists
        if grep -q "$MOUNT_POINT1" /etc/fstab; then
            log "Entry for $MOUNT_POINT1 already exists in /etc/fstab"
        else
            echo "$FSTAB_ENTRY" | sudo tee -a /etc/fstab > /dev/null
            log "Entry added to /etc/fstab for persistent mounting"
        fi
    elif [[ "$AUTO_FSTAB" = false ]]; then
        read -p "Do you want to add this mount to /etc/fstab for persistent mounting? (y/n): " -n 1 -r
        echo
        if [[ $REPLY =~ ^[Yy]$ ]]; then
            FSTAB_ENTRY="//${STORAGE_ACCOUNT}.file.core.windows.net/$APPLOGS_SHARE_NAME $MOUNT_POINT1 cifs nofail,vers=3.0,username=$STORAGE_ACCOUNT,password=$STORAGE_KEY,dir_mode=0777,file_mode=0777,serverino,uid=$(id -u oracle),gid=$(id -g oracle) 0 0"
            
            # Check if entry already exists  
            if grep -q "$MOUNT_POINT1" /etc/fstab; then
                log "Entry for $MOUNT_POINT1 already exists in /etc/fstab"
            else
                echo "$FSTAB_ENTRY" | sudo tee -a /etc/fstab > /dev/null
                log "Entry added to /etc/fstab"
            fi
        fi
    fi
else
    log "Failed to mount Azure file share"
    exit 1
fi

# Install the rsync utility
log "Installing rsync utility..."
sudo yum -y install rsync

# Copy files
log "Copying files from $MOUNT_POINT to $DEST_DIR..."
rsync -avh --progress "$MOUNT_POINT/" "$DEST_DIR/" || error "Failed to copy files"

# Change ownership and permissions
log "Setting ownership to oracle:oracle for $DEST_DIR..."
chown -R oracle:oracle "$DEST_DIR" || error "Failed to change ownership"

log "Setting permissions to 755 for $DEST_DIR..."
chmod -R 755 "$DEST_DIR" || error "Failed to change permissions"

# Unmount
log "Unmounting Azure File Share..."
umount "$MOUNT_POINT"

# Logout from Azure
az logout &> /dev/null

log "Successfully copied Azure File Share content to $DEST_DIR"
log "Operation completed successfully!""
```
