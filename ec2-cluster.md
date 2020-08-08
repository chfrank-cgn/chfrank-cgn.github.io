# EC2 Cluster

Even on AWS EC2, it can make a lot of sense to create an unmanaged Kubernetes cluster instead of using EKS, to keep the Kubernetes control plane under your control and ownership.

[Rancher](https://rancher.com/) offers node and cluster drivers for Amazon EC2, and in this article, we'll be using the Rancher node driver through [Terraform](https://www.terraform.io/) to create the cluster and set up a [node pool](https://rancher.com/docs/rancher/v2.x/en/cluster-provisioning/rke-clusters/node-pools/) for it. For more details on Rancher's options for cluster creation, look at this  [post](https://rancher.com/blog/2020/build-kubernetes-clusters-on-azure) on the Rancher [blog](https://rancher.com/blog/) or the Rancher [documentation](https://rancher.com/docs/rancher/v2.x/en/cluster-provisioning/).

## Provider

I'm assuming that you have set up Terraform already. As a first step, we need to define the [Rancher2 provider](https://www.terraform.io/docs/providers/rancher2/index.html):

```
provider "rancher2" {
  api_url = var.rancher-url
  token_key = var.rancher-token
  insecure = true
}
```

## Main

Let's continue with the plan the plan definitions for the actual cluster resources - the cluster itself, its name, and its node pools.

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

With the credentials above, we set one or more node templates:

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
  }
}
```

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
    ignore_docker_version = false
    network {
      plugin = "flannel"
    }
  }
}
```

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

We're almost ready, just let's wait for the cluster to become active:

```
resource "rancher2_cluster_sync" "sync_ec2" {
  cluster_id =  rancher2_cluster.cluster_ec2.id
  node_pool_ids = [rancher2_node_pool.nodepool_ec2.id]
}
```

Hat tip to Anders Nyvang of Coop, who pointed me to the cluster sync resource - much better than my previous local-exec hack!

### Syslog

As the final step, we enable Rancher's built-in logging:

```
resource "rancher2_cluster_logging" "ec2_syslog" {
  name = "ec2_syslog"
  cluster_id = rancher2_cluster_sync.sync_ec2.id
  kind = "syslog"
  syslog_config {
    endpoint = "your.syslog.host:514"
    protocol = "udp"
    program = "ec2-${random_id.instance_id.hex}"
    severity = "notice"
    ssl_verify = false
  }
}
```

## Validation

To validate a successful build, I usually enable Rancher's built-in monitoring and quickly deploy the "Hello World" of Kubernetes, a WordPress instance, from the Rancher catalog.
Never run a Kubernetes cluster without monitoring or logging!

## Troubleshooting

The best place for troubleshooting during plan execution is the output of the pod running Rancher - it provides detailed information on what Rancher is currently doing, and complete error messages if something goes wrong.

You can find sample plan files for this Rancher node driver installation on my [GitHub](https://github.com/chfrank-cgn/Rancher/tree/master/ec2-cluster-1).

Happy Ranching!

*(Last update: 8/8/20, cf)*
