# OpenStack Cluster

[OpenStack](https://www.openstack.org/) is still a viable cloud operating system and in use by many ISPs worldwide. OpenStack installations might differ slightly, so this article's setup is for a Green-IT public cloud environment in Amsterdam, at [leaf.cloud](https://www.leaf.cloud/)

[Rancher](https://rancher.com/) offers node drivers for OpenStack. However, in this article, we will be using [Terraform](https://www.terraform.io/), and a [custom node](https://rancher.com/docs/rancher/v2.x/en/cluster-provisioning/rke-clusters/custom-nodes/) setup, together with the OpenStack [cloud provider](https://rancher.com/docs/rke/latest/en/config-options/cloud-providers/openstack/) to build our green Kubernetes cluster.

## Provider

I'm assuming that you have set up Terraform already. As a first step, we need to define two providers, an [OpenStack provider](https://registry.terraform.io/providers/terraform-provider-openstack/openstack/latest/docs) for the infrastructure:

```
provider "openstack" {
  user_name   = var.lc-user
  tenant_name = "chfrank"
  password    = var.lc-password
  auth_url    = "https://the.greenedge.cloud:5000"
}
```

and a [Rancher2 provider](https://www.terraform.io/docs/providers/rancher2/index.html) for the Kubernetes cluster itself:

```
provider "rancher2" {
  api_url = var.rancher-url
  token_key = var.rancher-token
  insecure = true
}
```

## Main

The next plan definitions are for the new cluster resources - the cluster itself, its name, and its instances.

### Random ID

To have a unique naming identifier for cluster and nodes, we use Random:

```
resource "random_id" "instance_id" {
 byte_length = 3
}
```

### Variables

I define most values, such as the Kubernetes version to use, or the number of nodes, as variables to make overall plan maintenance easier:

```
variable "k8version" {
  default = "v1.15.9-rancher1-1"
}
variable "numnodes" {
    default = 3
}
...
```

### Cluster

We define the Rancher cluster, using the name from above and set Kubernetes networking and version:

```
resource "rancher2_cluster" "cluster_lc" {
  name         = "lc-${random_id.instance_id.hex}"
  description  = "Terraform"

  rke_config {
    kubernetes_version = var.k8version
    cloud_provider {
      name = "openstack"
      openstack_cloud_provider {
        ...
      }
    ignore_docker_version = false
    network {
      plugin = "flannel"
    }
  }
}
```

### Cloud Provider

Let's look at the cloud provider definition in more detail. In the global section, we define the credentials and access points:

```
        global {
          auth_url = "https://the.greenedge.cloud:5000"
          username = var.lc-user
          password = var.lc-password
          tenant_name = "chfrank"
          domain_name = "Default"
        }
```

In the storage section, we define parameters to access Cinder:

```
        block_storage {
          ignore_volume_az = true
          trust_device_path = false
          bs_version = "v2"
        }
```

And finally, in the load balancer section, we define parameters for Kubernetes to ask Neutron to build a load balancer:

```
        load_balancer {
          lb_version = "v2"
          manage_security_groups = false
          use_octavia = true
          subnet_id = var.subnet_id
          floating_network_id = var.floating_id
          create_monitor = false
        }
```

The floating network id is the network id from the external network, where the floating IP addresses live - make sure to have some allocated.

The subnet id is the subnet id (not network id) from your worker nodes' internal network; Octavia will use this network for its amphorae. To allow communication, add a rule to the security allowing access on the high ports (30000-32767) between all nodes on that network.

### Instances

For the cluster, we need several compute instances; Terraform brought new control elements with 0.12, one of which is an easy way to define multiple resources of the same type (count):

```
resource "openstack_compute_instance_v2" "vm_lc" {
  name         = "rke-${random_id.instance_id.hex}-${count.index}"
  flavor_name = var.flavor
  image_name = var.image
  key_pair = var.keypair
  security_groups = ["default"]
  count = var.numnodes
  user_data = data.template_file.startup-script_data.rendered

  network {
    name = var.network
  }
}
```

### Data

We need to pass the Rancher registration command into the startup script above, so we need a data template that allows us to substitute Terraform variables in local files:

```
data "template_file" "startup-script_data" {
  template = file("${path.module}/files/startup-script")
  vars = {
    registration_command = "${rancher2_cluster.cluster_lc.cluster_registration_token.0.node_command} 
                            --etcd --controlplane --worker"
  }
  depends_on = [rancher2_cluster.cluster_lc]
}
```

The registration command will only be available after successfully creating the cluster in Rancher, hence the dependency.

### Cluster sync

We're almost ready; just let's wait for the cluster to become active, using a timer:

```
resource "null_resource" "before" {
  depends_on = [rancher2_cluster.cluster_lc]
}

# Delay hack part 2
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
resource "rancher2_app_v2" "syslog_lc" {
  lifecycle {
    ignore_changes = all
  }
  cluster_id = rancher2_cluster.cluster_lc.id
  name = "rancher-logging"
  namespace = "cattle-logging-system"
  repo_name = "rancher-charts"
  chart_name = "rancher-logging"
  chart_version = var.logchart

  depends_on = [openstack_compute_instance_v2.vm_lc]
}
```

For the new v2 app resources, it can be beneficial to add a dependency to the control plane to ensure that the cluster is still accessible while the app resources are being destroyed.

Never run a Kubernetes cluster without monitoring or logging!

## Validation

To validate a successful build, I usually enable Rancher's monitoring app and deploy the "Hello World" of Kubernetes, a WordPress instance, from the Rancher catalog.

## Load Balancer

If you've set up the cloud provider correctly, creating an Octavia load balancer from Helm should work right away.

## Storage Class

To use Cinder volumes as storage, we'll need to define a storage class:

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cinder
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/cinder
reclaimPolicy: Delete
parameters:
  availability: europe-nl-ams1
```

## Troubleshooting

The best place for troubleshooting during plan execution is the pod running Rancher - it will provide detailed information on what Rancher is currently doing and complete error messages should something go wrong.

You can find sample plan files for this Rancher custom node installation on my [GitHub](https://github.com/chfrank-cgn/Rancher/tree/master/lc-cluster).

Happy Ranching!

*(Last update: 1/7/21, cf)*
