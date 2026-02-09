# Secrets Management with SOPS + age

This repository uses [SOPS](https://github.com/getsops/sops) with [age](https://github.com/FiloSottile/age) encryption to manage Kubernetes secrets securely in Git.

## Prerequisites

Install the required tools:

```bash
# Install age (Ubuntu/Debian)
sudo apt install age

# Install sops
sudo curl -Lo /usr/local/bin/sops https://github.com/getsops/sops/releases/download/v3.11.0/sops-v3.11.0.linux.amd64
sudo chmod +x /usr/local/bin/sops
```

## Initial Setup (One-time per cluster)

### 1. Generate age key (if not already done)

```bash
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
```

### 2. Create the SOPS secret in the cluster

```bash
# Get the private key file
cat ~/.config/sops/age/keys.txt

# Create secret in flux-system namespace
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=$HOME/.config/sops/age/keys.txt
```

## Creating a New Secret

### 1. Create the secret YAML file (unencrypted)

```bash
cat > infrastructure/home/secrets/my-secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: default
type: Opaque
stringData:
  username: my-user
  password: my-password
EOF
```

### 2. Encrypt the file with SOPS

```bash
sops --encrypt --in-place infrastructure/home/secrets/my-secret.yaml
```

### 3. Add to kustomization.yaml

Edit `infrastructure/home/secrets/kustomization.yaml`:

```yaml
resources:
  - my-secret.yaml
```

### 4. Commit and push

```bash
git add infrastructure/home/secrets/
git commit -m "feat: add my-secret"
git push
```

### 5. Trigger Flux sync

```bash
flux reconcile kustomization infra-controllers --with-source
```

## Editing an Existing Secret

```bash
# Decrypt, edit, and re-encrypt in one command
sops infrastructure/home/secrets/my-secret.yaml
```

This opens the file in your default editor with decrypted values. When you save and close, SOPS automatically re-encrypts.

## Viewing a Secret (Decrypted)

```bash
sops --decrypt infrastructure/home/secrets/my-secret.yaml
```

## Configuration

The `.sops.yaml` file in the repository root defines encryption rules:

```yaml
creation_rules:
  - path_regex: .*/secrets/.*\.yaml$
    age: age1krtqjq3s9tqg3x6w02r60epx2pfyg9yer49vy05g4vanqatar5aqc7n87j
    encrypted_regex: ^(data|stringData)$
```

- `path_regex`: Only files in `**/secrets/**/*.yaml` are encrypted
- `encrypted_regex`: Only `data` and `stringData` fields are encrypted (Kubernetes Secret values)

## Security Notes

1. **Never commit unencrypted secrets** - Always encrypt before committing
2. **Backup your age private key** - Store it securely outside the repository
3. **The private key is the "secret zero"** - It must be manually created in each cluster

## Troubleshooting

### "no matching creation rules found"

The file path doesn't match the regex in `.sops.yaml`. Ensure the file is in a `secrets/` directory.

### Flux fails to decrypt

1. Verify the `sops-age` secret exists in `flux-system` namespace
2. Check the key file format (should start with `AGE-SECRET-KEY-`)
3. Check Flux logs: `kubectl logs -n flux-system deploy/kustomize-controller`

