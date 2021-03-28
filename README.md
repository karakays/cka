# Certified Kubernetes Admninistrator

![cka-logo](https://training.linuxfoundation.org/wp-content/uploads/2020/12/kubernetes-cka-color.svg)

## Kubernetes components

|name| task
|---|---
|api-server| Router for outbound (from kubectl) and inbound requests (from kubelet)
|controller| Monitors workers
|etcd| Distributed datastore
|scheduler| Assigns tasks to workers
|container runtime | Provides runtime for pods e.g. docker, CRI-O or container
|kubelet| Node agent that talks to api-server

## Node roles

Which nodes have what components?

|node| components
|---|---
|master| api-server, controller, etcd, scheduler
|worker| container runtime, kubelet

Q>>> control-plane vs. master node  
Q>>> kubectl talks to api-server only?  
Q>>> kubectl connection refused when in worker nodes because there is no api-server? 


### `kubectl`

`kubectl` can talk to multiple clusters. Cluster configuration such as api-server address, authentication details (password-based or mTLS) etc. are hold in `.kube/config`.

Get configured clusters

`kubectl config get-contexts`

Get current context

`kubectl config current-context`

Switch cluster context. Any command from then on will go to bar cluster. 

`kubectl config use-context bar`

## k8s distributions

### `minikube`

`minikube` bundles k8s components into a single image and provides a single-node k8s cluster.

`minikube start`

#### Drivers

`minikube` can be deployed in various ways
* in a container as part of existing docker environment
* in a VM like VirtualBox
* or bare-metal

`minikube config get driver`

It is important to know that minikube comes with a container runtime for itself. This runtime is different than what is instaled by docker-desktop. If minikube driver is docker, this can cause
confusion with existing `docker` clients for instance when building a docker image with `docker`.Trying to consume such
image in minikube pods will cause error as minikube has a different local registry that docker. Solve this problem by running `eval $(minikube docker-env)` adjusts the environment so that docker client points to minikube's container runtime.

As an alternative to `minikube`, docker-desktop with version 18.06 started to include k8s server and client (which can replace minikube)

### As a multi-node cluster

`kubeadm`

#### Basic cluster setup

Initialize master node

```
kubeadm init --apiserver-advertise-address $(hostname -i) --pod-network-cidr 10.5.0.0/16
```

Initialize POD network

```
kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter.yaml
```

Echo join-command for worker nodes to use to join the cluster

```
kubeadm token create --print-join-command
```

### As a managed service

Clusters is provisioned and managed by the provider.

* EKS: Elastic Kubernetes Service
    - prepare context
      `aws eks update-kubeconfig --name bar-cluster`

* GKE: Google Kubernetes Engine

* AKS: Azure Kubernetes Service

### Resource management with `kubectl`

#### Create

Create resource from file in _non-idempotent_ way

`kubectl create -f resource.yml`

Create or update resource from file in _idempotent_ way

`kubectl create -f resource.yml`

#### Get

Get overview of resource

`kubectl get [resource]`

Get multiple resources at once

`kubectl get pods,svc`

Get details of resource

`kubectl describe [resource] `

#### Update

`kubectl apply -f resource.yml`

`kubectl edit [resource] [name]`

`kubectl set image [resource] [name] nginx=nginx:1.17`

#### Delete

Delete resource  

`kubectl delete [resource] [name]`


#### Create pods

Create a deployment from image and runs it
`kubectl run [name] --image=[image]`

Create a service for existing pod ???? Check
`kubectl expose pod foo --name=bar --port=80 --target-port=8000`

Create a service for deployment????? Check
`kubectl expose deployment foo --name=myservice --port=80 --tarrget-port=8000`

## Resources

### Pods

It represents smallest deployable unit. It is a logical group of containers which together form a unit of service by sharing same resources.

`Ready` column show number of containers in that pod.

```
$ kubectl get pod
NAME                              READY   STATUS    RESTARTS   AGE
myapp-pod                         2/2     Running   0          5s
```

`kubectl describe <resource> [name]`

`kubectl describe node minikube`

`kubectl describe pod `

`kubectl delete pod [name]`

### ReplicaSets

Pods alone cannot scale. ReplicaSet holds pod definition as a template and is able to scale the pods up/down.

ReplicaSets need to know which pods belong to the set. This mapping is specified in `spec.selector.matchLabels` field.  Only those pods matching in the labels section are considered for the replica set.

```
$ kubectl get replicaset
NAME              DESIRED   CURRENT   READY   AGE
foo-replica-set   4         4         0       5s
```

|||
|---|---
|desired | desired # pods
|current | created # pods
|ready | actual running # pods

```
kubectl scale replicaset myreplicaset --replicas=6 
```

### Deployments

This resource brings declarative deploments. A deployment creates a new replica-set and associated pods under-the-hood. Scalability is still managed via replica-sets but deployment adds rollout and rollback deployment capabilities on top of it. Most of the time, deployment manifest brings replica-sets and pods implicitly so there is no need for additional manifests for replica-sets or pods.

```
kubectl create deployment foo --image=bar
```

`--record` flag is used to log the deployment command in the revision history. This can be seen in rollout history.

```
kubectl apply -f deploy.yml --record
```

Deployment strategies
1. recreate
    - stop current release. deploy new release at once. causes down-time.
2. rolling update
    - deploy releases at controlled rate. zero downtime.

A deployment is considered updated if pod template is changed. Deployment phase, called rollout, is triggered. In that phase, first, new replica-set with 0 current pods are created. Then new replica-set is scaled up while old replica-set is scaled down. This repeated until all pods are migrated.

How does rollout determine this?
- It ensures that at least 75% of desired number of pods are up at a time
- Also ensures at most 125% of desired number of pods are up at a time

Let's see an example of rollout. You have a replica-set of 40 pods and you trigger a new rollout. In the beginning, %25 of pods (10 in total) in stale replica-set goes to `Terminating` state one by one. Meanwhile, %25 of total number from new replica-set is put
in `Pending` set. During this, rollout state looks like

```
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
my-app-deployment   30/40   0           30          2d14h
```

When all 10 pods in old replica-set is in `Terminating`state, `Pending` containers start to go into `ContainerCreating state`. During this, rollout state looks like below. See that ready containers are still 30 but up-to-date containers is 10.

```
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
my-app-deployment   30/40   10           30          2d14h
```

Then, previous batch with `ContainerCreating` state start to go into `Running state and also more new pods go into `Pending` and old pods into `Terminating` state`. At this moment, number of up-to-date pods increase but ready pods are still the same although more new pods go into `Running` state because exact number old pods are terminated.

```
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
my-app-deployment   30/40   38           30          2d14h
```

Finally, number of ready pods increase while remaining new pods go into `Ready` state one by one.

```
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
my-app-deployment   30/40   40           30          2d14h
my-app-deployment   31/40   40           31          2d14h
my-app-deployment   32/40   40           32          2d14h
my-app-deployment   33/40   40           33          2d14h
my-app-deployment   34/40   40           34          2d14h
my-app-deployment   35/40   40           35          2d14h
my-app-deployment   36/40   40           36          2d14h
my-app-deployment   37/40   40           37          2d14h
my-app-deployment   38/40   40           38          2d14h
my-app-deployment   39/40   40           39          2d14h
my-app-deployment   40/40   40           40          2d14h
```

The whole process can be monitored with
```
kubectl rollout status deployment my-app-deployment -w

kubectl get deployments.apps my-app-deployment -w

kubectl get pods -w
```

To check deployment revision history. The last row shows current deployment revision.

```
$ kubectl rollout history [deployment] [name]
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         kubectl apply --filename=resources/deploy-def.yml --record=true
```

#### Rollback

Rollback deploys the previous deployment in the revision history. This causes a new revision in the history.

```
kubectl rollout undo [name]

```

```
$ kubectl rollout history [deployment] [name]
REVISION  CHANGE-CAUSE
1         <none>
3         kubectl apply --filename=resources/deploy-def.yml --record=true
4         <none>
```

```
kubectl scale deployment --replicas=3 foo
```

### ConfigMap

Non-confidential configuration resource. Pods can use it as 

* Environment variables in container
* Command-line args for container
* Configuration files in a volume
* Use k8s API to poll on ConfigMap

### k8s network model

#### Prerequisites

Every pod gets its own address. Containers on the same pod share the network space. This is unlike docker services where each container gets a separate IP address.

???Network model requirements

1. Pods can communicate with any pod in the cluster without NAT.
    - kube-proxy proxies service requests directly to target pods. No node network or NAT is involved.

2. Agents on a node can communicate with any pods on the same node.

There are many solutions out there to satify such network requirements. Unlike the built-in overlay network implementation in docker, k8s relies on third-party implementations. To name a few AWS VPC CNI, Google Compute Engine, Flannel, Cisco etc.

### Service

Service is an abstraction that defines a set of logical pods that expose applications to other pods or outside world. Service expands to multiple nodes in the cluster. This is very similar to overlay network in Docker.

Service-pod association is made through selectors in service definition. 

#### kube-proxy

In every node, there runs a `kube-proxy` which is responsible to proxy inbound traffic to the service (VIP and port) and then to backend pods (available service endpoints). `kube-proxy` watches control plane for service endpoint objects and installs iptables rules for available backend pods. Service request is forwarded to one of the backend pod:targetPort directly without NAT and node network space.

There are different proxying modes. One of them happens at the user-space where traffic is interfered by kube-proxy. In this mode, kubeproxy opens a local random port (in iptables rule) for each service and traffic is redirected to kubeproxy first. kubeproxy then resolves backend ports to redirect to. This is not efficient because there is an additional hop (kubeproxy) which happens in the user-space. Another proxy mode which is efficient and the default is `iptables` mode where proxying happens at the kernel level. In this mode, kube-proxy is responsible for installing iptables rules only by syncing with the control plane and does not interfere traffic.

![kube-proxy-in-iptables-mode](https://d33wubrfki0l68.cloudfront.net/27b2978647a8d7bdc2a96b213f0c0d3242ef9ce0/e8c9b/images/docs/services-iptables-overview.svg)

#### Publishing services

ServiceTypes

* ClusterIP (default)
    - Cluster-internal service.

* NodePort
    - Binds a static port *on each* node. `ClusterIP` service is automatically created. The intention is to let applications be accessed externally (outside cluster) at <node-ip>:<node-port>. This is similar to port publishing in docker. 

* LoadBalancer
    - Exposes service externally to cloud provider's load balancer (e.g. AWS ELB). Cloud provider is responsible assigning an external IP for the load balancer and load balancing algorithm.
    
