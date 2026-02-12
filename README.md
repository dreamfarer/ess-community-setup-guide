# How to Setup ESS Community on Hetzner Cloud
This guide walks you through setting up Element Server Suite (ESS) Community on Hetzner Cloud. By the end of this guide, you will be able to use Element to its full extent on the platform of your choice.

### What is Element?

Element is an **open-source**, **decentralized**, **end-to-end encrypted**, **cross-platform messaging** application built on the Matrix protocol. It is available natively for Windows, macOS, Linux, Android, and iOS, and also offers a web client for any browser.

Element combines many of the features we love from other messaging apps: the privacy and anonymity of Signal; servers (_called spaces_), chat and voice channels, including video and screen sharing, and fine-grained permissions similar to Discord; and polls and live-location sharing like WhatsApp.

### Target Audience

This guide is aimed at beginners and non-tech-savvy users who want a robust and **ready-to-use** solution for a small community quickly. If needed, adjusting the configuration later is straightforward and non-destructive.

### Investment
You will need approximately **$15 per month**, an account at [Hetzner](https://www.hetzner.com/), and a domain purchased from a registrar of your choice (_e.g., [Cloudflare](https://www.cloudflare.com/)_). For beginners, completing this guide is expected to take between 1–3 hours.

### Disclaimer

This guide was written in early 2026. Since I do not regularly set up Element, parts of it may become outdated over time. It is tailored to macOS, so you may need to slightly adjust command-line interface (CLI) operations if you are using Linux, and more significantly if you are using Windows.

The official, up-to-date documentation can be found here: [ess-helm](https://github.com/element-hq/ess-helm), [hetzner-k3s](https://github.com/vitobotta/hetzner-k3s).

### Need help?
Don't hesitate to [open](https://github.com/dreamfarer/ess-community-setup-guide/issues/new/choose) an issue on GitHub, and I will gladly try to help you.

### Want to help?

[Open](https://github.com/dreamfarer/ess-community-setup-guide/issues/new/choose) an issue on GitHub and let me know where you struggled, or fork this repository, make improvements directly, and submit a pull request.

## Prerequisites
Before following the installation instructions, you will need to complete the following:
1. Have a macOS device with [brew](https://brew.sh/) installed.
2. Create an account on [Hetzner](https://console.hetzner.com/projects) and create a new project.
3. Generate and save a _Read & Write_ API token by following [this](https://docs.hetzner.com/cloud/api/getting-started/generating-api-token/) guide.
4. Buy a domain at a domain name registrar of your choice (_this guide uses [Cloudflare](https://www.cloudflare.com/)_).
5. Follow [this](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent?platform=mac) guide to create an SSH key if you don't already have one.

## Installation
First, you are going to create the configuration files needed, then you'll be setting up Kubernetes, and finally, you are going to install the Element Server Suite.

### 1. Configuration
Chose a **permanent** folder where you will store every configuration. For the entire guide, you will always work in this folder.

1. Download [cluster.yaml](https://github.com/dreamfarer/ess-community-setup-guide/blob/main/cluster.yaml). Replace `<your token>` with the Hetzner API token you've generated before.

     *If you wish to further configure k3s settings (e.g., adjust server location), follow [this](https://vitobotta.github.io/hetzner-k3s/Creating_a_cluster/) guide for a complete example with all options.*

2. Download [hostnames.yaml](https://github.com/element-hq/ess-helm/blob/main/charts/matrix-stack/ci/fragments/quick-setup-hostnames.yaml). Replace each `your.tld` with your domain name (*e.g., example.com*).

3. Download [tls.yaml](https://github.com/element-hq/ess-helm/blob/main/charts/matrix-stack/ci/fragments/quick-setup-letsencrypt.yaml).

4. Download [synapse.yaml](https://github.com/dreamfarer/ess-community-setup-guide/blob/main/synapse.yaml).

    _Find all available configuration options [here](https://element-hq.github.io/synapse/latest/usage/configuration/config_documentation.html), if you wish to customize Synapse._

### 2. Set up K3S
For this step, you'll have to open your terminal at the same directory where you have stored your configurations. Use this terminal session for the entirety of the remaining guide.

1. Install `hetzner-k3s`, `kubectl` and `helm`:
    ```bash
    brew install vitobotta/tap/hetzner_k3s
    brew install kubectl
    brew install helm
    ```

2. Create your Kubernetes cluster on Hetzner. This will take about 2–3 minutes.
    ```bash
    hetzner-k3s create --config cluster.yaml
    ```

    _If you did not adjust `masters_pool` or `worker_node_pools` in `cluster.yaml`, this action will result in created services which are billed hourely when used and sum up to about 15$ per month._

3. Once finished, set up an environment variable:
    ```bash
    export KUBECONFIG=./kubeconfig
    ```
    _Be advised that this environment variable will have to be set every time you open a new terminal session._

4. Verify that your Kubernetes cluster has successfully been created and is running:
    ```bash
    kubectl get nodes
    ```
    Make sure it says `Ready` under the _status_ value.

### 3. Set up DNS
Without closing your terminal, open your browser of choice.

1. Go to [Hetzner Console](https://console.hetzner.com/projects), then open your project, and finally chose *Servers* in the left-hand bar. Copy the value under *Public IP* (*e.g., 42.161.13.12*).

2. Create six DNS records of type *A*. Follow [this](https://developers.cloudflare.com/dns/manage-dns-records/how-to/create-dns-records/) guide if you have bought your domain at Cloudflare. No matter of your domain name registrar, the DNS record table should ultimately look akin to this:

    | Type | Name               | IP            | Proxy status |
    |------|--------------------|---------------|--------------|
    | A    | `<domain>`         | `<Public IP>` | DNS only     |
    | A    | account.`<domain>` | `<Public IP>` | DNS only     |
    | A    | admin.`<domain>`   | `<Public IP>` | DNS only     |
    | A    | chat.`<domain>`    | `<Public IP>` | DNS only     |
    | A    | matrix.`<domain>`  | `<Public IP>` | DNS only     |
    | A    | mrtc.`<domain>`    | `<Public IP>` | DNS only     |

    *Make sure to replace `<Public IP>` with the value from Hetzner Console and `<domain>` with your domain name (e.g., example.com). Also make sure that you set the proxy status for each record to `DNS only` to avoid using a reverse proxy.*

### 4. Configure Certificates
Return to your terminal. Periodically run the following command until it returns the public IP address of your Kubernetes cluster (*replace `<domain>` with your domain name*):
```bash
dig +short A chat.<domain>
```
_It may take some time for the DNS to propagate your records. We hereby make sure that your Kubernetes cluster can be reached via your domain before continuing._

1. Install cert-manager using _Helm_:
    ```bash
    helm install \
    cert-manager oci://quay.io/jetstack/charts/cert-manager \
    --version v1.19.3 \
    --namespace cert-manager \
    --create-namespace \
    --set crds.enabled=true
    ```
   _Find the latest version of cert-manager [here](https://artifacthub.io/packages/helm/jetstack/cert-manager)._

2. Verify that cert-manager is running:
    ```bash
    kubectl get pods -n cert-manager
    ```
   Make sure that all pods are in the `Running` state.

3. Create the Let's Encrypt issuer:
   ```bash
   kubectl apply -f - <<EOF
   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
     name: letsencrypt-prod
   spec:
     acme:
       server: https://acme-v02.api.letsencrypt.org/directory
       privateKeySecretRef:
         name: letsencrypt-prod-private-key
       solvers:
       - http01:
           ingress:
             class: traefik
   EOF
   ```

4. Verify that the issuer has been created:
    ```bash
    kubectl get clusterissuer letsencrypt-prod
    ```
    Make sure that the issuer is ready.

### 5. Set up Element Server Suite (ESS)
1. Create the `ess` namespace:
    ```bash
    kubectl create namespace ess
    ```

2. Install ESS using _Helm_:
    ```bash
    helm upgrade --install \
    --namespace "ess" \
    ess \
    oci://ghcr.io/element-hq/ess-helm/matrix-stack \
    -f hostnames.yaml \
    -f tls.yaml \
    -f synapse.yaml \
    --wait
    ```
   _Add optional additional configuration files by appending `-f <file>` to the command. The same command is also used to update ESS after changes to configuration files._

3. Create your first user:
    ```bash
    kubectl exec -n ess -it \
    deploy/ess-matrix-authentication-service \
    -- mas-cli manage register-user
    ```
   _Make sure to select `Create user` after entering your desired username and password, else the user will not be created. It might also be beneficial to make this user an admin._

4. Either download the desktop or mobile app or navigate to `https://app.<domain>`. Enjoy Element!