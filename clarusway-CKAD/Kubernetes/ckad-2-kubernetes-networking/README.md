# Hands-on Kubernetes-CKAD-02 : Kubernetes Networking

Purpose of this hands-on training is to give students the knowledge of Kubernetes Services.

## Learning Outcomes

At the end of the this hands-on training, students will be able to;

- Explain the benefits of logically grouping `Pods` with `Services` to access an application.

- Explore the service discovery options available in Kubernetes.

- Learn different types of Services in Kubernetes.

## Outline

- Part 1 - Setting up the Kubernetes Cluster

- Part 2 - Services, Load Balancing, and Networking in Kubernetes

- Part 3 - Ingress

## Part 1 - Setting up the Kubernetes Cluster

- Launch a Kubernetes Cluster of Ubuntu 22.04 with two nodes (one master, one worker) using the [Cloudformation Template to Create Kubernetes Cluster](../ckad-1-kubernetes-basic-operations/cfn-template-to-create-k8s-cluster.yml). *Note: Once the master node up and running, worker node automatically joins the cluster.*

>*Note: If you have problem with kubernetes cluster, you can use this link for lesson.*
>https://killercoda.com/playgrounds

- Check if Kubernetes is running and nodes are ready.

```bash
kubectl cluster-info
kubectl get no
```

## Part 2 - Services, Load Balancing, and Networking in Kubernetes

Kubernetes networking addresses four concerns:

- Containers within a Pod use networking to communicate via loopback.

- Cluster networking provides communication between different Pods.

- The Service resource lets you expose an application running in Pods to be reachable from outside your cluster.

- You can also use Services to publish services only for consumption inside your cluster.

### Service

An abstract way to expose an application running on a set of Pods as a network service.

With Kubernetes you don't need to modify your application to use an unfamiliar service discovery mechanism.

Kubernetes gives Pods their own IP addresses and a single DNS name for a set of Pods, and can load-balance across them.

### Motivation

Kubernetes Pods are mortal. They are born and when they die, they are not resurrected. If you use a Deployment to run your app, it can create and destroy Pods dynamically.

Each Pod gets its own IP address, however in a Deployment, the set of Pods running in one moment in time could be different from the set of Pods running that application a moment later.

### Service Discovery

The basic building block starts with the Pod, which is just a resource that can be created and destroyed on demand. Because a Pod can be moved or rescheduled to another Node, any internal IPs that this Pod is assigned can change over time.

If we were to connect to this Pod to access our application, it would not work on the next re-deployment. To make a Pod reachable to external networks or clusters without relying on any internal IPs, we need another layer of abstraction. K8s offers that abstraction with what we call a `Service Deployment`.

`Services` provide network connectivity to Pods that work uniformly across clusters.
K8s services provide discovery and load balancing. `Service Discovery` is the process of figuring out how to connect to a service.

- Service Discovery is like networking your Containers.

- DNS in Kubernetes is an `Built-in Service` managed by `Kube-DNS`.

- DNS Service is used within PODs to find other services running on the same Cluster.

- Multiple containers running with-in same POD don’t need DNS service, as they can contact each other.

- Containers within same POD can connect to other container using `PORT` on `localhost`.

- To make DNS work POD always need `Service Definition`.

- Kube-DNS is a database containing key-value pairs for lookup.

- Keys are names of services and values are IP addresses on which those services are running.

### Defining and Deploying Services

- Let's define a setup to observe the behaviour of `services` in Kubernetes and how they work in practice.

- Create a folder and name it service-lessons.

```bash
mkdir service-lessons
cd service-lessons
```

- Create a file named `web-flask.yaml` and explain fields of it.

```yaml
apiVersion: apps/v1 
kind: Deployment 
metadata:
  name: web-flask-deploy
spec:
  replicas: 3 
  selector:  
    matchLabels:
      app: web-flask
  template: 
    metadata:
      labels:
        app: web-flask
    spec:
      containers:
      - name: web-flask-pod
        image: clarusway/cw_web_flask1
        ports:
        - containerPort: 5000
```

- Create the web-flask Deployment.
  
```bash
kubectl apply -f web-flask.yaml
```

- Show the Pods detailed information and learn their IP addresses:

```bash
kubectl get pods -o wide
```

- We get an output like below.

