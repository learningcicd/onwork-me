### build.pkr.hcl

```bash
#!/usr/bin/env bash
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

usage(){ echo "Usage: sudo bash $0 [--account <storageAccount>] [--container <container>]"; }

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

# Network connectivity check
check_connectivity() {
    echo "🔍 Checking network connectivity..."
    
    if ! curl -I https://www.microsoft.com --max-time 10 >/dev/null 2>&1; then
        echo "ERROR: No internet connectivity"
        exit 1
    fi
    
    if ! nslookup login.microsoftonline.com >/dev/null 2>&1; then
        echo "ERROR: Cannot resolve Azure DNS names"
        exit 1
    fi
    
    echo "✅ Network connectivity OK"
}

check_connectivity

# Your Azure download logic here
echo "🔐 Downloading from Azure..."
echo "   Account: $ACCOUNT"
echo "   Container: $CONTAINER"
echo "   Destination: $DEST"

# ... rest of your download and setup logic ...

echo "✅ Download script completed"
```
