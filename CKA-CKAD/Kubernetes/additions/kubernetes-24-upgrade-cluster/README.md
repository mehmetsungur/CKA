# Hands-on Kubernetes-24: Upgrade A Kubernetes Cluster

Purpose of the this hands-on training is to give students the knowledge of upgrading a Kubernetes cluster.

## Learning Outcomes

At the end of the this hands-on training, students will be able to;

- Upgrade a Kubernetes Cluster

## Outline

- Part 1 - Setting up the Kubernetes Cluster

- Part 2 - Upgrade A Cluster

## Part 1 - Setting up the Kubernetes Cluster

- Launch a Kubernetes Cluster of Ubuntu 22.04 with two nodes (one master, one worker) using the [Cloudformation Template to Create Kubernetes Cluster](../../class-prep/cfn-template-to-create-k8s-cluster.yml). *Note: Once the master node up and running, worker node automatically joins the cluster.*

>*Note: If you have problem with kubernetes cluster, you can use this link for lesson.*
>https://killercoda.com/playgrounds

- Check if Kubernetes is running and nodes are ready.

```bash
kubectl cluster-info
kubectl get no
```

## Part 2 - Upgrade A Cluster

- The way that you upgrade a cluster depends on how you initially deployed it and on any subsequent changes.

- At a high level, the steps you perform are:

  - Upgrade the control plane
  - Upgrade the nodes in your cluster
  - Upgrade clients such as kubectl
  - Adjust manifests and other resources based on the API changes that accompany the new Kubernetes version

### Upgrade approaches

#### kubeadm

- If your cluster was deployed using the kubeadm tool, refer to Upgrading kubeadm clusters for detailed information on how to upgrade the cluster.

- Once you have upgraded the cluster, remember to install the latest version of kubectl.

#### Manual deployments

- You should manually update the control plane following this sequence:

  - etcd (all instances)
  - kube-apiserver (all control plane hosts)
  - kube-controller-manager
  - kube-scheduler
  - cloud controller manager, if you use one

- At this point you should install the latest version of kubectl.

- For each node in your cluster, drain that node and then either replace it with a new node that uses the 1.29 kubelet, or upgrade the kubelet on that node and bring the node back into service.

### Upgrading kubeadm cluster

- Firstly check the cluster version.

```bash
kubectl version
```

- You will get an output like this.

```bash
Client Version: v1.28.5
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
Server Version: v1.28.5
```

#### Switching to another Kubernetes package repository

- This step should be done upon upgrading from one to another Kubernetes minor release in order to get access to the packages of the desired Kubernetes minor version.

- Open the file that defines the Kubernetes apt repository using a text editor of your choice:

```bash
sudo vi /etc/apt/sources.list.d/kubernetes.list
```
* You should see a single line with the URL that contains your current Kubernetes minor version. For example, if you're using v1.28, you should see this:

```bash
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /
```

- Change the version in the URL to the next available minor release, for example:

```bash
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /
```

- Save the file and exit your text editor. 

#### Determine which version to upgrade to. 

- Find the latest patch release for Kubernetes 1.29 using the OS package manager.

```bash
# Find the latest 1.29 version in the list.
# It should look like 1.29.x-*, where x is the latest patch.
sudo apt update
apt-cache madison kubeadm
```

#### Upgrading control plane nodes

- The upgrade procedure on control plane nodes should be executed one node at a time. Pick a control plane node that you wish to upgrade first. It must have the /etc/kubernetes/admin.conf file.

- Upgrade kubeadm:

```bash
# replace x in 1.29.0-1.1 with the latest patch version
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.29.0-1.1' && \
sudo apt-mark hold kubeadm
```

- Verify that the download works and has the expected version:

```bash
kubeadm version
```

- Verify the upgrade plan:

```bash
sudo kubeadm upgrade plan
```

- You will get an output like below.

