# Cloud Pak for Data 4.5 - OLM based Backup and Restore 


## Requirements
* [ ] OCS (CSI Based persistent storage)
* [ ] Restic backup on S3-compatible object store/Minio
* [ ] CPBDR tool
* [ ] OpenShift Cluster Admin permission


## Downloand and Install CPDBR tool

1. cpd-cli tool download


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


