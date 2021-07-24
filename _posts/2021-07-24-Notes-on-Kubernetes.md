---
title: Notes on Kubernetes
created: '2021-01-30T04:24:30.203Z'
modified: '2021-05-08T21:11:47.531Z'
---

### Kubernetes 

* Kubernetes is a system for running many different types of containers over multiple different machines.
* K8s is a system of processes that run on multiple machines to provide master and worker architecture to deploy workloads.
* Master controls what each node does. Gets a config of how much work load of a particular image is needed, and creates them.
* Nodes can be VM or physical machine. The master creates workload containers on them.
* Nodes have a container runtime (like docker) running on them. The nodes use this container runtime to create containers.
* Master + Nodes is called a cluster.
* SUMMARY: 
	* You send a request to the cluster via the Kubernetes API, which is exposed by the Master.
	* The Master has information about the worker machines, or Nodes, in the cluster, and fulfills your request by communicating to the Nodes via the Kubelet. 
	* Node Kubelets communicate information back and forth with the Master to make sure your request is fulfilled and maintained across the cluster.

### Types of processes:
* **kube-apiserver**: Takes the yaml configs and creates the objects. 
	* Uses etcd as the distributed key-value database for configs.
	* etcd stores the actual state of the system and the desired state of the system.
	* Data presented in `kubectl get <xyz>` come from etcd.
	* Changes to config are saved in etcd/
	* When a pod goes down, that value is updated in the etcd.
* **kube-scheduler**: Connects to the etcd database using a "watch" which is a pub-sub model.
	* When a new pod has to be deployed, scheduler decides which replica of the node a pod will go in (load balance pod across nodes.)
* **kube-controller-manager**: Brain of the operation
	* Control process for actual workers like namespace-controller, deployment-controller, replicaset-controller.
	* Keeps a track of the current state and moves it towards the desired state. 
* **kubelet**: Lives on every single node and connect to the api server on the master node.
	* Talks to the container run time (eg docker) and does the ACTUAL work of creating a container in a pod.
	* Performs liveness probes.
* **kube-proxy**: Talks to the api server and creates the service objects.
* Notice how the first three are of docker-desktop type here. All the k8s processes are extensible, pluggable etc.

```sh
[rritesh-a02:rritesh:~] $ kubectl get pods --namespace=kube-system
	NAME                                     READY   STATUS    RESTARTS   AGE
	coredns-f9fd979d6-ktrdv                  1/1     Running   1          40h
	coredns-f9fd979d6-mvdnl                  1/1     Running   1          40h
	etcd-docker-desktop                      1/1     Running   1          40h
	kube-apiserver-docker-desktop            1/1     Running   1          40h
	kube-controller-manager-docker-desktop   1/1     Running   1          40h
	kube-proxy-4p74z                         1/1     Running   1          40h
	kube-scheduler-docker-desktop            1/1     Running   1          40h
	storage-provisioner                      1/1     Running   2          40h
	vpnkit-controller                        1/1     Running   1          40h

$  kubectl cluster-info
	Kubernetes master is running at https://kubernetes.docker.internal:6443
	KubeDNS is running at https://kubernetes.docker.internal:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

$ kubectl get nodes
	NAME             STATUS   ROLES    AGE   VERSION
	docker-desktop   Ready    master   41h   v1.19.3
```
* Load balancer is used to balance and route requests from external parties to the nodes.

```
		 ----> Node [Containers]
		|						    |		
Master---->   Node [Containers] <--- ---- LoadBalancer <---- Requests	
		|							|
		 ----> Node [Containers]    
```

### Minikube
* CLI used to set up k8s cluster locally. minikube is used to manage the node VMs.
* minikube driver can be set to docker or virtual box

