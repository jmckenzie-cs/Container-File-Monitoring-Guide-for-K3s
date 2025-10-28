# Container File Monitoring Guide for K3s

This guide provides instructions for monitoring files inside Docker containers running on K3s hosts for File Integrity Monitoring (FIM) purposes. The **recommended approach is host-based monitoring** for security and centralization benefits.

## FIM Monitoring Priority Levels for K3s Containers

For effective File Integrity Monitoring, prioritize monitoring targets based on security impact and threat detection value.

### 🔴 REQUIRED - Critical Security Monitoring

These paths are **essential** for detecting security incidents and must be monitored:

**Container Overlay Filesystems (Application Files):**
```bash
# Primary target: Container's live filesystem
/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/<snapshot-id>/fs
```

**Why Required:**
- Contains all executable files and application code
- Detects malware injection, backdoors, and unauthorized modifications
- Shows real-time changes to running applications
- Critical for compliance and incident response

**K3s System Configuration:**
```
/etc/rancher/k3s/                       # Cluster configuration
```

**Why Required:**
- Controls cluster security policies
- Unauthorized changes can compromise entire cluster
- Required for regulatory compliance

### 🟡 RECOMMENDED - Enhanced Security Coverage

These paths provide **important additional security visibility**:

**Container Runtime Configuration:**
```
/var/lib/rancher/k3s/agent/containerd/  # Container runtime settings
```

**Why Recommended:**
- Detects container escape attempts
- Monitors runtime security configurations
- Identifies privilege escalation attempts

**Persistent Volumes (Application Data):**
```
/var/lib/rancher/k3s/storage/           # K3s local storage
/var/lib/kubelet/pods/                  # Pod volumes
```

**Why Recommended:**
- Persistent data survives container restarts
- Contains application databases and user data
- Detects data tampering and unauthorized access

**Container Logs (Runtime Activity):**
```
/var/log/pods/<namespace>_<pod-name>_<pod-uid>/
/var/lib/rancher/k3s/agent/containerd/io.containerd.grpc.v1.cri/containers/
```

**Why Recommended:**
- **Note:** Container logs are NOT in the overlay filesystem
- Provides activity correlation and forensic evidence
- Detects suspicious runtime behavior
- Required for complete audit trails

### 🟢 OPTIONAL - Comprehensive Coverage

These paths provide **additional operational insights**:

**K3s Agent Data:**
```
/var/lib/rancher/k3s/agent/             # Agent state and cache
```

**Why Optional:**
- Primarily operational data
- Lower security impact
- Useful for troubleshooting cluster issues

**Container Root Directories:**
```
/var/lib/rancher/k3s/agent/containerd/io.containerd.grpc.v1.cri/sandboxes/
```

**Why Optional:**
- Container metadata and sandbox information
- Less critical for security monitoring
- Useful for forensic analysis

### Implementation Priority

**Start with Required paths** for immediate security coverage, then add Recommended paths for comprehensive monitoring. Add Optional paths only if you have sufficient monitoring capacity and need detailed operational visibility.

## Host-Based Container Monitoring (Recommended)

### Understanding Container Overlay Filesystems

**What is an Overlay Filesystem?**

An **overlay filesystem** is a layered filesystem technology that Docker uses to create container images and running containers. It works by stacking multiple filesystem layers on top of each other.

**How Container Layers Work:**
```
┌─────────────────────┐  ← Container Layer (writable)
├─────────────────────┤  ← Application Layer (read-only)
├─────────────────────┤  ← Dependencies Layer (read-only)
├─────────────────────┤  ← OS Layer (read-only)
└─────────────────────┘  ← Base Layer (read-only)
```

**The Overlay Components:**

- **Lower Directories (LowerDir):** Read-only layers from the Docker image (base OS, dependencies, application code)
- **Upper Directory (UpperDir):** Writable layer specific to each container (runtime changes)
- **Work Directory (WorkDir):** Temporary space used by the overlay driver
- **Merged Directory (MergedDir):** ⭐ **The unified view combining all layers - THIS IS WHAT YOU MONITOR**

**Why Monitor the Merged Directory?**

The merged directory contains:
- ✅ All files from the original Docker image
- ✅ All files created/modified by the running container
- ✅ The exact filesystem view the container sees
- ✅ Real-time changes as they happen

**Practical Example:**

If a container runs nginx, the merged directory shows:
```bash
# Files from image layers (read-only):
/usr/bin/nginx          # From nginx image
/etc/nginx/nginx.conf   # From nginx image
/var/log/nginx/         # From nginx image

# Files from container layer (writable):
/var/log/nginx/access.log   # Created at runtime
/etc/nginx/custom.conf      # Added by application
/tmp/nginx.pid             # Runtime process file
```

