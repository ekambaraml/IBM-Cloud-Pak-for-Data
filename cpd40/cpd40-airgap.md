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
Project Admins | An administrator of the following projects: 1. The IBM Cloud Pak for Data platform operator project (cpd-operators), 2. The project where you plan to install Cloud Pak for Data

### Getting Product and Installers
   - This section of the steps should be run on a internet facing client machine to obtain the tools and images required for the installations.

- Client Machine Requirements
  - RHEL 8.x Linux (Preferred)
  - skope - 1.2 or latest
  - openssl 1.11 or latest


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

# Install tools
sudo yum install httpd-tools podman skopeo jq -y

yum install python3
alternatives --set python /usr/bin/python3
pip3 install pyyaml

```

- Cloud Pak for Data
```
export OFFLINEDIR=${HOMEDIR}/offline
mkdir -p  ${OFFLINEDIR}
cd ${OFFLINEDIR}
export CASECTL_RESOLVE_DEPENDENCIES=false

# IBM Foundation Services
# (Cloud Pak for Data, IBM Foundation Service and CPD Schduling)
wget http://icpfs1.svl.ibm.com/zen/cp4d-builds/4.0.0/gmc1/ibm-cp-datacore/2.0.0-134/ibm-cp-datacore-2.0.0-134.tgz
get https://github.com/IBM/cloud-pak/blob/master/repo/case/ibm-cp-common-services-1.4.0.tgz?raw=true -O ibm-cp-common-services-1.4.0.tgz
wget http://icpfs1.svl.ibm.com/zen/cp4d-builds/4.0.0/gmc1/ibm-cpd-scheduling/1.2.0/ibm-cpd-scheduling-1.2.0.tgz

cloudctl case save --case ${OFFLINEDIR}/ibm-cp-datacore-2.0.0-134.tgz --outputdir ${OFFLINEDIR}  -t 1
cloudctl case save --case ${OFFLINEDIR}/ibm-cp-common-services-1.4.0.tgz  --outputdir ${OFFLINEDIR}  -t 1
cloudctl case save --case ${OFFLINEDIR}/ibm-cpd-scheduling-1.2.0.tgz --outputdir ${OFFLINEDIR}  -t 1

# Watson Knowledge Catelog and required cases
# (Common Core Services, Data Refinery, DB2U Operator, DB2 as a Service, IBM Information Server, Watson Knowledge Catalog)

wget http://icpfs1.svl.ibm.com/zen/cp4d-builds/4.0.0/gmc1/ibm-ccs/1.0.0-749/ibm-ccs-1.0.0-749.tgz
wget http://icpfs1.svl.ibm.com/zen/cp4d-builds/4.0.0/gmc1/ibm-datarefinery/1.0.0-238/ibm-datarefinery-1.0.0-238.tgz
wget http://icpfs1.svl.ibm.com/zen/cp4d-builds/4.0.0/gmc1/ibm-db2uoperator/4.0.0-3731.2361/ibm-db2uoperator-4.0.0-3731.2361.tgz
wget http://icpfs1.svl.ibm.com/zen/cp4d-builds/4.0.0/gmc1/ibm-db2aaservice/4.0.0-1228.749/ibm-db2aaservice-4.0.0-1228.749.tgz
wget http://icpfs1.svl.ibm.com/zen/cp4d-builds/4.0.0/gmc1/ibm-iis/4.0.0-355/ibm-iis-4.0.0-355.tgz
wget http://icpfs1.svl.ibm.com/zen/cp4d-builds/4.0.0/gmc1/ibm-wkc/4.0.0-416/ibm-wkc-4.0.0-416.tgz

cloudctl case save --case ${OFFLINEDIR}/ibm-ccs-1.0.0-749.tgz --outputdir ${OFFLINEDIR}  -t 1
cloudctl case save --case ${OFFLINEDIR}/ibm-datarefinery-1.0.0-238.tgz --outputdir ${OFFLINEDIR}  -t 1
cloudctl case save --case ${OFFLINEDIR}/ibm-db2uoperator-4.0.0-3731.2361.tgz --outputdir ${OFFLINEDIR}  -t 1
cloudctl case save --case ${OFFLINEDIR}/ibm-db2aaservice-4.0.0-1228.749.tgz  --outputdir ${OFFLINEDIR}  -t 1
cloudctl case save --case ${OFFLINEDIR}/ibm-iis-4.0.0-355.tgz --outputdir ${OFFLINEDIR}  -t 1
cloudctl case save --case ${OFFLINEDIR}/ibm-wkc-4.0.0-416.tgz --outputdir ${OFFLINEDIR}  -t 1
# end of download