```bash
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.28.5
[upgrade/versions] kubeadm version: v1.29.0
[upgrade/versions] Target version: v1.29.0
[upgrade/versions] Latest version in the v1.28 series: v1.28.5

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       TARGET
kubelet     1 x v1.28.5   v1.29.0
            1 x v1.29.0   v1.29.0

Upgrade to the latest stable version:

COMPONENT                 CURRENT   TARGET
kube-apiserver            v1.28.5   v1.29.0
kube-controller-manager   v1.28.5   v1.29.0
kube-scheduler            v1.28.5   v1.29.0
kube-proxy                v1.28.5   v1.29.0
CoreDNS                   v1.10.1   v1.11.1
etcd                      3.5.9-0   3.5.10-0

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.29.0

***
```

- This command checks that your cluster can be upgraded, and fetches the versions you can upgrade to. It also shows a table with the component config version states.

- Choose a version to upgrade to, and run the appropriate command. For example:

```bash
sudo kubeadm upgrade apply v1.29.0
```

- Once the command finishes you should see:

```bash
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.29.0". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```

#### For the other control plane nodes

- Same as the first control plane node but use:

```bash
sudo kubeadm upgrade node
```

- instead of:

```bash
sudo kubeadm upgrade apply
```

- Also calling kubeadm upgrade plan and upgrading the CNI provider plugin is no longer needed.

#### Drain the node

- Prepare the node for maintenance by marking it unschedulable and evicting the workloads:

```bash
kubectl get node
kubectl drain kube-master --ignore-daemonsets
```

#### Upgrade kubelet and kubectl

- Upgrade the kubelet and kubectl:

```bash
# replace x in 1.29.0-1.1* with the latest patch version
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet='1.29.0-1.1' kubectl='1.29.0-1.1' && \
sudo apt-mark hold kubelet kubectl
```

- Restart the kubelet:

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

#### Uncordon the node,

- Bring the node back online by marking it schedulable:

```bash
kubectl uncordon kube-master
```

- Check the versions

```bash
kubectl version
kubelet --version
```

### Upgrade worker nodes

- The upgrade procedure on worker nodes should be executed one node at a time or few nodes at a time, without compromising the minimum required capacity for running your workloads.

#### Switching to another Kubernetes package repository

- Connect to the worker node.

- This step should be done upon upgrading from one to another Kubernetes minor release in order to get access to the packages of the desired Kubernetes minor version.

- Open the file that defines the Kubernetes apt repository using a text editor of your choice:

```bash
sudo vi /etc/apt/sources.list.d/kubernetes.list
```
* You should see a single line with the URL that contains your current Kubernetes minor version. For example, if you're using v1.28, you should see this:

```bash
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /
```

- Change the version in the URL to the next available minor release, for example:

```bash
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /
```

- Save the file and exit your text editor. 

- Upgrade kubeadm:

```bash
# replace x in 1.29.0-1.1 with the latest patch version
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.29.0-1.1' && \
sudo apt-mark hold kubeadm
```

#### Call "kubeadm upgrade"

- For worker nodes this upgrades the local kubelet configuration:

```bash
sudo kubeadm upgrade node
```

#### Drain the node

- Prepare the node for maintenance by marking it unschedulable and evicting the workloads:

```bash
# execute this command on a control plane node
# replace kube-worker with the name of your node
kubectl drain kube-worker --ignore-daemonsets
```

- Upgrade kubelet and kubectl

```bash
# replace x in 1.29.0-1.1* with the latest patch version
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet='1.29.0-1.1' kubectl='1.29.0-1.1' && \
sudo apt-mark hold kubelet kubectl
```

- Restart the kubelet:

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

#### Uncordon the node
- Bring the node back online by marking it schedulable:

```bash
# execute this command on a control plane node
# replace kube-worker with the name of your node
kubectl uncordon kube-worker
```

#### Verify the status of the cluster

- After the kubelet is upgraded on all nodes verify that all nodes are available again by running the following command from anywhere kubectl can access the cluster:

```bash
kubectl get nodes
```

- The STATUS column should show Ready for all your nodes, and the version number should be updated.

Referances:

- https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/

- https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/

- https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes/