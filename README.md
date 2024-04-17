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

# Install MetalLB
# What this service does, is it assigns external IP addresses
# to each LoadBalancer service in the cluster.
# So, MetalLB keeps track of IP addresses and assigns them to the LoadBalancer services.
# The ingress controllers in turn route the requests to the
# correct pod...
# 10.0.0.23/32 will give me a single address (remember to exclude this from DHCP!).
#
# Don't forget to port-forward 80,443 to the IP address of the kourier LoadBalancer.
microk8s enable metallb:10.0.0.23/32

# Install knative on the cluster
echo 'N;' | microk8s.enable knative

# check status of knative pods
kubectl get pods -n knative-serving
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
         class: kourier.ingress.networking.knative.dev
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

# Set the default domain name, you can set other domain names based on selectors
# Source: https://knative.dev/docs/serving/using-a-custom-domain/

# NOTE: Modify the below command with a real DNS name

kubectl patch configmap config-domain \
    -n knative-serving \
    --type merge \
    -p '{"data": {"example.no": ""}}'
```

### Restart the VM for good measure

It's a good idea to restart the server at this point.

```sh
sudo shutdown -r 0
```

### Verifying that the TLS setup works

Before we continue, we should check to see that the entire certificate workflow works:

```sh
# First make sure that kourier has been assigned an IP.
# It should not say <pending> under EXTERNAL-IP, if it does
# then MetalLB has not been enabled/configured correctly.
#
# NOTE: Don't forget to port-forward 80,443 to this IP address.
kubectl get service -n knative-serving kourier

# Deploy an example knative service
kubectl apply -f https://raw.githubusercontent.com/knative/docs/main/docs/serving/autoscaling/autoscale-go/service.yaml

# Wait until the HTTPS route is ready
kubectl get route autoscale-go

# Open the URL in your browser and see if it works
```

Now, don't be like me and waste 3 hours troubleshooting this, and check that you've actually
added the DNS record to your zone... That step must still be manually performed... 🤦