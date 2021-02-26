# VMware Cluster

On VMware's ESX virtualization platform, it's pretty easy to create Kubernetes clusters and keep the Kubernetes control plane under your control and ownership.

[Rancher](https://rancher.com/) offers node drivers for vSphere. In this article, we'll be using the Rancher node driver through [Terraform](https://www.terraform.io/) to create the cluster and set up a [node pool](https://rancher.com/docs/rancher/v2.x/en/cluster-provisioning/rke-clusters/node-pools/) for it. For more details on Rancher's options for cluster creation, look at the Rancher [documentation](https://rancher.com/docs/rancher/v2.x/en/cluster-provisioning/).

## Terraform Provider

I'm assuming that you have set up Terraform already. As a first step, we need to define the [Rancher2 provider](https://www.terraform.io/docs/providers/rancher2/index.html):

```
provider "rancher2" {
  api_url = var.rancher-url
  token_key = var.rancher-token
  insecure = true
}
```

## Terraform Main

Let's continue with the cluster resources' plan definitions - the cluster itself, its name, and its node pools.

### Random ID

I use Random to have a unique naming identifier for the cluster and its nodes:

```
resource "random_id" "instance_id" {
 byte_length = 3
}
```

### Variables

I define most values, such as the Kubernetes version to use, the ESX Template, or the number of nodes, as variables to make overall plan maintenance easier:

```
variable "k8version" {
  default = "v1.19.7-rancher1-1"
}
variable "image" { 
    default = "/Datacenter/Folder/Templates/ubuntu-lts-mini"
}
variable "numnodes" {
    default = 3
}
...
```

### Credentials

The next step is to define the vSphere credentials so that Terraform and Rancher can create the ESX instances:

```
resource "rancher2_cloud_credential" "credential_co" {
  name = "vsphere-${random_id.instance_id.hex}"
  vsphere_credential_config {
    vcenter = var.vcenter-server
    username = var.vcenter-user
    password = var.vcenter-pass
  }
}
```

### Node template

With the credentials from above, we set one or more node templates, using the option to clone from a template:

```
resource "rancher2_node_template" "template_co" {
  name = "node-${random_id.instance_id.hex}"
  cloud_credential_id = rancher2_cloud_credential.credential_co.id
  engine_install_url = var.dockerurl
  vsphere_config {
    clone_from = var.image
    cpu_count = var.cpucount
    creation_type = "template"
    datacenter = var.vcenter-datacenter
    datastore = var.vcenter-datastore
    disk_size = var.disksize
    folder = var.vcenter-folder
    memory_size = var.memory
    network = [ var.vcenter-network ]
    pool = var.vcenter-pool
  }
}
```

### Cluster template

Creating a cluster template is optional; it helps to ensure uniform characteristics in your Kubernetes cluster deployments.

In this case, we start with a fresh template:

```
resource "rancher2_cluster_template" "template_co" {
  name = "cluster-${random_id.instance_id.hex}"
  template_revisions {
    name = "v1"
    default = true
    cluster_config {
      cluster_auth_endpoint {
        enabled = false
      }
      rke_config {
        kubernetes_version = var.k8version
        ignore_docker_version = false
        network {
          plugin = "flannel"
        }
        cloud_provider {
          vsphere_cloud_provider {
        ...
          }
        }
      }
    }
  }
}
```

### Cloud provider

Let's look at the cloud provider definition in more detail. Here, we define the credentials and access points:

```
            global {
              insecure_flag = true
            }
            virtual_center {
              datacenters = var.vcenter-datacenter
              name        = var.vcenter-server
              user        = var.vcenter-user
              password    = var.vcenter-pass
            }
            workspace {
              server            = var.vcenter-server
              datacenter        = var.vcenter-datacenter
              folder            = "/"
              default_datastore = var.vcenter-datastore
            }
```

### Cluster

Now that we have all set up, it's time to define the Kubernetes cluster, using the cluster template from above:

```
resource "rancher2_cluster" "cluster_co" {
  name         = "vm-${random_id.instance_id.hex}"
  description  = "Terraform"
  cluster_template_id = rancher2_cluster_template.template_co.id
  cluster_template_revision_id = rancher2_cluster_template.template_co.default_revision_id
}
```

### Node pool

We need several compute instances; in our case, we use three nodes and give all roles to all nodes:

```
resource "rancher2_node_pool" "nodepool_co" {
  cluster_id = rancher2_cluster.cluster_co.id
  name = "combined-pool"
  hostname_prefix = "rke-${random_id.instance_id.hex}-"
  node_template_id = rancher2_node_template.template_co.id
  quantity = var.numnodes
  control_plane = true
  etcd = true
  worker = true
}
```

### Cluster sync

We're almost ready; let's wait for the cluster to become active, using a timer:

```
resource "null_resource" "before" {
  depends_on = [rancher2_cluster.cluster_co,rancher2_node_pool.nodepool_co]
}

resource "null_resource" "delay" {
  provisioner "local-exec" {
    command = "sleep ${var.delaysec}"
  }

  triggers = {
    "before" = "null_resource.before.id"
  }
}
```

### Syslog

As the final step, we enable Rancher's logging app from the new marketplace:

```
resource "rancher2_app_v2" "syslog_co" {
  lifecycle {
    ignore_changes = all
  }
  cluster_id = rancher2_cluster.cluster_co.id
  name = "rancher-logging"
  namespace = "cattle-logging-system"
  repo_name = "rancher-charts"
  chart_name = "rancher-logging"
  chart_version = var.logchart

  depends_on = [rancher2_node_pool.nodepool_co]
}
```

For the new v2 app resources, it can be beneficial to add a dependency to the control plane to ensure that the cluster is still accessible while the app resources are being destroyed.

Never run a Kubernetes cluster without monitoring or logging!

## Validation

To validate a successful build, I usually enable Rancher's monitoring app and deploy the "Hello World" of Kubernetes, a WordPress instance, from the Rancher catalog.

## Storage Class

To use vSpehere storage, we'll need to define a storage class:

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  name: thin
provisioner: kubernetes.io/vsphere-volume
parameters:
  diskformat: thin
reclaimPolicy: Delete
```

## Troubleshooting

The best place for troubleshooting during plan execution is the pod running Rancher - it provides detailed information on what Rancher is currently doing and complete error messages if something goes wrong.

You can find sample plan files for this Rancher node driver installation on my [GitHub](https://github.com/chfrank-cgn/Rancher/tree/master/vm-cluster).

Happy Ranching!

*(Last update: 2/21/21, cf)*
