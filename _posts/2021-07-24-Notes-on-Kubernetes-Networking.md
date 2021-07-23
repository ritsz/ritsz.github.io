### Reference 

* [Service networking](https://www.youtube.com/watch?v=NFApeJRXos4)
* [Ingress networking](https://www.youtube.com/watch?v=40VfZ_nIFWI)
* [Understanding K8s networking](https://www.youtube.com/watch?v=U35C0EPSwoY)
* [Kubernetes networking with flannel](https://blog.laputa.io/kubernetes-flannel-networking-6a1cb1f8ec7c)
* [Write a CNI plugin from scratch](https://www.youtube.com/watch?v=zmYxdtFzK6s)

### Introduction
* Kubernetes imposes the following fundamental requirements on any networking implementation (barring any intentional network segmentation policies):
  *  pods on a node can communicate with all pods on all nodes without NAT
  * agents on a node (e.g. system daemons, kubelet) can communicate with all pods on that node
  * Note: For those platforms that support Pods running in the host network (e.g. Linux): pods in the host network of a node can communicate with all pods on all nodes without NAT
* kubeproxy programs the linux kernel to intercept connections to the clusterIP and load balance them across all the Pods backing that clusterIP service (across all endpoints associated with the service).

### Network Address Translations
* When the destination is a **clusterIP**, kubeproxy NATs the destination from service IP to one Pod IP. (Note: The service on Node1 to NAT the connection to a Pod on Node2, services do NAT-ing cluster wide).
* In a **Pod to service communication**:
  * `[Pod1, Service2]`, NAT happens by kubeproxy and packed is changed to `[Pod1, Pod2]`.
  * On the return traffic, the NAT is reversed such that `[Pod2, Pod1]` changes to `[Service2, Pod1]`.
  * Hence Pod1 doesn't know about any NAT that was applied.
* Traffic from **external client to a NodePort** is also load balanced using NAT.
  * NodePort is a port on each node that is assigned to a service. 
  * `[ClientIP, Node1:NodePort]` is NAT to `[Node1, Pod2]`. 
  * Notice that source ClientIP is changed to Node1 ip address, and destination Node1 IP address is changed to Pod's IP.
  * Source IP is also NAT-ed so that Pod replies back to the Node1 IP, and not directly to the external ClientIP.
  * The NAT is reversed so that external client feels it got a reply from the `Node1:NodePort` 
* ServiceIPs can be advertised externally using BGP. This requires the underlying infrastructure (IaaS network) to be running BGP. Since the serviceIPs are advertised externally, clients can connect to serviceIP's directly. The `[ClientIP, Service2]` is NAT to `[Node1, Pod2]`. (Source changed to NodeIP so that return traffic can again be reverse NAT- ed).
* Load balancer exposes an external IP address for the services. Load balancer NATs the external IP to a `NodeIP:NodePort` (and hence load balances across the nodes). `[ClientIP, ExternalIP]` NATs to `[ClientIP, Node1:NodePort]`. This then NATs like any NodePort NAT-ing to [`Node1, Pod2]`.
* For ingress controllers, an Ingress Pod is added (running for example Nginx). Load balancer maps connection to a NodePort. NodePort maps connection to Ingress Pod. Ingress Pod takes a look at the HTTP request and maps the connection to a Pod serving that HTTP route. It also updates the source IP to itself.

```sh
  [ClientIP, External IP] -> [ClientIP, Node1] -> [ClientIP, IngressPod] -> [IngressPod, Pod2]
```
* Application Load Balancer: Load balancer provided by the cloud provider that does the work of L7 ingress controller routing as well.
* Flannel uses VXLAN, Calico uses IPIP for overlay networking.

### Test Setup

```yaml
Master-1: 10.192.214.204
Worker-1: 10.78.180.28
worker-2: 10.161.119.138
```

```sh
[root@sc2-rdops-vm06-dhcp-214-204 ~]# ip route
default via 10.192.223.253 dev eth0 proto dhcp metric 100
10.192.192.0/19 dev eth0 proto kernel scope link src 10.192.214.204 metric 100
```
### Install docker

```
[root@sc2-rdops-vm06-dhcp-214-204 ~]# ## List of commands
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

dnf install https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.6-3.3.el7.x86_64.rpm

dnf remove podman-manpages
dnf install docker-ce --allowerasing
systemctl enable docker
systemctl start docker
```
* New `docker0` interface is created.

```sh
Master:  
  docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
          inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
          ether 02:42:58:bf:b9:80  txqueuelen 0  (Ethernet)
          RX packets 0  bytes 0 (0.0 B)
          RX errors 0  dropped 0  overruns 0  frame 0
          TX packets 0  bytes 0 (0.0 B)
          TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
  
  eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
          inet 10.192.214.204  netmask 255.255.224.0  broadcast 10.192.223.255
          inet6 fe80::39ff:fe42:dd2a  prefixlen 64  scopeid 0x20<link>
          inet6 fd01:1:2:2916:0:39ff:fe42:dd2a  prefixlen 64  scopeid 0x0<global>
          inet6 fd01:1:2:2916:0:a:0:5f  prefixlen 128  scopeid 0x0<global>
          ether 02:00:39:42:dd:2a  txqueuelen 1000  (Ethernet)
          RX packets 244505  bytes 193196862 (184.2 MiB)
          RX errors 0  dropped 20069  overruns 0  frame 0
          TX packets 17578  bytes 1643757 (1.5 MiB)
          TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
  
Worker:
  docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
          inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
          ether 02:42:b3:9f:f5:cf  txqueuelen 0  (Ethernet)
          RX packets 0  bytes 0 (0.0 B)
          RX errors 0  dropped 0  overruns 0  frame 0
          TX packets 0  bytes 0 (0.0 B)
          TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
  
  eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
          inet 10.78.180.28  netmask 255.255.240.0  broadcast 10.78.191.255
          inet6 fd01:0:106:5:0:8dff:fe6c:dc15  prefixlen 64  scopeid 0x0<global>
          inet6 fd01:0:106:5:0:a:0:18d2  prefixlen 128  scopeid 0x0<global>
          inet6 fe80::8dff:fe6c:dc15  prefixlen 64  scopeid 0x20<link>
          ether 02:00:8d:6c:dc:15  txqueuelen 1000  (Ethernet)
          RX packets 1156239  bytes 465947268 (444.3 MiB)
```
* `docker0` becomes the default gateway for `172.17.0.0/16` network.

```sh
[root@sc1-10-78-180-28 worker]# ip route
default via 10.78.191.254 dev eth0 proto dhcp metric 100
10.78.176.0/20 dev eth0 proto kernel scope link src 10.78.180.28 metric 100
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
```
* `swapoff -a`

### Install kubernetes

```
> cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

> dnf install kubeadm -y 
> systemctl enable kubelet
> systemctl start kubelet
```
* Start kubeadm:

```
> kubeadm config images pull

> kubeadm init --pod-network-cidr=10.244.0.0/16 
  ### [This Pod CIDR is needed by Flannel.]
  ### Output: kubeadm join 10.192.214.204:6443 --token cd2u4c.ym9urw5rxull3v1c \
  ###    --discovery-token-ca-cert-hash \
  ### sha256:074971dc6dd1a0eb8dadd7cb76cffb312e502ca8d4be5fdec6e84c234594868a

> mkdir -p $HOME/.kube

> sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
> sudo chown $(id -u):$(id -g) $HOME/.kube/config

> kubectl get nodes
  NAME                                         STATUS     ROLES                  AGE   VERSION
  sc2-rdops-vm06-dhcp-214-204.eng.vmware.com   NotReady   control-plane,master   90s   v1.21.0

> kubectl get services
  NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
  kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   59s
```

### Installing flannel

```sh
> kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml
  
  Warning: policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
  podsecuritypolicy.policy/psp.flannel.unprivileged created
  clusterrole.rbac.authorization.k8s.io/flannel created
  clusterrolebinding.rbac.authorization.k8s.io/flannel created
  serviceaccount/flannel created
  configmap/kube-flannel-cfg created
  daemonset.apps/kube-flannel-ds created
```
* New `veth` interfaces are created.

```sh
> ip link show type veth
  6: vethd6318fa7@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master cni0 state UP mode DEFAULT group default
      link/ether e2:d2:22:ce:7d:fe brd ff:ff:ff:ff:ff:ff link-netnsid 0
  7: veth5600089d@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master cni0 state UP mode DEFAULT group default
      link/ether d6:2f:ae:f8:6a:a1 brd ff:ff:ff:ff:ff:ff link-netnsid 1
```
* A new CNI bridge is created, `docker0` is DOWN.

```sh
> ip link show type bridge
  3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
      link/ether 02:42:58:bf:b9:80 brd ff:ff:ff:ff:ff:ff
  5: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT group default qlen 1000
      link/ether 36:fa:37:27:e4:ec brd ff:ff:ff:ff:ff:ff
  
> bridge link show
  6: vethd6318fa7@docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 master cni0 state forwarding priority 32 cost 2
  7: veth5600089d@docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 master cni0 state forwarding priority 32 cost 2
```
* Check the new `cni0` and `flannel` interface on `master`. It got the `10.244.0.1/24` network, with flannel getting `10.244.0.0`

```sh
  cni0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
  	        inet 10.244.0.1  netmask 255.255.255.0  broadcast 10.244.0.255
  	        inet6 fe80::34fa:37ff:fe27:e4ec  prefixlen 64  scopeid 0x20<link>
  	        ether 36:fa:37:27:e4:ec  txqueuelen 1000  (Ethernet)
  	        RX packets 9385  bytes 759944 (742.1 KiB)
  	        RX errors 0  dropped 0  overruns 0  frame 0
  	        TX packets 9545  bytes 884905 (864.1 KiB)
  	        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
  	flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
  	        inet 10.244.0.0  netmask 255.255.255.255  broadcast 10.244.0.0
  	        inet6 fe80::84e8:1cff:fec9:1479  prefixlen 64  scopeid 0x20<link>
  	        ether 86:e8:1c:c9:14:79  txqueuelen 0  (Ethernet)
  	        RX packets 0  bytes 0 (0.0 B)
  	        RX errors 0  dropped 0  overruns 0  frame 0
  	        TX packets 0  bytes 0 (0.0 B)
  	        TX errors 0  dropped 13 overruns 0  carrier 0  collisions 0
```
* Flannel configurations reflects the above observation. `FLANNEL_NETWORK` is the CIDR for the whole flannel overlay network and `FLANNEL_SUBNET` is the subnet CIDR for the node.

```sh
> cat /run/flannel/subnet.env
  	FLANNEL_NETWORK=10.244.0.0/16
  	FLANNEL_SUBNET=10.244.0.1/24
  	FLANNEL_MTU=1450
  	FLANNEL_IPMASQ=true
```
* After flannel is installed, check the new routes on the nodes

```sh
> ip route
  	default via 10.192.223.253 dev eth0 proto dhcp metric 100
  	10.192.192.0/19 dev eth0 proto kernel scope link src 10.192.214.204 metric 100
  	10.244.0.0/24 dev cni0 proto kernel scope link src 10.244.0.1
  	172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
```
* Notice the new CNI bridge interface and ip route rule 3 which routes packets within the same node via the cni0 interface.
* [**Optional**] No workload runs on the master node in production. Remove this taint so that pods run on master as well.

```sh
kubectl taint nodes --all node-role.kubernetes.io/master-
```
* Add worker-1 to the kubernetes cluster.

```sh
kubeadm join 10.192.214.204:6443 --token cd2u4c.ym9urw5rxull3v1c \
      --discovery-token-ca-cert-hash \
      sha256:074971dc6dd1a0eb8dadd7cb76cffb312e502ca8d4be5fdec6e84c234594868a
      
  This node has joined the cluster:
  * Certificate signing request was sent to apiserver and a response was received.
  * The Kubelet was informed of the new secure connection details. 
```
* Confirm on `worker-1` that `kubectl` commands work.

```sh
[root@sc1-10-78-180-28 worker]# kubectl config get-contexts
  CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
  
  *         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin
  
[root@sc1-10-78-180-28 worker]# kubectl get nodes
  NAME                                         STATUS   ROLES                  AGE     VERSION
  sc1-10-78-180-28.eng.vmware.com             Ready    <none>                 2m15s   v1.21.0
  sc2-rdops-vm06-dhcp-214-204.eng.vmware.com   Ready    control-plane,master   18m     v1.21.0
```
* Check the new `cni0` and `flannel` interface on `worker-1`. It got the `10.244.2.1/24` network, with flannel getting `10.244.2.0`

```sh
  cni0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
          inet 10.244.2.1  netmask 255.255.255.0  broadcast 10.244.2.255
          inet6 fe80::f05e:80ff:febc:cffb  prefixlen 64  scopeid 0x20<link>
          ether f2:5e:80:bc:cf:fb  txqueuelen 1000  (Ethernet)
          RX packets 1  bytes 28 (28.0 B)
          RX errors 0  dropped 0  overruns 0  frame 0
          TX packets 6  bytes 536 (536.0 B)
          TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
  flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
          inet 10.244.2.0  netmask 255.255.255.255  broadcast 10.244.1.0
          inet6 fe80::e80b:2aff:fe19:8978  prefixlen 64  scopeid 0x20<link>
          ether ea:0b:2a:19:89:78  txqueuelen 0  (Ethernet)
          RX packets 0  bytes 0 (0.0 B)
          RX errors 0  dropped 0  overruns 0  frame 0
          TX packets 0  bytes 0 (0.0 B)
          TX errors 0  dropped 12 overruns 0  carrier 0  collisions 0
```
* Flannel configurations reflects the above observation. `FLANNEL_NETWORK` is the CIDR for the whole flannel overlay network and `FLANNEL_SUBNET` is the subnet CIDR for the node.

```sh
[root@sc1-10-78-180-28 worker]# cat /run/flannel/subnet.env
  FLANNEL_NETWORK=10.244.0.0/16
  FLANNEL_SUBNET=10.244.2.1/24
  FLANNEL_MTU=1450
  FLANNEL_IPMASQ=true
```
```sh
[root@sc1-10-78-180-28 ~]# ip route
  default via 10.78.191.254 dev eth0 proto dhcp metric 100
  10.78.176.0/20 dev eth0 proto kernel scope link src 10.78.180.28 metric 100
  10.244.0.0/24 via 10.244.0.0 dev flannel.1 onlink
  172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
```
* Install one more worker node:
	
```sh
	cni0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
	        inet 10.244.3.1  netmask 255.255.255.0  broadcast 10.244.3.255
	        inet6 fe80::548b:cff:fe91:6a1d  prefixlen 64  scopeid 0x20<link>
	        ether 56:8b:0c:91:6a:1d  txqueuelen 1000  (Ethernet)
	        RX packets 2  bytes 56 (56.0 B)
	        RX errors 0  dropped 0  overruns 0  frame 0
	        TX packets 7  bytes 626 (626.0 B)
	        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
	flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
	        inet 10.244.3.0  netmask 255.255.255.255  broadcast 10.244.3.0
	        inet6 fe80::1413:88ff:fec4:8b88  prefixlen 64  scopeid 0x20<link>
	        ether 16:13:88:c4:8b:88  txqueuelen 0  (Ethernet)
	        RX packets 0  bytes 0 (0.0 B)
	        RX errors 0  dropped 0  overruns 0  frame 0
	        TX packets 0  bytes 0 (0.0 B)
	        TX errors 0  dropped 10 overruns 0  carrier 0  collisions 0
```
* Install kuard pods

```sh
	NAME                               READY   STATUS    RESTARTS   AGE     IP           NODE                                       
	kuard-deployment-bc45ccb59-6p4p5   1/1     Running   0          7m42s   10.244.2.2   sc1-10-78-180-28.eng.vmware.com             
	kuard-deployment-bc45ccb59-pgsmc   1/1     Running   0          7m36s   10.244.3.3   sc-rdops-vm12-dhcp-119-138.eng.vmware.com   
	kuard-deployment-bc45ccb59-ztddc   1/1     Running   0          7m39s   10.244.3.2   sc-rdops-vm12-dhcp-119-138.eng.vmware.com   
```
* Pods are getting IP address from the `FLANNEL_SUBNET` of that particular node, that is `10.244.[0-3].1/24` , and flannel ip is `10.244.[0-3].0`
* Checking the `ip route` on master again to understand how the packets flow:

```sh
[root@sc2-rdops-vm06-dhcp-214-204 ~]# ip route
  default via 10.192.223.253 dev eth0 proto dhcp metric 100
  10.192.192.0/19 dev eth0 proto kernel scope link src 10.192.214.204 metric 100
  10.244.0.0/24 dev cni0 proto kernel scope link src 10.244.0.1
  10.244.1.0/24 via 10.244.1.0 dev flannel.1 onlink
  10.244.2.0/24 via 10.244.2.0 dev flannel.1 onlink
  10.244.3.0/24 via 10.244.3.0 dev flannel.1 onlink
  172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
```
* Ignore the `docker0` bridge. It is down
* Default gateway is the one the host has for internet. (`10.192.223.253`)
* Connections to `10.192.192.0/19` network go via `eth0`. This is the network the host/nodes are attached to. (**underlay network**)
* Connections to `10.244.0.0/24` (local Pods communications, ie the local flanel subnet) go via the `cni0` network.
* Same routes are present on worker. `cni0` for local pod network and `flannel.1` for other pod networks.
* Verifying the network routes on the `worker-2`, we see the same behaviour there as well (except the local flannel subnet `10.244.3.0/24` is different and is routed through `cni0`)

```sh
[root@sc-rdops-vm12-dhcp-119-138 worker]# ip route
	default via 10.161.127.253 dev eth0 proto dhcp metric 100
	10.161.96.0/19 dev eth0 proto kernel scope link src 10.161.119.138 metric 100
	10.244.0.0/24 via 10.244.0.0 dev flannel.1 onlink
	10.244.1.0/24 via 10.244.1.0 dev flannel.1 onlink
	10.244.2.0/24 via 10.244.2.0 dev flannel.1 onlink
	10.244.3.0/24 dev cni0 proto kernel scope link src 10.244.3.1
	172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
```
* The nodes have iptables rules to allow forward traffic if src or dst is the Pod CIDR range. CNI adds this iptable rule.

```sh
[root@sc2-rdops-vm06-dhcp-214-204 ~]# iptables -S FORWARD
  -P FORWARD DROP
  -A FORWARD -m comment --comment "kubernetes forwarding rules" -j KUBE-FORWARD
  -A FORWARD -m conntrack --ctstate NEW -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
  -A FORWARD -m conntrack --ctstate NEW -m comment --comment "kubernetes externally-visible service portals" -j KUBE-EXTERNAL-SERVICES
  -A FORWARD -s 10.244.0.0/16 -j ACCEPT
  -A FORWARD -d 10.244.0.0/16 -j ACCEPT
```
* The nodes have iptable rules to allow traffic NAT-ing when source/destination is not the Pod network

```sh
[root@sc2-rdops-vm06-dhcp-214-204 ~]# iptables -t nat -S POSTROUTING
  -P POSTROUTING ACCEPT
  -A POSTROUTING -m comment --comment "kubernetes postrouting rules" -j KUBE-POSTROUTING
  -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
  -A POSTROUTING -s 10.244.0.0/16 -d 10.244.0.0/16 -j RETURN
  -A POSTROUTING -s 10.244.0.0/16 ! -d 224.0.0.0/4 -j MASQUERADE --random-fully
  -A POSTROUTING ! -s 10.244.0.0/16 -d 10.244.0.0/24 -j RETURN
  -A POSTROUTING ! -s 10.244.0.0/16 -d 10.244.0.0/16 -j MASQUERADE --random-fully
```
* curl from one pod (`10.244.3.2`) to another pod (`10.244.2.2`) gets encapsulated by Host IP addresses.

```sh
  Internet Protocol Version 4, Src: 10.161.119.138, Dst: 10.78.180.28 
  User Datagram Protocol, Src Port: 57456, Dst Port: 8472
  Virtual eXtensible Local Area Network
    Flags: 0x0800, VXLAN Network ID (VNI)
  Internet Protocol Version 4, Src: 10.244.3.2, Dst: 10.244.2.2
  Transmission Control Protocol, Src Port: 59712, Dst Port: 8080, Seq: 0, Len: 0
```

#### Kubeadm reset cluster

```sh
kubeadm reset
etcdctl rm --recursive registry
rm -rf /var/lib/cni
rm -rf /run/flannel
rm -rf /etc/cni
ifconfig cni0 down
ip link delete cni0
ip link delete flannel.1
nmcli
```

### CNI
* CNI Binary: Handles connectivity - configures the network interfaces on the pods.
	
```sh
	[root@sc2-rdops-vm06-dhcp-214-204 ~]# ls /opt/cni/bin/
	bandwidth  bridge  dhcp  firewall  flannel  host-device  host-local  ipvlan  loopback  macvlan  portmap  ptp  sbr  static  tuning  vlan
```
* CNI Deamon: Handles reachability - configures routing across the cluster.