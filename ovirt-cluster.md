# oVirt Cluster

Rancher 2.x does not include a node driver or a cloud provider for [oVirt](https://www.ovirt.org/), the virtualization platform behind [Red Hat Virtualization](https://www.redhat.com/en/technologies/virtualization/enterprise-virtualization). However, to build a Rancher Kubernetes cluster on oVirt, we can use [Terraform](https://www.terraform.io/) and a [custom node](https://rancher.com/docs/rancher/v2.x/en/cluster-provisioning/rke-clusters/custom-nodes/) setup.

We'll use the same Terraform setup as for the GCP Cluster, and you can find all the plan files on [GitHub](https://github.com/chfrank-cgn/Rancher/tree/master/ovirt-cluster). A big shoutout to my colleague, [Alex Behrendt](mailto:alexander.behrendt@cbc.de), who is an oVirt wizard and supported me with the intricacies of the platform!

## Preparations

A couple of steps are needed to prepare the platform:

- Download, compile and install the custom Terraform [provider plugin](https://github.com/oVirt/terraform-provider-ovirt) for oVirt
- Obtain the oVirt Cluster-ID from the management database: ```select cluster_id, name from cluster where name like 'xxxxxx';```

## Provider

To set up a custom Rancher cluster on oVirt, we first need to configure two providers, the custom oVirt provider plugin for the infrastructure:

```
provider "ovirt" {
  username = var.ovirt-user
  url = var.ovrit-url
  password = var.ovirt-pass
}
```

and the [Rancher2 provider](https://www.terraform.io/docs/providers/rancher2/index.html) for Kubernetes:

```
provider "rancher2" {
  api_url = var.rancher-url
  token_key = var.rancher-token
}
```

## Main

The next plan definitions are for the actual cluster resources - the cluster itself, its name and its instances.

### Random ID

To have a unique naming identifier for cluster and nodes, we use Random:

```
resource "random_id" "instance_id" {
 byte_length = 3
}
```

### Cluster

We define the Rancher cluster, using the name from above and set Kubernetes networking:

```
resource "rancher2_cluster" "cluster_tf" {
  name = "dcc-${random_id.instance_id.hex}"
  description = "Terraform"

  rke_config {
    ignore_docker_version = false
    network {
      plugin = "flannel"
    }
  }
}
```

### Instances

For our cluster, we need several compute instances - in this case, we configure three nodes for the control plane and three more nodes as workers; for brevity, we only show the control node definitions.

We were not using DHCP on this platform, so we're going to define node names and IP addresses in static Terrform lists, indexed by count:

```
variable "vm_control" {
  description = "Unique names for the control nodes"
  default = ["node-1","node-2","node-3"]
}
variable "vm_nic_ip_address_control" {
  description = "The IP addresses for the control nodes"
  default = ["192.168.0.1","192.168.0.2","192.168.0.3"]
}
```

The individual instances are based on oVirt templates, which already include all platform specific settings and have docker installed:

```
resource "ovirt_vm" "control" {
  count = 3
  name = var.vm_control[count.index]
  clone = "false"
  high_availability = "true"
  cluster_id = var.cluster_id
  memory = var.vm_memory
  template_id = var.vm_template_id
  cores = var.vm_cpu_cores
  sockets = var.vm_cpu_sockets
  threads = var.vm_cpu_threads

  initialization {
    host_name = var.vm_control[count.index]
    dns_search = var.vm_dns_search
    dns_servers = var.vm_dns_servers
    custom_script = data.template_file.startup-script_control.rendered
    nic_configuration {
      label = var.vm_nic_device
      boot_proto = var.vm_nic_boot_proto
      address = var.vm_nic_ip_address_control[count.index]
      gateway = var.vm_nic_gateway
      netmask = var.vm_nic_netmask
      on_boot = var.vm_nic_on_boot
    }
  }
}
```

### Data

We need to pass the Rancher registration command into the startup script above, so we need a data template that allows us to substitute Terraform variables in local files:

```
data "template_file" "startup-script_control" {
  template = file("${path.module}/files/startup-script")
  vars = {
    registration_command = "${rancher2_cluster.cluster_tf.cluster_registration_token.0.node_command} --etcd --controlplane"
  }
  depends_on = [rancher2_cluster.cluster_tf]
}
```

The registration command will only be available after the successful creation of the cluster in Rancher, hence the dependency. The startup script itself is in YAML format and contains only one line; everything else has been defined in the oVirt template already:

```
runcmd:
- ${registration_command}
```

## Result

After terraform init / plan / apply, the resulting Kubernetes cluster will be fully provisioned on oVirt and available in Rancher.

Happy Ranching!

*(Last update: 3/14/20, cf)*
