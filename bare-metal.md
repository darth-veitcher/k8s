Assumptions:

* Base install of Ubuntu 18.04 (LTS) - this may work on earlier versions but I haven't tested. Other linux distros likely to also work with sensible modifications for package managers etc.
* Using k3s as the distribution (most things will work if kubeadm)

# Basic hardening and setup
First, let's set the hostname to the primary fqdn we want to access the box on.

```bash
export FQDN=asimov.jamesveitch.dev
sudo hostnamectl set-hostname $FQDN
```

## Local user
Create local admin user, add to sudo and ensure can access kubectl then login as them. This user will not have a password assigned so you'll need to copy a SSH key across with `ssh-copy-id` (remote) or generating a key locally (not recommended) and then copying the public key into `~/.ssh/authorized_keys`.

```bash
# set this user as your cloudbox user for ease
export LOCAL_ADMIN=adminlocal
sudo useradd -m \
  -c "local administrative user" \
  -s /bin/bash \
  $LOCAL_ADMIN
echo "$LOCAL_ADMIN ALL=(ALL:ALL) NOPASSWD:ALL" | sudo tee -a /etc/sudoers > /dev/null

su --shell /bin/bash --login $LOCAL_ADMIN
```

Now as the user copy our public key in.

```bash
mkdir -p ~/.ssh
tee ~/.ssh/authorized_keys > /dev/null <<EOF
contentsofyoursshpublickeygohere
EOF
```

Check you can login via ssh as the user with that key before proceeding further...

## Firewall + Fail2Ban
```bash
# Install ufw, set defaults, enable SSH + HTTPS only
sudo apt-get update; \
sudo apt-get install ufw; \
sudo ufw default deny incoming; \
sudo ufw default allow outgoing; \
sudo ufw allow 22/tcp; \
sudo ufw allow 443/tcp; \
sudo ufw allow 80/tcp; \
sudo ufw enable; \
sudo ufw status
```

Now install Fail2Ban
```bash
sudo apt-get install -y fail2ban
```

Remove the root login ability in `/etc/ssh/sshd_config` and only allow our $LOCAL_ADMIN user with a publickey.

```bash
sudo sed -i 's/^.*PermitRootLogin.*$/PermitRootLogin no/g' /etc/ssh/sshd_config; \
sudo sed -i 's/^.*PasswordAuthentication.*$/PasswordAuthentication no/g' /etc/ssh/sshd_config; \
sudo sed -i 's/^.*PubkeyAuthentication.*$/PubkeyAuthentication yes/g' /etc/ssh/sshd_config

# This assumes we're logged in as $LOCAL_ADMIN, else change $USER to compensate
echo AllowUsers $USER | sudo tee > /dev/null -a /etc/ssh/sshd_config
```

Now restart the daemon

```bash
sudo systemctl daemon-reload
sudo systemctl restart sshd
```

Trying to ssh into `localhost` should give a `Permission denied (publickey)` error. Hopefully an attempt with the public key should work... 