```sh
$ minikube config set driver docker : Use local machine's docker runtime.
	
[rritesh@rritesh-a02:~] $ kubectl get nodes
		NAME       STATUS   ROLES                  AGE   VERSION
		minikube   Ready    control-plane,master   56s   v1.20.2

[rritesh@rritesh-a02:~] $ kubectl cluster-info
		Kubernetes master is running at https://127.0.0.1:55000
		KubeDNS is running at https://127.0.0.1:55000/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

		
$ minikube config set driver virtualbox : Use virtualbox to install a minikube vm to act as the node.
	
[rritesh@rritesh-a02:~] $ kubectl get nodes
		NAME       STATUS   ROLES                  AGE   VERSION
		minikube   Ready    control-plane,master   15m   v1.20.2

[rritesh@rritesh-a02:~] $ kubectl cluster-info
		Kubernetes master is running at https://192.168.99.100:8443
		KubeDNS is running at https://192.168.99.100:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

### kubectl and kubernetes objects
* Used for managing the containers in the node.
* Local Development:
	* Install kubectl: CLI for interacting with master
	* Install VM driver like virtual box:  Create VMs that will act as node.
	* Install minikube: Create a node on the VMs.

* Docker compose:
	* Each entry can optionally create an image.
	* Each entry points to a container we want to create.
	* Each entry defines the networking requirements.

* Kubernetes:
	* Expects all images to be prebuilt. (Doesn't have a build process. )
	* One config file per object we want to create.
	* Manually set up all networking. 

* kubectl uses 2 Yaml config files to create objects in a k8s cluster. 
* Different objects serve different purposes, eg running a container, monitoring a container, setting up networking etc.
* Objects types can be:
	* `StatefulSet`
	* `ReplicaController`
	* `Pod`
	* `Service`
	* `Namespace`
	* `Event`
	* `EndPoints`
	* `configMap`
	* `componentStatus`
	* `controllerRevision`
	... etc.
* These objects are running on the nodes.
* The objects on the node that runs one or more container(s) is called a POD.
* Unlike `docker-compose`, we don't create containers with kubectl, we create objects.
* The smallest thing kubectl can deploy is a Pod (ie an object with one or more container(s) in it).
* Multi container pods are used to group together and deploy containers that need to deploy together to work correctly. (Very tightly coupled and tightly integrated containers)
* Example: A pod has 3 containers: postgres, logger, backup-manager. All 3 need to deploy together.
* Service type object sets up networking in a k8s cluster.
* Services is an abstraction which defines a logical set of Pods and a policy by which to access them (sometimes this pattern is called a micro-service). 
* Subtypes:
	* `ClusterIP` -> Exposes the service on a cluster-internal IP. Choosing this value makes the service only reachable from within the cluster.
	* `NodePort` -> Expose the service to the outside world.
	* `LoadBalancer` -> Exposes the service externally using a cloud provider's load balancer. NodePort and ClusterIP Services, to which the
					  external load balancer routes, are automatically created.
	
* `Ingress` -> Ingress is not a Service type, but it acts as the entry point for your cluster. It lets you consolidate your routing rules into a single resource as it can expose multiple services under the same IP address.

### Node

`kube-proxy ----> Service NodePort ----> Pod[container]`
* Every node has a program called kube-proxy. This program is the one single window to the outside world for the containers in the Node.
* The request comes from outside world to kube-proxy, which makes the decision about which Service to route the request to.
* The Service object, if it is `NodePort` type, will forward the request to correct port of the container in the Pod object.

```sh
[rritesh-a02:rritesh:~/Study/Docker/simplek8s] $ cat client-node-port.yaml
	apiVersion: v1
	kind: Service
	metadata:
	  name: client-node-port
	spec:
	  type: NodePort
	  ports:
	    - port: 3050
	      targetPort: 3000
	      nodePort: 31515
	    selector:
	      component: web

[rritesh-a02:rritesh:~/Study/Docker/simplek8s] $ cat client-pod.yaml
	apiVersion: v1
	kind: Pod
	metadata:
	  name: client-pod
	  labels:
	    component: web
	spec:
	  containers:
	    - name: client
	      image: stephengrider/multi-client
	      ports:
	        - containerPort: 3000
