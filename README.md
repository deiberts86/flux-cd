# FluxCD HomeLab Rancher Cluster
Home lab is now leveraging SUSE Harvester HCI (Hyper Converged Infrastructure) that replaced my ESX-I host I had. KubeVirt is used under the hood on top of RKE2. Virtual machines are containerized and deployed ontop of Kubernetes (K8s).

More [HERE](https://harvesterhci.io/)

## Harvester With FluxCD
The concept to pair Harvester with FluxCD is to control components on Harvester with DevOps. Since this is a home lab, I'm not going to over complicate the setup for my personal use. However, I will incorporate common skus that are typically seen out in the real world. There might be instances where I host local DevSecOps related content but that will be referenced in another project.

- FUTURE plans:
  - Install Core components with FluxCD
    - Cert Manager
    - Container Registry with Proxy-Cache
    - OIDC
    - Object Storage
    - Virtual Routers
    - vCluster with Rancher MCM
    - Gitlab or Gitea

### Setup
Pre-Requisite:
- You will need to setup Harvester HCI
  - The requirements and setup are found [HERE](https://docs.harvesterhci.io/v1.5/)
- Plan your networking layout (I.E, if you're going to have logical vlans, build them now on your router / switching environment)
- Create a bastion host on your Harvester cluster once online
  - If you logically separated your vlans, ensure it lives in the management vlan
- Get your Harvester `kubeconfig` while logged in your bastion host
  - copy it to a save location.
  - ensure you have `kubectl` and `fluxcli` installed.
