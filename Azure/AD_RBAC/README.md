# RBAC, Kubernetes and Azure Active Directory

This document will provide some guidance on how to implement RBAC with Kubernetes with Azure Active Directory (AAD) in a secure fashion and adhering to a few security First Principles, namely:

* Apply least privileged access
* Segregation of responsibility
* Minimise attack surface
* Apply security in a layered approac

[Azure Active Directory](https://azure.microsoft.com/en-us/services/active-directory/)(AAD) helps you manage user identities and create intelligence-driven access policies to secure your resources. By managing Kubernetes users in AAD, maintenance is simplified as all users are managed centrally, and users have a single identity with which to access all services, not just Kubernetes. 

**RBAC in Kubernetes may be thought of as the following:**

![RBAC](https://github.com/shanepeckham/AKS_Security/blob/master/Images/Snip20180709_3.png)

For example, can user devops view pods in namespace devops?

## To manage RBAC in Kubernetes, apart from resources and operations, we need the following elements:

From [Bitnami](https://docs.bitnami.com/kubernetes/how-to/configure-rbac-in-your-kubernetes-cluster/)

* **Roles and ClusterRoles**: Both consist of rules. The difference between a Role and a ClusterRole is the scope: in a Role, the rules are applicable to a single namespace, whereas a ClusterRole is cluster-wide, so the rules are applicable to more than one namespace. ClusterRoles can define rules for cluster-scoped resources (such as nodes) as well. Both Roles and ClusterRoles are mapped as API Resources inside our cluster.

* **Subjects**: These correspond to the entity that attempts an operation in the cluster. There are three types of subjects:
* **User Accounts**: These are global, and meant for humans or processes living outside the cluster. There is no associated resource API Object in the Kubernetes cluster.
* **Service Accounts**: This kind of account is namespaced and meant for intra-cluster processes running inside pods, which want to authenticate against the API.
* **RoleBindings and ClusterRoleBindings**: Just as the names imply, these bind subjects to roles (i.e. the operations a given user can perform). As for Roles and ClusterRoles, the difference lies in the scope: a RoleBinding will make the rules effective inside a namespace, whereas a ClusterRoleBinding will make the rules effective in all namespaces.
You can find examples of each API element in the Kubernetes official documentation.

## Users in Kubernetes from [Kubernetes docs](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens)

* All Kubernetes clusters have two categories of users: service accounts managed by Kubernetes, and normal users.

* Normal users are assumed to be managed by an outside, independent service, in this case AAD. Kubernetes does not have objects which represent normal user accounts. Regular users cannot be added to a cluster through an API call.

* In contrast, service accounts are users managed by the Kubernetes API. They are bound to specific namespaces, and created automatically by the API server or manually through API calls. Service accounts are tied to a set of credentials stored as Secrets, which are mounted into pods allowing in-cluster processes to talk to the Kubernetes API.


## AAD Groups

* AAD Groups are containers that contain user and computer objects within them as members. We can use these groups to bind to Role and Cluster Role Bindings and then simply add users as members to Groups. 

**The following diagram illustrates conceptually how AAD groups fit into RBAC with Kubernetes:**

![Flow](https://github.com/shanepeckham/AKS_Security/blob/master/Images/Snip20180710_11.png)

We map 1:M Roles to RoleBindings (or Cluster Roles and Cluster Role Bindings for cluster wide scope), we bind 1:M RoleBindings to an AAD Group, and we associate 1:M Users with an AAD Group. In the case of ServiceAccounts which are managed exclusively within Kubernetes, we can associate these 1:1 indirectly with an AAD Group, but we will cover this later.

### Apply Least Priviliged Access

We should always start from a position of applying the least permissions or priviliges to an account, thus in the context of Kubernetes for example, the minimum permissions would be the ability to view an object, such as Pods, for a single Namespace. This does not include watching or spooling logs, merely being able to list Pods and see their status. 

We would then create an AAD Group that represents this minimum privilege and apply a RoleBinding to this AAD Group. We then simply need to add all new users to this AAD Group until they need more privileges.

Determining which Roles are required for operations within Kubernetes resources can be quite a time consuming task, and a typical approach to trace missing permissions is to enable the [Audit Policy](https://kubernetes.io/docs/tasks/debug-application-cluster/audit/) via an Admission Controller and then user Jordan Ligget's tool [audit2RBAC](https://kubernetes.io/docs/tasks/debug-application-cluster/audit/) to help quickly identify missing permissions. This is unfortunately not possible in AKS at the moment but can be used within [ACS-Engine](https://github.com/Azure/acs-engine).

### Implementing Least Priviliged Access

**The diagram below illustrates how we could map AAD Groups to a layered RBAC permissions approach**:

![Layered](https://github.com/shanepeckham/AKS_Security/blob/master/Images/Snip20180710_7.png)

This is however a simplified approach as it does not implement restrictions on resources that may be accessed. In reality a user's access is defined by the intersection of three components, that of Scope e.g. Cluster or Namespace, Resources e.g. Pods, Services and Verbs, e.g. Create, Exec, List.

Thus a more secure approach would be to introduce layers of resource restrictions on this model. Clearly a balance needs to be found between maintenance and control, thus it may make sense to aggregate typical resources that are required to deploy solutions on Kubernetes. We do not want to create too many AAD Groups, or rather we do not want users to be members of too many AAD Groups as these are reflected in the token and can increase login times.

**Thus it is viable to create many AAD Groups representing Resource, Scope and Verb permissions, but care should then be taken to ensure that the user is added to the least amount of AAD Groups to enable those permissions.**

The following diagram illustrates adding only the necessary AAD Groups with permissions to the three users to achieve the required access:

![Minimal](https://github.com/shanepeckham/AKS_Security/blob/master/Images/Snip20180710_9.png)

Here we can see that user devops is only a member of AAD Group ClusterCreateAdmin, user devopsbot is a member of two groups ClusterViewOps and NS1CreateDev and devopsbot2 is a member of NS1CreateDev. 

It is of course feasible to only bind cluster scoped roles to AAD Groups to really minimize the number of groups but there is of course a trade off of granular control.

## Enable RBAC on cluster with AD integration

Follow initial setup as documented [here](https://docs.microsoft.com/en-us/azure/aks/aad-integration) or for an automated setup look at the [Apps Script](https://github.com/shanepeckham/AKS_Security/blob/master/Azure/AD_RBAC/AADApps/README.md)

## Apply RBAC to Azure AD users and groups

### Log on to Kubernetes as the AAD admin

Run 
```
az aks get-credentials --name [clustername] --resource-group [resourcegroup] --admin
```

## Set Up Sample Least Privileged Access RBAC

1. Amend and run the RBAC.sh script to include your domain in section three as your AAD administrator

This script will create two users:

* devopsbot
* devopsbot2

And the following AD Groups:

* ClusterViewAdmin
* ClusterViewDev
* ClusterViewOps
* ClusterCreateAdmin
* ClusterCreateDev
* ClusterCreateOps
* NS1ViewAdmin
* NS1ViewDev
* NS1ViewOps
* NS1CreateAdmin
* NS1CreateDev
* NS1CreateOps

2. Add the users to your subscription

3. Create the Roles and Bindings

Run the following command:

```
kubectl create -f https://github.com/shanepeckham/AKS_Security/tree/master/Azure/AD_RBAC
```
 
### Log in as devopsbot

As user devopsbot connect to AKS with kubectl

```az aks get-credentials --name devopsaksad --resource-group devopsakdsad ```

When you try to run a kubectl command you will be presented with the following:

```To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code [code] to authenticate```

You will then be directed to an OAUTH permission request, see below:

![Permissions](https://github.com/shanepeckham/AKS_Security/blob/master/Images/Snip20180627_2.png)

![Grant client AD application access](https://github.com/shanepeckham/AKS_Security/blob/master/Images/Snip20180627_3.png)

If you try to run ```kubectl get nodes -n devops``` you will receive the following message:

```
You must be logged in to the server (Unauthorized)
```

### Remove the ability to run az aks get credentials

You will need to remove this role permission from the user otherwise there is nothing stopping any developer from simply running 
```
az aks get-credentials --name [cluster] -resource-group [group] --admin
```
and getting full administration access to the cluster.

Instead, follow these steps:

Clone the role relevant to your cluster users and amend the action for the provider Microsoft.ContainerService. Remove permitted action “Microsoft.ContainerService/managedClusters/accessProfiles/read” - see below:

![Amend action](https://github.com/shanepeckham/AKS_Security/blob/master/Images/Snip20180711_14.png)

It does mean that the user will need to be provided with a kubeconfig file to work with so they can get a new token.

Run the following script:

```
az role definition create --role-definition '{
    "Name": "Custom Container Service",
    "Description": "Cannot read Container service credentials",
    "Actions": [
        "Microsoft.ContainerService/managedClusters/read"
    ],
    "DataActions": [
    ],
    "NotDataActions": [
    ],
    "AssignableScopes": [
        "/subscriptions/[xxx-xxx-xxx-xxx-xxx-xxx-xxx-xxx]"
    ]
}'
```
Add the custom role to the user:

![custom role](https://github.com/shanepeckham/AKS_Security/blob/master/Images/Snip20180711_12.png)


When you try to get credentials you will see this:

![fail](https://github.com/shanepeckham/AKS_Security/blob/master/Images/Snip20180711_13.png)



**Now run the script RBACConfig1.sh

Now if user devopsbot issues the following command, it should be successful:

```
kubectl get pods -n devops
```
```
kubectl get pods --all-namespaces
```
```
kubectl auth can-i get pods
```

The last command should return 'yes'

![yes](https://github.com/shanepeckham/AKS_Security/blob/master/Images/Snip20180628_4.png)

However, if the user devops bot tries to issue a watch on the pods:

```kubectl get pods -n devops -w```

You will receive the following:

```Error from server (Forbidden): unknown (get pods)```

### Grant DevOpsBot namespace scoped list permissions by binding roles to the group (for create, update, watch pods)

```
az ad group member add --group K8DevOpsEdit --member-id [objectId]                  
```   

### Add ClusterRoles and ClusterRoleBindings to view pods and podlogs and assign to group K8Cluster-View

```
kubectl create -f https://raw.githubusercontent.com/shanepeckham/AKS_Security/master/Sample%20Implementation/Roles%20and%20RoleBindings/New/K8ClusterView.yaml
```



Create the cluster role for the to allow the granting of roles at the cluster level. Anyone added to this group will be allowed to grant roles at the same scope to others





### Service Accounts

[Service Accounts](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/) can be used to run Pods and are native to Kubernetes and as such they cannot be managed within AAD. If no Service Account is specified when a Pod is run, default Service Account for that Namespace is used - note Service Accounts are scoped to a Namespace. 
If there is a requirement to control Service Account Priviliges as granularly as with User Accounts, then the following approach can be considered. 

As with mapping Roles and RoleBindings to an AAD Group, we can use the same mechanism and indeed YAML file to map to a Service Account. By doing this, we are in fact using a Service Account as an internal to Kubernetes representation of a Service Account. This means that although we are creating many Service Accounts, their permission profiles look exactly the same as the AAD Groups that we currently use, see below for an example:

```
# This role binding allows members in group DevOpsGroup to pod-list-podlogs-list in the "devops" namespace.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: ns1ViewAdmin
  namespace: ns1
subjects:
- kind: Group
  name: "ns1ViewAdmin" # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: ns1ViewAdmin
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: ns1ViewAdmin # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```


### Aggregated Roles

ClusterRoles can be created by combining other ClusterRoles using an aggregationRule. The permissions of aggregated ClusterRoles are controller-managed, and filled in by unioning the rules of any ClusterRole that matches the provided label selector. An example aggregated ClusterRole:
```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: [] # Rules are automatically filled in by the controller manager.
Creating a ClusterRole that matches the label selector will add rules to the aggregated ClusterRole. In this case rules can be added to the “monitoring” ClusterRole by creating another ClusterRole that has the label rbac.example.com/aggregate-to-monitoring: true.

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: monitoring-endpoints
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
# These rules will be added to the "monitoring" role.
rules:
- apiGroups: [""]
  resources: ["services", "endpoints", "pods"]
  verbs: ["get", "list", "watch"]
```
