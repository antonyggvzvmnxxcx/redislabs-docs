---
Title: Deploy Redis Enterprise Software on Kubernetes
linkTitle: Kubernetes 
description: How to install Redis Enterprise Software on Kubernetes.
weight: 10
alwaysopen: false
categories: ["Platforms"]
aliases: [
  /platforms/kubernetes/getting-started/quick-start/,
  /platforms/kubernetes/getting-started/quick-start.md,
  /platforms/kubernetes/deployment/quick-start/,
  /platforms/kubernetes/deployment/quick-start.md, 
  /kubernetes/deployment/quick-start.md,
  /kubernetes/deployment/quick-start/
]
---

To deploy Redis Enterprise Software on Kubernetes, you first need to install the Redis Enterprise operator.

This quick start guide is for generic Kubernetes distributions ([kOps](https://kops.sigs.k8s.io)) as well as:

 * [Azure Kubernetes Service](https://azure.microsoft.com/en-us/services/kubernetes-service/) (AKS)
 * [Rancher](https://rancher.com/products/rancher/) / [Rancher Kubernetes Engine](https://rancher.com/products/rke/) (RKE)
 * [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine) (GKE)
 * [Amazon Elastic Kubernetes Service](https://aws.amazon.com/eks/) (EKS)

If you're running either OpenShift or VMWare Tanzu, we provide specific getting started guides for installing the Redis Enterprise Operator on these platforms:

* [Redis Enterprise on OpenShift]({{< relref "/kubernetes/deployment/openshift/_index.md" >}})
* [Redis Enterprise on VMWare Tanzu]({{< relref "/kubernetes/deployment/tanzu/_index.md" >}})

## Prerequistes

To deploy the Redis Enterprise operator, you'll need:

- a Kubernetes cluster in a [supported distribution]({{<relref "content/kubernetes/reference/supported_k8s_distributions.md">}})
- a minimum of three worker nodes
- a Kubernetes client (kubectl)
- access to DockerHub, RedHat Container Catalog, or a private repository that can hold the required images.

See your version's [release notes]({{<relref "/kubernetes/release-notes/_index.md">}}) for a list of required images.

### Namespaces

The Redis Enterprise operator manages a single Redis Enterprise cluster in a single namespace.

Throughout this guide, you should assume that each command is applied to the namespace in
which the Redis Enterprise cluster operates.

You can set the default namespace for these operations by running:

```
kubectl config set-context --current --namespace=my-namespace
```

## Install the operator

The Redis Enterprise operator [definition and reference materials](https://github.com/RedisLabs/redis-enterprise-k8s-docs) are available on GitHub, while the
operator implementation is published as a Docker container. The operator
definitions are [packaged as a single generic YAML file](https://github.com/RedisLabs/redis-enterprise-k8s-docs/blob/master/bundle.yaml).

### Download the bundle

To ensure that you pull the correct version of the bundle, check versions tags listed with the [operator releases on GitHub](https://github.com/RedisLabs/redis-enterprise-k8s-docs/releases)
or by [using the GitHub API](https://docs.github.com/en/rest/reference/repos#releases).

1. Download the bundle for the latest release:

    ```
    VERSION=`curl --silent https://api.github.com/repos/RedisLabs/redis-enterprise-k8s-docs/releases/latest | grep tag_name | awk -F'"' '{print $4}'`
    curl --silent -O https://raw.githubusercontent.com/RedisLabs/redis-enterprise-k8s-docs/$VERSION/bundle.yaml
    ```

If you need a different release, replace `VERSION` in the above with a specific release tag.

### Apply the bundle

```
kubectl apply -f bundle.yaml
```

  You should see a result similar to this:

  ```sh
  role.rbac.authorization.k8s.io/redis-enterprise-operator created
  serviceaccount/redis-enterprise-operator created
  rolebinding.rbac.authorization.k8s.io/redis-enterprise-operator created
  customresourcedefinition.apiextensions.k8s.io/redisenterpriseclusters.app.redislabs.com configured
  customresourcedefinition.apiextensions.k8s.io/redisenterprisedatabases.app.redislabs.com configured
  deployment.apps/redis-enterprise-operator created
  ```

#### Verify that the operator is running

Check the operator deployment to verify it's running in your namespace: 

```sh
kubectl get deployment -l name=redis-enterprise-operator
```

You should see a result similar to this:

```sh
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
redis-enterprise-operator   1/1     1            1           0m36s
```

### Test the operator

A cluster is created by creating a custom resource with the kind "RedisEnterpriseCluster"
that contains the specification of the cluster option. See the
[Redis Enterprise Cluster API reference](https://github.com/RedisLabs/redis-enterprise-k8s-docs/blob/master/redis_enterprise_cluster_api.md)
for more information on the various options available.

You can test the operator by creating a minimal cluster by following this procedure:

1. Create a file called `simple-cluster.yaml` that defines a Redis Enterprise cluster with three nodes:

    ```
    cat <<EOF > simple-cluster.yaml
    apiVersion: "app.redislabs.com/v1"
    kind: "RedisEnterpriseCluster"
    metadata:
      name: "test-cluster"
    spec:
      nodes: 3
    EOF
    ```

    This will request a cluster with three Redis Enterprise nodes using the
    default requests (i.e., 2 CPUs and 4GB of memory per node).
    If you want to test with a larger configuration, you can
    specify the node resources. For example, this configuration increases the memory:

    ```
    cat <<EOF > simple-cluster.yaml
    apiVersion: "app.redislabs.com/v1"
    kind: "RedisEnterpriseCluster"
    metadata:
      name: "test-cluster"
    spec:
      nodes: 3
      redisEnterpriseNodeResources:
        limits:
          cpu: 2000m
          memory: 16Gi
        requests:
          cpu: 2000m
          memory: 16Gi
    EOF
    ```

    See the [Redis Enterprise hardware requirements]({{< relref "/rs/administering/designing-production/hardware-requirements.md">}}) for more
    information on sizing Redis Enterprise node resource requests.

2. Create the CRD in the namespace with the file `simple-cluster.yaml`:

    ```
    kubectl apply -f simple-cluster.yaml
    ```

    You should see a result similar to this:

    ```
    redisenterprisecluster.app.redislabs.com/test-cluster created
    ```

3. You can verify the creation of the with:

    ```
    kubectl get rec
    ```

    You should see a result similar to this:

    ```
    NAME           AGE
    test-cluster   1m
    ```


   At this point, the operator will go through the process of creating various
   services and pod deployments. You can track the progress by examining the
   StatefulSet associated with the cluster:

   ```
   kubectl rollout status sts/test-cluster
   ```

   or by simply looking at the status of all of the resources in your namespace:

   ```
   kubectl get all
   ```

4. Once the cluster is running, you can create a test database. First, define the database with the following YAML file:

   ```
   cat <<EOF > smalldb.yaml
   apiVersion: app.redislabs.com/v1alpha1
   kind: RedisEnterpriseDatabase
   metadata:
     name: smalldb
   spec:
     memory: 100MB
   EOF
   ```

   Next, apply the database:

   ```
   kubectl apply -f smalldb.yaml
   ```

5. The connectivity information for the database is now stored in a Kubernetes
   secret using the same name but prefixed with `redb-`:

   ```
   kubectl get secret/redb-smalldb -o yaml
   ```

   From this secret you can get the service name, port, and password for the
   default user.


## Operator overview {#overview}

The Redis Enterprise operator uses [custom resource definitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions) (CRDs) to create and manage Redis Enterprise clusters (REC) and Redis Enterprise databases (REDB).

The operator is a deployment that runs within a given namespace. These operator pods must run with sufficient privileges to create the Redis Enterprise cluster resources within that namespace.

When the operator is installed, the following resources are created:

 * a service account under which the operator will run
 * a set of roles to define the privileges necessary for the operator to perform its tasks
 * a set of role bindings to authorize the service account for the correct roles (see above)
 * the CRD for a Redis Enterprise cluster (REC) 
 * the CRD for a Redis Enterprise database (REDB)
 * the operator itself (a deployment)

The operator currently runs within a single namespace and is scoped to operate only on the Redis Enterprise cluster in that namespace.