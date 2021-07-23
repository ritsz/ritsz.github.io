---
title: Notes-on-HAProxy
created: '2021-04-29T06:35:11.733Z'
modified: '2021-05-07T07:09:38.048Z'
---

* HAProxy is used as a loadbalancer for the APIServer on the CPVMs.
* HAProxy creation configuration example:

```json
{
  'network.nameservers': '10.170.16.48, 10.142.7.1', 
  'network.management_gateway': '10.191.191.254', 
  'appliance.root_pwd': 'vmware', 
  'network.hostname': 'haproxy.local', 
  'network.management_ip': '10.191.181.52/20', 
  'loadbalance.dataplane_port': '5556', 
  'network.workload_gateway': '192.168.1.1', 
  'appliance.ca_cert_key': '<snip>, 
  'network.frontend_ip': '172.16.10.2/24', 
  'appliance.ca_cert': '<snip>', 
  'network.workload_ip': '192.168.1.2/16', 
  'loadbalance.haproxy_user': 'wcp', 
  'loadbalance.haproxy_pwd': 'vmware', 
  'network.frontend_gateway': '172.16.10.1', 
  'appliance.permit_root_login': 'True', 
  'loadbalance.service_ip_range': '192.168.0.0/24'
}
```

### Network information
* Service IP range: `192.168.0.1 - 192.168.0.254`
* Nameservers: `10.170.16.48, 10.142.7.1`
* Mgmt gateway: `10.191.191.254`
* workload and frontend gateway: `192.168.1.1` and `172.16.10.1` (Both IP addresses of the **external-gateway VM** that gets deployed along with the HAProxy VM)
* SSH Username/Passwd : `root/vmware`.
* NOTE: **A client connecting to the kube-apiserver through the HAProxy load-balancer would be connecting through the workload network. (A dev user persona would be in the workload network)**
* NOTE: **Frontend network and workload network must need to route to each other for the successful wcp enablement.**

### HAProxy VM 
* The `netstat` output (ports under use) for the haproxy machine:

```sh
root@haproxy [ ~ ]# netstat -tnlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 192.168.0.1:443         0.0.0.0:*               LISTEN      5638/haproxy
tcp        0      0 192.168.0.1:6443        0.0.0.0:*               LISTEN      5638/haproxy
tcp        0      0 192.168.0.2:80          0.0.0.0:*               LISTEN      5638/haproxy
tcp        0      0 10.191.181.52:5556      0.0.0.0:*               LISTEN      1534/dataplaneapi
tcp        0      0 10.191.181.52:22        0.0.0.0:*               LISTEN      950/sshd
```
* haproxy processes:

```sh
5 S haproxy   5638  1085  0  80   0 - 25754 ep_pol 06:28 ?        00:00:00 /usr/sbin/haproxy -sf 5626 -x /run/haproxy.sock -Ws -f /etc/haproxy/haproxy.cfg -p /var/run/haproxy.pid -S /var/run/haproxy-master.sock
4 S root      1534     1  0  80   0 - 180616 futex_ Apr28 ?       00:01:24 /usr/local/bin/dataplaneapi --log-level=warning --log-to=stdout --scheme=https --haproxy-bin=/usr/sbin/haproxy --config-file=/etc/haproxy/haproxy.cfg --reload-cmd=/usr/bin/systemctl reload haproxy --reload-delay=30 --tls-host=10.191.181.52 --tls-port=5556 --tls-certificate=/etc/haproxy/server.crt --tls-key=/etc/haproxy/server.key --userlist=controller --update-map-files
```
* interface and route configuration

