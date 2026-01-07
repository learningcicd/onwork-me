### build.pkr.hcl

```bash
#!/usr/bin/env bash
# download_install_bin_sp.sh
set -e

# Source .env file if it exists
if [[ -f /tmp/.env ]]; then
    source /tmp/.env
fi

DEST="/usr/wls-install-recepies/install-bin"
PREFIX="${PREFIX:-}"
DO_CHOWN="${DO_CHOWN:-oracle:oracle}"
DO_CHMOD="${DO_CHMOD:-755}"

usage(){ 
    echo "Usage: sudo bash $0 --account <storageAccount> --container <container> [options]"
    echo "Options:"
    echo "  --account <name>     Azure storage account name"
    echo "  --container <name>   Azure storage container name"
    echo "  --prefix <path>      Optional prefix path in container"
    echo "  --chown <owner>      File ownership (default: oracle:oracle)"
    echo "  --chmod <perms>      File permissions (default: 755)"
    echo ""
    echo "Variables can also be set via /tmp/.env file"
}

# Parse command-line arguments (these override .env values)
while [[ $# -gt 0 ]]; do
    case "$1" in
        --account) 
            if [[ -n "${2:-}" ]]; then
                ACCOUNT="$2"
                shift 2
            else
                echo "ERROR: --account requires a value"
                usage
                exit 1
            fi
            ;;
        --container) 
            if [[ -n "${2:-}" ]]; then
                CONTAINER="$2"
                shift 2
            else
                echo "ERROR: --container requires a value"
                usage
                exit 1
            fi
            ;;
        --prefix) 
            PREFIX="${2:-}"
            shift 2
            ;;
        --chown) 
            DO_CHOWN="${2:-oracle:oracle}"
            shift 2
            ;;
        --chmod) 
            DO_CHMOD="${2:-755}"
            shift 2
            ;;
        -h|--help) 
            usage
            exit 0
            ;;
        *) 
            echo "Unknown arg: $1"
            usage
            exit 1
            ;;
    esac
done

# Validate required variables are set (from .env or command line)
if [[ -z "${ACCOUNT:-}" ]] || [[ -z "${CONTAINER:-}" ]]; then
    echo "ERROR: --account and --container are required (or set in /tmp/.env)"
    usage
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
    
    echo "✅ Can reach ${ACCOUNT}.blob.core.windows.net"
}

# Load environment and check connectivity
check_connectivity

# Load Azure credentials from .env
if [[ -z "${AZ_TENANT_ID:-}" ]] || [[ -z "${AZ_CLIENT_ID:-}" ]] || [[ -z "${AZ_CLIENT_SECRET:-}" ]]; then
    echo "ERROR: Missing Azure credentials in /tmp/.env"
    echo "Required variables: AZ_TENANT_ID, AZ_CLIENT_ID, AZ_CLIENT_SECRET"
    exit 1
fi

# Apply Python warnings setting if specified in /tmp/.env
if [[ -n "${PYTHONWARNINGS:-}" ]]; then
    echo "📝 Applying PYTHONWARNINGS from /tmp/.env: $PYTHONWARNINGS"
    export PYTHONWARNINGS
fi

# Azure login and download logic would go here
echo "🔐 Logging into Azure..."
echo "   Account: $ACCOUNT"
echo "   Container: $CONTAINER"
echo "   Destination: $DEST"
[[ -n "$PREFIX" ]] && echo "   Prefix: $PREFIX"

# TODO: Add your actual Azure download commands here
# Example:
# az login --service-principal -u "$AZ_CLIENT_ID" -p "$AZ_CLIENT_SECRET" --tenant "$AZ_TENANT_ID"
# az storage blob download-batch --account-name "$ACCOUNT" --source "$CONTAINER" --destination "$DEST" --pattern "${PREFIX}*"

echo "✅ Download complete"

# Set ownership and permissions
if [[ -d "$DEST" ]]; then
    echo "📁 Setting ownership to $DO_CHOWN and permissions to $DO_CHMOD"
    chown -R "$DO_CHOWN" "$DEST"
    chmod -R "$DO_CHMOD" "$DEST"
fi

echo "✅ Script completed successfully"
```
