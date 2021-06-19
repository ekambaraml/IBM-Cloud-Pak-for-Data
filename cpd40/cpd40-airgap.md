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
  
  
