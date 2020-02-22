# OpenStack

[OpenStack](https://www.openstack.org/) is still a viable cloud operating system and in use by many ISPs. The setup in this article was performed on a teutoStack public cloud environment, operated by [teuto.net](https://teuto.net/) in [Bielefeld](https://en.wikipedia.org/wiki/Bielefeld_Conspiracy).

OpenStack integration for Kubernetes itself has been around for some time and consists of two components: The OpenStack [cloud provider](https://rancher.com/docs/rancher/v2.x/en/cluster-provisioning/rke-clusters/options/cloud-providers/) and the OpenStack [node driver](https://rancher.com/docs/rancher/v2.x/en/admin-settings/drivers/node-drivers/). The cloud provider is available in Rancher by default; Rancher also includes a node driver. However, that's not enabled by default. 

To build a Rancher Kubernetes cluster on OpenStack, there are two options: The OpenStack node driver or a [custom cluster](https://rancher.com/docs/rancher/v2.x/en/cluster-provisioning/rke-clusters/custom-nodes/) setup.

For easier access, all example files below are available on [GitHub](https://github.com/chfrank-cgn/Rancher/tree/master/openstack).

## Cloud Provider

To enable the OpenStack cloud provider, choose the "Custom" option during cluster creation for the cloud provider in the Rancher GUI and then add the following information into the cluster configuration (Edit YAML) as an example - substitute actual values as needed:

```
rancher_kubernetes_engine_config:
...
  cloud_provider:
    name: "openstack"
    openstackCloudProvider: 
      block_storage: 
        ignore-volume-az: true
        trust-device-path: false
        bs-version: "v2"
      global: 
        auth-url: "https://api.openstack.net:5000/v3" # Keystone Auth URL
        domain-name: "Default" # Identity v3 Domain Name
        tenant-id: "616a8b01b5f94f99acd00a844f8f46c3" # Project ID
        username: "user" # OpenStack Username
        password: "pass" # OpenStack Password
      load_balancer:
        lb-version: "v2"
        subnet-id: "f339e543-a67f-45fa-8157-4a58b0940e0b"
        floating-network-id: "ca27ca05-2870-47b3-ad2f-535d04c9e736"
        create-monitor: false
        manage-security-groups: true
        monitor-max-retries: 0
        use-octavia: true
      metadata: 
        request-timeout: 0
  ignore_docker_version: false
  ...
```

With this information, Kubernetes gets access to the OpenStack API, to create and delete resources, and also access to Cinder volumes and the Octavia load balancer. Without this configuration, the cluster would still work fine, just without any access to Cinder or Octavia.

## Node Driver

To create a Kubernetes cluster on OpenStack with the built-in node driver, the driver needs to be enabled in Rancher's node driver configuration page, and then the following information entered in a node template, again with example values - substitute actual values as needed:

```
"authUrl": "https://api.openstack.net:5000/v3",
"availabilityZone": "Zone1",
"domainName": "Default",
"flavorName": "standard.2.1905",
"floatingipPool": "extern",
"imageName": "ubuntu-18.04-bionic-amd64",
"keypairName": "rancher",
"netName": "intern",
"sshPort": "22",
"sshUser": "ubuntu",
"tenantId": "616a8b01b5f94f99acd00a844f8f46c3",
"username": "user"
```

Afterward, cluster creation is straightforward, as with all other cloud providers

### Security Groups

To enable automatic cluster setup, the following [firewall rules](https://rancher.com/docs/rancher/v2.x/en/installation/requirements/ports/) need to be defined between Rancher and the OpenStack tenant:

- ssh, http and https in both directons
- 2376 (docker) from Rancher to the tenant nodes
- 2376, 2379, 2380, 6443, and 10250 between the tenant nodes

## Custom Cluster

Alternatively, the cluster can be built from individually created instances and the help of a startup script, to install and enable docker:

```
#!/bin/sh
apt-get update
apt-get -y install apt-transport-https jq software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
apt-get update
apt-get -y install docker-ce=18.06.3~ce~3-0~ubuntu
usermod -G docker -a ubuntu
exit 0
```

### Security Groups

To enable cluster creation from existing nodes, the following firewall rules need to be defined for the OpenStack tenant:

- ssh from a workstation
- http and https to Rancher

## Cinder

For access to Cinder block storage, apply the following storage class definition:

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cinder
provisioner: kubernetes.io/cinder
reclaimPolicy: Delete
parameters:
  availability: nova
```

No further action is needed for the OpenStack load balancer.

Happy Stacking!

*(Last update: 2/22/20, cf)*
