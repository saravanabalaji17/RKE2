Now I understand. You want **only 3 VMs total**, and Keepalived must run on those same 3 nodes. No separate load balancer VMs.

This is possible using **RKE2 + Rancher Manager + Keepalived VIP**.

We can avoid HAProxy by using a floating VIP directly across the three RKE2 nodes, provided the RKE2 ingress controller listens on the node IPs.

## **1. Final architecture – 3 VMs only**

[RKE2 using Kube-VIP for high-availability deployment on-prem](https://images.openai.com/static-rsc-4/oH0o8yy-W2MBRzAnTuGdy1GaEW5nIstykzHT5TSo8H17H6IHausSh6AszqsNXdbE0JutWwNGvlIo56QKXqmnZ6U0AsLWicURpTowZwE9PY7sNae2lz5pYJ1S7XTd1qaGs0_y6En40JMSvO96Bv4OO2xZB27LDVNUEnw5Ss3Rv6g?purpose=inline)

[linkedin.com](https://www.linkedin.com/pulse/rke2-using-kube-vip-high-availability-deployment-on-prem-kiran-reddy-4q8lc?utm_source=chatgpt.com)

| Node      | IP address    | Components                         |
| --------- | ------------- | ---------------------------------- |
| rancher01 | 192.168.44.21 | RKE2 + etcd + Rancher + Keepalived |
| rancher02 | 192.168.44.22 | RKE2 + etcd + Rancher + Keepalived |
| rancher03 | 192.168.44.23 | RKE2 + etcd + Rancher + Keepalived |

One VIP across 3 nodes

```
Rancher URL:
https://rancher.example.com

DNS:
rancher.example.com -> 192.168.44.100

VIP:
192.168.44.100
```

Keepalived election:

```
Normal:
rancher01 = MASTER (VIP owner)
rancher02 = BACKUP
rancher03 = BACKUP

If rancher01 fails:
rancher02 = MASTER (VIP owner)
rancher03 = BACKUP

If rancher02 also fails:
rancher03 = MASTER (VIP owner)
```

RKE2 embedded etcd provides control-plane HA, while Keepalived provides the floating IP. Three server nodes are sufficient for an HA RKE2 cluster.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.rke2.io\&sz=32)

RKE2+1



## **2. Prerequisites**

All three nodes should be Ubuntu 24.04 or another supported Linux OS.

| Requirement | Each node        |
| ----------- | ---------------- |
| vCPU        | 4                |
| RAM         | 16 GB            |
| Disk        | 100 GB           |
| Network     | Same VLAN/subnet |

You need:

- A static IP for each node.
- One unused VIP.
- DNS record pointing to the VIP.
- Layer-2 network support for VRRP and floating IP movement.

## **3. Install RKE2 on three nodes**

First, configure the first node.

**rancher01 – 192.168.44.21**

```
sudo mkdir -p /etc/rancher/rke2
sudo vi /etc/rancher/rke2/config.yaml
```

```
token: "Rancher-HA-Secret-ChangeMe"

tls-san:
  - rancher.example.com
  - rke2.example.com
  - 192.168.44.100
```

Install:

```
curl -sfL https://get.rke2.io | sudo sh -

sudo systemctl enable --now rke2-server
```

Monitor:

```
sudo journalctl -u rke2-server -f
```

**rancher02 – 192.168.44.22**

Create:

```
sudo mkdir -p /etc/rancher/rke2
sudo vi /etc/rancher/rke2/config.yaml
```

```
server: https://rke2.example.com:9345

token: "Rancher-HA-Secret-ChangeMe"

tls-san:
  - rancher.example.com
  - rke2.example.com
  - 192.168.44.100
```

Install and start:

```
curl -sfL https://get.rke2.io | sudo sh -

sudo systemctl enable --now rke2-server
```

**rancher03 – 192.168.44.23**

Use the same configuration as rancher02.

```
curl -sfL https://get.rke2.io | sudo sh -

sudo systemctl enable --now rke2-server
```

Use the same tested RKE2 version on all three nodes. The VIP must be available before joining nodes 2 and 3.

Reference: [RKE2 HA installation](https://docs.rke2.io/install/ha?utm_source=chatgpt.com).

## **4. Install Keepalived on all three nodes**

Install:

```
sudo apt update

sudo apt install -y keepalived
```

Find the network interface:

```
ip -br addr
```

Assume it is `ens33`.

### **rancher01 – Priority 150**

```
sudo vi /etc/keepalived/keepalived.conf
```

```
global_defs {
    router_id RANCHER01
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 150
    advert_int 1

    unicast_src_ip 192.168.44.21

    unicast_peer {
        192.168.44.22
        192.168.44.23
    }

    authentication {
        auth_type PASS
        auth_pass Rancher1
    }

    virtual_ipaddress {
        192.168.44.100/24 dev ens33
    }
}
```

### **rancher02 – Priority 140**

```
global_defs {
    router_id RANCHER02
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 140
    advert_int 1

    unicast_src_ip 192.168.44.22

    unicast_peer {
        192.168.44.21
        192.168.44.23
    }

    authentication {
        auth_type PASS
        auth_pass Rancher1
    }

    virtual_ipaddress {
        192.168.44.100/24 dev ens33
    }
}
```

### **rancher03 – Priority 130**

```
global_defs {
    router_id RANCHER03
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 130
    advert_int 1

    unicast_src_ip 192.168.44.23

    unicast_peer {
        192.168.44.21
        192.168.44.22
    }

    authentication {
        auth_type PASS
        auth_pass Rancher1
    }

    virtual_ipaddress {
        192.168.44.100/24 dev ens33
    }
}
```

Start Keepalived on all three:

```
sudo systemctl enable --now keepalived

sudo systemctl status keepalived
```

Check VIP:

```
ip addr show ens33
```

Expected: rancher01 owns the VIP.

This basic configuration demonstrates VIP election. For production, add Keepalived health checks for the RKE2 API and ingress endpoints, and validate VRRP firewall rules. Use stronger network security rather than relying on VRRP password authentication.

## **5. Configure DNS**

Create:

```
rancher.example.com  A  192.168.44.100
rke2.example.com     A  192.168.44.100
```

Test:

```
nslookup rancher.example.com
```

## **6. Verify RKE2 cluster**

On rancher01:

```
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml

kubectl get nodes -o wide
```

Expected:

```
NAME       STATUS   ROLES
rancher01  Ready    control-plane,etcd,master
rancher02  Ready    control-plane,etcd,master
rancher03  Ready    control-plane,etcd,master
```

Check ingress:

```
kubectl get pods -n kube-system -o wide
```

Check the ingress controller is listening on the appropriate node ports for HTTP and HTTPS. Current RKE2 v1.36 uses Traefik by default; ingress exposure must be verified before relying on the VIP for Rancher traffic.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.rke2.io\&sz=32)

RKE2+1



## **7. Install Rancher Manager**

On rancher01:

Install Helm:

```
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Add Rancher repository:

```
helm repo add rancher-stable \
https://releases.rancher.com/server-charts/stable

helm repo update
```

Install cert-manager using its official Helm repository and a version compatible with your Rancher release:

```
helm repo add jetstack https://charts.jetstack.io

helm repo update

kubectl create namespace cert-manager

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --set crds.enabled=true
```

Verify:

```
kubectl get pods -n cert-manager
```

Install Rancher:

```
kubectl create namespace cattle-system
```

```
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.example.com \
  --set replicas=3 \
  --set bootstrapPassword='ChangeThisStrongPassword' \
  --set ingress.tls.source=rancher
```

Use a Rancher chart version explicitly selected from the compatibility matrix for your RKE2/Kubernetes version.

## **8. Validate Rancher HA**

```
kubectl get pods -n cattle-system -o wide
```

Expected:

```
NAME                        READY
rancher-xxxxxxxxxx-xxxxx    1/1
rancher-xxxxxxxxxx-yyyyy    1/1
rancher-xxxxxxxxxx-zzzzz    1/1
```

Check replicas:

```
kubectl get deployment rancher -n cattle-system
```

Expected:

```
NAME      READY
rancher   3/3
```

Open:

```
https://rancher.example.com
```

## **9. Test Keepalived failover**

On rancher01:

```
sudo systemctl stop keepalived
```

On rancher02:

```
ip addr show ens33
```

Expected:

```
VIP 192.168.44.100
```

Now rancher02 owns the VIP.

Test Rancher:

```
curl -k -I https://rancher.example.com
```

Restart Keepalived:

```
sudo systemctl start keepalived
```

With default preemption, rancher01 should regain the VIP when its higher priority becomes available.

## **10. Final architecture**

**Users / Browser**

`https://rancher.example.com`

**Keepalived VIP**

`192.168.44.100`

MASTER election across three nodes

Priority 150

**rancher01**

RKE2 + etcd

Keepalived

Rancher replica

Priority 140

**rancher02**

RKE2 + etcd

Keepalived

Rancher replica

Priority 130

**rancher03**

RKE2 + etcd

Keepalived

Rancher replica

**One important distinction:** Keepalived provides active/passive VIP failover, not round-robin load balancing. Rancher replicas and Kubernetes ingress provide application-level distribution. If the VIP owner fails, another node takes over.

Official references:

- [RKE2 HA](https://docs.rke2.io/install/ha?utm_source=chatgpt.com)
- [Rancher HA installation](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade?utm_source=chatgpt.com)
- [RKE2 ingress controllers](https://docs.rke2.io/networking/networking_services?_highlight=ingress\&utm_source=chatgpt.com)

**One detail to confirm before you implement this:** Are you planning to use Ubuntu 24.04 on all three nodes? I can then provide the complete copy-paste installation procedure with Keepalived health checks and Rancher installation.






















Version compatibility confirmed

Your selected versions are compatible:

| Component       | Required version          |
| --------------- | ------------------------- |
| RKE2            | `v1.35.7+rke2r1`          |
| Kubernetes      | `v1.35.7`                 |
| Rancher Manager | `v2.15.1`                 |
| Keepalived      | Ubuntu repository version |
| Helm            | Helm 3                    |

Rancher 2.15.1 explicitly includes RKE2 Kubernetes 1.35.7 in its supported versions.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://documentation.suse.com\&sz=32)

SUSE® Rancher Manager v2.15+1



## **1. Three-node configuration**

We will continue with this design:

| Hostname  | IP             | Keepalived priority |
| --------- | -------------- | ------------------- |
| rancher01 | 192.168.44.21  | 150                 |
| rancher02 | 192.168.44.22  | 140                 |
| rancher03 | 192.168.44.23  | 130                 |
| VIP       | 192.168.44.100 | Floating            |

DNS:

```
rancher.example.com -> 192.168.44.100
rke2.example.com    -> 192.168.44.100
```

## **2. Install exact RKE2 version**

Run on all three nodes, after preparing the appropriate RKE2 configuration files:

```
curl -sfL https://get.rke2.io | \
sudo env INSTALL_RKE2_VERSION="v1.35.7+rke2r1" sh -
```

### **Node 1: rancher01**

`/etc/rancher/rke2/config.yaml`

```
token: "Rancher-HA-Secret-ChangeMe"

tls-san:
  - rancher.example.com
  - rke2.example.com
  - 192.168.44.100
```

Start:

```
sudo systemctl enable --now rke2-server
```

### **Node 2: rancher02**

```
server: https://rke2.example.com:9345

token: "Rancher-HA-Secret-ChangeMe"

tls-san:
  - rancher.example.com
  - rke2.example.com
  - 192.168.44.100
```

Start:

```
sudo systemctl enable --now rke2-server
```

### **Node 3: rancher03**

Use the same configuration as node 2.

```
sudo systemctl enable --now rke2-server
```

All three nodes must have the exact same RKE2 version.

## **3. Verify RKE2**

On rancher01:

```
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml

kubectl get nodes -o wide
```

Check:

```
kubectl get pods -A
```

## **4. Install Rancher Manager 2.15.1**

Add repository:

```
helm repo add rancher-stable \
https://releases.rancher.com/server-charts/stable

helm repo update
```

Verify chart availability:

```
helm search repo rancher-stable/rancher --versions | grep 2.15.1
```

Create namespace:

```
kubectl create namespace cattle-system
```

Install Rancher:

```
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --version 2.15.1 \
  --set hostname=rancher.example.com \
  --set replicas=3 \
  --set bootstrapPassword='ChangeThisStrongPassword' \
  --set ingress.tls.source=rancher
```

If you are using the Rancher Prime distribution, use the authenticated `rancher-prime` repository instead; its chart name is also `rancher`.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://rancher.github.io\&sz=32)

SUSE® Rancher Manager v2.15+1



## **5. Verify Rancher**

```
kubectl get pods -n cattle-system -o wide
```

```
kubectl get deployment rancher -n cattle-system
```

Expected:

```
NAME      READY   UP-TO-DATE   AVAILABLE
rancher   3/3     3            3
```

Access:

```
https://rancher.example.com
```

**Important:** Keepalived must monitor RKE2/API and ingress health, not just the node. Otherwise, a failed service on the VIP owner could leave the VIP there. Also configure Rancher replica spreading across the three nodes.

Official references:

- [RKE2 1.35.7 release notes](https://documentation.suse.com/cloudnative/rke2/latest/en/release-notes/v1.35.X.html?utm_source=chatgpt.com)
- [Rancher 2.15.1 compatibility matrix](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-15-1/?utm_source=chatgpt.com)
- [Rancher 2.15 installation guide](https://documentation.suse.com/cloudnative/rancher-manager/latest/en/installation-and-upgrade/install-rancher.html?utm_source=chatgpt.com)
