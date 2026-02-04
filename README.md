### build.pkr.hcl

```bash
#!/bin/bash
set -e

# Source environment variables from .env file
if [ -f ".env" ]; then
    source .env
else
    echo "Error: .env file not found"
    exit 1
fi

# Validate required environment variables
if [ -z "$GITLAB_TOKEN" ]; then
    echo "Error: GITLAB_TOKEN not set in .env file"
    exit 1
fi

if [ -z "$GITLAB_PROJECT_ID" ]; then
    echo "Error: GITLAB_PROJECT_ID not set in .env file"
    exit 1
fi

if [ -z "$GITLAB_URL" ]; then
    GITLAB_URL="https://gitlab.com"
fi

if [ -z "$GITLAB_BRANCH" ]; then
    GITLAB_BRANCH="main"
fi

# Create .ssh directory if it doesn't exist
SSH_DIR="/home/oracle/.ssh"
sudo mkdir -p "$SSH_DIR"

# Download my_azure_key
echo "Downloading my_azure_key..."
curl -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
    "${GITLAB_URL}/api/v4/projects/${GITLAB_PROJECT_ID}/repository/files/my_azure_key/raw?ref=${GITLAB_BRANCH}" \
    -o /tmp/my_azure_key

if [ ! -f /tmp/my_azure_key ]; then
    echo "Error: Failed to download my_azure_key"
    exit 1
fi

# Download my_azure_key.pub
echo "Downloading my_azure_key.pub..."
curl -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
    "${GITLAB_URL}/api/v4/projects/${GITLAB_PROJECT_ID}/repository/files/my_azure_key.pub/raw?ref=${GITLAB_BRANCH}" \
    -o /tmp/my_azure_key.pub

if [ ! -f /tmp/my_azure_key.pub ]; then
    echo "Error: Failed to download my_azure_key.pub"
    exit 1
fi

# Copy files to destination with appropriate names
echo "Copying files to $SSH_DIR..."
sudo cp /tmp/my_azure_key "$SSH_DIR/my_azure_key"
sudo cp /tmp/my_azure_key.pub "$SSH_DIR/authorized_keys"

# Set ownership
echo "Setting ownership to oracle:oracle..."
sudo chown oracle:oracle "$SSH_DIR/my_azure_key"
sudo chown oracle:oracle "$SSH_DIR/authorized_keys"

# Set permissions
echo "Setting permissions..."
sudo chmod 400 "$SSH_DIR/my_azure_key"       # -r--------
sudo chmod 644 "$SSH_DIR/authorized_keys"     # -rw-r--r--

# Clean up temporary files
rm -f /tmp/my_azure_key /tmp/my_azure_key.pub

echo "SSH keys successfully configured!"
echo "Private key: $SSH_DIR/my_azure_key (400)"
echo "Authorized keys: $SSH_DIR/authorized_keys (644)"

```

# GitLab Configuration
GITLAB_TOKEN=your_gitlab_personal_access_token_here
GITLAB_PROJECT_ID=12345678
GITLAB_URL=https://gitlab.com
GITLAB_BRANCH=main
