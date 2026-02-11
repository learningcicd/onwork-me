### build.pkr.hcl

```bash
#!/bin/bash

# Script to restart Azure VMSS sequentially (A -> B -> C)
# Each VMSS restarts 1 node at a time

set -e

# Get the directory where the script is located
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# Load environment variables from .env file if it exists
if [ -f "$SCRIPT_DIR/.env" ]; then
    # Source .env file, but don't fail if it doesn't exist
    source "$SCRIPT_DIR/.env"
else
    echo "Warning: .env file not found at $SCRIPT_DIR/.env"
    echo "Please create a .env file with your Azure credentials."
fi

# Configuration
RESOURCE_GROUP="${RESOURCE_GROUP:-your-resource-group}"
# Set defaults if not defined in .env
if [ -z "${VMSS_NAMES+x}" ]; then
    VMSS_NAMES=("A" "B" "C")
fi
DELAY_BETWEEN_RESTARTS="${DELAY_BETWEEN_RESTARTS:-30}"  # seconds to wait between restarting nodes
MAX_WAIT_SECONDS="${MAX_WAIT_SECONDS:-600}"  # max wait for instance to be running

# Azure Service Principal Configuration (optional)
# Set these in .env file or as environment variables:
# AZURE_CLIENT_ID, AZURE_CLIENT_SECRET, AZURE_TENANT_ID
AZURE_CLIENT_ID="${AZURE_CLIENT_ID:-}"
AZURE_CLIENT_SECRET="${AZURE_CLIENT_SECRET:-}"
AZURE_TENANT_ID="${AZURE_TENANT_ID:-}"
AUTH_METHOD="${AUTH_METHOD:-interactive}"  # "interactive" or "service-principal"

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Function to print colored output
log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

log_warning() {
    echo -e "${YELLOW}[WARNING]${NC} $1"
}

# Function to check if Azure CLI is installed
check_az_cli() {
    if ! command -v az &> /dev/null; then
        log_error "Azure CLI is not installed. Please install it first."
        exit 1
    fi
    log_info "Azure CLI is installed."
}

# Function to authenticate with Azure using Service Principal
authenticate_service_principal() {
    log_info "Authenticating with Azure using Service Principal..."
    
    if [ -z "$AZURE_CLIENT_ID" ] || [ -z "$AZURE_CLIENT_SECRET" ] || [ -z "$AZURE_TENANT_ID" ]; then
        log_error "Service principal credentials are incomplete."
        log_error "Please set AZURE_CLIENT_ID, AZURE_CLIENT_SECRET, and AZURE_TENANT_ID environment variables."
        return 1
    fi
    
    if az login --service-principal -u "$AZURE_CLIENT_ID" -p "$AZURE_CLIENT_SECRET" --tenant "$AZURE_TENANT_ID" &> /dev/null; then
        log_info "Successfully authenticated with Service Principal"
        return 0
    else
        log_error "Failed to authenticate with Service Principal"
        return 1
    fi
}

# Function to authenticate with Azure (interactive or service principal)
authenticate_azure() {
    if [ "$AUTH_METHOD" == "service-principal" ]; then
        if ! authenticate_service_principal; then
            return 1
        fi
    else
        log_info "Authenticating with Azure interactively..."
        if ! az login &> /dev/null; then
            log_error "Failed to authenticate. Please run 'az login' manually."
            return 1
        fi
        log_info "Successfully authenticated interactively"
    fi
    
    # Verify authentication
    if ! az account show &> /dev/null; then
        log_error "Failed to verify Azure authentication"
        return 1
    fi
    
    return 0
}

# Function to validate VMSS exists
validate_vmss() {
    local vmss_name=$1
    if ! az vmss show --name "$vmss_name" --resource-group "$RESOURCE_GROUP" &> /dev/null; then
        log_error "VMSS '$vmss_name' not found in resource group '$RESOURCE_GROUP'"
        return 1
    fi
    return 0
}

# Function to get instance IDs for a VMSS
get_instance_records() {
    local vmss_name=$1
    local instances=$(az vmss list-instances --name "$vmss_name" --resource-group "$RESOURCE_GROUP" --query "[].{id:instanceId,name:name}" -o tsv)
    echo "$instances"
}

# Function to restart a single instance
restart_instance() {
    local vmss_name=$1
    local instance_id=$2
    local instance_name=$3

    if [[ "$instance_id" =~ ^[0-9]+$ ]]; then
        log_info "Restarting instance $instance_id in VMSS '$vmss_name'..."

        if az vmss restart --name "$vmss_name" --instance-ids "$instance_id" --resource-group "$RESOURCE_GROUP"; then
            log_info "Successfully restarted instance $instance_id"
            return 0
        fi
        log_error "Failed to restart instance $instance_id"
        return 1
    fi

    log_info "Restarting instance $instance_name (flexible) in VMSS '$vmss_name'..."

    if az vm restart --name "$instance_name" --resource-group "$RESOURCE_GROUP"; then
        log_info "Successfully restarted instance $instance_name"
        return 0
    fi

    log_error "Failed to restart instance $instance_name"
    return 1
}

# Function to wait for instance to be running
wait_for_instance() {
    local vmss_name=$1
    local instance_id=$2
    local instance_name=$3
    local max_wait=$MAX_WAIT_SECONDS
    local elapsed=0

    local label="$instance_id"
    if ! [[ "$instance_id" =~ ^[0-9]+$ ]]; then
        label="$instance_name"
    fi

    log_info "Waiting for instance $label to be running..."

    while [ $elapsed -lt $max_wait ]; do
        local state=""
        local prov=""

        if [[ "$instance_id" =~ ^[0-9]+$ ]]; then
            state=$(az vmss get-instance-view --name "$vmss_name" --resource-group "$RESOURCE_GROUP" --instance-id "$instance_id" --query "statuses[?starts_with(code, 'PowerState/')].code | [0]" -o tsv)
            prov=$(az vmss get-instance-view --name "$vmss_name" --resource-group "$RESOURCE_GROUP" --instance-id "$instance_id" --query "statuses[?starts_with(code, 'ProvisioningState/')].code | [0]" -o tsv)
        else
            state=$(az vm get-instance-view --name "$instance_name" --resource-group "$RESOURCE_GROUP" --query "instanceView.statuses[?starts_with(code, 'PowerState/')].code | [0]" -o tsv)
            prov=$(az vm get-instance-view --name "$instance_name" --resource-group "$RESOURCE_GROUP" --query "instanceView.statuses[?starts_with(code, 'ProvisioningState/')].code | [0]" -o tsv)
        fi

        if [ -n "$state" ] || [ -n "$prov" ]; then
            log_info "Current state: power=$state provisioning=$prov"
            if [ "$state" = "PowerState/running" ]; then
                log_info "Instance $label is running"
                return 0
            fi
        else
            log_warning "Power state not available yet for instance $label"
        fi

        sleep 10
        elapsed=$((elapsed + 10))
    done

    log_warning "Instance $label did not reach running state within timeout period"
    return 1
}

# Function to restart a single VMSS (one instance at a time)
restart_vmss() {
    local vmss_name=$1
    
    log_info "========================================"
    log_info "Starting restart of VMSS: $vmss_name"
    log_info "========================================"
    
    # Validate VMSS exists
    if ! validate_vmss "$vmss_name"; then
        return 1
    fi
    
    # Get all instance IDs
    local instance_records=$(get_instance_records "$vmss_name")
    
    if [ -z "$instance_records" ]; then
        log_error "No instances found in VMSS '$vmss_name'"
        return 1
    fi
    
    # Count instances
    local instance_count=$(echo "$instance_records" | wc -l)
    log_info "Found $instance_count instances in VMSS '$vmss_name'"
    
    # Restart each instance one at a time
    local instance_number=1
    while read -r instance_id instance_name; do
        local label="$instance_id"
        if ! [[ "$instance_id" =~ ^[0-9]+$ ]]; then
            label="$instance_name"
        fi

        log_info "[Instance $instance_number/$instance_count] Processing instance: $label"

        if restart_instance "$vmss_name" "$instance_id" "$instance_name"; then
            # Wait for instance to be running before moving to next one
            wait_for_instance "$vmss_name" "$instance_id" "$instance_name"
            
            # Add delay between restarts (except for the last instance)
            if [ $instance_number -lt $instance_count ]; then
                log_info "Waiting $DELAY_BETWEEN_RESTARTS seconds before restarting next instance..."
                sleep $DELAY_BETWEEN_RESTARTS
            fi
        else
            log_error "Failed to restart instance $label in VMSS '$vmss_name'. Aborting VMSS restart."
            return 1
        fi
        
        instance_number=$((instance_number + 1))
    done <<< "$instance_records"
    
    log_info "========================================"
    log_info "VMSS '$vmss_name' restart completed successfully"
    log_info "========================================"
    return 0
}

# Main script
main() {
    log_info "Azure VMSS Sequential Restart Script"
    log_info "Resource Group: $RESOURCE_GROUP"
    log_info "VMSSs to restart: ${VMSS_NAMES[@]}"
    
    # Validate prerequisites
    check_az_cli
    
    # Authenticate with Azure
    if ! authenticate_azure; then
        exit 1
    fi
    
    # Restart each VMSS sequentially
    local failed_vmss=()
    
    for vmss_name in "${VMSS_NAMES[@]}"; do
        if ! restart_vmss "$vmss_name"; then
            failed_vmss+=("$vmss_name")
        fi
        
        # Add delay between VMSS restarts (except for the last one)
        if [ "$vmss_name" != "${VMSS_NAMES[-1]}" ]; then
            log_info "Waiting 60 seconds before starting next VMSS restart..."
            sleep 60
        fi
    done
    
    # Summary
    log_info ""
    log_info "========================================"
    log_info "RESTART SUMMARY"
    log_info "========================================"
    
    if [ ${#failed_vmss[@]} -eq 0 ]; then
        log_info "All VMSSs restarted successfully!"
        exit 0
    else
        log_error "The following VMSSs encountered errors: ${failed_vmss[@]}"
        exit 1
    fi
}

# Run main function
main

```

# GitLab Configuration
# Azure Configuration
RESOURCE_GROUP=""
VMSS_NAMES=("A" "B")
DELAY_BETWEEN_RESTARTS=30

# Authentication Method: "interactive" or "service-principal"
AUTH_METHOD="service-principal"

# Azure Service Principal Credentials (required for service-principal auth)
AZURE_CLIENT_ID=""
AZURE_CLIENT_SECRET=""
AZURE_TENANT_ID=""
