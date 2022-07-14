---
title: Using kube-vip with RKE2 Api-Server
author: Andy Clemenko and Brian Durden
notes: https://www.markdownguide.org/basic-syntax/
---

# Bootstrapping an HA RKE2 Cluster easily without a LoadBalancer

---

> **Table of Contents**:
> * [Whoami](#whoami)
> * [Prerequisites](#prerequisites)
> * [Manual Provisioning](#ManualProvisioning)
> * [Conclusion](#conclusion)

---

## Whoami

Who are you? Why are you cool?

## Prerequisites

The only pre-req here, other than having access to VMs for this task (I use [Harvester](https://github.com/harvester/harvester) here is either control over DNS entries for each of your nodes or just adding them into an /etc/hosts file on each node. This assumes linux-based VMs, I used Ubuntu 20.04 LTS. This can work in any VM provisioner that supports cloud-init or just lets you SSH in to run a script.

## The Problem
On-prem K8S installations tend to lack cloud-native services like Load Balancing. When building an HA (high availability) cluster, the K8S api-server must be hosted across multiple control-plane/master nodes in the event of a failure. Traditionally for K8S deployments and services, this would require usage of MetalLB, HAProxy, nginx, or some loadbalancer service/daemonset. Unfortunately this case cannot work for the api-server as there would be a circular dependency introduced as these services are deployed via the api-server, they are dynamic deployments/daemonsets/statefulsets.

## The Solution

Enter kube-vip, the solution to this problem as it can be run as a static Pod or static DaemonSet as part of the first RKE2 master. Utilizing code already supported in K3S, we can inject this static DaemonSet and do a quick change/replace of some minimal config items and we are good. It's easy enough to drop into any Ansible playbook or using a more cloud-native approach: cloud-init.

A VIP is short for a Virtual-IP and is common industry vernacular. Essentially a single network interface (such as eth0 for instance) binds to multiple IP addresses instead of just one. If you are using DHCP for VM IP assignment, please ensure there are static IPs available to be claimed as well. For instance, on my workload setup, the first 10 IP addresses are static and my DHCP server only assigns IPs from x.y.z.12 through x.y.z.254.

## Prep
In this guide, I will be setting up a 3-node HA RKE2 cluster. I use the `.lol` domain but swap out for the domain of your choosing. 

| Host       | Type       | IP       | Notes                                                                                                                         |
|------------|------------|----------|-------------------------------------------------------------------------------------------------------------------------------|
| rke2a      | VM         | 10.0.1.2 | etcd                                                                                                                          |
| rke2b      | VM         | 10.0.1.3 | etcd                                                                                                                          |
| rke2c      | VM         | 10.0.1.4 | etcd                                                                                                                          |
| rke2master | VIP        | 10.0.1.5 | You will define this IP on your own. Make sure that it is not currently allocated to a node (and remove from DHCP allocation) |

If you do not have a DNS server available/configured, the `/etc/hosts` file on each node will need to include the following.
```txt
rke2a 10.0.1.2
rke2b 10.0.1.3
rke2c 10.0.1.4
rke2master 10.0.1.5
```

As we progress below, we'll be defining variables and such assuming we wish to construct this into a script and a cloud-init within our VM provisioner.

## Manual Provisioning

### Grab the RKE2 goods
Firstly, we need to grab the RKE2 goods for our VM. This needs to be done on all 3 VMs. That's easy to do using the `get.rke2.io` endpoint:
```console
root@rke2a:~# curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.20 sh -
[INFO]  finding release for channel v1.20
[INFO]  using v1.20.15+rke2r2 as release
[INFO]  downloading checksums at https://github.com/rancher/rke2/releases/download/v1.20.15+rke2r2/sha256sum-amd64.txt
[INFO]  downloading tarball at https://github.com/rancher/rke2/releases/download/v1.20.15+rke2r2/rke2.linux-amd64.tar.gz
[INFO]  verifying tarball
[INFO]  unpacking tarball file to /usr/local
```
###  Grab kube-vip from K3S
Next we need to create a manifests directory for storing our kube-vip static manifests, grab kube-vip's RBAC objects as well as the DaemonSet spec; only on the first VM. This is done using `wget https://kube-vip.io/manifests/rbac.yaml -O /var/lib/rancher/rke2/server/manifests/kube-vip-rbac.yaml` to copy the rbac manifest into the directory we've created. Follow this by retrieving the k3s kube-vip configuration using the `kube-vip.io/k3s` host as part of a curl request. Note our VIP address and VIP interface (the real interface name on our VM we want the VIP bound to) are used here. We are defining them using a bash export call.
```console
root@rke2a:~# export RKE2_VIP_IP=10.0.1.5
root@rke2a:~# export RKE2_VIP_INTERFACE=eth0
root@rke2a:~# mkdir -p /var/lib/rancher/rke2/server/manifests/
root@rke2a:~# wget https://kube-vip.io/manifests/rbac.yaml -O /var/lib/rancher/rke2/server/manifests/kube-vip-rbac.yaml
--2022-07-14 18:57:18--  https://kube-vip.io/manifests/rbac.yaml
Resolving kube-vip.io (kube-vip.io)... 34.75.209.36, 52.5.183.223
Connecting to kube-vip.io (kube-vip.io)|34.75.209.36|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 792 [application/x-yaml]
Saving to: ‘/var/lib/rancher/rke2/server/manifests/kube-vip-rbac.yaml’

/var/lib/rancher/rke2/server/ 100%[=================================================>]     792  --.-KB/s    in 0s      

2022-07-14 18:57:18 (223 MB/s) - ‘/var/lib/rancher/rke2/server/manifests/kube-vip-rbac.yaml’ saved [792/792]

root@rke2a:~# curl -sL kube-vip.io/k3s |  vipAddress=${RKE2_VIP_IP} vipInterface=${RKE2_VIP_INTERFACE} sh | sudo tee /var/lib/rancher/rke2/server/manifests/vip.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  creationTimestamp: null
  name: kube-vip-ds
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: kube-vip-ds
  template:
    metadata:
      creationTimestamp: null
      labels:
        name: kube-vip-ds
    spec:
      containers:
      - args:
        - manager
        env:
        - name: vip_arp
          value: "true"
        - name: vip_interface
          value: eth0
        - name: port
          value: "6443"
        - name: vip_cidr
          value: "32"
        - name: cp_enable
          value: "true"
        - name: cp_namespace
          value: kube-system
        - name: svc_enable
          value: "false"
        - name: vip_address
          value: 10.0.1.5
        image: ghcr.io/kube-vip/kube-vip:v0.3.7
        imagePullPolicy: Always
        name: kube-vip
        resources: {}
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
            - NET_RAW
            - SYS_TIME
      hostNetwork: true
      nodeSelector:
        node-role.kubernetes.io/master: "true"
      serviceAccountName: kube-vip
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
  updateStrategy: {}
status:
  currentNumberScheduled: 0
  desiredNumberScheduled: 0
  numberMisscheduled: 0
  numberReady: 0
```

### Configure RKE2
Next we need to create a basic rke2 configuration file. This is picked up by our later RKE2 service init. We only need to create a `/etc/rancher/rke2` directory and then define a simple yaml file inside. This is only necessary on the first VM.
```console
mkdir -p /etc/rancher/rke2
touch /etc/rancher/rke2/config.yaml
echo "tls-san:" >> /etc/rancher/rke2/config.yaml 
echo "  - ${HOSTNAME}.lol" >> /etc/rancher/rke2/config.yaml
echo "  - ${HOSTNAME}" >> /etc/rancher/rke2/config.yaml
echo "  - rke2master.lol" >> /etc/rancher/rke2/config.yaml
echo "  - rke2master" >> /etc/rancher/rke2/config.yaml
```

Contents of the file we just created:
```yaml
token: my-shared-secret
tls-san:
  - rke2a.lol
  - rke2a
  - rke2master.lol
  - rke2master
```

### Configure /etc/hosts
The last step of prep is to edit the `/etc/hosts` file on all VMs to reflect our table above. Our `/etc/hosts` file for the first VM should look like this on an Ubuntu box. The `rke2master` entry should use the loopback IP (127.0.0.1) on the first VM. Here we are using static-defined IPs without a DNS server and defining `/etc/hosts` on the first VM.
```console
root@rke2a:~# cat /etc/hosts
127.0.0.1 localhost
10.0.1.2 rke2a
10.0.1.3 rke2b 
10.0.1.4 rke2c 
127.0.0.1 rke2master

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
```


## We have LiftOff
Now that our VM is properly configured, let's light this candle! Init and start the RKE2 service:
```console
root@rke2a:~# systemctl enable rke2-server.service
root@rke2a:~# systemctl start rke2-server.service
```
### Verifying
Kubectl is included as part of the RKE2 install, but it is located in a non-default directory. Use it to determine the node state and wait for it to go ready:
```console
/var/lib/rancher/rke2/bin/kubectl \
        --kubeconfig /etc/rancher/rke2/rke2.yaml get nodes
```

Once the node is ready, you can inspect the state of the pods:
```
/var/lib/rancher/rke2/bin/kubectl \
        --kubeconfig /etc/rancher/rke2/rke2.yaml get pods --all-namespaces
```

### Adding Nodes
Once this first node is finished, log into each additional node to perform the joining. Here we're setting `/etc/hosts` on rke2b:
```text
root@rke2b:~# cat /etc/hosts
127.0.0.1 localhost
10.0.1.2 rke2a
10.0.1.3 rke2b 
10.0.1.4 rke2c 
10.0.1.5 rke2master

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
```

Next set the rke2 config with the token and server address of the first node.
```console
export TOKEN="my-shared-secret"

mkdir -p /etc/rancher/rke2
touch /etc/rancher/rke2/config.yaml
echo "token: ${TOKEN}" >> /etc/rancher/rke2/config.yaml
echo "server: https://rke2master.lol:9345" >> /etc/rancher/rke2/config.yaml
echo "tls-san:" >> /etc/rancher/rke2/config.yaml
echo "  - ${HOSTNAME}.lol" >> /etc/rancher/rke2/config.yaml
echo "  - ${HOSTNAME}" >> /etc/rancher/rke2/config.yaml
echo "  - rke2master.lol" >> /etc/rancher/rke2/config.yaml
echo "  - rke2master" >> /etc/rancher/rke2/config.yaml
```

Contents of the file we just created:
```yaml
token: my-shared-secret
server: https://rke2master.lol:9345
tls-san:
  - rke2b.lol
  - rke2b
  - rke2master.lol
  - rke2master
```

Kick the process off on and it should join your master node in an HA configuration.
```console
root@rke2b:~# curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.20 sh -
[INFO]  finding release for channel v1.20
[INFO]  using v1.20.15+rke2r2 as release
[INFO]  downloading checksums at https://github.com/rancher/rke2/releases/download/v1.20.15+rke2r2/sha256sum-amd64.txt
[INFO]  downloading tarball at https://github.com/rancher/rke2/releases/download/v1.20.15+rke2r2/rke2.linux-amd64.tar.gz
[INFO]  verifying tarball
[INFO]  unpacking tarball file to /usr/local

root@rke2b:~# systemctl enable rke2-server.service
root@rke2b:~# systemctl start rke2-server.service
```

## The Solution (Cloud-Init Provisioning)
That's a big list of stuff to do and you can imagine it can become very toilsome to manage at scale. Let's leverage our VM provisioner to automate all this effort and turn it into a point and click affair!

In this example, I'll use Harvester to build a cloud-init that I can use as a template for each node.


## Conclusion

What did they just read.

![success](img/success.jpg)
