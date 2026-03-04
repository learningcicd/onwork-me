### build.pkr.hcl

```bash

#!/bin/bash

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ENV_FILE="$SCRIPT_DIR/.env"
VMSS_SERVICE_MAP_FILE="$SCRIPT_DIR/vmss_svc.json"

usage() {
	cat <<EOF
Usage: $0 [-o <output-file>]

Reads RESOURCE_GROUP and VMSS_NAMES from .env in the same directory as this script.
Reads manual VMSS-to-service mapping from vmss_svc.json (if present).

Fetches VMs in all configured VMSSes and outputs JSON grouped by zone.

Output format:
[
	{
		"vmss_name": "<vmss-name>",
		"service_name": "<service-name-or-empty>",
		"zone": "<zone>",
		"vms": [
			{
				"vm_name": "<vm-name>",
				"computer_name": "<computer-name>",
				"ip_address": "<private-or-public-ip>"
			}
		]
	}
]

Options:
	-o   Output file path (optional). Default: ./vmss-svc-mapping.json
	-h   Show this help message
EOF
}

OUTPUT_FILE="$SCRIPT_DIR/vmss-svc-mapping.json"

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
	if ! INSTANCES_JSON=$(az vmss list-instances \
		--resource-group "$RESOURCE_GROUP" \
		--name "$vmss_name" \
		-o json 2>&1); then
		if echo "$INSTANCES_JSON" | grep -qi "virtualmachineScaleset/virtualMachines"; then
			echo "Error: Missing Azure RBAC permission to read VMSS instances for VMSS '$vmss_name' in resource group '$RESOURCE_GROUP'." >&2
			echo "Grant Reader (or Virtual Machine Contributor/Contributor) on the VMSS or resource group and retry." >&2
		else
			echo "Error: Failed to fetch instances for VMSS '$vmss_name':" >&2
			echo "$INSTANCES_JSON" >&2
		fi
		exit 1
	fi

	if ! NICS_JSON=$(az vmss nic list \
		--resource-group "$RESOURCE_GROUP" \
		--vmss-name "$vmss_name" \
		-o json 2>&1); then
		if echo "$NICS_JSON" | grep -qi "virtualmachineScalesets/networkinterfaces\|virtualmachineScaleset/virtualMachines"; then
			echo "Error: Missing Azure RBAC permission to read VMSS NICs for VMSS '$vmss_name' in resource group '$RESOURCE_GROUP'." >&2
			echo "Grant Reader (or Virtual Machine Contributor/Contributor) on the VMSS or resource group and retry." >&2
		else
			echo "Error: Failed to fetch NICs for VMSS '$vmss_name':" >&2
			echo "$NICS_JSON" >&2
		fi
		exit 1
	fi

	if [[ -z "$INSTANCES_JSON" || "$INSTANCES_JSON" == "[]" ]]; then
		continue
	fi

	$PYTHON_BIN - "$vmss_name" "$INSTANCES_JSON" "$NICS_JSON" >> "$TMP_ROWS_FILE" <<'PY'
import json
import sys

vmss_name = sys.argv[1]
instances_json = sys.argv[2]
nics_json = sys.argv[3]

instances = json.loads(instances_json)
nics = json.loads(nics_json)

ip_by_instance = {}
for nic in nics:
    vm_ref = (nic.get("virtualMachine") or {}).get("id", "")
    if not vm_ref:
        continue
    instance_id = vm_ref.rstrip("/").split("/")[-1]
    ip = ""
    ip_configs = nic.get("ipConfigurations") or []
    if ip_configs:
        ip = ip_configs[0].get("privateIpAddress") or ""
    if instance_id and ip:
        ip_by_instance[instance_id] = ip

for item in instances:
    instance_id = str(item.get("instanceId", ""))
    vm_name = item.get("name") or ""
    computer_name = ((item.get("osProfile") or {}).get("computerName")) or vm_name
    zones = item.get("zones") or []
    zone = str(zones[0]) if zones else "no-zone"
    ip = ip_by_instance.get(instance_id, "")

	if not ip:
		print(
			f"Warning: IP lookup failed for VM '{vm_name}' (computer_name '{computer_name}', instance '{instance_id}', VMSS '{vmss_name}').",
			file=sys.stderr,
		)

    print(f"{vmss_name}\t{zone}\t{vm_name}\t{computer_name}\t{ip}")
PY
done

RESULT_JSON=$($PYTHON_BIN - "$TMP_ROWS_FILE" "$VMSS_SERVICE_MAP_FILE" <<'PY'
import json
import sys
from collections import defaultdict

rows_file = sys.argv[1]
service_map_file = sys.argv[2]

grouped = defaultdict(lambda: defaultdict(list))
service_name_map = {}

try:
	with open(service_map_file, "r", encoding="utf-8") as fh:
		payload = json.load(fh)

	for item in payload.get("vmss_service_map", []):
		vmss = item.get("vmss_name")
		service = item.get("service_name", "")
		if vmss and vmss not in service_name_map:
			service_name_map[vmss] = service
except FileNotFoundError:
	pass
except Exception:
	pass

with open(rows_file, "r", encoding="utf-8") as fh:
		for line in fh:
				line = line.rstrip("\n")
				if not line:
						continue
				parts = line.split("\t")
				if len(parts) < 5:
						continue
				vmss_name, zone, vm_name, computer_name, ip_address = parts[0], parts[1], parts[2], parts[3], parts[4]
				grouped[vmss_name][zone].append(
						{
								"vm_name": vm_name,
								"computer_name": computer_name,
								"ip_address": ip_address,
						}
				)

output = []
for vmss_name in sorted(grouped.keys()):
		zones = grouped[vmss_name]
		for zone in sorted(zones.keys(), key=lambda value: (value == "no-zone", value)):
				zone_vms = sorted(zones[zone], key=lambda item: item.get("vm_name", ""))
				output.append(
						{
								"vmss_name": vmss_name,
								"service_name": service_name_map.get(vmss_name, ""),
								"zone": zone,
								"vms": zone_vms,
						}
				)

print(json.dumps(output, indent=2))
PY
)

printf '%s\n' "$RESULT_JSON" > "$OUTPUT_FILE"
echo "JSON written to $OUTPUT_FILE"






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
VMSS_SERVICE_MAP_FILE="$SCRIPT_DIR/vmss_svc.json"

usage() {
	cat <<EOF
Usage: $0 [-o <output-file>]

Reads RESOURCE_GROUP and VMSS_NAMES from .env in the same directory as this script.
Reads manual VMSS-to-service mapping from vmss_svc.json (if present).

Fetches VMs in all configured VMSSes and outputs JSON grouped by zone.

Output format:
[
	{
		"vmss_name": "<vmss-name>",
		"service_name": "<service-name-or-empty>",
		"zone": "<zone>",
		"vms": ["<vm-name-1>", "<vm-name-2>"]
	}
]

Options:
	-o   Output file path (optional). Default: ./vmss-svc-mapping.json
	-h   Show this help message
EOF
}

OUTPUT_FILE="$SCRIPT_DIR/vmss-svc-mapping.json"

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

RESULT_JSON=$($PYTHON_BIN - "$TMP_ROWS_FILE" "$VMSS_SERVICE_MAP_FILE" <<'PY'
import json
import sys
from collections import defaultdict

rows_file = sys.argv[1]
service_map_file = sys.argv[2]

grouped = defaultdict(lambda: defaultdict(list))
service_name_map = {}

try:
	with open(service_map_file, "r", encoding="utf-8") as fh:
		payload = json.load(fh)

	for item in payload.get("vmss_service_map", []):
		vmss = item.get("vmss_name")
		service = item.get("service_name", "")
		if vmss and vmss not in service_name_map:
			service_name_map[vmss] = service
except FileNotFoundError:
	pass
except Exception:
	pass

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
								"service_name": service_name_map.get(vmss_name, ""),
								"zone": zone,
								"vms": sorted(zones[zone]),
						}
				)

print(json.dumps(output, indent=2))
PY
)

printf '%s\n' "$RESULT_JSON" > "$OUTPUT_FILE"
echo "JSON written to $OUTPUT_FILE"

		
```

