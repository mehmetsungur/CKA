# Hands-on Kubernetes-CKAD-10 : Managing Resources for Containers

Purpose of this hands-on training is to give students the knowledge of managing resources for containers.

## Learning Outcomes

At the end of the this hands-on training, students will be able to;

- managing resources for containers in Kubernetes.

## Outline

- Part 1 - Setting up the Kubernetes Cluster

- Part 2 - Managing Resources for Containers

- Part 3 - Requests

- Part 4 - Configure Max, Minumum and Default Memory and CPU Requests and Limits for a resource under the a Namespace

- Part 5 - Configure Memory and CPU Quotas for a Namespace

## Part 1 - Setting up the Kubernetes Cluster

- Launch a Kubernetes Cluster of Ubuntu 22.04 with two nodes (one master, one worker) using the [Cloudformation Template to Create Kubernetes Cluster](../ckad-1-kubernetes-basic-operations/cfn-template-to-create-k8s-cluster.yml). *Note: Once the master node up and running, worker node automatically joins the cluster.*

>*Note: If you have problem with kubernetes cluster, you can use this link for lesson.*
>https://killercoda.com/playgrounds

- Check if Kubernetes is running and nodes are ready.

```bash
kubectl cluster-info
kubectl get no
```

## Part 2 - Managing Resources for Containers

- When we specify a Pod, we can optionally specify how much of each resource a Container needs. The most common resources to specify are CPU and memory (RAM); there are others.

- When we specify the resource request for Containers in a Pod, the scheduler uses this information to decide which node to place the Pod on. 

- When we specify a resource limit for a Container, the kubelet enforces those limits so that the running container is not allowed to use more of that resource than the limit we set. The kubelet also reserves at least the request amount of that system resource specifically for that container to use.

## Part 3 - Requests

- If the node where a Pod is running has enough of a resource available, it's possible (and allowed) for a container to use more resource than its request for that resource specifies. However, a container is not allowed to use more than its `resource limit`.

- For example, if we set a `memory request` of 256 MiB for a container, and that container is in a Pod scheduled to a Node with 8GiB of memory and no other Pods, then the container can try to use more RAM.

- If we set a `memory limit` of 4GiB for that Container, the kubelet (and container runtime) enforce the limit. The runtime `prevents` the container from using more than the configured `resource limit`. 

- For example: when a process in the container tries to consume more than the allowed amount of memory, the system kernel terminates the process that attempted the allocation, with an out of memory (OOM) error.

> **Note:** If a Container specifies its own memory limit, but does not specify a memory request, Kubernetes automatically assigns a memory request that matches the limit. Similarly, if a Container specifies its own CPU limit, but does not specify a CPU request, Kubernetes automatically assigns a CPU request that matches the limit.

- Let's see this. Create a yaml file named `nginx-deploy.yaml`. Notice that, there is 30 replicas.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    environment: dev
spec:
  replicas: 30
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

- Create the deployment with `kubectl apply` command.

```bash
kubectl apply -f nginx-deploy.yaml
```

- List the pods and notice that all pods are running. Because, we do not specify any resources.

```bash
kubectl get po -o wide
```

- Delete the deployment.

```bash
kubectl delete -f clarus-deploy.yaml 
```

- Update the deployment as below. We add resources section. In our case, we have one worker node that is aws t3 medium. It has 2 vCPUs and 4  GiB memory. We specify 500 Mi for memory request.  

- Limits and requests for CPU resources are measured in cpu units. One cpu, in Kubernetes, is equivalent to `1 vCPU/Core` for cloud providers and 1 hyperthread on bare-metal Intel processors. We make resources.requests.cpu 100 m.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    environment: dev
spec:
  replicas: 20
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "500Mi"
            cpu: "100m"
          limits:
            memory: "750Mi"
            cpu: "750m"
