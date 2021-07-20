### References
* KubeAcademy - ClusterAPI: [course](https://kube.academy/courses/cluster-api)
* Joe Beda on ClusterAPI: [blog](https://tanzu.vmware.com/content/blog/cluster-api-is-a-big-deal-joe-beda-craig-mcluckie-tell-you-why)
* An elevated view of TKGs: [YouTube](https://www.youtube.com/watch?v=BCPU8rGDf_M)
* Micheal West's blog on TKC: [blog](https://blogs.vmware.com/vsphere/2020/06/vsphere-7-with-kubernetes-network-service-part-2-tanzu-kubernetes-cluster.html)
* Ben Corrie's 3 layer cake explaination: [README](https://github.com/corrieb/pacific/blob/master/docs/tkg-service/cluster-lifecycle/creation-basics.md)
* CAPI Provides Self Service Infrastructure: [blog](https://tanzu.vmware.com/content/blog/taking-kubernetes-to-the-people-how-cluster-api-promotes-self-service-infrastructure)
* Cluster API Provider for vSphere - Getting started: [CAPV](https://github.com/kubernetes-sigs/cluster-api-provider-vsphere/blob/master/docs/getting_started.md)
* Behind the Scenes with Cluster API Provider vSphere: [blog](https://neonmirrors.net/post/2020-04/capv-overview/)
* ClusterAPI Demystified: [blog](https://theithollow.com/2019/11/04/clusterapi-demystified/)
* Cluster API Provider for WCP: [CAPW](https://gitlab.eng.vmware.com/core-build/cluster-api-provider-wcp)
* CAPW Docs: [CAPW](https://gitlab.eng.vmware.com/core-build/cluster-api-provider-wcp/-/blob/master/docs/capi-v1alpha2-migration.md)
* Cluster API and Machine Health: [blog](https://orcunuso.wordpress.com/2021/04/18/cluster-api-and-machinehealthchecks/)

## Introduction

* **ClusterAPI** provides declarative lifecycle management of conformant Kubernets clusters both on-prem and in cloud, and enables the management of supporting infrastructure (VMs, network, load balancer, etc).
* ClusterAPI is a project of SIG Cluster Lifecycle. (Same group that created the kubeadm tool)
* ClusterAPI is a set of **CRDs** that allows kubernetes to know how to interface with the infrastructure provider.
* ClusterAPI also knows how an object of types **Machine** and **Cluster** represented.
* The kubernetes cluster into which the ClusterAPI components are installed is called the **Management cluster**.
* The Management cluster accepts manifests that declare how the workload cluster needs to be (eg. size of cluster, size of each node, overlay network types) and creates that workload cluster.
* Simply editing the cluster spec will modify the workload cluster.
* ClusterAPI uses **Providers** to interact with the underlying IaaS platforms. 
* ClusterAPI has many different providers available. One of them is **ClusterAPI Provider for Vsphere (CAPV)**, that allows ClusterAPI to know how to communicate to vSphere to deploy Virtual Machines based on templates.
* When a management cluster is deployed on vSphere using `clusterctl`or the `tkg-cli` tools, CAPV is used as the provider, and interfaces with CAPI, interacts with vSphere resources to realise the management cluster.
* When a management cluster is deployed on vSphere using WCP, **ClusterAPI Provider for WCP (CAPW)** is used instead of CAPV.
* There are different kinds of providers:
  * Infrastructure providers like CAPA, CAPV, CAPZ.
  * Bootstrap providers to support different ways to bootstrap a cluster or a node. eg CABPK.