```text
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE
web-flask-7cbb5f5967-24smj          1/1     Running   0          15m   172.16.166.175   node1
web-flask-7cbb5f5967-94g6f          1/1     Running   0          15m   172.16.166.177   node1
web-flask-7cbb5f5967-gjtkx          1/1     Running   0          15m   172.16.166.176   node1
```

In the output above, for each Pod the IPs are internal and specific to each instance. If we were to redeploy the application, then each time a new IP will be allocated.

We now check we can ping a Pod inside the cluster.

- Create a `forcurl.yaml` file to create a Pod that pings a Pod inside the cluster.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: forcurl
spec:
  containers:
  - name: forcurl
    image: clarusway/forping
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
```

- Create a `forcurl` pod and log into the container.

```bash
kubectl get pods
kubectl apply -f forcurl.yaml
kubectl exec -it forcurl -- sh
/ # ping 10.244.1.19
```

- Show the Pods detailed information and learn their IP addresses again.

```bash
kubectl get pods -o wide
```

- Scale the deployment down to zero.

```bash
kubectl scale deploy web-flask-deploy --replicas=0
```

- List the pods again and note that there is no pod in web-flask-deploy.

```bash
kubectl get pods -o wide
```

- Scale the deployment up to three replicas.

```bash
kubectl scale deploy web-flask-deploy --replicas=3
```

- List the pods again and note that the pods are changed.

```bash
kubectl get pods -o wide
```

- Create a `web-svc.yaml` file with following content and explain fields of it.

```yaml
apiVersion: v1
kind: Service   
metadata:
  name: web-flask-svc
  labels:
    app: web-flask
spec:
  type: ClusterIP  
  ports:
  - port: 3000  
    targetPort: 5000
  selector:
    app: web-flask
```
  
```bash
kubectl apply -f web-svc.yaml
```

- List the services.

```bash
kubectl get svc -o wide
```

```text
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE     SELECTOR
kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP                  4h39m   <none>
web-flask-svc   ClusterIP   10.98.173.110   <none>        3000/TCP                 28m     app=web-flask
```

- Display information about the `web-flask-svc` Service.

```bash
kubectl describe svc web-flask-svc
```

```text
Name:              web-flask-svc
Namespace:         default
Labels:            app=web-flask
Annotations:       <none>
Selector:          app=web-flask
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4 
IP:                10.105.60.230
IPs:               10.105.60.230
Port:              <unset>  3000/TCP
TargetPort:        5000/TCP
Endpoints:         172.16.180.5:5000,172.16.180.6:5000
Session Affinity:  None
Events:            <none>
```

- Go to the pod and ping the deployment which has service with ClusterIP and see the ip address of service. 

```bash
kubectl exec -it forcurl -- sh
/ # curl <IP of service web-flask-svc>:3000
/ # ping web-flask-svc 
/ # curl web-flask-svc:3000
```

- As we see kubernetes services provide DNS resolution.

### NodePort

- Change the service type of `web-flask-svc` service to `NodePort` to use the Node IP and a static port to expose the service outside the cluster. So we get the yaml file below.

```yaml
apiVersion: v1
kind: Service   
metadata:
  name: web-flask-svc
  labels:
    app: web-flask
spec:
  type: NodePort  
  ports:
  - port: 3000  
    targetPort: 5000
  selector:
    app: web-flask
```

- Configure the web-flask-svc service via apply command.

```bash
kubectl apply -f web-svc.yaml
```

- List the services again. Note that kubernetes exposes the service in a random port within the range `30000-32767` using the Node’s primary IP address.

```bash
kubectl get svc -o wide
```

```
kubectl exec -it forcurl -- sh
/ # curl <IP of service web-flask-svc>:3000
/ # ping web-flask-svc 
/ # curl web-flask-svc:3000
```

- We can visit `http://<public-node-ip>:<node-port>` and access the application. Pay attention to load balancing. 

Note: Do not forget to open the Port `<node-port>` in the security group of your node instance.

### Endpoints

As Pods come-and-go (scaling up and down, failures, rolling updates etc.), the Service dynamically updates its list of Pods. It does this through a combination of the label selector and a construct called an Endpoint object.

Each Service that is created automatically gets an associated Endpoint object. This Endpoint object is a dynamic list of all of the Pods that match the Service’s label selector.

Kubernetes is constantly evaluating the Service’s label selector against the current list of Pods in the cluster. Any new Pods that match the selector get added to the Endpoint object, and any Pods that disappear get removed. This ensures the Service is kept up-to-date as Pods come and go.

