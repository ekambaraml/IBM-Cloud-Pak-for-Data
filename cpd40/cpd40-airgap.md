## AirGap Install

### Deployment Architecture


### Requirements

The following URLs needs to accessible from the Internet facing client machine for mirroring Product Images and Install tools

Repository | Usage
---|---
cp.icr.io/cp | Images that are pulled from the IBM Entitled Registry that require an entitlement key to download. Most of the IBM Cloud Pak for Data software uses this tag.
icr.io/cpopen | Publicly available images that are provided by IBM and that don't require an entitlement key to download.The IBM Cloud Pak for Data operators use this tag.
quay.io/opencloudio | IBM opensource images that are available on quay.io. The IBM Cloud Pak® foundational services software uses this tag.
docker.io/ibmcom | Images publicly available for  db2u and db2as 
https://github.com/IBM/ | Installers
https://raw.githubusercontent.com/IBM/cloud-pak/ | Installer

### User Roles & Permissions Requirements

Permissions | Roles
-----|----
OpenShift Cluster Admin | Downloading Product images, Installing

### Getting Product and Installers

- Create directory for installation tool downloading
```
export HOMEDIR=/root/cpd40
mkdir -p  ${HOMEDIR}
cd ${HOMEDIR}
```

- IBM Cloud Pak installer
```
wget https://github.com/IBM/cloud-pak-cli/releases/download/v3.7.1/cloudctl-linux-amd64.tar.gz
tar -xf cloudctl-linux-amd64.tar.gz
cp cloudctl-linux-amd64 /usr/bin/cloudctl
```
- OpenShift clients
```
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.6.20/openshift-client-linux-4.6.20.tar.gz
tar xvf openshift-client-linux-4.6.20.tar.gz
cp oc /usr/bin
cp kubectl /usr/bin
oc version
```

- Cloud Pak for Data

OFFLINEDIR= ${HOMEDIR}/offline
mkdir -p  ${OFFLINEDIR}
cd ${OFFLINEDIR}

get https://github.com/IBM/cloud-pak/blob/master/repo/case/ibm-cp-common-services-1.4.0.tgz?raw=true -O ibm-cp-common-services-1.4.0.tgz
wget http://icpfs1.svl.ibm.com/zen/cp4d-builds/4.0.0/gmc1/ibm-cp-datacore/2.0.0-134/ibm-cp-datacore-2.0.0-134.tgz
wget http://icpfs1.svl.ibm.com/zen/cp4d-builds/4.0.0/gmc1/ibm-ccs/1.0.0-749/ibm-ccs-1.0.0-749.tgz
wget http://icpfs1.svl.ibm.com/zen/cp4d-builds/4.0.0/gmc1/ibm-datarefinery/1.0.0-238/ibm-datarefinery-1.0.0-238.tgz
wget http://icpfs1.svl.ibm.com/zen/cp4d-builds/4.0.0/gmc1/ibm-db2uoperator/4.0.0-3731.2361/ibm-db2uoperator-4.0.0-3731.2361.tgz
wget http://icpfs1.svl.ibm.com/zen/cp4d-builds/4.0.0/gmc1/ibm-db2aaservice/4.0.0-1228.749/ibm-db2aaservice-4.0.0-1228.749.tgz
wget http://icpfs1.svl.ibm.com/zen/cp4d-builds/4.0.0/gmc1/ibm-iis/4.0.0-355/ibm-iis-4.0.0-355.tgz
wget http://icpfs1.svl.ibm.com/zen/cp4d-builds/4.0.0/gmc1/ibm-wkc/4.0.0-416/ibm-wkc-4.0.0-416.tgz



export CASE_NAME=ibm-wkc
export CASE_VERSION=4.0.0-416
export CASE_ARCHIVE=${CASE_NAME}-${CASE_VERSION}.tgz
export CASE_INVENTORY_SETUP=wkcOperatorSetup
export OFFLINEDIR=/root/offline
export CASECTL_RESOLVE_DEPENDENCIES=false
export USE_SKOPEO=true
export PORTABLE_DOCKER_REGISTRY_HOST=api.wkc-devopsbox5.cp.fyre.ibm.com
export PORTABLE_DOCKER_REGISTRY_PORT=5000
export PORTABLE_DOCKER_REGISTRY=$PORTABLE_DOCKER_REGISTRY_HOST:$PORTABLE_DOCKER_REGISTRY_PORT
export PORTABLE_DOCKER_REGISTRY_USER=admin
export PORTABLE_DOCKER_REGISTRY_PASSWORD=adminpass
export PORTABLE_DOCKER_REGISTRY_PATH=/etc/docker/registry
cloudctl case launch \
  --case $OFFLINEDIR/${CASE_ARCHIVE} \
  --inventory $CASE_INVENTORY_SETUP \
  --action configure-creds-airgap \
  --args "--registry $PORTABLE_DOCKER_REGISTRY --user $PORTABLE_DOCKER_REGISTRY_USER --pass $PORTABLE_DOCKER_REGISTRY_PASSWORD" -t 1
