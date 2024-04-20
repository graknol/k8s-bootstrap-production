# Bootstrap K8S

Follow this guide to bootstrap a production-ready Kubernetes cluster.

## Prerequisites

You need the following:

- SSH access to a fresh installation of Ubuntu Server
  - microk8s _just works™️_ on Ubuntu, so best to just stick with it
- Time ⏰

## Enabling Unattended Upgrades

Before we continue, we should enable a critical security feature, namely the _unattended-upgrades_ package.
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

# Enable the community add-on repository
microk8s enable community

# Tell istio and helm where kubectl lives
kubectl config view --raw > ~/.kube/config
# Make sure the configuration file is only readable by you!
chmod go-r ~/.kube/config
```

## Configuring Azure AD for RBAC

[Dex](https://github.com/dexidp/dex), no questions.

## Installing Istio

For the networking layer, we'll be using Istio.

```sh
# Download the lastest release
curl -L https://istio.io/downloadIstio | sh -

# Add istio to your PATH
tee -a ~/.bashrc << EOF
# Add istioctl to path
export PATH="\$PATH:$(find ~+ -maxdepth 1 -type d -name 'istio-*' | sort -r | head -1)/bin"
EOF
source ~/.bashrc

# Deploy istio to our cluster
istioctl install --set profile=default -y
```

## Installing MetalLB

What this service does, is it assigns external IP addresses to each LoadBalancer service in the cluster.

It elects one node to start advertising on Layer 2;

> "Hey, it's me! 10.0.0.23, I'm on the MAC address AA:BB:CC:DD:EE"

If that node ever dies, then MetalLB will elect a new node to start advertising that same IP. Clever, right?

```sh
# 10.0.0.23/32 will give me a single address (remember to exclude this from DHCP!).
# Don't forget to port-forward 80,443 to the IP address of the istio-system/istio-ingressgateway LoadBalancer.
microk8s enable metallb:10.0.0.23/32
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

_Now, don't be like me and waste 3 hours troubleshooting this, and check that you've actually
added the DNS record to your zone... That step must still be manually performed... 🤦_

## Automatic HTTP -> HTTPS redirect

Since we're using the HTTP-01 challenge type to get certificates from Let's Encrypt,
we can't simply redirect all traffic from port 80 to 443,
because then how will Let's Encrypt be able to get the verification token from our server?

There's a trick!
We create a dedicated gateway that matches all requests on port 80.
Then you filter out the ones that request the `/.well-known/acme-challenge` path.
The remaining requests are redirected to HTTPS. Simple, yet efficient.

_NOTE: To make this work, you'll need to make sure you do not redirect port 80 in
your other gateways. For obvious reasons, that would break the HTTP-01 challenge again._

```sh
# Create the HTTP -> HTTPS gateway and virtual service
# NOTE: This is what allows us to redirect, without breaking the HTTP-01 challenge
cat <<EOF | kubectl apply -f -
kind: Gateway
apiVersion: networking.istio.io/v1beta1
metadata:
  name: http-to-https-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        protocol: HTTP
        name: http
      hosts:
        - '*'
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: redirect-to-https-except-http01
  namespace: istio-system
spec:
  gateways:
    - istio-system/http-to-https-gateway
  hosts:
    - "*"
  # Redirect request to HTTPS if it's not a HTTP-01 challenge
  http:
    - match:
        - withoutHeaders:
            :path:
              prefix: /.well-known/acme-challenge/
      redirect:
        scheme: https
EOF
```

## Continuous Deployment

Hands down, the best system out there: [Argo CD](https://github.com/argoproj/argo-cd)

Installing it is easy, let's get to it!

```sh
# Source: https://argo-cd.readthedocs.io/en/stable/getting_started/#1-install-argo-cd
# Source: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#istio

# NOTE: Do NOT change the namespace name, as there are ClusterRoleBinding resources that reference it in the installation manifest
kubectl create namespace argocd

# Enable mTLS on the argocd namespace
kubectl label namespace argocd istio-injection=enabled

# Allow both mTLS and plaintext
cat <<EOF | kubectl create -f -
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default"
  namespace: "argocd"
spec:
  mtls:
    mode: PERMISSIVE
EOF

# Create a directory for our kustomization
rm -rf ~/kustomization-argocd
mkdir ~/kustomization-argocd
pushd ~/kustomization-argocd

# Download the installation manifest
curl -kLs -o install.yaml https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Create our kustomization
tee kustomization.yaml << EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ./install.yaml
patches:
- path: ./patch.yaml
EOF

# Create our patch
tee patch.yaml << EOF
# Use --insecure so Ingress can send traffic with HTTP
# env was added because of https://github.com/argoproj/argo-cd/issues/3572 error
---
apiVersion: apps/v1
kind: Deployment
metadata:
 name: argocd-server
spec:
 template:
   spec:
     containers:
     - args:
       - /usr/local/bin/argocd-server
       - --insecure
       name: argocd-server
       env:
       - name: ARGOCD_MAX_CONCURRENT_LOGIN_REQUESTS_COUNT
         value: "0"
EOF

popd

# Install Argo CD with our patches into the namespace
kubectl apply -k ~/kustomization-argocd -n argocd --wait=true

# Create the certificate
# NOTE: Replace the example values below with your own
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: argo-example-com
  # Must be created in the same namespace as the istio-ingressgateway deployment
  namespace: istio-system
spec:
  # Secret names are always required.
  secretName: argo-example-com-tls
  duration: 2160h # 90d
  renewBefore: 720h # 30d
  # You should fill out the subject with as much information as possible
  subject:
    organizations:
    - Example LLC
  isCA: false
  privateKey:
    algorithm: ECDSA
    size: 256
    # Why not P-384? Well, I spent 20+ hours trying to
    # get TLS working through Istio...
    # and it turns out that Envoy only supports P-256 🤦
    # Source: https://github.com/envoyproxy/envoy/issues/10855#issuecomment-618023133
  usages:
    - server auth
    - client auth
  commonName: argo.example.com
  dnsNames:
  - argo.example.com
  issuerRef:
    name: letsencrypt-http01-issuer
    kind: ClusterIssuer
EOF

# Create the gateway
cat <<EOF | kubectl apply -f -
kind: Gateway
apiVersion: networking.istio.io/v1beta1
metadata:
  name: argo-gateway
  # We want to keep gateways separate from apps (security)
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        protocol: HTTPS
        name: https
      hosts:
        - argo.example.com
      tls:
        mode: SIMPLE
        credentialName: argo-example-com-tls
    # IMPORTANT!
    # If you include port 80 in your gateways,
    # then the HTTP -> HTTPS redirect will work,
    # but the ACME challenge HTTP-01 will break.
    # Let the dedicated gateway do the HTTP -> HTTPS
    # redirection for you,
    # and just don't think about it. :-)
EOF

# Create the virtual service to route traffic to Argo CD
# NOTE: Modify the domain below
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: argo-virtualservice
  namespace: argocd
spec:
  hosts:
    - argo.example.com
  gateways:
    - istio-system/argo-gateway
  http:
  - route:
    - destination:
        host: argocd-server
        port:
          number: 80
EOF
```

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
cat <<EOF | kubectl create -f -
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
    size: 256
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
EOF
```

## Secrets Management

I'll just link to these for now:

- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets/tree/main)
  - Let's you put secrets in Git
- [External Secrets](https://github.com/external-secrets/external-secrets)
  - Let's you pull secrets from Azure Key Vault
- [Reloader](https://github.com/stakater/Reloader)
  - Reloads your pods when ConfigMaps and Secrets are updated.
  - _NOTE: Reloads due to certificate renewals are handled by cert-manager_
