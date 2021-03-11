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

## How is k8s shipped?

### single-node
`minikube` bundles k8s components in a single image and provides k8s service in a single node.

`minikube start`

### multi-node
`kubeadm`

### managed service providers

* AWS EKS: Elastic Kubernetes Service

* GCP GKE: Google Kubernetes Engine


### Resource management with `kubectl`

#### Create

Create resource from file in _non-idempotent_ way
`kubectl create -f [resource]`

Create or update resource from file in _idempotent_ way
`kubectl create -f [resource]`

#### Get

Get overview of resource
`kubectl get [resource]`

Get details of resource
`kubectl describe [resource] `

#### Update

```
kubectl apply -f resource.yml
```

```
kubectl edit [resource] [name]
```

```
kubectl set image [resource] [name] nginx=nginx:1.17
```

#### Delete

Delete resource
`kubectl delete [resource] [name]`

## Terms

### Pods

It represents smallest deployable unit. It is a logical group of containers which together form a unit of service by sharing same resources.

Create and start a pod

`kubectl run nginx --image nginx`

`kubectl get pod`

`kubectl describe <resource> [name]`

`kubectl describe node minikube`

`kubectl describe pod `


`kubectl delete pod [name]`

### ReplicaSets

Monitors the replica set and ensures desired number of pods are always up. If replica number goes below, it creates new pods. 

ReplicaSets cannot make guess how to create pods. For this reason, there is `spec.template` to create pods.

ReplicaSets need to know which pods belong to the set. This mapping is specified in `spec.selector.matchLabels` field.  Only those pods matching in the labels section are considered for the replica set.

```
$ kubectl get replicaset
NAME              DESIRED   CURRENT   READY   AGE
foo-replica-set   4         4         0       5s
```

|
|---|---
|desired | desired # pods
|current | created # pods
|ready | actual running # pods

```
kubectl scale replicaset myreplicaset --replicas=6 
```

### Deployments

Declarative deploys for pods and replicasets.

Creating deployment sets up a new replica-set and associated pods under-the-hood.

deployments -> replicaSets -> pods

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
- Deployment ensures that at the time at least 75% of desired number of pods are up. 
- Deployment also ensures at the time at most 125% of desired number of pods are up.

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

Then, previous batch with `ContainerCreating` state start to go into `Running state and also more new pods go into `Pending` and old pods into `Terminating` state. At this moment, number of up-to-date pods increase but ready pods are still the same although more new pods go into `Running` state because exact number old pods are terminated.
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

### Manifest

kind: Pod | ReplicaSets | Deployment

Create a new resource
`kubectl create -f mypod.yml`

Create pod spec
`kubectl run pod --image redis --dry-run=client -o yaml > pod.yaml`

Edit manifest of existing resource
`kubectl edit replicaset myapp-rs`

Update resource
`kubectl apply -f mypod.yml`

`kubectl replace -f myrs.yml`

