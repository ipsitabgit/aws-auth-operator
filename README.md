# AWS Auth Operator

This is a custom K8s operator which helps manage updates to the "aws-auth" config map in AWS EKS.

## Architecture and Basic Flow

[Reference](TODO-DIAGRAM)

## Quick Start
(TODO : Build and push the image to Docker Hub)
(TODO : Helm chart doc)
- Execute `kubectl apply -f deploy-manifests/manifests.yaml`. This would deploy the following:
  - namespace
  - crd
  - service account
  - rbac resources
  - deployment
- Run `kubectl get po -n aws-auth-controller-system` , Should see the workload running
    
    ```
    NAME                                                     READY   STATUS    RESTARTS   AGE
    aws-auth-controller-controller-manager-55858f98b-ctfrs   2/2     Running   0          54m

    ```

- Once the controller starts running , execute `kubectl apply -f sample2.yaml` . This deploys the `EksAuthMap` custom resource 
- Check the logs of the controller and should see the reconcile operation

  ```
    2022/01/18 10:38:37 RECONCILING AWS AUTH CONTROLLER
    2022/01/18 10:38:37 false
    2022/01/18 10:38:37 RECONCILER: FETCHING EXISTING aws-auth CONFIGMAP
    2022/01/18 10:38:37 RECONCILER: Unmarshalling aws-auth CONFIGMAP
    2022/01/18 10:38:37 [{arn:aws:iam::849180847351:role/dummy-jet-eks-role1 dumy-jet-eks role [ADMIN] []} {arn:aws:iam::849180847351:role/dummy-jet-eks-role3 dumy-jet-eks-my3 role [ADMIN NODE] []} {arn:aws:iam::849180847351:role/dummy-jet-eks-role4 dumy-jet-eks-my4 role [READONLY] []} {arn:aws:iam::849180847351:role/dummy-jet-eks-role5 dumy-jet-eks-my5 role [NSADMIN] [ns1 ns2 ns3]} {arn:aws:iam::849180847351:user/galaxy-automation-user galaxy-automation user [NODE] []}]
    2022/01/18 10:38:37 RECONCILER: ITERATING THROUGH ALL THE RBAC CONFIGS
    2022/01/18 10:38:37 ARN: arn:aws:iam::849180847351:role/dummy-jet-eks-role1 ~ User: dumy-jet-eks
    2022/01/18 10:38:37 RECONCILER: VALIDATED GROUPS FROM ALLOWED LIST
    2022/01/18 10:38:37 RECONCILER: Adding provided roles to mapRoles
    2022/01/18 10:38:37 ARN: arn:aws:iam::849180847351:role/dummy-jet-eks-role3 ~ User: dumy-jet-eks-my3
    2022/01/18 10:38:37 RECONCILER: VALIDATED GROUPS FROM ALLOWED LIST
    2022/01/18 10:38:37 RECONCILER: VALIDATED GROUPS FROM ALLOWED LIST
    2022/01/18 10:38:37 RECONCILER: Adding provided roles to mapRoles
    .....
    2022/01/18 10:38:38 RECONCILER: Adding provided roles to mapRoles
    2022/01/18 10:38:38 ARN: arn:aws:iam::849180847351:user/galaxy-automation-user ~ User: galaxy-automation
    2022/01/18 10:38:38 RECONCILER: VALIDATED GROUPS FROM ALLOWED LIST
    2022/01/18 10:38:38 RECONCILER: Adding provided users to mapUsers
    2022/01/18 10:38:38 RECONCILER: Marshalling back mapRoles and mapUsers
    2022/01/18 10:38:38 RECONCILER: Updating aws-auth Config Map

  ```
- Check the configMap if its updated - `kubectl edit cm aws-auth -n kube-system`  

    
## EksAuthMap Custom Resource

This CR should be submitted for the Controller to reconcile.

| Schema | Description |
| ---- | --- |
| apiVersion | operators.apphosting.com/v1 |
| kind | EksRbac |
| spec.config | List of roles/users which needs additional RBAC |
| spec.config[0].arn | ARN of the role/user |
| spec.config[0].username | username to be assosciated with the config |
| spec.config[0].groups | List of groups.Supported Groups - `ADMIN`, `NODE`, `READONLY`, `NSADMIN` ,`WRITE`|
| spec.config[0].namespaces | List of namespaces which requires admin privileges |
| spec.config[0].type | Supported values : `role` , `user` |

### More about `groups`

`ADMIN` - This group essentially maps to `system:masters` which gives the complete administrative access to the cluster.

`NODE` - This group would map to `system:nodes` and `system:bootstrappers`

`READONLY` - This group gives a complete read access to the entire cluster. On giving this group, readonly ClusterRole and assosciated ClusterRoleBindings gets created.

`NSADMIN` - Users might just require administrative privilege on specific namespaces alone.  
You need to supply a list of namespaces where this access is needed. 
If none provided, then it would assume that access is only needed for `default` namespace. Additionally, this creates the namespaces provided, and also the assosciated Roles and Role Bindings for each namespace.

`WRITE` - This will allow users/roles to have the edit access on the K8s cluster resources.

## Developer Notes:

- Clone the repo locally
- Run the command - `make docker-build docker-push IMG=<YOUR_REPO>/rbac_op:<YOUR_TAG>`
- This would build and push the container image to the mentioned repository
- To deploy , execute - `make deploy IMG=<YOUR_REPO>/rbac_op:<YOUR_TAG>`
- This would deploy the operator to the cluster-context set at ~/.kube/config.

If you need to see the manifests that are being deployed , run:

`make gentest IMG=<YOUR_REPO>/rbac_op:<YOUR_TAG>`
   
#### Notes:
- Operator matches the given inputs with the already configured configMap and performs the necessary "inserts" or "updates".
- If no change is determined, no updates are made to the configMap.
- Currently the operator does not have the capability to delete an entry from the configMap.