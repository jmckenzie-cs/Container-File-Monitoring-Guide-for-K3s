# K3s Container File Monitoring - Required Paths Only

## Required Monitoring Paths

Monitor these two critical paths for security compliance:

### 1. Container Live Filesystems
```bash
/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/<snapshot-id>/fs
```
- Contains all executable files and application code
- Detects malware injection and unauthorized modifications
- **⚠️ Changes on restart** - requires dynamic discovery

### 2. K3s System Configuration
```bash
/etc/rancher/k3s/
```
- Controls cluster security policies
- Prevents cluster compromise
- **✅ Stable path** - can be hardcoded

## Alternative Stable Paths (Hardcode-Friendly)

If you prefer hardcoded paths that don't change on container restart, monitor these instead:

### 1. Persistent Volumes (Recommended Alternative)
```bash
# K3s local storage (stable across restarts)
/var/lib/rancher/k3s/storage/

# Pod volumes (stable within pod lifecycle)
/var/lib/kubelet/pods/<pod-uid>/volumes/
```
**Benefits:**
- ✅ Paths don't change on container restart
- ✅ Contains persistent application data
- ✅ Survives container crashes and restarts
- ⚠️ Pod UID changes if pod is deleted and recreated

### 2. Host Bind Mounts (Most Stable)
```bash
# If containers mount host directories, monitor the host paths directly
/opt/app-data/          # Example: application data directory
/var/app-logs/          # Example: application log directory
/etc/app-config/        # Example: application configuration
```
**Benefits:**
- ✅ Completely stable - never change
- ✅ Easy to configure and maintain
- ✅ Direct host filesystem access
- ⚠️ Only works if applications use host bind mounts

### 3. Container Runtime Directories (Moderately Stable)
```bash
# Container metadata and state (by container ID)
/var/lib/rancher/k3s/agent/containerd/io.containerd.grpc.v1.cri/containers/<container-id>/

# Pod sandbox directories (by pod sandbox ID)
/var/lib/rancher/k3s/agent/containerd/io.containerd.grpc.v1.cri/sandboxes/<sandbox-id>/
```
**Benefits:**
- ✅ More predictable than snapshot-id
- ✅ Contains container configuration and metadata
- ⚠️ Container ID still changes on restart, but less frequently than snapshot-id

## Finding Stable Paths

### Get Pod UID for Volume Monitoring
```bash
# Get pod UID (stable within pod lifecycle)
POD_NAME="your-pod-name"
NAMESPACE="default"
POD_UID=$(kubectl get pod "$POD_NAME" -n "$NAMESPACE" -o jsonpath='{.metadata.uid}')

echo "Monitor pod volumes: /var/lib/kubelet/pods/$POD_UID/volumes/"
```

### Discover Bind Mounts
```bash
# Find host paths mounted into containers
crictl ps --format table | tail -n +2 | while read line; do
    CONTAINER_ID=$(echo "$line" | awk '{print $1}')
    CONTAINER_NAME=$(echo "$line" | awk '{print $6}')

    echo "Container: $CONTAINER_NAME"
    crictl inspect "$CONTAINER_ID" | jq -r '.info.runtimeSpec.mounts[] | select(.type == "bind") | "  Host: \(.source) -> Container: \(.destination)"'
    echo
done
```

### Check K3s Storage Usage
```bash
# List all persistent volume directories
ls -la /var/lib/rancher/k3s/storage/

# Find recently active storage
find /var/lib/rancher/k3s/storage/ -type f -mtime -1 | head -10
```

## How to Get Snapshot IDs

**IMPORTANT:** The `<snapshot-id>` **changes every time a container is restarted**. Each container restart creates a new snapshot with a new ID.

### Implications for FIM Monitoring:
- ❌ **Never hardcode snapshot-id paths** - they become invalid after restart
- ✅ **Use dynamic discovery scripts** - automatically find current paths
- ✅ **Update monitoring configuration** after container restarts
- ✅ **Monitor container lifecycle events** to trigger path updates

