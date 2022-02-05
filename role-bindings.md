# Role Bindings

In addition to creating infrastructure, we can use Terraform also to manage Rancher configuration items. In this case, we will use Terraform to create and maintain cluster and project role bindings for users on a given cluster.

The plans below use user role bindings but will work the same for groups when using an external authentication provider.

## For_Each

Terraform has a potent language element, [for_each](https://www.terraform.io/language/meta-arguments/for_each), which we will use to read the role bindings from a .csv file.

The combination of for_each and a unique ID in the first column of the .csv file will allow us to use Terraform plans for all CRUD operations on the role bindings.

## Provider

First, we define the [Rancher2 provider](https://registry.terraform.io/providers/rancher/rancher2/latest/docs):

```
provider "rancher2" {
  api_url = var.rancher-url
  token_key = var.rancher-token
}
```

and create a [data source](https://registry.terraform.io/providers/rancher/rancher2/latest/docs/data-sources/cluster) for the downstream Kubernetes Cluster:

```
data "rancher2_cluster" "rbac_cluster" {
  name = var.rbac-cluster
}
```

## Cluster Role Bindings

We set up a data file with the user names and available cluster roles from the Rancher UI:

```
id,user,role
cr01,user1,nodes-view
cr02,user2,cluster-owner
```

Please note the unique ID in the first column - this is needed for Terraform to identify and act on changes.

We then define the data file in our plan:

```
locals {
  cluster_roles = csvdecode(file(var.rbac-cr-file))
}
```

We read the user names in the first loop and obtain the corresponding user ids:

```
data "rancher2_user" "rbac_user" {
  for_each = { for inst in local.cluster_roles : inst.id => inst }

  username = each.value.user
}
```

In the next and final loop, we bind the roles to the users:

```
resource "rancher2_cluster_role_template_binding" "plan_cl_role" {
  name = "cr-${each.key}"
  for_each = { for inst in local.cluster_roles : inst.id => inst }

  cluster_id = data.rancher2_cluster.rbac_cluster.id
  role_template_id = each.value.role
  user_id = data.rancher2_user.rbac_user[each.key].id
}
```

After terraform init / plan / apply, the users will have the cluster roles assigned or modified as defined in the data file.

## Project Role Bindings

To achieve the same for project roles bindings, we need an extra column in our data file:

```
id,project,user,role
pr01,PROJ,user1,project-member
pr02,PROJ,user2,project-owner
```

and an extra loop to obtain the corresponding project ids:

```
data "rancher2_project" "rbac_project" {
  for_each = { for inst in local.project_roles : inst.id => inst }

  cluster_id = data.rancher2_cluster.rbac_cluster.id
  name = each.value.project
}
```

You can find sample plan files for the user and project role bindings on my [GitHub](https://github.com/chfrank-cgn/Rancher/tree/master/role-binding). 

Happy Ranching!

*(Last update: 2/5/22, cf)*
