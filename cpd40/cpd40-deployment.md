# Cloud Pak for Data 4.0 Deployments

## 1.0 Prerequisites

- 1.a Config global pull secrets

  (NOTE: please run below command to check if already exist secrets for the registry. you can skip if exist.)

  - Check, if it is already configured
  ```
  oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | grep us.icr.io

   pull_secret=$(echo -n "iamapikey:a21xr8XGsjyX78ou-dVAX4yhjOJdRfvez4sMG3wD79Gv" | base64 -w0)

   oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | sed -e 's|:{|:{"us.icr.io/4_0_rc1":{"auth":"'$pull_secret'"\},|' > /tmp/dockerconfig.json
   
   oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=/tmp/dockerconfig.json
   ```

- 1.b Config ImageContentSourcePolicy
    ```
    cat << EOF | oc apply -f -
    apiVersion: operator.openshift.io/v1alpha1
    kind: ImageContentSourcePolicy
    metadata:
    name: mirror-config
    spec:
    repositoryDigestMirrors:
    - mirrors:
        - us.icr.io/4_0_rc1
        source: quay.io/opencloudio
    - mirrors:
        - us.icr.io/4_0_rc1
        source: icr.io/cpopen
    - mirrors:
        - us.icr.io/4_0_rc1
        source: docker.io/ibmcom
    - mirrors:
        - us.icr.io/4_0_rc1
        source: cp.icr.io/cp/cpd
    - mirrors:
        - us.icr.io/4_0_rc1
        source: cp.icr.io/cp
    EOF
    ```

    waiting for all nodes Ready.
    ```
    watch "oc get nodes; oc get mcp"
    ```

- 2. Install Bedrock:
  
```
cat << EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: opencloud-operators
  namespace: openshift-marketplace
spec:
  displayName: IBMCS Test Operators
  publisher: IBM
  sourceType: grpc
  image: docker.io/ibmcom/ibm-common-service-catalog:latest
  updateStrategy:
    registryPoll:
      interval: 45m
EOF

cat << EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: operatorgroup
  namespace: ibm-common-services
spec:
  targetNamespaces:
  - ibm-common-services
EOF

cat << EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ibm-common-service-operator
  namespace: ibm-common-services
spec:
  channel: v3
  installPlanApproval: Automatic
  name: ibm-common-service-operator
  source: opencloud-operators
  sourceNamespace: openshift-marketplace
EOF

```

oc get CatalogSource -n openshift-marketplace 

- 3. Install ibm-zen 1.1.0-279
```
Create catalogsource:
cat << EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ibm-zen-operator-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: quay.io/opencloudio/ibm-zen-operator-catalog@sha256:fa94584adfe62d046aca5a7ee31e287b539f06e8c57a6789106397fe3d99f72a
  imagePullPolicy: Always
  displayName: Cloud Pak for Data
  publisher: IBM
EOF
```

Modify the zen-operator entry under spec.operators so that it points to the ibm-zen-operator-catalog

```
oc patch --namespace ibm-common-services operandregistry common-service  --type='json' -p='[{ "op": "replace", "path": "/spec/operators/15/sourceName", "value": "ibm-zen-operator-catalog" }, { "op": "replace", "path": "/spec/operators/15/channel", "value": "stable-v1" }]'
```


oc edit operandregistry common-service -n ibm-common-services

```
Create operandrequest in zen
oc new-project zen
cat << EOF | oc apply -f -
apiVersion: operator.ibm.com/v1alpha1
kind: OperandRequest
metadata:
  name: zen-service
  namespace: zen
spec:
  requests:
    - operands:
        - name: ibm-zen-operator
        - name: ibm-cert-manager-operator
      registry: common-service
      registryNamespace: ibm-common-services
EOF
```

Create zenservice CRD in zen:
```
cat << EOF | oc apply -f -
apiVersion: zen.cpd.ibm.com/v1
kind: ZenService
metadata:
  name: lite-cr
  namespace: zen
spec:
  csNamespace: ibm-common-services
  storageClass: nfs-client
  zenCoreMetaDbStorageClass: nfs-client
  generateAdminPassword: false
  iamIntegration: false
  cert_manager_enabled: true
  upgrade_3_5_x_to_4_0: false
  cloudpakfordata: true
  cpdlegacy: false
  version: "4.0.0"
EOF
```

Check status by running:
```
oc get zenservice lite-cr -o json -n zen | jq ".status.zenStatus"
"InProgress"


oc get zenservice lite-cr -o json -n zen | jq ".status.zenStatus"
"Completed"
```

once show Completed. zen installed.



### 4. Install CCS 1.0.0-639
Download case ibm-ccs-1.0.0-639.tgz

Create catalogsource in openshift-marketplace namespace:
```
cloudctl case launch --case ibm-ccs-1.0.0-639.tgz --namespace openshift-marketplace --action installCatalog --inventory ccsSetup --tolerance=1
```



