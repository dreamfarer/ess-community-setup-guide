# How to Setup ESS Community on Hetzner Cloud
This guide navigates you through setting up Element Server Suite (ESS) Community on Hetzner Cloud. At the end, you will have your own Matrix-based Element messenger running that can be accessed from web, mobile and desktop.

### Target Audience
This guide is targeted towards beginners and non-tech-savy people that just want a **ready-to-use**, **open-source**, **decentralized**, **europe-hosted**, **private** and **end-to-end encrypted** way of communication for a small community. It requires financial means of about **15€ per month** and creating an account at [Hetzner](https://www.hetzner.com/) and a domain name registrar of your choice (e.g. [Cloudflare](https://www.cloudflare.com/)).

### Disclaimer
This guide has been written in early 2026 and since I do not need to setup Element again, this guide will probably get out of sync with the official documentations. It is tailored towards MacOS, requiring you to adjust command-line interface (cli) operations slightly if you are using Linux and heavily if you are using Windows.

The offical general, up-to-date documentations can be found here: [ess-helm](https://github.com/element-hq/ess-helm), [hetzner-k3s](https://github.com/vitobotta/hetzner-k3s).

### Need help?
Don't hesitate to [open](https://github.com/dreamfarer/ess-community-setup-guide/issues/new/choose) an issue on GitHub and I will gladly try to help you.

## Prerequisites
Before following the installation instructions, you will need to complete the following:
1. Have a MacOS device with [brew](https://brew.sh/) installed
2. Create an account on [Hetzner](https://console.hetzner.com/projects) and create a new project.
3. Generate and save a *Read & Write* API token by following [this](https://docs.hetzner.com/cloud/api/getting-started/generating-api-token/) guide.
4. Buy a domain at a domain name registrar of your choice (this guide uses [Cloudflare](https://www.cloudflare.com/)).
5. Follow [this](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent?platform=mac) guide to create an SSH key if you don't already have one.

## Installation
First of all, you are going to create the configuration files needed, then you'll be setting up Kubernetes and finally you are going to install the Element Server Suite.

### 1. Configuration
Chose a permanent folder where you will store every configuration. From now on, you will always work in this folder.

1. Download [cluster.yaml](https://github.com/dreamfarer/ess-community-setup-guide/blob/main/cluster.yaml). Update `<your token>` to the Htzner API token you've generated before.

     *If you wish to further configure k3s settings, follow [this](https://vitobotta.github.io/hetzner-k3s/Creating_a_cluster/) guide for a complete example with all options.*

2. Download [hostnames.yaml](https://github.com/element-hq/ess-helm/blob/main/charts/matrix-stack/ci/fragments/quick-setup-hostnames.yaml). Replace each `your.tld` with your domain name (*e.g. example.com*).

3. Download [tls.yaml](https://github.com/element-hq/ess-helm/blob/main/charts/matrix-stack/ci/fragments/quick-setup-letsencrypt.yaml).

### 2. Setup K3S
For this step, you'll have to open your terminal at the same directory where you have stored your configurations. Use this terminal session for the entirety of the remaining guide.

1. Install `hetzner-k3s`, `kubectl` and `helm`:
    ```bash
    brew install vitobotta/tap/hetzner_k3s
    brew install kubectl
    brew install helm
    ```

2. Create your Kubernetes cluster on Hetzner. This will take about 2-3 minutes.
    ```bash
    hetzner-k3s create --config cluster.yaml
    ```

    *If you did not adjust `masters_pool` or `worker_node_pools` in `cluster.yaml`, this action will result in created services which are billed hourely when used and sum up to about 15€ per month.*

3. Once finished, setup an environment variable:
    ```bash
    export KUBECONFIG=./kubeconfig
    ```

4. Verify that your Kubernetes cluster has suggessfully been created and is running:
    ```bash
    kubectl get nodes
    ```
    Make sure it says `Ready` under the *status* value.

### 3. Setup DNS
Without closing your terminal, open your browser of choice.

1. Go to [Hetzner Console](https://console.hetzner.com/projects), then open your project and finally chose *Servers* in the left-hand bar. Copy the value under *Public IP* (*e.g. 48.161.13.12*).

2. Create six DNS records of type *A*. Follow [this](https://developers.cloudflare.com/dns/manage-dns-records/how-to/create-dns-records/) guide if you have bought your domain at Cloudflare. No matter of your domain name registrar, the DNS record table should ultimately look akin to this:

    | Type | Name               | IP            | Proxy status |
    |------|--------------------|---------------|--------------|
    | A    | `<domain>`         | `<Public IP>` | DNS only     |
    | A    | account.`<domain>` | `<Public IP>` | DNS only     |
    | A    | admin.`<domain>`   | `<Public IP>` | DNS only     |
    | A    | chat.`<domain>`    | `<Public IP>` | DNS only     |
    | A    | matrix.`<domain>`  | `<Public IP>` | DNS only     |
    | A    | mrtc.`<domain>`    | `<Public IP>` | DNS only     |

    *Make sure to replace `<Public IP>` with the value from Hetzner Console and `<domain>` with your domain name (e.g. example.com). Also make sure that you set the proxy status for each record to `DNS only` to avoid using a reverse proxy.*

### 4. Configure Certificates
Return to your terminal. Periodically run the following command until it returns the public IP address of the your Kubernetes cluster (*replace `<domain>` with your domain name*):
```bash
dig +short A chat.<domain>
```
*It may takes some time for the DNS to propagate your records. We hereby make sure that your Kubernetes cluster can be reached via your domain before continuing.*