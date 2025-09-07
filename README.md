# Hetzner Bare Metal Bootc Deployment

This repository builds and deploys a bootc (boot container) image to Hetzner bare metal servers using GitHub Actions.

## Purpose

Build and deploy a base image to Hetzner bare metal servers using bootc technology for immutable, container-based deployments.

## Setup for Automated Deployment

### Prerequisites

1. A Hetzner bare metal server running a bootc-compatible OS (Fedora CoreOS, RHEL 9.4+, CentOS Stream 9, etc.)
2. SSH access to the server as root
3. The server should have `bootc` installed (the workflow will install it if missing)

### Required GitHub Configuration

Set up the following in your GitHub repository:

#### Repository Variables (Settings → Secrets and variables → Actions → Variables tab)

1. **`HETZNER_SERVER_IPS`** - Comma-separated list of server IP addresses
   ```
   Example: 1.2.3.4,5.6.7.8,9.10.11.12
   ```

#### Repository Secrets (Settings → Secrets and variables → Actions → Secrets tab)

1. **`HETZNER_SSH_PRIVATE_KEY`** - Your SSH private key for accessing the Hetzner server(s)
   ```bash
   # Generate a new SSH key pair (if you don't have one)
   ssh-keygen -t ed25519 -C "github-actions@yourdomain.com" -f ~/.ssh/hetzner_deploy
   
   # Add the public key to your server(s) authorized_keys
   ssh-copy-id -i ~/.ssh/hetzner_deploy.pub root@YOUR_SERVER_IP
   
   # Copy the private key content to GitHub secrets
   cat ~/.ssh/hetzner_deploy
   ```

2. **`GITHUB_TOKEN`** - This is automatically provided by GitHub Actions for container registry access

### Deployment Options

#### Fully Automated GitOps Deployment
The deployment workflow automatically triggers when the build workflow completes successfully on the `main` branch. It will deploy to all servers specified in the `HETZNER_SERVER_IPS` environment variable and automatically reboot them to activate the new image.

**Automated GitOps Flow:**
1. Push code to `main` branch
2. Build workflow creates new bootc image
3. Deploy workflow automatically stages new image on all servers
4. **Servers are automatically rebooted to activate the new image**
5. Deployment verification confirms the new image is active

#### Emergency Rollback
A separate manual workflow is available for emergency rollbacks to the previous working image.

**Manual Rollback Flow:**
1. Go to Actions → "Rollback bootc image on Hetzner bare metal servers"
2. Click "Run workflow"
3. Type "ROLLBACK" in the confirmation field (required for safety)
4. Optionally provide a reason for the rollback
5. All servers are automatically rolled back to the previous image
6. **Servers are automatically rebooted to activate the previous image**
7. Rollback verification confirms the previous image is active

This approach provides:
- **Full automation**: Complete hands-off deployment from code to production
- **Parallel rollouts**: All servers updated simultaneously
- **Emergency rollback**: Quick recovery from failed deployments
- **Atomic operations**: Both deployments and rollbacks are all-or-nothing
- **Verification**: Automatic confirmation that new/previous images are active
- **Fast feedback**: Quick detection of deployment issues
- **Consistency**: All deployments triggered by code changes
- **Traceability**: Every deployment tied to a specific commit

⚠️ **Important**: Both workflows will automatically reboot your servers. Ensure you have appropriate monitoring and alerting in place.

### Multi-Server Deployment

The workflow supports deploying to multiple servers simultaneously:

- **Parallel deployment**: All servers are updated concurrently using GitHub Actions matrix strategy
- **Fail-safe**: If one server fails, others continue (`fail-fast: false`)
- **Individual monitoring**: Each server deployment is tracked separately
- **Scalable**: Add as many servers as needed to the list

### Deployment Process

The deployment workflow:

1. **Connects to your Hetzner servers** via SSH
2. **Installs bootc** if not already present
3. **Logs into the container registry** to pull your image
4. **Switches the bootc image** using `bootc switch` command
5. **Automatically reboots** the servers to activate the new image
6. **Waits for servers** to come back online
7. **Verifies the deployment** after reboot to confirm new image is active

### Understanding Bootc Deployments

- **Staged vs Active**: After `bootc switch`, the new image is staged and activated immediately after reboot
- **Rollback capability**: Bootc maintains the previous image for easy rollback if needed
- **Immutable updates**: The entire OS is replaced atomically, ensuring consistency
- **Automatic activation**: The workflow handles the complete deployment cycle including reboots

### Manual Operations

#### Check deployment status on your servers:
```bash
# Single server
ssh root@YOUR_SERVER_IP "bootc status"

# Multiple servers
for ip in 1.2.3.4 5.6.7.8; do
  echo "=== Server $ip ==="
  ssh root@$ip "bootc status"
done
```

#### Rollback to previous image (if needed):
```bash
# Preferred method - Use GitHub Actions rollback workflow:
# 1. Go to Actions → "Rollback bootc image on Hetzner bare metal servers"
# 2. Click "Run workflow", type "ROLLBACK", and provide reason
# 3. All servers will be automatically rolled back

# Manual rollback (emergency use only):
# Single server
ssh root@YOUR_SERVER_IP "bootc rollback && reboot"

# Multiple servers
for ip in 1.2.3.4 5.6.7.8; do
  echo "Rolling back $ip..."
  ssh root@$ip "bootc rollback && reboot"
  sleep 30  # Wait between rollbacks
done
```

#### Emergency procedures:
```bash
# Check if a server failed to come back online after deployment
ssh root@YOUR_SERVER_IP "bootc status"

# Manual rollback via GitHub Actions (preferred method):
# 1. Go to Actions → "Rollback bootc image on Hetzner bare metal servers"
# 2. Click "Run workflow"
# 3. Type "ROLLBACK" and provide reason
# 4. Workflow will rollback all servers automatically

# Manual rollback via SSH (emergency only):
ssh root@YOUR_SERVER_IP "bootc rollback && reboot"

# Check rollback status:
ssh root@YOUR_SERVER_IP "bootc status"
```

### Rollback Workflow Features

The rollback workflow provides:
- **Safety confirmation**: Must type "ROLLBACK" to prevent accidental rollbacks
- **Reason tracking**: Optional reason field for audit purposes
- **Pre-flight checks**: Verifies rollback image is available before proceeding
- **Parallel rollback**: All servers rolled back simultaneously
- **Automatic reboot**: Servers automatically rebooted to activate previous image
- **Verification**: Confirms previous image is active after rollback
- **Detailed reporting**: Complete rollback report with status and next steps
- **Fault tolerance**: Individual server failures don't stop other rollbacks

## Image Contents

This bootc image includes:
- Fedora bootc base
- Cockpit web console (accessible on port 9090)
- Podman container runtime
- System monitoring tools (lm_sensors, sysstat)
- Performance tuning (tuned)
- Caddy web server (configured via systemd container)

## Customization

Edit the `Containerfile` to add additional software or configuration to your bootc image.
