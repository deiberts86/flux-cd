# FluxCD HomeLab Rancher Cluster

Home lab is now leveraging SUSE Harvester HCI (Hyper Converged Infrastructure) that replaced my ESX-I host I had. KubeVirt is used under the hood on top of RKE2. Virtual machines are containerized and deployed ontop of Kubernetes (K8s).

More [Get Harvester](https://harvesterhci.io/)

## Harvester With FluxCD

The concept to pair Harvester with FluxCD is to control components on Harvester with DevOps. Since this is a home lab, I'm not going to over complicate the setup for my personal use. However, I will incorporate common skus that are typically seen out in the real world. There might be instances where I host local DevSecOps related content but that will be referenced in another project.

- CURRENTLY Installed:
  - Cert-Manager
  - FluxCD
  - Harbor with Proxy-Cache
  - Homepage
  - Minio-Operator
  - Nebula-Sync (for Pihole)

- FUTURE plans:
  - Keycloak-X
  - Object Storage with MinIO
  - Virtual Routers (KubeOVN | More to come on this)
  - Gitlab or Gitea

### Setup

Pre-Requisites:

- You will need to setup Harvester HCI, note this will take some time to do this correctly based on your own requirements.
  - The requirements and setup are found [Harvester Docs](https://docs.harvesterhci.io)
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

### Upgrade Flux

Per "Cluster" that was bootstrapped will require this function if you need or want to upgrade FluxCD. Note, my path is a tad different but what this command below really does under the hood is literally update the pods or configuration of the deployments for FluxCD. A new branch should be created for this upgrade process.

```sh
export FLUXVER=v2.7.5
export KUBECONFIG=<harvester-kubeconfig-path.kubeconfig>
flux install --version=$FLUXVER --export > ./clusters/<cluster-name>/flux/flux-system/gotk-components.yaml
```

You should see a change on the original `gotk-components.yaml` while executing a `git status` command. Add this new change, commit, and push to your branch. Validate and merge branch to main (assuming you're using main as the source of truth for your cluster of choice).

### Leverage SOPS

Found [Leverage SOPs Readme](./sops-readme.md)
