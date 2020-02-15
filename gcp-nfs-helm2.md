# GCP NFS Storage Class

In this article, we're looking at setting up the NFS Client Provisioner on a Kubernetes cluster using Helm v2. 

Although Kubernetes mainly targets cloud-native stateless applications, there might be a need for persistent storage for particular applications. If you're looking for shared volumes, in Kubernetes parlance with "RWX" or "ReadWriteMany" access, NFS is a very good choice - the protocol is robust, and there are many rock-solid implementations around. Furthermore, there are well established provider-side backup and recovery solutions available - cloud-native storage providers mainly only allow for application-side backup and recovery, which does not have me fully convinced.

Down below, we'll look at all the individual steps - this Terraform plan has been successfully executed multiple times, and you can find the complete source code on [GitHub](https://github.com/chfrank-cgn/Rancher/tree/master/gcp-nfs-helm2).

## Provider

To set up a the NFS Storage Class with Helm, we need to create two providers, a [Kubernetes provider](https://www.terraform.io/docs/providers/kubernetes/index.html) to create the service account:

```
provider "kubernetes" {
  config_path = "${path.module}/../gcp-cluster/.kube/config"
}
```

and the actual [Helm provider](https://www.terraform.io/docs/providers/helm/index.html):

```
provider "helm" {
  service_account = "tiller"

  kubernetes {
    config_path = "${path.module}/../gcp-cluster/.kube/config"
  }
}
```

Both providers will use a kube_config file for access.

## Main

To use Helm v2 on a Kubernets cluster, we first need to create a service account:

```
resource "kubernetes_service_account" "tiller" {
  metadata {
    name = "tiller"
    namespace = "kube-system"
  }
}
```

and then bind an admin role to it:

```
resource "kubernetes_cluster_role_binding" "tiller_clusterrolebinding" {
  metadata {
    name = "tiller-clusterrolebinding"
  }
  role_ref {
    kind      = "ClusterRole"
    name      = "cluster-admin"
    api_group = "rbac.authorization.k8s.io"
  }
  subject {
    kind      = "ServiceAccount"
    name      = "tiller"
    namespace = "kube-system"
  }

  depends_on = [kubernetes_service_account.tiller]
}
```

## Helm

Once we've created the service account, Helm can install the NFS client provisioner:

```
resource "helm_release" "nfs_client" {
  name = "nfs-client"
  chart = "stable/nfs-client-provisioner"

  set { 
    name = "nfs.server"
    value= "xxxxxxxx"
  }
  set { 
    name = "nfs.path"
    value= "/mnt/some/path"
  }

  depends_on = [kubernetes_cluster_role_binding.tiller_clusterrolebinding]
}
```

There's no need for a separate "helm init" step - the Helm provider will take care of initialization if it hasn't been done already.

Happy NFSing!

*(Last update: 2/15/20, cf)*
