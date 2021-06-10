# Deploying Cloud Pak for Data 4.0 

#### PreRequisites
- [ ] OpenShift 4.6x cluster
- [ ] 3 Worker Nodes with minimum 16 core, 64 GB Ram each
- [ ] Persistent Storage NFS/OCS/Portworx - 500GB
- [ ] Two namespaces example. ibm-common-services, zen
- [ ] Users with cluster admin role
- [ ] Image Registry - 200GB


#### 1.0 Getting CPD Entitlements


#### 2.0 Downloading

<b>Download cloudctl</b>
```

wget https://github.com/IBM/cloud-pak-cli/releases/download/v3.7.1/cloudctl-linux-amd64.tar.gz
tar xvf cloudctl-linux-amd64.tar.gz
chmod +x cloudctl-linux-amd64
cp cloudctl-linux-amd64 /usr/local/bin/cloudctl

```

<b>Download openshift client</b>
```
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.6.20/openshift-client-linux-4.6.20.tar.gz
tar xvf openshift-client-linux-4.6.20.tar.gz
cp oc /usr/bin
# oc version
Client Version: 4.6.20
Server Version: 4.6.20

cp kubectl /usr/bin

```


<b>Download Cloud Pak for Data 4.0 RC1</b>

CPD Lite, Common Core Services, DB2aaservice, DB2U Operator, iis, WKC

```
wget https://ibm-open-platform.ibm.com/repos/cpd/v4.0/rc1/case/ibm-zen/1.1.0-279/ibm-zen-1.1.0-279.tgz
wget https://ibm-open-platform.ibm.com/repos/cpd/v4.0/rc1/case/ibm-ccs/1.0.0-639/ibm-ccs-1.0.0-639.tgz
wget https://ibm-open-platform.ibm.com/repos/cpd/v4.0/rc1/case/ibm-db2aaservice/4.0.0-1151.344/ibm-db2aaservice-4.0.0-1151.344.tgz
wget https://ibm-open-platform.ibm.com/repos/cpd/v4.0/rc1/case/ibm-db2uoperator/4.0.0-3506-2130/ibm-db2uoperator-4.0.0-3506-2130.tgz
wget https://ibm-open-platform.ibm.com/repos/cpd/v4.0/rc1/case/ibm-iis/4.0.0-280/ibm-iis-4.0.0-280.tgz
wget https://ibm-open-platform.ibm.com/repos/cpd/v4.0/rc1/case/ibm-wkc/4.0.0-323/ibm-wkc-4.0.0-323.tgz

```

#### 3.0 Registry Mirrors

- Create pullsecret
```

oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | grep us.icr.io
pull_secret=$(echo -n "iamapikey:<Entitlement Key>" | base64 -w0)
oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | sed -e 's|:{|:{"us.icr.io/4_0_rc1":{"auth":"'$pull_secret'"\},|' > /tmp/dockerconfig.json
oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=/tmp/dockerconfig.json
```

  
watch "oc get nodes; oc get mcp"   



### 1. Prereq steps(Only need run one time)
- 1.a Config global pull secrets
(NOTE: please run below command to check if already exist secrets for the registry. you can skip if exist.)
```
oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | grep us.icr.io
oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | grep hyc-cp4d-team-bootstrap-2-docker-local.artifactory.swg-devops.com
```

#### for us.icr.io:
```
pull_secret=$(echo -n "iamapikey:<apikey>" | base64 -w0)
oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | sed -e 's|:{|:{"us.icr.io/4_0_rc1":{"auth":"'$pull_secret'"\},|' > /tmp/dockerconfig.json
oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=/tmp/dockerconfig.json
```
for hyc-cp4d-team-bootstrap-2-docker-local.artifactory.swg-devops.com

NOTE: ibm-zen-operator-catalog hardcoded this image: hyc-cp4d-team-bootstrap-2-docker-local.artifactory.swg-devops.com/ibm-zen-operator-bundle@sha256:5508dfe8e385ff30115f78ad121967263da5179df278848c1237cebf0c7c49bd
We need create pull secrets for this registry as well.

```
pull_secret=$(echo -n "<Key>" | base64 -w0)
oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | sed -e 's|:{|:{"hyc-cp4d-team-bootstrap-2-docker-local.artifactory.swg-devops.com":{"auth":"'$pull_secret'"\},|' > /tmp/dockerconfig.json
oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=/tmp/dockerconfig.json
```