cloudctl case launch \
 --case $OFFLINEDIR/${CASE_ARCHIVE} \
 --inventory $CASE_INVENTORY_SETUP \
 --action init-registry \
 --args "--registry $PORTABLE_DOCKER_REGISTRY_HOST --user $PORTABLE_DOCKER_REGISTRY_USER --pass $PORTABLE_DOCKER_REGISTRY_PASSWORD --dir $PORTABLE_DOCKER_REGISTRY_PATH" -t 1
cloudctl case launch \
 --case $OFFLINEDIR/${CASE_ARCHIVE} \
 --inventory $CASE_INVENTORY_SETUP \
 --action start-registry \
 --args "--registry $PORTABLE_DOCKER_REGISTRY_HOST --port $PORTABLE_DOCKER_REGISTRY_PORT --user $PORTABLE_DOCKER_REGISTRY_USER --pass $PORTABLE_DOCKER_REGISTRY_PASSWORD --dir $PORTABLE_DOCKER_REGISTRY_PATH" -t 1
 
 
- Tools
sudo yum install httpd-tools podman skopeo jq -y

yum install python3
alternatives --set python /usr/bin/python3
pip3 install pyyaml

podman pull docker.io/library/registry:2.6
podman tag localhost/registry:2.6 docker.io/library/registry:2.6
podman save -o registry.tar docker.io/library/registry:2.6
podman load --input /tmp/registry.tar


```
# Download Cases
wget http://icpfs1.svl.ibm.com/zen/cp4d-builds/4.0.0/gmc1/ibm-cpd-scheduling/1.2.0/ibm-cpd-scheduling-1.2.0.tgz


# IBM Foundataion and CPD Zen
ZEN_CASES="ibm-cp-datacore-2.0.0-134.tgz ibm-cp-common-services-1.4.0.tgz ibm-cpd-scheduling-1.2.0.tgz"
for CASE_FILE in $ZEN_CASES
do
  echo "cloudctl case save --case $OFFLINEDIR/${CASE_FILE}  --outputdir $OFFLINEDIR --tolerance=1"
done

 
# WKC Service

WKC_CASES="ibm-ccs-1.0.0-749.tgz  ibm-datarefinery-1.0.0-238.tgz ibm-db2uoperator-4.0.0-3731.2361.tgz ibm-db2aaservice-4.0.0-1228.749.tgz ibm-iis-4.0.0-355.tgz ibm-wkc-4.0.0-416.tgz"
for CASE_FILE in $WKC_CASES
do
 echo "cloudctl case save --case $OFFLINEDIR/${CASE_FILE}  --outputdir $OFFLINEDIR --tolerance=1"
done


- You need to Run the following commands one per registry (even if you more services)
- Save Credentials for Portable Registry
cloudctl case launch   --case $OFFLINEDIR/${CASE_ARCHIVE}   \
  --inventory $CASE_INVENTORY_SETUP   --action configure-creds-airgap  \
  --args "--registry $PORTABLE_DOCKER_REGISTRY --user $PORTABLE_DOCKER_REGISTRY_USER --pass $PORTABLE_DOCKER_REGISTRY_PASSWORD" \
  --tolerance=1

- Credentials for ic.io
cloudctl case launch --case /root/offline/ibm-wkc-4.0.0-416.tgz \
  --inventory wkcOperatorSetup  --action configure-creds-airgap \
  --args "--registry ${IBM_REGISTRY} --user ${IBM_REGISTRY_USER} --pass ${IBM_REGISTRY_TOKEN}"

- Credentials for cp.ic.io
cloudctl case launch  --case $OFFLINEDIR/${CASE_ARCHIVE} \
  --inventory $CASE_INVENTORY_SETUP --action configure-creds-airgap \
  --args "--registry cp.icr.io  --user ${IBM_REGISTRY_USER} --pass ${IBM_REGISTRY_TOKEN}" \
  -t 1

- Mirror Registry
- You need to Run only, it will create mirror for all cases

cloudctl case launch   --case $OFFLINEDIR/${CASE_ARCHIVE}   \
  --inventory $CASE_INVENTORY_SETUP   --action mirror-images \
  --args "--registry $PORTABLE_DOCKER_REGISTRY  --inputDir $OFFLINEDIR" \
  --t 1

```
**Bastion Inside the Networ**k


Setup a Private Registry



## Custom namespace

cloudctl case launch \
  --case $OFFLINEDIR/${CASE_ARCHIVE} \
  --inventory ${CASE_INVENTORY_SETUP} \
  --action configureClusterAirgap \
  --namespace common-service \
  --args "--registry $PORTABLE_DOCKER_REGISTRY --inputDir $OFFLINEDIR" -t 1
  
  
  This causes the Restart of the Nodes master/workers
  
  
