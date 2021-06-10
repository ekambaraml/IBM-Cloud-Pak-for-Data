# Cloud Pak for Data 4.0 

## Cluster Requirements
- 3 Worker nodes 16 core, 64GB RAM
- Storage Class
- Image Registry
- Mirror Registry for AirGap Install
- python is 

## Download installers

- Download cloudctl
```
wget https://github.com/IBM/cloud-pak-cli/releases/download/v3.7.1/cloudctl-linux-amd64.tar.gz
tar xvf cloudctl-linux-amd64.tar.gz
chmod +x cloudctl-linux-amd64
cp cloudctl-linux-amd64 /usr/local/bin/cloudctl
```

- Download openshift client
```


    wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.6.20/openshift-client-linux-4.6.20.tar.gz
tar xvf openshift-client-linux-4.6.20.tar.gz
cp oc /usr/bin
# oc version
Client Version: 4.6.20
Server Version: 4.6.20

cp kubectl /usr/bin
```

- Download Cloud Pak for Data Services

  * 1.0 CPD Lite
  wget https://ibm-open-platform.ibm.com/repos/cpd/v4.0/rc1/case/ibm-zen/1.1.0-279/ibm-zen-1.1.0-279.tgz

  * 2.0 Common Core Services
  wget https://ibm-open-platform.ibm.com/repos/cpd/v4.0/rc1/case/ibm-ccs/1.0.0-637/ibm-ccs-1.0.0-637.tgz
 
   * 3.0 DB2aaservice
   wget https://ibm-open-platform.ibm.com/repos/cpd/v4.0/rc1/case/ibm-db2aaservice/4.0.0-1151.344/ibm-db2aaservice-4.0.0-1151.344.tgz

   * 4.0 DB2U Operatpr
   wget https://ibm-open-platform.ibm.com/repos/cpd/v4.0/rc1/case/ibm-db2uoperator/4.0.0-3506-2130/ibm-db2uoperator-4.0.0-3506-2130.tgz

   * 5.0 IIS
   wget https://ibm-open-platform.ibm.com/repos/cpd/v4.0/rc1/case/ibm-iis/4.0.0-268/ibm-iis-4.0.0-268.tgz

   * 6.0 WKC
   wget https://ibm-open-platform.ibm.com/repos/cpd/v4.0/rc1/case/ibm-wkc/4.0.0-311/ibm-wkc-4.0.0-311.tgz


# Mirror Image Registry

- Registry namespaces
  - ibmcom - All IBM images from dockerhub.io/ibmcom - no entitlement required
  - cp - for Images from cp.icr.io/cp Cloud Pak images requires entitlement
  - opencloudio - IBM foundationa services images from quay.io/opencloudio 

- Permission
  - User with permission to write/read
    - Write - Get images into this registry (OpenShift admin)
    - Read - During the cloud pak for Data install (Cloud Pak for Data Admin)
    - 
- Storage - 500 GB to support storage images for cloud Pak for Data Installation

- SSL certificate





## Installing the IBM Common Service Operator
## Install Bedrock and Zen Operator

https://github.ibm.com/PrivateCloud/zen-operator/wiki/Install-Bedrock-and-Zen-Operator


- 1.0 Create a namespace for bedrock 

   ```
   oc new-project ibm-common-services
   ```

- 2.0 Create bedrock catalog source
  

    ```
    apiVersion: operators.coreos.com/v1alpha1
    kind: CatalogSource
    metadata:
    name: opencloud-operators
    namespace: openshift-marketplace
    spec:
    displayName: IBMCS Operators
    publisher: IBM
    sourceType: grpc
    image: docker.io/ibmcom/ibm-common-service-catalog:latest
    updateStrategy:
        registryPoll:
        interval: 45m
    ```
        oc apply -f opencloudio-source.yaml

        oc -n openshift-marketplace get catalogsource opencloud-operators -o jsonpath="{.status.connectionState.lastObservedState}"

        READY

- 3.0 Create an operator group for bedrock
  
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

    oc apply -f 

- 4.0 CatalogSource

```
    ibm-operator-catalog.yaml

    apiVersion: operators.coreos.com/v1alpha1
    kind: CatalogSource
    metadata:
    name: ibm-operator-catalog
    namespace: openshift-marketplace
    spec:
    displayName: ibm-operator-catalog
    publisher: IBM Content
    sourceType: grpc
    image: docker.io/ibmcom/ibm-operator-catalog
    updateStrategy:
        registryPoll:
        interval: 45m

    oc apply -f ibm-operator-catalog.yaml
```
- 5.0 Deploy bedrock using the following subscription
```
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

- 6.0 new project 

- 7.0 Deploy Zen operator
```
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

- 8.0 Create zen CR
```
cat << EOF | oc apply -f -
apiVersion: zen.cpd.ibm.com/v1
kind: ZenService
metadata:
  name: lite-cr
  namespace: zen
spec:
  csNamespace: ibm-common-services
  iamIntegration: true
  version: 4.0.0
  storageClass: managed-nfs-storage
  cloudpakfordata: true #set this key if you want the ui to be cp4d
  #zenCoreMetaDbStorageClass: <special-storage-for-metadb-storage> #if using a second sc for zen metastoredb
  #cert_manager_enabled: false # set this to false only if you have not yet migrated to the new certs. The default for this is true.

EOF
  ```

  - Status
  ```
   oc get ZenService lite-cr -o yaml

  ```

  - Get Initial password
  
  ```
  user: admin

  oc extract secret/admin-user-details --keys=initial_admin_password --to=-

  ```



## 2.0 Common Core Services

- 2.1 Registry Key
  ```
    Release: 4.0-rc1-apikey read only:
    Username: iamapikey
    Key: a21xr8XGsjyX78ou-dVAX4yhjOJdRfvez4sMG3wD79Gv


    pull_secret=$(echo -n "iamapikey:a21xr8XGsjyX78ou-dVAX4yhjOJdRfvez4sMG3wD79Gv" | base64 -w0)

    oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | sed -e 's|:{|:{"us.icr.io/4_0_rc1":{"auth":"'$pull_secret'"\},|' > /tmp/dockerconfig.json

    oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=/tmp/dockerconfig.json

  ```

- 2.2 Registry


- 2.3 Create ICSP

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



Install Bedrock:

```
    oc new-project ibm-common-services

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


## Install ibm-zen 1.1.0-279
Create catalogsource:

    ```
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


    Modify the zen-operator entry under spec.operators so that it points to the ibm-zen-operator-catalog
    oc edit operandregistry common-service -n ibm-common-services
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
    Create zenservice CRD in zen:
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
    Check status by running:
    oc get zenservice lite-cr -o json -n zen | jq ".status.$service_status"
    once show Completed. zen installed.
    ```
