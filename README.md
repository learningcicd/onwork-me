### build.pkr.hcl

```bash
locals {
  as_root   = "chmod +x {{ .Path }}; {{ .Vars }} sudo -E bash '{{ .Path }}'"
  as_oracle = "chmod +x {{ .Path }}; {{ .Vars }} sudo -u oracle -g oracle -E bash '{{ .Path }}'"
}

build {
  sources = [
    "source.azure-arm.vm"
  ]

  provisioner "file" {
    source      = "../recepies-managed/.env"
    destination = "/tmp/.env"
  }

  provisioner "shell" {
    execute_command = local.as_root
    environment_vars = [
      "CREDENTIAL_ENV=/tmp/.env",
    ]
    scripts = [
      "../recepies-managed/configure-disk-mounts.sh",
      "../recepies-managed/configure_ownership.sh",
      "../recepies-managed/create-folder.sh",
      "../recepies-managed/download_packages.sh",
    ]
  }

  provisioner "shell" {
    execute_command = local.as_oracle
    environment_vars = [
      "CREDENTIAL_ENV=/tmp/.env",
    ]
    scripts = [
      "../recepies-managed/install-ora-weblogic.sh",
    ]
  }

  provisioner "shell" {
    execute_command = local.as_root
    scripts = [
      "../recepies-managed/download_domain.sh",
    ]
  }
  provisioner "file" {
  source      = "../recepies-managed/create_jupiter-startup.sh"
  destination = "/tmp/create_jupiter-startup.sh"
  }
  
  provisioner "file" {
    source      = "../recepies-managed/create_jupiter-startup.sh"
    destination = "/domains/DynamicDomain/bin/create_jupiter-startup.sh"
  }

  provisioner "shell" {
    inline = [
      "chmod +x /tmp/create_jupiter-startup.sh",
    ]
  }

  provisioner "shell" {
    script = "../recepies-managed/enable-ms-jupiter.sh"  # singular
  } 

  provisioner "shell" {
    execute_command = local.as_root
    inline = ["rm -f /tmp/.env && sync"]
  }

  provisioner "shell" {
    execute_command = local.as_root
    inline = ["/usr/sbin/waagent -force -deprovision+user && export HISTSIZE=0 && sync"]
    only = ["azure-arm"]
  }
}

```

### download_domain.sh
```bash
#!/bin/bash

set -e

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

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
REQUIRED_VARS=("AZURE_FS_ENDPOINT" "AZ_TENANT_ID" "AZ_CLIENT_ID" "AZ_CLIENT_SECRET" "AZ_SUBSCRIPTION_ID")
for var in "${REQUIRED_VARS[@]}"; do
    if [[ -z "${!var}" ]]; then
        error "Required environment variable $var is not set in $ENV_FILE"
    fi
done

# Parse Azure File Share endpoint
# Expected format: https://<storage-account>.file.core.windows.net/<share-name>
STORAGE_ACCOUNT=$(echo "$AZURE_FS_ENDPOINT" | sed -E 's|https://([^.]+)\..*|\1|')
SHARE_NAME=$(echo "$AZURE_FS_ENDPOINT" | sed -E 's|.*/([^/]+)/?$|\1|')

log "Storage Account: $STORAGE_ACCOUNT"
log "Share Name: $SHARE_NAME"

# Destination directory
DEST_DIR="/domains"
MOUNT_POINT="/mnt/azure_fileshare"

# Create directories if they don't exist
mkdir -p "$DEST_DIR"
mkdir -p "$MOUNT_POINT"

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
az login --service-principal \
    --username "$AZ_CLIENT_ID" \
    --password "$AZ_CLIENT_SECRET" \
    --tenant "$AZ_TENANT_ID" &> /dev/null || error "Azure authentication failed"

az account set --subscription "$AZ_SUBSCRIPTION_ID" || error "Failed to set subscription"

# Get storage account key
log "Retrieving storage account key..."
STORAGE_KEY=$(az storage account keys list \
    --account-name "$STORAGE_ACCOUNT" \
    --query "[0].value" \
    --output tsv) || error "Failed to retrieve storage account key"

# Check if already mounted and unmount if necessary
if mountpoint -q "$MOUNT_POINT"; then
    log "Unmounting existing mount at $MOUNT_POINT"
    umount "$MOUNT_POINT"
fi

# Mount Azure File Share
log "Mounting Azure File Share..."
mount -t cifs \
    "//${STORAGE_ACCOUNT}.file.core.windows.net/${SHARE_NAME}" \
    "$MOUNT_POINT" \
    -o username="$STORAGE_ACCOUNT",password="$STORAGE_KEY",dir_mode=0755,file_mode=0755,serverino || error "Failed to mount Azure File Share"

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
log "Operation completed successfully!"
```