```


- Portabl Registry Mirror Setup
Please update the following values based on your environments

```


export USE_SKOPEO=true
export PORTABLE_DOCKER_REGISTRY_HOST=10.17.5.154
export PORTABLE_DOCKER_REGISTRY_PORT=5000
export PORTABLE_DOCKER_REGISTRY=$PORTABLE_DOCKER_REGISTRY_HOST:$PORTABLE_DOCKER_REGISTRY_PORT
export PORTABLE_DOCKER_REGISTRY_USER=admin
export PORTABLE_DOCKER_REGISTRY_PASSWORD=adminpass
export PORTABLE_DOCKER_REGISTRY_PATH=/etc/docker/registry


export IBM_REGISTRY_USER=cp
export IBM_REGISTRY_TOKEN=<TBD>
export IBM_REGISTRY=us.icr.io

```

- Initialise portable Registry container
```
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
 
 # verify the local registry
 podman login -u $PORTABLE_DOCKER_REGISTRY_USER -p $PORTABLE_DOCKER_REGISTRY_PASSWORD $PORTABLE_DOCKER_REGISTRY --tls-verify=false
 
 ```
 

- Save Credentials for Portable Registry
You need to Run the following commands one per entitled registry (cp.icr.io, c

```
cloudctl case launch   --case $OFFLINEDIR/${CASE_ARCHIVE}   \
  --inventory $CASE_INVENTORY_SETUP   --action configure-creds-airgap  \
  --args "--registry $PORTABLE_DOCKER_REGISTRY --user $PORTABLE_DOCKER_REGISTRY_USER --pass $PORTABLE_DOCKER_REGISTRY_PASSWORD" \
  --tolerance=1

- Credentials for ic.io
# Set the IBM Registry Entitlement token before running the command, This need be obtained from myibm.ibm.com

cloudctl case launch --case $OFFLINEDIR/${CASE_ARCHIVE} \
  --inventory wkcOperatorSetup  --action configure-creds-airgap \
  --args "--registry ${IBM_REGISTRY} --user ${IBM_REGISTRY_USER} --pass ${IBM_REGISTRY_TOKEN}"

- Credentials for cp.ic.io
cloudctl case launch  --case $OFFLINEDIR/${CASE_ARCHIVE} \
  --inventory $CASE_INVENTORY_SETUP --action configure-creds-airgap \
  --args "--registry cp.icr.io  --user ${IBM_REGISTRY_USER} --pass ${IBM_REGISTRY_TOKEN}" \
  -t 1

```

- Run the mirroring
  - This process will take few hours depends on your external network connectivity speeds. This dowdloads and transfer all images into local portalble registry.

```

cloudctl case launch   --case $OFFLINEDIR/ibm-cp-datacore-2.0.0-134.tgz   \
  --inventory cpdPlatformOperator   \
  --action mirror-images \
  --args "--registry $PORTABLE_DOCKER_REGISTRY  --inputDir $OFFLINEDIR" \
  --t 1

cloudctl case launch   --case $OFFLINEDIR/ibm-cpd-scheduling-1.2.0.tgz   \
  --inventory schedulerSetup   \
  --action mirror-images \
  --args "--registry $PORTABLE_DOCKER_REGISTRY  --inputDir $OFFLINEDIR" \
  --t 1

cloudctl case launch   --case $OFFLINEDIR/ibm-cp-common-services-1.4.0.tgz   \
  --inventory ibmCommonServiceOperatorSetup   \
  --action mirror-images \
  --args "--registry $PORTABLE_DOCKER_REGISTRY  --inputDir $OFFLINEDIR" \
  --t 1
  
cloudctl case launch   --case $OFFLINEDIR/ibm-ccs-1.0.0-749.tgz   \
  --inventory ccsSetup   \
  --action mirror-images \
  --args "--registry $PORTABLE_DOCKER_REGISTRY  --inputDir $OFFLINEDIR" \
  --t 1
  
cloudctl case launch   --case $OFFLINEDIR/ibm-datarefinery-1.0.0-238.tgz   \
  --inventory datarefinerySetup   \
  --action mirror-images \
  --args "--registry $PORTABLE_DOCKER_REGISTRY  --inputDir $OFFLINEDIR" \
  --t 1

