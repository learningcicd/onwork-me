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

# Configuration (must be provided in .env or environment)
RESOURCE_GROUP="${RESOURCE_GROUP:-}"
VMSS_NAMES="${VMSS_NAMES:-}"
DELAY_BETWEEN_RESTARTS="${DELAY_BETWEEN_RESTARTS:-}"
MAX_WAIT_SECONDS="${MAX_WAIT_SECONDS:-}"
VMSS_SERVICE_MAP_FILE="${VMSS_SERVICE_MAP_FILE:-$SCRIPT_DIR/vmss_svc.json}"
HEALTH_CHECK_INTERVAL_SECONDS="${HEALTH_CHECK_INTERVAL_SECONDS:-15}"

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

# Function to get mapped systemd service name for a VMSS from vmss_svc.json
get_service_name_for_vmss() {
    local vmss_name=$1

    if [ ! -f "$VMSS_SERVICE_MAP_FILE" ]; then
        log_warning "VMSS service map file not found: $VMSS_SERVICE_MAP_FILE"
        echo ""
        return 0
    fi

    local python_bin=""
    if command -v python3 &> /dev/null; then
        python_bin="python3"
    elif command -v python &> /dev/null; then
        python_bin="python"
    else
        log_error "Python is required to parse $VMSS_SERVICE_MAP_FILE but was not found"
        return 1
    fi

    local service_name
    service_name=$($python_bin - "$VMSS_SERVICE_MAP_FILE" "$vmss_name" <<'PY'
import json
import sys

path = sys.argv[1]
target_vmss = sys.argv[2]

try:
    with open(path, "r", encoding="utf-8") as f:
        payload = json.load(f)
except Exception:
    print("")
    raise

for item in payload.get("vmss_service_map", []):
    if item.get("vmss_name") == target_vmss:
        print(item.get("service_name", ""))
        break
else:
    print("")
PY
)

    echo "$service_name"
}

# Function to run systemctl action (stop/start) for mapped service on an instance
run_service_command_on_instance() {
    local vmss_name=$1
    local instance_id=$2
    local instance_name=$3
    local service_name=$4
    local action=$5

    if [ -z "$service_name" ]; then
        return 0
    fi

    if [ "$action" != "stop" ] && [ "$action" != "start" ]; then
        log_error "Invalid service action: $action"
        return 1
    fi

    log_info "Running 'systemctl $action ${service_name}.service' on instance..."

    local command="systemctl $action ${service_name}.service"

    if [[ "$instance_id" =~ ^[0-9]+$ ]]; then
        if az vmss run-command invoke \
            --resource-group "$RESOURCE_GROUP" \
            --name "$vmss_name" \
            --instance-id "$instance_id" \
            --command-id RunShellScript \
            --scripts "$command" &> /dev/null; then
            log_info "Successfully ran service $action on instance $instance_id"
            return 0
        fi

        log_error "Failed to run service $action on instance $instance_id"
        return 1
    fi

    if az vm run-command invoke \
        --resource-group "$RESOURCE_GROUP" \
        --name "$instance_name" \
        --command-id RunShellScript \
        --scripts "$command" &> /dev/null; then
        log_info "Successfully ran service $action on instance $instance_name"
        return 0
    fi

    log_error "Failed to run service $action on instance $instance_name"
    return 1
}

