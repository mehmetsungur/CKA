# Hands-on Kubernetes-CKAD-17: Helm Basics

The purpose of this hands-on training is to give students the knowledge of basic operations of Helm.

## Learning Outcomes

At the end of this hands-on training, students will be able to;

- Learn basic operations of Helm

- Learn how to create Helm Chart

## Outline

- Part 1 - Setting up the Kubernetes Cluster

- Part 2 - Basic Operations with Helm

## Part 1 - Setting up the Kubernetes Cluster

- Launch a Kubernetes Cluster of Ubuntu 22.04 with two nodes (one master, one worker) using the [Cloudformation Template to Create Kubernetes Cluster](../ckad-1-kubernetes-basic-operations/cfn-template-to-create-k8s-cluster.yml). *Note: Once the master node up and running, worker node automatically joins the cluster.*

>*Note: If you have problem with kubernetes cluster, you can use this link for lesson.*
>https://killercoda.com/playgrounds

- Check if Kubernetes is running and nodes are ready.

```bash
kubectl cluster-info
kubectl get no
```

## Part 2 - Basic Operations with Helm

### [Three Big Concepts](https://helm.sh/docs/intro/using_helm/)

- A `Chart` is a Helm package. It contains all of the resource definitions necessary to run an application, tool, or service inside of a Kubernetes cluster. Think of it like the Kubernetes equivalent of a Homebrew formula, an Apt dpkg, or a Yum RPM file.

- A `Repository` is the place where charts can be collected and shared. It's like Perl's CPAN archive or the Fedora Package Database, but for Kubernetes packages.

- A `Release` is an instance of a chart running in a Kubernetes cluster. One chart can often be installed many times into the same cluster. And each time it is installed, a new release is created. Consider a MySQL chart. If we want two databases running in your cluster, we can install that chart twice. Each one will have its own release, which will in turn have its own release name.

- With these concepts in mind, we can now explain Helm like this:

**Helm installs charts into Kubernetes, creating a new release for each installation. And to find new charts, we can search Helm chart repositories.**

* Install Helm [version 3+](https://github.com/helm/helm/releases) on Jenkins Server. [Introduction to Helm](https://helm.sh/docs/intro/). [Helm Installation](https://helm.sh/docs/intro/install/).

```bash
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
helm version
```

### helm search: Finding Charts

- Helm comes with a powerful search command. It can be used to search two different types of sources:

- `helm search hub` searches the Artifact Hub, which lists helm charts from dozens of different repositories.

- `helm search repo` searches the repositories that we have added to your local helm client (with helm repo add). This search is done over local data, and no public network connection is needed.

- We can find publicly available charts by running helm search hub:

```bash
helm search hub
```

- Searches for all wordpress charts on Artifact Hub.

```bash
helm search hub wordpress
```

- We can add the repository using the following command.

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

- Using helm search repo, we can find the names of the charts in repositories we have already added:

```bash
helm search repo bitnami
```

- Type the `helm install command` to install a chart.

```bash
helm repo update
helm install mysql-release bitnami/mysql
```

- We get a simple idea of the features of this MySQL chart by running `helm show chart bitnami/mysql`. Or we could run `helm show all bitnami/mysql` to get all information about the chart.

```bash
helm show chart bitnami/mysql
helm show all bitnami/mysql
```

- Installing the way we have here will only use the default configuration options for this chart. Many times, you will want to customize the chart to use your preferred configuration. To see what options are configurable on a chart, use helm show values.


```bash
helm show values bitnami/mysql # helm show values bitnami/mysql | yq e
```

- Whenever we install a chart, a new release is created. So one chart can be installed multiple times into the same cluster. And each can be independently managed and upgraded.

- Install a new release with bitnami/wordpress chart.

```bash
helm install my-release \
  --set wordpressUsername=admin \
  --set wordpressPassword=password \
  --set mariadb.auth.rootPassword=secretpassword \
    bitnami/wordpress
```

- It's easy to see what has been released using Helm.

```bash
helm list
```

- Uninstall a release.

```bash
helm uninstall my-release
helm uninstall mysql-release
```