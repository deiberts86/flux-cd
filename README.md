# FluxCD HomeLab Cluster
My homelab is now leveraging SUSE Harvester HCI (Hyper Converged Infrastructure) that replaced my ESX-I host I had for more than five years. KubeVirt is used under the hood on top of RKE2. Virtual machines are containerized and deployed ontop of Kubernetes (K8s).

More [HERE](https://harvesterhci.io/)

#### Table of Contents
- [Harvester With FluxCD](#harvester-with-fluxcd)
- [Setup](#setup)
- [Install FluxCD](#install-fluxcd)
- [Upgrade FluxCD](#upgrade-fluxcd)
- [Leverage SOPS](#leverage-sops)

## Harvester With FluxCD
The concept to pair Harvester with FluxCD is to control components on Harvester with DevOps. Since this is a home lab, I'm not going to over complicate the setup for my personal use. However, I will incorporate common skus that are typically seen out in the real world. There might be instances where I host local DevSecOps related content but that will be referenced in another project.

- FUTURE plans:
  - Install Core components with FluxCD
    - Cert Manager
    - Container Registry with Proxy-Cache
    - Keycloak-X
    - Object Storage with MinIO
    - Virtual Routers (KubeOVN | More to come on this)
    - Gitlab or Gitea

### Setup
Pre-Requisites:
- You will need to setup Harvester HCI, note this will take some time to do this correctly based on your own requirements.
  - The requirements and setup are found [HERE](https://docs.harvesterhci.io)
  - Plan your networking layout (I.E, if you're going to have logical vlans, build them now on your router / switching environment)
- Create a bastion host on your Harvester cluster once online
  - If you logically separated your vlans, ensure it lives in the management vlan
- Get your Harvester `kubeconfig` while logged in your bastion host
  - copy it to a save location
  - ensure you have `kubectl` and `fluxcli` installed
    ```sh
    # Install kubectl
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    # Install fluxcli
    curl -s https://fluxcd.io/install.sh | sudo bash
    ```

### Install FluxCD
There are quite a few ways to bootstrap FluxCD but I choose for my homelab to bootstrap it with `FluxCLI`. Ideally for production environments, IaC or some automated fashion through Github actions or Gitlab runners. 

Bootstrap Setup for FluxCD on GitHub:
```sh
# Bootstrap
export GITHUB_OWNER=<name>
export GITHUB_BRANCH=<branch-name>
export GITHUB_REPO=<your-repo-name>
export GITHUB_PATH=<your-path-in-repo/flux>
read -s GITHUB_TOKEN
# Paste your created Github token when it goes to the next line and press enter. This prevents it being stored in history.
flux bootstrap github --owner=$GITHUB_OWNER --repository=$GITHUB_REPO   --branch=$GITHUB_BRANCH --path=$GITHUB_PATH --token-auth
```

If done correctly, you should see the following from your bastion host CLI:
```sh
✔ helm-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ all components are healthy
```

### Upgrade FluxCD
Per "Cluster" that was bootstrapped will require this function if you need or want to upgrade FluxCD. Note, my path is a tad different but what this command below really does under the hood is literally update the pods or configuration of the deployments for FluxCD. A new branch should be created for this upgrade process.
```sh
export FLUXVER=v2.7.5
flux install --version=$FLUXVER --export > ./clusters/<cluster-name>/flux/flux-system/gotk-components.yaml
```

You should see a change on the original `gotk-components.yaml` while executing a `git status` command. Add this new change, commit, and push to your branch. Validate and merge branch to main (assuming you're using main as the source of truth for your cluster of choice).

Example with `flux check`:
```sh
flux check
► checking prerequisites
✔ Kubernetes 1.33.5+rke2r1 >=1.32.0-0
► checking version in cluster
✔ distribution: flux-v2.7.5
✔ bootstrapped: true
► checking controllers
✔ helm-controller: deployment ready
► ghcr.io/fluxcd/helm-controller:v1.4.5
✔ kustomize-controller: deployment ready
► ghcr.io/fluxcd/kustomize-controller:v1.7.3
✔ notification-controller: deployment ready
► ghcr.io/fluxcd/notification-controller:v1.7.5
✔ source-controller: deployment ready
► ghcr.io/fluxcd/source-controller:v1.7.4
► checking crds
✔ alerts.notification.toolkit.fluxcd.io/v1beta3
✔ buckets.source.toolkit.fluxcd.io/v1
✔ externalartifacts.source.toolkit.fluxcd.io/v1
✔ gitrepositories.source.toolkit.fluxcd.io/v1
✔ helmcharts.source.toolkit.fluxcd.io/v1
✔ helmreleases.helm.toolkit.fluxcd.io/v2
✔ helmrepositories.source.toolkit.fluxcd.io/v1
✔ kustomizations.kustomize.toolkit.fluxcd.io/v1
✔ ocirepositories.source.toolkit.fluxcd.io/v1
✔ providers.notification.toolkit.fluxcd.io/v1beta3
✔ receivers.notification.toolkit.fluxcd.io/v1
✔ all checks passed
```

### Leverage SOPS
Found [HERE](./sops-readme.md)