# Function to wait until mapped service is active on an instance
wait_for_service_active_on_instance() {
    local vmss_name=$1
    local instance_id=$2
    local instance_name=$3
    local service_name=$4
    local max_wait=$MAX_WAIT_SECONDS
    local elapsed=0

    if [ -z "$service_name" ]; then
        return 0
    fi

    local label="$instance_id"
    if ! [[ "$instance_id" =~ ^[0-9]+$ ]]; then
        label="$instance_name"
    fi

    log_info "Waiting for ${service_name}.service to be active on instance $label..."

    while [ $elapsed -lt $max_wait ]; do
        local command="systemctl is-active --quiet ${service_name}.service"

        if [[ "$instance_id" =~ ^[0-9]+$ ]]; then
            if az vmss run-command invoke \
                --resource-group "$RESOURCE_GROUP" \
                --name "$vmss_name" \
                --instance-id "$instance_id" \
                --command-id RunShellScript \
                --scripts "$command" &> /dev/null; then
                log_info "Service ${service_name}.service is active on instance $label"
                return 0
            fi
        else
            if az vm run-command invoke \
                --resource-group "$RESOURCE_GROUP" \
                --name "$instance_name" \
                --command-id RunShellScript \
                --scripts "$command" &> /dev/null; then
                log_info "Service ${service_name}.service is active on instance $label"
                return 0
            fi
        fi

        sleep "$HEALTH_CHECK_INTERVAL_SECONDS"
        elapsed=$((elapsed + HEALTH_CHECK_INTERVAL_SECONDS))
    done

    log_error "Service ${service_name}.service did not become active on instance $label within timeout"
    return 1
}

# Function to wait for Azure instance health state to become healthy, if available
wait_for_instance_health() {
    local vmss_name=$1
    local instance_id=$2
    local instance_name=$3
    local max_wait=$MAX_WAIT_SECONDS
    local elapsed=0

    local label="$instance_id"
    if ! [[ "$instance_id" =~ ^[0-9]+$ ]]; then
        label="$instance_name"
    fi

    log_info "Waiting for instance $label health state to become healthy..."

    while [ $elapsed -lt $max_wait ]; do
        local health_state=""

        if [[ "$instance_id" =~ ^[0-9]+$ ]]; then
            health_state=$(az vmss get-instance-view \
                --name "$vmss_name" \
                --resource-group "$RESOURCE_GROUP" \
                --instance-id "$instance_id" \
                --query "statuses[?starts_with(code, 'HealthState/')].displayStatus | [0]" \
                -o tsv 2> /dev/null || true)
        else
            health_state=$(az vm get-instance-view \
                --name "$instance_name" \
                --resource-group "$RESOURCE_GROUP" \
                --query "instanceView.statuses[?starts_with(code, 'HealthState/')].displayStatus | [0]" \
                -o tsv 2> /dev/null || true)
        fi

        if [ -z "$health_state" ] || [ "$health_state" = "None" ]; then
            log_warning "HealthState not reported for instance $label; skipping health wait"
            return 0
        fi

        log_info "Current health state for instance $label: $health_state"
        if echo "$health_state" | grep -qi "healthy"; then
            log_info "Instance $label reported healthy"
            return 0
        fi

        sleep "$HEALTH_CHECK_INTERVAL_SECONDS"
        elapsed=$((elapsed + HEALTH_CHECK_INTERVAL_SECONDS))
    done

    log_error "Instance $label did not reach healthy state within timeout"
    return 1
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
    local service_name=""
    
    log_info "========================================"
    log_info "Starting restart of VMSS: $vmss_name"
    log_info "========================================"
    
    # Validate VMSS exists
    if ! validate_vmss "$vmss_name"; then
        return 1
    fi

    service_name=$(get_service_name_for_vmss "$vmss_name")
    if [ -n "$service_name" ]; then
        log_info "Mapped app service for VMSS '$vmss_name': ${service_name}.service"
    else
        log_warning "No mapped app service found for VMSS '$vmss_name'; proceeding without systemctl stop/start"
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

        if [ -n "$service_name" ]; then
            if ! run_service_command_on_instance "$vmss_name" "$instance_id" "$instance_name" "$service_name" "stop"; then
                log_error "Failed to gracefully stop ${service_name}.service before restart. Aborting VMSS restart."
                return 1
            fi
        fi

        if restart_instance "$vmss_name" "$instance_id" "$instance_name"; then
            # Wait for instance to be running before moving to next one
            if ! wait_for_instance "$vmss_name" "$instance_id" "$instance_name"; then
                log_error "Instance $label failed to reach running state. Aborting VMSS restart."
                return 1
            fi

            if [ -n "$service_name" ]; then
                if ! run_service_command_on_instance "$vmss_name" "$instance_id" "$instance_name" "$service_name" "start"; then
                    log_error "Failed to gracefully start ${service_name}.service after restart. Aborting VMSS restart."
                    return 1
                fi

                if ! wait_for_service_active_on_instance "$vmss_name" "$instance_id" "$instance_name" "$service_name"; then
                    log_error "Service ${service_name}.service did not become active after restart. Aborting VMSS restart."
                    return 1
                fi
            fi

            if ! wait_for_instance_health "$vmss_name" "$instance_id" "$instance_name"; then
                log_error "Instance $label remained unhealthy after restart. Aborting VMSS restart."
                return 1
            fi
            
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

    if [ -z "$RESOURCE_GROUP" ] || [ -z "$VMSS_NAMES" ] || [ -z "$DELAY_BETWEEN_RESTARTS" ] || [ -z "$MAX_WAIT_SECONDS" ]; then
        log_error "Missing required configuration. Please set RESOURCE_GROUP, VMSS_NAMES, DELAY_BETWEEN_RESTARTS, and MAX_WAIT_SECONDS in .env."
        exit 1
    fi

    read -r -a VMSS_NAMES <<< "$VMSS_NAMES"

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

## ENV

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
```

## vmss
```
{
	"vmss_service_map": [
		{
			"vmss_name": "swldjmss",
			"service_name": "commpjms"
		}
	]
}
```

## Restart SVC

```
#!/bin/bash

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ENV_FILE="$SCRIPT_DIR/.env"

usage() {
	cat <<EOF
Usage: $0 [-o <output-file>]

Reads RESOURCE_GROUP and VMSS_NAMES from .env in the same directory as this script.

Fetches VMs in all configured VMSSes and outputs JSON grouped by zone.

Output format:
[
	{
		"vmss_name": "<vmss-name>",
		"zone": "<zone>",
		"vms": ["<vm-name-1>", "<vm-name-2>"]
	}
]

Options:
	-o   Output file path (optional). If omitted, prints to stdout.
	-h   Show this help message
EOF
}

OUTPUT_FILE=""

while getopts ":o:h" opt; do
	case "$opt" in
		o) OUTPUT_FILE="$OPTARG" ;;
		h)
			usage
			exit 0
			;;
		:)
			echo "Error: Option -$OPTARG requires an argument." >&2
			usage
			exit 1
			;;
		\?)
			echo "Error: Invalid option -$OPTARG" >&2
			usage
			exit 1
			;;
	esac
