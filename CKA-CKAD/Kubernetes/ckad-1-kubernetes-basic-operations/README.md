# Hands-on Kubernetes-CKAD-01 : Kubernetes Basic Operations

Purpose of the this hands-on training is to give students the knowledge of basic operations in Kubernetes cluster.

## Learning Outcomes

At the end of the this hands-on training, students will be able to;

- Learn basic operations of nodes, pods, deployments, replicasets in Kubernetes

- Learn how to update and rollback deployments in Kubernetes

- Learn uses of namespace in Kubernetes

## Outline

- Part 1 - Setting up the Kubernetes Cluster

- Part 2 - Basic Operations in Kubernetes

- Part 3 - Namespaces in Kubernetes

- Part 4 - Deployment Rolling Update and Rollback in Kubernetes

## Part 1 - Setting up the Kubernetes Cluster

- Launch a Kubernetes Cluster of Ubuntu 22.04 with two nodes (one master, one worker) using the [Cloudformation Template to Create Kubernetes Cluster](./cfn-template-to-create-k8s-cluster.yml). *Note: Once the master node up and running, worker node automatically joins the cluster.*

>*Note: If you have problem with kubernetes cluster, you can use this link for lesson.*
>https://killercoda.com/playgrounds

- Check if Kubernetes is running and nodes are ready.

```bash
kubectl cluster-info
kubectl get node
```

## Part 2 - Basic Operations in Kubernetes

- Show the names and short names of the supported API resources as shown in the example:

|NAME|SHORTNAMES|
|----|----------|
|deployments|deploy
|events     |ev
|endpoints  |ep
|nodes      |no
|pods       |po
|services   |svc

```bash
kubectl api-resources
```

- To view kubectl commands:

```
kubectl help
```

- View the nodes in the cluster using.

```bash
kubectl get nodes
```

### pods
  
- Create yaml file named `mypod.yaml` and explain fields of it.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: mynginx
    image: nginx
    ports:
    - containerPort: 80
```

- Create a pod with `kubectl create` command.

```bash
kubectl create -f mypod.yaml
```

- List the pods.

```bash
kubectl get pods
```

- List pods in `ps output format` with more information (such as node name).
  
```bash
kubectl get pods -o wide
```

- Show details of pod.

```bash
kubectl describe pods/nginx-pod
```

- Show details of pod in `yaml format`.
  
```bash
kubectl get pods/nginx-pod -o yaml
```

- Delete the pod.

```bash
kubectl delete -f mypod.yaml
# or
kubectl delete pod nginx-pod
```

### ReplicaSets

- Create yaml file named `myreplicaset.yaml` and explain fields of it.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
  labels:
    environment: dev
spec:
  # modify replicas according to your case
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: mynginx
        image: nginx
        ports:
        - containerPort: 80
```

- Create the replicaset with `kubectl apply` command.

```bash
kubectl apply -f myreplicaset.yaml
```

- List the replicasets.

```bash
kubectl get replicaset
```

- List pods with more information.
  
```bash
kubectl get pods -o wide
```

- Show details of replicasets.

```bash
kubectl describe replicaset <replicaset-name>
```

- Delete replicasets

```bash
kubectl delete replicaset <replicaset-name>
```

#### Pod Selector

- The `.spec.selector` field is a label selector. 

- The `.spec.selector` field and `.spec.template.metadata` field must be coupled.

### Deployments