# Enable Avahi (Discovery) on VPN and LAN
As per [Wikipedia](https://en.wikipedia.org/wiki/Avahi_%28software%29).

>Avahi is a free zero-configuration networking (zeroconf) implementation, including a system for multicast DNS/DNS-SD service discovery. It is licensed under the GNU Lesser General Public License (LGPL).
>
>Avahi is a system which enables programs to publish and discover services and hosts running on a local network. For example, a user can plug a computer into a network and have Avahi automatically advertise the network services running on its machine, facilitating user access to those services.

We will setup our nodes to publish themselves on both the LAN (so can be accessed via their hostnames) and VPN (optional).

```bash
# from docker
export DEBIAN_FRONTEND=noninteractive; \
sudo apt-get update -y; \
sudo apt-get -qq install -y avahi-daemon avahi-utils
```

The main configuration is held in `/etc/avahi/avahi-daemon.conf`. We can modify the `allow-interfaces` line (to limit which interfaces we advertise on). This will be useful for when we want to only enable it on *internal* interfaces (e.g. our Cloud node shouldn't try and broadcast across the internet). We'll leave this for now.

```bash
# only allow certain interfaces
sudo sed -i 's/^.*allow-interfaces.*$/allow-interfaces=bond0, wg0, docker0, kilo0/g' /etc/avahi/avahi-daemon.conf
```

Reload / restart the daemon

```bash
sudo systemctl daemon-reload; \
sudo systemctl restart avahi-daemon.service
```

# Setup Wireguard (VPN)
See [Wireguard vs OpenVPN on a local Gigabit Network](https://snikt.net/blog/2018/12/13/wireguard-vs-openvpn-on-a-local-gigabit-network/) for a performance comparison. I've gone with Wireguard over OpenVPN based on it being incorporated into the Linux Kernel and increased performance versus OpenVPN. In addition, there's a useful walkthrough on [How to setup your own VPN server using WireGuard on Ubuntu](https://securityespresso.org/tutorials/2019/03/22/vpn-server-using-wireguard-on-ubuntu/) that I leaned on during this process.

## Install Wireguard
```bash
sudo add-apt-repository -y ppa:wireguard/wireguard; \
sudo apt-get install -y wireguard; \
sudo modprobe wireguard  # activate kernal module
```

???+ check "Check Kernel Module"
    To check if the module is loaded use `lsmod | grep wireguard`. You should see something like the below.

    ```bash
    root@banks:~# lsmod | grep wireguard
    wireguard             212992  0
    ip6_udp_tunnel         16384  1 wireguard
    udp_tunnel             16384  1 wireguard
    ```

Configure a ufw profile for it to allow through the firewall.

```bash
sudo tee /etc/ufw/applications.d/wireguard <<EOF
# file: /etc/ufw/applications.d/wireguard
# Used to configure the ufw definition for wireguard vpn server

[wireguard]
title=Wireguard VPN Server
description=WireGuard is an extremely simple yet fast and modern VPN that utilizes state-of-the-art cryptography
ports=51820
EOF

sudo ufw app update wireguard
sudo ufw allow wireguard
```
<!-- 
## Keys
You will need to generate a key-pair for every peer (device) that is connected, including things like mobile phones etc. The iOS WireGuard client allow you to generate the keys on the device itself (if you want).

```bash
# Generate public/private keypair
sudo -s  # do this as root
cd /etc/wireguard; \
umask 077; \
wg genkey | sudo tee privatekey | wg pubkey | sudo tee publickey
exit  # drop back to your user
```

## Configure
We need to create a network interface now for the wireguard VPN. Common convention is to use `wg0` as a name for this. In addition we also need to choose a `subnet` for the VPN addresses. As I've got a `192.168.0.1/24` configuration at home I'll use `10.10.0.1/24` for the VPN.

Note the highlighted IP address we assign to each node here. It will need to be incremented for each to provide a unique address. In addition I'm dropping in the primary network interface based on the default routing but if this isn't what you want replace the inline code with your preferred network interface.

**NB:** `mypublickeycontents` should be the contents of the public key you generated on your **home** server so that the seedbox can authenticate it.

```bash
sudo -s  # run the below commands as root
export HOME_PUBLIC_KEY=mypublickeycontents
sudo tee /etc/wireguard/wg0.conf <<EOF
# file: /etc/wireguard/wg0.conf
# Server configuration (we allow peers to connect into us)
[Interface]
Address = 10.10.0.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o $(ip -4 route | grep default | awk '{print $5}') -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o $(ip -4 route | grep default | awk '{print $5}') -j MASQUERADE
PrivateKey = $(cat /etc/wireguard/privatekey)

[Peer]
PublicKey = $HOME_PUBLIC_KEY
AllowedIPs = 10.10.0.2/32
EOF
exit  # drop back to your user
```

Enable forwarding of packets in the host kernel.


```bash
sudo tee /etc/sysctl.conf << EOF
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
EOF
sudo sysctl -p
```

Finally we can start the `wg0` interface.

```bash
wg-quick up wg0
```

Hopefully you'll see something like the below output.

```bash
root@banks:/etc/wireguard# wg-quick up wg0
[#] ip link add wg0 type wireguard
[#] wg setconf wg0 /dev/fd/63
[#] ip -4 address add 10.10.0.1/24 dev wg0
[#] ip -6 address add fd86:ea04:1111::1/64 dev wg0
[#] ip link set mtu 1420 up dev wg0
[#] iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o bond0 -j MASQUERADE; ip6tables -A FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -A POSTROUTING -o bond0 -j MASQUERADE
```

??? check "Checking status"
    The check the status of wireguard run the `wg` command.

    ```bash
    root@banks:/etc/wireguard# wg
    interface: wg0
        public key: I6ZHsLe44SHNH44xE86AI0VEnm8CfzrQUrxSCJVjAEw=
        private key: (hidden)
        listening port: 51820
    ```

    In addition, we should now have an additional `route` appear for our VPN subnet.

    ```bash hl_lines="3"
    root@banks:/etc/wireguard# ip route
    default via 192.168.0.1 dev bond0 proto dhcp src 192.168.0.94 metric 100 
    10.10.0.0/24 dev wg0 proto kernel scope link src 10.10.0.1 
    172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
    ...
    ```

## Systemd service
Assuming the above works we can now enable the `wg0` interface on boot.

```bash
sudo systemctl daemon-reload
sudo systemctl enable wg-quick@wg0.service
```

You can then manually start `sudo service wg-quick@wg0 start` and check status `service wg-quick@wg0 status` of the service. -->

# Kubernetes (k3s)
Setup ufw to allow access to kubernetes from localhost and the IP_CIDR of the cluster

```bash
# Specify the CIDR we're going to use for our kubernetes pods
export CIDR=10.244.0.0/16

# Needs 6443 (TCP) and 8472 (UDP)
# see: https://rancher.com/docs/k3s/latest/en/installation/installation-requirements/#networking
# see: https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-14-04
# see: https://askubuntu.com/a/488647
sudo tee /etc/ufw/applications.d/k3s <<EOF
# file: /etc/ufw/applications.d/openssh-server
# Used to configure the ufw definition for Lightweight Kubernetes (k3s)

[k3s]
title=Lightweight Kubernetes (k3s)
description=K3s is a highly available, certified Kubernetes distribution designed for production workloads in unattended, resource-constrained, remote locations or inside IoT appliances.
ports=443,6443/tcp|8472/udp
EOF

sudo ufw app update k3s
sudo ufw allow from $CIDR to any app k3s
sudo ufw allow in on wg0 to any app k3s
sudo ufw allow in on cni0 to any app k3s
sudo ufw allow in on kube-bridge to any app k3s
```

Install lightweight kubernetes (k3s) and label the node. We'll disable the use of traefik as, instead, we'll use nginx as the ingress and also the inbuilt loadbalancer as we'lkl use metallb.

In addition we'll make use of the *experimental* support for running [HA with an embedded database](https://rancher.com/docs/k3s/latest/en/installation/ha-embedded/).

```bash
# export K3S_TOKEN=$(head -c48 /dev/urandom | base64); \
export K3S_CLUSTER_SECRET=$(head -c48 /dev/urandom | base64); \
export K3S_KUBECONFIG_MODE="644"; \
export INSTALL_K3S_EXEC="--no-deploy traefik --no-deploy servicelb --cluster-cidr $CIDR"; \
curl -sfL https://get.k3s.io | sh -s - server --cluster-init --no-flannel

kubectl label node $(hostnamectl --static) \
    topology.kubernetes.io/region=home \
    topology.kubernetes.io/zone=mancave \
    topology.rook.io/rack=dellboy
```

After launching the first server, join the other servers to the cluster using the shared secret.

```bash
# export K3S_TOKEN="SECRET"; \
export K3S_CLUSTER_SECRET="SECRET"; \
export K3S_KUBECONFIG_MODE="644"; \
export K3S_URL="https://10.10.0.1:6443"; \
curl -sfL https://get.k3s.io | sh -s - server ${K3S_URL} --no-flannel
```

```bash
# k3s writes config to /etc/rancher/k3s/k3s.yaml
# by convention this should be available to our user at ~/.kube/config
kubectl config view --raw >~/.kube/config
```

# Install Kilo
Kilo automates the creation of a wireguard mesh between nodes in our cluster.

curl -LO https://raw.githubusercontent.com/squat/kilo/master/manifests/kilo-k3s.yaml

Now edit this and create a `full` mesh (so even nodes in the same region arte routed over wireguard).

```+diff
       - name: kilo
         image: squat/kilo
         args:
         - --kubeconfig=/etc/kubernetes/kubeconfig
         - --hostname=$(NODE_NAME)
+        - --mesh-granularity=full
         env:
         - name: NODE_NAME
```

Now apply it

```bash
kubectl apply -f kilo-k3s.yaml
```

# LoadBalancer
As we deployed k3s without a native LoadBalancer we need to quickly configure MetalLB.

```bash
export METALLB_VERSION=v0.9.3; \
kubectl apply -f https://raw.githubusercontent.com/google/metallb/${METALLB_VERSION}/manifests/namespace.yaml; \
kubectl apply -f https://raw.githubusercontent.com/google/metallb/${METALLB_VERSION}/manifests/metallb.yaml; \
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

Now configure it to hand out certain IP ranges.
```bash
tee metallb.configmap.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.0.141-192.168.0.200
    - name: core
      protocol: layer2
      addresses:
      - 192.168.0.121-192.168.0.140
      auto-assign: false
EOF
kubectl apply -f metallb.configmap.yaml
```

# Ingress
We now have kubernetes installed with a non-root user configured to control it with `kubectl`. Let's install `helm` and use it to setup a `nginx` ingress. First though we need to deal with a known [issue]() where helm isn't aware of our k3s installation (because it doesn't write to a default `~/.kube/config` file).

```bash
# k3s writes config to /etc/rancher/k3s/k3s.yaml
# by convention this should be available to our user at ~/.kube/config
kubectl config view --raw >~/.kube/config
```

```bash
# install, add stable repo and then nginx/ingress chart as a NodePort (we've only got one IP)
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash; \
helm repo add stable https://kubernetes-charts.storage.googleapis.com/; \
helm install my-ingress stable/nginx-ingress \
    --set controller.kind=DaemonSet \
    --set controller.service.type=NodePort \
    --set controller.hostNetwork=true
```

# LetsEncrypt
We'll use `cert-manager` to administer our certificates from LetsEncrypt and, for the sake of ease, work with the http challenge (which assumes your fqdn is reachable by their servers).

```bash
export CERT_MANAGER_VERSION=v0.14.1; \
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/${CERT_MANAGER_VERSION}/cert-manager.yaml
```

Firstly we'll use the LE staging server for testing purposes. Setup a cluster wide issuer.

```bash
export CF_API_KEY=$(echo -n myCFtoken43278y | base64)
tee cf.secret.yaml <<EOF
# Secret used to hold the cloudflare API token
# * Permissions:
#   * Zone - DNS - Edit
#   * Zone - Zone - Read
# * Zone Resources:
#   * Include - All Zones *(just selecting the domain in question will result in an error later on)*
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager
type: Opaque
stringData:
  api-token: $CF_API_KEY
EOF
kubectl apply -f cf.secret.yaml
```

```bash
tee certmgr.staging.yaml <<EOF
# Cluster Issuer which uses the DNS challenge for a set domain and falls back to HTTP
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt
  namespace: default
spec:
  acme:
    # The ACME server URL and email address for ACME registration
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: myrealemailfromcloudflareaccount@gmail.com
    # Name of the secret to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-key
    solvers:
    # Enable DNS01 validation for zone
    - dns01:
        cloudflare:
          email: myrealemailfromcloudflareaccount@gmail.com
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token
      selector:
        dnsZones:
        - 'domain.com'
    # Enable HTTP01 validations as fallback
    - http01:
       ingress:
        class: nginx
EOF
kubectl apply -f certmgr.staging.yaml
```
