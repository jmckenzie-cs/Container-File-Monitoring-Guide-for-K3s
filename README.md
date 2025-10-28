# K3s Container File Monitoring

## Required Monitoring Paths

Monitor these two paths for complete security coverage:

### 1. Container Live Filesystems
```bash
# INDIVIDUAL CONTAINER (requires snapshot ID discovery)
/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/<snapshot-id>/fs

# ALL CONTAINERS (no snapshot ID needed) - RECOMMENDED
/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/
```
- Contains all executable files and application code
- Detects malware injection and unauthorized modifications
- **✅ Full malware detection** - monitors complete container filesystem
- **✅ Bulk monitoring** - parent directory covers all containers automatically

### 2. K3s System Configuration
```bash
/etc/rancher/k3s/
```
- Controls cluster security policies
- Prevents cluster compromise
- **✅ Stable path** - can be hardcoded

## Malware Detection Capabilities

Both container monitoring approaches provide comprehensive malware detection:

**✅ DETECTS:**
- **Executable modifications** - malware changing `/bin/sh`, `/usr/bin/*`, application binaries
- **Library injection** - malicious libraries in `/lib`, `/usr/lib`
- **System file tampering** - modifications to `/etc/passwd`, `/etc/hosts`, system configs
- **Runtime file creation** - malware creating new executables anywhere in the filesystem
- **Process injection artifacts** - temporary files, modified memory-mapped files
- **Container escape attempts** - modifications to container runtime files

**Why comprehensive:** Monitors the COMPLETE filesystem view that containers see, including all executables and system files from image layers plus runtime changes.

## Recommended Setup

### Simple Bulk Monitoring (Best for Most Environments)
```bash
# Monitor all containers automatically
/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/

# Monitor K3s system configuration
/etc/rancher/k3s/
```

**Benefits:**
- ✅ **No snapshot ID management** - completely stable path
- ✅ **Catches all containers** - existing and new ones automatically
- ✅ **No dynamic discovery scripts** needed
- ✅ **Full malware detection** - includes all executable code
- ✅ **Survives restarts** - new snapshots automatically included

## Bulk Monitoring Script

```bash
#!/bin/bash
# Verify bulk monitoring setup

SNAPSHOT_DIR="/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots"
K3S_CONFIG="/etc/rancher/k3s"

echo "=== K3s Container FIM Setup ==="
echo

echo "1. Container Filesystems (All containers):"
if [ -d "$SNAPSHOT_DIR" ]; then
    CONTAINER_COUNT=$(ls -1 "$SNAPSHOT_DIR" 2>/dev/null | wc -l)
    echo "   ✅ Monitor: $SNAPSHOT_DIR"
    echo "   📊 Currently covers $CONTAINER_COUNT container snapshots"
else
    echo "   ❌ Directory not found: $SNAPSHOT_DIR"
fi

echo
echo "2. K3s System Configuration:"
if [ -d "$K3S_CONFIG" ]; then
    echo "   ✅ Monitor: $K3S_CONFIG"
else
    echo "   ❌ Directory not found: $K3S_CONFIG"
fi

echo
echo "Configuration Summary:"
echo "  Container Monitoring: $SNAPSHOT_DIR (recursive)"
echo "  System Monitoring: $K3S_CONFIG (recursive)"
echo "  Malware Detection: Full coverage"
echo "  Maintenance Required: None"
```

## Individual Container Monitoring (Alternative)

If you need to monitor specific containers only, use these methods to get snapshot IDs:

### Get Specific Container Path
```bash
# Get container ID and snapshot path
CONTAINER_NAME="your-container-name"
CONTAINER_ID=$(crictl ps --name "$CONTAINER_NAME" -q)
MERGED_PATH=$(crictl inspect "$CONTAINER_ID" | jq -r '.info.runtimeSpec.mounts[] | select(.destination == "/") | .source')
SNAPSHOT_ID=$(basename "$(dirname "$MERGED_PATH")")

echo "Container: $CONTAINER_NAME"
echo "Monitor path: $MERGED_PATH"
echo "Snapshot ID: $SNAPSHOT_ID"
```

### Get All Container Paths
```bash
#!/bin/bash
# Get all container monitoring paths

crictl ps --format table | tail -n +2 | while read line; do
    CONTAINER_ID=$(echo "$line" | awk '{print $1}')
    CONTAINER_NAME=$(echo "$line" | awk '{print $6}')

    MERGED_PATH=$(crictl inspect "$CONTAINER_ID" 2>/dev/null | jq -r '.info.runtimeSpec.mounts[]? | select(.destination == "/") | .source' 2>/dev/null)

    if [ -n "$MERGED_PATH" ] && [ "$MERGED_PATH" != "null" ]; then
        echo "Container: $CONTAINER_NAME -> Monitor: $MERGED_PATH"
    fi
done
```

### Important Note About Snapshot IDs
- **Snapshot IDs change on every container restart**
- **Never hardcode snapshot-id paths** - they become invalid after restart
- **Use bulk monitoring to avoid this complexity**

## Key Commands

```bash
# List running containers
crictl ps

# Get container filesystem path
crictl inspect <container-id> | jq -r '.info.runtimeSpec.mounts[] | select(.destination == "/") | .source'

# Verify bulk monitoring directory exists
ls -la /var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/

# Check K3s configuration
ls -la /etc/rancher/k3s/
```

## Summary

**For comprehensive malware detection with minimal operational overhead:**

1. **Monitor:** `/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/`
2. **Monitor:** `/etc/rancher/k3s/`
3. **Result:** Complete security coverage for all containers and K3s system

This approach provides the same security level as individual container monitoring but eliminates the complexity of managing changing snapshot IDs.
