## OnLine Installation of Cloud Pak for Data


```
#!/bin/sh

# STORAGE="managed-nfs-storage"
STORAGE="portworx-shared-gp3 --override-config portworx"
NAMESPACE=zen
USER=ocadmin
SERVICES="lite wsl wkc wml ds spark rstudio db2wh dmc"
for assembly in $SERVICES
do
./cpd-cli adm --arch x86_64 -a $assembly  -n $NAMESPACE -r repo.yaml  --accept-all-licenses --apply
./cpd-cli install -r repo.yaml -a $assembly -n $NAMESPACE  --storageclass $STORAGE \
  --transfer-image-to $(oc registry info)/$NAMESPACE  \
  --target-registry-username $USER --target-registry-password $(oc whoami -t) \
  --cluster-pull-prefix image-registry.openshift-image-registry.svc:5000/$NAMESPACE \
  --insecure-skip-tls-verify=true --accept-all-licenses --latest-dependency
done
```

### Upgrading CPD Assembly
./cpd-cli patch --repo repo.yaml --action download --assembly rstudio --patch-name cpd-3.5.3-spark-patch-2  --download-path spark --version 3.5.3

./cpd-cli patch --patch-name <cpd-3.5.0-bigsql-patch-341> -r repo.yaml -a <big-sql>  -n zen   \
  --transfer-image-to <default-route-openshift-image-registry.apps.cluster.example.com/zen> \
  --target-registry-username ocadmin --target-registry-password $(oc whoami -t) \
  --cluster-pull-prefix image-registry.openshift-image-registry.svc:5000/zen \
  --insecure-skip-tls-verify=true --accept-all-licenses --action transfer --dry-run


### Patching CPD Assembly


### Monitoring pods 

oc get pods | egrep -iv '1/1|2/2|3/3|4/4|completed'

watch "oc get pods | egrep -iv '1/1|2/2|3/3|4/4|completed'"

  
### How to Create Profile
  
  
1. Get APIKEY from cloud pack for data user profile

2. ./cpd-cli config users set cpd-admin-user --username admin --apikey <APIKEY>

3. ./cpd-cli config profiles set cpd-admin-profile --user cpd-admin-user --url <https://zen-cpd-zen.apps.cluster.example.com>

4. ./cpd-cli service-instance list --profile cpd-admin-profile
