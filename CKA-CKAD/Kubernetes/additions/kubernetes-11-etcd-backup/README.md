# Hands-on Kubernetes-11 : Kubernetes Backup and Restore

The purpose of this hands-on training is to give students the knowledge of taking backup and restoring a cluster from backup.

## Learning Outcomes

At the end of this hands-on training, students will be able to;

- Explain the etcd.

- Take a cluster backup.

- Restore cluster from backup.

## Outline

- Part 1 - Setting up the Kubernetes Cluster

- Part 2 - Cluster Backup

- Part 3 - Restore Cluster From Backup.

## Part 1 - Setting up the Kubernetes Cluster

- Launch a Kubernetes Cluster of Ubuntu 22.04 with two nodes (one master, one worker) using the [Cloudformation Template to Create Kubernetes Cluster](../../class-prep/cfn-template-to-create-k8s-cluster.yml). *Note: Once the master node up and running, worker node automatically joins the cluster.*

>*Note: If you have problem with kubernetes cluster, you can use this link for lesson.*
>https://killercoda.com/playgrounds

- Check if Kubernetes is running and nodes are ready.

```bash
kubectl cluster-info
kubectl get no
```

## Part 2 - Cluster Backup

- `etcd` is a consistent and highly-available key-value store used as Kubernetes' backing store for all cluster data.

- Firstly, we will check the `etcd-kube-master` pod in kube-system namespace is running.

```bash
kubectl get po -n kube-system
kubectl get po etcd-kube-master  -n kube-system
```

- Let's find, at what address can we reach the ETCD cluster and where the ETCD server certificate file is located from the kube-master node?

```bash
kubectl describe pod etcd-kube-master -n kube-system
```

- Check the --listen-client-urls, ETCD server certificate file, ETCD server key file and ETCD CA Certificate file.

```bash
--listen-client-urls=https://127.0.0.1:2379,https://172.31.45.8:2379
...
--cert-file=/etc/kubernetes/pki/etcd/server.crt
...
--key-file=/etc/kubernetes/pki/etcd/server.key
...
--trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

- Check this path.

```bash
ls /etc/kubernetes/pki/etcd
```

### Data Directory

- When first started, etcd stores its configuration into a data directory specified by the data-dir configuration parameter.

- A user should avoid restarting an etcd member with a data directory from an out-of-date backup. Using an out-of-date data directory can lead to inconsistency.

- Let's see where the data directory of etcd is.

```bash
kubectl describe pod etcd-kube-master -n kube-system
```
- Check the --data-dir. We will get a PATH as below.

```bash
--data-dir=/var/lib/etcd
```

### etcdctl commands

- `etcdctl` is the CLI tool used to interact with `etcd`.

- `etcdctl` can interact with ETCD Server using 2 API versions. Version 2 and Version 3.  

- The API version used by etcdctl to speak to etcd may be set to version 2 or 3 via the ETCDCTL_API environment variable. By default, etcdctl on master (3.4) uses the v3 API, and earlier versions (3.3 and earlier) default to the v2 API.

- We will install etcdctl.

```bash
sudo apt  install etcd-client
```

- Let's see the version of `etcdctl`.

```bash
etcdctl --version # This is the version 2 command.
```

or 

```bash
etcdctl version # This is the version 3 command.
```

- To set the version of API set the environment variable ETCDCTL_API command.

```bash
export ETCDCTL_API=3
```

- Now we are ready for backup and restore operations. But before, we run an application.

- Create a deployment file

```bash
cat << EOF > clarusshop.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clarusshop-deployment
  labels:
    app: clarusshop
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
      - name: clarusshop
        image: clarusway/clarusshop
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: clarusshop-service
spec:
  selector:
    app: clarusshop
  type: NodePort
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30001
EOF
```

- Execute the application.

```bah
kubectl apply -f clarusshop.yaml
```

- Check the application.

```bash
kubectl get po
kubectl get svc
```

- We can visit `http://<public-node-ip>:30001` and access the application or we can use curl command.

```bash
curl localhost:30001
```

> Do not forget to open 30001 port from the security group.

- It's time to take a backup of the cluster. For this, we will use `etcd snapshot save` command.

```bash
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
snapshot save /opt/snapshot-backup.db
```

These fields are mandatory.

> --cacert : Verify certificates of TLS-enabled secure servers using this CA bundle.
>
> --cert   : Identify secure client using this TLS certificate file.
>
> --key    : Identify secure client using this TLS key file
>
> --endpoints=127.0.0.1:2379 : This is the default as ETCD is running on master node and exposed on localhost 2379.

- Now we have a backup file named snapshot-backup.db. From this file, we restore our cluster.

- Let's assume that we have a problem and we can not reach the application. To simulate this one delete the applicatiın.

```bash
kubectl delete -f clarusshop.yaml
```

- Check that application is deleted.

```bash
kubectl get po
```

- And we decide the restore the cluster immediately.

```bash
sudo ETCDCTL_API=3 etcdctl  snapshot restore /opt/snapshot-backup.db \
--data-dir /var/lib/etcd-backup
```

### Static Pods

- Static Pods are managed directly by the kubelet daemon on a specific node, without the API server observing them. Unlike Pods that are managed by the control plane (for example, a Deployment); instead, the kubelet watches each static Pod (and restarts it if it fails).

- Static Pods are always bound to one Kubelet on a specific node.

- The kubelet automatically tries to create a mirror Pod on the Kubernetes API server for each static Pod. This means that the Pods running on a node are visible on the API server, but cannot be controlled from there. The Pod names will be suffixed with the node hostname with a leading hyphen.

- Let's modify etcd configuration file to use the new data directory. For this we change the `etcd.yaml` file at `etc/kubernetes/manifests/` directory.

```bash
sudo vi /etc/kubernetes/manifests/etcd.yaml
```

- Modify the files as shown below:

```yaml
  volumes:
  - hostPath:
      path: /var/lib/etcd-backup
      type: DirectoryOrCreate
    name: etcd-data
```

- When we modify any manifest file under the ```etc/kubernetes/manifests/``` folder, kubelet create the pod again. 

- Wait a while and check the app is running.

```bash
kubectl get deploy
kubectl get svc
curl localhost:30001
```

- Restart the controller and scheduler pod.

```bash
kubectl -n kube-system delete po kube-scheduler-kube-master
kubectl -n kube-system delete po kube-controller-manager-kube-master
```

