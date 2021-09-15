# Equinix Metal Cookbook

Recipe for setting up a Kubernetes cluster on Equinix Metal and installing
Sextant to deploy and manage blockchain networks.

## Install Prerequisite Tools

You will need the up to date versions of the following tools installed -

* [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
* [helm](https://helm.sh/docs/intro/install/#through-package-managers)

## Setting up for Equinix Metal

* Log in to [Equinix Metal Portal](https://console.equinix.com/)
* Under user profile, set up `Personal SSH Keys` for server access
* Optionally set up [Personal API Keys](https://metal.equinix.com/developers/api/)
* Create `New Project` (we used one project to deploy all servers in all
  locations)

## Provisioning Equinix Servers for Kubernetes

* Select `Servers`/`On-Demand`.
* Choose a location (`Amsterdam`, `Chicago`, `Dallas`, ...).
* Choose a `Server Type` (_We used `c3.small.x86`_).
* Choose an Operating System. [RKE documentation lists supported
  versions](https://docs.rke2.io/install/requirements/) (_We tested RKE2 on
  `Ubuntu 18.04LTS` and `Ubuntu 20.04LTS`_).
* Select the number of servers and server names. We recommend using at least
  three servers as controllers for HA when creating Kubernetes cluster, and any
  number after can be used as agent nodes.
  (_In our project we used three servers for the Admin cluster for Sextant
  and between 5-6 for the three blockchain network clusters_).
* Optionally `Add user data` (_handy feature to customize server provisioning_).
* Optionally `Configure IPs` (_we kept defaults_).
* Optionally `Customize SSH keys` (_we are using keys already configured for the
  project_).

## Set up BGP

* Set up
  [Local BGP](https://metal.equinix.com/developers/docs/networking/local-global-bgp/)
  for the project.
* For each deployed server under Details/BGP/Manage, click on `Enable BGP`
  (_Note: you should enable BGP on at least two servers, preferably all_).

## Set up [RKE2 Kubernetes cluster](https://rancher.com/docs/rancher/v2.5/en/installation/resources/k8s-tutorials/ha-rke2/)

### Set up first node

* Log into the first server (_We used `<location>-c3-small-01` on each cluster_).
* Run the install command:

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.21 sh - ;
systemctl enable rke2-server.service;
systemctl start rke2-server.service;
```

* Gather server IP address and the generated node token:

```bash
cat /var/lib/rancher/rke2/server/node-token
```

* Add the IP address and node token to `/etc/rancher/rke2/config.yaml` file:

```bash
server: https://<server>:9345
token: <token from server node>
```

* Confirm that the first node is running:

```bash
/var/lib/rancher/rke2/bin/kubectl \
--kubeconfig /etc/rancher/rke2/rke2.yaml get nodes
```

* To access the cluster from your workstation, copy the kubeconfig file
  `/etc/rancher/rke2/rke2.yaml` to your localhost, replace `server:
  [LOAD-BALANCER-DNS]:6443` with server external IP address.

**Note**: We have tried setting up BGP/bird on systems to use for cluster load
balancing, only to find that there is a conflict with MetalLB we plan to use for
our deployments ingress. For our project, we opted to use individual system IPs
for cluster access. If the first system fails, swap that IP of the failed RKE2
Server with another RKE2 Server node in `/etc/rancher/rke2/config.yaml` on all
nodes, as well your local workstation kubeconfig and Sextant. For a Kubernetes
enterprise cluster, we strongly recommend setting up a load balancer for the
cluster access. One solution is to use [Equinix guide to set up HAProxy load
balancer outside the clusters](https://metal.equinix.com/developers/guides/load-balancing-ha/)

### Set up the remaining RKE2 Server or Agent nodes

* Log into the other servers (at least two more servers), copy
  `/etc/rancher/rke2/config.yaml` file from the first server, and execute steps:

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.21 sh - ;
systemctl enable rke2-server.service;
systemctl start rke2-server.service
```

* Optionally for fourth+ servers enable/start `rke2-agent.service`
  instead of `rke2-server.service`:

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.21 sh - ;
systemctl enable rke2-agent.service;
systemctl start rke2-agent.service
```

## Install [Longhorn](https://longhorn.io/docs/1.2.0/deploy/install/install-with-helm/)

* Add Helm chart repo:

```bash
helm repo add longhorn https://charts.longhorn.io
```

* Install longhorn:

```bash
kubectl create namespace longhorn-system;
helm -n longhorn-system install longhorn longhorn/longhorn
```

## Install [MetalLB](https://metallb.universe.tf)

* Enable BPG for all systems in a cluster via Equinix Metal console.
* Request external IP range in the same region via Equinix Metal console (_We
  chose /30 for each cluster_).
* Gather hostnames and internal (`bond0:0`) addresses.
* Create YAML files for the cluster:

```yaml
controller:
  image:
    tag: main
speaker:
  image:
    tag: main

configInline:
  peers:
    - peer-address: 169.254.255.1
      source-address: <system 1 bond0:0 IP address>
      router-id: <system 1 bond0:0 IP address>
      peer-asn: 65530
      my-asn: 65000
      password: <"Project BGP password">
      node-selectors:
        - match-labels:
            kubernetes.io/hostname: <system 1 hostname>
    - peer-address: 169.254.255.2
      source-address: <system 1 bond0:0 IP address>
      router-id: <system 1 bond0:0 IP address>
      peer-asn: 65530
      my-asn: 65000
      password: <"Project BGP password">
      node-selectors:
        - match-labels:
            kubernetes.io/hostname: <system 1 hostname>

    - peer-address: 169.254.255.1
      source-address: <system 2 bond0:0 IP address>
      router-id: <system 2 bond0:0 IP address>
      peer-asn: 65530
      my-asn: 65000
      password: <"Project BGP password">
      node-selectors:
        - match-labels:
            kubernetes.io/hostname: <system 2 hostname>
    - peer-address: 169.254.255.2
      source-address: <system 2 bond0:0 IP address>
      router-id: <system 2 bond0:0 IP address>
      peer-asn: 65530
      my-asn: 65000
      password: <"Project BGP password">
      node-selectors:
        - match-labels:
            kubernetes.io/hostname: <system 2 hostname>

  address-pools:
    - name: default
      auto-assign: true
      protocol: bgp
      addresses:
      - <external IP address range requested/30>
```

* Add Helm chart repo:

```bash
helm repo add metallb https://metallb.github.io/metallb
```

* Install MetalLB:

```bash
kubectl create namespace metallb-system;
helm -n metallb-system install metallb metallb/metallb \
  -f <config file from step 2.yaml>
```

## Installing Sextant on Equinix Metal

### <a name="sign-up"></a>Sign up for Evaluation

If you haven't done so already please sign up for an evaluation
[here](https://www.blockchaintp.com/sextant/equinix-metal) in order to obtain
credentials

### Prepare Cluster

* Use
  [helm repo](https://helm.sh/docs/intro/using_helm/#helm-repo-working-with-repositories)
  command to add BTP repo:

```bash
helm repo add sextant https://btp-charts-stable.s3.amazonaws.com/charts/
```

* Create namespace `sextant` for Sextant:

```bash
kubectl create namespace sextant
kubectl config set-context --current --namespace=sextant
```

* Assuming you've signed up for an [evaluation](#sign-up) above, use the
  credentials provided by BTP to create a Kubernetes secret so you can access
  the BTP repo:

```bash
CLIENT_UNAME=<'client name'>
CLIENT_EMAIL=<'client email'>
CLIENT_PWORD=<'client password'>

kubectl create secret docker-registry btp-lic \
  --docker-server=https://dev.catenasys.com:8084/ --docker-username=$CLIENT_UNAME \
  --docker-password=$CLIENT_PWORD --docker-email=$CLIENT_EMAIL
```

* Create Sextant helm chart values file `values-equinix-demo.yaml`:

```yaml
sextant:
  imagePullSecrets:
    enabled: true
    value:
      - name: btp-lic
  postgres:
    enabled: true
    image:
      repository: postgres
      tag: 11
    user: postgres
    host: localhost
    database: postgres
    port: 5432
    password: postgres
    tls:
    persistence:
      enabled: true
      annotations: {}
      accessModes:
        - "ReadWriteOnce"
      storageClass: "longhorn"
      size: "40Gi"
global:
  storageClass: longhorn
persistence:
  enabled: true
  size: 40Gi
networkPolicy:
  enabled: true
```

### Install Sextant Enterprise

* Install Sextant using `helm`:

```bash
helm install -f values-equinix-demo.yaml equinix-demo sextant/sextant-enterprise
```

The output should look something like this:

```text
NAME: equinix-demo
LAST DEPLOYED: Mon Aug 23 00:51:36 2021
NAMESPACE: sextant
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Get the initial Sextant application username and password by running this command
  kubectl describe pod/equinix-demo-sextant-enterprise-0|grep INITIAL_

2. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods -l "app=sextant-enterprise" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl port-forward $POD_NAME 8080:80

```

* Save Admin Credentials

Run this command:

```bash
kubectl describe pod/equinix-demo-sextant-enterprise-0|grep INITIAL_
```

Make a note of the username and password for admin access to
Sextant | Enterprise. You will need these to log into Sextant | Enterprise.
Note that these details will persist even if you restart or delete/reinstall
Sextant | Enterprise.

### Accessing Sextant

#### Option 1 - Port Forwarding

You can use port forwarding using this command:

```bash
kubectl port-forward equinix-demo-sextant-enterprise-0 8080:80
```

Connect to Sextant

```bash
http://localhost:8080
```

#### Option 2 - Using Load Balancer

If you want a persistent connection to your Sextant | Enterprise instance,
you will need to create a load balancer.

(_Note that while this is acceptable for this evaluation we recommend setting
up a k8s ingress controller for long term access._)

```bash
kubectl expose pod/equinix-demo-sextant-enterprise-0 --type=LoadBalancer \
  --name=equinix-demo-sextant-enterprise-0-lb --port=80 --target-port=80
```

Obtain external IP:

```bash
kubectl get all -o wide | grep LoadBalancer | awk '{print $4}'
```

Connect to Sextant:

```bash
http://<EXTERNAL-IP>:80
```
