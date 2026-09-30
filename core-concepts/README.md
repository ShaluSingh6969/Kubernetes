# Kubernetes

## Master Node
- Kubernetes API server (kubectl helps us communicate with it)
- Scheduler
- Controller Manager
- etcd (store state of our cluster)

### The API Server
The Central management enitity, only component that directly connects to etcd.

Core functionality
- External: via kubectl
- Nodes: via kubelet which runs on each node
- Persistent state of objects via: etcd

### etcd
- distributed key-value pair store
- Primmary datastore of Kubernetes
- Stores and replicate all Kubernetes cluser states.
- Runs in high availability mode (3 of them)
    - Requires a quorum for the following:
        - Elect a new ETCD member
        - Update the datastore 

### Scheduler
Schedules Pods on nodes
- Plays "Tetris" on all nodes
- Once the Pod is placed on a node, the scheduler is done.
- Kubelets then takes over to deploy and observe the Pod.

### Controller-manager
A daemon that embeds controllers inside the master node, such as:
- replication controller
- endpoints controller
- namespace controller
- serviceaccounts controller

## Worker Nodes
- kubelet
- kube-proxy
- container-runtime (e.g. docker) (running inside a pod)

### kubelet
- Focused on running containers
- Runs on all nodes
- Pods are defined by a JSON or YAML file
    - Called a Pod Manifest
- Has an internal HTTP server.
    - Read-only view on port 10255
    - Kubelets URLs that you can curl
        - /health - health check
        - /pods
        - /spec

### Container Runtime
(Used to be Docker)
1. Docker client (kubelet handles this)
    - docker build
    - docker pull
    - docker run
2. Container Runtime (runs on each node)
    - CRI (interface)
    - Containers
    - Images
        - A read-only template with the instructions for creating a container
        - An image may be based on another image
        - Can be ready-made
3. Registry

## Creating a Pod

![Pod Creation Flow](./images/pod_creation_flow.png)

## Some `kubectl` commands
```
# (node) gives the control plane and worker nodes and other objects we want to know about just replacing nodes
kubectl get nodes/pod/service (use -A to get in all namespace)

# To create a pod
kubectl apply -f <podmanifest.yml>

# To see API server endpoing
kubectl config view
# or directly
kubectl cluster-info

# To further debug and diagnose cluster problems, use
kubectl cluster-info dump

# To check where pods are runnnig
kubectl get pods -o wide

# to create different namespaces within same cluster (can add in metadata as namespace to force in certain namespace)
kubectl create namespace/ns <namespace-name>

# to get object inside certain ns other than default we need to provide the ns
kubectl get pods -n <namespace_name>

# to scale up pods horizontally
kubectl scale deployment backend --replicas=3

# brings up detailed description about resource (best when a pod is not working)
kubectl descirbe RESOURCE RESOURCE_NAME

# to Delete a pod
kubectl delete -f podmanifest.yml

# with specifying object
kubectl delete pod <pod_name>
```

## In real Kubernetes Cluster (multi-node)
We normally have
- Control plane node(s)
- Worker nodes
Pods are distributed by
- kube-scheduler

## How Kuberenteds decides where to place pods
It uses:
- CPU/memory availability
- node labels
- taints/tolerations
- affinity rules
- topology spread constraints

Exaple yaml
```
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: zone
          operator: In
          values:
          - zone-a
```

### Adding resource quota to the namespace
```
apiVersion: v1
kind: ResourceQuota
metadata:
    name: tiny-rq
spec:
    hard:
        cpu: "1"
        memory: 1Gi
```
- We can now attach it to a namespace 
```
kubectl apply -f <resourcequota.yml> -n <namespace_name>

# can see the ns details again using
kubectl describe ns <namespace_name>
```

## How to Ensure pods go to different nodes

### Option 1: Replica scheduling (default behavior)
```
replicas: 3
```
Kubernetes will try to spread them automatically if muliple nodes exist

### Option 2: Pod Anti-Afiinity (best practice)
```
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: myapp
      topologyKey: kubernetes.io/hostname
```
Ensures pods do NOT land on same node

### Option 3: Toplogy Spread Constraints (modern best practice)
```
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: kubernetes.io/hostname
  whenUnsatisfiable: ScheduleAnyway
  labelSelector:
    matchLabels:
      app: myapp
```

## Resource management
```
kubectl top nodes (we need to integrate it using manifest provided by kubernetes, bunch of pods collecting information)

kubectl get apiservice | grep metrics (to check if metric-server is running, then i can use kubectl top nodes)
```
![](./images/kubectl_get_top_nodes.png)

### Resource consumption management for pods
**Requests:** is the parameter we can set for our pods to request minimum amount of resources.
**Limit:** maximum amount your Pod's container is allowed to consume before being cut off.

```
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    resources:
      request:
        cpu: 250m (millicore, basically we can specify even fraction of core we require)
        memory: "65M"
      limits:
        cpu: 500m
        memory: "130M"
```

## Readiness V/S Liveness

**Liveness:** if it fails, pod is dead

**Readiness:** if it fails, pod not ready yet
- Same configuration as a liveness probe
- the Pod is NOT added to the LoadBalancer until it passes the readiness probe
![](./images/liveness_readiness_probes.png)

