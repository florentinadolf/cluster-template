# ⛵ Cluster Template (LAN-Only Fork)

> [!CAUTION]
> **This is a modified fork of the original [onedr0p/cluster-template](https://github.com/onedr0p/cluster-template) project.**
>
> ## Key Modifications from Upstream
>
> This fork has been **significantly modified** to support **LAN-only** deployments without external internet access:
> - **Removed:** All Cloudflare dependencies (cloudflared tunnel, external-dns with Cloudflare provider)
> - **Removed:** External internet access capabilities and Flux webhook receiver
> - **Added:** k8s-gateway for automatic DNS resolution within your local network
> - **Added:** Flexible certificate management (user-provided CA or self-signed certificates)
> - **Changed:** Flux uses periodic sync only (use `task reconcile` for immediate sync)
>
> **⚠️ IMPORTANT: The upstream project maintainers and community will NOT provide support for this fork.** If you encounter issues specific to these modifications, do not seek help in the upstream project's support channels. This fork is provided as-is for users who need a LAN-only Kubernetes cluster without Cloudflare integration.
>
> If you need the original functionality with Cloudflare support and external access, please use the upstream project instead: https://github.com/onedr0p/cluster-template

Welcome to a template designed for deploying a single Kubernetes cluster at home on bare-metal or virtual machines (VMs). This project aims to simplify the process and make Kubernetes accessible within your local network.

At its core, this project leverages [makejinja](https://github.com/mirkolenz/makejinja), a powerful tool for rendering templates. By reading configuration files—such as [cluster.yaml](./cluster.sample.yaml) and [nodes.yaml](./nodes.sample.yaml)—Makejinja generates the necessary configurations to deploy a Kubernetes cluster with a modular and extensible approach to cluster deployment and management.

With this approach, you'll gain a solid foundation to build and manage your Kubernetes cluster efficiently within your local network.

## ✨ Features

A Kubernetes cluster deployed with [Talos Linux](https://github.com/siderolabs/talos) and an opinionated implementation of [Flux](https://github.com/fluxcd/flux2) using [GitHub](https://github.com/) as the Git provider, and [sops](https://github.com/getsops/sops) to manage secrets.

- **Required:** Some knowledge of [Containers](https://opencontainers.org/), [YAML](https://noyaml.com/), [Git](https://git-scm.com/), and a local domain (e.g., `home.local`).
- **Included components:** [flux](https://github.com/fluxcd/flux2), [cilium](https://github.com/cilium/cilium), [cert-manager](https://github.com/cert-manager/cert-manager), [spegel](https://github.com/spegel-org/spegel), [reloader](https://github.com/stakater/Reloader), [envoy-gateway](https://github.com/envoyproxy/gateway), and [k8s-gateway](https://github.com/ori-edge/k8s_gateway).

**Other features include:**

- Dev env managed w/ [mise](https://mise.jdx.dev/)
- Workflow automation w/ [GitHub Actions](https://github.com/features/actions)
- Dependency automation w/ [Renovate](https://www.mend.io/renovate)
- Flux `HelmRelease` and `Kustomization` diffs w/ [flux-local](https://github.com/allenporter/flux-local)

Does this sound cool to you? If so, continue to read on! 👇

## 🚀 Let's Go!

There are **4 stages** outlined below for completing this project, make sure you follow the stages in order.

### Stage 1: Machine Preparation

> [!IMPORTANT]
> If you have **3 or more nodes** it is recommended to make 3 of them controller nodes for a highly available control plane. This project configures **all nodes** to be able to run workloads. **Worker nodes** are therefore **optional**.
>
> **Minimum system requirements**
> | Role    | Cores    | Memory        | System Disk               |
> |---------|----------|---------------|---------------------------|
> | Control/Worker | 4 | 16GB | 256GB SSD/NVMe |

1. Head over to the [Talos Linux Image Factory](https://factory.talos.dev) and follow the instructions. Be sure to only choose the **bare-minimum system extensions** as some might require additional configuration and prevent Talos from booting without it. Depending on your CPU start with the Intel/AMD system extensions (`i915`, `intel-ucode` & `mei` **or** `amdgpu` & `amd-ucode`), you can always add system extensions after Talos is installed and working.

2. This will eventually lead you to download a Talos Linux ISO (or for SBCs a RAW) image. Make sure to note the **schematic ID** you will need this later on.

3. Flash the Talos ISO or RAW image to a USB drive and boot from it on your nodes.

4. Verify with `nmap` that your nodes are available on the network. (Replace `192.168.1.0/24` with the network your nodes are on.)

    ```sh
    nmap -Pn -n -p 50000 192.168.1.0/24 -vv | grep 'Discovered'
    ```

### Stage 2: Local Workstation

> [!TIP]
> It is recommended to set the visibility of your repository to `Public` so you can easily request help if you get stuck.

1. Create a new repository by using this template, then clone the new repo you just created and `cd` into it.

2. **Install** the [Mise CLI](https://mise.jdx.dev/getting-started.html#installing-mise-cli) on your workstation.

3. **Activate** Mise in your shell by following the [activation guide](https://mise.jdx.dev/getting-started.html#activate-mise).

4. Use `mise` to install the **required** CLI tools:

    ```sh
    mise trust
    pip install pipx
    mise install
    ```

   📍 _**Having trouble installing the tools?** Try unsetting the `GITHUB_TOKEN` env var and then run these commands again_

   📍 _**Having trouble compiling Python?** Try running `mise settings python.compile=0` and then run these commands again_

5. Logout of GitHub Container Registry (GHCR) as this may cause authorization problems when using the public registry:

    ```sh
    docker logout ghcr.io
    helm registry logout ghcr.io
    ```

### Stage 3: Cluster Configuration

> [!WARNING]
> If any of the commands fail with `command not found` or `unknown command` it means `mise` is either not installed or configured incorrectly.

1. Generate the config files from the sample files:

    ```sh
    task init
    ```

2. Fill out `cluster.yaml` and `nodes.yaml` configuration files using the comments in those files as a guide.

   **Important `cluster.yaml` fields:**
   - `domain`: Your local domain (e.g., `home.local`, `homelab.internal`)
   - `cluster_gateway_addr`: IP for the internal Envoy gateway (must be unused in your `node_cidr`)
   - `cluster_dns_gateway_addr`: IP for k8s-gateway DNS server (must be unused in your `node_cidr`)
   - `ca_cert_path` and `ca_key_path` (optional): Paths to your CA certificate and key for trusted HTTPS certificates
     - If not provided, self-signed certificates will be generated automatically
     - If using a custom CA, browsers will need to import the CA certificate for trusted HTTPS

3. **(Optional) Configure your CA certificate:**

   If you want to avoid browser certificate warnings, provide paths to your CA certificate and private key:

   ```yaml
   # In cluster.yaml
   ca_cert_path: "/path/to/your/ca.crt"
   ca_key_path: "/path/to/your/ca.key"
   ```

   Your CA certificate will be used to sign all cluster certificates, providing valid HTTPS when the CA is trusted by client devices.

4. Template out the kubernetes and talos configuration files. If any issues come up, be sure to read the error and adjust your config files accordingly.

    ```sh
    task configure
    ```

5. Push your changes to git:

   📍 _**Verify** all the `./kubernetes/**/*.sops.*` files are **encrypted** with SOPS_

    ```sh
    git add -A
    git commit -m "chore: initial commit :rocket:"
    git push
    ```

> [!TIP]
> Using a **private repository**? Make sure to paste the public key from `github-deploy.key.pub` into the deploy keys section of your GitHub repository settings. This will make sure Flux has read/write access to your repository.

### Stage 4: Bootstrap Talos, Kubernetes, and Flux

> [!WARNING]
> It might take a while for the cluster to be setup (10+ minutes is normal). During which time you will see a variety of error messages like: "couldn't get current server API group list," "error: no matching resources found", etc. 'Ready' will remain "False" as no CNI is deployed yet. **This is normal.** If this step gets interrupted, e.g. by pressing <kbd>Ctrl</kbd> + <kbd>C</kbd>, you likely will need to [reset the cluster](#-reset) before trying again

1. Install Talos:

    ```sh
    task bootstrap:talos
    ```

2. Push your changes to git:

    ```sh
    git add -A
    git commit -m "chore: add talhelper encrypted secret :lock:"
    git push
    ```

3. Install cilium, coredns, spegel, flux and sync the cluster to the repository state:

    ```sh
    task bootstrap:apps
    ```

4. Watch the rollout of your cluster happen:

    ```sh
    kubectl get pods --all-namespaces --watch
    ```

## 📣 Post Installation

### ✅ Verifications

1. Check the status of Cilium:

    ```sh
    cilium status
    ```

2. Check the status of Flux and if the Flux resources are up-to-date and in a ready state:

   📍 _Run `task reconcile` to force Flux to sync your Git repository state_

    ```sh
    flux check
    flux get sources git flux-system
    flux get ks -A
    flux get hr -A
    ```

3. Check TCP connectivity to the internal gateway:

   📍 _The variables are only placeholders, replace them with your actual values_

    ```sh
    nmap -Pn -n -p 443 ${cluster_gateway_addr} -vv
    ```

4. Check you can resolve DNS for `echo` via k8s-gateway:

   📍 _The variables are only placeholders, replace them with your actual values_

    ```sh
    dig @${cluster_dns_gateway_addr} echo.${domain}
    ```

5. Check the status of your wildcard `Certificate`:

    ```sh
    kubectl -n network describe certificates
    ```

### 🏠 LAN DNS Configuration

> [!IMPORTANT]
> **This cluster is LAN-only.** All applications are accessible via the `envoy-internal` gateway on your local network only.

`k8s-gateway` provides automatic DNS resolution for HTTPRoutes in your cluster. For this to work, your LAN DNS server must be configured to forward DNS queries for your domain to the k8s-gateway IP (`${cluster_dns_gateway_addr}`).

**Common DNS server configurations:**

#### pfSense / Unbound

Add to DNS Resolver > Advanced settings > Custom options:

```
server:
  local-zone: "home.local." transparent

forward-zone:
  name: "home.local."
  forward-addr: <cluster_dns_gateway_addr>
```

#### Pi-hole

Add to `/etc/dnsmasq.d/02-custom.conf`:

```
server=/home.local/<cluster_dns_gateway_addr>
```

#### Router DNS Forwarding

Most routers allow setting conditional forwarding or custom DNS records. Configure your router to forward queries for `*.home.local` to the k8s-gateway IP.

**Testing DNS:**

```sh
# From any device on your LAN
dig echo.home.local
nslookup echo.home.local
```

### 🔒 Certificate Trust

#### Self-Signed Certificates (Default)

If you did not provide `ca_cert_path` and `ca_key_path` in `cluster.yaml`, self-signed certificates are automatically generated.

**Expected behavior:**
- Browsers will show security warnings
- HTTPS still works, but requires accepting certificate exceptions
- Functional but not ideal for production use

#### Custom CA Certificates (Recommended)

If you provided CA certificate paths, import your CA certificate to client devices for trusted HTTPS:

**Windows:**
1. Open `certmgr.msc`
2. Trusted Root Certification Authorities > Certificates
3. Action > All Tasks > Import
4. Select your CA certificate file
5. Restart browser

**macOS:**
1. Open Keychain Access
2. System > Certificates
3. File > Import Items
4. Select your CA certificate
5. Double-click certificate > Trust > Always Trust
6. Restart browser

**Linux:**
```sh
sudo cp ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

**iOS/Android:**
Email the CA certificate to yourself and install it from Settings > Security.

### 🔄 Flux Sync

This cluster uses **periodic sync** instead of webhooks. Flux checks the Git repository for changes on a schedule (default: every few minutes).

To force an immediate sync after pushing changes:

```sh
task reconcile
```

Or manually:

```sh
flux reconcile source git flux-system
flux reconcile kustomization flux-system
```

## 💥 Reset

> [!CAUTION]
> **Resetting** the cluster **multiple times in a short period of time** could lead to being **rate limited by DockerHub**.

There might be a situation where you want to destroy your Kubernetes cluster. The following command will reset your nodes back to maintenance mode.

```sh
task talos:reset
```

## 🛠️ Talos and Kubernetes Maintenance

### ⚙️ Updating Talos node configuration

> [!TIP]
> Ensure you have updated `talconfig.yaml` and any patches with your updated configuration. In some cases you may need to not only apply the configuration, but also upgrade Talos for the changes to take effect.

```sh
# (Re)generate the Talos config
task talos:generate-config
# Apply the config to the node
task talos:apply-node IP=? MODE=?
# e.g. task talos:apply-node IP=10.10.10.10 MODE=auto
```

### ⬆️ Updating Talos and Kubernetes versions

> [!TIP]
> Ensure the `talosVersion` and `kubernetesVersion` in `talenv.yaml` are up-to-date with the version you wish to upgrade to.

```sh
# Upgrade node to a newer Talos version
task talos:upgrade-node IP=?
# e.g. task talos:upgrade-node IP=10.10.10.10
```

```sh
# Upgrade cluster to a newer Kubernetes version
task talos:upgrade-k8s
# e.g. task talos:upgrade-k8s
```

### ➕ Adding a node to your cluster

At some point you might want to expand your cluster to run more workloads and/or improve the reliability of your cluster. Keep in mind it is recommended to have an **odd number** of control plane nodes for quorum reasons.

You don't need to re-bootstrap the cluster to add new nodes. Follow these steps:

1. **Prepare the new node**: Review the [Stage 1: Machine Preparation](#stage-1-machine-preparation) section and boot your new node into maintenance mode.

2. **Get the node information**: While the node is in maintenance mode, retrieve the disk and MAC address information needed for configuration:

   ```sh
   talosctl get disks -n <ip> --insecure
   talosctl get links -n <ip> --insecure
   ```

3. **Update the configuration**: Read the documentation for [talhelper](https://budimanjojo.github.io/talhelper/latest/) and extend the `talconfig.yaml` file manually with the new node information (including the disk and MAC address from step 2).

4. **Generate and apply the configuration**:

   ```sh
   # Render your talosconfig based on the talconfig.yaml file
   task talos:generate-config

   # Apply the configuration to the node
   task talos:apply-node IP=?
   # e.g. task talos:apply-node IP=10.10.10.10
   ```

The node should join the cluster automatically and workloads will be scheduled once they report as ready.

## 🤖 Renovate

[Renovate](https://www.mend.io/renovate) is a tool that automates dependency management. It is designed to scan your repository around the clock and open PRs for out-of-date dependencies it finds. Common dependencies it can discover are Helm charts, container images, GitHub Actions and more! In most cases merging a PR will cause Flux to apply the update to your cluster.

To enable Renovate, click the 'Configure' button over at their [Github app page](https://github.com/apps/renovate) and select your repository. Renovate creates a "Dependency Dashboard" as an issue in your repository, giving an overview of the status of all updates. The dashboard has interactive checkboxes that let you do things like advance scheduling or reattempt update PRs you closed without merging.

The base Renovate configuration in your repository can be viewed at [.renovaterc.json5](.renovaterc.json5). By default it is scheduled to be active with PRs every weekend, but you can [change the schedule to anything you want](https://docs.renovatebot.com/presets-schedule), or remove it if you want Renovate to open PRs immediately.

## 🐛 Debugging

Below is a general guide on trying to debug an issue with a resource or application. For example, if a workload/resource is not showing up or a pod has started but in a `CrashLoopBackOff` or `Pending` state. These steps do not include a way to fix the problem as the problem could be one of many different things.

1. Check if the Flux resources are up-to-date and in a ready state:

   📍 _Run `task reconcile` to force Flux to sync your Git repository state_

    ```sh
    flux get sources git -A
    flux get ks -A
    flux get hr -A
    ```

2. Do you see the pod of the workload you are debugging:

    ```sh
    kubectl -n <namespace> get pods -o wide
    ```

3. Check the logs of the pod if it's there:

    ```sh
    kubectl -n <namespace> logs <pod-name> -f
    ```

4. If a resource exists try to describe it to see what problems it might have:

    ```sh
    kubectl -n <namespace> describe <resource> <name>
    ```

5. Check the namespace events:

    ```sh
    kubectl -n <namespace> get events --sort-by='.metadata.creationTimestamp'
    ```

Resolving problems that you have could take some tweaking of your YAML manifests in order to get things working, other times it could be an external factor like permissions on an NFS server. If you are unable to figure out your problem, please understand that **the upstream project will not provide support for this fork**.

## 🧹 Tidy up

Once your cluster is fully configured and you no longer need to run `task configure`, it's a good idea to clean up the repository by removing the [templates](./templates) directory and any files related to the templating process. This will help eliminate unnecessary clutter and resolve any "duplicate registry" warnings from Renovate.

1. Tidy up your repository:

    ```sh
    task template:tidy
    ```

2. Push your changes to git:

    ```sh
    git add -A
    git commit -m "chore: tidy up :broom:"
    git push
    ```

## ❔ What's next

There's a lot to absorb here, especially if you're new to these tools. Take some time to familiarize yourself with the tooling and understand how all the components interconnect. Dive into the documentation of the various tools included — they are a valuable resource. This shouldn't be a production environment yet, so embrace the freedom to experiment. Move fast, break things intentionally, and challenge yourself to fix them.

Below are some optional considerations you may want to explore.

### Secrets

SOPS is an excellent tool for managing secrets in a GitOps workflow. However, it can become cumbersome when rotating secrets or maintaining a single source of truth for secret items.

For a more streamlined approach to those issues, consider [External Secrets](https://external-secrets.io/latest/). This tool allows you to move away from SOPS and leverage an external provider for managing your secrets. External Secrets supports a wide range of providers, from cloud-based solutions to self-hosted options.

### Storage

If your workloads require persistent storage with features like replication or connectivity to NFS, SMB, or iSCSI servers, there are several projects worth exploring:

- [rook-ceph](https://github.com/rook/rook)
- [longhorn](https://github.com/longhorn/longhorn)
- [openebs](https://github.com/openebs/openebs)
- [democratic-csi](https://github.com/democratic-csi/democratic-csi)
- [csi-driver-nfs](https://github.com/kubernetes-csi/csi-driver-nfs)
- [csi-driver-smb](https://github.com/kubernetes-csi/csi-driver-smb)
- [synology-csi](https://github.com/SynologyOpenSource/synology-csi)

These tools offer a variety of solutions to meet your persistent storage needs, whether you're using cloud-native or self-hosted infrastructures.

## 🙋 Support

> [!WARNING]
> As noted in the caution block above, the upstream project will NOT provide support for this fork's modifications.

**For fork-specific issues:** Open an issue in this repository if you encounter problems with LAN-only setup, certificate management, or other modifications.

**For general help:** Questions about Kubernetes, Talos, and Flux may be answered by the broader community, but please clarify you're using this fork.

**For the original template:** If you need Cloudflare support and external access, see the [upstream repository](https://github.com/onedr0p/cluster-template) and join the [Home Operations Discord](https://discord.gg/home-operations).

## 🙌 Related Projects

If this repo is too hot to handle or too cold to hold check out these following projects.

- **[onedr0p/cluster-template](https://github.com/onedr0p/cluster-template)** - _The original upstream project with Cloudflare integration and external access_
- [ajaykumar4/cluster-template](https://github.com/ajaykumar4/cluster-template) - _A template for deploying a Talos Kubernetes cluster including Argo for GitOps_
- [khuedoan/homelab](https://github.com/khuedoan/homelab) - _Fully automated homelab from empty disk to running services with a single command._
- [mitchross/k3s-argocd-starter](https://github.com/mitchross/k3s-argocd-starter) - starter kit for k3s, argocd
- [ricsanfre/pi-cluster](https://github.com/ricsanfre/pi-cluster) - _Pi Kubernetes Cluster. Homelab kubernetes cluster automated with Ansible and FluxCD_
- [techno-tim/k3s-ansible](https://github.com/techno-tim/k3s-ansible) - _The easiest way to bootstrap a self-hosted High Availability Kubernetes cluster. A fully automated HA k3s etcd install with kube-vip, MetalLB, and more. Build. Destroy. Repeat._

## 🤝 Thanks

This project is based on the excellent work by [@onedr0p](https://github.com/onedr0p) and the [cluster-template](https://github.com/onedr0p/cluster-template) community. All credit for the original template design and implementation goes to them.
