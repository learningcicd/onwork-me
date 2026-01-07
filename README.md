### build.pkr.hcl

```bash
#!/usr/bin/env bash
# download_install_bin_sp.sh
set -e

# Source environment variables from .env
if [[ -f /tmp/.env ]]; then
    source /tmp/.env
else
    echo "ERROR: /tmp/.env not found"
    exit 1
fi

DEST="/usr/wls-install-recepies/install-bin"
PREFIX="${PREFIX:-}"
DO_CHOWN="${DO_CHOWN:-oracle:oracle}"
DO_CHMOD="${DO_CHMOD:-755}"

usage(){ 
    echo "Usage: sudo bash $0 [--account <storageAccount>] [--container <container>] [--prefix <prefix>]"
}

# Command-line arguments can override .env values (optional)
while [[ $# -gt 0 ]]; do
    case "$1" in
        --account) ACCOUNT="$2"; shift 2 ;;
        --container) CONTAINER="$2"; shift 2 ;;
        --prefix) PREFIX="$2"; shift 2 ;;
        --chown) DO_CHOWN="$2"; shift 2 ;;
        --chmod) DO_CHMOD="$2"; shift 2 ;;
        -h|--help) usage; exit 0 ;;
        *) echo "Unknown arg: $1"; usage; exit 1 ;;
    esac
done

# Validate required variables
if [[ -z "${ACCOUNT:-}" ]] || [[ -z "${CONTAINER:-}" ]]; then
    echo "ERROR: ACCOUNT and CONTAINER must be set in /tmp/.env"
    exit 1
fi

# Validate Azure credentials
if [[ -z "${AZ_TENANT_ID:-}" ]] || [[ -z "${AZ_CLIENT_ID:-}" ]] || [[ -z "${AZ_CLIENT_SECRET:-}" ]]; then
    echo "ERROR: Azure credentials missing in /tmp/.env"
    echo "Required: AZ_TENANT_ID, AZ_CLIENT_ID, AZ_CLIENT_SECRET"
    exit 1
fi

# Network connectivity check function
check_connectivity() {
    echo "🔍 Checking network connectivity..."
    
    # Test basic internet connectivity
    if ! curl -I https://www.microsoft.com --max-time 10 >/dev/null 2>&1; then
        echo "ERROR: No internet connectivity. Please check your network connection."
        exit 1
    fi
    
    # Test DNS resolution
    if ! nslookup login.microsoftonline.com >/dev/null 2>&1; then
        echo "ERROR: Cannot resolve Azure DNS names. Checking DNS settings..."
        echo "Current DNS servers:"
        cat /etc/resolv.conf
        echo "Consider adding public DNS servers like 8.8.8.8 or 1.1.1.1"
        exit 1
    fi
    
    # Test Azure endpoints
    local endpoints=(
        "login.microsoftonline.com"
        "${ACCOUNT}.blob.core.windows.net"
    )
    
    for endpoint in "${endpoints[@]}"; do
        if ! curl -s --connect-timeout 5 "https://${endpoint}" >/dev/null 2>&1; then
            echo "WARNING: Cannot reach ${endpoint}. This may cause authentication issues."
        else
            echo "✅ Can reach ${endpoint}"
        fi
    done
}

# Load creds and configuration
[[ -f "/tmp/.env" ]] || { echo "ERROR: /tmp/.env not found"; exit 1; }
# shellcheck disable=SC1091
dos2unix /tmp/.env 2>/dev/null || sed -i 's/\r$//' /tmp/.env
source /tmp/.env
: "${AZ_TENANT_ID:?Missing AZ_TENANT_ID in /tmp/.env}"
: "${AZ_CLIENT_ID:?Missing AZ_CLIENT_ID in /tmp/.env}"
: "${AZ_CLIENT_SECRET:?Missing AZ_CLIENT_SECRET in /tmp/.env}"
# AZ_SUBSCRIPTION_ID is optional

# Apply Python warnings setting if specified in /tmp/.env
[[ -n "${PYTHONWARNINGS:-}" ]] && {
    echo "📝 Applying PYTHONWARNINGS from /tmp/.env: $PYTHONWARNINGS"
    export PYTHONWARNINGS
}

[[ $EUID -eq 0 ]] || { echo "Run as root (sudo)."; exit 1; }

# Run connectivity check
check_connectivity

mkdir -p "$DEST"; chmod "$DO_CHMOD" "$DEST"; chown "$DO_CHOWN" "$DEST" || true

install_azcli() {
    echo "Installing Azure CLI on RHEL/CentOS..."
    rpm --import https://packages.microsoft.com/keys/microsoft.asc
    
    cat > /etc/yum.repos.d/azure-cli.repo <<EOF
[azure-cli]
name=Azure CLI
baseurl=https://packages.microsoft.com/yumrepos/azure-cli
enabled=1
gpgcheck=1
gpgkey=https://packages.microsoft.com/keys/microsoft.asc
EOF
    
    dnf install -y azure-cli
}

install_azcli
az version

# Configure SSL/TLS and certificate handling
configure_ssl() {
    echo "🔒 Configuring SSL certificates..."
    
    # Update CA certificates
    if command -v update-ca-certificates >/dev/null 2>&1; then
        update-ca-certificates >/dev/null 2>&1 || true
    elif command -v update-ca-trust >/dev/null 2>&1; then
        update-ca-trust >/dev/null 2>&1 || true
    fi
    
    # Set Azure CLI SSL configuration
    export AZURE_CLI_DISABLE_CONNECTION_VERIFICATION=0
    export PYTHONHTTPSVERIFY=1
    export REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
    
    # Alternative paths for different distros
    if [[ ! -f "$REQUESTS_CA_BUNDLE" ]]; then
        for ca_bundle in "/etc/ssl/certs/ca-bundle.crt" "/etc/pki/tls/certs/ca-bundle.crt"; do
            if [[ -f "$ca_bundle" ]]; then
                export REQUESTS_CA_BUNDLE="$ca_bundle"
                break
            fi
        done
    fi
}

configure_ssl

# Suppress only the urllib3 warning, not disable verification
export PYTHONWARNINGS="ignore:Unverified HTTPS request"

echo "🔐 Logging in (allow-no-subscriptions)..."
# Add retry logic and better error handling
for attempt in 1 2 3; do
    echo "Login attempt $attempt/3..."
    echo "AZ_CLIENT_ID=$AZ_CLIENT_ID"
    echo "AZ_TENANT_ID=$AZ_TENANT_ID"
    echo "AZ_CLIENT_SECRET=$AZ_CLIENT_SECRET"
    
    if az login --debug \
        --service-principal \
        --username "$AZ_CLIENT_ID" \
        --password "$AZ_CLIENT_SECRET" \
        --tenant "$AZ_TENANT_ID" \
        --allow-no-subscriptions; then
        echo "✅ Login successful"
        break
    else
        echo "❌ Login attempt $attempt failed"
        if [[ $attempt -eq 3 ]]; then
            echo "ERROR: All login attempts failed. Please check:"
            echo "1. Network connectivity to Azure"
            echo "2. Service Principal credentials are correct"
            echo "3. Tenant ID is correct"
            echo "4. DNS resolution is working"
            exit 1
        fi
        sleep 5
    fi
done

[[ -n "${AZ_SUBSCRIPTION_ID:-}" ]] && {
    echo "Setting subscription to ${AZ_SUBSCRIPTION_ID}..."
    az account set --subscription "$AZ_SUBSCRIPTION_ID" >/dev/null 2>&1 || true
}

# Sanity checks with better error handling
echo "🔍 Sanity checks: verify container and listing blobs."
if ! az storage container show --name "$CONTAINER" --account-name "$ACCOUNT" --auth-mode login >/dev/null 2>&1; then
    echo "ERROR: Cannot access container '$CONTAINER' in storage account '$ACCOUNT'"
    echo "Please verify:"
    echo "1. Storage account name is correct"
    echo "2. Container exists"
    echo "3. Service principal has proper permissions (Storage Blob Data Reader/Contributor)"
    echo "4. Network allows access to ${ACCOUNT}.blob.core.windows.net"
    exit 1
fi

BLOB_COUNT=$(az storage blob list \
    --container-name "$CONTAINER" \
    --account-name "$ACCOUNT" \
    --auth-mode login \
    ${PREFIX:+--prefix "$PREFIX"} \
    --query 'length(@)' -o tsv 2>/dev/null || echo 0)

echo "Found ${BLOB_COUNT} blobs under ${CONTAINER}/${PREFIX:-}"

if [[ "$BLOB_COUNT" -eq 0 ]]; then
    echo "WARNING: No blobs found. Check the prefix path if specified."
fi

# Download with better error handling
echo "⬇️ Downloading blobs from ${ACCOUNT}/${CONTAINER} → ${DEST} …"
if ! az storage blob download \
    --account-name "$ACCOUNT" \
    --auth-mode login \
    --container-name "$CONTAINER" \
    --name "jdk1.8.0_271.tar.gz" \
    --file "${DEST}/jdk1.8.0_271.tar.gz"; then
    echo "ERROR: Download of JDK failed. Check network connectivity and permissions."
    exit 1
fi

if ! az storage blob download \
    --account-name "$ACCOUNT" \
    --auth-mode login \
    --container-name "$CONTAINER" \
    --name "weblogic_binaries.tar" \
    --file "${DEST}/weblogic_binaries.tar"; then
    echo "ERROR: Download of WEBLOGIC failed. Check network connectivity and permissions."
    exit 1
fi

chmod -R "$DO_CHMOD" "$DEST" || true
chown -R "$DO_CHOWN" "$DEST" || true
echo "✅ All blobs downloaded to ${DEST}"
```
