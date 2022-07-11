# Cloud Pak for Data 4.5 - Backup and Restore using Volume Backup

This instruction is for backing up and restoring all volumes in the cloud pak for data instance namespace.


```mermaid
graph LR

    Installing-CPDBR --> Backup-Volumes[Backup Volumes in CPD namespace]
    Backup-Volumes --> Restore-Volumes


```




## Requirements

* [ ] NFS/S3 based storage PVC
* [ ] CPDBR Tool
* [ ] Cluster Admin permission for backup
## 2.0 Installing CPDBR Utility

### 2.1 Installing CPDBR tool
```
IMAGE_REGISTRY=`oc get route -n openshift-image-registry | grep image-registry | awk '{print $2}'`
echo $IMAGE_REGISTRY
NAMESPACE=cpd
echo $NAMESPACE
CPU_ARCH=`uname -m`
echo $CPU_ARCH
BUILD_NUM=`./cpd-cli backup-restore version | grep "Build Number" |cut -d : -f 2 | xargs`
echo $BUILD_NUM


# Pull cpdbr image from IBM Cloud Container Registry
podman pull icr.io/cpopen/cpd/cpdbr:4.0.0-${BUILD_NUM}-${CPU_ARCH}
# Push image to internal registry
podman login -u kubeadmin -p $(oc whoami -t) $IMAGE_REGISTRY --tls-verify=false
podman tag icr.io/cpopen/cpd/cpdbr:4.0.0-${BUILD_NUM}-${CPU_ARCH} $IMAGE_REGISTRY/$NAMESPACE/cpdbr:4.0.0-${BUILD_NUM}-${CPU_ARCH}
podman push $IMAGE_REGISTRY/$NAMESPACE/cpdbr:4.0.0-${BUILD_NUM}-${CPU_ARCH} --tls-verify=false


```

cpd-cli backup/restore utility version
```
./cpd-cli backup-restore version
backup-restore
	Version: 4.0.0
	Build Date: 2022-05-24T00:23:10
	Build Number: 927
	CPD Release Version: 4.5.0


```

### 2.2 Storage Volumes for Backup



```
cat <<EOF |oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cpdbr-pvc
  namespace: cpd
spec:
  storageClassName: managed-nfs-storage 
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 200Gi
EOF
```
* Check PVC status
    ```
    oc get pvc | grep cpdbr
    ```

### 2.3 Saving Repository secret
setup the repository secret for PVC

    ```
    # setup the repository secret for S3
    echo -n 'restic' > RESTIC_PASSWORD
    echo -n 'minio' > AWS_ACCESS_KEY_ID
    echo -n 'minio123' > AWS_SECRET_ACCESS_KEY
    cat /path/to/public.crt > CA_CERT_DATA

    oc create secret generic -n cpd cpdbr-repo-secret \
        --from-file=./RESTIC_PASSWORD \
        --from-file=./AWS_ACCESS_KEY_ID \
        --from-file=./AWS_SECRET_ACCESS_KEY \
        --from-file=./CA_CERT_DATA
    ```


## 3.0 Backup

* Initialize the cpdbr first with pvc name and s3 storage.  Note that the bucket must exist.
    ```
    ./cpd-cli backup-restore init -n cpd --pvc-name cpdbr-pvc --image-prefix=image-registry.openshift-image-registry.svc:5000/cpd --log-level=debug --verbose --provider=local



    ./cpd-cli backup-restore quiesce -n cpd

    ./cpd-cli backup-restore volume-backup create -n cpd --dry-run cpb-backup-01

    ./cpd-cli backup-restore volume-backup create -n cpd cpd-backup-01 --skip-quiesce=true

    ./cpd-cli backup-restore unquiesce -n cpd

    ./cpd-cli backup-restore volume-backup status -n cpd cpd-backup-01

    ./cpd-cli backup-restore volume-backup list -n cpd

    ./cpd-cli backup-restore volume-backup list --details -n cpd

    ./cpd-cli backup-restore volume-backup logs -n cpd cpd-backup-01


    ./cpd-cli backup-restore reset -n cpd --force
    ```     

### 4.0 Restore
```

./cpd-cli backup-restore quiesce -n cpd

./cpd-cli backup-restore volume-restore create --from-backup cpd-backup-01 -n cpd --dry-run cpd-restore-01


./cpd-cli backup-restore volume-restore create --from-backup cpd-backup-01 -n cpd cpd-restore-01 --skip-quiesce=true


./cpd-cli backup-restore volume-restore status -n cpd  cpd-restore-01

./cpd-cli backup-restore volume-restore list -n cpd

./cpd-cli backup-restore volume-restore logs -n cpd cpd-restore-01

./cpd-cli backup-restore reset -n cpd --force

```