- List the Endpoints.

```bash
kubectl get ep -o wide
```

- Scale the deployment up to ten replicas and list the `Endpoints`.

```bash
kubectl scale deploy web-flask-deploy --replicas=10
```

- List the `Endpoints` and explain that the Service has an associated `Endpoint` object with an always-up-to-date list of Pods matching the label selector.

```bash
kubectl get ep -o wide 
```

> Open a browser on any node and explain the `loadbalancing` via browser. (Pay attention to the host ip and node name and note that `host ips` and `endpoints` are same)
>
> http://[public-node-ip]:[node-port]

- Delete all objects.

```bash
kubectl delete -f .
```

## Part 3 - Ingress

- Create a folder and name it ingress-lesson.

```bash
mkdir ingress-lesson
cd ingress-lesson
```

- Create a file named `clarusshop.yaml` for clarusshop deployment object.

```yaml
apiVersion: apps/v1 
kind: Deployment 
metadata:
  name: clarusshop-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: clarusshop 
  template: 
    metadata:
      labels:
        app: clarusshop
    spec:
      containers:
      - name: clarusshop-pod
        image: clarusway/clarusshop
        ports:
        - containerPort: 80
```

- Create a file named `clarusshop-svc.yaml` for clarusshop service object.

```yaml
apiVersion: v1
kind: Service   
metadata:
  name: clarusshop-svc
  labels:
    app: clarusshop
spec:
  type: NodePort  
  ports:
  - port: 80  
    targetPort: 80
    nodePort: 30001
  selector:
    app: clarusshop
```

- Create a file named `account.yaml` for account deployment object.

```yaml
apiVersion: apps/v1 
kind: Deployment 
metadata:
  name: account-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: account
  template: 
    metadata:
      labels:
        app: account
    spec:
      containers:
      - name: account-pod
        image: clarusway/clarusshop:account
        ports:
        - containerPort: 80
```

- Create a file named `account-svc.yaml` for account service object.

```yaml
apiVersion: v1
kind: Service   
metadata:
  name: account-svc
  labels:
    app: account
spec:
  type: NodePort  
  ports:
  - port: 80  
    targetPort: 80
    nodePort: 30002
  selector:
    app: account
```

- Create the objects.

```bash
kubectl apply -f .
```

### Ingress

- Briefly explain ingress and ingress controller. For additional information a few portal can be visited like;

- https://kubernetes.io/docs/concepts/services-networking/ingress/
    