```
bash -lc 'source ./.env; for vmss in "${VMSS_NAMES[@]}"; do az vmss list-instances -g "$RESOURCE_GROUP" -n "$vmss" --query "[].{vmss_name:'"$vmss"',zone:zones[0],vm_name:name}" -o json; done | python3 -c "import sys,json,collections; out=[]; d=collections.defaultdict(lambda: collections.defaultdict(list));\n[ d[i.get(\"vmss_name\")][str(i.get(\"zone\") or \"no-zone\")].append(i.get(\"vm_name\")) for arr in [json.loads(x) for x in sys.stdin.read().splitlines() if x.strip()] for i in arr if i.get(\"vm_name\") ];\n[ out.append({\"vmss_name\":v,\"zone\":z,\"vms\":sorted(n)}) for v in sorted(d) for z,n in sorted(d[v].items(), key=lambda kv:(kv[0]==\"no-zone\",kv[0])) ];\nprint(json.dumps(out,indent=2))"'
```

## TESTIN

```
#!/bin/bash

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ENV_FILE="$SCRIPT_DIR/.env"
MAPPING_FILE="${VMSS_MAPPING_FILE:-$SCRIPT_DIR/vmss-svc-mapping.json}"
DELAY_SECONDS=45

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

log_info() {
  echo -e "${GREEN}[INFO]${NC} $1"
}

log_warning() {
  echo -e "${YELLOW}[WARN]${NC} $1"
}

log_error() {
  echo -e "${RED}[ERROR]${NC} $1"
}

require_command() {
  local cmd="$1"
  if ! command -v "$cmd" >/dev/null 2>&1; then
    log_error "Required command not found: $cmd"
    exit 1
  fi
}

load_env() {
  if [[ ! -f "$ENV_FILE" ]]; then
    log_error ".env file not found at $ENV_FILE"
    exit 1
  fi

  source "$ENV_FILE"

  RESOURCE_GROUP="${RESOURCE_GROUP:-}"
  SSH_USERNAME="${SSH_USERNAME:-}"
  SSH_PASSWORD="${SSH_PASSWORD:-}"
  AUTH_METHOD="${AUTH_METHOD:-interactive}"
  AZURE_CLIENT_ID="${AZURE_CLIENT_ID:-}"
  AZURE_CLIENT_SECRET="${AZURE_CLIENT_SECRET:-}"
  AZURE_TENANT_ID="${AZURE_TENANT_ID:-}"

  if [[ -z "$RESOURCE_GROUP" ]]; then
    log_error "RESOURCE_GROUP is required in .env"
    exit 1
  fi

  if [[ -z "$SSH_USERNAME" || -z "$SSH_PASSWORD" ]]; then
    log_error "SSH_USERNAME and SSH_PASSWORD are required in .env"
    exit 1
  fi
}

authenticate_azure() {
  if az account show >/dev/null 2>&1; then
    log_info "Azure CLI is already authenticated."
    return 0
  fi

  if [[ "$AUTH_METHOD" == "service-principal" ]]; then
    if [[ -z "$AZURE_CLIENT_ID" || -z "$AZURE_CLIENT_SECRET" || -z "$AZURE_TENANT_ID" ]]; then
      log_error "AUTH_METHOD=service-principal but AZURE_CLIENT_ID/AZURE_CLIENT_SECRET/AZURE_TENANT_ID are missing."
      exit 1
    fi

    log_info "Authenticating with Azure service principal..."
    az login --service-principal \
      --username "$AZURE_CLIENT_ID" \
      --password "$AZURE_CLIENT_SECRET" \
      --tenant "$AZURE_TENANT_ID" >/dev/null
    return 0
  fi

  log_info "Authenticating with Azure interactively..."
  az login >/dev/null
}

prompt_zone() {
  local zone_input
  read -r -p "Enter Azure zone number to process (e.g., 1, 2, 3): " zone_input

  if [[ -z "$zone_input" ]]; then
    log_error "Zone input cannot be empty."
    exit 1
  fi

  SELECTED_ZONE="$zone_input"
  log_info "Selected zone: $SELECTED_ZONE"
}

get_zone_rows() {
  local python_bin="python3"
  if ! command -v python3 >/dev/null 2>&1; then
    python_bin="python"
  fi

  "$python_bin" - "$MAPPING_FILE" "$SELECTED_ZONE" <<'PY'
import json
import sys

path = sys.argv[1]
zone = str(sys.argv[2])

with open(path, "r", encoding="utf-8") as f:
    payload = json.load(f)

if not isinstance(payload, list):
    raise SystemExit("Mapping file must contain a JSON array")

for item in payload:
    if str(item.get("zone", "")) != zone:
        continue

    vmss_name = item.get("vmss_name", "")
    service_name = item.get("service_name", "")
    vms = item.get("vms", [])

    if not vmss_name or not service_name or not isinstance(vms, list):
        continue

    for vm_name in vms:
        if vm_name:
            print(f"{vmss_name}\t{service_name}\t{vm_name}")
PY
}

resolve_vm_ip() {
  local vm_name="$1"

  local ip
  ip=$(az vm show \
    --resource-group "$RESOURCE_GROUP" \
    --name "$vm_name" \
    --show-details \
    --query "privateIps" \
    -o tsv 2>/dev/null || true)

  if [[ -n "$ip" && "$ip" != "null" ]]; then
    echo "$ip"
    return 0
  fi

  ip=$(az vm show \
    --resource-group "$RESOURCE_GROUP" \
    --name "$vm_name" \
    --show-details \
    --query "publicIps" \
    -o tsv 2>/dev/null || true)

  if [[ -n "$ip" && "$ip" != "null" ]]; then
    echo "$ip"
    return 0
  fi

  echo ""
}

restart_service_over_ssh() {
  local vm_name="$1"
  local vmss_name="$2"
  local service_name="$3"

  local vm_ip
  vm_ip="$(resolve_vm_ip "$vm_name")"

  if [[ -z "$vm_ip" ]]; then
    log_error "Could not resolve IP for VM '$vm_name' (VMSS '$vmss_name')."
    return 1
  fi

  log_info "Restarting ${service_name}.service on VM '$vm_name' ($vm_ip), VMSS '$vmss_name'"

  if sshpass -p "$SSH_PASSWORD" ssh \
      -o StrictHostKeyChecking=no \
      -o UserKnownHostsFile=/dev/null \
      -o ConnectTimeout=15 \
      "$SSH_USERNAME@$vm_ip" \
      "sudo systemctl restart ${service_name}.service && sudo systemctl is-active --quiet ${service_name}.service"; then
    log_info "Service restart successful: $vm_name -> ${service_name}.service"
    return 0
  fi

  log_error "Service restart failed on VM '$vm_name' for ${service_name}.service"
  return 1
}

main() {
  require_command az
  require_command ssh
  require_command sshpass

  if ! command -v python3 >/dev/null 2>&1 && ! command -v python >/dev/null 2>&1; then
    log_error "Python is required (python3 or python) to parse mapping file."
    exit 1
  fi

  load_env

  if [[ ! -f "$MAPPING_FILE" ]]; then
    log_error "Mapping file not found: $MAPPING_FILE"
    log_warning "Generate it first using restart-svc.sh (output: vmss-svc-mapping.json)."
    exit 1
  fi

  authenticate_azure
  prompt_zone

  mapfile -t zone_rows < <(get_zone_rows)

  if [[ ${#zone_rows[@]} -eq 0 ]]; then
    log_warning "No VMs found in zone '$SELECTED_ZONE' with valid vmss_name/service_name in $MAPPING_FILE"
    exit 0
  fi

  log_info "Found ${#zone_rows[@]} VM(s) to process in zone '$SELECTED_ZONE'."

  local index=1
  local total=${#zone_rows[@]}

  for row in "${zone_rows[@]}"; do
    IFS=$'\t' read -r vmss_name service_name vm_name <<< "$row"

    log_info "[$index/$total] VMSS='$vmss_name' VM='$vm_name' Service='${service_name}.service'"

    if ! restart_service_over_ssh "$vm_name" "$vmss_name" "$service_name"; then
      log_error "Stopping due to failure on VM '$vm_name'."
      exit 1
    fi

    if [[ $index -lt $total ]]; then
      log_info "Waiting ${DELAY_SECONDS}s before next VM..."
      sleep "$DELAY_SECONDS"
    fi

    index=$((index + 1))
  done

  log_info "Completed service restarts for zone '$SELECTED_ZONE'."
}

main
```

```
RG="<RESOURCE_GROUP>"
VMSS="<VMSS_NAME>"

echo -e "VMSS\tInstanceID\tComputerName\tPrivateIP\tZone"

instances=$(az vmss list-instances -g $RG -n $VMSS -o json)
nics=$(az vmss nic list -g $RG --vmss-name $VMSS -o json)

echo "$instances" | jq -c '.[]' | while read vm
do
    instance=$(echo $vm | jq -r '.instanceId')
    computer=$(echo $vm | jq -r '.osProfile.computerName')
    zone=$(echo $vm | jq -r '.zones[0] // "NA"')

    ip=$(echo "$nics" | jq -r ".[] | select(.virtualMachine.id | endswith(\"/$instance\")) | .ipConfigurations[0].privateIpAddress")

    printf "%-10s %-10s %-15s %-15s %-5s\n" "$VMSS" "$instance" "$computer" "$ip" "$zone"
done
```
