# Cilium CNI Installation for Talos

This directory contains manifests for installing Cilium as the CNI (Container Network Interface) on Talos Kubernetes clusters with kube-proxy replacement.

## Files

- `install-cilium.yaml` - Cilium installation manifest (generated from Helm chart)
- `loadbalancer-ippool.yaml` - LoadBalancer IP pool definition
- `L2Announcement.yaml` - L2 announcement policy

## Configuration

### Cilium Features Enabled

- **kube-proxy Replacement**: Cilium fully replaces kube-proxy functionality
- **KubePrism Integration**: Connects to Talos KubePrism on localhost:7445
- **L2 Announcements**: Enables LoadBalancer services on bare-metal via L2 (ARP)
- **External IPs**: Support for external IP addresses on services
- **IPAM Mode**: Kubernetes native IP address management

### Talos-Specific Settings

- **Security Context**: SYS_MODULE capability removed (required for Talos)
- **Cgroup Settings**: Uses Talos cgroupv2 at /sys/fs/cgroup
- **No Auto-mount**: Cgroup auto-mount disabled (Talos provides this)

### LoadBalancer IP Pool

**IP Range**: 192.168.1.201 - 192.168.1.210

This range is used for LoadBalancer service IPs. Services with type `LoadBalancer` will receive an IP from this pool and be announced via L2 (ARP) on the local network.

## Regenerating Manifests

If you need to update the Cilium version or configuration:

```bash
# Add/update Cilium Helm repo
helm repo add cilium https://helm.cilium.io/
helm repo update cilium

# Generate new manifest
helm template cilium cilium/cilium \
  --namespace kube-system \
  --set ipam.mode=kubernetes \
  --set kubeProxyReplacement=true \
  --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
  --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
  --set cgroup.autoMount.enabled=false \
  --set cgroup.hostRoot=/sys/fs/cgroup \
  --set k8sServiceHost=localhost \
  --set k8sServicePort=7445 \
  --set l2announcements.enabled=true \
  --set externalIPs.enabled=true \
  > install-cilium.yaml
```

To use a specific version, add `--version X.Y.Z` to the helm template command.

## Updating IP Pool

To change the LoadBalancer IP range, edit `loadbalancer-ippool.yaml`:

```yaml
spec:
  blocks:
  - start: "192.168.1.201"  # Change start IP
    stop: "192.168.1.210"    # Change end IP
```

Alternatively, you can use CIDR notation:

```yaml
spec:
  blocks:
  - cidr: "192.168.1.200/29"  # Provides 192.168.1.200-207
```

## Deployment via Omni ExtraManifests

These manifests are referenced in the Omni cluster configuration at:
`omni-talos/talos/prod/patches/extraManifests.yml`

During cluster bootstrap:
1. Cilium is installed first (networking must be ready before other components)
2. LoadBalancer IP pool is created
3. L2 announcement policies are applied
4. LoadBalancer services can immediately receive IPs from the pool

## Prerequisites

The following Talos machine config patches are required (already configured in your setup):

- **CNI disabled**: `cluster.network.cni.name: none` (patches/cni.yml)
- **kube-proxy disabled**: (patches/disable-kubeproxy.yml)

## Verification

After cluster bootstrap, verify Cilium is running:

```bash
# Check Cilium pods
kubectl get pods -n kube-system -l k8s-app=cilium

# Check Cilium status (if cilium CLI is installed)
cilium status

# Verify L2 announcement resources
kubectl get ciliumloadbalancerippool
kubectl get ciliuml2announcementpolicy

# Test LoadBalancer service
kubectl create service loadbalancer test --tcp=80:80
kubectl get svc test
```

## Troubleshooting

### Cilium Pods Not Starting

Check security context and cgroup settings:
```bash
kubectl logs -n kube-system -l k8s-app=cilium
```

### LoadBalancer IPs Not Assigned

Check the IP pool:
```bash
kubectl get ciliumloadbalancerippool default-pool -o yaml
kubectl describe ciliuml2announcementpolicy default-l2-policy
```

### L2 Announcements Not Working

Verify network interface matching in L2AnnouncementPolicy:
```bash
# List node network interfaces
kubectl get nodes -o json | jq '.items[].status.addresses'
```

The policy includes common interface patterns: `eth*`, `ens*`, `enp*s*`. Adjust if your nodes use different interface names.

## Notes

- This is plain YAML output from Helm template - no Helm is required at runtime
- All manifests are applied during Talos cluster bootstrap via extraManifests
- Cilium version is embedded in the generated manifest (check install-cilium.yaml)
- For production, consider pinning to a specific Cilium version rather than using latest