```

- Let's create the deployment.

```bash
kubectl apply -f nginx-deploy.yaml
```

- List the pods and notice that just 7 pods are running. Because of the memory resource requests.

```bash
kubectl get po -o wide
```

- Delete the deployment.

```bash
kubectl delete -f nginx-deploy.yaml 
```

### limits

- We have seen that requests are used for during scheduling. But, limits are used during execution. For example, when a process in the container tries to consume more than the specified limit of memory, the system kernel terminates the process that attempted the allocation, with an out of memory (OOM) error. 

- If a container try to exceed its memory limit, the system terminate the container. But if a container try to exceed its CPU limit, the system does not terminate container, but throttle its CPU usage.

## Part 4 - Configure Max, Minumum and Default Memory and CPU Requests and Limits for a resource under the a Namespace

### If we do not specify a CPU limit for a Container, then one of these situations applies:

- The Container has no upper bound on the CPU resources it can use. The Container could use all of the CPU resources available on the Node where it is running.

- The Container is running in a namespace that has a default CPU limit, and the Container is automatically assigned the default limit. Cluster administrators can use a LimitRange to specify a default value for the CPU limit.

### If we do not specify a memory limit for a Container, one of the following situations applies:

- The Container has no upper bound on the amount of memory it uses. The Container could use all of the memory available on the Node where it is running which in turn could invoke the OOM Killer. Further, in case of an OOM Kill, a container with no resource limits will have a greater chance of being killed.

- The Container is running in a namespace that has a default memory limit, and the Container is automatically assigned the default limit. Cluster administrators can use a LimitRange to specify a default value for the memory limit.

### LimitRange object:

- In kubernetes, we can assign deault memory and CPU requests and limits for Namespaces with the `LimitRange` object.

- Now, we will create a LimitRange object. Create a file and name it `limitrange.yaml`.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
spec:
  limits:
  - default:  # this section defines default limits
      cpu: "1"
      memory: "500Mi"
    defaultRequest: # this section defines default requests
      cpu: "0.5"
      memory: "300Mi"
    max:    # max and min define the limit range
      cpu: "1.5"
      memory: "900Mi"
    min:
      cpu: "100m"
      memory: "100Mi"
    type: Container
```

- Create the LimitRange in the resources-limits namespace and describe it.

```bash
kubectl apply -f limitrange.yaml
kubectl describe limitrange cpu-limit-range
```

- Create a file and name it `limit-pod.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: limit-pod
spec:
  containers:
  - name: nginx
    image: nginx
```

- Create the pod.

```bash
kubectl apply -f limit-pod.yaml
```

- View the Pod's specification:

```bash
kubectl get pod limit-pod --output=yaml
```

- The output shows that the Pod has default cpu and memory values.

```yaml
 containers:
  - image: nginx
    imagePullPolicy: Always
    name: nginx
    resources:
      limits:
        cpu: "1"
        memory: 500Mi
      requests:
        cpu: 500m
        memory: 300Mi
```

- Delete the pod.

```bash
kubectl delete -f limit-pod.yaml
```

- This time, we will try to exceed the provided maximum memory limit value.. Update the `limit-pod.yaml` as below.

```bash
apiVersion: v1
kind: Pod
metadata:
  name: limit-pod
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      limits:
        memory: "950Mi"  # max memory limit range is 900Mi but we define 950Mi.
```

- Create the pod.

```bash
kubectl apply -f limit-pod.yaml
```

- We get an output like below.

```bash
Error from server (Forbidden): error when creating "limit-pod.yaml": pods "limit-pod" is forbidden: maximum memory usage per Container is 900Mi, but limit is 950Mi
```

## Part 5 - Configure Memory and CPU Quotas for a Namespace

- if we want to limit the memory and the CPU request `total` for all containers running in a namespace. For this, we can use `ResourceQuota` object. We can also restrict the totals for memory and CPU limit with `ResourceQuota` object.

- Create a namespace named `resources-limits`.

```bash
kubectl create ns resources-limits
```

