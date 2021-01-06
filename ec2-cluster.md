# EC2 Cluster

Even on AWS EC2, it can make a lot of sense to create an unmanaged Kubernetes cluster instead of using EKS, to keep the Kubernetes control plane under your control and ownership.

[Rancher](https://rancher.com/) offers node and cluster drivers for Amazon EC2, and in this article, we'll be using the Rancher node driver through [Terraform](https://www.terraform.io/) to create the cluster and set up a [node pool](https://rancher.com/docs/rancher/v2.x/en/cluster-provisioning/rke-clusters/node-pools/) for it. For more details on Rancher's options for cluster creation, look at this  [post](https://rancher.com/blog/2020/build-kubernetes-clusters-on-azure) on the Rancher [blog](https://rancher.com/blog/) or the Rancher [documentation](https://rancher.com/docs/rancher/v2.x/en/cluster-provisioning/).

## Cloud Provider

Unlike other cloud providers, the in-tree AWS provider does not use a separate set of credientials, but relies on IAM instance profiles assigned to the nodes; for details on the IAM policies, have a look at Rancher's [provider documentation](https://rancher.com/docs/rke/latest/en/config-options/cloud-providers/aws/).

All resources that should be accessible to the cloud provider need to be tagged with a unique tag in the form of "kubernetes.io/cluster/cluster-id". In the example below I opted for a fixed value of "rancher" as I manage some resources (VPCs and subnets) outside of Terraform.

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

Let's continue with the plan definitions for the actual cluster resources - the cluster itself, its name, and its node pools.

### Random ID

I use Random to have a unique naming identifier for the cluster and its nodes:

```
resource "random_id" "instance_id" {
 byte_length = 3
}
```

### Credentials

Next step is to define the EC2 credentials so that Terraform and Rancher can create the EC2 instances:

```
resource "rancher2_cloud_credential" "credential_ec2" {
  name = "EC2 Credentials"
  amazonec2_credential_config {
    access_key = var.ec2-access-key
    secret_key = var.ec2-secret-key
  }
}
```

### Node template

With the credentials from above, we set one or more node templates:

```
resource "rancher2_node_template" "template_ec2" {
  name = "EC2 Node Template"
  cloud_credential_id = rancher2_cloud_credential.credential_ec2.id
  engine_install_url = var.dockerurl
  amazonec2_config {
    ami = var.image
    region = var.ec2-region
    security_group = [var.ec2-secgroup]
    subnet_id = var.ec2-subnet
    vpc_id = var.ec2-vpc
    zone = var.ec2-zone
    root_size = var.disksize
    instance_type = var.type
    iam_instance_profile = "rancher-combined-control-worker"
    tags = "kubernetes.io/cluster/rancher,owned"
  }
}
```

Note: Here's where you set the IAM policy and define the resource tag

### Variables

I define most values, such as the Kubernetes version to use, the Amazon Machine Image or the number of nodes, as variables, to make overall plan maintenance easier:

```
variable "k8version" {
  default = "v1.15.12-rancher2-3"
}
variable "image" {
    default = "ami-024e928dca73bfe66"
}
variable "numnodes" {
    default = 3
}
...
```

### Cluster

Now that we have all set up, it's time to define the Kubernetes cluster, using the name from above and set Kubernetes networking and version:

```
resource "rancher2_cluster" "cluster_ec2" {
  name         = "ec2-${random_id.instance_id.hex}"
  description  = "Terraform"

  rke_config {
    kubernetes_version = var.k8version
    cloud_provider {
      name = "aws"
      aws_cloud_provider {
        global {
          kubernetes_cluster_tag = "rancher"
        }
      }
    }
    ignore_docker_version = false
    network {
      plugin = "flannel"
    }
  }
}
```

Note: Again, here we are defining the resource tag - there is no correct value, it just needs to be consistent across all areas.

### Node pool

We need several compute instances; in our case, we use three nodes and give all roles to all nodes:

```
resource "rancher2_node_pool" "nodepool_ec2" {
  cluster_id = rancher2_cluster.cluster_ec2.id
  name = "nodepool"
  hostname_prefix = "rke-${random_id.instance_id.hex}-"
  node_template_id = rancher2_node_template.template_ec2.id
  quantity = var.numnodes
  control_plane = true
  etcd = true
  worker = true
}
```

### Cluster sync

We're almost ready, just let's wait for the cluster to become active, using a timer:

```
resource "null_resource" "before" {
  depends_on = [rancher2_cluster.cluster_ec2,rancher2_node_pool.nodepool_ec2]
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
resource "rancher2_app_v2" "syslog_ec2" {
  lifecycle {
    ignore_changes = all
  }
  cluster_id = rancher2_cluster.cluster_ec2.id
  name = "rancher-logging"
  namespace = "cattle-logging-system"
  repo_name = "rancher-charts"
  chart_name = "rancher-logging"
  chart_version = var.logchart

  depends_on = [rancher2_app_v2.syslog_crd_ec2,rancher2_cluster.cluster_ec2,rancher2_node_pool.nodepool_ec2]
}
```

For the new v2 app resources, it can be beneficial to add a dependency to the control plane, to make sure that the cluster is still accessible while the app resources are being destroyed.

Never run a Kubernetes cluster without monitoring or logging!

## Validation

To validate a successful build, I usually enable Rancher's monitoring app and deploy the "Hello World" of Kubernetes, a WordPress instance, from the Rancher catalog.

## Load Balancer

If you've set up the IAM permissions correctly and tagged all resources, including VPC, creation of a Amazon ELB from Helm should work right away.

## Storage Class

To use Amazon EBS as storage, you'll need to define a storage class:

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
reclaimPolicy: Delete
parameters:
  type: gp2
volumeBindingMode: WaitForFirstConsumer
```

The option WaitForConsumer will help with zonal assignment

## Troubleshooting

The best place for troubleshooting during plan execution is the output of the pod running Rancher - it provides detailed information on what Rancher is currently doing, and complete error messages if something goes wrong.

You can find sample plan files for this Rancher node driver installation and a sample IAM policy on my [GitHub](https://github.com/chfrank-cgn/Rancher/tree/master/ec2-cluster-1).

Happy Ranching!

*(Last update: 1/6/21, cf)*
