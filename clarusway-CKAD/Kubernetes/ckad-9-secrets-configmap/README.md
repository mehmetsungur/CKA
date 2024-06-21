# Hands-on Kubernetes-CKAD-09 : Managing Secrets and ConfigMaps

Purpose of the this hands-on training is to give students the knowledge of Kubernetes Secrets and config-map

## Learning Outcomes

At the end of the this hands-on training, students will be able to;

- Explain the Kubernetes Secrets

- Share sensitive data (such as passwords) using Secrets.

- Learn configuration management for applications in Kubernetes using ConfigMaps.

## Outline

- Part 1 - Setting up the Kubernetes Cluster

- Part 2 - Kubernetes Secrets

- Part 3 - ConfigMaps in Kubernetes

## Part 1 - Setting up the Kubernetes Cluster

- Launch a Kubernetes Cluster of Ubuntu 22.04 with two nodes (one master, one worker) using the [Cloudformation Template to Create Kubernetes Cluster](../ckad-1-kubernetes-basic-operations/cfn-template-to-create-k8s-cluster.yml). *Note: Once the master node up and running, worker node automatically joins the cluster.*

>*Note: If you have problem with kubernetes cluster, you can use this link for lesson.*
>https://killercoda.com/playgrounds

- Check if Kubernetes is running and nodes are ready.

```bash
kubectl cluster-info
kubectl get no
```

## Part 2 - Kubernetes Secrets

## Creating your own Secrets 

### Creating a Secret Using kubectl

- Secrets can contain user credentials required by Pods to access a database. For example, a database connection string consists of a username and password. You can store the username in a file ./username.txt and the password in a file ./password.txt on your local machine.

```bash
# Create files needed for the rest of the example.
echo -n 'admin' > ./username.txt
echo -n '1f2d1e2e67df' > ./password.txt
```

- The kubectl create secret command packages these files into a Secret and creates the object on the API server. The name of a Secret object must be a valid DNS subdomain name. Show types of secrets with opening : (Kubetnetes Secret Types)[https://kubernetes.io/docs/concepts/configuration/secret/]

```bash
kubectl create secret generic db-user-pass --from-file=./username.txt --from-file=./password.txt
```

- The output is similar to:

```bash
secret "db-user-pass" created
```

- Default key name is the filename. You may optionally set the key name using `[--from-file=[key=]source]`.

```bash
kubectl create secret generic db-user-pass-key --from-file=username=./username.txt --from-file=password=./password.txt
```

>Note:
>Special characters such as `$`, `\`, `*`, `=`, and `!` will be interpreted by your shell and require escaping. In most shells, the easiest way to escape the password is to surround it with single quotes (`'`). For example, if your actual password is S!B\*d$zDsb=, you should execute the command this way:
>
>```bash
>kubectl create secret generic dev-db-secret --from-literal=username=devuser --from-literal=password='S!B\*d$zDsb='
>```
>You do not need to escape special characters in passwords from files (--from-file).

- You can check that the secret was created:

```bash
kubectl get secrets
```

- The output is similar to:

```bash
NAME                  TYPE                                  DATA      AGE
db-user-pass          Opaque                                2         51s
```

You can view a description of the secret:

```bash
kubectl describe secrets/db-user-pass
```

Note: The commands kubectl get and kubectl describe avoid showing the contents of a secret by default. This is to protect the secret from being exposed accidentally to an onlooker, or from being stored in a terminal log.

The output is similar to:
```bash
Name:            db-user-pass
Namespace:       default
Labels:          <none>
Annotations:     <none>

Type:            Opaque

Data
====
password.txt:    12 bytes
username.txt:    5 bytes
```

### Creating a Secret manually

- You can also create a Secret in a file first, in JSON or YAML format, and then create that object. The name of a Secret object must be a valid DNS subdomain name. The Secret contains two maps: data and stringData. The data field is used to store arbitrary data, encoded using base64. The stringData field is provided for convenience, and allows you to provide secret data as unencoded strings.

- For example, to store two strings in a Secret using the data field, convert the strings to base64 as follows:

```bash
echo -n 'admin' | base64
```

- The output is similar to:

```bash
YWRtaW4=
```

```bash
echo -n '1f2d1e2e67df' | base64
```

- The output is similar to:

```bash
MWYyZDFlMmU2N2Rm
```

- Write a Secret that looks like this named `secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

- Now create the Secret using `kubectl apply`:

```bash
kubectl apply -f ./secret.yaml
```

- The output is similar to:

```bash
secret "mysecret" created
```

### Decoding a Secret

- Secrets can be retrieved by running kubectl get secret. For example, you can view the Secret created in the previous section by running the following command:

```bash
kubectl get secret mysecret -o yaml
```

- The output is similar to:

```yaml
apiVersion: v1
data:
  password: MWYyZDFlMmU2N2Rm
  username: YWRtaW4=
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"password":"MWYyZDFlMmU2N2Rm","username":"YWRtaW4="},"kind":"Secret","metadata":{"annotations":{},"name":"mysecret","namespace":"default"},"type":"Opaque"}
  creationTimestamp: "2021-10-06T12:51:08Z"
  name: mysecret
  namespace: default
  resourceVersion: "38986"
  uid: fb55a84e-24f5-461d-a000-e7dab7c34200
