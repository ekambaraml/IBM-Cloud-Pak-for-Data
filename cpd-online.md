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


### Patching CPD Assembly


### Monitoring pods 

oc get pods | egrep -iv '1/1|2/2|3/3|4/4|completed'

watch "oc get pods | egrep -iv '1/1|2/2|3/3|4/4|completed'"