```
frontend  Link encap:Ethernet  HWaddr 00:50:56:b1:8a:41
          inet addr:172.16.10.2  Bcast:172.16.10.255  Mask:255.255.255.0
          inet6 addr: fe80::250:56ff:feb1:8a41/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:1090 errors:0 dropped:0 overruns:0 frame:0
          TX packets:570 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:81720 (81.7 KB)  TX bytes:50736 (50.7 KB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:1620 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1620 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:118238 (118.2 KB)  TX bytes:118238 (118.2 KB)

management Link encap:Ethernet  HWaddr 00:50:56:b1:20:5e
          inet addr:10.191.181.52  Bcast:10.191.191.255  Mask:255.255.240.0
          inet6 addr: fd01:3:7:404:250:56ff:feb1:205e/64 Scope:Global
          inet6 addr: fe80::250:56ff:feb1:205e/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:4201452 errors:0 dropped:2161 overruns:0 frame:0
          TX packets:49082 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:336892980 (336.8 MB)  TX bytes:12356489 (12.3 MB)

workload  Link encap:Ethernet  HWaddr 00:50:56:b1:8d:33
          inet addr:192.168.1.2  Bcast:192.168.255.255  Mask:255.255.0.0
          inet6 addr: fe80::250:56ff:feb1:8d33/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:945788 errors:0 dropped:69 overruns:0 frame:0
          TX packets:464312 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:1073535193 (1.0 GB)  TX bytes:1041890581 (1.0 GB)

root@haproxy [ ~ ]# ip r
default via 10.191.191.254 dev management proto static
10.191.176.0/20 dev management proto kernel scope link src 10.191.181.52
172.16.10.0/24 dev frontend proto kernel scope link src 172.16.10.2
192.168.0.0/16 dev workload proto kernel scope link src 192.168.1.2
```
* `ping` to `www.google.com` goes via `_gateway` which is the `192.168.1.1` ip address. www.google.com is pinged via 'extrenal-gateway' VM's workload IP address.:

```sh
root@haproxy [ ~ ]# tracepath -4 www.google.com
 1?: [LOCALHOST]                      pmtu 1500
 1:  _gateway                                              0.608ms asymm 64
 1:  _gateway                                              0.358ms asymm 64
 ...<snip>...

root@haproxy [ ~ ]# ping _gateway
PING _gateway (192.168.1.1) 56(84) bytes of data.
64 bytes from _gateway (192.168.1.1): icmp_seq=1 ttl=64 time=0.461 ms
64 bytes from _gateway (192.168.1.1): icmp_seq=2 ttl=64 time=0.422 ms
```

### HAProxy configuration
* The configuration file has configurations about 'frontends' and 'backends'.
* Format of the configuration file:

```conf
global
    # global settings here

defaults
    # defaults here

frontend
    # a frontend that accepts requests from clients

backend
    # servers that fulfill the requests
```

#### Backend
* A backend is a set of servers that receives forwarded requests. Backends are defined in the backend section of the HAProxy configuration. In its most basic form, a backend can be defined by:
	* which load balance algorithm to use
  * a list of servers and ports.
* A backend can contain one or many servers in it.

