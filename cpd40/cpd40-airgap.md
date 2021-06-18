




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
  
  