done

if [[ ! -f "$ENV_FILE" ]]; then
	echo "Error: .env file not found at $ENV_FILE" >&2
	exit 1
fi

source "$ENV_FILE"

RESOURCE_GROUP="${RESOURCE_GROUP:-}"

if [[ -z "$RESOURCE_GROUP" ]]; then
	echo "Error: RESOURCE_GROUP is missing in .env" >&2
	exit 1
fi

if ! command -v az >/dev/null 2>&1; then
	echo "Error: Azure CLI 'az' is not installed or not in PATH." >&2
	exit 1
fi

if ! command -v python3 >/dev/null 2>&1 && ! command -v python >/dev/null 2>&1; then
	echo "Error: Python is required but was not found in PATH." >&2
	exit 1
fi

if ! az account show >/dev/null 2>&1; then
	echo "Error: Azure CLI is not authenticated. Run 'az login' first." >&2
	exit 1
fi

VMSS_LIST=()
if declare -p VMSS_NAMES >/dev/null 2>&1 && declare -p VMSS_NAMES 2>/dev/null | grep -q "declare -a"; then
	VMSS_LIST=("${VMSS_NAMES[@]}")
else
	VMSS_NAMES_STRING="${VMSS_NAMES:-}"
	VMSS_NAMES_STRING="${VMSS_NAMES_STRING//,/ }"
	read -r -a VMSS_LIST <<< "$VMSS_NAMES_STRING"
fi

FILTERED_VMSS_LIST=()
for vmss in "${VMSS_LIST[@]}"; do
	if [[ -n "$vmss" ]]; then
		FILTERED_VMSS_LIST+=("$vmss")
	fi