### Create create_jupiter-startup

```bash
#!/bin/bash
# Script to create server-specific startup script in /domains/DynamicDomain/bin/

TARGET_DIR="/domains/DynamicDomain/bin"

# Determine server name first
MANAGEDSERVER_NAME=Jupiter
HOSTNAME=$(hostname)
SERVER_NAME="${MANAGEDSERVER_NAME}-${HOSTNAME}"

TARGET_FILE="${TARGET_DIR}/${SERVER_NAME}.sh"

# Create the target directory if it doesn't exist
mkdir -p "${TARGET_DIR}"

# Create the server-specific startup script
cat > "${TARGET_FILE}" << 'EOF'
MANAGEDSERVER_NAME=Jupiter
ADMIN_URL=t3://${ADMIN_HOST}:7001

# Get last character of hostname
HOSTNAME=$(hostname)



# Construct full server name
SERVER_NAME="${MANAGEDSERVER_NAME}-${HOSTNAME}"

# Paths
SCRIPT_PATH="/domains/DynamicDomain/bin/${SERVER_NAME}.sh"
SERVICE_NAME="${SERVER_NAME}.service"
SERVICE_PATH="/etc/systemd/system/${SERVICE_NAME}"
LOG_PATH="/var/log/${SERVER_NAME}.log"

echo "Creating startup script for $SERVER_NAME..."
sudo cp -rp /domains/DynamicDomain/bin/startManagedWebLogic.sh /domains/DynamicDomain/bin/${SERVER_NAME}.sh
chmod +x "$SCRIPT_PATH"
echo "Script made executable."

# Create boot.properties before starting the managed server
DOMAIN_PATH="/domains/DynamicDomain"
SERVER_DIR="${DOMAIN_PATH}/servers/${SERVER_NAME}"
SECURITY_DIR="${SERVER_DIR}/security"
BOOT_FILE="${SECURITY_DIR}/boot.properties"

if [ ! -f "$BOOT_FILE" ]; then
  echo "Creating boot.properties for $SERVER_NAME..."
  mkdir -p "$SECURITY_DIR"
  chown -R oracle:oracle "$SECURITY_DIR"
  cat << BOOTEOF > "$BOOT_FILE"
username=weblogic
password=Welcome1
BOOTEOF
  chmod 600 "$BOOT_FILE"
  echo "boot.properties created at $BOOT_FILE"
else
  echo "boot.properties already exists at $BOOT_FILE — skipping creation."
fi
EOF

# Make the created file executable
chmod +x "${TARGET_FILE}"

echo "Successfully created ${TARGET_FILE}"
echo "File permissions set to executable"
```

### enable-ms-jupiter.sh

