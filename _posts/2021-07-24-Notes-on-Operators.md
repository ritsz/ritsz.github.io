---
title: Notes on Operators
created: '2021-05-06T16:15:45.411Z'
modified: '2021-05-08T20:43:45.977Z'
---

#### Table of contents

    * [Reference](#reference)
    * [Introduction](#introduction)
    * [Controllers](#controllers)
    * [Creating controllers using kubebuilder](#creating-controllers-using-kubebuilder)
    * [Kube-rbac-proxy](#kube-rbac-proxy)
    * [Webhooks](#webhooks)
    * [Kustomization](#kustomization)
    * [TODO](#todo)

### Reference
* Let's build a Kubernetes Operator in Go : [vmware{Code}@YouTube](https://www.youtube.com/watch?v=8Ex7ybi273g)
* Build a simple vm operator using kubebuilder: [codeconnect-vm-operator](https://github.com/embano1/codeconnect-vm-operator/blob/main/README.md)
* Zero to Operator in 90 mins, Ross Solly: [CNCF@YouTube](https://www.youtube.com/watch?v=KBTXBUVNF2I)
* How controllers are implemented under the covers: [cloudark@medium](https://cloudark.medium.com/kubernetes-custom-controllers-b6c7d0668fdf)

### Introduction

* Operator = Custom Resource Definition + custom controller.
* An **operator** is an application specific *controller* that extends the Kubernetes API to create, configure, and manage instances of applications on behalf of a kubernetes user.
* Say your App needs 4 Deployments, 1 Lbsvc, service accounts, PVC etc. Managing each of them as a separate yaml file will be difficult.
* We create a custom resource (say MyApp) that reads a single yaml file and the operator creates the necessary resources for us.
* Levels of operator
    1. Allows basic install.
    2. Allows upgrades.
    3. Allows full lifecycle support. (allows backup/failure recovery)
    4. Allows Insights. (metrics and analysis)
    5. Autopilot. (Automatic horizontal/vertical scaling, config tuning)

### Controllers
* The logic runs in a pod. A control loop logic that watches a resource and tries to reconcile the desired state to the actual state.
* Single responsible principle: One controller should watch and manage one resource.
* Decoupling via event-driven messaging.
* Eventual consistent by design. Cannot assume or rely on order.
* API Server (etcd) is the source of truth. Keep the controller stateless.

### Creating controllers using kubebuilder
* create a directory and `go mod init`

```sh
> mkdir myoperator; cd myoperator
> go mod init myoperator
> 
> more go.mod                                                                                                             
module myoperator

go 1.16
```
* "Scaffolding" is the first step of building an operator which kubebuilder will initialize the operator from a brand-new directory. The first step is initialization using `kubebuilder init`.

```sh
> kubebuilder init --domain codeconnect.vmworld.com
Writing scaffold for you to edit...
Get controller runtime:
$ go get sigs.k8s.io/controller-runtime@v0.5.0
Update go.mod:
$ go mod tidy
Running make:
$ make
/Users/rritesh/go/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
go fmt ./...
go vet ./...
go build -o bin/manager main.go
Next: define a resource with:
$ kubebuilder create api

❯ ls                                                                                                                            
Dockerfile Makefile   PROJECT    bin        config     go.mod     go.sum     hack       main.go

❯ ls bin                                                                                                                        
manager

❯ ls hack                                                                                                                       
boilerplate.go.txt
```
* `kubebuilder create api` creates the api and controllers boilerplate code.

```sh
❯ kubebuilder create api --group vm --version v1alpha1 --kind VmGroup                                                         
Create Resource [y/n]
y
Create Controller [y/n]
y
Writing scaffold for you to edit...
api/v1alpha1/vmgroup_types.go
controllers/vmgroup_controller.go
Running make:
$ make
/Users/rritesh/go/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
go fmt ./...
go vet ./...
go build -o bin/manager main.go

❯ ls                                                                                                                
Dockerfile  Makefile    PROJECT     api         bin         config      controllers go.mod      go.sum      hack        main.go

❯ ls bin                                                                                                                      
manager

❯ ls config                                                                                                                    
certmanager crd         default     manager     prometheus  rbac        samples     webhook

❯ ls controllers                                                                                                               
suite_test.go         vmgroup_controller.go
```
* The `api/v1alpha1` folder has `vmgroup_types.go` file.
* K8s objects have at least three sections: `metadata`, `spec` and `status` besides the `kind` and `apiVersion`.
* `Spec` section is where you declare the desired state of the object.
* `Status` section is the current state of the object. 
* The operator's job is reconciling these two sections in a loop.
* `kubebuilder` markers can be used for validations on the spec.

```go
  // VmGroupSpec defines the desired state of VmGroup
  type VmGroupSpec struct {
      // +kubebuilder:validation:Required
      // +kubebuilder:validation:Minimum=1
      // +kubebuilder:validation:Maximum=4
      CPU int32 `json:"cpu"`
      // +kubebuilder:validation:Required
      // +kubebuilder:validation:Minimum=1
      // +kubebuilder:validation:Maximum=8
      Memory int32 `json:"memory"`
      // +kubebuilder:validation:Required
      Template string `json:"template"`
      // +kubebuilder:validation:Required
      // +kubebuilder:validation:Minimum=1
      Replicas int32 `json:"replicas"`
  }

  // VmGroupStatus defines the observed state of VmGroup
  type VmGroupStatus struct {
      // +kubebuilder:validation:Optional
      Phase           StatusPhase `json:"phase"`
      CurrentReplicas *int32      `json:"currentReplicas,omitempty"`
      DesiredReplicas int32       `json:"desiredReplicas"`
      LastMessage     string      `json:"lastMessage"`
  }
```
* `kubebuilder` specifies `printcolumn` markers that defines what is printed when `kubectl get` is done on the resources. This defines the column name and the value from the spec or status.

```go
  // +kubebuilder:object:root=true
  // +kubebuilder:validation:Optional
  // +kubebuilder:subresource:status
  // +kubebuilder:subresource:scale:specpath=.spec.replicas,statuspath=.status.desiredReplicas
  // +kubebuilder:resource:shortName={"vg"}
  // +kubebuilder:printcolumn:name="Phase",type=string,JSONPath=`.status.phase`
  // +kubebuilder:printcolumn:name="Current",type=integer,JSONPath=`.status.currentReplicas`
  // +kubebuilder:printcolumn:name="Desired",type=integer,JSONPath=`.spec.replicas`
  // +kubebuilder:printcolumn:name="CPU",type=integer,JSONPath=`.spec.cpu`
  // +kubebuilder:printcolumn:name="Memory",type=integer,JSONPath=`.spec.memory`
  // +kubebuilder:printcolumn:name="Template",type=string,JSONPath=`.spec.template`
  // +kubebuilder:printcolumn:name="Last_Message",type=string,JSONPath=`.status.lastMessage
```
* The controller business logic goes into `controller/vmgroup_controller.go`
* The controller business logic has:
    * `VmGroupReconciler` that has embedded type `client.Client`. `Client` knows how to perform CRUD operations on Kubernetes objects. Since `Client` is an embedded type, we can either do `r.Client.Get` or just `r.Get`.
    * `Reconcile` which has the business logic of reconciling the current and the desired state.
    * `SetupWithManager` which sets up the control loop with the Manager process. It declares that it is watching on `VmGroup` using `NewControllerManagedBy`. `NewControllerManagedBy` also accepts filters about what events on the watched resource it is interested in, eg `event.CreateEvent`, `event.UpdateEvent`, `event.DeleteEvent`, or `event.GenericEvent`.

```go
  type VmGroupReconciler struct {
      client.Client
      Log    logr.Logger
      Scheme *runtime.Scheme
  }

  func (r *VmGroupReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
      ctx := context.Background()
      _ = r.Log.WithValues("vmgroup", req.NamespacedName)

      vg := &vmv1alpha1.VmGroup{}
      _ = r.Get(ctx, req.NamespacedName, vg) \\ r.Client.Get
  }

  func (r *VmGroupReconciler) SetupWithManager(mgr ctrl.Manager) error {
      return ctrl.NewControllerManagedBy(mgr).
          For(&vmv1alpha1.VmGroup{}).
          Complete(r)
  }
```
* The `kubebuilder` also creates the yaml file for the CRD. This is generated using `make manifests` in the `Makefile` generated by `kubebuilder`. The CRD yaml file is generated independent of the controller logic implementation. Update `api/v1alpha1/vmgroup_types.go` and run `make manifests` to update the CRD manifest yaml files.

```sh
  > make manifests                                                                                  
  /Users/rritesh/go/bin/controller-gen "crd:preserveUnknownFields=false,crdVersions=v1,trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases

  ❯ head -n 20 config/crd/bases/vm.codeconnect.vmworld.com_vmgroups.yaml                          

  apiVersion: apiextensions.k8s.io/v1
  kind: CustomResourceDefinition
  metadata:
    annotations:
      controller-gen.kubebuilder.io/version: v0.2.4
    creationTimestamp: null
    name: vmgroups.vm.codeconnect.vmworld.com
  spec:
    group: vm.codeconnect.vmworld.com
    names:
      kind: VmGroup
      listKind: VmGroupList
      plural: vmgroups
      shortNames:
      - vg
      singular: vmgroup
    scope: Namespaced
    versions:
    - additionalPrinterColumns:
      - jsonPath: .status.phase
        name: Phase
        type: string
      - jsonPath: .status.currentReplicas
        name: Current
        type: integer
      - jsonPath: .spec.replicas
        name: Desired
        type: integer
```
* `kubebuilder` also has markers that can are used to create RBACs.

```go
  // +kubebuilder:rbac:groups=vm.codeconnect.vmworld.com,resources=vmgroups,verbs=get;list;watch;create;update;patch;delete
  // +kubebuilder:rbac:groups=vm.codeconnect.vmworld.com,resources=vmgroups/status,verbs=get;update;patch

  func (r *VmGroupReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
```
* The above `kubebuilder` markers generate RBAC yaml like these. This creates a ClusterRole called `manager-role` and binds it to the default service account.

```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    creationTimestamp: null
    name: manager-role
  rules:
  - apiGroups:
    - vm.codeconnect.vmworld.com
    resources:
    - vmgroups
    verbs:
    - create
    - delete
    - get
    - list
    - patch
    - update
    - watch
  - apiGroups:
    - vm.codeconnect.vmworld.com
    resources:
    - vmgroups/status
    verbs:
    - get
    - patch
    - update
---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: manager-rolebinding
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: manager-role
  subjects:
  - kind: ServiceAccount
    name: default
    namespace: system
```
* Controllers always do defensive programming. Check if the object is marked for deletion. A controller uses custom finalizers to mark objects that need some special handling by it. The controller will add a finalizer string to the object when it is getting created. When the same object is getting deleted, the finalizer strings is checked to see if controller needs to perform some special cleanups.

```go
      // is object marked for deletion?
      if !vg.ObjectMeta.DeletionTimestamp.IsZero() {
          log.Info("VmGroup marked for deletion")
          // The object is being deleted
          if containsString(vg.ObjectMeta.Finalizers, finalizerID) {
              // our finalizer is present, so lets handle any external dependency
              if err := r.deleteExternalResources(ctx, r.Finder, vg); err != nil {
                  // if fail to delete the external dependency here, return with error
                  // so that it can be retried
                  return ctrl.Result{}, err
              }

              // remove our finalizer from the list and update it.
              vg.ObjectMeta.Finalizers = removeString(vg.ObjectMeta.Finalizers, finalizerID)
              if err := r.Update(ctx, vg); err != nil {
                  return ctrl.Result{}, errors.Wrap(err, "could not remove finalizer")
              }
          }
          // finalizer already removed, nothing to do
          return ctrl.Result{}, nil
      }

      // register our finalizer if it does not exist
      if !containsString(vg.ObjectMeta.Finalizers, finalizerID) {
          vg.ObjectMeta.Finalizers = append(vg.ObjectMeta.Finalizers, finalizerID)
          if err := r.Update(ctx, vg); err != nil {
              return ctrl.Result{}, errors.Wrap(err, "could not add finalizer")
          }
      }
```
* Once the `manager` binary is generated using the `api`, `controllers` and `main.go`, a container image is created.

```dockerfile
# Build the manager binary
FROM golang:1.13 as builder

WORKDIR /workspace
# Copy the Go Modules manifests
COPY go.mod go.mod
COPY go.sum go.sum
# cache deps before building and copying source so that we don't need to re-download as much
# and so that source changes don't invalidate our downloaded layer
RUN go mod download

# Copy the go source
COPY main.go main.go
COPY api/ api/
COPY controllers/ controllers/

# Build
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 GO111MODULE=on go build -a -o manager main.go

# Use distroless as minimal base image to package the manager binary
# Refer to https://github.com/GoogleContainerTools/distroless for more details
FROM gcr.io/distroless/static:nonroot
WORKDIR /
COPY --from=builder /workspace/manager .
USER nonroot:nonroot

ENTRYPOINT ["/manager"]
```
* Build the container image using docker.

```sh
docker build Dockerfile -t controller:latest
```
* Load the container image into the kind cluster.

```sh
❯ kubectl config get-contexts                                                                               
CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
          docker-desktop   docker-desktop   docker-desktop
*         kind-kind        kind-kind        kind-kind        default

❯ kind load docker-image controller:latest                                                                     
Image: "controller:latest" with ID "sha256:e853646b77f4864d4b0e7c82137d87a114d50ff3dfcf2a704ecfed66884f4afc" not yet present on node "kind-worker", loading...
Image: "controller:latest" with ID "sha256:e853646b77f4864d4b0e7c82137d87a114d50ff3dfcf2a704ecfed66884f4afc" not yet present on node "kind-control-plane", loading...
Image: "controller:latest" with ID "sha256:e853646b77f4864d4b0e7c82137d87a114d50ff3dfcf2a704ecfed66884f4afc" not yet present on node "kind-worker2", loading...
```
* Create the secrets/config maps that the controller will use:

```sh
❯ kubectl create ns codeconnect-vm-operator-system                                                         
namespace/codeconnect-vm-operator-system created

 ❯ kubectl -n codeconnect-vm-operator-system create secret generic vc-creds --from-literal='VC_USER=Administrator@vsphere.local' --from-literal='VC_PASS=Admin!23' --from-literal='VC_HOST=10.170.100.72'
secret/vc-creds created
```
* `Kustomize` the operator yaml files and apply to `kind` cluster.

```sh
❯ kustomize build config/default | kubectl apply -f -                             
namespace/codeconnect-vm-operator-system created
customresourcedefinition.apiextensions.k8s.io/vmgroups.vm.codeconnect.vmworld.com configured
role.rbac.authorization.k8s.io/codeconnect-vm-operator-leader-election-role created
clusterrole.rbac.authorization.k8s.io/codeconnect-vm-operator-manager-role created
clusterrole.rbac.authorization.k8s.io/codeconnect-vm-operator-proxy-role created
Warning: rbac.authorization.k8s.io/v1beta1 ClusterRole is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 ClusterRole
clusterrole.rbac.authorization.k8s.io/codeconnect-vm-operator-metrics-reader created
rolebinding.rbac.authorization.k8s.io/codeconnect-vm-operator-leader-election-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/codeconnect-vm-operator-manager-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/codeconnect-vm-operator-proxy-rolebinding created
service/codeconnect-vm-operator-controller-manager-metrics-service created
deployment.apps/codeconnect-vm-operator-controller-manager created
```
* To deploy this image to a kubernetes cluster that is not `kind` or `docker on mac`, for example a kubernetes cluster of nimbus VMs, push the images to docker hub, and change the operator yaml file to point to docker hub image. When the `operator.yaml` is applied, all the nodes on which the pods get created will pull the image `docker.io/ritsz/controller:latest` to their local docker image cache.

```sh
❯ docker tag 2d907535c5a0 ritsz/controller:latest
❯ docker push ritsz/controller
❯ kustomize build config/default > operator.yaml
❯ cat operator.yaml
  ...<snip>...
    spec:
      containers:
      - args:
        - --secure-listen-address=0.0.0.0:8443
        - --upstream=http://127.0.0.1:8080/
        - --logtostderr=true
        - --v=10
        image: gcr.io/kubebuilder/kube-rbac-proxy:v0.5.0
        name: kube-rbac-proxy
        ports:
        - containerPort: 8443
          name: https
      - args:
        - --metrics-addr=127.0.0.1:8080
        - --enable-leader-election
        - -insecure
        command:
        - /manager
        env:
        - name: VC_USER
          valueFrom:
            secretKeyRef:
              key: VC_USER
              name: vc-creds
        - name: VC_PASS
          valueFrom:
            secretKeyRef:
              key: VC_PASS
              name: vc-creds
        - name: VC_HOST
          valueFrom:
            secretKeyRef:
              key: VC_HOST
              name: vc-creds
        image: ritsz/controller:latest
        name: manager
```
* On one of the nodes, use `crictl` to check the cached images

```sh
[root@sc2-rdops-vm06-dhcp-214-204 codeconnect-vm-operator]$ crictl images | grep ritsz
ritsz/controller                     latest              2d907535c5a0b       60MB
```

### Kube-rbac-proxy
* https://brancz.com/2018/02/27/using-kube-rbac-proxy-to-secure-kubernetes-workloads/

### Webhooks
* 

### Kustomization

### TODO
* Zero to Operator in 90 mins, Ross Solly: [CNCF@YouTube](https://www.youtube.com/watch?v=KBTXBUVNF2I)
* How controllers are implemented under the covers: [cloudark@medium](https://cloudark.medium.com/kubernetes-custom-controllers-b6c7d0668fdf)
* Writing kube controllers for everyone: https://www.youtube.com/watch?v=AUNPLQVxvmw 
* Writing kubernetes controllers for CRDs: https://www.youtube.com/watch?v=7wdUa4Ulwxg
* ElasticSearch: Writing kubernetes operators: the hard parts: https://www.youtube.com/watch?v=wMqzAOp15wo
* How we built contour  (Ingress Controller): https://www.youtube.com/watch?v=4usXJE0EwHo
* Controllers: Lambda function for infrastructure: https://www.youtube.com/watch?v=TM-2GgQ6Q2A