# oVirt

[oVirt](https://www.ovirt.org/) is the open-source community project behind [Red Hat Virtualization](https://www.redhat.com/en/technologies/virtualization/enterprise-virtualization) and well-integrated into [Red Hat OpenShift](https://www.openshift.com/). oVirt integration into Kubernetes itself has been around for some time, too, as a [cloud provider](https://rancher.com/docs/rancher/v2.x/en/cluster-provisioning/rke-clusters/options/cloud-providers/). 

Together with my colleague [Alex Behrendt](mailto:alexander.behrendt@cbc.de) we attempted to enable the oVirt cloud provider on a Rancher / oVirt in-house container run-time platform; details on how to integrate oVirt with OKD can be found [here](https://blogs.ovirt.org/2019/01/ovirt-openshift-part-1/).

## oVirt Cloud Provider

An example configuration for the cloud provider can be found in the Rancher documentation above, and editing the cluster definition led to the correct cloud-config file:

```
[connection]
uri = https://ovirt.company.com/ovirt-engine/api
username = user@company.com
password = password
[filters]
# Search query used to find nodes
vms = tag=cow
```

With this information, Kubernetes should get access to the oVirt API to import node resources. Also, the correct option was passed to the controller:

```
kube-controller-manager --cloud-config=/etc/kubernetes/cloud-config ... --cloud-provider=ovirt ...
```

Alas, the cluster didn't start. The log file showed the following error message:

```
controllermanager.go:230] error building controller context: cloud provider could not be initialized: unknown cloud provider "ovirt"
```

Where did we go wrong?

## Deprecated

It turns out we didn't go wrong. As of Kubernetes 1.16, the oVirt Cloud Provider is no longer included with Kubernetes ([Pull request #72178](https://github.com/kubernetes/kubernetes/pull/72178), [commit 1ca80d3](https://github.com/kubernetes/kubernetes/commit/1ca80d3b32f95d893e0b152c3ac2fcbc460982dc)) and an enhancement request to reintroduce the [oVirt out-of-tree cloud provider](https://github.com/kubernetes/enhancements/issues/673) was recently closed for Kubernetes 1.18. 

For the time being, there seems to be no direct integration between Kubernetes and oVirt.

*(Last update: 3/13/20, cf)*


