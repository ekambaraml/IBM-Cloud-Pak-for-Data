## 1.0 Requirements

- OpenShift 4.6 cluster
- 3 worker nodes
- Persistent Storage
- User with cluster admin role
- Bastion Host (RHEL 8.x)


### 2.0 Download CPD Installer
```
wget https://github.com/IBM/cloud-pak-cli/releases/download/v3.8.0/cloudctl-linux-amd64.tar.gz
tar -xf cloudctl-linux-amd64.tar.gz
cp cloudctl-linux-amd64 /usr/bin/cloudctl
cloudctl version
```

## 3.0 Setup Bastion Host
```
sudo yum install -y httpd-tools podman ca-certificates openssl skopeo jq bind-utils git
yum install -y python2
alternatives --set python /usr/bin/python2


yum install -y python39
alternatives --set python /usr/bin/python3
pip3 install pyyaml
```

## 4.0 Update the OpenShift version

```
oc get clusterversion
oc adm upgrade --to-latest=true


5.0 Setup NFS provisioner
Update deployment.yaml with NFS server and share path


$ git clone https://github.com/ekambaraml/nfs-ocp43
$ cd nfs-ocp43
$ vi deployment.yaml


follow the instruction in the git document to complete the NFS persistent storge setup


$ oc get sc
NAME                            PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
managed-nfs-storage (default)   nfs/provisioner   Delete          Immediate           false                  112s
```

## 5.0 Node Configuration
```
https://www.ibm.com/docs/en/cloud-paks/cp-data/3.5.0?topic=tasks-changing-required-node-settings
HAPROXY timeout
CRIO setting
Kernel setting
Kubelet
```

## 6.0 Registry Mirroring


#### 6.1  Download cp-datacore, use the --no-dependency option to mirror cpd-platform operator
cloudctl case save \
  --case ${CASE_REPO_PATH}/ibm-cp-datacore-2.0.1.tgz \
  --no-dependency \
  --tolerance=1 \
  --outputdir ${OFFLINEDIR}


#### 6.2  Store the IBM Entitled Registry credentials
cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-cp-datacore-2.0.1.tgz \
  --inventory cpdPlatformOperator \
  --action configure-creds-airgap \
  --args "--registry cp.icr.io --user cp --pass ${REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}" \
  --tolerance=1


#### 6.3 Store Private Registry credentials
cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-cp-datacore-2.0.1.tgz \
  --inventory cpdPlatformOperator \
  --action configure-creds-airgap \
  --tolerance=1 \
  --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD}"

#### 6.4 Mirror CPD images
```
cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-cp-datacore-2.0.1.tgz \
  --inventory cpdPlatformOperator \
  --action mirror-images \
  --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
```

#### 6.5  Save IBM Foundataion Service CASE

```
cloudctl case save \
--case ${CASE_REPO_PATH}/ibm-cp-common-services-1.4.1.tgz \
--outputdir ${OFFLINEDIR}
```

#### 6.6  Mirror IBM Foundation Service images
```
cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-cp-common-services-1.4.1.tgz \
  --inventory ibmCommonServiceOperatorSetup \
  --action mirror-images \
  --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY}_USER --pass ${PRIVATE_REGISTRY}_PASSWORD --inputDir ${OFFLINEDIR}"
```

Validate the Images in private registry
```
curl -k -u ${PRIVATE_REGISTRY_USER}:${PRIVATE_REGISTRY_PASSWORD} https://${PRIVATE_REGISTRY}/v2/_catalog?n=6000 |  python -m json.tool
curl -k -u ${PRIVATE_REGISTRY_USER}:${PRIVATE_REGISTRY_PASSWORD} https://${PRIVATE_REGISTRY}/v2/_catalog?n=6000 |  python -m json.tool | wc -l
```



## 7.0 Private Registry Access configuration


#### 7.1 Global Pull secret


##### 7.1a Create util-add-pull-secret.sh file using below code
```
# usage" util-add-pull-secret.sh <repository> <user> <password>


# cat util-add-pull-secret.sh
#!/bin/bash


if [ "$#" -lt 3 ]; then
  echo "Usage: $0 <repo-url> <artifactory-user> <API-key>" >&2
  exit 1
fi


# set -x


REPO_URL=$1
REPO_USER=$2
REPO_API_KEY=$3


pull_secret=$(echo -n "$REPO_USER:$REPO_API_KEY" | base64 -w0)
oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | sed -e 's|:{|:{"'$REPO_URL'":{"auth":"'$pull_secret'","email":"not-used"\},|' > /tmp/dockerconfig.json
oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=/tmp/dockerconfig.json
```

