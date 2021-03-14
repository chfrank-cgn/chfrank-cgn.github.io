# GCP NFS Storage Class

In this article, we're looking at setting up the NFS Client Provisioner on a Kubernetes cluster using Helm v3. 

Although Kubernetes mainly targets cloud-native stateless applications, there might be a need for persistent storage for particular applications. Suppose you're looking for shared volumes in Kubernetes parlance with "RWX" or "ReadWriteMany" access. In that case, NFS is an excellent choice - the protocol is robust, and there are many rock-solid implementations around. Furthermore, there are well-established provider-side backup and recovery solutions; some cloud-native storage providers only allow for application-side backup and recovery, which does not have me fully convinced.

We'll look at all the individual steps below - this Terraform plan has been applied successfully multiple times, and you can find the complete source code on my [GitHub](https://github.com/chfrank-cgn/Rancher/tree/master/gcp-nfs-helm3).

## Provider

To set up an NFS Storage Class with Helm, we first need to define the [Helm provider](https://registry.terraform.io/providers/hashicorp/helm/latest/docs):

```
provider "helm" {
  kubernetes {
    config_path = "${path.module}/../gcp-cluster/.kube/config"
  }
}
```

The providers will use a kube_config file for access.

## Helm

Once we've defined the provider, Helm can install the NFS client provisioner:

```
resource "helm_release" "nfs_client" {
  name = "nfs-client"
  repository = "https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/"
  chart = "nfs-subdir-external-provisioner"
  namespace = "kube-public"

  set { 
    name = "nfs.server"
    value= "nfs-server.host.name"
  }
  set { 
    name = "nfs.path"
    value= "/mnt/some/path"
  }
}
```

With Helm v3, there's no need anymore for a separate "helm init" step.

## Validation

To validate a successful installation, I usually enable Rancher's monitoring app and deploy the "Hello World" of Kubernetes, a WordPress instance, from the Rancher catalog and enable persistent storage.

On the NFS server, the volumes will then look like this:

```
(21879)nfs-server:/mnt/some/path$ ll
total 8
drwxrwxrwx 3 nfs nfs ... wordpress-data-wordpress-mariadb-0-pvc-f5d9b971-5db3-4a19-8886-ccd6fd47236d
drwxrwxrwx 3 nfs nfs ... wordpress-wordpress-pvc-36b5c1ac-544e-410a-b71f-d8ecfeea0b48
```

Happy NFSing!

*(Last update: 3/14/21, cf)*