- Create a yaml file and name it `resource-quota.yaml`.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-cpu
  namespace: resources-limits
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```

- Create the ResourceQuota for resources-limits namespace.

```bash
kubectl apply -f resource-quota.yaml --namespace=resources-limits
```

- View detailed information about the ResourceQuota.

```bash
kubectl get resourcequota mem-cpu --namespace=resources-limits --output=yaml
```

- The ResourceQuota places these requirements on the resources-limits namespace:

  - Every Container must have a memory request, memory limit, cpu request, and cpu limit.

  - The memory request total for all Containers must not exceed 1 GiB.

  - The memory limit total for all Containers must not exceed 2 GiB.

  - The CPU request total for all Containers must not exceed 1 cpu.

  - The CPU limit total for all Containers must not exceed 2 cpu.

- Let's create a file named `clarus-pod.yaml` as below.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: quota-mem-cpu
  namespace: resources-limits
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: quota-mem-cpu
    resources:
      limits:
        memory: "800Mi"
        cpu: "800m"
      requests:
        memory: "600Mi"
        cpu: "400m"
```

- Create the Pod.

```bash
kubectl apply -f clarus-pod.yaml
```

- Once again, view detailed information about the ResourceQuota:

```bash
kubectl get resourcequota mem-cpu --namespace=resources-limits --output=yaml
```

- The output shows the quota along with how much of the quota has been used. We can see that the memory and CPU requests and limits for our Pod do not exceed the quota.

```yaml
status:
  hard:
    limits.cpu: "2"
    limits.memory: 2Gi
    requests.cpu: "1"
    requests.memory: 1Gi
  used:
    limits.cpu: 800m
    limits.memory: 800Mi
    requests.cpu: 400m
    requests.memory: 600Mi
```

- We will attempt to create a second Pod. Create a file and name it `clarus-pod-2.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: quota-mem-cpu-2
  namespace: resources-limits
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: quota-mem-cpu-2
    resources:
      limits:
        memory: "1Gi"
        cpu: "800m"
      requests:
        memory: "700Mi"
        cpu: "400m"
```

- In the configuration file, we can see that the Pod has a memory request of 700 MiB. Notice that the sum of the used memory request and this new memory request exceeds the memory request quota. 600 MiB + 700 MiB > 1 GiB.

- Create the Pod.

```bash
kubectl apply -f clarus-pod-2.yaml
```

- The second Pod does not get created. The output shows that creating the second Pod would cause the memory request total to exceed the memory request quota.

```
Error from server (Forbidden): error when creating "clarus-pod-2.yaml": pods "quota-mem-cpu-2" is forbidden: exceeded quota: mem-cpu, requested: requests.memory=700Mi, used: requests.memory=600Mi, limited: requests.memory=1Gi
```

- Delete the pods.

```bash
kubectl delete -f clarus-pod.yaml
kubectl delete -f clarus-pod-2.yaml
```

### Define a quota for pod (Optional)

- Create a namespace named `pod-quota`.

```bash
kubectl create ns pod-quota
```

- Create a file named `pod-quota.yaml`.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pod-limit
  namespace: pod-quota
spec:
  hard:
    pods: "10"
```

- Create it.

```bash
kubectl apply -f pod-quota.yaml
```

- Try the create a deployment with `20 replicas`.

```bash
kubectl create deploy demo --image=nginx --replicas=20 -n pod-quota
```

- Check that just 10 pods are created.

```bash
$ kubectl -n pod-quota get deploy
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
demo   10/20   10           10          83s
```

- Check the replicaset and see the error.

```bash
$ kubectl -n pod-quota describe rs demo-664dc5dfd4 
Name:           demo-664dc5dfd4
Namespace:      pod-quota
...
Conditions:
  Type             Status  Reason
  ----             ------  ------
  ReplicaFailure   True    FailedCreate
Events:
  Type     Reason            Age                   From                   Message
  ----     ------            ----                  ----                   -------
  Normal   SuccessfulCreate  4m11s                 replicaset-controller  Created pod: demo-664dc5dfd4-6crrw
 ...
  Warning  FailedCreate      4m11s                 replicaset-controller  Error creating: pods "demo-664dc5dfd4-dg6dg" is forbidden: exceeded quota: pod-limit, requested: pods=1, used: pods=10, limited: pods=10
  ...
```