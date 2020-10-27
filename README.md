# Kubernetes HA Options

## About
This document gives an overview of the  Kubernetes(K8s) HA implementation options on bare metal. The Cloud providers sell K8s HA as managed solution  out of the box in Stacked etcd topology way:
by regional clustering and zonal. 
- __Zonal cluster topology__ when a cluster's control plane and nodes all run in a single compute zone. 
- __Regional cluster__, when the control plane and nodes are replicated across multiple zones within a single region.

There are two main things that you need to take in to the account  by selecting your HA topology option.
- Application HA and critically  
- Control plane(CP) HA
- Etcd HA 


- HA topology selection 

- build steps

# Possible HA clustering topologies     
- [x] Stacked etcd topology (Bare Metal)
  - [x] Regional clustering (Cloud)
  - [x] Zonal clustering (Cloud)
- [x] External etcd topology (Bare Metal)

![stacked](https://d33wubrfki0l68.cloudfront.net/d1411cded83856552f37911eb4522d9887ca4e83/b94b2/images/kubeadm/kubeadm-ha-topology-stacked-etcd.svg)
Figure 1.1 __Stacked__ - multi master cluster where `etcd` running in same CP node.

![external](https://d33wubrfki0l68.cloudfront.net/ad49fffce42d5a35ae0d0cc1186b97209d86b99c/5a6ae/images/kubeadm/kubeadm-ha-topology-external-etcd.svg)  
Figure 1.2 __External__ - multi master where `etcd` cluster  running outside of CP 

## Pros and Cons

| Topology Pros & Cons            | Mitigation    | Stacked etcd | External etcd |
|:--------------------------------|:--------------|:-------------|:--------------|
| default                         |               |   Yes        |  No           |
| Setup complexity                |               |  simple      |  complex      |
| Hard replication management     |               |  simple      |  hard         |
| Coupled architecture            | more CP nodes | CP fail risk | No CP fail risk  |
| minimum CP nodes per HA N/2+1   |               | 3            |   6           |
| Infrastructure needs            |               | 3            |   x2 = 6      |
| HA nodes workload reduction     |               | No           | Yes           |
| Etcd Security                   |               |              | Higher        |

>The stacked solution looks more optimal in terms of HA, complexity and resource vise. For External etcd solution should be special business case 
{.is-info}

# Prerequisites 

- x3 machines for the control-plane nodes
- x3 machines for worker nodes 
- full network connectivity between all machines
- sudo privileges on all machines 
- ssh access to all machines in the cluster
- `kubeadm` and `kubelet` installed on all above machines
- TCP Load Balancer 
- x3 extra machines for External etcd topology

# Stacked etcd solution deployment
## AWS LAB 

Setup EC2 instances on AWS

```buildoutcfg
ab@ZNATA:~ $ aws ec2 describe-instances --filters "Name=instance-type,Values=t2.medium" --query "Reservations[*].Instances[*].{Server_Name:Tags[0].Value,Instance_ID:InstanceId,Private_IP:PrivateIpAddress,InstanceType:InstanceType}"
----------------------------------------------------------------------
|                          DescribeInstances                         |
+--------------+-----------------------+-------------+---------------+
| InstanceType |      Instance_ID      | Private_IP  |  Server_Name  |
+--------------+-----------------------+-------------+---------------+
|  t2.medium   |  i-0145c21953badcc53  |  10.0.0.72  |  jump_box     |
|  t2.medium   |  i-0bef6d87830298397  |  10.0.1.176 |  master1      |
|  t2.medium   |  i-00379eff66e83cdc2  |  10.0.1.130 |  worker1      |
|  t2.medium   |  i-0fa0940c17b683050  |  10.0.1.35  |  etcd1        |
|  t2.medium   |  i-06bddaab1b66629bb  |  10.0.1.96  |  master2      |
|  t2.medium   |  i-09022d7cd88a63880  |  10.0.1.36  |  worker2      |
|  t2.medium   |  i-06b24a8cc5e3aeca6  |  10.0.1.98  |  etcd2        |
|  t2.medium   |  i-043de3c4991f1ad43  |  10.0.1.27  |  master3      |
|  t2.medium   |  i-0cd30789635c2020a  |  10.0.1.20  |  worker3      |
|  t2.medium   |  i-0160d575ce938af02  |  10.0.1.190 |  etcd3        |
+--------------+-----------------------+-------------+---------------+

```
Lets use x3 master and x3 worker EC2 instances + 1 jump server for Stacked etc deployment  

### Prepare env

to prepare environment use ansible playbook for:

- os prerequisites for docker, NTP, swap, 
- install Docker engine
- move docker root to /apps/docker 
- set client certs for docker registry
- install kubeadm, kublet  
 
<details>
    <summary>Bootstraping env</summary>
     
     ### `Ansible bootstrap x3 master x3 worker nodes`:
    
```buildoutcfg
[ec2-user@jump trailkube]$ ansible-playbook -i hosts.ini site.yml 

PLAY [kube-master] **************************************************************************************************************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************************************************************************************************************
ok: [10.0.1.27]
ok: [10.0.1.176]
ok: [10.0.1.96]

TASK [os : Update and upgrade apt packages] *************************************************************************************************************************************************************************
ok: [10.0.1.27]
ok: [10.0.1.96]
ok: [10.0.1.176]

TASK [os : Add required prerequisite packages] **********************************************************************************************************************************************************************
ok: [10.0.1.27]
ok: [10.0.1.176]
ok: [10.0.1.96]

TASK [os : Download pip2] *******************************************************************************************************************************************************************************************
ok: [10.0.1.96]
ok: [10.0.1.27]
ok: [10.0.1.176]

TASK [os : Run pip2 install] ****************************************************************************************************************************************************************************************
changed: [10.0.1.27]
changed: [10.0.1.96]
changed: [10.0.1.176]

TASK [os : Set NTP service] *****************************************************************************************************************************************************************************************
ok: [10.0.1.27]
ok: [10.0.1.96]
ok: [10.0.1.176]

TASK [os : Set Rsyslog Blacklist] ***********************************************************************************************************************************************************************************
ok: [10.0.1.176] => (item=/home/ec2-user/trailkube/roles/os/templates/01-blocklist.conf.j2)
ok: [10.0.1.96] => (item=/home/ec2-user/trailkube/roles/os/templates/01-blocklist.conf.j2)
ok: [10.0.1.27] => (item=/home/ec2-user/trailkube/roles/os/templates/01-blocklist.conf.j2)

TASK [os : Restart Rsyslog] *****************************************************************************************************************************************************************************************
changed: [10.0.1.96]
changed: [10.0.1.27]
changed: [10.0.1.176]

TASK [os : Disable system swap] *************************************************************************************************************************************************************************************
changed: [10.0.1.176]
changed: [10.0.1.27]
changed: [10.0.1.96]

TASK [os : Remove current swaps from fstab] *************************************************************************************************************************************************************************
ok: [10.0.1.27]
ok: [10.0.1.176]
ok: [10.0.1.96]

TASK [os : Disable swappiness] **************************************************************************************************************************************************************************************
ok: [10.0.1.27] => (item={u'name': u'vm.swappiness', u'value': u'0'})
ok: [10.0.1.96] => (item={u'name': u'vm.swappiness', u'value': u'0'})
ok: [10.0.1.176] => (item={u'name': u'vm.swappiness', u'value': u'0'})

TASK [os : Create Ceph dir] *****************************************************************************************************************************************************************************************
ok: [10.0.1.176]
ok: [10.0.1.27]
ok: [10.0.1.96]

TASK [docker : get version] *****************************************************************************************************************************************************************************************
changed: [10.0.1.176]
changed: [10.0.1.27]
changed: [10.0.1.96]

TASK [docker : debug] ***********************************************************************************************************************************************************************************************
ok: [10.0.1.176] => {
    "msg": "Repo is : focal"
}
ok: [10.0.1.27] => {
    "msg": "Repo is : focal"
}
ok: [10.0.1.96] => {
    "msg": "Repo is : focal"
}

TASK [docker : Install docker and compose modules for Python] *******************************************************************************************************************************************************
ok: [10.0.1.27]
ok: [10.0.1.96]
ok: [10.0.1.176]

TASK [docker : Add Docker GPG key] **********************************************************************************************************************************************************************************
ok: [10.0.1.96]
ok: [10.0.1.176]
ok: [10.0.1.27]

TASK [docker : Add Docker APT repository] ***************************************************************************************************************************************************************************
ok: [10.0.1.96]
ok: [10.0.1.27]
ok: [10.0.1.176]

TASK [docker : Install list of packages] ****************************************************************************************************************************************************************************
ok: [10.0.1.27]
ok: [10.0.1.176]
ok: [10.0.1.96]

TASK [docker : Install docker-ce] ***********************************************************************************************************************************************************************************
ok: [10.0.1.27]
ok: [10.0.1.176]
ok: [10.0.1.96]

TASK [docker : Enable  and start  docker service] *******************************************************************************************************************************************************************
ok: [10.0.1.176]
ok: [10.0.1.27]
ok: [10.0.1.96]

TASK [docker : pause] ***********************************************************************************************************************************************************************************************
Pausing for 10 seconds
(ctrl+C then 'C' = continue early, ctrl+C then 'A' = abort)
ok: [10.0.1.176]

TASK [docker : Stop service docker, if started] *********************************************************************************************************************************************************************
changed: [10.0.1.176]
changed: [10.0.1.27]
changed: [10.0.1.96]

TASK [docker : pause] ***********************************************************************************************************************************************************************************************
Pausing for 10 seconds
(ctrl+C then 'C' = continue early, ctrl+C then 'A' = abort)
ok: [10.0.1.176]

TASK [docker : copy daemon.json] ************************************************************************************************************************************************************************************
ok: [10.0.1.27]
ok: [10.0.1.96]
ok: [10.0.1.176]

TASK [docker : Rsync to /apps/docker. Moving docker by LG standard] *************************************************************************************************************************************************
changed: [10.0.1.176 -> 10.0.1.176]
changed: [10.0.1.27 -> 10.0.1.27]
changed: [10.0.1.96 -> 10.0.1.96]

TASK [docker : Starting cleanup. Capture dir list to delete] ********************************************************************************************************************************************************
ok: [10.0.1.27]
ok: [10.0.1.96]
ok: [10.0.1.176]

TASK [docker : Complete clenup. Delete all except network] **********************************************************************************************************************************************************

TASK [docker : Enable  and start  docker service again] *************************************************************************************************************************************************************
changed: [10.0.1.176]
changed: [10.0.1.27]
changed: [10.0.1.96]

TASK [docker : Add the user {{ ansible_user }} to group of docker] **************************************************************************************************************************************************
ok: [10.0.1.96]
ok: [10.0.1.27]
ok: [10.0.1.176]

TASK [docker : Create Registry dir] *********************************************************************************************************************************************************************************
ok: [10.0.1.176]
ok: [10.0.1.27]
ok: [10.0.1.96]

TASK [docker : Copy CA] *********************************************************************************************************************************************************************************************
ok: [10.0.1.176]
ok: [10.0.1.27]
ok: [10.0.1.96]

TASK [docker : Copy Client Cert] ************************************************************************************************************************************************************************************
ok: [10.0.1.176]
ok: [10.0.1.27]
ok: [10.0.1.96]

TASK [docker : Copy Client Key] *************************************************************************************************************************************************************************************
ok: [10.0.1.176]
ok: [10.0.1.27]
ok: [10.0.1.96]

TASK [ha : Add Kubernetes GPG key] **********************************************************************************************************************************************************************************
ok: [10.0.1.176]
ok: [10.0.1.27]
ok: [10.0.1.96]

TASK [ha : Add Kubernetes APT repository] ***************************************************************************************************************************************************************************
ok: [10.0.1.176]
ok: [10.0.1.27]
ok: [10.0.1.96]

TASK [ha : Install kubeadm, kublet packages] ************************************************************************************************************************************************************************
ok: [10.0.1.27]
ok: [10.0.1.96]
ok: [10.0.1.176]

PLAY [kube-worker] **************************************************************************************************************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************************************************************************************************************
ok: [10.0.1.130]
ok: [10.0.1.36]
ok: [10.0.1.20]

TASK [os : Update and upgrade apt packages] *************************************************************************************************************************************************************************
ok: [10.0.1.130]
ok: [10.0.1.36]
ok: [10.0.1.20]

TASK [os : Add required prerequisite packages] **********************************************************************************************************************************************************************
ok: [10.0.1.130]
ok: [10.0.1.36]
ok: [10.0.1.20]

TASK [os : Download pip2] *******************************************************************************************************************************************************************************************
ok: [10.0.1.130]
ok: [10.0.1.36]
ok: [10.0.1.20]

TASK [os : Run pip2 install] ****************************************************************************************************************************************************************************************
changed: [10.0.1.130]
changed: [10.0.1.36]
changed: [10.0.1.20]

TASK [os : Set NTP service] *****************************************************************************************************************************************************************************************
ok: [10.0.1.130]
ok: [10.0.1.36]
ok: [10.0.1.20]

TASK [os : Set Rsyslog Blacklist] ***********************************************************************************************************************************************************************************
ok: [10.0.1.130] => (item=/home/ec2-user/trailkube/roles/os/templates/01-blocklist.conf.j2)
ok: [10.0.1.36] => (item=/home/ec2-user/trailkube/roles/os/templates/01-blocklist.conf.j2)
ok: [10.0.1.20] => (item=/home/ec2-user/trailkube/roles/os/templates/01-blocklist.conf.j2)

TASK [os : Restart Rsyslog] *****************************************************************************************************************************************************************************************
changed: [10.0.1.130]
changed: [10.0.1.36]
changed: [10.0.1.20]

TASK [os : Disable system swap] *************************************************************************************************************************************************************************************
changed: [10.0.1.130]
changed: [10.0.1.36]
changed: [10.0.1.20]

TASK [os : Remove current swaps from fstab] *************************************************************************************************************************************************************************
ok: [10.0.1.130]
ok: [10.0.1.36]
ok: [10.0.1.20]

TASK [os : Disable swappiness] **************************************************************************************************************************************************************************************
ok: [10.0.1.130] => (item={u'name': u'vm.swappiness', u'value': u'0'})
ok: [10.0.1.36] => (item={u'name': u'vm.swappiness', u'value': u'0'})
ok: [10.0.1.20] => (item={u'name': u'vm.swappiness', u'value': u'0'})

TASK [os : Create Ceph dir] *****************************************************************************************************************************************************************************************
ok: [10.0.1.130]
ok: [10.0.1.36]
ok: [10.0.1.20]

TASK [docker : get version] *****************************************************************************************************************************************************************************************
changed: [10.0.1.130]
changed: [10.0.1.36]
changed: [10.0.1.20]

TASK [docker : debug] ***********************************************************************************************************************************************************************************************
ok: [10.0.1.130] => {
    "msg": "Repo is : focal"
}
ok: [10.0.1.36] => {
    "msg": "Repo is : focal"
}
ok: [10.0.1.20] => {
    "msg": "Repo is : focal"
}

TASK [docker : Install docker and compose modules for Python] *******************************************************************************************************************************************************
ok: [10.0.1.130]
ok: [10.0.1.36]
ok: [10.0.1.20]

TASK [docker : Add Docker GPG key] **********************************************************************************************************************************************************************************
ok: [10.0.1.36]
ok: [10.0.1.130]
ok: [10.0.1.20]

TASK [docker : Add Docker APT repository] ***************************************************************************************************************************************************************************
ok: [10.0.1.130]
ok: [10.0.1.36]
ok: [10.0.1.20]

TASK [docker : Install list of packages] ****************************************************************************************************************************************************************************
ok: [10.0.1.130]
ok: [10.0.1.36]
ok: [10.0.1.20]

TASK [docker : Install docker-ce] ***********************************************************************************************************************************************************************************
ok: [10.0.1.130]
ok: [10.0.1.36]
ok: [10.0.1.20]

TASK [docker : Enable  and start  docker service] *******************************************************************************************************************************************************************
ok: [10.0.1.130]
ok: [10.0.1.36]
ok: [10.0.1.20]

TASK [docker : pause] ***********************************************************************************************************************************************************************************************
Pausing for 10 seconds
(ctrl+C then 'C' = continue early, ctrl+C then 'A' = abort)
ok: [10.0.1.130]

TASK [docker : Stop service docker, if started] *********************************************************************************************************************************************************************
changed: [10.0.1.130]
changed: [10.0.1.36]
changed: [10.0.1.20]

TASK [docker : pause] ***********************************************************************************************************************************************************************************************
Pausing for 10 seconds
(ctrl+C then 'C' = continue early, ctrl+C then 'A' = abort)
ok: [10.0.1.130]

TASK [docker : copy daemon.json] ************************************************************************************************************************************************************************************
ok: [10.0.1.130]
ok: [10.0.1.36]
ok: [10.0.1.20]

TASK [docker : Rsync to /apps/docker. Moving docker by LG standard] *************************************************************************************************************************************************
changed: [10.0.1.130 -> 10.0.1.130]
changed: [10.0.1.36 -> 10.0.1.36]
changed: [10.0.1.20 -> 10.0.1.20]

TASK [docker : Starting cleanup. Capture dir list to delete] ********************************************************************************************************************************************************
ok: [10.0.1.130]
ok: [10.0.1.36]
ok: [10.0.1.20]

TASK [docker : Complete clenup. Delete all except network] **********************************************************************************************************************************************************

TASK [docker : Enable  and start  docker service again] *************************************************************************************************************************************************************
changed: [10.0.1.130]
changed: [10.0.1.36]
changed: [10.0.1.20]

TASK [docker : Add the user {{ ansible_user }} to group of docker] **************************************************************************************************************************************************
ok: [10.0.1.130]
ok: [10.0.1.36]
ok: [10.0.1.20]

TASK [docker : Create Registry dir] *********************************************************************************************************************************************************************************
ok: [10.0.1.130]
ok: [10.0.1.36]
ok: [10.0.1.20]

TASK [docker : Copy CA] *********************************************************************************************************************************************************************************************
ok: [10.0.1.130]
ok: [10.0.1.36]
ok: [10.0.1.20]

TASK [docker : Copy Client Cert] ************************************************************************************************************************************************************************************
ok: [10.0.1.130]
ok: [10.0.1.36]
ok: [10.0.1.20]

TASK [docker : Copy Client Key] *************************************************************************************************************************************************************************************
ok: [10.0.1.130]
ok: [10.0.1.36]
ok: [10.0.1.20]

TASK [ha : Add Kubernetes GPG key] **********************************************************************************************************************************************************************************
ok: [10.0.1.130]
ok: [10.0.1.36]
ok: [10.0.1.20]

TASK [ha : Add Kubernetes APT repository] ***************************************************************************************************************************************************************************
ok: [10.0.1.130]
ok: [10.0.1.36]
ok: [10.0.1.20]

TASK [ha : Install kubeadm, kublet packages] ************************************************************************************************************************************************************************
ok: [10.0.1.36]
ok: [10.0.1.130]
ok: [10.0.1.20]

PLAY RECAP **********************************************************************************************************************************************************************************************************
10.0.1.130                 : ok=35   changed=7    unreachable=0    failed=0   
10.0.1.176                 : ok=35   changed=7    unreachable=0    failed=0   
10.0.1.20                  : ok=33   changed=7    unreachable=0    failed=0   
10.0.1.27                  : ok=33   changed=7    unreachable=0    failed=0   
10.0.1.36                  : ok=33   changed=7    unreachable=0    failed=0   
10.0.1.96                  : ok=33   changed=7    unreachable=0    failed=0   

```
</details>

### Update Security groups for below ports

__Control Plane Nodes__

|Protocol|	Direction  |Port Range|	Purpose   |	Used By|
|:------|:-------------|:---------|:----------|:----------|
|TCP    |    	Inbound|	6443* |	Kubernetes API server|	All|
|TCP    |	    Inbound|	2379-2380|	etcd server client API|	kube-apiserver, etcd|
|TCP    |    	Inbound|	10250 |	Kubelet API|	Self, Control plane|
|TCP    |	    Inbound|	10251 |	kube-scheduler|	Self|
|TCP    |	    Inbound|	10252 |	kube-controller-manager|	Self|

__Worker Nodes__

|Protocol|	Direction  |Port Range|	Purpose   |	Used By|
|:------|:-------------|:---------|:----------|:----------|
|TCP|	Inbound|	10250|	Kubelet API|	Self, Control plane|
|TCP|	Inbound|	30000-32767|	NodePort Services|	All|

### Create Load Balancer 

Create TCP load balancer for API `:6433`
and test it. As alternative LB troubleshooting option you can run simple python HTTPServer on port 6443 
``` 
ubuntu@master1:~$ nc -v  eu-reg1-vpc1-lb1-810156c505df9163.elb.eu-central-1.amazonaws.com 6443
nc: connect to eu-reg1-vpc1-lb1-810156c505df9163.elb.eu-central-1.amazonaws.com port 6443 (tcp) failed: Connection refused
```
### Update hosts files [optional]
`/etc/hosts`

by adding

```buildoutcfg
10.0.1.176   master1      
10.0.1.130   worker1      
10.0.1.35    etcd1        
10.0.1.96    master2      
10.0.1.36    worker2      
10.0.1.98    etcd2        
10.0.1.27    master3      
10.0.1.20    worker3      
10.0.1.190   etcd3        
```

### Start init 

create config on master1 node 

```buildoutcfg
root@master1:~# mkdir /etc/kubernetes/kubeadm
root@master1:~# vim /etc/kubernetes/kubeadm/kubeadm-config.yaml
```

update POD CIDR and LB details 

apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: stable
controlPlaneEndpoint: "eu-reg1-vpc1-lb1-810156c505df9163.elb.eu-central-1.amazonaws.com:6443"
networking:
  podSubnet: 10.12.208.0/20

issue init command on master1 node
```
sudo kubeadm init --node-name master1  --config=/etc/kubernetes/kubeadm/kubeadm-config.yaml --upload-certs
```
 
<details>
    <summary>Init output</summary>
     
     ### Save join commands for future use:
```buildoutcfg
ubuntu@master1:~$ sudo kubeadm init --node-name master1  --config=/etc/kubernetes/kubeadm/kubeadm-config.yaml --upload-certs
W1027 21:03:25.205356  131695 common.go:77] your configuration file uses a deprecated API spec: "kubeadm.k8s.io/v1beta1". Please use 'kubeadm config migrate --old-config old.yaml --new-config new.yaml', which will write the new, similar spec using a newer API version.
W1027 21:03:25.626058  131695 configset.go:348] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.19.3
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [eu-reg1-vpc1-lb1-810156c505df9163.elb.eu-central-1.amazonaws.com kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local master1] and IPs [10.96.0.1 10.0.1.176]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost master1] and IPs [10.0.1.176 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost master1] and IPs [10.0.1.176 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 10.020331 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.19" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
74785b700a725c0f84649605d0241a4f15f6463bcde37358c6e85e98f6b6626c
[mark-control-plane] Marking the node master1 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node master1 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: 4supa6.8qzb2esmlexfgor1
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join eu-reg1-vpc1-lb1-810156c505df9163.elb.eu-central-1.amazonaws.com:6443 --token 4supa6.8qzb2esmlexfgor1 \
    --discovery-token-ca-cert-hash sha256:63ef61b19e6a521781f23b57b9907cc7e83092d2f5b6cf846299d31ba5d8c1e7 \
    --control-plane --certificate-key 74785b700a725c0f84649605d0241a4f15f6463bcde37358c6e85e98f6b6626c

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join eu-reg1-vpc1-lb1-810156c505df9163.elb.eu-central-1.amazonaws.com:6443 --token 4supa6.8qzb2esmlexfgor1 \
    --discovery-token-ca-cert-hash sha256:63ef61b19e6a521781f23b57b9907cc7e83092d2f5b6cf846299d31ba5d8c1e7 
```
</details>

get access to cluster for kubectl

```buildoutcfg
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Check nodes

```buildoutcfg
ubuntu@master1:~$ kubectl get nodes
NAME      STATUS     ROLES    AGE     VERSION
master1   NotReady   master   3m59s   v1.19.3

```
### Deploy CNI - calico

- download manifest `curl https://docs.projectcalico.org/manifests/calico.yaml -O`
- edid your POD CIDR
```buildoutcfg
 - name: CALICO_IPV4POOL_CIDR
   value: "10.12.208.0/20"
```
-apply config          
```
ubuntu@master1:~$ kubectl create -f  calico.yaml --record
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.apps/calico-node created
serviceaccount/calico-node created
deployment.apps/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
```
- Check status
```
ubuntu@master1:~$ kubectl get pods -n kube-system
NAME                                     READY   STATUS    RESTARTS   AGE
calico-kube-controllers-7d569d95-fn8nx   1/1     Running   0          32s
calico-node-52n9h                        1/1     Running   0          32s
coredns-f9fd979d6-f5lj7                  1/1     Running   0          5m6s
coredns-f9fd979d6-wfwm4                  1/1     Running   0          42m
etcd-master1                             1/1     Running   0          42m
kube-apiserver-master1                   1/1     Running   0          42m
kube-controller-manager-master1          1/1     Running   0          42m
kube-proxy-5ngk6                         1/1     Running   0          42m
kube-scheduler-master1                   1/1     Running   0          42m
```
### Joinig other Control Plane nodes

<details>
    <summary>Init output</summary>
     
     ### same for master2 and master 3:
```
ubuntu@master2:~$ sudo kubeadm join eu-reg1-vpc1-lb1-810156c505df9163.elb.eu-central-1.amazonaws.com:6443 --token 4supa6.8qzb2esmlexfgor1     --discovery-token-ca-cert-hash sha256:63ef61b19e6a521781f23b57b9907cc7e83092d2f5b6cf846299d31ba5d8c1e7     --control-plane --certificate-key 74785b700a725c0f84649605d0241a4f15f6463bcde37358c6e85e98f6b6626c
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks before initializing the new control plane instance
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[download-certs] Downloading the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost master2] and IPs [10.0.1.96 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost master2] and IPs [10.0.1.96 127.0.0.1 ::1]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [eu-reg1-vpc1-lb1-810156c505df9163.elb.eu-central-1.amazonaws.com kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local master2] and IPs [10.96.0.1 10.0.1.96]
[certs] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[certs] Using the existing "sa" key
[kubeconfig] Generating kubeconfig files
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[check-etcd] Checking that the etcd cluster is healthy
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
[etcd] Announced new etcd member joining to the existing etcd cluster
[etcd] Creating static Pod manifest for "etcd"
[etcd] Waiting for the new etcd member to join the cluster. This can take up to 40s
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[mark-control-plane] Marking the node master2 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node master2 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]

This node has joined the cluster and a new control plane instance was created:

* Certificate signing request was sent to apiserver and approval was received.
* The Kubelet was informed of the new secure connection details.
* Control plane (master) label and taint were applied to the new node.
* The Kubernetes control plane instances scaled up.
* A new etcd member was added to the local/stacked etcd cluster.

To start administering your cluster from this node, you need to run the following as a regular user:

        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config

Run 'kubectl get nodes' to see this node join the cluster.

ubuntu@master2:~$ 
```
</details>

Check status
```
ubuntu@master1:~$ kubectl get pods -n kube-system
NAME                                     READY   STATUS    RESTARTS   AGE
calico-kube-controllers-7d569d95-fn8nx   1/1     Running   0          12m
calico-node-52n9h                        0/1     Running   0          12m
calico-node-9j5jp                        0/1     Running   0          6m48s
calico-node-l95zg                        0/1     Running   0          4m5s
coredns-f9fd979d6-f5lj7                  1/1     Running   0          16m
coredns-f9fd979d6-wfwm4                  1/1     Running   0          53m
etcd-master1                             1/1     Running   0          54m
etcd-master2                             1/1     Running   0          9m7s
etcd-master3                             1/1     Running   0          6m44s
kube-apiserver-master1                   1/1     Running   0          54m
kube-apiserver-master2                   1/1     Running   0          9m11s
kube-apiserver-master3                   1/1     Running   0          6m48s
kube-controller-manager-master1          1/1     Running   1          54m
kube-controller-manager-master2          1/1     Running   0          9m10s
kube-controller-manager-master3          1/1     Running   0          6m47s
kube-proxy-5ngk6                         1/1     Running   0          53m
kube-proxy-n7sgs                         1/1     Running   0          9m12s
kube-proxy-slvg6                         1/1     Running   0          6m48s
kube-scheduler-master1                   1/1     Running   1          54m
kube-scheduler-master2                   1/1     Running   0          9m10s
kube-scheduler-master3                   1/1     Running   0          6m47s
```
### Joining worker nodes
```
sudo kubeadm join eu-reg1-vpc1-lb1-810156c505df9163.elb.eu-central-1.amazonaws.com:6443 --token 4supa6.8qzb2esmlexfgor1 \
    --discovery-token-ca-cert-hash sha256:63ef61b19e6a521781f23b57b9907cc7e83092d2f5b6cf846299d31ba5d8c1e7 
```

- check status 
```  
ubuntu@master1:~$ kubectl get nodes
NAME      STATUS   ROLES    AGE    VERSION
master1   Ready    master   57m    v1.19.3
master2   Ready    master   12m    v1.19.3
master3   Ready    master   10m    v1.19.3
worker1   Ready    <none>   101s   v1.19.3
worker2   Ready    <none>   62s    v1.19.3
worker3   Ready    <none>   35s    v1.19.3
```
## ETCD members check 

```buildoutcfg
ubuntu@master1:~$ sudo etcdctl --endpoints="https://10.0.1.176:2379" member list --write-out=table --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key
+------------------+---------+---------+-------------------------+-------------------------+------------+
|        ID        | STATUS  |  NAME   |       PEER ADDRS        |      CLIENT ADDRS       | IS LEARNER |
+------------------+---------+---------+-------------------------+-------------------------+------------+
| 3ebcd08e21d0b465 | started | master1 | https://10.0.1.176:2380 | https://10.0.1.176:2379 |      false |
| ba608381a32211c5 | started | master3 |  https://10.0.1.27:2380 |  https://10.0.1.27:2379 |      false |
| ffb4ed65d6a9b40b | started | master2 |  https://10.0.1.96:2380 |  https://10.0.1.96:2379 |      false |
+------------------+---------+---------+-------------------------+-------------------------+------------+

```