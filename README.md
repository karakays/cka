# ckad

## Components

|name| task
|---|---
|api-server| meets outbound requests from kubectl and inbound requests from kubelets
|controller| monitors workers
|etcd| distributed datastore
|scheduler| assigns tasks to workers
|container runtime
|kubelet| agent

## Node roles / components

Which nodes have what components?

|node| components
|---|---
|master| api-server, controller, etcd, scheduler
|worker| container runtime, kubelet

Q>>> control-plane vs. master node
Q>>> kubectl talks to api-server only?
Q>>> kubectl connection refused when in worker nodes because there is no api-server?

## k8s cluster provisioning

### As a single-node cluster

`minikube` bundles k8s components including container runtime in a single image and provides k8s service in a single node.

`minikube start`

Q>>> minikube vs Docker desktop
Q>>> is minikube a container running in docker-desktop with iamge gcr.io/k8s-minikube/kicbase (from dd dashboard). That
means kubernetes single cluster runs as a docker container.
Q>>> local register of docker desktop is different from minikube local registry? not only that minikube has docker
container runtime (as k8s component)?
that's why we need `eval $(minikube docker-env)?
Q>>> minikube -p minikube docker-env

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

#### Prerequisite

Every pod gets its own address. Container on the same pod share their network spaces. This is unlike docker services where each container gets a separate IP address.

Network model requirements

????? Check confirm
1. Pods can communicate with all other pods on other pods without NAT.

2. Agents on a node can communicate with any pods on the same node.

There are many solutions out there to satify such network requirements. Unlike the built-in overlay network implementation in docker, k8s relies on third-party implementations. To name a few AWS VPC CNI, Google Compute Engine, Flannel, Cisco etc.

### Service

Service is an abstraction that defines a set of logical pods that expose applications to other pods or outside world. Service expands to multiple nodes in the cluster. This is very similar to overlay network in Docker.

How does service know about pods? This association is made through selectors in service definition. 

#### kube-proxy

Every node runs a `kube-proxy`. `kube-proxy` is responsible to proxy inbound traffic to backend pods via round-robin algorithm. For this purpose, it watches control-plane for addition/removal of service and endpoint objects. For each service, it opens a random local port on local node which is called proxy port.

Check ???
I still don't understand communication flow here. If pod-a makes request to service-b, is it  pod -> service-a -> proxy
port -> choose available nodes (from pods) --> arrives at chosen node --> proxies to pod
OR
proxy port -> pod port?

[kube-proxy](https://d33wubrfki0l68.cloudfront.net/e351b830334b8622a700a8da6568cb081c464a9b/13020/images/docs/services-userspace-overview.svg)

#### How to access node from host?
>>> Host and node are in the same network and can access each other by IP without NAT. question? what type of network is >>> this ? who creates this network?
???
I was able to access NodePort through localhost or primary network interface in host in play-with-k8s.com.

Service types

* ClusterIP (default)
    - cluster-internal service.

* NodePort
    - Binds a static port *on each* node to the service. `ClusterIP` service is automatically created. The intention is to let applications be accessed externally (outside cluster) at <node-ip>:<node-port>. This is similar to port publishing in docker. 

* LoadBalancer
    - Exposes service externally using cloud provider's load balancer (for insance AWS ELB). Cloud provider is responsible assigning an external IP for the load balancer. Cloud provider is also responsible for load balancing algorithm.
    
