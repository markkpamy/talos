# Flux Operator Installation via Omni ExtraManifests

This directory contains manifests for bootstrapping Flux CD via the Flux Operator on Talos clusters managed by Omni.

## Files

- `flux-namespace.yaml` - Creates the flux-system namespace
- `flux-operator-install.yaml` - Installs the Flux Operator
- `flux-instance.yaml` - FluxInstance CRD that configures Flux and syncs to fleet-infra repo

**Note**: Secrets (sops-age and flux-system) are created manually after cluster bootstrap for security.

## Setup Instructions

### 1. Commit and Push to GitHub

```bash
cd E:/GitRepos/talos
git add common/flux/
git commit -m "feat: add Flux Operator manifests for Omni bootstrap"
git push origin main
```

### 2. Update fleet-infra Repository

The `extraManifests.yml` in your fleet-infra repo references these manifests.

```bash
cd E:/GitRepos/fleet-infra
git add omni-talos/
git commit -m "feat: configure Flux Operator bootstrap in Omni extraManifests"
git push origin main
```

### 3. Deploy Cluster via Omni

Apply the Talos cluster configuration in Omni. The extraManifests will be applied during cluster bootstrap:
1. Creates flux-system namespace
2. Installs Flux Operator
3. Creates FluxInstance (but won't sync yet - needs secrets)

### 4. Create Required Secrets Manually

After cluster bootstrap, create the required secrets:

#### Create SOPS Age Secret

```bash
# Export your Age private key to a file
cat > /tmp/age.key <<EOF
# created: YYYY-MM-DDTHH:MM:SSZ
# public key: age14xt22jc4d8p4t5sc7k0fnsy7fuxcgy464rrgkfzt2z2mmmzy835qpkckhd
AGE-SECRET-KEY-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
EOF

# Create the secret in the cluster
kubectl create secret generic sops-age \
  --from-file=age.agekey=/tmp/age.key \
  -n flux-system

# Clean up the temporary file
rm /tmp/age.key
```

#### Create GitHub Authentication Secret

```bash
# Create secret with GitHub credentials
kubectl create secret generic flux-system \
  --from-literal=username=git \
  --from-literal=password=YOUR_GITHUB_TOKEN_HERE \
  -n flux-system
```

### 5. Verify Flux Sync

After creating both secrets, Flux will automatically start syncing:

```bash
# Watch FluxInstance status
kubectl get fluxinstance -n flux-system

# Check GitRepository sync status
kubectl get gitrepository -n flux-system

# Check Kustomization sync status
kubectl get kustomization -n flux-system

# View all Flux resources
flux get all -A
```

## How It Works

1. **flux-namespace.yaml** - Creates the flux-system namespace first
2. **flux-operator-install.yaml** - Installs the Flux Operator CRDs and controller
3. **flux-instance.yaml** - Creates a FluxInstance that:
   - Deploys all Flux controllers
   - Configures sync to `https://github.com/markkpamy/fleet-infra`
   - Points to `clusters/prod-talos-cluster` path
   - Waits for manually created secrets (sops-age, flux-system)
4. **Manual secrets** (created post-bootstrap):
   - `sops-age` - Provides the Age key for SOPS decryption
   - `flux-system` - Provides GitHub credentials for repo access

## Flux Instance Configuration

The FluxInstance is configured with:
- **Version**: 2.7.x (latest patch version)
- **Controllers**: source, kustomize, helm, notification, image-reflector, image-automation
- **Sync**: GitHub repo with 1-minute interval
- **Path**: clusters/prod-talos-cluster (matching your existing GitOps structure)

## Notes

- The Age recipient used in fleet-infra SOPS files: `age14xt22jc4d8p4t5sc7k0fnsy7fuxcgy464rrgkfzt2z2mmmzy835qpkckhd`
- No secrets are committed to the talos repository - all secrets are created manually
- Flux won't start syncing until both secrets (sops-age and flux-system) are created
- Once secrets are created, Flux will automatically decrypt all SOPS files in fleet-infra
- This setup reuses your existing fleet-infra GitOps configuration