Configure Unsafe SysCtl for Db2U, custom-kubelet
```
 cat << EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  name: custom-kubelet
spec:
  machineConfigPoolSelector:
    matchLabels:
      custom-kubelet: sysctl
  kubeletConfig:
    allowedUnsafeSysctls:
      - "kernel.msg*"
      - "kernel.shm*"
      - "kernel.sem"
EOF
```
Label all Worker Nodes:-
```
oc label machineconfigpool worker custom-kubelet=sysctl
```


1.b Config ImageContentSourcePolicy
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
  - mirrors:
    - us.icr.io/4_0_rc1
    source: hyc-cp4d-team-bootstrap-2-docker-local.artifactory.swg-devops.com
EOF

waiting for all nodes Ready.
watch 'oc get nodes; oc get mcp'
```

### 2. Install Bedrock:
```
oc new-project ibm-common-services

cat << EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: opencloud-operators
  namespace: openshift-marketplace
spec:
  displayName: IBM Common Service Operator Catalog
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

Check Operator Status:-

oc get csv -l "operators.coreos.com/ibm-common-service-operator.ibm-common-services=" --namespace ibm-common-services
oc get csv -l "operators.coreos.com/ibm-namespace-scope-operator.ibm-common-services=" --namespace ibm-common-services
oc get csv -l "operators.coreos.com/ibm-odlm.ibm-common-services=" --namespace ibm-common-services

```

### 3. Install ibm-zen 1.1.0-279
```
oc get pod -l "olm.catalogSource=ibm-zen-operator-catalog" --namespace openshift-marketplace

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
  displayName: Cloud Pak for Data Operator Catalog
  publisher: IBM
EOF
```



Modify the zen-operator entry under spec.operators so that it points to the ibm-zen-operator-catalog
oc edit operandregistry common-service -n ibm-common-services

```
oc patch --namespace ibm-common-services operandregistry common-service  --type='json' -p='[{ "op": "replace", "path": "/spec/operators/15/sourceName", "value": "ibm-zen-operator-catalog" }, { "op": "replace", "path": "/spec/operators/15/channel", "value": "stable-v1" }]'
```

Create operandrequest in zen
```
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
oc get zenservice lite-cr -o json -n zen | jq ".status.$service_status"
```
once show Completed. zen installed.


### 4. Install CCS 1.0.0-639
Download case ibm-ccs-1.0.0-639.tgz
Create catalogsource in openshift-marketplace namespace:
```
 cloudctl case launch --case ibm-ccs-1.0.0-639.tgz --namespace openshift-marketplace --action installCatalog --inventory ccsSetup --tolerance=1
```
Create subscription in ibm-common-services namespace:
```
 cloudctl case launch --case ibm-ccs-1.0.0-639.tgz --namespace ibm-common-services --action installOperator --inventory ccsSetup --tolerance=1
```
Create ccs CRD in zen:

```
cat << EOF | oc apply -f -
apiVersion: ccs.cpd.ibm.com/v1beta1
kind: CCS
metadata:
  name: ccs-cr
  namespace: zen
spec:
  version: "4.0.0"
  size: "small"
  storageClass: "nfs-client"
  cert_manager_enabled: true
  license:
    accept: true
    license: "Enterprise"
  docker_registry_prefix: cp.icr.io/cp/cpd