**Benefits for FIM Monitoring:**
- **Complete visibility** - See all files the container can access
- **Real-time changes** - Detect modifications as they happen
- **Efficient storage** - Docker reuses image layers between containers
- **Isolation** - Each container has its own writable layer
- **Persistence tracking** - Know what changes survive container restarts

### Container File Paths on K3s Host

**Container filesystems (overlay) - PRIMARY MONITORING TARGET:**
```
/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/
```

**Container logs:**
```
/var/log/pods/
/var/lib/rancher/k3s/agent/containerd/io.containerd.grpc.v1.cri/containers/
```

**Container root directories:**
```
/var/lib/rancher/k3s/agent/containerd/io.containerd.grpc.v1.cri/sandboxes/
```

### K3s-Specific Paths

**K3s data directory:**
```
/var/lib/rancher/k3s/
```

**K3s configuration:**
```
/etc/rancher/k3s/
```

**Containerd socket:**
```
/run/k3s/containerd/containerd.sock
```

### Finding Container Merged Filesystem (Live Files)

The container's merged filesystem contains all the live files as seen by the container:

```bash
# Get container ID
CONTAINER_ID=$(crictl ps --name <container-name> -q)

# Get merged filesystem path (Method 1)
MERGED_PATH=$(crictl inspect $CONTAINER_ID | jq -r '.info.runtimeSpec.mounts[] | select(.destination == "/") | .source')

# Alternative method (Method 2)
crictl inspect $CONTAINER_ID | grep -A1 -B1 "MergedDir"
```

**Typical merged path format:**
```
/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/<snapshot-id>/fs
```

### Monitor Persistent Volumes from Host

**Container volumes (persistent data):**
```
/var/lib/rancher/k3s/storage/
/var/lib/kubelet/pods/<pod-uid>/volumes/
```

## Alternative Monitoring Options

### Option 1: Monitor from Inside the Container

If your FIM agent runs inside the container, monitor these paths:

```
/                 # Root filesystem of the container
/app              # Common application directory
/etc              # Configuration files
/var/log          # Application logs
/usr/local/bin    # Custom binaries
/opt              # Optional software packages
```

**Pros:**
- Direct access to container filesystem
- No path translation needed
- Sees files as the application sees them

**Cons:**
- Requires agent deployment in each container
- Agent can be compromised if container is compromised

### Option 2: Using Kubernetes Volume Mounts

Mount host directories into containers for monitoring:

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: host-fim-data
      mountPath: /fim-monitor
      readOnly: true
  volumes:
  - name: host-fim-data
    hostPath:
      path: /opt/fim/container-data
      type: Directory
```

## Practical Commands

### Accessing Container Files

**Use kubectl exec:**
```bash
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec -it <pod-name> -- ls -la /app
```

**Use crictl (containerd CLI):**
```bash
crictl exec -it <container-id> /bin/bash
crictl exec -it <container-id> ls -la /
```

### Finding Container Information

**List running containers:**
```bash
crictl ps
```

**Get container details:**
```bash
crictl inspect <container-id>
```

**Find container's filesystem:**
```bash
crictl inspect <container-id> | grep -i merged
```

### FIM Implementation Script

```bash
#!/bin/bash
# Get container's filesystem path for monitoring

CONTAINER_NAME="$1"
if [ -z "$CONTAINER_NAME" ]; then
    echo "Usage: $0 <container-name>"
    exit 1
fi

# Get container ID
CONTAINER_ID=$(crictl ps --name "$CONTAINER_NAME" -q)
if [ -z "$CONTAINER_ID" ]; then
    echo "Container '$CONTAINER_NAME' not found"
    exit 1
fi

# Get merged filesystem path
MERGED_PATH=$(crictl inspect "$CONTAINER_ID" | jq -r '.info.runtimeSpec.mounts[] | select(.destination == "/") | .source')

echo "Container: $CONTAINER_NAME"
echo "Container ID: $CONTAINER_ID"
echo "Monitor path: $MERGED_PATH"

# Verify path exists
if [ -d "$MERGED_PATH" ]; then
    echo "✓ Path exists and is accessible"
    echo "Sample files:"
    ls -la "$MERGED_PATH" | head -10
else
    echo "✗ Path not found or not accessible"
fi
```

## Security Considerations

- Monitor from host to avoid agent compromise
- Use read-only access where possible
- Monitor both container files and host paths
- Implement alerting for unauthorized changes
- Regular baseline updates for legitimate changes

## Troubleshooting

**Container not found:**
- Verify container is running: `crictl ps`
- Check container name spelling
- Ensure proper permissions to access crictl

**Path not accessible:**
- Verify running as root or appropriate user
- Check SELinux/AppArmor policies
- Ensure containerd is running properly
