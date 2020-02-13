GCP Cluster
==========

Rancher 2.x does not include a node driver for GCP, just a cluster driver for GKE, which should be sufficient for most needs. However, to build a Rancher Kubernetes cluster on GCP, we can use Terraform and the custom cluster setup.



Terraform directory setup
-------------------------

First step is to set up a new project (directory) for the GCP cluster:

```
-rw-r--r-- 1 cfrank ewscom  341 Jan 21 06:10 data.tf
drwxr-xr-x 2 cfrank ewscom 4096 Jan 19 06:48 files
-rw-r--r-- 1 cfrank ewscom 2076 Feb 11 09:25 main.tf
-rw-r--r-- 1 cfrank ewscom  278 Feb  8 04:46 output.tf
-rw-r--r-- 1 cfrank ewscom  319 Jan 21 07:38 provider.tf
-rw-r--r-- 1 cfrank ewscom  364 Jan 19 15:08 terraform.tf
-rw-r--r-- 1 cfrank ewscom  156 Feb 11 09:23 terraform.tfvars
-rw-r--r-- 1 cfrank ewscom  509 Feb  9 07:56 variables.tf
```

In the following, we'll exame the individual files - you can find the source on [GitHub]([https://github.com/chfrank-cgn/Rancher/tree/master/gcp-cluster](https://github.com/chfrank-cgn/Rancher/tree/master/gcp-cluster)



Result
------

After a terraform apply, the resulting Kubernetes cluster should look like this:

```
NAME           STATUS   ROLES                      AGE   VERSION   INTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
rke-1ebacc-0   Ready    controlplane,etcd,worker   43m   v1.15.9   10.240.0.80   Ubuntu 18.04.3 LTS   5.0.0-1029-gcp   docker://18.6.3
rke-1ebacc-1   Ready    controlplane,etcd,worker   43m   v1.15.9   10.240.0.78   Ubuntu 18.04.3 LTS   5.0.0-1029-gcp   docker://18.6.3
rke-1ebacc-2   Ready    controlplane,etcd,worker   43m   v1.15.9   10.240.0.81   Ubuntu 18.04.3 LTS   5.0.0-1029-gcp   docker://18.6.3
```



*(Last update: 2/13/20, cf)*