EOF
```

Check status by running:
```
oc get ccs ccs-cr -o json -n zen | jq ".status.ccsStatus"
```
once show Completed. ccs installed.


### 5. Install db2uoperator 4.0.0-3506-2130
Download case ibm-db2uoperator-4.0.0-3506-2130.tgz
Create catalogsource in openshift-marketplace namespace:
 cloudctl case launch --case ibm-db2uoperator-4.0.0-3506-2130.tgz --namespace ibm-common-services --action installCatalog --inventory db2uOperatorSetup --tolerance=1
Create subscription in ibm-common-services namespace:
 cloudctl case launch --case ibm-db2uoperator-4.0.0-3506-2130.tgz --namespace ibm-common-services --action installOperator --inventory db2uOperatorSetup --tolerance=1
6. Install db2aaservices 4.0.0-1151.344
Download case ibm-db2aaservice-4.0.0-1151.344.tgz
Create catalogsource in openshift-marketplace namespace:
 cloudctl case launch --case ibm-db2aaservice-4.0.0-1151.344.tgz --namespace ibm-common-services --action installCatalog --inventory db2aaserviceOperatorSetup  --tolerance=1
Create subscription in ibm-common-services namespace:
 cloudctl case launch --case ibm-db2aaservice-4.0.0-1151.344.tgz --namespace ibm-common-services --action installOperator --inventory db2aaserviceOperatorSetup --tolerance=1
Create db2aaserviceservices CRD in zen:
cat << EOF | oc apply -f -
apiVersion: databases.cpd.ibm.com/v1
kind: Db2aaserviceService
metadata:
  name: db2aaserviceservice-cr
spec:
  db_type: db2aaservice
  upgrade_3_5_x_to_4_0: false
  license:
    accept: true
    license: "Enterprise"
EOF
Check status by running:
oc get db2aaserviceservice db2aaserviceservice-cr -o json -n zen | jq ".status.$service_status"
once show Completed. zen installed.
7. Install WKC 4.0.0-311
Download case ibm-wkc-4.0.0-311.tgz
Create catalogsource in openshift-marketplace namespace:
cloudctl case launch --case ibm-wkc-4.0.0-311.tgz --namespace ibm-common-services --action installCatalog --inventory wkcOperatorSetup --tolerance=1
Create subscription in ibm-common-services namespace:
cloudctl case launch --case ibm-wkc-4.0.0-311.tgz --namespace ibm-common-services --action installOperator --inventory wkcOperatorSetup --tolerance=1
Create wkc CRD in zen:
cat << EOF | oc apply -f -
apiVersion: wkc.cpd.ibm.com/v1beta1
kind: WKC
metadata:
  name: wkc-core-cr
spec:
  version: "4.0.0"
  storageClass: nfs-client
  license:
    accept: true
    license: "Enterprise"
  storage_class_name: nfs-client
  docker_registry_prefix: cp.icr.io/cp/cpd
EOF
Check status by running:
oc get wkc wkc-core-cr -o json -n zen | jq ".status.$service_status"
once show Completed. zen installed.
8. Install IIS 4.0.0-268
Download case ibm-iis-4.0.0-268.tgz
Crate restriced scc for iis:
cat << EOF | oc apply -f -
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: true
allowPrivilegedContainer: false
allowedCapabilities: null
apiVersion: security.openshift.io/v1
defaultAddCapabilities: null
fsGroup:
  type: RunAsAny
kind: SecurityContextConstraints
metadata:
  annotations:
    kubernetes.io/description: WKC/IIS provides all features of the restricted SCC
      but runs as user 10032.
  name: wkc-iis-scc
readOnlyRootFilesystem: false
requiredDropCapabilities:
- KILL
- MKNOD
- SETUID
- SETGID
runAsUser:
  type: MustRunAs
  uid: 10032
seLinuxContext:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
volumes:
- configMap
- downwardAPI
- emptyDir
- persistentVolumeClaim
- projected
- secret
users:
- system:serviceaccount:zen:wkc-iis-sa
EOF
install iis operator in ibm-common-services namespace:
cloudctl case launch --case ibm-iis-4.0.0-268.tgz --namespace ibm-common-services --action installOperatorNative --inventory iisOperatorSetup --tolerance=1
Create IIS CRD in zen:
cat << EOF | oc apply -f -
apiVersion: iis.cpd.ibm.com/v1alpha1
kind: IIS
metadata:
  name: iis-cr
spec:
  version: "4.0.0"
  storageClass: nfs-client
  license:
    accept: true
    license: "Enterprise"
  storage_class_name: nfs-client
  docker_registry_prefix: cp.icr.io/cp/cpd
  use_dynamic_provisioning: true
EOF
Check status by running:
oc get iis iis-cr -o json -n zen | jq ".status.$service_status"
once show Completed. zen installed.

9. Install UG:
Create UG CRD in zen

```
cat << EOF | oc apply -f -
apiVersion: wkc.cpd.ibm.com/v1alpha1
kind: UG
metadata:
  name: ug-cr
spec:
  version: "4.0.0"
  storageClass: nfs-client
  license:
    accept: true
  storage_class_name: nfs-client
  docker_registry_prefix: cp.icr.io/cp/cpd
EOF
```

Check status by running:
oc get ug ug-cr -o json -n zen | jq ".status.$service_status"
once show Completed. zen installed.




#### 4.0 Cluster Pre-Requisites

#### 5.0 Installing control plane

#### 6.0 Installing WKC

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
