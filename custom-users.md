## How to Create Customer users and privilages

```
htpasswd -c  -Bb htpasswd.txt ocadmin ocadmin
htpasswd -Bb htpasswd.txt cpdadmin cpdadmin
htpasswd -Bb htpasswd.txt laxman laxman

cat  htpasswd.txt 
ocadmin:$2y$05$b0sP7oBqyOCxEnNKcmtH4O/9KeSzozZCXOze3Z41byw/Ql10CY0om
cpdadmin:$2y$05$fmP2081PNIUG97gjTuUjeeLcuXJc16mZKJpZ16drng6z.gozAQJaS
laxman:$2y$05$rrrJ.axydROSjav7ujLB5OFVQSfejbaUs3j9xYxFpBPaaWQiTvDFK

oc create secret generic htpass-secret --from-file=htpasswd=htpasswd.txt --dry-run=client -o yaml -n openshift-config | oc replace -f -


```

## OpenShift Cluster admin
oc adm policy add-cluster-role-to-user cluster-admin ocadmin

## Project admin
 oc adm policy add-role-to-user cluster-admin cpdadmin -n cpd
 oc adm policy add-role-to-user cluster-admin cpdadmin -n cpd-common-services
 

###  Check Status

```
oc get rolebindings -n cpd-common-services
NAME                    ROLE                               AGE
admin                   ClusterRole/admin                  69m
cluster-admin           ClusterRole/cluster-admin          86s
system:deployers        ClusterRole/system:deployer        69m
system:image-builders   ClusterRole/system:image-builder   69m
system:image-pullers    ClusterRole/system:image-puller    69m

```
```
# oc get rolebindings -n cpd
NAME                    ROLE                               AGE
admin                   ClusterRole/admin                  70m
cluster-admin           ClusterRole/cluster-admin          6m33s
system:deployers        ClusterRole/system:deployer        70m
system:image-builders   ClusterRole/system:image-builder   70m
system:image-pullers    ClusterRole/system:image-puller    70m
```

```
# oc describe rolebinding.rbac -n cpd
Name:         admin
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  ClusterRole
  Name:  admin
Subjects:
  Kind  Name     Namespace
  ----  ----     ---------
  User  ocadmin  


Name:         cluster-admin
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  ClusterRole
  Name:  cluster-admin
Subjects:
  Kind  Name      Namespace
  ----  ----      ---------
  User  cpdadmin  


Name:         system:deployers
Labels:       <none>
Annotations:  openshift.io/description:
                Allows deploymentconfigs in this namespace to rollout pods in this namespace.  It is auto-managed by a controller; remove subjects to disa...
Role:
  Kind:  ClusterRole
  Name:  system:deployer
Subjects:
  Kind            Name      Namespace
  ----            ----      ---------
  ServiceAccount  deployer  cpd


Name:         system:image-builders
Labels:       <none>
Annotations:  openshift.io/description:
                Allows builds in this namespace to push images to this namespace.  It is auto-managed by a controller; remove subjects to disable.
Role:
  Kind:  ClusterRole
  Name:  system:image-builder
Subjects:
  Kind            Name     Namespace
  ----            ----     ---------
  ServiceAccount  builder  cpd


Name:         system:image-pullers
Labels:       <none>
Annotations:  openshift.io/description:
                Allows all pods in this namespace to pull images from this namespace.  It is auto-managed by a controller; remove subjects to disable.
Role:
  Kind:  ClusterRole
  Name:  system:image-puller
Subjects:
  Kind   Name                        Namespace
  ----   ----                        ---------
  Group  system:serviceaccounts:cpd  
  ```
  
