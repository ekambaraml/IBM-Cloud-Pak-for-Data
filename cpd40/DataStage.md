#1 Cluster Preparations

- Cluster Compute sizing 
  - Node disk size
- Persistent Storage
- Disk Performance
    ```
    Disk latency test
    dd if=/dev/zero of=/PVC_mount_path/testfile bs=4096 count=1000 oflag=dsync

    The value must be comparable to or better than: 2.5 MB/s.

    Disk throughput test
    dd if=/dev/zero of=/PVC_mount_path/testfile bs=1G count=1 oflag=dsync

    The value must be comparable to or better than: 209 MB/s.
    ```

- Node Configurations
  - Crio
  - Kernel
  - Node tuning

    oc label machineconfigpool worker db2u-kubelet=sysctl

    cat << EOF | oc apply -f -
    apiVersion: machineconfiguration.openshift.io/v1
    kind: KubeletConfig
    metadata:
    name: db2u-kubelet
    spec:
    machineConfigPoolSelector:
        matchLabels:
        db2u-kubelet: sysctl
    kubeletConfig:
        allowedUnsafeSysctls:
        - "kernel.msg*"
        - "kernel.shm*"
        - "kernel.sem"
    EOF


# IBM Foundation and CP4D

# Deploying DatatStage on CPD 4.0



```
[root@cxbastion81 cxgov]# oc get pvc
NAME                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
data-dsx-influxdb-0         Bound    pvc-664c1b7d-046f-4203-b75c-2464d3b700c5   10Gi       RWO            managed-nfs-storage   19h
datadir-zen-metastoredb-0   Bound    pvc-fd00cc7a-a6cc-4ba2-aa5b-e652421f14f5   10Gi       RWO            managed-nfs-storage   19h
datadir-zen-metastoredb-1   Bound    pvc-256a018a-4475-444a-8e63-5deb1215a6cc   10Gi       RWO            managed-nfs-storage   19h
datadir-zen-metastoredb-2   Bound    pvc-ad061b3a-f7d8-48db-96d1-beb63af0dfd8   10Gi       RWO            managed-nfs-storage   19h
user-home-pvc               Bound    pvc-b27a150f-9eec-480f-a76b-9ce2670c5d06   10Gi       RWX            managed-nfs-storage   19h
```

```
wget https://github.com/IBM/cloud-pak-cli/releases/download/v3.7.0/cloudctl-linux-amd64.tar.gz

oc -n ibm-common-services get pods | grep db2u-operator

```

# Service Catalog Source
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ibm-operator-catalog
  namespace: openshift-marketplace
spec:
  displayName: "IBM Operator Catalog" 
  publisher: IBM
  sourceType: grpc
  image: icr.io/cpopen/ibm-operator-catalog:latest
  updateStrategy:
    registryPoll:
      interval: 45m
EOF


# Installing DB2 as service
oc -n cpd  get Db2aaserviceService

# DB2 operator
oc -n ibm-common-services get pods | grep ibm-db2aaservice-cp4d-operator-controller-manager

oc -n cpd  get Db2aaserviceService db2aaservice-cr -o=jsonpath='{$.status.db2aaserviceStatus}'
Completed

# Install InfoSphere Information Server
export CASE_SAVED_DIR=$(pwd)/case_saved
export CASE_REPO_PATH=https://github.com/IBM/cloud-pak/raw/master/repo/case
export IIS_CASE=ibm-iis-4.0.0.tgz

cloudctl case launch --case ${CASE_SAVED_DIR}/${IIS_CASE} --tolerance 1 --namespace ibm-common-services --action installOperatorNative --inventory iisOperatorSetup
oc get pod -n ibm-common-services | grep ibm-cpd-iis-operator

oc create -f iis-cr.yaml

oc get IIS iis-cr -o=jsonpath='{$.status.iisStatus}'

watch "oc get pods | egrep -iv '1/1|2/2|3/3|4/4|completed'"



### Check that the ibm-cpd-iis-operator pod is up and running
oc get pod -n ibm-common-services | grep ibm-cpd-iis-operator



## Uninstalling DataStage
```
oc delete  DataStageService datastage-cr
oc delete IIS iis-cr
oc delete Db2aaserviceService db2aaservice-cr

```