The `<snapshot-id>` is required to build the full monitoring path. Use these methods:

### Method 1: Extract from Container (Recommended)
```bash
# Get container ID and snapshot path
CONTAINER_NAME="your-container-name"
CONTAINER_ID=$(crictl ps --name "$CONTAINER_NAME" -q)
MERGED_PATH=$(crictl inspect "$CONTAINER_ID" | jq -r '.info.runtimeSpec.mounts[] | select(.destination == "/") | .source')
SNAPSHOT_ID=$(basename "$(dirname "$MERGED_PATH")")

echo "Monitor path: $MERGED_PATH"
echo "Snapshot ID: $SNAPSHOT_ID"
```

### Method 2: Batch Discovery for All Containers
```bash
#!/bin/bash
# Get all container monitoring paths

crictl ps --format table | tail -n +2 | while read line; do
    CONTAINER_ID=$(echo "$line" | awk '{print $1}')
    CONTAINER_NAME=$(echo "$line" | awk '{print $6}')

    MERGED_PATH=$(crictl inspect "$CONTAINER_ID" 2>/dev/null | jq -r '.info.runtimeSpec.mounts[]? | select(.destination == "/") | .source' 2>/dev/null)

    if [ -n "$MERGED_PATH" ] && [ "$MERGED_PATH" != "null" ]; then
        SNAPSHOT_ID=$(basename "$(dirname "$MERGED_PATH")")
        echo "Container: $CONTAINER_NAME -> Monitor: $MERGED_PATH"
    fi
done
```

### Method 3: Direct Path Extraction
```bash
# If you already have the full path, extract snapshot ID
FULL_PATH="/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/abc123def456/fs"
SNAPSHOT_ID=$(echo "$FULL_PATH" | cut -d'/' -f11)
echo "Snapshot ID: $SNAPSHOT_ID"
```

## Quick Setup Script

**Note:** This script should be run regularly or triggered by container lifecycle events since snapshot-ids change on restart.

```bash
#!/bin/bash
# Setup FIM monitoring for K3s containers

echo "=== K3s Container FIM Setup ==="
echo

echo "1. K3s System Configuration:"
echo "   Monitor: /etc/rancher/k3s/"
echo

echo "2. Container Filesystems:"
crictl ps --format table | tail -n +2 | while read line; do
    CONTAINER_ID=$(echo "$line" | awk '{print $1}')
    CONTAINER_NAME=$(echo "$line" | awk '{print $6}')

    MERGED_PATH=$(crictl inspect "$CONTAINER_ID" 2>/dev/null | jq -r '.info.runtimeSpec.mounts[]? | select(.destination == "/") | .source' 2>/dev/null)

    if [ -n "$MERGED_PATH" ] && [ "$MERGED_PATH" != "null" ]; then
        echo "   Container: $CONTAINER_NAME"
        echo "   Monitor: $MERGED_PATH"
        echo
    fi
done
```

## Key Commands

```bash
# List running containers
crictl ps

# Get container filesystem path
crictl inspect <container-id> | jq -r '.info.runtimeSpec.mounts[] | select(.destination == "/") | .source'

# Verify path exists
ls -la /var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/<snapshot-id>/fs
```

## Automating Path Updates

Since snapshot-ids change on restart, consider these automation approaches:

### Option 1: Periodic Discovery
```bash
# Run every 5 minutes to catch container restarts
*/5 * * * * /usr/local/bin/update-fim-paths.sh
```

### Option 2: Container Event Monitoring
```bash
# Watch for container start events
crictl events --follow | grep "container.*started" | while read event; do
    echo "Container started, updating FIM paths..."
    /usr/local/bin/update-fim-paths.sh
done
```

### Option 3: Kubernetes Event Watching
```bash
# Watch for pod events
kubectl get events --watch --field-selector involvedObject.kind=Pod | while read event; do
    if [[ "$event" == *"Started"* ]]; then
        echo "Pod started, updating FIM paths..."
        /usr/local/bin/update-fim-paths.sh
    fi
done
```
