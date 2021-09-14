# Equinix Metal Cookbook

Recipe for setting up a Kubernetes cluster on Equinix Metal and installing Sextant
for using and deploying blockchain networks.

## Setting up for Equinix Metal

* [Log in to Equinix Metal Portal](https://console.equinix.com/)
* Under user profile, set up "Personal SSH Keys" for server access
* [Optionally set up Personal API Keys](https://metal.equinix.com/developers/api/)
* Create "New Project" (we used one project to deploy all servers in all
  locations)

## Provisioning Equinix Servers

* Select Servers/On-Demand
* Choose a location (Amsterdam, Chicago, Dallas, etc.)
* Choose a Server Type (we used `c3.small.x86`)
* Choose an Operating System. [RKE documentation lists supported
  versions](https://docs.rke2.io/install/requirements/) (we tested RKE2 on
  `Ubuntu 18.04LTS` and `Ubuntu 20.04LTS`)
* Select the number of servers and server names. We recommend using at least three servers nodes for HA, and any number after can be used as agent nodes.
  (we used three servers for the Admin cluster for Sextant and between 5-6 for
  the three blockchain network clusters).
* Optionally "Add user data" (handy feature to customize server provisioning)
* Optionally "Configure IPs" (we kept defaults)
* Optionally "Customize SSH keys" (we are using keys already configured for the
  project)

## Set up BGP

* [Set up Local BGP for the project](https://metal.equinix.com/developers/docs/networking/local-global-bgp/)
* For each deployed server under Details/BGP/Manage, click on "Enable BGP"
  (_Note_: you should enable BGP on at least two servers, preferably all)

## Set up [RKE2 Kubernetes cluster](https://rancher.com/docs/rancher/v2.5/en/installation/resources/k8s-tutorials/ha-rke2/)

### Set up first node

* Log into the first server (we used `<location>-c3-small-01` on each cluster)
* Run the install command:

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.21 sh - ;
systemctl enable rke2-server.service;
systemctl start rke2-server.service;
```

* Gather server IP address and the generated node token.

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
  [LOAD-BALANCER-DNS]:6443` with server external IP address

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
systemctl start rke2-server.service;
```

* Optionally for fourth+ servers enable/start `rke2-agent.service` instead of `rke2-server.service`

```bash
systemctl enable rke2-agent.service
systemctl start rke2-agent.service
```

## [Install Longhorn](https://longhorn.io/docs/1.2.0/deploy/install/install-with-helm/)

* Add Helm chart repo

```bash
helm repo add longhorn https://charts.longhorn.io
```

* Install longhorn

```bash
kubectl create namespace longhorn-system
helm -n longhorn-system install longhorn longhorn/longhorn
```

## [Install MetalLB](https://metallb.universe.tf)

* Enable BPG for all systems in a cluster via Equinix Metal console
* Request external IP range in the same region via Equinix Metal console (we
  chose /30 for each cluster)
* Gather hostnames and internal (`bond0:0`) addresses
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

* Add Helm chart repo

```bash
helm repo add metallb https://metallb.github.io/metallb
```

* Install MetalLB

```bash
kubectl create namespace metallb-system
helm -n metallb-system install metallb metallb/metallb \
  -f <config file from step 2.yaml>
```

## Setting up Sextant
[Link to Sextant Install goes here](https://docs.blockchaintp.com/en/latest/sextant/daml/overview/)
