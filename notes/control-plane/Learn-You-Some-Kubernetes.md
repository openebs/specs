# Table of Contents
0. [Kubernetes Made Easy](#kubernetes-made-easy)
1. [Kubernetes from Scratch](#kubernetes-from-scratch)
2. [Kubernetes on Fedora - Single Node](#kubernetes-on-fedora---single-node)
3. [Kubernetes on Fedora - Multi Node](#kubernetes-on-fedora---multi-node)
4. [Kubernetes on Ubuntu](#kubernetes-on-ubuntu)
5. [Others](#others)

## Kubernetes Made Easy

#### Kubernetes Jargons

- Host roles can be:
  - master represented as `a` or
  - minion represented as `i` or
  - both represented as `ai`
- Networking can be:
  - CNI based or
  - Flannel based
- Admission Controllers - ??
- NodePort visibility - ??
- federation ??


#### FAQs w.r.t Kubernetes Installation

- What is the starting point ?
  - Download the latest release from following link:
    - [releases](https://github.com/kubernetes/kubernetes/releases)
  - Does the `cluster` directory present in the releases' tar file maps to following link ?
    - https://github.com/kubernetes/kubernetes/tree/master/cluster
- CLOUD_PROVIDER vs KUBERNETES_PROVIDER options. How different as these ?


## Hardening Kubernetes

#### Lint the scripts

## Kubernetes from Scratch

- Refer [scratch](http://kubernetes.io/docs/getting-started-guides/scratch/)

#### Cloud Provider 
- Specs - pkg/cloudprovider/cloud.go
- Manages load balancers, nodes, & networking routes

#### On IPs

- Need to provide a block of IPs to Kubernetes to be used as PODs IPs
- Pod to pod communication via IP of the pod 
  - overlay network 
    - via traffic encapsulation
  - without overlay  
    - Configure switches, routers to be aware of POD IPs
    - Better performance
- Supports CNI network plugin interface
- IPv6 is not supported for POD IPs
  - e.g. use 10.10.0.0/16 as the range for the cluster, 
    - with up to 256 nodes using 10.10.0.0/24 through 10.10.255.0/24, respectively.
  - e.g. A /24 per node supports 254 pods per machine and is a common choice. 
    - If IPs are scarce, a /26 (62 pods per machine) or even a /27 (30 pods) may be sufficient.
- Kubernetes allocates an IP to each service.
  - However these IPs need not be routable
  - kube-proxy takes care of translating Service IPs to POD IPs before traffic leaves the node
  - Need to allocate a block of IPs for the services 
    - e.g. SERVICE_CLUSTER_IP_RANGE = "10.0.0.0/16" means 65534 services can be active simultaneously
- Pick up a static IP for master node
  - e.g. **MASTER_IP**
- Open access to apiserver ports 80 &/or 443
 - Enable ipv4 forwarding: sysctl net.ipv4.ip_forward = 1
- Can have fine grained networking policy between PODs using Network Policy resource

#### Regions

- Clusters can be represented as regions
  - Provide a cluster name e.g. **CLUSTER_NAME**

#### Binaries

- The Kubernetes binaries required are:
  - etcd, docker/rkt, kubelet, kube-proxy, kube-apiserver, kube-controller-manager, kube-scheduler
  - https://github.com/kubernetes/kubernetes/releases/latest
  - Kubernetes binary has all its dependant binaries
- **What should run outside the container ?**
  - *docker, kubelet, kube-proxy* ~ same way as we run the system daemons
  - Hence bare binaries are reqd
- **What should run as-a container ?**
  - *etcd, kube-apiserver, kube-controller-manager, & kube-scheduler*
  - Hence images are required
  - Get the images from gcr.io/google_containers/hyperkube:$TAG
- Building images is also possible
  - Release contain files s.a ./kubernetes/server/bin/kube-apiserver.tar
  - Convert above file to an image *docker load -i kube-apiserver.tar*
  - Similarly for etcd *cd kubernetes/cluster/images/etcd; make*
  - NOTE: **use the etcd version provided in the Kubernetes distribution**
  - Check the value of TAG in the *kubernetes/cluster/images/etcd/Makefile*
- Setting the env vars
  - HYPERKUBE_IMAGE=gcr.io/google_containers/hyperkube:$TAG
  - ETCD_IMAGE=gcr.io/google_containers/etcd:$ETCD_VERSION
- Verify the image & tag
  - *docker images*
- NOTE - hyperkube is a all-in-one binary
  - hyperkube kubelet runs kubelet
  - hyperkube apiserver runs apiserver

#### HTTPS access to apiserver

- refer [creating certificates](http://kubernetes.io/docs/admin/authentication/#creating-certificates)
- Need to prepare the certs & credentials
- Master needs a cert to act as an HTTPS server
- kubelets optionally need certs
  - to identify themselves as clients of the master
  - & also while serving its API over HTTPS
- Generate a root cert
  - use this to sign the master, kublet & kubectl certs
- Files
  - CA_CERT is put at apiserver node @ */srv/kubernetes/ca.crt*
  - MASTER_CERT signed by CA_CERT & put at apiserver node @ */srv/kubernetes/server.crt*
  - MASTER_KEY put at apiserver node @ */srv/kubernetes/server.key*
  - KUBELET_CERT *optional*
  - KUBELET_KEY *optional*


#### Credentials

- Admin needs token or password to get identified
  - tokens are alphanumeric e.g. 32 chars
- Tokens need to be stored in a file for the apiserver to read
  - */var/lib/kube-apiserver/known_tokens.csv*
- How to distribute credentials to clients ?
  - Put them into a **kubeconfig** file
  - refer [kubeconfig](http://kubernetes.io/docs/user-guide/kubeconfig-file/)
  - Use kubectl commandline options

#### kubconfig file for administrators

```bash

# non-https option
kubectl config set-cluster $CLUSTER_NAME --server=http://$MASTER_IP --insecure-skip-tls-verify=true

# https option
kubectl config set-cluster $CLUSTER_NAME --certificate-authority=$CA_CERT --embed-certs=true --server=https://$MASTER_IP

# Set the default cluster to use
kubectl config set-context $CONTEXT_NAME --cluster=$CLUSTER_NAME --user=$USER
kubectl config use-context $CONTEXT_NAME
```

#### kubeconfig file for others

- What are others here ?
  - kubelet & kube-proxy
- There are 3 possible options
  - same credentials as the admin
  - one token & kubeconfig file for all kubelets, one for all kube-proxy & one for admin
  - different credentials for every kubelet, etc
- How to do it ?
  - copy $HOME/.kube/config
  - or learn the code @ cluster/gce/configure-vm.sh
  - or a template
  - to dest @ 
    - /var/lib/kube-proxy/kubeconfig
    - /var/lib/kubelet/kubeconfig

#### Configuring Base Software on Nodes

- Base OS + docker + kubelet + kube-proxy
- Docker version depends on kubelet version
- Need to install docker with Kubernetes specific options
  - else remove the networking settings

```bash
iptables -t nat -F
ip link set docker0 down
ip link delete docker0
```

#### Configure Docker as per Kubernetes

- Create own bridge for the per-node CIDR ranges
  - set --bridge=cbr0
- Docker should not manage iptables for host-ports & instead let kube-proxy do this
  - set --iptables=false
- If we want POD IP to be routable
  - set --ip-masq=false
- If using flannel, need to consider the extra packet size due to udp encapsulation
  - set --mtu=
- If you want to connect to a private registry without using SSL
  - --insecure-registry $CLUSTER_SUBNET
- Perhaps need to increase the no. of open files for docker
  - DOCKER_NOFILE=1000000
- NOTE: *Ensire docker works correctly before proceeding with rest of install*

#### Configure kubelet - the node agent

- All nodes should run kubelet
- Args to consider:
  - HTTPS approach
    - set --api-servers=https://$MASTER_IP
    - set --kubeconfig=/var/lib/kubelet/kubeconfig
  - HTTP approach
    - set --api-server=http://$MASTER_IP
  - set --config=/etc/kubernetes/manisfests
  - set --cluster-dns= address of DNS server
  - set --cluster-domain= dns domain prefix
  - set --docker-root=
  - set --root-dir=
  - set --configure-cbr0=
  - set --register-node=

#### Configure kube-proxy

- just the HTTPS or HTTP approach

#### Networking

- Each node needs to be allocated its own CIDR for pod networking
  - NODE_X_POD_CIDR
- A bridge needs to be created on each node cbr0
  - The bridge needs an address from $NODE_X_BRIDGE_ADDR
  - By convention the first IP is used i.e. NODE_X_BRIDGE_ADDR
  - e.g. if NODE_X_POD_CIDR = 10.0.0.0/16
  - then NODE_X_BRIDGE_ADDR is 10.0.0.1/16
- automatic approach
  - set --configure-cbr0=true in kubelet option & restart kublet service
  - NOTE: kubelet will configure cbr0 automatically
- manual approach
  - set --configure-cbr0=false on kubelet & restart
  - create bridge
    - *`ip link add name cbr0 type bridge`*
  - set mtu
    - *`ip link set dev cbr0 mtu 140`*
  - add node's network to the bridge ~ docker will go on other side of bridge
    - *`ip addr add $NODE_X_BRIDGE_ADDR` dev cbr0*
  - turn it up
    - *`ip link set dev cbr0 up`*
  - If PODs need to communicate with each other
    - turn off docker's IP masquerading
  - masquerade for the destination IPs outside the cluster networking
    - iptables -t nat -A POSTROUTING ! ${CLUSTER_SUBNET} -m addrtype ! --dst-type LOCAL -j MASQUERADE

#### House Keeping Tasks

- configure log rotation e.g. logrotate
- setup liveness monitoring e.g. supervisord

#### Bootstrapping the Cluster

- Node services e.g. kubelet, kube-proxy & docker are typically started & managed using automation approaches
- The remaining master components of Kubernetes are all configured & managed by Kubernetes

#### etcd pod

- 2 approaches
  - Run one etcd instance, with its log written to a durable storage (RAID, GCE PD)
  - Run 3 to 5 etcd instances
    - logs need not be written to durable storage because storage is replicated
    - run a single apiserver which connects to one of the etcd nodes
- Run an etcd instance
  - copy *`cluster/saltbase/salt/etcd/etcd.manifest`*
  - Make necessary modifications
  - Start the pod by putting it into the kubelet manifest directory
- So etcs runs as a pod

#### apiserver pod

- Each one of them runs as *a pod on the master node*
- Start with a provided template
- Flags that need to be set
  - --cloud-provider=
  - --cloud-config=
  - --address=${MASTER_IP} 
    - or --bind-address=127.0.0.1 & --address=127.0.0.1 to run a proxy on a master node
  - --cluster-name=$CLUSRER_NAME
  - --service-cluster-ip-range=$SERVICE_CLUSTER_IP_RANGE
  - --etcd-servers=http://127.0.0.1:4001
  - --tls-cert-file=/srv/kubernetes/server.cert
  - --tls-private-key-file=/srv/kubernetes/server.key
  - --admission-control=$RECOMMENDED_LIST
  - --allow-priviledged=true
    - if you trust your cluster user to run pods as root
- For HTTP only approach
  - --token-auth-file=/dev/null
  - --insecure-bind-address=$MASTER_IP
  - --advertise-address=$MASTER_IP
- For HTTPS approach
  - --client-ca-file=/srv/kubernetes/ca.crt
  - --token-auth-file=/srv/kubernetes/known_tokens.csv
  - --basic-auth-file=/srv/kubernetes/basic_auth.csv
- POD mounts several file system directories of the node
  - via hostPath
  - /etc/ssl - allows apiserver to find SSL root certs to authenticate the cloud provider
    - not required if bare metal
  - /srv/kubernetes - allows apiserver to read certs & credentials stored on node disk
    - these could instead be stored on a persistent disk e.g. GCE PD
  - optionally mount /var/log as well & redirect output
    - this is reqd. if *logs need to be accessible from the root filesystem with tools like journalctl*

#### scheduler pod

- use the required manifest file

#### controller-manager pod

- use the required manifest
- following flags may be considered
  - --cluster-name=$CLUSTER_NAME
  - --cluster-cidr=
  - --allocate-node-cidrs=
  - --cloud-provider=
  - --cloud-config=
  - --service-account-private-key-file=/srv/kubernetes/server.key
  - --master=127.0.0.1:8080

#### Cloud Providers

- --cloud-provuder
  - aws, gce, mesos, vagrant, unset
  - unset is used for bare metal
- some of these cloud providers require a config file setting

#### Starting & Verifying apiserver, etcd & controller-manager pods **`TAG - SUPPORT`**

- Place each pod template into kubelet config dir
  - i.e. --config= option of kubelet
- docker ps | grep apiserver
- echo $(curl -s http://localhost:8080/healthz)
- curl -s http://localhost:8080/api
- if --register-node=true is set for kubelets
  - kubelet will begin self-registering with the apiserver
  - kubectl get nodes

#### Cluster Addon - DNS

#### Cluster Addon - Logging

#### Cluster Addon - Container Resource Monitoring


## Kubernetes on Fedora - Single Node

#### Install Kubernetes

- services apiserver, scheduler, controller-manager, kubelet & kube-proxy will be managed by systemd
- configuration resides at /etc/kubernetes
- services will be divided between hosts
  - fed-master - some IP address
    - runs apiserver, controller-manager & scheduler & etcd
  - fed-node - some other IP address
    - runs kubelet, kube-proxy & docker
- install kubernetes on all hosts
  - --enablerepo=updates-testing directive in yum command ensures most recent Kuberenetes version is installed
  - you may want to download the RPM & do a yum install rpm

#### Other installs

- install etcd & iptables
- add master & node to /etc/hosts on all machines
  - not required if hostnames are already in DNS
  - make sure communication via ping works
- disable the *firewalld* & *iptables-services* on both hosts
  - Docker does not play well with other firewall rule managers

```bash
systemctl disable iptables-services firewalld
systemctl stop iptables-services firewalld
```

#### Edit Kubernetes Config on all hosts

- edit /etc/kubernetes/config
  - this will be same across all hosts

```bash

# Comma separated list of nodes in the etcd cluster
KUBE_MASTER="--master=http://fed-master:8080"

# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=false"
```

#### etcd.conf on master

```bash
# /etc/etcd/etcd.conf
# Let it listen to all IP instead of 127.0.0.1
# etcd 2.0 uses port 2379 & 2380, older versions used 4001 & 7001
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
```

#### /var/run on master

```bash
mkdir /var/run/kubernetes
chown kube:kube /var/run/kubernetes
chmod 750 /var/run/kubernetes
```

#### Start appropriate services on master

```bash
for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler; do
	systemctl restart $SERVICES
	systemctl enable $SERVICES
	systemctl status $SERVICES
done
```

#### node.json for fed-node on master

- create a node.json file on master
  - it contains info about nodes (*not master*)
- Run it against kubectl
  - `kubectl create -f ./node.json`
- Verify via kubectl get nodes
- NOTE - Check if the option to auto-register works out
- NOTE - This is just a representation & not actual provisioning of fed-node

#### kubelet configuration on node

```bash
# /etc/kubernetes/kubelet
# Kubernetes kubelet (node) config

# The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--address=0.0.0.0"

# You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=fed-node"

# location of the api-server
KUBELET_API_SERVER="--api-servers=http://fed-master:8080"

# Add your own!
#KUBELET_ARGS=""
```

#### Start appropriate services on node

```bash
for SERVICES in kube-proxy kubelet docker; do 
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES 
done
```

- Run *`kubectl get nodes`* @ master
- To delete the node from cluster
  - kubectl delete -f ./node.json

## Kubernetes on Fedora - Multi Node

#### Assumptions

- Nodes should have different names & labels
- Kubernetes master is running etcd, apiserver, control-manager & scheduler
- Kubernetes nodes are running kube-proxy, docker & kubelet
- Nodes should have Fedora installed
- We shall use kernel based vxlan as flannel's backend

#### Introducting Flannel

- Install flannel on Kubernetes nodes
- Flannel on each node configures an overlay network that docker uses
- Flannel runs on each node to setup a unique class-C container network
- Flannel provides udp & vxlan among other overlay networking backend options

#### flannel config on master

- flannel-config.json
  - choose an IP that is not part of public IP range

```json
{
    "Network": "18.16.0.0/16",
    "SubnetLen": 24,
    "Backend": {
        "Type": "vxlan",
        "VNI": 1
     }
}
```

- add config to etcd server on master
  - *`etcdctl set /coreos.com/network/config < flannel-config.json`*
- verify **`TAG - SUPPORT`**
  - *`etcdctl get /coreos.com/network/config`*

#### Commands on all Kubernetes nodes

- w.r.t flannel

```bash
# /etc/sysconfig/flanneld
# Flanneld configuration options

# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD="http://fed-master:2379"

# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
FLANNEL_ETCD_KEY="/coreos.com/network"

# Any additional options that you want to pass
FLANNEL_OPTIONS=""
```

- By default, flannel uses the interface for default route.
  - If multiple interfaces & want to use a non-default one
  - add *`-iface=`*
- Enable flannel service
  - systemctl enable flanneld
- Start flannel service
  - systemctl start flanneld
- flannel vs docker bridge
  - if docker is running then stop docker
  - delete docker bridge ~ docker0
  - start flanneld
  - restart docker or reboot system

```bash
  systemctl stop docker
  ip link delete docker0
  systemctl start flanneld
  systemctl start docker
```

#### Verify flannel & cluster configurations on Nodes `TAG - SUPPORT`

```bash
# ip -4 a|grep inet
    inet 127.0.0.1/8 scope host lo
    inet 192.168.122.77/24 brd 192.168.122.255 scope global dynamic eth0
    inet 18.16.29.0/16 scope global flannel.1
    inet 18.16.29.1/24 scope global docker0
```

- docker0 is assigned a subnet 18.16.29.1/24 on each Kubernetes node
- IP addresses of flannel & docker are in the same network

- If it is a 1 master, 3 nodes cluster, you will notice a similar output
  - one block for each node showing the subnets they have been assigned
  - try associating the subnet to a node via the MAC address & public IP

```bash
# curl -s http://fed-master:4001/v2/keys/coreos.com/network/subnets | python -mjson.tool

{
    "node": {
        "key": "/coreos.com/network/subnets",
            {
                "key": "/coreos.com/network/subnets/18.16.29.0-24",
                "value": "{\"PublicIP\":\"192.168.122.77\",\"BackendType\":\"vxlan\",\"BackendData\":{\"VtepMAC\":\"46:f1:d0:18:d0:65\"}}"
            },
            {
                "key": "/coreos.com/network/subnets/18.16.83.0-24",
                "value": "{\"PublicIP\":\"192.168.122.36\",\"BackendType\":\"vxlan\",\"BackendData\":{\"VtepMAC\":\"ca:38:78:fc:72:29\"}}"
            },
            {
                "key": "/coreos.com/network/subnets/18.16.90.0-24",
                "value": "{\"PublicIP\":\"192.168.122.127\",\"BackendType\":\"vxlan\",\"BackendData\":{\"VtepMAC\":\"92:e2:80:ba:2d:4d\"}}"
            }
    }
}
```

#### Review subnet file generated by flannel `TAG - SUPPORT`

```bash
# cat /run/flannel/subnet.env
FLANNEL_SUBNET=18.16.29.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=false
```

#### Test cross host container communication `TAG - SUPPORT`

- verify multi-node cluster setup for overlay networking setup by flannel
  - run the commands on any 2 nodes
    - `docker run -it fedora:latest bash`
    - this will place you inside the container
  - install iproute & iputils packages to install ip & ping utilities
    - `yum -y install iproute iputils`
    - `setcap cap_net_raw-ep /usr/bin/ping`
  - note the IP address
    - `ip -4 a l eth0 | grep inet`
    - inet 18.16.29.4/24 scope global eth0
  - repeat the same thing at the other node & its container
  - ping from within a container to other container's IP

## Kubernetes on Ubuntu

#### Assumptions

- All nodes have docker & bridge-utils installed
- Nodes reachable to each other
- Master node needs to be reachable to internet
- all remote servers can be ssh logged in without a password by using key authentication
- 1 master & 2 nodes

#### git clone locally & configure

- `git clone --depth 1 https://github.com/kubernetes/kubernetes.git`
- customizing versions can be done via:
  - `export KUBE_VERSION=1.2.0`
  - `export FLANNEL_VERSION=0.5.0`
  - `export ETCD_VERSION=2.2.0`
- configure cluster info

```bash
# cluster/ubuntu/config-default.sh

export nodes="vcap@10.10.103.250 vcap@10.10.103.162 vcap@10.10.103.223"
export roles="ai i i"
export NUM_NODES=${NUM_NODES:-3}
export SERVICE_CLUSTER_IP_RANGE=192.168.3.0/24
export FLANNEL_NET=172.16.0.0/16
```

- ai refers to master as well as node roles
- i refers to node role
- FLANNET_NET should not conflict with SERVICE_CLUSTER_IP_RANGE

#### Bring up the cluster

- cd cluster/
- Run `KUBERNETES_PROVIDER=ubuntu ./kube-up.sh`
  - script uses scp to copy binaries & config files to all machines
  - You will need to type sudo password when prompted

#### Verify `TAG - SUPPORT`

- cd cluster/ubuntu/binaries
- make the kubectl availalbe via PATH
- kubectl get nodes

#### Troubleshoot `TAG - SUPPORT`

- Check /var/log/upstart/etcd.log
- Bring down & up the cluster
  - `KUBERNETES_PROVIDER=ubuntu ./kube-down.sh`
  - `KUBERNETES_PROVIDER=ubuntu ./kube-up.sh`
- Customizations
  - edit `/etc/default/{component_name}`
  - then restart `sudo service {component_name} restart`
- Upgrading the cluster
  - cd cluster/
  - Run `KUBERNETES_PROVIDER=ubuntu ./kube-push.sh [-m|-n <node id>] <version>`
  - -m refers to master
  - -n <node id> refers to specific node
  - default is upgrade all nodes
  - **Where will the new binaries be sourced from ?**
    - tree cluster/ubuntu/binaries
  - NOTE: This script only replaces the binaries. It does not delete any resource
- Verify Upgrade
  - At master `kubectl version`
  - At node `cd /opt/bin && sudo ./kubelet --version`


## Others

#### Cluster Troubleshooting - TODO

- http://kubernetes.io/docs/admin/cluster-troubleshooting/

#### More on Networking - TODO

- http://kubernetes.io/docs/admin/networking/
