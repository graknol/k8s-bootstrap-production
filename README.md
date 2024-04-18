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

## Configuring Azure AD for RBAC

If you want to login with Azure AD, you have to follow this guide:
[Using Azure AD to Authenticate to Kubernetes](https://medium.com/@olemarkus/using-azure-ad-to-authenticate-to-kubernetes-eb143d3cce10)

To configure the API server in microk8s, follow this guide: [Configure OIDC for a MicroK8s cluster](https://microk8s.io/docs/oidc-dex)

**TL;DR**  
- Create an Azure AD App Registration for the API server
- Create an Azure AD App Registration for kubectl
- Configure the API server to use OIDC (the first app reg.)
- Configure kubectl to trust the cluster CA, and use OIDC (the second app reg.)
- Bind the user to roles
  * I highly recommend managing this through GitOps, as everything is just YAML

## Installing Istio

For the networking layer, we'll be using Istio.

```sh
# Enable the community add-on repository
microk8s enable community

# Install Istio
microk8s.enable istio

# Alias istioctl to make it easier to use
sudo snap alias microk8s.istioctl istioctl
```

## Installing Knative

We want to run serverless loads on our cluster. And what better way to do so than to use Knative?  
Luckily for us, microk8s got us covered:

```sh
# Source: https://ubuntu.com/blog/getting-started-with-knative-1

# Enable helm
microk8s enable helm

# Tell helm where kubectl lives
kubectl config view --raw > ~/.kube/config
# Make sure the configuration file is only readable by you!
chmod go-r ~/.kube/config

# Install MetalLB
# What this service does, is it assigns external IP addresses
# to each LoadBalancer service in the cluster.
# So, MetalLB keeps track of IP addresses and assigns them to the LoadBalancer services.
# The ingress controllers in turn route the requests to the
# correct pod...
# 10.0.0.23/32 will give me a single address (remember to exclude this from DHCP!).
#
# Don't forget to port-forward 80,443 to the IP address of the Istio Gateway.
microk8s enable metallb:10.0.0.23/32

# Install knative operator
kubectl apply -f https://github.com/knative/operator/releases/download/knative-v1.13.3/operator.yaml

# Check the status of knative operator
kubectl get deployment knative-operator

# Install knative serving
# NOTE: Replace the domain with your actual domain for the knative mesh
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Namespace
metadata:
  name: knative-serving
---
apiVersion: operator.knative.dev/v1beta1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  config:
    domain:
      "knative.example.com": ""
EOF

# Check the status of knative serving
kubectl get deployment -n knative-serving

# If the READY status is True, you can continue
kubectl get KnativeServing knative-serving -n knative-serving

# Enable mTLS on the knative-serving namespace
# This will automatically add mutual TLS to all
# pods running in the namespace; neat!
kubectl label namespace knative-serving istio-injection=enabled

# Knative serving requires that we allow both mTLS and plaintext
# Source: https://knative.dev/docs/install/installing-istio/#using-istio-mtls-feature-with-knative
# Source: https://istio.io/latest/docs/reference/config/security/peer_authentication/
cat <<EOF | kubectl create -f -
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default"
  namespace: "knative-serving"
spec:
  mtls:
    mode: PERMISSIVE
EOF

# Verify that istio is installed correctly
istioctl verify-install
```

## Installing cert-manager

Certificates! We need certificates!

```sh
microk8s enable cert-manager
```

Great, now we need to configure the resource which will issue certificates on demand.
Since we most likely do not have direct access to the DNS zone, we need to use the
HTTP-01 challenge method. This just means that each and every service will be issued
a unique certificate, as only the DNS-01 challenge supports wildcard certificates.

```sh
# Source: https://knative.dev/docs/serving/encryption/enabling-automatic-tls-certificate-provisioning/#creating-a-clusterissuer

# NOTE: Modify the below command with a real email address
cat <<EOF | kubectl create -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-http01-issuer
spec:
  acme:
    privateKeySecretRef:
      name: letsencrypt
    # Specify an email address here
    email: <your.email@example.com>
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
    - http01:
       ingress:
         class: istio
EOF

# You can check the status if you want to:
kubectl get clusterissuer -n cert-manager letsencrypt-http01-issuer -o yaml
```

Next, we need to install the net-certmanager-controller, which is the bridge between
Knative and cert-manager:

```sh
# It is most likely not installed already...
kubectl get deployment net-certmanager-controller -n knative-serving

# Install it
kubectl apply -f https://github.com/knative/net-certmanager/releases/download/knative-v1.13.0/release.yaml

# Configure the controller to use the ClusterIssuer that we created earlier
kubectl patch configmap config-certmanager \
    -n knative-serving \
    --type merge \
    -p '{"data": {"issuerRef": "kind: ClusterIssuer\nname: letsencrypt-http01-issuer\n"}}'

# Configure the knative network to enable TLS on ingress (network from outside into the cluster) and redirect HTTP traffic to HTTPS
kubectl patch configmap config-network \
    -n knative-serving \
    --type merge \
    -p '{"data": {"auto-tls": "Enabled", "external-domain-tls": "Enabled", "http-protocol": "Enabled"}}'
```

### Verifying that the TLS setup works

Before we continue, we should check to see that the entire certificate workflow works:

```sh
# First make sure that istio has been assigned an IP.
# It should not say <pending> under EXTERNAL-IP, if it does
# then MetalLB has not been enabled/configured correctly.
#
# NOTE: Don't forget to port-forward 80,443 to this IP address.
kubectl get service -n istio-system istio-ingressgateway

# Deploy an example knative service
kubectl apply -f https://raw.githubusercontent.com/knative/docs/main/docs/serving/autoscaling/autoscale-go/service.yaml

# Wait until the HTTPS route is ready
kubectl get route autoscale-go

# Open the URL in your browser and see if it works

# Remove the example service
kubectl delete configuration.serving.knative.dev/autoscale-go
kubectl delete revision.serving.knative.dev/autoscale-go-00001
kubectl delete route.serving.knative.dev/autoscale-go
kubectl delete service.serving.knative.dev/autoscale-go
```

*Now, don't be like me and waste 3 hours troubleshooting this, and check that you've actually
added the DNS record to your zone... That step must still be manually performed... 🤦*

## Installing Kafka

This step will install a Kafka cluster in your Kubernetes cluster.

```sh
# First add the helm repository for Strimzi
helm repo add strimzi https://strimzi.io/charts/

# Install the strimzi operator
helm install strimzi-operator strimzi/strimzi-kafka-operator \
  --namespace kafka \
  --create-namespace
  
# Disable sidecar on kafka namespace
# Kafka terminates TLS on its own, and mTLS would result in errors.
# When Istio sidecar is disabled, it's basically just a regular
# ingress gateway (like NGINX) without all the extras that Istio provides.
kubectl label namespace kafka istio-injection=disabled

# TODO: Add steps to create a kafka cluster
# Source: https://strimzi.io/quickstarts/
```

### Creating a certificate for Kafka brokers

We need a certificate that the kafka brokers can use.

NOTE: Modify the spec below to suit your domain and subject information.
```sh
# Source: https://cert-manager.io/v1.1-docs/usage/certificate/
# Specification: https://cert-manager.io/docs/reference/api-docs/#cert-manager.io/v1.CertificateSpec
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: kafka-example-com
  namespace: kafka
spec:
  # Secret names are always required.
  secretName: kafka-example-com-tls
  duration: 2160h # 90d
  renewBefore: 720h # 30d
  # You should fill out the subject with as much information as possible
  subject:
    organizations:
    - example
  isCA: false
  privateKey:
    algorithm: ECDSA
    size: 384
  usages:
    - server auth
    - client auth
  # Here, we'll just use the same certificate for all of our nodes
  # They will live on separate subdomains (routed through NGINX Ingress Controller)
  # So, list them out here:
  dnsNames:
  - kafka-1.example.com
  - kafka-2.example.com
  - kafka-3.example.com
  # We'll use the ClusterIssuer that we created earlier
  issuerRef:
    name: letsencrypt-http01-issuer
    kind: ClusterIssuer
```