```conf
backend domain-c50:b42f9558-a4eb-4e8f-8f0f-9a8f295f0a8e-kube-system-kube-apiserver-lb-svc-kube-apiserver
  mode tcp
  balance roundrobin
  option tcp-check
  log-tag domain-c50:b42f9558-a4eb-4e8f-8f0f-9a8f295f0a8e-kube-system-kube-apiserver-lb-svc-kube-apiserver
  server domain-c50:b42f9558-a4eb-4e8f-8f0f-9a8f295f0a8e-kube-system-kube-apiserver-lb-svc-192.168.128.0:6443 192.168.128.0:6443 check weight 100 verify none
  server domain-c50:b42f9558-a4eb-4e8f-8f0f-9a8f295f0a8e-kube-system-kube-apiserver-lb-svc-192.168.128.1:6443 192.168.128.1:6443 check weight 100 verify none
  server domain-c50:b42f9558-a4eb-4e8f-8f0f-9a8f295f0a8e-kube-system-kube-apiserver-lb-svc-192.168.128.2:6443 192.168.128.2:6443 check weight 100 verify none
```
* `192.168.128.[0-2]` are the IP address of CPVM on the workload network. **The above backend configuration refers to the kube-apiserver load balancing.**
* A client connecting to the kube-apiserver through the HAProxy load-balancer (VIP) will be load-balanced to either of the three VMs' 6443 port number.
* The balancing happens roundrobin. Mode TCP means this is a layer 4 load balancer. (Another type of load balancer is the layer 7 load balancer, ie Mode http, where the load balancing can happen based on the content of the user's requests.)

#### Frontend
* A frontend defines how requests should be forwarded to backends.

```conf
frontend domain-c50:b42f9558-a4eb-4e8f-8f0f-9a8f295f0a8e-kube-system-kube-apiserver-lb-svc
  mode tcp
  bind 192.168.0.1:443 name domain-c50:b42f9558-a4eb-4e8f-8f0f-9a8f295f0a8e-kube-system-kube-apiserver-lb-svc-192.168.0.1:nginx
  bind 192.168.0.1:6443 name domain-c50:b42f9558-a4eb-4e8f-8f0f-9a8f295f0a8e-kube-system-kube-apiserver-lb-svc-192.168.0.1:kube-apiserver
  log-tag domain-c50:b42f9558-a4eb-4e8f-8f0f-9a8f295f0a8e-kube-system-kube-apiserver-lb-svc
  option tcplog
  use_backend domain-c50:b42f9558-a4eb-4e8f-8f0f-9a8f295f0a8e-kube-system-kube-apiserver-lb-svc-kube-apiserver if { dst_port 6443 }
  use_backend domain-c50:b42f9558-a4eb-4e8f-8f0f-9a8f295f0a8e-kube-system-kube-apiserver-lb-svc-nginx if { dst_port 443 }
```
* A bind setting assigns a listener to a given IP address and port.
* The use_backend setting chooses a backend pool of servers to respond to incoming requests if a given condition is true. It is followed by an ACL statement, such as if path_beg `/api/`, that allows HAProxy to select a specific backend based on some criteria. Here we can see that if Port is 6443, kube-apiserver backend is used, while if port is 443, the ngnix backend is used.
* In both the cases, the frontend IP address is `192.168.0.1`.
* A client on the workload network connecting to `192.168.0.1` would be load-balanced to one of the CPVM on the backend. If client is connecting to `443`, load-balancer redirects to the `443` port on the CPVM, or to port `6443` if the client is connecting to the kube-apiserver.

#### Configuration changes
* A new frontend+backend pair is added for each tanzu guest cluster:
* Frontend at `192.168.0.3:6443`, and backend at `172.16.10.3:6443` for the guest cluster API server.

```conf
frontend domain-c50:b42f9558-a4eb-4e8f-8f0f-9a8f295f0a8e-vm-testing-rritesh-guest-cluster-control-plane-service
  mode tcp
  bind 192.168.0.3:6443 name domain-c50:b42f9558-a4eb-4e8f-8f0f-9a8f295f0a8e-vm-testing-rritesh-guest-cluster-control-plane-service-192.168.0.3:apiserver
  log-tag domain-c50:b42f9558-a4eb-4e8f-8f0f-9a8f295f0a8e-vm-testing-rritesh-guest-cluster-control-plane-service
  option tcplog
  use_backend domain-c50:b42f9558-a4eb-4e8f-8f0f-9a8f295f0a8e-vm-testing-rritesh-guest-cluster-control-plane-service-apiserver if { dst_port 6443 }

backend domain-c50:b42f9558-a4eb-4e8f-8f0f-9a8f295f0a8e-vm-testing-rritesh-guest-cluster-control-plane-service-apiserver
  mode tcp
  balance roundrobin
  option tcp-check
  log-tag domain-c50:b42f9558-a4eb-4e8f-8f0f-9a8f295f0a8e-vm-testing-rritesh-guest-cluster-control-plane-service-apiserver
  server domain-c50:b42f9558-a4eb-4e8f-8f0f-9a8f295f0a8e-vm-testing-rritesh-guest-cluster-control-plane-service-172.16.10.3:6443 172.16.10.3:6443 check weight 100 verify none
```