```
* K8s uses a label selector system to decide which Pod the request goes to. The Pod's name is client-pod, with a label component called web. The Pod has a container called client inside it, with port `3000` opened for outside world (nginx service is running in the image provided on port `3000`).
* The Service is a `NodePort` type. When the service comes up, it sees it has to do port forwarding to target port `3000` of any other object that has the key value pair of `component:web` in it's metadata label (eg client-pod)
* Forwarding happens because `Service::spec::selector` matches the Pod::metadata::label, and the targetPort matches the containerPort
* Other PODs in the Node can also connect using port (3050).
* `nodePort` (31515) is the port that the outside world will use to connect to the container's port 3000.
* `nodePort` needs to be between 30000-32767.

### Deploying

```sh
kubectl apply -f client-pod.yaml
kubectl apply -f client-node-port.yaml
```
* One container created:

```sh
[rritesh-a02:rritesh:~/Study/Docker/simplek8s] $ docker ps
CONTAINER ID   IMAGE                        COMMAND                  CREATED         STATUS         PORTS     NAMES
96f11964e269   stephengrider/multi-client   "nginx -g 'daemon of…"   6 minutes ago   Up 6 minutes             k8s_client_client-pod_default_e7699f38-2a14-4785-a296-77fa0a9ba8ba_0
```
* kubectl get status

```sh
[rritesh-a02:rritesh:~/Study/Docker/simplek8s] $ kubectl get pods
NAME         READY   STATUS    RESTARTS   AGE
client-pod   1/1     Running   0          7m35s

[rritesh-a02:rritesh:~/Study/Docker/simplek8s] $ kubectl get services
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
client-node-port   NodePort    10.108.38.185   <none>        3050:31515/TCP   82s
kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP          78m
```
* The kubernetes clusterIP is internal to the k8s cluster. 
* If the node has been deployed using docker-for-desktop, localhost:31515 should be able to access the POD container functionality.
* If the container crashes or is killed, k8s will restart that container.

```sh
[rritesh-a02:rritesh:~] $ kubectl get pods
NAME         READY   STATUS    RESTARTS   AGE
client-pod   1/1     Running   1          8h

[rritesh-a02:rritesh:~] $ docker ps
CONTAINER ID   IMAGE                          COMMAND                  CREATED         STATUS                          PORTS     NAMES
04b461789a79   stephengrider/multi-client     "nginx -g 'daemon of…"   6 minutes ago   Up 6 minutes                              k8s_client_client-pod_default_e7699f38-2a14-4785-a296-77fa0a9ba8ba_1

[rritesh-a02:rritesh:~] $ docker kill 04b461789a79
04b461789a79

[rritesh-a02:rritesh:~] $ kubectl get pods
NAME         READY   STATUS    RESTARTS   AGE
client-pod   1/1     Running   2          8h

[rritesh-a02:rritesh:~] $ docker ps
CONTAINER ID   IMAGE                          COMMAND                  CREATED          STATUS                          PORTS     NAMES
aee80d2ee170   stephengrider/multi-client     "nginx -g 'daemon of…"   58 seconds ago   Up 57 seconds                             k8s_client_client-pod_default_e7699f38-2a14-4785-a296-77fa0a9ba8ba_2
```
* Kubernetes has a builtin DNS resolver.
```sh
[rritesh-a02:rritesh:~] $ kubectl get pods --namespace=kube-system
	NAME                                     READY   STATUS    RESTARTS   AGE
	coredns-f9fd979d6-ktrdv                  1/1     Running   1          40h
	coredns-f9fd979d6-mvdnl                  1/1     Running   1          40h
	etcd-docker-desktop                      1/1     Running   1          40h
	kube-apiserver-docker-desktop            1/1     Running   1          40h
	kube-controller-manager-docker-desktop   1/1     Running   1          40h
	kube-proxy-4p74z                         1/1     Running   1          40h
	kube-scheduler-docker-desktop            1/1     Running   1          40h
	storage-provisioner                      1/1     Running   2          40h
	vpnkit-controller                        1/1     Running   1          40h
```
* This DNS resolver helps services talk to each other using just service names.
* The service FQDNs are like `<service-name>.<namespace>.svc.cluster.local`
* Services in the same namespace and connect to each other via just `<service-name>`
* Services across namespaces can have the same name, so the <`service-name>.<namespace>` is needed to resolve say `nginx.dev.svc.cluster.local` and `nginx.prod.svc.cluster.local`
* kubectl describe gets the detailed information about an object.

```sh
$ kubectl describe pods client-pod
Name:         client-pod
Namespace:    default
Priority:     0
Node:         docker-desktop/192.168.65.3
Start Time:   Wed, 03 Feb 2021 01:45:13 +0530
Labels:       component=web
Annotations:  <none>
Status:       Running
IP:           10.1.0.21
IPs:
  IP:  10.1.0.21