##### 7.1b Add private registry pull secret 
```
$ chmod +x util-add-pull-secret.sh
$ ./util-add-pull-secret.sh ${PRIVATE_REGISTRY} ${PRIVATE_REGISTRY_USER} ${PRIVATE_REGISTRY_PASSWORD}
```

##### 7.1c Get the registry pull secret status
```
oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq
```

#### 7.2 ImageContentSourcePolicy

```
cat <<EOF |oc apply -f -
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: cloud-pak-for-data-mirror
spec:
  repositoryDigestMirrors:
  - mirrors:
    - ${PRIVATE_REGISTRY}
    - ${PRIVATE_REGISTRY}/ibmcom
    source: docker.io/ibmcom
  - mirrors:
    - ${PRIVATE_REGISTRY}
    source: quay.io/opencloudio
  - mirrors:
    - ${PRIVATE_REGISTRY}
    - ${PRIVATE_REGISTRY}/cp
    - ${PRIVATE_REGISTRY}/cp/cpd
    source: cp.icr.io/cp/cpd
  - mirrors:
    - ${PRIVATE_REGISTRY}
    - ${PRIVATE_REGISTRY}/cp
    source: cp.icr.io/cp
  - mirrors:
    - ${PRIVATE_REGISTRY}
    - ${PRIVATE_REGISTRY}/cpopen
    source: icr.io/cpopen
EOF


The above will cause nodes restart. Check the status with following commands
$ oc get nodes
$ oc get mcp
```

## 8.0 Service Catalog Source


#### 8.1 IBM Foundational services

```
cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-cp-common-services-1.4.1.tgz \
  --inventory ibmCommonServiceOperatorSetup \
  --namespace openshift-marketplace \
  --action install-catalog \
  --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"


Troubleshooting: Missing icr.io in CatalogSource of opencloud-operators. opencloud-operators will be not able to start due to image name issue, the image name will be missing registry prefix. To fix the issue edit catalogsource adding registry prefix:
oc edit CatalogSource -n openshift-marketplace


    displayName: IBMCS Operators
    image: **icr.io**/cpopen/ibm-common-service-catalog:latest
```

#### 8.2 IBM Cloud Pak for Data
```

cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-cp-datacore-2.0.1.tgz \
  --inventory cpdPlatformOperator \
  --namespace openshift-marketplace \
  --action install-catalog \
  --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
```


## 9.0 Project/NameSpace configuration
This deployment will be using two namespaces. Single namespace for both Bedrock and  Cloud Pak for Data operators. Second namespace for the CPD services installations.


#### 9.1 Project creations

```
oc new-project ${IBM_COMMON_SERVICE}
oc new-project ${CPD_INSTANCE}
```

#### 9.2 OperatorGroups

```
This installation will be single namespace for IBM Foundation and CPD Operators. Please make sure only one operator group is defined in the namespace, OLM will give error, if you create two operator group in a single namespace
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha2
kind: OperatorGroup
metadata:
  name: operatorgroup
  namespace: ${IBM_COMMON_SERVICE}
spec:
  targetNamespaces:
  - ${IBM_COMMON_SERVICE}
EOF


Status
oc get OperatorGroup operatorgroup -n ${IBM_COMMON_SERVICE} -o yaml
```

#### 9.3  IBM NamespaceScope Operator
To watch IBM Common Services and Cloud Pak for Data namespaces

```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ibm-namespace-scope-operator
  namespace: ${IBM_COMMON_SERVICE}
spec:
  channel: v3
  installPlanApproval: Automatic
  name: ibm-namespace-scope-operator
  source: opencloud-operators
  sourceNamespace: openshift-marketplace
EOF


Wait for the Namescope operator pod to come up before running the next command 
oc get pods -n ${IBM_COMMON_SERVICE}


cat <<EOF |oc apply -f -
apiVersion: operator.ibm.com/v1
kind: NamespaceScope
metadata:
  name: cpd-operators
  namespace: ${IBM_COMMON_SERVICE}
spec:
  namespaceMembers:
  - ${IBM_COMMON_SERVICE}
  - ${CPD_INSTANCE}
EOF
```

#### 9.4  ConfigMap for Deploying in custom namespaces
 
