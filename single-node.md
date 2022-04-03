# Single-Node Rancher

SUSE Rancher offers a convenient docker image for a quick and well-documented [single-node installation](https://rancher.com/docs/rancher/v2.6/en/installation/other-installation-methods/single-node-docker/). This image includes an embedded k3s Kubernetes cluster to run Rancher and is well-suited for fast, on-the-fly installations.

However, there are situations where you might need a bit more control over the setup without sacrificing the speed and ease of installation. The following design was inspired by the blog post ["Powerful Single Node RKE2 on Hetzner"](https://blog.alphabravo.io/posts/2021/single-node-rke2-pt1/) by the [AB Engineering Team](mailto:devops@alphabravo.io) and will use RKE as the Kubernetes distribution to install Rancher.

Our goal is to create a repeatable process creating a fresh Rancher instance every time.

## Preparations

To prepare, we create a modest VM with a current flavor of Linux on a (cloud) provider of our choice and install Docker.

We copy [RKE](https://github.com/rancher/rke/releases) and [Helm](https://github.com/helm/helm/releases) binaries to a directory of our choice, e.g., to /usr/local/bin/.

We create an RKE [config file](https://rancher.com/docs/rke/latest/en/installation/) suitable for the chosen VM as the final preparation step, e.g., in ~/rancher.

## Leave No Trace

We want to ensure that each start of RKE and Rancher is entirely fresh and new, so we want to make all data ephemeral - quite the opposite of typical a Rancher (HA) installation.

To do this, we create temporary file systems for Docker, containerd, and Rancher:

```
tmpfs	/var/lib/docker     tmpfs	defaults	1	2
tmpfs	/var/lib/rancher    tmpfs	defaults	1	2
tmpfs	/var/lib/containerd tmpfs	defaults	1	2
```

and make sure that we have etcd-Snapshotting disabled in the cluster configuration:

```
services:
  etcd:
    snapshot: false
```

After a reboot, there will be no data from a previous run.

## Start Rancher

To start Rancher, we follow the steps for a [Rancher installation](https://rancher.com/docs/rancher/v2.6/en/installation/install-rancher-on-k8s/):

In the directory with the RKE configuration, e.g., rancher, we execute:

```
cd rancher
rke up
cd ..
```

Once our Kubernetes cluster becomes available, we add the necessary Helm repos:

```
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

and create the namespaces:

```
kubectl --kubeconfig=rancher/kube_config_cluster.yml create namespace cattle-system
kubectl --kubeconfig=rancher/kube_config_cluster.yml create namespace cert-manager
```

Next, we install cert-manager and wait for it to finish:

```
kubectl --kubeconfig=rancher/kube_config_cluster.yml apply -f \
  https://github.com/jetstack/cert-manager/releases/download/v1.5.1/cert-manager.crds.yaml

helm --kubeconfig=rancher/kube_config_cluster.yml install cert-manager \
  jetstack/cert-manager --namespace cert-manager --version v1.5.1

kubectl --kubeconfig=rancher/kube_config_cluster.yml -n cert-manager \
  rollout status deploy/cert-manager
```

As the final step, we install Rancher:

```
helm --kubeconfig=rancher/kube_config_cluster.yml \
    install rancher rancher-stable/rancher \
    --namespace cattle-system \
    --set hostname=rancher.host.name \
    --set bootstrapPassword="SuperSecret" \
    --set replicas=1 

kubectl --kubeconfig=rancher/kube_config_cluster.yml -n cattle-system \
  rollout status deploy/rancher
```

We'll give Rancher some time to settle after the deployment finishes and then log in to the GUI.

## Stop Rancher

Once we're finished with our Rancher instance, we can shut Kubernetes down:

```
cd rancher
rke remove --force
cd ..
```

and then our host:

```
shutdown -h now
```

We could turn these steps into a set of shell scripts, Ansible playbooks, or Terraform plans to automate.

Happy Ranching!

*(Last update: 4/3/22, cf)*
