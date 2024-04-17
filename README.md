# Bootstrap K8S
Follow this guide to bootstrap a production-ready Kubernetes cluster.

## Prerequisites

You need the following:
- SSH access to a fresh installation of Ubuntu Server
    * microk8s *just works™️* on Ubuntu, so best to just stick with it
- Time ⏰

## Enabling Unattended Upgrades

Before we continue, we should enable a critical security feature, namely the *unattended-upgrades* package.
This will keep critical packages up-to-date automatically, **including the Linux kernel**.

This effectively gives us a zero-maintenance server. Just forget about it, and it'll happily keep itself up-to-date and secure.

```sh
sudo apt update -y
sudo apt install unattended-upgrades -y

# Set the timezone to Norway
# This is so that the automatic reboot happens when we expect
sudo timedatectl set-timezone Europe/Oslo

# Configure the unatteded-upgrades package
# This is interactive, and the easiest way to configure it
sudo dpkg-reconfigure -plow unattended-upgrades

# Source: https://serverfault.com/a/1044131
# You should enable Automatic-Reboot and set the reboot time to 02:00
# TODO: Add the fully configured files (20 and 50) to this GitHub repository and use wget -O ... to pull it and configure it automatically; skipping the interactive step above.
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

## Installing microk8s

Installing microk8s is really easy:

```sh
# Source: https://ubuntu.com/blog/getting-started-with-knative-1

sudo snap install --classic microk8s
microk8s.status --wait-ready
sudo snap alias microk8s.kubectl kubectl
```

## Installing Knative

We want to run serverless loads on our cluster. And what better way to do so than to use Knative?  
Luckily for us, microk8s got us covered:

```sh
# Source: https://ubuntu.com/blog/getting-started-with-knative-1

# Enable the community add-on repository
microk8s enable community

# Install knative on the cluster
echo 'N;' | microk8s.enable knative

# check status of knative pods
kubectl get pods -n knative-serving
```

## Installing cert-manager

Certificates! We need certificates!

```sh
# Install helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Tell helm where kubectl lives
kubectl config view --raw > ~/.kube/config
# Make sure the configuration file is only readable by you!
chmod go-r ~/.kube/config

# Add the jetstack repository, which is where cert-manager lives
helm repo add jetstack https://charts.jetstack.io --force-update
helm repo update

# Install cert-manager
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

Great, now we need to configure the resource which will issue certificates on demand.
Since we most likely do not have direct access to the DNS zone, we need to use the
HTTP-01 challenge method. This just means that each and every service will be issued
a unique certificate, as only the DNS-01 challenge supports wildcard certificates.

```sh
kubectl apply -f https://raw.githubusercontent.com/graknol/k8s-bootstrap-production/main/yaml/http-01-cluster-issuer.yaml
```