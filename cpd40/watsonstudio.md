
## 13.0 Installing Watson Studio


#### 13.1 Save the Watson Studio CASE
```
cloudctl case save \
--case ${CASE_REPO_PATH}/ibm-wsl-2.0.0.tgz \
--outputdir ${OFFLINEDIR}
```

#### 13.2 Mirror the WS Images
```
cloudctl case launch \
 --case ${OFFLINEDIR}/ibm-wsl-2.0.0.tgz \
 --inventory wslSetup --action mirror-images \
 --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
```

#### 13.3 Service CatalogSource

```
cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-wsl-2.0.0.tgz \
  --inventory wslSetup \
  --namespace openshift-marketplace \
  --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
```

#### 13.4 Service Subscription
```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  annotations:
  name: ibm-cpd-ws-operator-catalog-subscription
  namespace: ${IBM_COMMON_SERVICE}
spec:
  channel: v2.0
  installPlanApproval: Automatic
  name: ibm-cpd-wsl
  source: ibm-cpd-ws-operator-catalog
  sourceNamespace: openshift-marketplace
EOF
```

#### 13.5 Watson Studio CRD Creation
```
cat <<EOF |oc apply -f -
apiVersion: ws.cpd.ibm.com/v1beta1
kind: WS
metadata:
  name: ws-cr
  namespace: ${CPD_INSTANCE}
spec:
  docker_registry_prefix: ${PRIVATE_REGISTRY}/cp/cpd
  license:
    accept: true
    license: ${CPD_LICENSE}
  version: 4.0.0
  storageVendor: ""
  storageClass: ${STORAGE_CLASS}                
EOF
```