- Open the offical [ingress-nginx]( https://kubernetes.github.io/ingress-nginx/deploy/ ) explain the `ingress-controller` installation steps for different architecture. We install ingress for bare metal.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/baremetal/deploy.yaml
```

- Create a file named `ing.yaml` for ingress object.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: clarusshop-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: clarusshop-svc
                port: 
                  number: 80
          - path: /account
            pathType: Prefix
            backend:
              service:
                name: account-svc
                port: 
                  number: 80
```

- Create the ingress object.

```bash
kubectl apply -f ing.yaml
```

> We can also create ingress with the following command.

```bash
kubectl create ingress clarusshop-ingress --rule="/account*=account-svc:80" --rule="/*=clarusshop-svc:80" --class=nginx --annotation="nginx.ingress.kubernetes.io/rewrite-target=/"
```

- Check the ingress object.

```bash
kubectl get ingress
```

- You will get an output like below.

```bash
NAME                 CLASS   HOSTS   ADDRESS         PORTS   AGE
clarusshop-ingress   nginx   *       172.31.21.219   80      83m
```

- Check the `ingress-nginx-controller` service.

```bash
kubectl get svc ingress-nginx-controller -n ingress-nginx
```

- You will get an output like below.

```bash
NAME                       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller   NodePort   10.103.234.64   <none>        80:32153/TCP,443:32407/TCP   30m
```

- This will give us a nodePort for http. In this case, it is `32153`. Using this port we can reach the services.

```bash
$ kubectl get ingress
NAME                 CLASS   HOSTS   ADDRESS         PORTS   AGE
clarusshop-ingress   nginx   *       172.31.21.219   80      83m
```

- Use the address of ingress and nodeport of the `ingress-nginx-controller` service to reach services.

```bash
$ curl 172.31.21.219:32153
<h1>WELCOME TO THE CLARUSSHOP</h1><h2>For account service:<br>/account</h2>
$ curl 172.31.21.219:32153/account
<h1>ACCOUNT SERVICE</h1>
```

- Alternatively, we can reach the services via `ingress-nginx-controller` service.

```bash
kubectl run forcurl --image=clarusway/forping
kubectl exec -it forcurl -- sh
/ # curl ingress-nginx-controller.ingress-nginx
<h1>WELCOME TO THE CLARUSSHOP</h1><h2>For account service:<br>/account</h2>
/ # curl ingress-nginx-controller.ingress-nginx/account
<h1>ACCOUNT SERVICE</h1>
/ # exit
```

- We can also visit `http://<public-node-ip>:<node-port of ingress-nginx-controller>` and access the application.

> Note: Usually, ingress is used with a cloud provider. When we create an ingress object, ingress create a load balancer on cloud environment. But, in our case we use bare metal. So, we reach the services via the `ingress-nginx-controller` service in the `ingress-nginx` namespace.

- Delete everything.

```bash
kubectl delete -f .
```

#### Host

- We can define a host name for ingress. Update the `ing.yaml` file as below.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: clarusshop-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: "clarusshop.clarusway.us"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: clarusshop-svc
                port: 
                  number: 80
          - path: /account
            pathType: Prefix
            backend:
              service:
                name: account-svc
                port: 
                  number: 80
```

- Apply the ingress object.

```bash
kubectl apply -f ing.yaml
```

> We can also create ingress with the following command.

```bash
kubectl create ingress clarusshop-ingress --rule="clarusshop.clarusway.us/*=clarusshop-svc:80" --rule="clarusshop.clarusway.us/account/*=account-svc:80" --class=nginx --annotation="nginx.ingress.kubernetes.io/rewrite-target=/"
```

- Check the ingress object.

```bash
kubectl get ingress
```

- You will get an output like below.

```bash
NAME                 CLASS   HOSTS                         ADDRESS         PORTS   AGE
clarusshop-ingress   nginx   clarusshop.clarusway.us       172.31.21.219   80      83m
```

- To reach application with `host` name, add the `clarusshop.clarusway.us` address to `/etc/hosts` file.

```bash
sudo sh -c "echo '172.31.21.219 clarusshop.clarusway.us' >> /etc/hosts"
```

- Check the host address.

```bash
sudo cat /etc/hosts
```

- You can reach the application using curl command.

```bash
curl clarusshop.clarusway.us:32153
```

- Delete all objects.

```bash
kubectl delete -f .
```

#### Name based virtual hosting

- Create a folder named `virtual-hosting`.

```bash
mkdir virtual-hosting && cd virtual-hosting
```

- Create two pods and services for nginx and apache.

```bash
kubectl run mynginx --image=nginx --port=80 --expose
kubectl run myapache --image=httpd --port=80 --expose
kubectl get po,svc
```

- Create ingress file named `mying.yaml` and use name based virtual hosting.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: "nginx.clarusway.us"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: mynginx
                port: 
                  number: 80
    - host: "apache.clarusway.us"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapache
                port: 
                  number: 80
```

- Create the ingress object.

```bash
kubectl apply -f mying.yaml
```

> We can also create ingress with the following command.

```bash
kubectl create ingress myingress \
  --rule="nginx.clarusway.us/*=mynginx:80" \
  --rule="apache.clarusway.us/*=myapache:80" \
  --class=nginx \
  --annotation=nginx.ingress.kubernetes.io/rewrite-target=/
```

- Check the ingress object.

```bash
kubectl get ingress
```

- You will get an output like below.

```bash
NAME        CLASS   HOSTS                                        ADDRESS         PORTS   AGE
myingress   nginx   nginx.clarusway.us,apache.clarusway.us       172.31.21.219   80      83m
```

- To reach application with `host` name, add the `nginx.clarusway.us,apache.clarusway.us` addresses to `/etc/hosts` file.

```bash
sudo sh -c "echo '172.31.21.219 nginx.clarusway.us' >> /etc/hosts"
sudo sh -c "echo '172.31.21.219 apache.clarusway.us' >> /etc/hosts"
```

- Check the host address.

```bash
sudo cat /etc/hosts
```

```bash
curl nginx.clarusway.us:32153
curl apache.clarusway.us:32153
```

- Delete the ingress object.

```bash
kubectl delete -f mying.yaml
```