```bash
#!/bin/bash
set -e

# Define variables
MANAGEDSERVER_NAME=Jupiter
HOSTNAME=$(hostname)

# Extract numeric suffix from hostname first
#SUFFIX=$(echo "${HOSTNAME}" | grep -o '[0-9]*$')

if [ -z "$SUFFIX" ]; then
    echo "No numeric suffix found in hostname, finding most recently created script..."
    
    # Find the most recently created Jupiter-*.sh file
    LATEST_SCRIPT=$(ls -t /domains/DynamicDomain/bin/${MANAGEDSERVER_NAME}-*.sh 2>/dev/null | head -1)
    
    if [ -n "$LATEST_SCRIPT" ]; then
        # Extract suffix from the filename
        SUFFIX=$(basename "$LATEST_SCRIPT" | grep -o '[0-9]\+' | head -1)
        echo "Found most recent script: $LATEST_SCRIPT (suffix: $SUFFIX)"
    else
        # No Jupiter scripts found, default to 1
        SUFFIX=1
        echo "No existing Jupiter scripts found, using suffix: $SUFFIX"
    fi
else
    echo "Detected suffix from hostname: $SUFFIX"
fi

SERVER_NAME="${MANAGEDSERVER_NAME}-${HOSTNAME}"
DOMAIN_PATH="/domains/DynamicDomain"
SCRIPT_PATH="/domains/DynamicDomain/bin/${SERVER_NAME}.sh"
SERVICE_NAME="${SERVER_NAME}.service"
SERVICE_PATH="/etc/systemd/system/${SERVICE_NAME}"
LOG_PATH="/var/log/${SERVER_NAME}.log"

echo "========================================"
echo "Creating systemd service for ${SERVER_NAME}..."
echo "Server Name: ${SERVER_NAME}"
echo "Suffix: ${SUFFIX}"
echo "Service name: ${SERVICE_NAME}"
echo "Script path: ${SCRIPT_PATH}"
echo "========================================"

# Verify the startup script exists
if [ ! -f "${SCRIPT_PATH}" ]; then
    echo "ERROR: Startup script not found at ${SCRIPT_PATH}"
    echo "Available scripts in /domains/DynamicDomain/bin/:"
    ls -la /domains/DynamicDomain/bin/*.sh 2>/dev/null || echo "  No .sh scripts found"
    exit 1
fi

# Create systemd service file
echo "Creating service file..."
sudo tee "${SERVICE_PATH}" > /dev/null << EOF
[Unit]
Description=WebLogic Managed Server - ${SERVER_NAME}
After=network.target

[Service]
Type=forking
User=oracle
Group=oracle
Environment="JAVA_HOME=/usr/java/default"
Environment="MW_HOME=/u01/oracle/middleware"
WorkingDirectory=${DOMAIN_PATH}
ExecStart=${SCRIPT_PATH}
ExecStop=${DOMAIN_PATH}/bin/stopManagedWebLogic.sh ${SERVER_NAME} t3://localhost:7001
StandardOutput=append:${LOG_PATH}
StandardError=append:${LOG_PATH}
Restart=on-failure
RestartSec=30
TimeoutStartSec=600
TimeoutStopSec=180

[Install]
WantedBy=multi-user.target
EOF

echo "Service file created at ${SERVICE_PATH}"

# Set permissions
sudo chmod 644 "${SERVICE_PATH}"

# Reload systemd
echo "Reloading systemd daemon..."
sudo systemctl daemon-reload

# Enable the service
echo "Enabling service ${SERVICE_NAME}..."
sudo systemctl enable "${SERVICE_NAME}"

# Check if service is enabled
echo "Verifying service ${SERVICE_NAME} is enabled..."
if sudo systemctl is-enabled "${SERVICE_NAME}" >/dev/null 2>&1; then
    echo "✓ Service ${SERVICE_NAME} is enabled"
else
    echo "⚠ Warning: Service may not be enabled properly"
fi

echo "========================================"
echo " Service ${SERVICE_NAME} has been created and enabled"
echo ""
echo "Next steps:"
echo "  Start:  sudo systemctl start ${SERVICE_NAME}"
echo "  Status: sudo systemctl status ${SERVICE_NAME}"
echo "  Logs:   sudo journalctl -u ${SERVICE_NAME} -f"
echo "  Stop:   sudo systemctl stop ${SERVICE_NAME}"
echo "========================================"
```