- Create yaml file named `mydeployment.yaml` and explain fields of it.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    environment: dev
spec:
  replicas: 3
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
kubectl apply -f mydeployment.yaml
```

- List the deployments.

```bash
kubectl get deployments
```

- List pods with more information.
  
```bash
kubectl get pods -o wide
```

- Show details of deployments.

```bash
kubectl describe deploy/nginx-deployment
```

- Print the logs for a container in a pod.

```bash
kubectl logs <pod-name>
```

- If there is a multi-container pod, we can print logs of one container.

```bash
kubectl logs <pod-name> -c <container-name>
```

- Execute a command in a container.

```bash
kubectl exec <pod-name> -- date
```

```bash
kubectl exec <pod-name> -- cat /usr/share/nginx/html/index.html
```

- Open a bash shell in a container.

```bash
kubectl exec -it <pod-name> -- bash
```

- List the ReplicaSets.

```bash
kubectl get rs
```

- Show details of ReplicaSets.

```bash
kubectl describe rs <rs-name>
```

- Scale the deployment up to five replicas.

```bash
kubectl scale deploy nginx-deployment --replicas=5
```

- But each time do we have to apply these commands for scaling? No because there will be our yml file and we can change it when we need scale.

>> Show when you apply mydeployment.yaml change, how differ?

- Delete a pod and show new pod is immediately created.

```bash
kubectl delete pod <pod-name>
kubectl get pods
```

- Delete deployments

```bash
kubectl delete deploy <deployment-name>
```

## Part 3 - Namespaces in Kubernetes

- List the current namespaces in a cluster using and explain them. *Kubernetes supports multiple virtual clusters backed by the same physical cluster. These virtual clusters are called `namespaces`.*

```bash
kubectl get namespace
NAME              STATUS   AGE
default           Active   118m
kube-node-lease   Active   118m
kube-public       Active   118m
kube-system       Active   118m
```

- Create a namespace using below command:

```bash
kubectl create namespace clarus-namespace
```

- Create pods in each namespace.

```bash
kubectl create deployment default-ns --image=nginx
kubectl create deployment clarus-ns --image=nginx -n=clarus-namespace
```

- List the deployments in `default` namespace.

```bash
kubectl get deployment
```

- List the deployments in `clarus-namespace`.

```bash
kubectl get deployment -n clarus-namespace
```

- List the all deployments.

```bash
kubectl get deployment -o wide --all-namespaces
```

- Delete the namespace.

```bash
kubectl delete namespaces clarus-namespace
```

## Part 4 - Deployment Rolling Update and Rollback in Kubernetes

- Create a new folder name it deployment-lesson.

```bash
mkdir deployment-lesson
cd deployment-lesson
```

- Create a `mydeploy.yaml` and input text below. Pay attention that image version is 1.0.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mydeploy
  labels:
    app: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1 
      maxSurge: 1 
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: mycontainer
        image: clarusway/clarusweb:1.0
        ports:
        - containerPort: 80
```

- Create the deployment with `kubectl apply` command.

```bash
kubectl apply -f mydeploy.yaml
```

- List the `Deployment`, `ReplicaSet` and `Pods` of `mydeploy` deployment using a label and note the name of ReplicaSet.

```bash
kubectl get deploy,rs,po -l app=myapp
```

- Describe deployment and note the image of the deployment. In our case, it is `clarusway/clarusweb:1.0`.

```bash
kubectl describe deploy mydeploy
```

- View previous rollout revisions.

```bash
kubectl rollout history deploy mydeploy
```

- Display details with revision number, in our case, is 1. And note name of image.

```bash
kubectl rollout history deploy mydeploy --revision=1
```

- Upgrade image.

```bash
kubectl set image deploy mydeploy mycontainer=clarusway/clarusweb:2.0
```

- Show the rollout history.

```bash
kubectl rollout history deploy mydeploy
```

- Display details about the revisions.

```bash
kubectl rollout history deploy mydeploy --revision=1
kubectl rollout history deploy mydeploy --revision=2
```

- List the `Deployment`, `ReplicaSet` and `Pods` of `mydeploy` deployment using a label and explain ReplicaSets.

```bash
kubectl get deploy,rs,po -l app=myapp
```

- Rollback to `revision 1`.

```bash
kubectl rollout undo deploy mydeploy --to-revision=1
```

- Show the rollout history and show that we have revision 2 and 3. Explain that original revision, which is `revision 1`, becomes `revision 3`.

```bash
kubectl rollout history deploy mydeploy
kubectl rollout history deploy mydeploy --revision=2
kubectl rollout history deploy mydeploy --revision=3
```

- Try to pull up the `revision 1`, that is no longer available.

```bash
kubectl rollout history deploy mydeploy --revision=1
```

- List the `Deployment`, `ReplicaSet` and `Pods` of `mynginx` deployment using a label, and explain that the original ReplicaSet has been scaled up back to three and second ReplicaSet has been scaled down to zero.

```bash
kubectl get deploy,rs,po -l app=myapp
```

- Delete the deployment.

```bash
kubectl delete deploy -l app=myapp
```

> Note: To see `CHANGE-CAUSE`, you can add `kubernetes.io/change-cause` annotation.