IBM Common services are deployed in the namespace "ibm-common-services", Cloud Pak for Data operators are in "cpd-operators" by default. ConfigMap is required when customer wants to use custom namespaces.
```

cat <<EOF |oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: common-service-maps
  namespace: kube-public
data: 
  common-service-maps.yaml: |
    namespaceMapping:
    - requested-from-namespace:
      - ${CPD_INSTANCE}
      map-to-common-service-namespace: ${IBM_COMMON_SERVICE}     
    defaultCsNs: ibm-common-services
EOF
```





## 10.0 Zen Bedrock Service Subscription


#### 10.1 Create IBM Common Service Subscription

```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ibm-common-service-operator
  namespace: ${IBM_COMMON_SERVICE}
spec:
  channel: v3
  installPlanApproval: Automatic
  name: ibm-common-service-operator
  source: opencloud-operators
  sourceNamespace: openshift-marketplace
EOF


Wait for all the pods are up and running
watch "oc get pods -n ${IBM_COMMON_SERVICE}"


Status Verification
oc --namespace ${IBM_COMMON_SERVICE} get csv
NAME                                          DISPLAY                                VERSION   REPLACES                                      PHASE
ibm-common-service-operator.v3.8.1            IBM Cloud Pak foundational services    3.8.1     ibm-common-service-operator.v3.8.0            Succeeded
ibm-namespace-scope-operator.v1.2.0           IBM NamespaceScope Operator            1.2.0     ibm-namespace-scope-operator.v1.1.1           Succeeded
operand-deployment-lifecycle-manager.v1.6.0   Operand Deployment Lifecycle Manager   1.6.0     operand-deployment-lifecycle-manager.v1.5.0   Succeeded
oc get po -n ${IBM_COMMON_SERVICE} 
oc get crd | grep operandrequest
```

## 11.0 Install CPD Service


#### 11.1 Create service Subscription

```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cpd-operator
  namespace: ${IBM_COMMON_SERVICE}    # The project that contains the Cloud Pak for Data operator
spec:
  channel: stable-v1
  installPlanApproval: Automatic
  name: cpd-platform-operator
  source: cpd-platform
  sourceNamespace: openshift-marketplace
EOF


Troubleshooting tips:If Deployments are not upto date after the subscription, if replicaset is also ready, Please check both deployment and replicasets. Check, if there is any reason they failed, please look for message for failure
oc get po -n ${IBM_COMMON_SERVICE}
oc get deployments -n ${IBM_COMMON_SERVICE}  
oc get rs -n ${IBM_COMMON_SERVICE}
```

#### 11.2 Installing Cloud Pak for Data (Zen/lite)

```
cat <<EOF |oc apply -f -
apiVersion: cpd.ibm.com/v1
kind: Ibmcpd
metadata:
  name: ibmcpd-cr                                     
  namespace: ${CPD_INSTANCE}                          
spec:
  license:
    accept: true
    license: ${CPD_LICENSE}                          
  storageClass: ${STORAGE_CLASS}
  zenCoreMetadbStorageClass: ${STORAGE_CLASS}   
  version: "4.0.1"
  csNamespace: ${IBM_COMMON_SERVICE}
EOF
```

Wait for the install complete
``` 
# Get the Intance details
oc project ${CPD_INSTANCE}
oc get operandrequest
oc get ZenService lite-cr -o jsonpath="{.status.zenStatus}{'\n'}"
oc get ZenService lite-cr -o jsonpath="{.status.url}{'\n'}"
```

Extract default password:
``` 
oc extract secret/admin-user-details --keys=initial_admin_password --to=-
``` 


## 12.0 Installing Watson Studio


#### 12.1 Save the Watson Studio CASE
```
cloudctl case save \
--case ${CASE_REPO_PATH}/ibm-wsl-2.0.0.tgz \
--outputdir ${OFFLINEDIR}
```

#### 12.2 Mirror the WS Images
```
cloudctl case launch \
 --case ${OFFLINEDIR}/ibm-wsl-2.0.0.tgz \
 --inventory wslSetup --action mirror-images \
 --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
```

#### 12.3 Service CatalogSource

```
cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-wsl-2.0.0.tgz \
  --inventory wslSetup \
  --namespace openshift-marketplace \
  --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
```

#### 12.4 Service Subscription
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

#### 12.5 Watson Studio CRD Creation
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
  storageClass: ${STORAGE_CLASS}                
EOF
```
