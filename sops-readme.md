# Install and configure SOPS for FluxCD with GPG

SOPS is a great tool to encrypting your sensitive information while using GIT to store the encrypted data. In this setup, we will use GPG to fulfill this process. There are many other ways to do this with other secrets managers (Azure Key vault, Hashicorp Vault, AWS Secrets Manager, etc.).

Reference:
[FluxCD with SOPS](https://fluxcd.io/flux/guides/mozilla-sops/)

## Pre-requisites

- A Kubernetes cluster
- FluxCD already bootstrapped

## Install SOPS & GnuPG on Workstation or Bastionhost

```bash
# Debian Builds
sudo apt update
sudo apt install -y gnupg2

# SOPS Install
# Download the binary
sudo curl -LO https://github.com/getsops/sops/releases/download/v3.11.0/sops-v3.11.0.linux.amd64

# Move the binary in to your PATH
sudo mv sops-v3.11.0.linux.amd64 /usr/local/bin/sops

# Make the binary executable
sudo chmod +x /usr/local/bin/sops
```

## Generate a GPG Key for SOPS

- Note, you need to ensure there is no passphrase as FluxCD can't leverage passwords itself. Refer to the `Reference` link.

```bash
# Set Env Vars and adjust accordingly
export KEY_NAME="homelab"
export KEY_COMMENT="flux secrets"

# This should create a new key and revocation file
gpg --batch --full-generate-key <<EOF
%no-protection
Key-Type: 1
Key-Length: 4096
Subkey-Type: 1
Subkey-Length: 4096
Expire-Date: 0
Name-Comment: ${KEY_COMMENT}
Name-Real: ${KEY_NAME}
EOF
```

Get `Fingerprint` (pubkey) and `Export` it

```bash
gpg --list-secret-keys "${KEY_NAME}"
export KEY_FP="$(gpg --list-secret-keys "${KEY_NAME}" \
 | sed '2!d' \
 | tr -d ' ')"
# Echo Output and validate it matches from listing your initial secret keys
echo "$KEY_FP"
```

FluxCD needs the private key to decrypt secrets you plan to encrypt. Create a kubernetes secret within `flux-system` namespace. Note, this is post bootstrapping of Flux itself.

```bash
gpg --export-secret-keys --armor "${KEY_FP}" |
kubectl create secret generic sops-gpg \
--namespace=flux-system \
--from-file=sops.asc=/dev/stdin
```

Backup your Key elsewhere within a password manager or encrypted storage.

```bash
gpg --export-secret-keys --armor "$KEY_FP" > sops-gpg-backup.asc
```

Now you can safely remove this GPG key from your BastionHost or Workstation. But, you can't "decrypt" secrets anymore if you remove the private key from your BastionHost or Workstation.  The public key is sufficient enough to encrypt because the private key lives within the Kubernetes cluster within the `flux-system` namespace.

```bash
# removes private key
gpg --delete-secret-keys "$KEY_FP"
# leaves the public key behind
```

If you ever needed to re-import the secret key, here's how to do this below.

```bash
gpg --import sops-gpg-backup.asc
```

## Configure your .sops.yaml file

For this setup, we will keep things simple and ask that you just put this new file at the root location of your `cluster/<clustername>/.sops.yaml`. Because I want this to cover everything, I did it the lazy way to ensure I don't have surprises later on. Basically these `creation_rules` covers every folder. Even though it's `lazy`, it covers my basis incase I forget a repo (which wouldn't be good).

```yaml
# Create .sops.yaml in your root FluxCD repo and add the following to the file.
creation_rules:
  - path_regex: .*\.ya?ml$
    encrypted_regex: '^(data|stringData)$'
    pgp: "${KEY_FP}"
  - path_regex: .*\.json
    encrypted_regex: '^(username|email|password|auth)$'
    pgp: "${KEY_FP}"
```

Add the following lines for kustomize which refers to a folder path.

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: <name>
  namespace: flux-system
spec:
  interval: 5m
  path: ./<path>
  prune: true
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
  decryption:
    provider: sops
    secretRef:
      name: sops-gpg
```

## Sample Secret

IMPORTANT: If using `stringData`, the MACS will change because FluxCD will interpret this and convert to `data` field.  Ideally, use base64 encoded value (key=value) within your Kubernetes secret within the `data` and then encrypt before commiting your code.

```yaml
apiVersion: v1
kind: Secret
metadata:
    name: sample-secret
    namespace: default
type: Opaque
data:
    password: <your-base64-encoded-secret>
```

Once the file is ready, simply execute the following:

```bash
sops -e --in-place <path>/sample-secret.sops.yaml 
```