type: Opaque
```

- Decode the password field:

```bash
echo 'MWYyZDFlMmU2N2Rm' | base64 --decode
```

- The output is similar to:

```bash
1f2d1e2e67df
```

### Using Secrets

- Firstly we get the parameters as plain environment variable.

- Create a file named `mysecret-pod.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: mycontainer
    image: redis
    env:
      - name: SECRET_USERNAME
        value: admin
      - name: SECRET_PASSWORD
        value: 1f2d1e2e67df
  restartPolicy: Never
```

- Create the pod.

```bash
kubectl apply -f mysecret-pod.yaml
```

- Connect the pod and check the environment variables.

```bash
kubectl exec -it secret-env-pod -- bash
root@secret-env-pod:/data# echo $SECRET_USERNAME
admin
root@secret-env-pod:/data# echo $SECRET_PASSWORD
1f2d1e2e67df
```

- Delete the pod.

```bash
kubectl delete -f mysecret-pod.yaml
```

- This time we get the environment variables from secret objects. Modify the mysecret-pod.yaml as below.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: mycontainer
    image: redis
    env:
      - name: SECRET_USERNAME
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: username
      - name: SECRET_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
  restartPolicy: Never
```

- Update the pod.

```bash
kubectl apply -f mysecret-pod.yaml
```

### Consuming Secret Values from environment variables

- Inside a container that consumes a secret in an environment variables, the secret keys appear as normal environment variables containing the base64 decoded values of the secret data. This is the result of commands executed inside the container from the example above:

- Enter into pod and type following command.

```bash
kubectl exec -it secret-env-pod -- bash
root@secret-env-pod:/data# echo $SECRET_USERNAME
admin
root@secret-env-pod:/data# echo $SECRET_PASSWORD
1f2d1e2e67df
```

## Part 3 - ConfigMaps in Kubernetes

- A ConfigMap is a dictionary of configuration settings. This dictionary consists of key-value pairs of strings. Kubernetes provides these values to your containers. Like with other dictionaries (maps, hashes, ...) the key lets you get and set the configuration value.

- A ConfigMap stores configuration settings for your code. Store connection strings, public credentials, hostnames, environment variables, container command line arguments and URLs in your ConfigMap.

- ConfigMaps bind configuration files, command-line arguments, environment variables, port numbers, and other configuration artifacts to your Pods' containers and system components at runtime.

- ConfigMaps allow you to separate your configurations from your Pods and components. 

- ConfigMap helps to makes configurations easier to change and manage, and prevents hardcoding configuration data to Pod specifications.

- ConfigMaps are useful for storing and sharing non-sensitive, unencrypted configuration information.

## Create and use ConfigMaps with `kubectl create configmap` command

There are three ways to create ConfigMaps using the `kubectl create configmap` command. Here are the options.

1. Use the contents of an entire directory with `kubectl create configmap my-config --from-file=./my/dir/path/`
   
2. Use the contents of a file or specific set of files with `kubectl create configmap my-config --from-file=./my/file.txt`
   
3. Use literal key-value pairs defined on the command line with `kubectl create configmap my-config --from-literal=key1=value1 --from-literal=key2=value2`

- To understand configmap, we will use `clarusway/clarusweb:env` image. Following is the Dockerfile and HTML for this image.

```Dockerfile
FROM nginx:latest
ENV COLOR="red"
RUN apt-get update ; apt-get install curl -y
WORKDIR /usr/share/nginx/html
COPY . /usr/share/nginx/html
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 CMD curl -f http://localhost/ || exit 1
CMD sed -e s/COLOR/"$COLOR"/ clarusweb.html > index.html ; rm clarusweb.html ; nginx -g 'daemon off;'
```

```html
<html>
<head>
<title>clarusweb</title>
</head>
<body style="background-color:COLOR;">
<h1>Welcome to Clarusway<h1>
</body>
</html>
```

