# Cloud Pak for Data 4.5 - OLM based Backup and Restore 


## Requirements
* [ ] OCP 4.6
* [ ] OADP official RedHat 1.0 Operator
* [ ] cpdbr-velero-plugin
* [ ] OCS (CSI Based persistent storage)
* [ ] Restic backup on S3-compatible object store/Minio
* [ ] CPBDR tool
* [ ] OpenShift Cluster Admin permission


## Downloand and Install CPDBR tool


```
./oc version
Client Version: 4.8.35
Server Version: 4.8.43
Kubernetes Version: v1.21.11+6b3cbdd
```

1. cpd-cli tool download
2. 

## Backing up an instance of Cloud Pak for Data



```
cpd-cli oadp backup create \
--include-namespaces=${PROJECT_CPD_INSTANCE} \
--exclude-resources='Event,Event.events.k8s.io' \
--default-volumes-to-restic \
--cleanup-completed-resources <instance_backup_name> \
--log-level=debug \
--verbose
```

### Backing up the operators and operator project (Express Installation)

* [ ] Operator namespace: ibm-common-services
* [ ] CPD Instance: cpd


