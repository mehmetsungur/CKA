# Hands-on Kubernetes-CKAD-16 : Use provided tools to monitor Kubernetes applications

Purpose of the this hands-on training is to give students the knowledge of  monitor applications in kubernetes

## Learning Outcomes

At the end of the this hands-on training, students will be able to;

- monitor applications

- Understand the Need for Metric Server

## Outline

- Part 1 - Setting up the Kubernetes Cluster

- Part 2 - Monitor the applications

## Part 1 - Setting up the Kubernetes Cluster

- Launch a Kubernetes Cluster of Ubuntu 22.04 with two nodes (one master, one worker) using the [Cloudformation Template to Create Kubernetes Cluster](../ckad-1-kubernetes-basic-operations/cfn-template-to-create-k8s-cluster.yml). *Note: Once the master node up and running, worker node automatically joins the cluster.*

>*Note: If you have problem with kubernetes cluster, you can use this link for lesson.*
>https://killercoda.com/playgrounds

- Check if Kubernetes is running and nodes are ready.

```bash
kubectl cluster-info
kubectl get no
```

## Part 2 - Monitor the applications

- Create a `php-apache` directory and change directory.

```bash
mkdir php-apache
cd php-apache
```

- Create a `php-apache.yaml` file as below.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  replicas: 1
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            memory: 500Mi
            cpu: 100m
          requests:
            memory: 250Mi
            cpu: 80m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache-service
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
    nodePort: 30002
  selector:
    run: php-apache 
  type: NodePort	
```

- Note how the `Deployment` and `Service` `yaml` files are merged in one file. 

- Deploy this `php-apache` file.

```bash
kubectl apply -f . 
```

- List the pods.

```bash
kubectl get po
```

- List the services.

```bash
kubectl get svc
```

- Let's check what web app presents us.

- On opening browser (http://<public-node-ip>:<node-port>) we see:

```text
OK!
```

- Alternatively, you can use;
```text
curl <public-worker node-ip>:<node-port>
OK!
```

- Do not forget to open the Port <node-port> in the security group of your node instance. 

- Try to get the resource consumption for nodes or pods with following command.

```bash
kubectl top node
kubectl top pods
```

- The `metrics` can't be calculated. So, the `metrics server` should be uploaded to the cluster.

### Install Metric Server 

- First Delete the existing Metric Server if any.

```bash
kubectl delete -n kube-system deployments.apps metrics-server
```

- Get the Metric Server form [GitHub](https://github.com/kubernetes-sigs/metrics-server/releases/tag/v0.7.0).

```bash
wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.7.0/components.yaml
```

- Edit the file `components.yaml`. You will select the `Deployment` part in the file. Add the below line to `containers.args field under the deployment object`.

```yaml
        - --kubelet-insecure-tls
``` 
(We have already done for this lesson)

```yaml
apiVersion: apps/v1
kind: Deployment
......
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=10250
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
......	
```

- Add `metrics-server` to your Kubernetes instance.

```bash
kubectl apply -f components.yaml
```

### Minukube

```bash
minikube addons enable metrics-server
```

- Wait 1-2 minute or so.

- Verify the existace of `metrics-server` run by below command

```bash
kubectl -n kube-system get pods
```

- Verify `metrics-server` can access resources of the pods and nodes.

```bash
kubectl top pods
kubectl top nodes
```

### Increase load

- First look at the services.

```bash
kubectl get svc
```

```bash
kubectl run -it --rm load-generator --image=busybox /bin/sh  

Hit enter for command prompt

while true; do wget -q -O- php-apache-service ; done 
```

### From any environment

```bash
while true; do wget -q -O- http://<puplic ip>:<port number of php-apache-service>; done
```

- Within a minute or so, we should see the higher CPU and Memory load by executing:

```bash
kubectl top pod
```