- Create a file named `clarusweb.yaml`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clarusweb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name:  web
          image: clarusway/clarusweb:env
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
  labels:
    app: web
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30001    
  selector:
    app: web
```

- Create the objects.

```bash
kubectl apply -f clarusweb.yaml
```

- Check the `nodeport:30001` and see that background is red.

- This time we change the background using environment variable. Update the deployment as below.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clarusweb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name:  web
          image: clarusway/clarusweb:env
          ports:
            - containerPort: 80
          env:
            - name: COLOR
              value: blue
```

- Update the deployment.

```bash
kubectl apply -f clarusweb.yaml
```

- Check the `nodeport:30001` and see that this time the background is blue.

- We will get the value of COL0R from a configmap.

```bash
kubectl create configmap color-config --from-literal=paint=green
```

- Update the deployment as below.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clarusweb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name:  web
          image: clarusway/clarusweb:env
          ports:
            - containerPort: 80
          env:
            - name: COLOR
              valueFrom:
                configMapKeyRef:
                  name: color-config
                  key: paint
```

- Update the deployment and see that this time the background is green.

```bash
kubectl apply -f clarusweb.yaml
```

- delete the objects.

```bash
kubectl delete -f clarusweb.yaml
```

## Configure all key-value pairs in a ConfigMap as container environment variables

- In case if you are using envFrom  instead of env  to create environmental variables in the container, the environmental names will be created from the ConfigMap's keys. If a ConfigMap  key has invalid environment variable name, it will be skipped but the pod will be allowed to start.


- Create  a `configmap.yaml` file.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: allkey-config
data:
  MY_VAR: "Hello"
  ANOTHER_VAR: "123" 
  VAR1: value1
  var2: value2
```

- Create the configmap.

```bash
kubectl apply -f configmap.yaml
```

- Create a file named `config-pod.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-env-pod
spec:
  containers:
  - name: config-container
    image: redis
    envFrom:
      - configMapRef:       # For secret use secretRef
          name: allkey-config
  restartPolicy: Never
```

- Create the pod and check the environment variables.

```bash
kubectl apply -f config-pod.yaml
kubectl exec config-env-pod -- env
```

- Delete the objects.

```bash
kubectl delete -f config-pod.yaml
kubectl delete -f configmap.yaml
```

## Using ConfigMaps as volumes in a Pod

- Create the `nginx-configmap.yaml` as follows:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  config: |
    Hello World
```

- Create the configmap.

```bash
kubectl apply -f nginx-configmap.yaml
```

- Create a file named `nginx-web.yaml`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name:  nginx
          image: nginx:latest
          ports:
            - containerPort: 80
          volumeMounts:
          - mountPath: /usr/share/nginx/html
            name: nginx-config-volume
            readOnly: true
      volumes:
      - name: nginx-config-volume
        configMap:
          name: nginx-config
          items:
          - key: config
            path: index.html
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30002
  selector:
    app: nginx
```

- Volume and volume mounting are common ways to place config files inside a container. We are selecting `config` key from `nginx-config` ConfigMap and put it inside the container at path `/usr/share/nginx/html` with the name `index.html`.

- Apply the files.

```bash
kubectl apply -f nginx-web.yaml
```

Let's see the message.

```bash
curl <node-ip>:30002
Hello World
```

- Delete everything.

```bash
kubectl delete -f .
```

### Optional

#### Using Secret as volumes in a Pod

- Create an `nginx-secret.yaml`.

```bash
kubectl create secret generic nginx-secret --from-literal=username=devuser --from-literal=password='devpassword'
```

- Check the `nginx-secret`.

```bash
kubectl get secret nginx-secret -o yaml
```

- Update `nginx-web.yaml` as below.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name:  nginx
          image: nginx:latest
          ports:
            - containerPort: 80
          volumeMounts:
          - mountPath: /usr/share/nginx/html
            name: nginx-config-volume
            readOnly: true
          - mountPath: /test
            name: secret-volume
      volumes:
      - name: nginx-config-volume
        configMap:
          name: nginx-config
          items:
          - key: config
            path: index.html
      - name: secret-volume
        secret:
          secretName: nginx-secret
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30002
  selector:
    app: nginx
```

- Apply it.

```bash
kubectl apply -f nginx-configmap.yaml
kubectl apply -f nginx-web.yaml
```

- Check the test folder inside the pod.

```bash
kubectl get pod
kubectl exec -it nginx-5b74bf97f-r6vw6 -- bash
cd /test
ls
cat password
cat username
```

- Notice that names of files inside the test folder are the `keys of nginx-secret` object and content of files are `values of nginx-secret` object`.

- Delete everything.

```bash
kubectl delete -f .
```