Containers:
  client:
    Container ID:   docker://1855cdd1bd23645620accacadbc298715a540ba93fd469175e4be8de10be594c
    Image:          stephengrider/multi-client
    Image ID:       docker-pullable://stephengrider/multi-client@sha256:855452509d6d9f13dbe1cd34fa3a21d7f6e7d1f0fafb38d1e715dda8e3d17f46
    Port:           3000/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 22 Feb 2021 10:04:06 +0530
    Last State:     Terminated
      Reason:       Error
      Exit Code:    255
      Started:      Sun, 21 Feb 2021 13:05:14 +0530
      Finished:     Mon, 22 Feb 2021 10:03:47 +0530
    Ready:          True
    Restart Count:  4
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-9s569 (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-9s569:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-9s569
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:          <none>
```
* Deleting objects: kubectl delete using the config file. This is imperative approach, not declarative.

```sh
$ kubectl delete -f client-pod.yaml
	pod "client-pod" deleted
```
* kubectl apply can update only a small amount of configs of a Pod. eg you cannot change the name of containers, or the ports exposed.
* Deployment objects are used to ensure the correct number of pods (and with correct config) exists. Pods are directly not used in production generally. Production machines generally use Deployment.
* Deployment has a pod template configuration that it uses to deploy a set of IDENTICAL pods.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client-deployment
spec:
  replicas: 1				<---- How many pods do we need
  selector:					<---- How does Deployment lookup the Pods it manages after it creates them.
    matchLabels:
      component: web		<---- All pods matching this label are managed by this depolyment.
  template:  				<---- template section exactly like pod yaml file
    metdata:
      labels:
        component: web
    spec:
      containers:
	        * name: client
          image: stephengrider/multi-client
          ports:
	            * containerPort: 3000
```
* Changing deployment may delete and start a new Pod (if non changable fields of a pod are changed):

```sh
$ kubectl apply -f client-deployment.yaml
deployment.apps/client-deployment configured

$ kubectl get pods
NAME                                 READY   STATUS        RESTARTS   AGE
client-deployment-7cb6c958f7-wg5p5   1/1     Terminating   0          64s
client-deployment-8b5864968-pc2f8    1/1     Running       0          5s

$ kubectl get deployments
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
client-deployment   1/1     1            1           11s

$ kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES
client-deployment-8b5864968-pc2f8   1/1     Running   0          12m   10.1.0.23   docker-desktop   <none>           <none>
```

### Datastore-etcd
* etcd is the core state store for Kubernetes. While there are important in-memory caches throughout the system, etcd is considered the system of record.
* The highly consistent nature of etcd provides for strict ordering of writes and allows clients to do atomic updates of a set of values.
* The idea of watch in etcd is critical for how Kubernetes works. One component can write to etcd and other componenents can immediately react to that change.
* The common pattern is for clients to mirror a subset of the database in memory and then react to changes of that database.
* Watches are used as an efficient mechanism to keep that cache up to date.

### Policy Layer: API Server
* This is the only component in the system that talks to etcd.
* The API Server is a policy component that provides filtered access to etcd. 
* The API Server allows various components to create, read, write, update and watch for changes of resources.
* Also supports watches. A component could write something to API server resource (REST API), and the update is watched by other components.
* Responsibilities:
	* Authentication and Authorization: Kubernetes has a pluggable auth system. There are some built in mechanisms for both authentication users and authorizing those users to access resources.
	* Admission controllers: Reject/Modify requests to make sure only valid data is allowed in the system.

### Scheduler:
* Looks for pods that aren't assigned a node, examines the state of the cluster, finds a node with free space and binds the pod to that node.

### Controller Manager:
* Code that brings current state of the system to the desired state
* Implement the behavior of ReplicaSet. (ReplicaSet ensures that there are a set number of replicas of a Pod Template running at any one time)
* Controller will watch both the ReplicaSet resource and a set of Pods based on the selector in that resource.
* It then takes action to create/destroy Pods in order to maintain a stable set of Pods as described in the ReplicaSet.

### Kubelet:
* Agent that sits on the node.
* This also authenticates to the API Server like any other component.
* It is responsible for watching the set of Pods that are bound to its node and making sure those Pods are running.
* It then reports back status as things change with respect to those Pods.
* The basic flow:
	* The user creates a Pod via the API Server and the API server writes it to etcd.
    * The scheduler notices an “unbound” Pod and decides which node to run that Pod on. It writes that binding back to the API Server.
    * The Kubelet notices a change in the set of Pods that are bound to its node. It, in turn, runs the container via the container runtime (i.e. Docker).
    * The Kubelet monitors the status of the Pod via the container runtime. As things change, the Kubelet will reflect the current status back to the API Server.

### Volumes:
* In containers, volume is the mechanism to allow a container to access a filesystem outside itself.
* In kubernetes, volume is an Object that allows a container to store data at the Pod level. The kubernetes volume can be accessed by any container in the Pod.
* The volume is tied to the Pod. So if the Pod ever dies, the volume is lost as well. The Deployment would recreate the Pod, the volume would also be new.
* Persistent Volume is like a Volume but have a lifecycle independent of any individual Pod that uses the PV.
* Persisten Volume Claim is a request for storage by a user. It is similar to a Pod. Pods consume node resources and PVCs consume PV resources. Pods can request specific levels of resources (CPU and Memory). Claims can request specific size and access modes (e.g., they can be mounted ReadWriteOnce, ReadOnlyMany or ReadWriteMany) 
	* You, as cluster administrator, create a PersistentVolume backed by physical storage. You do not associate the volume with any Pod.
    * You, now taking the role of a developer / cluster user, create a PersistentVolumeClaim that is automatically bound to a suitable PersistentVolume.
    * You create a Pod that uses the above PersistentVolumeClaim for storage.

### StorageClass:
* A StorageClass provides a way for administrators to describe the "classes" of storage they offer.
* Different classes might map to quality-of-service levels, or to backup policies, or to arbitrary policies determined by the cluster administrators.
* Kubernetes itself is unopinionated about what classes represent. This concept is sometimes called "profiles" in other storage systems.
* The StorageClass can be found out by this command:

```sh
$ kubectl get storageclass
	NAME                 PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
	hostpath (default)   docker.io/hostpath   Delete          Immediate           false                  16d
```
* On different cloud providers, the default StorageClass will be their implementation. For example VMware has VsphereVolume that can provision from VSAN Datastores.

* Secret: `kubectl create secret generic pgpassword --from-literal PGPASSWORD=postgres`

* Nginx ingress controller installation
```sh
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/provider/cloud-generic.yaml
```

#### Deploy on Ubuntu cluster on virtual box 
* Ubuntu: setup networking

```sh
/etc/netplan/01-host-only.yaml
	network:
	  version: 2
	  renderer: networkd
	  ethernets:
	    enp0s8: # this is your interface name for your NAT network
	      dhcp4: no
	      addresses: [192.168.99.20/24]
	      gateway4: 192.168.99.1
	      nameservers:
	        addresses: [192.168.99.1, 8.8.8.8]

$ sudo netplan generate
$ sudo netplan apply
```
* Install Kubernetes
```sh
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
sudo apt-get install kubeadm kubelet kubectl

root@kubemaster:~# kubeadm config images pull
[config/images] Pulled k8s.gcr.io/kube-apiserver:v1.20.4
[config/images] Pulled k8s.gcr.io/kube-controller-manager:v1.20.4
[config/images] Pulled k8s.gcr.io/kube-scheduler:v1.20.4
[config/images] Pulled k8s.gcr.io/kube-proxy:v1.20.4
[config/images] Pulled k8s.gcr.io/pause:3.2
[config/images] Pulled k8s.gcr.io/etcd:3.4.13-0
[config/images] Pulled k8s.gcr.io/coredns:1.7.0
```
* To start using your cluster, you need to run the following as a regular user:

```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
* Alternatively, if you are the root user, you can run `export KUBECONFIG=/etc/kubernetes/admin.conf`
* After copy of config

```sh
	root@kubemaster:~# kubectl cluster-info
	Kubernetes control plane is running at https://192.168.99.20:6443
	KubeDNS is running at https://192.168.99.20:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```
