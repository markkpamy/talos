# CoreDNS Custom Configuration for Talos

This directory contains custom CoreDNS configuration for the Talos Kubernetes cluster.

## Files

- `coredns-custom-dns.yaml` - CoreDNS ConfigMap with custom domain forwarding

## Configuration

### Custom Domain Forwarding

**Domain**: `toolsera.lan`
**DNS Server**: `192.168.1.184`

All DNS queries for `*.toolsera.lan` domains will be forwarded to the specified DNS server at `192.168.1.184`.

### Default CoreDNS Functionality

The configuration preserves all default CoreDNS functionality:
- Cluster DNS resolution (cluster.local)
- External DNS forwarding via /etc/resolv.conf
- Health and readiness checks
- Prometheus metrics
- DNS caching and load balancing

## Deployment

This ConfigMap is applied during cluster bootstrap via Talos extraManifests:
- Referenced in: `prod/patches/extraManifests.yml`
- Applied automatically during cluster initialization

## Modifying DNS Configuration

To change the DNS server IP or add additional domains:

1. Edit `coredns-custom-dns.yaml`
2. Add or modify server blocks in the Corefile
3. Commit and push changes
4. Apply to existing cluster: `kubectl apply -f coredns-custom-dns.yaml`
5. Restart CoreDNS: `kubectl -n kube-system rollout restart deployment coredns`

### Adding Additional Custom Domains

To add more custom domains, add additional server blocks:

```yaml
another-domain.local:53 {
    errors
    cache 30
    forward . 192.168.1.100
}
```

## Verification

After deployment, verify the configuration:

```bash
# Check CoreDNS ConfigMap
kubectl -n kube-system get configmap coredns -o yaml

# Check CoreDNS pods
kubectl -n kube-system get pods -l k8s-app=kube-dns

# Test DNS resolution
kubectl run -it --rm dns-test --image=nicolaka/netshoot --restart=Never -- nslookup test.toolsera.lan
```

## Troubleshooting

### DNS Not Resolving Custom Domain

1. Check CoreDNS ConfigMap applied correctly:
   ```bash
   kubectl -n kube-system get configmap coredns -o yaml | grep toolsera
   ```

2. Check CoreDNS logs:
   ```bash
   kubectl -n kube-system logs -l k8s-app=kube-dns
   ```

3. Verify DNS server reachability from cluster:
   ```bash
   kubectl run -it --rm net-test --image=nicolaka/netshoot --restart=Never -- ping 192.168.1.184
   ```

### CoreDNS Not Updating

If changes aren't taking effect:
```bash
kubectl -n kube-system rollout restart deployment coredns
```

Note: CoreDNS automatically reloads the Corefile every 2 minutes via the `reload` plugin, but a manual restart ensures immediate updates.

## Notes

- This ConfigMap **replaces** the default CoreDNS ConfigMap
- Ensure the default configuration blocks are preserved when making changes
- The `log` plugin in the custom domain block helps with debugging but may impact performance under high load
- Consider removing `log` in production after verifying functionality
