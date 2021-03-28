# Kubespray Bare Metal Experiment

## Component Planning

![cluster-planning](./images/k8s-bare-metal-with-kubespray.png)

- 3 Etcd VMs:  2 CPU - 2 GB RAM - 40 GB Storage
- 3 K8S Masters: 2 CPU - 4 GB RAM - 40 GB Storage
- 3 K8S Worker: 2 CPU - 3 GB RAM - 40 GB Storage
- Kube API Server Endpoint IP: 192.168.122.10

## Software specs

- Software LB: HaProxy + Keepalived
- OS: CentOS 7
- Container Runtime: containerd (Worker, Master Nodes)
- Etcd deploy type: Host
- K8S version: v1.20.5 (on time writing this document - 03/2021)

## Setup

### 1. Setup K8S Masters

- Setup IP and hostname for K8S Masters
- Disable firewalld and selinux
  ```bash
  systemctl disable firewalld
  systemctl stop firewalld
  sudo setenforce 0
  sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
  ```

#### Setup HAProxy 

Install HAProxy and Keepalived and psmisc - foll killall command

```bash
yum -y install keepalived haproxy psmisc
```

Configure HAProxy

```bash
# file /etc/haproxy/haproxy.cfg
# ...
# round robin balancing for apiserver
#---------------------------------------------------------------------
#---------------------------------------------------------------------
# apiserver frontend which proxys to the masters
#---------------------------------------------------------------------
frontend apiserver
    bind *:8443
    mode tcp
    option tcplog
    default_backend apiserver
#---------------------------------------------------------------------
# round robin balancing for apiserver
#---------------------------------------------------------------------
backend apiserver
    option httpchk GET /healthz
    http-check expect status 200
    mode tcp
    option ssl-hello-chk
    balance     roundrobin
        server controller01 192.168.122.21:6443 check
        server controller02 192.168.122.22:6443 check
        server controller03 192.168.122.23:6443 check
```

Restart HAProxy

```bash
sudo systemctl restart haproxy
sudo systemctl enable haproxy
```

#### Setup Keepalived

**In controller01**

Configure Keepalived

```bash
# File /etc/keepalived/keepalived.conf
vrrp_script chk_haproxy {
  script "killall -0 haproxy"
  interval 2
  weight 2
}

vrrp_instance VI_1 {
  interface eth0
  state MASTER
  advert_int 1
  virtual_router_id 21
  priority 101
  unicast_src_ip 192.168.122.21    ##Master 1 IP Address
  unicast_peer {
      192.168.122.22               ##Master 2 IP Address
      192.168.122.23               ##Master 2 IP Address
   }
  virtual_ipaddress {
    192.168.122.20                 ##Shared Virtual IP address
  }
  track_script {
    chk_haproxy
  }
}
```

Restart keepalived

```bash
sudo systemctl start keepalived
sudo systemctl enable keepalived
```

**In controller02**

Configure Keepalived

```bash
# File /etc/keepalived/keepalived.conf
vrrp_script chk_haproxy {
  script "killall -0 haproxy"
  interval 2
  weight 2
}

vrrp_instance VI_1 {
  interface eth0
  state MASTER
  advert_int 1
  virtual_router_id 21
  priority 100
  unicast_src_ip 192.168.122.22    ##Master 1 IP Address
  unicast_peer {
      192.168.122.21               ##Master 2 IP Address
      192.168.122.23               ##Master 2 IP Address
   }
  virtual_ipaddress {
    192.168.122.20                 ##Shared Virtual IP address
  }
  track_script {
    chk_haproxy
  }
}
```

Restart keepalived

```bash
sudo systemctl start keepalived
sudo systemctl enable keepalived
```

**In controller03**

Configure Keepalived

```bash
# File /etc/keepalived/keepalived.conf
vrrp_script chk_haproxy {
  script "killall -0 haproxy"
  interval 2
  weight 2
}

vrrp_instance VI_1 {
  interface eth0
  state MASTER
  advert_int 1
  virtual_router_id 21
  priority 99
  unicast_src_ip 192.168.122.23    ##Master 1 IP Address
  unicast_peer {
      192.168.122.22               ##Master 2 IP Address
      192.168.122.21               ##Master 2 IP Address
   }
  virtual_ipaddress {
    192.168.122.20                 ##Shared Virtual IP address
  }
  track_script {
    chk_haproxy
  }
}
```

Restart keepalived

```bash
sudo systemctl start keepalived
sudo systemctl enable keepalived
```

### 2. Setup Etcd Nodes

- Setup IP and hostname for Etcd Nodes
- Disable firewalld and selinux
  ```bash
  systemctl disable firewalld
  systemctl stop firewalld
  sudo setenforce 0
  sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
  ```

### 3. Setup K8S Workers

- Setup IP and hostname for K8S Workers
- Disable firewalld and selinux
  ```bash
  systemctl disable firewalld
  systemctl stop firewalld
  sudo setenforce 0
  sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
  ```

### 4. Setup Bastion 

Clone kubespray repo

```bash
git clone https://github.com/kubernetes-sigs/kubespray
```

Setup virtualenv for python3

```bash
virtualenv venv
```

Install required packages

```bash
source venv/bin/active
pip install -r requirements.txt
```

Prepare Inventory File:

```bash
cp -rfp inventory/sample inventory/cluster-01
```

Update Inventory File

```ini
# file inventory/cluster-01/inventory.ini
# ## Configure 'ip' variable to bind kubernetes services on a
# ## different ip than the default iface
# ## We should set etcd_member_name for etcd cluster. The node that is not a etcd member do not need to set the value, or can set the empty string value.
[all]
controller01   ansible_host=192.168.122.21
controller02   ansible_host=192.168.122.22
controller03   ansible_host=192.168.122.23

worker01   ansible_host=192.168.122.51
worker02   ansible_host=192.168.122.52
worker03   ansible_host=192.168.122.53

etcd01   ansible_host=192.168.122.31
etcd02   ansible_host=192.168.122.32
etcd03   ansible_host=192.168.122.33

# ## configure a bastion host if your nodes are not directly reachable
# [bastion]
# bastion ansible_host=x.x.x.x ansible_user=some_user

[kube_control_plane]
controller01
controller02
controller03

[etcd]
etcd01
etcd02
etcd03

[kube-node]
worker01
worker02
worker03

[calico-rr]

[k8s-cluster:children]
kube_control_plane
kube-node
calico-rr
```

Update kubespray configuration files

```yaml
# file inventory/cluster-01/group_vars/k8s-cluster/k8s-cluster.yml
container_manager: containerd
# file inventory/cluster-01/group_vars/etcd.yml
etcd_deployment_type: host
# fileinventory/cluster-01/group_vars/all/all.yml
loadbalancer_apiserver:
   address: 192.168.122.20
   port: 8443
## Deactivate Internal loadbalancers for apiservers at around line 26
loadbalancer_apiserver_localhost: false
```

### 5. Run install command:

```bash
ansible-playbook -i inventory/cluster-01/inventory.ini --become \
--user=cloud --ask-pass --ask-become-pass --become-user=root cluster.yml
```

## References

- https://github.com/gregbkr/kubernetes-kargo-logging-monitoring
- https://kubernetes.io/docs/setup/production-environment/tools/kubespray/
- https://computingforgeeks.com/deploy-kubernetes-cluster-centos-kubespray/