# GCP Cluster

Rancher 2.x does not include a node driver for GCP, just a cluster driver for GKE, which should be more than sufficient for most needs. However, to build a Rancher Kubernetes cluster on GCP, we can use [Terraform](https://www.terraform.io/) and a [custom node](https://rancher.com/docs/rancher/v2.x/en/cluster-provisioning/rke-clusters/custom-nodes/) setup.

For Terraform to do this, we need to complete a couple of steps.

## Terraform directory setup

First step is to set up a new terraform environment (i.e. a directory) for the GCP cluster, in which we create the plan files (i.e. the desired state definitions):

```
-rw-r--r-- 1 cfrank ewscom  341 Jan 21 06:10 data.tf
drwxr-xr-x 2 cfrank ewscom 4096 Jan 19 06:48 files
-rw-r--r-- 1 cfrank ewscom 2076 Feb 11 09:25 main.tf
-rw-r--r-- 1 cfrank ewscom  278 Feb  8 04:46 output.tf
-rw-r--r-- 1 cfrank ewscom  319 Jan 21 07:38 provider.tf
-rw-r--r-- 1 cfrank ewscom  364 Jan 19 15:08 terraform.tf
-rw-r--r-- 1 cfrank ewscom  509 Feb  9 07:56 variables.tf
```

The file naming follows the convention as proposed by HashiCorp, but you're free to use your own naming, all files with a  .tf ending will be examined. As we go along, we'll look at all the individual files - this Terraform plan has been successfully executed multiple times and you can find the complete source code on [GitHub](https://github.com/chfrank-cgn/Rancher/tree/master/gcp-cluster).

Terraform is a state-based infrastructure orchestration tool, for storage of the actual plan state we'll use the [Terraform cloud](https://www.hashicorp.com/blog/announcing-terraform-cloud/):

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

The Terraform cloud not only allows remote storage of plan states, but also offers integration into major source code revision tools, allwoing you to trigger plan execution with a simple commit.

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

The next plan definitions are for the actual cluster resources - the cluster itself, its name and its instances.

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

For the cluster, we need a number of compute instances; Terraform 0.12 brought new control elements, one of which is an easy way to define multiple resources of the same type (count):

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

## Data

We need to pass the Rancher registration command into the startup script above, so we need a data template which allows us to substitute Terraform variables in local files:

```
data "template_file" "startup-script_data" {
  template = file("${path.module}/files/startup-script")
  vars = {
    registration_command = "${rancher2_cluster.cluster_gcp.cluster_registration_token.0.node_command} --etcd --controlplane --worker"
  }
  depends_on = [rancher2_cluster.cluster_gcp]
}
```

The registration command will only be available after successful creation of the cluster in Rancher, hence the dependency.

## Variables

Certain values, such as the Kubernetes version to use or the number of nodes, are defined as variables, to make overall code maintenance easier:

```
variable "k8version" {
  default = "v1.15.9-rancher1-1"
}
variable "numnodes" {
    default = 3
}
```

## Output

And last, but not least, to facilitate command line ssh access, we show the external IP addresses upon successful completion:

```
output "Public" {  
  value = "${google_compute_instance.vm_gcp.*.network_interface.0.access_config.0.nat_ip}"
}
```

Result
------

After terraform init / plan / apply, the resulting Kubernetes cluster will be fully provisioned in Rancher and look like this in kubectl:

```
NAME           STATUS   ROLES                      AGE   VERSION   INTERNAL-IP   OS-IMAGE
rke-1ebacc-0   Ready    controlplane,etcd,worker   43m   v1.15.9   10.240.0.80   Ubuntu 18.04.3 LTS
rke-1ebacc-1   Ready    controlplane,etcd,worker   43m   v1.15.9   10.240.0.78   Ubuntu 18.04.3 LTS
rke-1ebacc-2   Ready    controlplane,etcd,worker   43m   v1.15.9   10.240.0.81   Ubuntu 18.04.3 LTS
```

Happy Ranching!

*(Last update: 2/23/20, cf)*
