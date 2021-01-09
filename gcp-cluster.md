# GCP Cluster

Rancher 2.x does not include a node driver for GCP, just a cluster driver for GKE, which should be more than sufficient for most needs. However, to build a Rancher Kubernetes cluster on GCP, we can use [Terraform](https://www.terraform.io/) and a [custom node](https://rancher.com/docs/rancher/v2.x/en/cluster-provisioning/rke-clusters/custom-nodes/) setup.

For Terraform to do this, we need to complete a couple of steps.

## Terraform directory setup

The first step is to set up a new terraform environment (i.e., a directory) for the GCP cluster, in which we create the plan files (i.e., the desired state definitions):

```
-rw-r--r-- 1 cfrank ewscom  341 Jan 21  2020 data.tf
drwxr-xr-x 2 cfrank ewscom 4096 Dec  9 09:03 files
-rw-r--r-- 1 cfrank ewscom 3053 Dec  9 09:03 main.tf
-rw-r--r-- 1 cfrank ewscom  268 Dec  9 09:07 output.tf
-rw-r--r-- 1 cfrank ewscom  319 Jan 21  2020 provider.tf
-rw-r--r-- 1 cfrank ewscom  687 Jan  9 08:30 terraform.tf
-rw-r--r-- 1 cfrank ewscom  637 Jan  9 08:28 variables.tf
-rw-r--r-- 1 cfrank ewscom   45 Jan  9 08:32 versions.tf
```

The file naming follows the convention proposed by HashiCorp, but you're free to use your naming scheme, all files with a  .tf ending will be examined. As we go along, we'll look at all the individual files - this Terraform plan has successfully been executed multiple times, and you can find the complete source code on [GitHub](https://github.com/chfrank-cgn/Rancher/tree/master/gcp-cluster).

Terraform is a state-based infrastructure orchestration tool; for storage of the actual plan state, we'll use the [Terraform cloud](https://www.hashicorp.com/blog/announcing-terraform-cloud/):

```
terraform {
  backend "remote" {
    organization = "xxxxxxxxxxx"
    workspaces {
      name = "compute-prod-us-central"
    }
  }
}
```

The Terraform cloud allows remote storage of plan states and offers integration into the main source code revision tools, allowing you to trigger plan execution with a simple commit

## Provider

To set up a custom Rancher cluster on GCE, we need to create two providers, a [Google Cloud Platform provider](https://www.terraform.io/docs/providers/google/index.html) for the infrastructure:

```
provider "google" {
  project     = "xxxxxxx-xxxxxx-xxxxxx"
  credentials = file("xxxxxxxxxxxx.json")
  region      = "us-central1"
  zone        = "us-central1-c"
}
```

and a [Rancher2 provider](https://www.terraform.io/docs/providers/rancher2/index.html) for Kubernetes:

```
provider "rancher2" {
  api_url = var.rancher-url
  token_key = var.rancher-token
  insecure = true
}
```

## Main

The next plan definitions are for the actual cluster resources - the cluster itself, its name, and its instances.

### Random ID

To have a unique naming identifier for cluster and nodes, we use Random:

```
resource "random_id" "instance_id" {
 byte_length = 3
}
```

### Cluster

We define the Rancher cluster, using the name from above and set Kubernetes networking and version:

```
resource "rancher2_cluster" "cluster_gcp" {
  name         = "gcp-${random_id.instance_id.hex}"
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

### Instances

For the cluster, we need several compute instances; Terraform 0.12 brought new control elements, one of which is an easy way to define multiple resources of the same type (count):

```
resource "google_compute_instance" "vm_gcp" {
  name         = "rke-${random_id.instance_id.hex}-${count.index}"
  machine_type = var.type
  count = var.numnodes

  boot_disk {
    initialize_params {
      image = var.image
      size = var.disksize
    }
  }
  metadata_startup_script = data.template_file.startup-script_data.rendered
}
```

### Data

We need to pass the Rancher registration command into the startup script above, so we need a data template that allows us to substitute Terraform variables in local files:

```
data "template_file" "startup-script_data" {
  template = file("${path.module}/files/startup-script")
  vars = {
    registration_command = "${rancher2_cluster.cluster_gcp.cluster_registration_token.0.node_command} --etcd --controlplane --worker"
  }
  depends_on = [rancher2_cluster.cluster_gcp]
}
```

The registration command will only be available after successfully creating the cluster in Rancher, hence the dependency.

### Variables

Specific values, such as the Kubernetes version to use or the number of nodes, are defined as variables to make overall code maintenance easier:

```
variable "k8version" {
  default = "v1.18.14-rancher1-1"
}
variable "numnodes" {
    default = 3
}
```

### Cluster sync

We're almost ready; just let's wait for the cluster to become active, using a timer:

```
resource "null_resource" "before" {
  depends_on = [rancher2_cluster.cluster_gcp]
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
resource "rancher2_app_v2" "syslog_gcp" {
  lifecycle {
    ignore_changes = all
  }
  cluster_id = rancher2_cluster.cluster_gcp.id
  name = "rancher-logging"
  namespace = "cattle-logging-system"
  repo_name = "rancher-charts"
  chart_name = "rancher-logging"
  chart_version = var.logchart

  depends_on = [google_compute_instance.vm_gcp]
}
```

For the new v2 app resources, it can be beneficial to add a dependency to the control plane to ensure that the cluster is still accessible while the app resources are being destroyed.

### Output

Last but not least, to facilitate command-line ssh access, we show the external IP addresses upon successful completion:

```
output "Public" {  
  value = "${google_compute_instance.vm_gcp.*.network_interface.0.access_config.0.nat_ip}"
}
```

## Result

After a terraform init / plan / apply, the resulting Kubernetes cluster will be fully provisioned in Rancher and look like this in kubectl:

```
NAME           STATUS   ROLES                      AGE   VERSION
rke-1dec3f-0   Ready    controlplane,etcd,worker   45m   v1.18.14
rke-1dec3f-1   Ready    controlplane,etcd,worker   45m   v1.18.14
rke-1dec3f-2   Ready    controlplane,etcd,worker   45m   v1.18.14
```

## Validation

To validate a successful build, I usually enable Rancher's monitoring app and deploy the "Hello World" of Kubernetes, a WordPress instance, from the Rancher catalog.

Never run a Kubernetes cluster without monitoring or logging!

## Troubleshooting

The best place for troubleshooting during plan execution is the pod running Rancher's output - it provides detailed information on what Rancher is currently doing and complete error messages if something goes wrong.

Happy Ranching!

*(Last update: 1/9/21, cf)*
