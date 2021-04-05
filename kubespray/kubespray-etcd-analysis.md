# Kubespray Etcd Life-Cycle analysis

<!-- TOC -->

- [Kubespray Etcd Life-Cycle analysis](#kubespray-etcd-life-cycle-analysis)
  - [Document Context](#document-context)
  - [Etcd Life-Cycles](#etcd-life-cycles)
  - [Etcd Install Workflow](#etcd-install-workflow)
    - [Download etcd binary](#download-etcd-binary)
    - [Setup etcd cluster](#setup-etcd-cluster)
    - [Ensure etcd CA certificate & private key files and etcd client certificate files exist **on first node - master node**](#ensure-etcd-ca-certificate--private-key-files-and-etcd-client-certificate-files-exist-on-first-node---master-node)
    - [Copy ca file and client cert file to other etcd nodes](#copy-ca-file-and-client-cert-file-to-other-etcd-nodes)
    - [Setup and start etcd service in each etcd nodes](#setup-and-start-etcd-service-in-each-etcd-nodes)
      - [Copy etcd file from download dir to service dir:](#copy-etcd-file-from-download-dir-to-service-dir)
      - [Create config files: `etcd.env`, `/etc/systemd/system/etcd.service`](#create-config-files-etcdenv-etcsystemdsystemetcdservice)
      - [Start service](#start-service)
      - [Join to etcd cluster (member node workflow)](#join-to-etcd-cluster-member-node-workflow)
  - [Etcd Add Node Workflow](#etcd-add-node-workflow)
  - [Etcd Replace Node Workflow](#etcd-replace-node-workflow)
  - [Recover Partition Network Failure Etcd Cluster](#recover-partition-network-failure-etcd-cluster)
  - [Etcd Disaster Recovery Workflow](#etcd-disaster-recovery-workflow)

<!-- /TOC -->

## Document Context

Get from [Kubespray Bare Metal Experiment](./kubespray-bare-metal.md) document. Some hightlight:

- Kubespray version: `v2.15.0`
- Etcd version: `etcd_version: v3.4.13`
- Etcd deploy type: `etcd_deployment_type: host`

## Etcd Life-Cycles

- Install
- Add Node
- Remove Node
- Recover Partition Network Failure
- Disaster Recovery

## Etcd Install Workflow

Install cycle occur when run `cluster.yml` playbook. Steps

### Download etcd binary 

Download etcd binary from URL and extract it:

```yaml
  etcd:
    container: "{{ etcd_deployment_type != 'host' }}"
    file: "{{ etcd_deployment_type == 'host' }}"
    enabled: true
    version: "{{ etcd_version }}"
    dest: "{{ local_release_dir }}/etcd-{{ etcd_version }}-linux-{{ image_arch }}.tar.gz"
    repo: "{{ etcd_image_repo }}"
    tag: "{{ etcd_image_tag }}"
    sha256: >-
      {{ etcd_binary_checksum if (etcd_deployment_type == 'host')
      else etcd_digest_checksum|d(None) }}
    url: "{{ etcd_download_url }}"
    unarchive: "{{ etcd_deployment_type == 'host' }}"
    owner: "root"
    mode: "0755"
    groups:
    - etcd
```

Result

```
[root@etcd01 ~]# ls -al /tmp/releases/
total 16972
drwxr-xr-x.  4 root root            91 Mar 30 10:44 .
drwxrwxrwt. 13 root root          4096 Apr  5 09:59 ..
drwxr-xr-x.  3 root 600260513      123 Aug 24  2020 etcd-v3.4.13-linux-amd64
-rwxr-xr-x.  1 root root      17373136 Mar 30 10:44 etcd-v3.4.13-linux-amd64.tar.gz
drwxr-xr-x.  2 root root             6 Mar 30 10:11 images

[root@etcd01 ~]# ls -al /tmp/releases/etcd-v3.4.13-linux-amd64
total 40568
drwxr-xr-x.  3 root 600260513      123 Aug 24  2020 .
drwxr-xr-x.  4 root root            91 Mar 30 10:44 ..
drwxr-xr-x. 14 root 600260513     4096 Aug 24  2020 Documentation
-rwxr-xr-x.  1 root 600260513 23847904 Aug 24  2020 etcd
-rwxr-xr-x.  1 root 600260513 17620576 Aug 24  2020 etcdctl
-rwxr-xr-x.  1 root 600260513    43094 Aug 24  2020 README-etcdctl.md
-rwxr-xr-x.  1 root 600260513     8431 Aug 24  2020 README.md
-rwxr-xr-x.  1 root 600260513     7855 Aug 24  2020 READMEv2-etcdctl.md
```

### Setup etcd cluster


```yaml
- hosts: etcd
  gather_facts: False
  any_errors_fatal: "{{ any_errors_fatal | default(true) }}"
  environment: "{{ proxy_disable_env }}"
  roles:
    - { role: kubespray-defaults }
    - role: etcd
      tags: etcd
      vars:
        etcd_cluster_setup: true
        etcd_events_cluster_setup: "{{ etcd_events_cluster_enabled }}"
      when: not etcd_kubeadm_enabled| default(false)
```

### Ensure etcd CA certificate & private key files and etcd client certificate files exist **on first node - master node**


```yaml
- include_tasks: "gen_certs_script.yml"
  when:
    - cert_management |d('script') == "script"
  tags:
    - etcd-secrets

- include_tasks: upd_ca_trust.yml
  tags:
    - etcd-secrets

- name: "Gen_certs | Get etcd certificate serials"
  command: "openssl x509 -in {{ etcd_cert_dir }}/node-{{ inventory_hostname }}.pem -noout -serial"
  register: "etcd_client_cert_serial_result"
  changed_when: false
  when:
    - inventory_hostname in groups['k8s-cluster']|union(groups['calico-rr']|default([]))|unique|sort
  tags:
    - master
    - network

```

Output:

```log
TASK [Check_certs | Set 'gen_certs' to true if expected certificates are not on the first etcd node] ******************************************************************************************************************************************
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/ca.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/admin-etcd01.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/admin-etcd01-key.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/member-etcd01.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/member-etcd01-key.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/admin-etcd02.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/admin-etcd02-key.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/member-etcd02.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/member-etcd02-key.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/admin-etcd03.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/admin-etcd03-key.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/member-etcd03.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/member-etcd03-key.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/node-controller01.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/node-controller01-key.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/node-controller02.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/node-controller02-key.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/node-controller03.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/node-controller03-key.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/node-worker01.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/node-worker01-key.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/node-worker02.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/node-worker02-key.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/node-worker03.pem)
ok: [etcd01] => (item=/etc/ssl/etcd/ssl/node-worker03-key.pem)
Monday 05 April 2021  11:32:29 -0400 (0:00:00.502)       0:03:31.923 ********** 
```

### Copy ca file and client cert file to other etcd nodes

```yaml

- name: Gen_certs | Gather etcd member and admin certs from first etcd node
  slurp:
    src: "{{ item }}"
  register: etcd_master_certs
  with_items:
    - "{{ etcd_cert_dir }}/ca.pem"
    - "{{ etcd_cert_dir }}/ca-key.pem"
    - "[{% for node in groups['etcd'] %}
        '{{ etcd_cert_dir }}/admin-{{ node }}.pem',
        '{{ etcd_cert_dir }}/admin-{{ node }}-key.pem',
        '{{ etcd_cert_dir }}/member-{{ node }}.pem',
        '{{ etcd_cert_dir }}/member-{{ node }}-key.pem',
        {% endfor %}]"
  delegate_to: "{{ groups['etcd'][0] }}"
  when:
    - inventory_hostname in groups['etcd']
    - sync_certs|default(false)
    - inventory_hostname != groups['etcd'][0]
  notify: set etcd_secret_changed

- name: Gen_certs | Write etcd member and admin certs to other etcd nodes
  copy:
    dest: "{{ item.item }}"
    content: "{{ item.content | b64decode }}"
    group: "{{ etcd_cert_group }}"
    owner: kube
    mode: 0640
  with_items: "{{ etcd_master_certs.results }}"
  when:
    - inventory_hostname in groups['etcd']
    - sync_certs|default(false)
    - inventory_hostname != groups['etcd'][0]
  loop_control:
    label: "{{ item.item }}"

- name: Gen_certs | Gather node certs from first etcd node
  slurp:
    src: "{{ item }}"
  register: etcd_master_node_certs
  with_items:
    - "[{% for node in (groups['k8s-cluster'] + groups['calico-rr']|default([]))|unique %}
        '{{ etcd_cert_dir }}/node-{{ node }}.pem',
        '{{ etcd_cert_dir }}/node-{{ node }}-key.pem',
        {% endfor %}]"
  delegate_to: "{{ groups['etcd'][0] }}"
  when:
    - inventory_hostname in groups['etcd']
    - inventory_hostname != groups['etcd'][0]
  notify: set etcd_secret_changed

- name: Gen_certs | Write node certs to other etcd nodes
  copy:
    dest: "{{ item.item }}"
    content: "{{ item.content | b64decode }}"
    group: "{{ etcd_cert_group }}"
    owner: kube
    mode: 0640
  with_items: "{{ etcd_master_node_certs.results }}"
  when:
    - inventory_hostname in groups['etcd']
    - inventory_hostname != groups['etcd'][0]
  loop_control:
    label: "{{ item.item }}"
```

Output

```log

TASK [Gen_certs | Gather etcd member and admin certs from first etcd node] ********************************************************************************************************************************************************************
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/ca.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/ca.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/ca-key.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/ca-key.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/admin-etcd01.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/admin-etcd01.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/admin-etcd01-key.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/admin-etcd01-key.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/member-etcd01.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/member-etcd01.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/member-etcd01-key.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/member-etcd01-key.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/admin-etcd02.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/admin-etcd02.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/admin-etcd02-key.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/admin-etcd02-key.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/member-etcd02.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/member-etcd02.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/member-etcd02-key.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/member-etcd02-key.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/admin-etcd03.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/admin-etcd03.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/admin-etcd03-key.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/admin-etcd03-key.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/member-etcd03.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/member-etcd03.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/member-etcd03-key.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/member-etcd03-key.pem)
Monday 05 April 2021  11:32:37 -0400 (0:00:04.200)       0:03:40.342 ********** 

TASK [Gen_certs | Write etcd member and admin certs to other etcd nodes] **********************************************************************************************************************************************************************
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/ca.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/ca.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/ca-key.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/ca-key.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/admin-etcd01.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/admin-etcd01.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/admin-etcd01-key.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/admin-etcd01-key.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/member-etcd01.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/member-etcd01.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/member-etcd01-key.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/member-etcd01-key.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/admin-etcd02.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/admin-etcd02.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/admin-etcd02-key.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/admin-etcd02-key.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/member-etcd02.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/member-etcd02.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/member-etcd02-key.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/member-etcd02-key.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/admin-etcd03.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/admin-etcd03.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/admin-etcd03-key.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/admin-etcd03-key.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/member-etcd03.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/member-etcd03.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/member-etcd03-key.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/member-etcd03-key.pem)
Monday 05 April 2021  11:32:45 -0400 (0:00:07.911)       0:03:48.253 ********** 

TASK [Gen_certs | Gather node certs from first etcd node] *************************************************************************************************************************************************************************************
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-controller01.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-controller01.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-controller01-key.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-controller01-key.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-controller02.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-controller02.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-controller02-key.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-controller02-key.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-controller03.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-controller03.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-controller03-key.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-controller03-key.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-worker01.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-worker01.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-worker01-key.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-worker01-key.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-worker02.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-worker02.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-worker02-key.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-worker02-key.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-worker03.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-worker03.pem)
ok: [etcd02 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-worker03-key.pem)
ok: [etcd03 -> 192.168.122.31] => (item=/etc/ssl/etcd/ssl/node-worker03-key.pem)
Monday 05 April 2021  11:32:49 -0400 (0:00:03.655)       0:03:51.909 ********** 

TASK [Gen_certs | Write node certs to other etcd nodes] ***************************************************************************************************************************************************************************************
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/node-controller01.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/node-controller01.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/node-controller01-key.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/node-controller01-key.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/node-controller02.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/node-controller02.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/node-controller02-key.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/node-controller02-key.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/node-controller03.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/node-controller03.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/node-controller03-key.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/node-controller03-key.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/node-worker01.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/node-worker01.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/node-worker01-key.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/node-worker01-key.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/node-worker02.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/node-worker02.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/node-worker02-key.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/node-worker02-key.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/node-worker03.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/node-worker03.pem)
changed: [etcd03] => (item=/etc/ssl/etcd/ssl/node-worker03-key.pem)
changed: [etcd02] => (item=/etc/ssl/etcd/ssl/node-worker03-key.pem)
Monday 05 April 2021  11:32:55 -0400 (0:00:06.883)       0:03:58.793 ********** 

```

### Setup and start etcd service in each etcd nodes

```yaml

- include_tasks: "install_{{ etcd_deployment_type }}.yml"
  when: is_etcd_master
  tags:
    - upgrade

- include_tasks: configure.yml
  when: is_etcd_master

- include_tasks: refresh_config.yml
  when: is_etcd_master

- name: Restart etcd if certs changed
  service:
    name: etcd
    state: restarted
    enabled: yes
  when: is_etcd_master and etcd_cluster_setup and etcd_secret_changed|default(false)

- name: Restart etcd-events if certs changed
  service:
    name: etcd-events
    state: restarted
    enabled: yes
  when: is_etcd_master and etcd_events_cluster_setup and etcd_secret_changed|default(false)

# After etcd cluster is assembled, make sure that
# initial state of the cluster is in `existing`
# state instead of `new`.
- include_tasks: refresh_config.yml
  when: is_etcd_master
```

#### Copy etcd file from download dir to service dir:

```yaml
---
- name: install | Download etcd and etcdctl
  include_tasks: "../../download/tasks/download_file.yml"
  vars:
    download: "{{ download_defaults | combine(downloads.etcd) }}"
  when: etcd_cluster_setup
  tags:
    - never
    - etcd

- name: install | Copy etcd and etcdctl binary from download dir
  copy:
    src: "{{ local_release_dir }}/etcd-{{ etcd_version }}-linux-{{ host_architecture }}/{{ item }}"
    dest: "{{ bin_dir }}/{{ item }}"
    mode: 0755
    remote_src: yes
  with_items:
    - etcd
    - etcdctl
  when: etcd_cluster_setup

```

Output

```log
included: /home/cloud/kubespray/roles/etcd/tasks/install_host.yml for etcd01, etcd03, etcd02
Monday 05 April 2021  11:32:57 -0400 (0:00:00.086)       0:04:00.148 ********** 

TASK [install | Copy etcd and etcdctl binary from download dir] *******************************************************************************************************************************************************************************
ok: [etcd02] => (item=etcd)
ok: [etcd01] => (item=etcd)
ok: [etcd03] => (item=etcd)
ok: [etcd02] => (item=etcdctl)
ok: [etcd01] => (item=etcdctl)
ok: [etcd03] => (item=etcdctl)
Monday 05 April 2021  11:32:58 -0400 (0:00:00.764)       0:04:00.913 ********** 
```

#### Create config files: `etcd.env`, `/etc/systemd/system/etcd.service`

```yaml

- include_tasks: refresh_config.yml
  when: is_etcd_master

- name: Configure | Copy etcd.service systemd file
  template:
    src: "etcd-{{ etcd_deployment_type }}.service.j2"
    dest: /etc/systemd/system/etcd.service
    backup: yes
  when: is_etcd_master and etcd_cluster_setup

- name: Configure | Copy etcd-events.service systemd file
  template:
    src: "etcd-events-{{ etcd_deployment_type }}.service.j2"
    dest: /etc/systemd/system/etcd-events.service
    backup: yes
  when: is_etcd_master and etcd_events_cluster_setup

```

Output

```log
[root@etcd01 cloud]# cat /etc/etcd.env 
# Environment file for etcd v3.4.13
ETCD_DATA_DIR=/var/lib/etcd
ETCD_ADVERTISE_CLIENT_URLS=https://192.168.122.31:2379
ETCD_INITIAL_ADVERTISE_PEER_URLS=https://192.168.122.31:2380
ETCD_INITIAL_CLUSTER_STATE=existing
ETCD_METRICS=basic
ETCD_LISTEN_CLIENT_URLS=https://192.168.122.31:2379,https://127.0.0.1:2379
ETCD_ELECTION_TIMEOUT=5000
ETCD_HEARTBEAT_INTERVAL=250
ETCD_INITIAL_CLUSTER_TOKEN=k8s_etcd
ETCD_LISTEN_PEER_URLS=https://192.168.122.31:2380
ETCD_NAME=etcd1
ETCD_PROXY=off
ETCD_INITIAL_CLUSTER=etcd1=https://192.168.122.31:2380,etcd2=https://192.168.122.32:2380,etcd3=https://192.168.122.33:2380
ETCD_AUTO_COMPACTION_RETENTION=8
ETCD_SNAPSHOT_COUNT=10000
# Flannel need etcd v2 API
ETCD_ENABLE_V2=true

# TLS settings
ETCD_TRUSTED_CA_FILE=/etc/ssl/etcd/ssl/ca.pem
ETCD_CERT_FILE=/etc/ssl/etcd/ssl/member-etcd01.pem
ETCD_KEY_FILE=/etc/ssl/etcd/ssl/member-etcd01-key.pem
ETCD_CLIENT_CERT_AUTH=true

ETCD_PEER_TRUSTED_CA_FILE=/etc/ssl/etcd/ssl/ca.pem
ETCD_PEER_CERT_FILE=/etc/ssl/etcd/ssl/member-etcd01.pem
ETCD_PEER_KEY_FILE=/etc/ssl/etcd/ssl/member-etcd01-key.pem
ETCD_PEER_CLIENT_CERT_AUTH=True




# CLI settings
ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
ETCDCTL_CACERT=/etc/ssl/etcd/ssl/ca.pem
ETCDCTL_KEY=/etc/ssl/etcd/ssl/admin-etcd01-key.pem
ETCDCTL_CERT=/etc/ssl/etcd/ssl/admin-etcd01.pem
```

```log
[root@etcd01 cloud]# cat /etc/systemd/system/etcd.service 
[Unit]
Description=etcd
After=network.target

[Service]
Type=notify
User=root
EnvironmentFile=/etc/etcd.env
ExecStart=/usr/local/bin/etcd
NotifyAccess=all
Restart=always
RestartSec=10s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
```

#### Start service

```yaml

- name: Configure | reload systemd
  systemd:
    daemon_reload: true
  when: is_etcd_master
# when scaling new etcd will fail to start
- name: Configure | Ensure etcd is running
  service:
    name: etcd
    state: started
    enabled: yes
  ignore_errors: "{{ etcd_cluster_is_healthy.rc == 0 }}"
  when: is_etcd_master and etcd_cluster_setup

```


#### Join to etcd cluster (member node workflow)

```yaml
---
- name: Join Member | Add member to etcd cluster  # noqa 301 305
  shell: "{{ bin_dir }}/etcdctl member add {{ etcd_member_name }} --peer-urls={{ etcd_peer_url }}"
  register: member_add_result
  until: member_add_result.rc == 0 or 'Peer URLs already exists' in member_add_result.stderr
  failed_when: member_add_result.rc != 0 and 'Peer URLs already exists' not in member_add_result.stderr
  retries: "{{ etcd_retries }}"
  delay: "{{ retry_stagger | random + 3 }}"
  environment:
    ETCDCTL_API: 3
    ETCDCTL_CERT: "{{ etcd_cert_dir }}/admin-{{ inventory_hostname }}.pem"
    ETCDCTL_KEY: "{{ etcd_cert_dir }}/admin-{{ inventory_hostname }}-key.pem"
    ETCDCTL_CACERT: "{{ etcd_cert_dir }}/ca.pem"
    ETCDCTL_ENDPOINTS: "{{ etcd_access_addresses }}"

- include_tasks: refresh_config.yml
  vars:
    etcd_peer_addresses: >-
      {% for host in groups['etcd'] -%}
        {%- if hostvars[host]['etcd_member_in_cluster'].rc == 0 -%}
          {{ "etcd"+loop.index|string }}=https://{{ hostvars[host].etcd_access_address | default(hostvars[host].ip | default(fallback_ips[host])) }}:2380,
        {%- endif -%}
        {%- if loop.last -%}
          {{ etcd_member_name }}={{ etcd_peer_url }}
        {%- endif -%}
      {%- endfor -%}

- name: Join Member | Ensure member is in etcd cluster
  shell: "set -o pipefail && {{ bin_dir }}/etcdctl member list | grep {{ etcd_access_address }} >/dev/null"
  args:
    executable: /bin/bash
  register: etcd_member_in_cluster
  changed_when: false
  check_mode: no
  tags:
    - facts
  environment:
    ETCDCTL_API: 3
    ETCDCTL_CERT: "{{ etcd_cert_dir }}/admin-{{ inventory_hostname }}.pem"
    ETCDCTL_KEY: "{{ etcd_cert_dir }}/admin-{{ inventory_hostname }}-key.pem"
    ETCDCTL_CACERT: "{{ etcd_cert_dir }}/ca.pem"
    ETCDCTL_ENDPOINTS: "{{ etcd_access_addresses }}"

- name: Configure | Ensure etcd is running
  service:
    name: etcd
    state: started
    enabled: yes
```

## Etcd Add Node Workflow

## Etcd Replace Node Workflow

## Recover Partition Network Failure Etcd Cluster

## Etcd Disaster Recovery Workflow
