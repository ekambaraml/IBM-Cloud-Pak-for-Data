
### Airgap Deployment of Watson Knowledge Catalog

![Deployment architecture model](https://github.ibm.com/ekambara/cpd-images/blob/master/danske.png)


## Topics

1. Deployment Planning
2. Setup Bastion Host
3. Registry Mirroring
4. Configuring cluster pull images
5. Node settings
6. Project creation and NamespaceScope
7. Custom Security Context Constraints
8. Create IBM Operator Catalog source
9. Installing IBM Cloud Pak foundation services
10. Installing Cloud Pak for Data platform
11. Installing Watson Knowledge Catalog

***


##  1.0 Deployment planning 


### Requirements

- OpenShift 4.6.31 or later

- Cluster Compute Requirement
  - [ ] Minimum 3 compute nodes
  - [ ] Each node should be of minimum 16core, 64GB RAM and 200GB storage
  - [ ] Total Compute requirement will be calculated based on the number of services and user workload. IBM Team will help size the cluster compute requirements.
  - [ ] Minimum Compute requirement for installation: 48 Core, 196 GB
  - [ ] OpenShift nodes should meet the configuration requirements for CPD services

- Cloud Pak for Data 4.0 Services
  - [ ] Cloud Pak for Data - Control Plane
  - [ ] DB2U Database
  - [ ] Watson Knowledge Catalog and its bundle

- Networking Internet access to Mirror IBM image registries
  - [ ] docker.io/ibmcom
  - [ ] quay.io/opencloudio
  - [ ] cp.icr.io/cp 
  - [ ] icr.io/cpopen
- Networking Internet access to download IBM installer and cases:
  - [ ] https://github.com/IBM/cloud-pak-cli/releases/download
  - [ ] https://github.com/IBM/cloud-pak/raw/master/repo/case
- Storage
  - [ ] Persistent Storage 1 TB
  - [ ] Registry Storage 300 GB (example, Artifactory)
- License
  - [ ] IBM Cloud Pak for Data Standard/Enterprise license
  - [ ] Access to IBM Entitlement Key (myibm.ibm.com)
- User Permissions
  - [ ] OpenShift Administrator ( Cluster preparation )
  - [ ] Cloud Pak for Data administrator ( WKC Installation )
  - [ ] Read/Write permission to Artifactory Registry 
- Bastion Host
   - [ ] RHEL 8x, 500GB disk
   - [ ] Skopeo 1.x
   - [ ] python 2.x & 3.x
   - [ ] pyyaml, jq

***
#### 1.1 IBM Common Services & CPD namespaces limitrange
***
- [ ] Check, if there is a cluster limit range enforced
  ```
  oc get limitrange
  ```
  If there is limitrange, please ensure the min. memory limit is lower than 40Mi
 ```
 cat <<EOF |oc apply -f -
 apiVersion: "v1"
 kind: "LimitRange"
 metadata:
   name: "resource-limits"
 spec:
   limits:
     - type: "Container"
       max:
         cpu: "8"
         memory: "6Gi"
       min:
         cpu: "10m"
         memory: "30Mi"
       default:
         cpu: "1000m"
         memory: "500Mi"
       defaultRequest:
         cpu: "20m"
         memory: "200Mi"
     - type: "Pod"
       max:
         cpu: "9"
         memory: "8Gi"
       min:
         cpu: "10m"
         memory: "30Mi"
 EOF
  ```
 Resource limits for cpd instance namespace
```
 cat <<EOF |oc apply -f -
 apiVersion: "v1"
 kind: "LimitRange"
 metadata:
   name: "cpd-resource-limits"
 spec:
   limits:
     - type: "Container"
       max:
         cpu: "8"
         memory: "16Gi"
       min:
         cpu: "10m"
         memory: "30Mi"
       default:
         cpu: "1000m"
         memory: "500Mi"
       defaultRequest:
         cpu: "20m"
         memory: "200Mi"
     - type: "Pod"
       max:
         cpu: "9"
         memory: "16Gi"
       min:
         cpu: "10m"
         memory: "30Mi"
 EOF
  ```

***
#### Namespaces and Deployment configurations
***
```
export IBM_COMMON_SERVICE=cpd-common-services  # Project namespace for IBM Foundational services operators
# export CPD_OPERATORS=cpd-operators             # Project namespace for Cloud Pak for Data Operators
export CPD_INSTANCE=cpd                        # Project namespace for Cloud Pak for Data Install
export STORAGE_TYPE=nfs                        # Specify the type of storage to use, such as ocs , nfs or portworx values nfs, ocs, portworx
export CPD_LICENSE=Enterprise                  # license: Enterprise|Standard  Specify the Cloud Pak for Data license you purchased
export STORAGE_CLASS=managed-nfs-storage       # Replace with the name of a RWX storage class
export ZEN_METADB=managed-nfs-storage          # (Recommended) Replace with the name of a RWO storage class

# Entitlement and Access tokens
export REGISTRY_USER=cp 
export REGISTRY_PASSWORD=<entitlement-key>      # Cloud Pak for Data Entitlement key from myibm.com 
export REGISTRY_SERVER=cp.icr.io

export PRIVATE_REGISTRY=<registry url>          #  example: ai.ibmcloudpack.com
export PRIVATE_REGISTRY_USER=<admin>
export PRIVATE_REGISTRY_PASSWORD=<adminpass>

export CASE_REPO_PATH=https://github.com/IBM/cloud-pak/raw/master/repo/case
export CASECTL_RESOLVE_DEPENDENCIES=false                         # This required for ibm-cp-datacore
export USE_SKOPEO=true
export HOMEDIR=</root/cpd40ga>
export OFFLINEDIR=${HOMEDIR}/offline

mkdir -p  ${OFFLINEDIR}
cd ${HOMEDIR}
```

## 2.0 Bastion Host Setup
##### 15mins

***
#### 2.1 Download client tools
***
```
# Skopeo 1.x version is required
sudo yum install -y httpd-tools podman ca-certificates openssl skopeo jq bind-utils git

# Python3 is not required. Only python2 needed for CASE
# yum install -y python39
# alternatives --set python /usr/bin/python3
# pip3 install pyyaml
# Python3 is required for node crio settings

# *** Python2 is required for DB2U installations ***
yum install -y python2
alternatives --set python /usr/bin/python2

# Cloud Pak Installer
wget https://github.com/IBM/cloud-pak-cli/releases/download/v3.8.0/cloudctl-linux-amd64.tar.gz
tar -xf cloudctl-linux-amd64.tar.gz
cp cloudctl-linux-amd64 /usr/bin/cloudctl
cloudctl version

# OpenShift client (if you don't have latest oc client)
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.6.30/openshift-client-linux-4.6.30.tar.gz
tar xvf openshift-client-linux-4.6.30.tar.gz
cp oc /usr/bin
cp kubectl /usr/bin
oc version
```
***
#### 2.2 Upgrade OpenShift cluster
***

Check current version
```
oc get clusterversion
```
Upgrade the OpenShift cluster to latest (4.6.31+)
```
oc adm upgrade --to-latest=true
```
***
#### 2.3 Configure Persistent Storage type and Storage Class, if not done already
***
OpenShift Container Storage(OCS), Portworx and NFS are the supported storage types
```
oc get sc

NAME                            PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
managed-nfs-storage (default)   nfs/provisioner   Delete          Immediate           false                  30h

```

##  3.0 Mirroring to Private Registry
##### Time: 4 ~ 6 hours; About 82GB 

***
#### 3.1 Registry login Verification
***
```
# Private registry access validation
podman login -u ${PRIVATE_REGISTRY_USER} -p ${PRIVATE_REGISTRY_PASSWORD} ${PRIVATE_REGISTRY} --tls-verify=false

# IBM registry access validation
podman login -u ${REGISTRY_USER} -p ${REGISTRY_PASSWORD} ${REGISTRY_SERVER} --tls-verify=false
```
***
#### 3.2 Downloading platform and services cases and Mirroring
***

```
# platform operator
# Download cp-datacore, use the --no-dependency option to mirror cpd-platform operator
cloudctl case save \
  --case ${CASE_REPO_PATH}/ibm-cp-datacore-2.0.0.tgz \
  --no-dependency \
  --tolerance=1 \
  --outputdir ${OFFLINEDIR}

# Store the IBM Entitled Registry credentials
cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-cp-datacore-2.0.0.tgz \
  --inventory cpdPlatformOperator \
  --action configure-creds-airgap \
  --args "--registry cp.icr.io --user cp --pass ${REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}" \
  --tolerance=1

# Store Private Registry credentials
cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-cp-datacore-2.0.0.tgz \
  --inventory cpdPlatformOperator \
  --action configure-creds-airgap \
  --tolerance=1 \
  --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD}"

cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-cp-datacore-2.0.0.tgz \
  --inventory cpdPlatformOperator \
  --action mirror-images \
  --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"

# ibm common services
cloudctl case save \
--case ${CASE_REPO_PATH}/ibm-cp-common-services-1.4.1.tgz \
--outputdir ${OFFLINEDIR}

cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-cp-common-services-1.4.1.tgz \
  --inventory ibmCommonServiceOperatorSetup \
  --action mirror-images \
  --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY}_USER --pass ${PRIVATE_REGISTRY}_PASSWORD --inputDir ${OFFLINEDIR}"

# DB2U
cloudctl case save \
--case ${CASE_REPO_PATH}/ibm-db2uoperator-4.0.1.tgz \
--outputdir ${OFFLINEDIR}

cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-db2uoperator-4.0.1.tgz \
  --inventory db2uOperatorSetup \
  --action mirror-images \
  --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY}_USER --pass ${PRIVATE_REGISTRY}_PASSWORD --inputDir ${OFFLINEDIR}"

# Watson Knowledge Catalog
cloudctl case save \
--case ${CASE_REPO_PATH}/ibm-wkc-4.0.0.tgz \
--outputdir ${OFFLINEDIR}

cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-wkc-4.0.0.tgz \
  --inventory wkcOperatorSetup \
  --action mirror-images \
  --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY}_USER --pass ${PRIVATE_REGISTRY}_PASSWORD --inputDir ${OFFLINEDIR}"
```
***
#### 3.3 Validate the mirrored images
***
```
curl -k -u ${PRIVATE_REGISTRY_USER}:${PRIVATE_REGISTRY_PASSWORD} https://${PRIVATE_REGISTRY}/v2/_catalog?n=6000 |  python -m json.tool
curl -k -u ${PRIVATE_REGISTRY_USER}:${PRIVATE_REGISTRY_PASSWORD} https://${PRIVATE_REGISTRY}/v2/_catalog?n=6000 |  python -m json.tool | wc -l

```


## 3.4 Danske Bank specific cluster restriction/Customization

***
#### 3.4.1  Node labeling
***
This is a shared cluster, node level settings should be applied to only the nodes selected for cloud pak for Data.  https://access.redhat.com/solutions/5688941

```
[root@cpdclustertwo40bs0 cpd40ga]# oc get nodes
NAME                                                  STATUS   ROLES    AGE   VERSION
cpdclustertwo40m00.cpdclustertwo40.ibmcloudpack.com   Ready    master   22d   v1.19.0+d670f74
cpdclustertwo40m01.cpdclustertwo40.ibmcloudpack.com   Ready    master   22d   v1.19.0+d670f74
cpdclustertwo40m02.cpdclustertwo40.ibmcloudpack.com   Ready    master   22d   v1.19.0+d670f74
cpdclustertwo40w00.cpdclustertwo40.ibmcloudpack.com   Ready    worker   22d   v1.19.0+d670f74
cpdclustertwo40w01.cpdclustertwo40.ibmcloudpack.com   Ready    worker   22d   v1.19.0+d670f74
cpdclustertwo40w02.cpdclustertwo40.ibmcloudpack.com   Ready    worker   22d   v1.19.0+d670f74
```
Labeling the Nodes
```
oc label node   cpdclustertwo40w00.cpdclustertwo40.ibmcloudpack.com  node-role.kubernetes.io/cpd=
oc label node   cpdclustertwo40w01.cpdclustertwo40.ibmcloudpack.com  node-role.kubernetes.io/cpd=
oc label node   cpdclustertwo40w02.cpdclustertwo40.ibmcloudpack.com  node-role.kubernetes.io/cpd=
```

List all the nodes labelled for CPD Install
```
oc get nodes -l node-role.kubernetes.io/cpd=
NAME                                                  STATUS   ROLES        AGE   VERSION
cpdclustertwo40w00.cpdclustertwo40.ibmcloudpack.com   Ready    cpd,worker   22d   v1.19.0+d670f74
cpdclustertwo40w01.cpdclustertwo40.ibmcloudpack.com   Ready    cpd,worker   22d   v1.19.0+d670f74
cpdclustertwo40w02.cpdclustertwo40.ibmcloudpack.com   Ready    cpd,worker   22d   v1.19.0+d670f74
```

***
#### 3.4.2 Create CPD MachineConfigPool
***
This will make the nodes in the machine configpool to restart.
```
cat <<EOF |oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: cpd
spec:
  machineConfigSelector:
    matchExpressions:
      - {key: machineconfiguration.openshift.io/role, operator: In, values: [worker, cpd]}
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/cpd: ""
EOF
```
List the MachineConfigPools
```
[root@cpdclustertwo40bs0 cpd40ga]# oc get mcp
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
cpd                                                         False     True       False      3              0                   0                     0                      11s
master   rendered-master-5b7bd1eb30cd6e1159861f351c6c6680   True      False      False      3              3                   3                     0                      22d
worker   rendered-worker-aefffe29447faff2d0ffbb9124aa5101   True      False      False      3              3                   3                     0                      22d

```

***
#### 3.4.3 Enabling Unsafe Sysctls on a selected nodes usinng the machine configpool
***
- https://docs.openshift.com/container-platform/4.5/nodes/containers/nodes-containers-sysctls.html#nodes-containers-sysctls[…]e_nodes-containers-using to enable unsafe sysctls
- Validate the custom configurations are applied only to these labeled nodes(for cpd)

Add a label to the machine config pool where containers with the unsafe sysctls will run:
```
 oc edit machineconfigpool cpd

  labels:
    custom-kubelet: sysctl
```
#### Create a KubeletConfig custom resource (CR):
(Note: This will trigger rolling update of the nodes labeled for CPD)
The above configurations will allow the DB2 as a service to make necessary changes during the deployments.  This will also trigger update to the nodes labeled for cpd.

```
cat <<EOF |oc apply -f -
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
oc get KubeletConfig

```
# oc get KubeletConfig
NAME             AGE
custom-kubelet   13s
```
```
oc get mcp
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
cpd      rendered-cpd-aefffe29447faff2d0ffbb9124aa5101      True      False      False      3              3                   3                     0                      11m
master   rendered-master-5b7bd1eb30cd6e1159861f351c6c6680   True      False      False      3              3                   3                     0                      22d
worker   rendered-worker-aefffe29447faff2d0ffbb9124aa5101   True      False      False      0              0                   0                     0                      22d

```

***
#### 3.4.4 Project and Quotas
***
This Installation will require two new projects and access to namespace openshift-maketplace for the configuring catalog services.


${IBM_COMMON_SERVICE}  - Namespace for IBM Foundation service and cpd operators (default name: ibm-common-service)
{CPD_INSTANCE}         - Namespace for CPD control plane , Watson Knowledge Studio and  its required components.

```
oc new-project ${IBM_COMMON_SERVICE}
oc new-project ${CPD_INSTANCE}
```

Apply annotations  namespace to limit pods in the namespace to schedule in the cpd nodes only
https://docs.openshift.com/container-platform/3.7/admin_guide/managing_projects.html
```
oc patch namespace ${IBM_COMMON_SERVICE} -p '{"metadata":{"annotations":{"openshift.io/node-selector":"node-role.kubernetes.io/cpd="}}}'
oc patch namespace ${CPD_INSTANCE}  -p '{"metadata":{"annotations":{"openshift.io/node-selector":"node-role.kubernetes.io/cpd="}}}'
```
***
#### 3.4.5 Configuring Quotas for  Namespace
***
https://docs.openshift.com/container-platform/4.6/applications/quotas/quotas-setting-per-project.html

##### Resource Quotas
A resource quota, defined by a ResourceQuota object, provides constraints that limit aggregate resource consumption per project. It can limit the quantity of objects that can be created in a project by type, as well as the total amount of compute resources and storage that might be consumed by resources in that project.


${IBM_COMMON_SERVICE} - IBM foundational service and IBM Operators will be installed in this common service namespace. Here the minimum resource requirements
 CPU request: 6          CPU limit:  12,  RAM request:  24 GB  RAM limit:  40 GB,  Pod count:  34       

```
oc project ${IBM_COMMON_SERVICE}

cat <<EOF |oc apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ibm-cs-compute-resources
spec:
  hard:
    pods: "34" 
    requests.cpu: "6" 
    requests.memory: 24Gi 
    limits.cpu: "12" 
    limits.memory: 40Gi
EOF
```

```
oc get ResourceQuota
NAME                       AGE   REQUEST                                                  LIMIT
ibm-cs-compute-resources   14s   pods: 0/34, requests.cpu: 0/6, requests.memory: 0/24Gi   limits.cpu: 0/12, limits.memory: 0/40Gi
```

${CPD_INSTANCE}, CPD control plane , Watson Knowledge Catalog and  its required components (CCS, DR, DB2U, IIS, UG).
CPU request: 32          CPU limit:  120,  RAM request:  80 GB  RAM limit:  300 GB,  Pod count:  124

```
oc project ${CPD_INSTANCE}

cat <<EOF |oc apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: zen-compute-resources
spec:
  hard:
    pods: "124" 
    requests.cpu: "32" 
    requests.memory: 80Gi 
    limits.cpu: "120" 
    limits.memory: 300Gi 
EOF

```

List the Project Namespaces and Quotas settings

```
oc get ResourceQuota -A
NAMESPACE             NAME                       AGE    REQUEST                                                    LIMIT
cpd-common-services   ibm-cs-compute-resources   113s   pods: 0/34, requests.cpu: 0/6, requests.memory: 0/24Gi     limits.cpu: 0/12, limits.memory: 0/40Gi
cpd                   zen-compute-resources      13s    pods: 0/124, requests.cpu: 0/32, requests.memory: 0/80Gi   limits.cpu: 0/120, limits.memory: 0/300Gi
```

***
#### Reduce the permissions of Namescope Operator
***
This need to be completed after the deployments are completed.

Reduce the permissions of NamespaceScope operator:
https://www.ibm.com/docs/en/cpfs?topic=co-authorizing-foundational-services-perform-operations-workloads-in-namespace

## 4.0 Global pull secret and Image Content Source policy
The following tasks should be performed by the OpenShift cluster admin.
##### Time: 30mins

***
#### 4.1 Global pull secret setup
***


This deployment is using the private registry. So the pull secret should be configured the "PRIVATE REGISTRY". Save the following script to a file named "util-add-pull-secret.sh". 
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
***
#### 4.2 Add pull secret for private registry
***
```
chmod +x util-add-pull-secret.sh
./util-add-pull-secret.sh ${PRIVATE_REGISTRY} ${PRIVATE_REGISTRY_USER} ${PRIVATE_REGISTRY_PASSWORD}
```
***
#### 4.3 Verify the pullsecrets
***
```
oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq

Output (example):

 "auths": {
    "ai.ibmcloudpack.com": {
      "auth": "YWRtaW46YWRtaW5wYXNz",
      "email": "not-used"
    },
```

***
#### 4.4 ImageContentSourcePolicy Setup
***
Note: This will cause the nodes to restart
#### Requires Cluster Admin permission

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

oc get imageContentSourcePolicy
oc get imageContentSourcePolicy  cloud-pak-for-data-mirror -o yaml
```
Cluster nodes will restart, Wait for all the nodes are ready
```
watch "oc get nodes"

NAME                              STATUS                     ROLES    AGE   VERSION
master0.lustier.cp.fyre.ibm.com   Ready,SchedulingDisabled   master   66m   v1.19.0+d670f74
master1.lustier.cp.fyre.ibm.com   Ready                      master   66m   v1.19.0+d670f74
master2.lustier.cp.fyre.ibm.com   Ready                      master   65m   v1.19.0+d670f74
worker0.lustier.cp.fyre.ibm.com   Ready,SchedulingDisabled   worker   57m   v1.19.0+d670f74
worker1.lustier.cp.fyre.ibm.com   Ready                      worker   57m   v1.19.0+d670f74
worker2.lustier.cp.fyre.ibm.com   Ready                      worker   57m   v1.19.0+d670f74

```

***
#### 4.5 Create user "cpdadmin"
***
Create user "cpdadmin" with project admin role permission on the projects
```
${IBM_COMMON_SERVICE}
${CPD_INSTANCE}
```


## 5.0 Node settings (Worker)
##### Time: 1 hr
https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-changing-required-node-settings


***
#### 5.1 CRIO settings
***
This change is required, if the existing setting does not meet the WKC requirements. 
##### This requires cluster administrator permission
- [ ] Requires python3 for CRIO commands

```
default_ulimits = [
        "nofile=66560:66560" # recommended values
]
pids_limit = 12288 # Recommended values
```

Check the current values
```
scp core@$(oc get nodes | grep worker | head -1 | awk '{print $1}'):/etc/crio/crio.conf /tmp/crio.conf
```
If current values are lower than recommended, please update the /tmp/crio.conf file and run the following commands
```
crio_conf=$(cat /tmp/crio.conf | python3 -c "import sys, urllib.parse; print(urllib.parse.quote(''.join(sys.stdin.readlines())))")
```
Create machine config file 51-worker-cp4d-crio-conf.yaml file
```
cat << EOF > /tmp/51-worker-cp4d-crio-conf.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
 labels:
   machineconfiguration.openshift.io/role: worker
 name: 51-worker-cp4d-crio-conf
spec:
 config:
   ignition:
     version: 2.2.0
   storage:
     files:
     - contents:
         source: data:,${crio_conf}
       filesystem: root
       mode: 0644
       path: /etc/crio/crio.conf
EOF
```
Apply the new machineconfig to the cluster by running the following command:
```
oc create -f /tmp/51-worker-cp4d-crio-conf.yaml
```

***
#### 5.2 Kernel Parameters
***
Pleaser refere this documentation for Kernerl parameter requirement for WKC
https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-changing-required-node-settings
Below settings are respective for node of 64 cpu and 256GB of RAM: 

```
cat <<EOF |oc apply -f -
apiVersion: tuned.openshift.io/v1
kind: Tuned
metadata:
  name: cp4d-wkc-ipc
  namespace: openshift-cluster-node-tuning-operator
spec:
  profile:
  - name: cp4d-wkc-ipc
    data: |
      [main]
      summary=Tune IPC Kernel parameters on OpenShift Worker Nodes running WKC Pods
      [sysctl]
      kernel.shmall = 134217728
      kernel.shmmax = 274877906944
      kernel.shmmni = 65536
      kernel.sem = 250 1024000 100 65536
      kernel.msgmax = 65536
      kernel.msgmnb = 65536
      kernel.msgmni = 262144
      vm.max_map_count = 262144
  recommend:
  - match:
    - label: node-role.kubernetes.io/cpd
    priority: 10
    profile: cp4d-wkc-ipc
EOF
```



***
#### 5.3 Configure Unsafe SysCtl for Db2U, custom-kubelet
***

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

oc label machineconfigpool worker custom-kubelet=sysctl

```

## 6.0 Creating service Catalog Source
##### Time: 30 mins
***
#### 6.1 IBM Foundational services
***

```
cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-cp-common-services-1.4.1.tgz \
  --inventory ibmCommonServiceOperatorSetup \
  --namespace openshift-marketplace \
  --action install-catalog \
  --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
```

***
#### 6.2 IBM Cloud Pak for Data
***
```
cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-cp-datacore-2.0.0.tgz \
  --inventory cpdPlatformOperator \
  --namespace openshift-marketplace \
  --action install-catalog \
  --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
```
***
#### 6.3 WKC service catalog source
***
WKC will install its prequired services also as part of the Installation.
##### Components: CPD Common Services, Data Refinery, DB2 as a Service, DB2U operator, IBM information Governer, Unified Governance
### Requirements on Bastion Host
- [ ] Python2 via yum
- [ ] pip2 install pyyaml

```
cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-wkc-4.0.0.tgz \
  --inventory wkcOperatorSetup \
  --namespace openshift-marketplace \
  --action install-catalog \
  --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
```

Status check
```
oc get CatalogSources -n openshift-marketplace
oc get pods -n openshift-marketplace
```


## 7.0 Project creation and NamespaceScope
##### Time: 15mins
***
#### 7.1 Create project names
***
OpenShift cluster admin has to create these namespaces

```
oc new-project ${IBM_COMMON_SERVICE}
oc new-project ${CPD_INSTANCE}
```
***
#### 7.2 OperatorGroups
***

_This installation will be single namespace for IBM Foundation and CPD Operators. Please make sure only one operator group is defined in the namespace, OLM will give error, if you create two operator group in a single namespace_

```
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

```

Status
```
oc get OperatorGroup operatorgroup -n ${IBM_COMMON_SERVICE} -o yaml

```
***
#### 7.3 IBM NamespaceScope Operator to watch IBM Common Services and Cloud Pak for Data namespaces
***
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
```

Wait for the Namescope operator pod to come up before running the next command
oc get pods -n ${IBM_COMMON_SERVICE}

```
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

***
### 7.4 ConfigMap for Deploying in custom namespaces
***
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

## 8.0 Custom Security Context Constraints
##### Time: 10 mins
Security Context Constraints are used in OpenShift to control what kind of privileges being requested for each pod is allowed on the platform.

***
#### 8.1 Custom SCC for Watson Knowledge Catalog
***
```
cat <<EOF |oc apply -f -
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
- system:serviceaccount:${CPD_INSTANCE}:wkc-iis-sa

EOF
```
Verify the creation of wkc scc

```
oc get scc wkc-iis-scc
```
***
#### 8.2 DB2 Custom SCC
***
This is created during the Install





## 9.0 Installing IBM Foundational Service 
##### Time: about 30 minutes
Please make sure the OperatorGroup, Namescope operator and ConfigMap are created for custom installations.  
_ODLM and custom bedrock namespace has a issue. We are using non ODLM Install_

Details are defined in the section #6.
1. [x] Download and Mirror Images to Private Registry
2. [x] OperatorGroup
3. [x] Namescope Operator
4. [x] ConfigMap for custom namespace
5. [x] Create CatalogSubscription
6. [ ] IBM Common Service Subscription


***
#### 9.1 Create IBM Common Service Subscription
***
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
```
Wait for all the pods are up and running
```
watch "oc get pods -n ${IBM_COMMON_SERVICE}"
```

##### *** ibm-cert-manager-operator will not start in a cluster where the namespace quota or cluster limitrange is enforced ***
Fix is planned for next refresh.
User may manually edit the ibm-cert-manager-operator to add the resources limits given below

```
name: ibm-cert-manager-operator
                    resources:
                      limits:
                        cpu: 100m
                        memory: 300Mi
                      requests:
                        cpu: 10m
                        memory: 50Mi
```

Status Verification
```
oc --namespace ${IBM_COMMON_SERVICE} get csv
oc get po -n ${IBM_COMMON_SERVICE} 
oc get crd | grep operandrequest
```



## 10.0 Installing Cloud Pak for Data (ZenService/lite)
##### Time: About 1hrs
Steps:
1. [x] Download and Mirror the images to Private Registry
2. [x] OperatorGroup
3. [x] Namescope Operator
4. [x] Create service CatalogSource
5. [ ] Create service Subscription
6. [ ] Installing the Cloud Pak for Data

***
#### 10.1 Create service Subscription
***
```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cpd-operator
  namespace: ${IBM_COMMON_SERVICE}    # The project that contains the Cloud Pak for Data operator
spec:
  channel: stable-v1
  installPlanApproval: Automatic
  name: cpd-platform-operator
  source: cpd-platform
  sourceNamespace: openshift-marketplace
EOF
```

#### Troubleshooting tips:
If Deployments are not upto date after the subscription, if replicaset is also ready, Please check
both deployment and replicasets.  Check, if there is any reason they failed, please look for message for failure


```
oc get deployments -n ${CPD_INSTANCE}  
oc get rs -n ${CPD_INSTANCE}
oc describe rc <rs-name> -n ${CPD_INSTANCE}

```

***
#### 10.2 Installing Cloud Pak for Data (Zen/lite)
***
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
  storageVendor: ${STORAGE_TYPE}
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
oc extract secret/admin-user-details --keys=initial_admin_password --to=-
```


## 11.0 Installing Watson Knowledge Catalog
##### Time: About 3 hours
Installing the Watson Knowledge Catalog on a existing Cloud Pak for Data platform

1. [x] Download and Mirror the images to your private registry
2. [x] Configure Node Settings
3. [x] Create custom/Restricted Security Context Constrants
4. [x] Create service CatalogSource
5. [ ] Create service Subscription
6. [ ] Install the WKC service

***
#### 11.1 Save Watson Knowledge Catalog
***
##### *** Note: This should have been completed in earlier steps ***
```
cloudctl case save \
--case ${CASE_REPO_PATH}/ibm-wkc-4.0.0.tgz \
--outputdir ${OFFLINEDIR}
```

***
#### 11.2 Create Catalog Source for WKC
***
##### *** Note: This should have been completed in earlier steps ***
```
cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-wkc-4.0.0.tgz \
  --inventory wkcOperatorSetup \
  --namespace openshift-marketplace \
  --action install-catalog \
  --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
```

Status check
```
oc get pods -n openshift-marketplace
```
***
#### 11.3 Create Service Subscription
***
```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    app.kubernetes.io/instance:  ibm-cpd-wkc-operator-catalog-subscription
    app.kubernetes.io/managed-by: ibm-cpd-wkc-operator
    app.kubernetes.io/name:  ibm-cpd-wkc-operator-catalog-subscription
  name: ibm-cpd-wkc-operator-catalog-subscription
  namespace: ${IBM_COMMON_SERVICE}     # The project that contains the Cloud Pak for Data operator
spec:
    channel: v1.0
    installPlanApproval: Automatic
    name: ibm-cpd-wkc
    source: ibm-cpd-wkc-operator-catalog
    sourceNamespace: openshift-marketplace
EOF
```

***
#### 11.4 Installing WKC using ODLM
***
```
cat <<EOF |oc apply -f -
apiVersion: wkc.cpd.ibm.com/v1beta1
kind: WKC
  name: wkc-cr     # This is the recommended name, but you can change it
  namespace: ${CPD_INSTANCE}    # Replace with the project where you will install Watson Knowledge Catalog
spec:
  license:
    accept: true
    license: ${LICENCE_TYPE}    # Specify the license you purchased
  version: 4.0.0
  storageVendor: ${STORAGE_TYPE}    # Specify the type of storage to use, such as ocs
  # install_wkc_core_only: true     # To install the core version of the service, remove the comment tagging from the beginning of the line.
  # wkc_db2u_set_kernel_params: True     
   docker_registry_prefix: ${PRIVATE_REGISTRY}/cp/cpd
   useODLM: true
   csNamespace: ${IBM_COMMON_SERVICE}

EOF
```

#### Example

Note: Sometime the parameters may have issue, in this case, here is example of WKC CR

```
cat wkc-odlm.yaml
apiVersion: wkc.cpd.ibm.com/v1beta1
kind: WKC
metadata:
  name: wkc-cr     # This is the recommended name, but you can change it
  namespace: cpd     # Replace with the project where you will install Watson Knowledge Catalog
spec:
  license:
    accept: true
    license: Enterprise
  version: 4.0.0
  storageVendor: ocs
  storageClass: ocs-storagecluster-cephfs
  docker_registry_prefix: danske.ibmcloudpack.com:5000/cp/cpd 
  useODLM: true


oc create -f wkc-odlm.yaml
```

Status Check
```
oc get WKC wkc-cr -o jsonpath='{.status.wkcStatus} {"\n"}'
oc get CCS ccs-cr -o jsonpath='{.status.ccsStatus} {"\n"}'
oc get DataRefinery datarefinery-sample -o jsonpath='{.status.datarefineryStatus} {"\n"}'
oc get Db2aaserviceService db2aaservice-cr -o jsonpath='{.status.db2aaserviceStatus} {"\n"}'
oc get IIS iis-cr -o jsonpath='{.status.iisStatus} {"\n"}'
oc get UG ug-cr -o jsonpath='{.status.ugStatus} {"\n"}'
```

***
## 12.0 Reduce the permissions of Namescope Operator
***
## 13.0 References

[1] Cloud Pak for Data Knowledge Center (https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0)