done

if [[ ${#FILTERED_VMSS_LIST[@]} -eq 0 ]]; then
	echo "Error: VMSS_NAMES is empty in .env" >&2
	exit 1
fi

PYTHON_BIN="python3"
if ! command -v python3 >/dev/null 2>&1; then
	PYTHON_BIN="python"
fi

TMP_ROWS_FILE=$(mktemp)
trap 'rm -f "$TMP_ROWS_FILE"' EXIT

for vmss_name in "${FILTERED_VMSS_LIST[@]}"; do
	if ! INSTANCE_TSV=$(az vmss list-instances \
		--resource-group "$RESOURCE_GROUP" \
		--name "$vmss_name" \
		--query "[].{vm_name:name, zone:zones[0]}" \
		-o tsv 2>&1); then
		if echo "$INSTANCE_TSV" | grep -qi "virtualmachineScaleset/virtualMachines"; then
			echo "Error: Missing Azure RBAC permission 'Microsoft.Compute/virtualMachineScaleSets/virtualMachines/read' for VMSS '$vmss_name' in resource group '$RESOURCE_GROUP'." >&2
			echo "Grant Reader (or Virtual Machine Contributor/Contributor) on the VMSS or resource group and retry." >&2
		else
			echo "Error: Failed to fetch instances for VMSS '$vmss_name':" >&2
			echo "$INSTANCE_TSV" >&2
		fi
		exit 1
	fi

	if [[ -z "$INSTANCE_TSV" ]]; then
		continue
	fi

	while IFS=$'\t' read -r vm_name zone; do
		if [[ -z "$vm_name" ]]; then
			continue
		fi

		if [[ -z "$zone" ]]; then
			zone="no-zone"
		fi

		printf '%s\t%s\t%s\n' "$vmss_name" "$zone" "$vm_name" >> "$TMP_ROWS_FILE"
	done <<< "$INSTANCE_TSV"
done

RESULT_JSON=$($PYTHON_BIN - "$TMP_ROWS_FILE" <<'PY'
import json
import sys
from collections import defaultdict

rows_file = sys.argv[1]

grouped = defaultdict(lambda: defaultdict(list))

with open(rows_file, "r", encoding="utf-8") as fh:
		for line in fh:
				line = line.rstrip("\n")
				if not line:
						continue
				vmss_name, zone, vm_name = line.split("\t", 2)
				grouped[vmss_name][zone].append(vm_name)

output = []
for vmss_name in sorted(grouped.keys()):
		zones = grouped[vmss_name]
		for zone in sorted(zones.keys(), key=lambda value: (value == "no-zone", value)):
				output.append(
						{
								"vmss_name": vmss_name,
								"zone": zone,
								"vms": sorted(zones[zone]),
						}
				)

print(json.dumps(output, indent=2))
PY
)

if [[ -n "$OUTPUT_FILE" ]]; then
	printf '%s\n' "$RESULT_JSON" > "$OUTPUT_FILE"
	echo "JSON written to $OUTPUT_FILE"
else
	printf '%s\n' "$RESULT_JSON"
fi
```

```
bash -lc 'source ./.env; for vmss in "${VMSS_NAMES[@]}"; do az vmss list-instances -g "$RESOURCE_GROUP" -n "$vmss" --query "[].{vmss_name:'"$vmss"',zone:zones[0],vm_name:name}" -o json; done | python3 -c "import sys,json,collections; out=[]; d=collections.defaultdict(lambda: collections.defaultdict(list));\n[ d[i.get(\"vmss_name\")][str(i.get(\"zone\") or \"no-zone\")].append(i.get(\"vm_name\")) for arr in [json.loads(x) for x in sys.stdin.read().splitlines() if x.strip()] for i in arr if i.get(\"vm_name\") ];\n[ out.append({\"vmss_name\":v,\"zone\":z,\"vms\":sorted(n)}) for v in sorted(d) for z,n in sorted(d[v].items(), key=lambda kv:(kv[0]==\"no-zone\",kv[0])) ];\nprint(json.dumps(out,indent=2))"'
```