```
livenessProbe:
  initialDelaySeconds: 2 # How soon after creation are we probing?
  periodSeconds: 5 # how often thereafter are we probing?
  timeoutSeconds: 1 # How long are we giving the container to respond?
  failureThreshold: 3 # How many consecutive failures before we kill this container?
  httpGet: # Method of Probe?
    path: /health
    port: 9876

readinessProbe:
  initialDelaySeconds: 2 # How soon after creation are we probing?
  periodSeconds: 5 # how often thereafter are we probing?
  timeoutSeconds: 1 # How long are we giving the container to respond?
  failureThreshold: 3 # How many consecutive failures before we kill this container?
  httpGet: # Method of Probe?
    path: /health
    port: 9876
```
## Some shortcuts to create pods
```
kubectl run demopod --image=nginx
kubectl port-forward demopod LOCALPORT:CONTAINERPORT
```
## to check logs of particular container inside the same pod
```
kubectl logs <pod_name> -c <container_name> # can use --all-containers to see logs from all the containers
kubectl logs <pod_name> -c <container_name> -f
kubectl logs <pod_name> -c <container_name> --since 10s
```

## to debug deployments
```
kubectl get pods -n kube-system | grep metrics-server
kubectl get endpoints -n kube-system metrics-server(endpoint name)
kubectl logs -n kube-system deployment/metrics-server
```

## copying files from host to container inside pod
```
kubectl cp <host_file> pod_name:<container_location>
```

## to delete all pods
```
kubectl delete pod --all

# be careful if we don't use persistent volume we will loose all of our data when creating the pod again
# This is where we can actually use **ConfigMap** resource (configuration files, environment variables, etc)
```

## labels
```
# to see the labels associated with a pod
kubectl get pods --show-labels

# to create labels from cli on pods
kubectl label pod demo-pod awesome=sauce

# to update an existing label
kubectl label pod demo-pod awesome=thinking --ooverwrite

# to remove the labels
kubectl label pod demo-pod <key_name>-

# to use label as column in get pod output
kubectl get pods -L <label_key_name>

# pods filtering
kubectl get pods --selector=(label_key)=(value)
```

### ConfigMap
```
kubectl create configmap/cm <name of cm> --from-file=<file from which we want to create configmap>
```
####
How to attach it to pod and then the volume to container.

**STEP 1:** add the configmap as volume to the pod.
**STEP 2:** add the volume to the container

```
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    # STEP 2: add the volume to the container
    volumeMounts:
      - name: dc-heroes
        mountPath: /heroes
    resources:
      requests:
        cpu: 250m #(millicore, basically we can specify even fraction of core we require)
        memory: "65M"
      limits:
        cpu: 500m
        memory: "130M"
   # STEP 1: add the configmap as
   # a volume to this pod
  volumes:
    - name: dc-heroes
      configMap:
        name: dem-heroes
```

Be careful when trying to mount an existing directory, use `subPath` approach.
```
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    # STEP 2: add the volume to the container
    volumeMounts:
      - name: dc-heroes
        mountPath: /etc/nginx/heroes.txt #overwrites the existing directory, BE CAREFUL!! (use Subpath)
        subPath: heroes.txt
```

## secrets
- Similar to ConfigMaps but are specifically intended to hold confidential data.
- by default are stored unencrypted in the API server's unerlying data store (etcd)
- anyone with API access can retrieve or modify a Secret, and so can anyone with access to etcd.
- anyone who is authorized to create a Pod in a namespace can use that access to read any Secret in that namespace; this includes indirect access such as the ability to create a Deployment.

## deployment vs replicasets

![](./images/deployment_vs_replicasets.png)

### to rollback deployment
```
kubectl rollout history deploy <deployment_name> (this history keep tracks of all the revsions after that run below)

kubectl rollout undo deploy <deployment-name>
```

Container volume integration to cloud storage.

![](./images/storage.png)

- **Storage class** determine to which type of storage we will store out container data to.
- **manual storage class** - choosing storage on cluter of nodes manually.

## Services

We can use below command to expose a deployment
```
kubectl expose deploy <deployment-name> -- this created the kind service.
```

## Network policy

### Key DevOps concept Networking (very important)

Many beginners assume:
- Kubernetes enforces NetworkPolicies

But the correct statement is:
- Kubernetes only defines the rules — the CNI plugin enforces them

### Networking in docker desktop

What Docker Desktop is doing?
- Docker Desktop Kubernetes uses an internal networking stack (typically based on):

1. vpnkit: internal bridge networking
2. kube-proxy: for Service routing, access to IP table

This means:

✔ Pods are connected through a virtual network

✔ IPs are assigned from a cluster CIDR

✔ kube-proxy handles Services

❌ No NetworkPolicy enforcement layer exists

### How service resource is applied over a pod

![service resource](./images/kind_service.png)

### multicontainer srevice mapping via multiports

![](./images/service_multiport_mapping.png)

### Types of services

1. **Cluster IP**
2. **Node port** - allows external access in the cluster, without using kubectl, bascially client just need to know the node and it's nodeport
3. **Load balancer** - still uses nodeports, but requires external machine from your cluster wiht outward facing IP address to accept incoming traffic.


![](./images/load-balancer-service-type.png)