cloudctl case launch   --case $OFFLINEDIR/ibm-db2aaservice-4.0.0-1228.712.tgz   \
  --inventory db2aaserviceOperatorSetup   \
  --action mirror-images \
  --args "--registry $PORTABLE_DOCKER_REGISTRY  --inputDir $OFFLINEDIR" \
  --t 1
  
cloudctl case launch   --case $OFFLINEDIR/ibm-db2uoperator-4.0.0-3731.2361.tgz   \
  --inventory db2uOperatorSetup   \
  --action mirror-images \
  --args "--registry $PORTABLE_DOCKER_REGISTRY  --inputDir $OFFLINEDIR" \
  --t 1
  
cloudctl case launch   --case $OFFLINEDIR/ibm-iis-4.0.0-355.tgz   \
  --inventory iisOperator   \
  --action mirror-images \
  --args "--registry $PORTABLE_DOCKER_REGISTRY  --inputDir $OFFLINEDIR" \
  --t 1  
  
cloudctl case launch   --case $OFFLINEDIR/ibm-wkc-4.0.0-416.tgz   \
  --inventory wkcOperatorSetup   \
  --action mirror-images \
  --args "--registry $PORTABLE_DOCKER_REGISTRY  --inputDir $OFFLINEDIR" \
  --t 1
  


```

- Verify and export the portable registry
  - Check the images
  ```
  curl -k -u $PORTABLE_DOCKER_REGISTRY_USER:$PORTABLE_DOCKER_REGISTRY_PASSWORD https://${PORTABLE_DOCKER_REGISTRY}/v2/_catalog?n=6000 | jq .repositories[]
  ```
  - Portable Registry Export
  ```
    podman save docker.io/library/registry:2.6 -o $OFFLINEDIR/cpdregistry.tar
  ```

### Part 2: Copy Images from portable registry to private registry
```
export PRIVATE_REGISTRY_URL=9.46.199.99:5000
export PRIVATE_REGISTRY_USER=admin
export PRIVATE_REGISTRY_PASS=adminpass

# Test the login
podman login -u $PRIVATE_REGISTRY_USER -p $PRIVATE_REGISTRY_PASS $PRIVATE_REGISTRY_URL --tls-verify=false

# Save private registry credentials

cloudctl case launch   --case $OFFLINEDIR/ibm-cp-datacore-2.0.0-134.tgz    \
  --inventory cpdPlatformOperator   --action configure-creds-airgap  \
  --args "--registry $PRIVATE_REGISTRY_URL --user $PRIVATE_REGISTRY_USER --pass $PRIVATE_REGISTRY_PASS" \
  --tolerance=1
  
# Copy images to private registry
cloudctl case launch \
  --case $OFFLINEDIR/ibm-cp-datacore-2.0.0-134.tgz  \
  --inventory cpdPlatformOperator \
  --action mirror-images \
  --args "--fromRegistry $PORTABLE_DOCKER_REGISTRY  --registry $PRIVATE_REGISTRY_URL --inputDir $OFFLINEDIR" \
  --t 1
  
```

### Part 2: Installing the Watson Knowledge Catalog

#### Pre-Installation tasks
1. Installing OpenShift Container Platform
2. Setting up Persistent Storage
3. Creating Project namespaces
4. Obtainting prerequisites
5. Mirror images to you container registry
6. Configuring your cluster to pull Cloud Pak for Data images
7. Installing Cloud Pak foundational services
8. Creating operator subscriptions
9. Creating Custom sercurity context constraints for services
10. Changing required node settings


- Create projects
  - $ oc new-project common-service
  - $ oc new-project zen

- Create Global pullsecret for local registry
```
oc extract secret/pull-secret -n openshift-config
.dockerconfigjson

export REGISTRY_USER=admin
export REGISTRY_PASSWORD=adminpass
export REGISTRY_SERVER=10.17.101.119:5000



pull_secret=$(echo -n "$REGISTRY_USER:$REGISTRY_PASSWORD" | base64 -w0)

oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | sed -e 's|:{|:{"10.17.101.119:5000":{"auth":"'$pull_secret'"\},|' > /tmp/dockerconfig.json

oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=/tmp/dockerconfig.json
```

- Create a configmap






## Custom namespace
```
cloudctl case launch \
  --case $OFFLINEDIR/ibm-cp-datacore-2.0.0-134.tgz \
  --inventory cpdPlatformOperator \
  --action configureClusterAirgap \
  --namespace common-service \
  --args "--registry $PORTABLE_DOCKER_REGISTRY --inputDir $OFFLINEDIR" -t 1
  
  
  This causes the Restart of the Nodes master/workers
  
  
