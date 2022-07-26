# Cloud Pak for Data Persistent volume storage migration

Cloud Pak for Data(CPD) uses persistent volumes to store application asset and data on Openshift cluster. CPD supports NFS, ODF/OCS, Portworx and other storage provisioners for dynamic Persistent volume creations. This instruction is for migrating the CPD clusters using NFS storage into ODF based storage. 

In this example we will be performing the in-place storage swap within the cluster.

##  Table of Contents
### [1.0 Architecture](https://github.ibm.com/CloudPakForDataSWATAssets/InstallPlusPlus/blob/master/backuprestore/nfs-ocs-migration.md#Arcitecture)
### [2.0 Requirements](https://github.ibm.com/CloudPakForDataSWATAssets/InstallPlusPlus/blob/master/backuprestore/nfs-ocs-migration.md#requirements)
### [3.0 Setup Client](https://github.ibm.com/CloudPakForDataSWATAssets/InstallPlusPlus/blob/master/backuprestore/nfs-ocs-migration.md#Setup)
### [4.0 Clone the CPD Instance](https://github.ibm.com/CloudPakForDataSWATAssets/InstallPlusPlus/blob/master/backuprestore/nfs-ocs-migration.md#requirements)
### [5.0 Change the Storage Class](https://github.ibm.com/CloudPakForDataSWATAssets/InstallPlusPlus/blob/master/backuprestore/nfs-ocs-migration.md#requirements)
### [6.0 Reinstate CPD Instance](https://github.ibm.com/CloudPakForDataSWATAssets/InstallPlusPlus/blob/master/backuprestore/nfs-ocs-migration.md#requirements)
### [7.0 Validate the CPD Instance](https://github.ibm.com/CloudPakForDataSWATAssets/InstallPlusPlus/blob/master/backuprestore/nfs-ocs-migration.md#requirements)
### [8.0 Cleanup](https://github.ibm.com/CloudPakForDataSWATAssets/InstallPlusPlus/blob/master/backuprestore/nfs-ocs-migration.md#requirements)
### [9.0 References](https://github.ibm.com/CloudPakForDataSWATAssets/InstallPlusPlus/blob/master/backuprestore/nfs-ocs-migration.md#requirements)

## 1.0 Architecture
![NFS-OCS](https://github.ibm.com/CloudPakForDataSWATAssets/InstallPlusPlus/blob/master/nfs-ocs-flow.png)

## 2.0 Requirements

   * [ ] OpenShift 4.8
   * [ ] CPD 4.0.x
   * [ ] CPD Cloner
   * [ ] Service Account
   * [ ] NFS Storage provisioner
   * [ ] ODF Storage for target 

         ODF storage should be installed with the following storage classes with sufficient storage capacity.
  
        - [ ] **ReadWriteMany** ocs-storagecluster-cephfs
        - [ ] **ReadWriteOnce** ocs-storagecluster-ceph-rbd 

* [ ] Storage for cloning the instance

   CPD deployment assets and PVC and PV contents are copied to this volume for cloning. This PVC volume should be sized based on the CPD Instance storage.


* [ ] Network access
```yaml
      Firewall access to download the packages for CPD Cloner
       - mirror.openshift.com
       - https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
```



## 3.0 Setup


### 3.1 CPD Cloner (cpc)

Clone CPC github repo and build the images
  
```sh
# install the pre-required utilities
yum install -y git 
yum install -y podman 
yum install -y jq

git clone github.ibm.com/CloudPakCloner/clonetool.git

cd clonetool-master

podman build -t clonetool:latest -f cpc/Dockerfile

podman build -t sincrr:latest -f sincrr/Dockerfile

```

### 3.2 Create CPC Project
```
# create new namespace for holding the CPD Cloner image
oc new-project zen-cpc
```

### 3.3 Create Service Account for CPC

```
oc create serviceaccount  clone-service-account -n zen

oc policy add-role-to-user system:image-puller system:serviceaccount:zen:clone-service-account -n zen-cpc
```

CPC jobs that will clone the PVC contents and metadata will be started into the CPD project that will be cloned or reinstated. To ensure the sincrr image can be pulled from the zen-cpc namespace in the internal registry, run the following command:


```sh
export project=<the project you want to clone or reinstate>
oc policy add-role-to-group system:image-puller system:serviceaccounts:$project --namespace zen-cpc
```
example:
```
export project=zen
oc policy add-role-to-user system:image-puller system:serviceaccount:$project:clone-service-account -n zen-cpc
```

### 3.4 push the sincrr image available from a registry

Make the sincrr image that is used by the cloner and reinstate jobs to be available from the image registry. If the customer don't have private registry, then use the OpenShift internal registry.  In this example we are demontrating using the openshift internal registry.


```sh
# images are pushed under the project zen-cpc, this should be created before pushing
REGISTRY=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
podman tag sincrr:latest ${REGISTRY}/zen-cpc/sincrr:latest
podman login ${REGISTRY} -u $(oc whoami) -p $(oc whoami -t) --tls-verify=false
podman push ${REGISTRY}/zen-cpc/sincrr:latest --tls-verify=false
```



### 3.5 Save CPD Operators setup (ibm-common-services)

capture current state of CPD deployments before scale them down. CPD will still be active until the cloner tool scale them down. create the script "capture-operator-status.sh"  to save number of replicas for the deployments to a file for later use. **This file should be preserved**



```
vi capture-operator-status.sh
```


```
# for specilised install, this namespace name will be different, example cpd-operators
oc project ibm-common-services
deployjson="deployments.json"
deploymentscsv="deployment_replicas.csv"

echo "Capture deployments state"
oc get deploy -ojson > "$deployjson"
cat "$deployjson" | jq -r '.items[]|[.metadata.name,.spec.replicas]|@csv' > "$deploymentscsv"
```



```
chmod +x capture-operator-status.sh
./capture-operator-status.sh
```

you should see a file called deployment_replicas.csv that contains all deployments and number of active replicas
```yaml
cat deployment_replicas.csv

"cert-manager-cainjector",1
"cert-manager-controller",1
"cert-manager-webhook",1
"configmap-watcher",1
"cpd-platform-operator-manager",1
"db2u-operator-manager",1
"ibm-cert-manager-operator",1
"ibm-common-service-operator",1
"ibm-common-service-webhook",1
"ibm-cpd-ccs-operator",1
"ibm-cpd-datarefinery-operator",1
"ibm-cpd-iis-operator",1
"ibm-cpd-wkc-operator",1
"ibm-db2aaservice-cp4d-operator-controller-manager",1
"ibm-namespace-scope-operator",1
"ibm-zen-operator",1
"meta-api-deploy",1
"operand-deployment-lifecycle-manager",1
"secretshare",1
```

### 3.6 Scaledown CPD Operators

This will bring down all the CPD Operators to 0. This is to prevent operators from interfering during the cloning operations.

```
vi scaledown-operators.sh
```

```
deploystates="deployment_replicas.csv"
while read d; do
     deployment=$(echo $d | cut -d',' -f 1 | sed 's/"//g')
     scaleto=0
     oc scale deploy $deployment --replicas ${scaleto}
done <${deploystates}
```
```
chmod +x scaledown-operators.sh
./scaledown-operators.sh
```
```yaml
deployment.apps/cert-manager-cainjector scaled
deployment.apps/cert-manager-controller scaled
deployment.apps/cert-manager-webhook scaled
deployment.apps/configmap-watcher scaled
deployment.apps/cpd-platform-operator-manager scaled
deployment.apps/db2u-operator-manager scaled
deployment.apps/ibm-cert-manager-operator scaled
deployment.apps/ibm-common-service-operator scaled
deployment.apps/ibm-common-service-webhook scaled
deployment.apps/ibm-cpd-ccs-operator scaled
deployment.apps/ibm-cpd-datarefinery-operator scaled
deployment.apps/ibm-cpd-iis-operator scaled
deployment.apps/ibm-cpd-wkc-operator scaled
deployment.apps/ibm-db2aaservice-cp4d-operator-controller-manager scaled
deployment.apps/ibm-namespace-scope-operator scaled
deployment.apps/ibm-zen-operator scaled
deployment.apps/meta-api-deploy scaled
deployment.apps/operand-deployment-lifecycle-manager scaled
deployment.apps/secretshare scaled
```

check all deployments are scaled down
```
oc get deploy

ex: oc get deploy -n ibm-common-services
```

wait for all pods to go down. It's fine if cert-manager pods remain with 0/1 running state
```
watch "oc get pods -n ibm-common-services"

```

```
oc project zen

oc get deploy
```

check the operator is trully down and not comming back up. Scale it back up after confirming
```
oc scale deploy zen-core --replicas=0

oc scale deploy zen-core --replicas=2
```



### 3.7 Create PVC for the clone/reinstate action

create a PVC for backup under the cpd instance namespace, example "zen". Once PVC is created check if it's bound
```yaml
vi cpc-pvc.yaml


apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
  name: cpd-claim
  namespace: zen
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  storageClassName: ocs-storagecluster-cephfs
```
```
oc apply -f cpc-pvc.yaml
```



### 4.0 Cloning the Cluster
The clonetool runs as a docker/podman container and its behaviour is dependent on the parameters passed as environment variables. You can specify all these on the command line via the -e option, but it is more convenient to update the parameters in an environment file.

Update the cpc_env.sh file in the config directory to match your clone/reinstate decisions, such as:

* The clone repository to use: PVC or S3 and their properties
* OpenShift server address
* Whether to use the OpenShift internal registry for the Cloud Pak images 
* Where to pull the sincrr image from



### 4.1 Update the CPC config/cpc_env.sh

CPC cloning tool uses the config/cpd_env.sh to for cloning operations. Based the action, the environment varible file parameter values should be updated. In this example, we will be using the PVC to store the cloned objects.

edit file cpc_env.sh. Change the **`CLONEREPOSITORY` to PVC**  and **`CLONEREPO_PVC` to cpd-claim** is the name of the PVC we created
```
vi config/cpc_env.sh
```

```
# CLONEREPOSITORY - This can be either S3 or PVC
CLONEREPOSITORY=PVC
CLONEREPO_PVC=cpd-claim
```


### 4.2 Clone zen to cpd-claim

cloning the cpd-instance project to a backup directory zen1-ppc in the PVC that is specified in environment file

- Save CPD deployment and sts to file


```
podman run -d \
   --env-file ./config/cpc_env.sh \
   -e ACTION=CLONE \
   -e PROJECT=zen \
   -e COSBUCKET=zen1-ppc \
   -e SERVER=$(oc whoami --show-server) \
   -e TOKEN=$(oc whoami -t) \
   clonetool
```

```
example:
podman run -d    --env-file ./config/cpc_env.sh    -e ACTION=CLONE    -e PROJECT=zen  -e SERVER=$(oc whoami --show-server)    -e TOKEN=$(oc whoami -t)    clonetool
```

Check progress
```
podman ps

podman logs --follow <container_id> 

```

Check status

[Full output](https://github.ibm.com/CloudPakForDataSWATAssets/InstallPlusPlus/blob/master/backuprestore/cpd-clone-output.log)

```
podman logs --follow c2ee72fc9ff2 
Contents of /tmp/clonedata inside the container: DATA
specs
state
[2022-07-14 17:40:56 INFO] Cloud pak cloner build number: 1. Last updated: 2-May-2021
W0714 17:40:56.337294      25 loader.go:223] Config not found: /tmp/kubeconfig
Logged into "https://api.swat-storage-migration.cp.fyre.ibm.com:6443" as "kube:admin" using the token provided.

You have access to 69 projects, the list has been suppressed. You can list all projects with ' projects'

Using project "default".
[2022-07-14 17:40:56 INFO] Wanting to clone
[2022-07-14 17:40:57 INFO] Start Time : 2022.07.14-17.40.56
[2022-07-14 17:40:57 INFO] I need to scale up
[2022-07-14 17:40:57 INFO] I need to scale down
Now using project "zen" on server "https://api.swat-storage-migration.cp.fyre.ibm.com:6443".
No resources found in zen namespace.
[2022-07-14 17:40:57 INFO] Starting job to clone metadata to PVC
error: unable to process template
  Required value: template.parameters[3]: parameter BACKUP_DIR is required and must be specified
error: no objects passed to create
[2022-07-14 17:40:57 INFO] Waiting for metadata cloning pod to become active (max 5 min)
Metadata pod ready: 
Metadata pod ready: 
Metadata pod ready: 
Metadata pod ready: 
Metadata pod ready: 
Metadata pod ready: 
Metadata pod ready: 
Metadata pod ready: 
Metadata pod ready: 
Metadata pod ready: 
Metadata pod ready: 
Metadata pod ready: 
Metadata pod ready: 
Metadata pod ready: 
Metadata pod ready: 
Metadata pod ready: 
Metadata pod ready: 
Metadata pod ready: 
Metadata pod ready: 
Metadata pod ready: 
Metadata pod ready: 

```


## 5.0 Swap the Storage Class

### 5.1 Change the NFS PV reclaim policy to "RETAIN"

In cpd instance namespace (zen), check the PV reclaim policy
```
oc get pv|grep -vi ocs|awk '{print $1}'
```

Update the reclaim policy from Delete to Retain
```
oc patch pv $(oc get pv|grep -vi ocs|awk '{print $1}') -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

Check that all the relevan PVs have changed their reclaim policy.  all NFS based CPD PVs are updated with "Retain" as reclaim policy to prevent the PV from deleting.

* Check the status
![status](https://github.ibm.com/CloudPakForDataSWATAssets/InstallPlusPlus/blob/master/backuprestore/pc-retain-status.png)

```sh
# oc get pv 
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                         STORAGECLASS                  REASON   AGE
local-pv-264a1e64                          600Gi      RWO            Delete           Bound    openshift-storage/ocs-deviceset-localblock-sc-0-data-4n57kc   localblock-sc                          5d23h
local-pv-4c5e91b9                          600Gi      RWO            Delete           Bound    openshift-storage/ocs-deviceset-localblock-sc-0-data-047n75   localblock-sc                          5d23h
local-pv-5f85275a                          600Gi      RWO            Delete           Bound    openshift-storage/ocs-deviceset-localblock-sc-0-data-15fsv4   localblock-sc                          5d23h
local-pv-94648197                          600Gi      RWO            Delete           Bound    openshift-storage/ocs-deviceset-localblock-sc-0-data-32f2gq   localblock-sc                          5d23h
local-pv-f1101638                          600Gi      RWO            Delete           Bound    openshift-storage/ocs-deviceset-localblock-sc-0-data-2cl6dl   localblock-sc                          5d23h
pvc-09edc477-5c69-45d3-9b19-0a0cbb233c7c   10Gi       RWO            Retain           Bound    zen/data-redis-ha-server-0                                    managed-nfs-storage                    5d17h
pvc-119ff324-a438-4af6-b8c4-38f75b303d96   30Gi       RWO            Retain           Bound    zen/database-storage-wdp-couchdb-0                            managed-nfs-storage                    5d17h
pvc-12848160-7be1-4b4d-8580-5f16c37806ac   20Gi       RWO            Retain           Bound    zen/0072-iis-dedicatedservices-pvc                            managed-nfs-storage                    5d14h
pvc-1b3e05a2-d60e-4442-b587-13d79ff10a84   10Gi       RWX            Retain           Bound    zen/user-home-pvc                                             managed-nfs-storage                    5d18h
pvc-2558dbc6-c732-4da1-921b-17117f82d2ba   10Gi       RWO            Retain           Bound    zen/datadir-zen-metastoredb-0                                 managed-nfs-storage                    5d18h
pvc-27452e61-7717-4cb1-bc68-9c60b982ec72   20Gi       RWX            Retain           Bound    zen/c-db2oltp-iis-meta                                        managed-nfs-storage                    5d14h
pvc-2e85267d-7f2a-481e-9532-095e63afe197   30Gi       RWX            Retain           Bound    zen/elasticsearch-master-backups                              managed-nfs-storage                    5d17h
pvc-30a99a91-f7c1-4f21-8577-5b4f8db02e13   40Gi       RWX            Retain           Bound    zen/c-db2oltp-wkc-data                                        managed-nfs-storage                    5d15h
pvc-3445ebdd-dd84-45ca-a000-5390b2c82df0   100Gi      RWO            Retain           Bound    zen/kafka-data-kafka-0                                        managed-nfs-storage                    5d14h
pvc-351b0028-1bb9-4196-aacf-f4b49b41ca98   40Gi       RWX            Retain           Bound    zen/c-db2oltp-iis-data                                        managed-nfs-storage                    5d14h
pvc-49b62454-33c0-4352-bebf-1da80aff8aaa   50Gi       RWX            Retain           Bound    zen/cc-home-pvc                                               managed-nfs-storage                    5d17h
pvc-4a7e423a-2437-4bf2-b872-9c99e7e88d61   10Gi       RWO            Retain           Bound    zen/data-redis-ha-server-2                                    managed-nfs-storage                    5d17h
pvc-53846dfd-1618-4431-84d7-2b97371c8e3f   90Gi       RWO            Retain           Bound    zen/cassandra-data-cassandra-0                                managed-nfs-storage                    5d14h
pvc-56da4564-7144-4c7b-9fd3-53ba5ecb1918   40Gi       RWX            Retain           Bound    zen/0072-iis-en-dedicated-pvc                                 managed-nfs-storage                    5d14h
pvc-5851f775-4585-4ecb-a5e4-0404b31a5410   30Gi       RWO            Retain           Bound    zen/database-storage-wdp-couchdb-1                            managed-nfs-storage                    5d17h
pvc-5af305d3-6978-4c8c-9355-3ec86d9cca0a   20Gi       RWX            Retain           Bound    zen/c-db2oltp-wkc-meta                                        managed-nfs-storage                    5d15h
pvc-5b0165cf-11fb-4b58-b675-d6f75809007f   30Gi       RWO            Retain           Bound    zen/elasticsearch-master-elasticsearch-master-0               managed-nfs-storage                    5d17h
pvc-6521331f-8366-46be-9e45-a008e36801f2   5Gi        RWO            Retain           Bound    zen/zookeeper-data-zookeeper-0                                managed-nfs-storage                    5d14h
pvc-71153708-1368-4a7e-bc80-3177dfc9e71d   10Gi       RWO            Retain           Bound    zen/data-rabbitmq-ha-0                                        managed-nfs-storage                    5d17h
pvc-75c1b57e-f17e-4c88-9bf6-aaffe090807e   100Mi      RWX            Retain           Bound    zen/0072-iis-sampledata-pvc                                   managed-nfs-storage                    5d14h
pvc-7d0121f7-541e-40c3-b799-6b5223c41132   10Gi       RWO            Retain           Bound    zen/data-rabbitmq-ha-2                                        managed-nfs-storage                    5d17h
pvc-7f8d5aff-c736-42c0-8cc7-67e343ced4fb   10Gi       RWO            Retain           Bound    zen/data-redis-ha-server-1                                    managed-nfs-storage                    5d17h
pvc-835d65e7-4d6b-4bda-8da5-e6375dda9a01   30Gi       RWO            Retain           Bound    zen/solr-data-solr-0                                          managed-nfs-storage                    5d14h
pvc-a72359d8-0451-4854-8a21-44c65b6b63d7   50Gi       RWX            Retain           Bound    zen/iis-db2u-backups                                          managed-nfs-storage                    5d14h
pvc-aa1f1374-a430-47c8-a2d0-3cf044d6c147   30Gi       RWO            Retain           Bound    zen/elasticsearch-master-elasticsearch-master-2               managed-nfs-storage                    5d17h
pvc-b1cdbdd9-deeb-4051-8817-6e4f3322a76c   10Gi       RWO            Retain           Bound    zen/datadir-zen-metastoredb-2                                 managed-nfs-storage                    5d18h
pvc-c4df298b-d12f-4d64-a230-a5a9616c3aed   10Gi       RWO            Retain           Bound    zen/datadir-zen-metastoredb-1                                 managed-nfs-storage                    5d18h
pvc-c9b7c5f6-1073-4ef6-8282-9ccf75e9311b   40Gi       RWX            Retain           Bound    zen/wkc-db2u-backups                                          managed-nfs-storage                    5d15h
pvc-d3ed7b7d-f2c4-4d6c-8ad2-52f1c6074732   5Gi        RWX            Retain           Bound    zen/0073-ug-omag-pvc                                          managed-nfs-storage                    5d13h
pvc-db778de1-4b45-44c8-8f75-ada75ef8015c   100Gi      RWX            Retain           Bound    zen/file-api-claim                                            managed-nfs-storage                    5d17h
pvc-e0fa7922-db07-43a2-862e-43d7cb7a9997   30Gi       RWO            Retain           Bound    zen/elasticsearch-master-elasticsearch-master-1               managed-nfs-storage                    5d17h
pvc-e6e17b06-90b7-46a4-bb45-7c596c1924eb   50Gi       RWO            Delete           Bound    openshift-storage/db-noobaa-db-pg-0                           ocs-storagecluster-ceph-rbd            5d23h
pvc-efa384fa-1294-4cde-b0bf-5f35558225e9   30Gi       RWO            Retain           Bound    zen/database-storage-wdp-couchdb-2                            managed-nfs-storage                    5d17h
pvc-f3ba9fc5-80ed-4139-9352-0ca8cd353161   100Gi      RWX            Retain           Bound    zen/cpd-claim                                                 ocs-storagecluster-cephfs              3d23h
pvc-f7de5d79-2f14-4a61-8b0b-dbb118362eae   10Gi       RWO            Retain           Bound    zen/data-dsx-influxdb-0                                       managed-nfs-storage                    5d18h
pvc-f7e40550-ad50-4303-8637-01ed3ea795d6   10Gi       RWO            Retain           Bound    zen/data-rabbitmq-ha-1                                        managed-nfs-storage                    5d17h
registry-storage                           200Gi      RWX            Retain           Bound    openshift-image-registry/image-registry-storage                                                      5d23h
```
```yaml
# oc get pods | grep -i clone
clone-job-0072-iis-dedicatedservices-pvc-0001-n652b       0/1     Completed          0          3d23h
clone-job-0072-iis-en-dedicated-pvc-0002-9cxsj            0/1     Completed          0          3d23h
clone-job-0072-iis-sampledata-pvc-0003-ppqx8              0/1     Completed          0          3d23h
clone-job-0073-ug-omag-pvc-0004-n572v                     0/1     Completed          0          3d23h
clone-job-c-db2oltp-iis-data-0005-5kgxq                   0/1     Completed          0          3d23h
clone-job-c-db2oltp-iis-meta-0006-sxz6s                   0/1     Completed          0          3d23h
clone-job-c-db2oltp-wkc-data-0007-lzcrm                   0/1     Completed          0          3d23h
clone-job-c-db2oltp-wkc-meta-0008-69lq4                   0/1     Completed          0          3d23h
clone-job-cassandra-data-cassandra-0-0009-zhfp8           0/1     Completed          0          3d23h
clone-job-cc-home-pvc-0010-lgfck                          0/1     Completed          0          3d23h
clone-job-data-dsx-influxdb-0-0011-slpnt                  0/1     Completed          0          3d23h
clone-job-data-rabbitmq-ha-0-0012-f54qv                   0/1     Completed          0          3d23h
clone-job-data-rabbitmq-ha-1-0013-gwz2r                   0/1     Completed          0          3d23h
clone-job-data-rabbitmq-ha-2-0014-w82cb                   0/1     Completed          0          3d23h
clone-job-data-redis-ha-server-0-0015-zsfcz               0/1     Completed          0          3d23h
clone-job-data-redis-ha-server-1-0016-vx5j6               0/1     Completed          0          3d23h
clone-job-data-redis-ha-server-2-0017-cxdtk               0/1     Completed          0          3d23h
clone-job-database-storage-wdp-couchdb-0-0018-7rd56       0/1     Completed          0          3d23h
clone-job-database-storage-wdp-couchdb-1-0019-7mssv       0/1     Completed          0          3d23h
clone-job-database-storage-wdp-couchdb-2-0020-r8vzp       0/1     Completed          0          3d23h
clone-job-datadir-zen-metastoredb-0-0021-gxq2b            0/1     Completed          0          3d23h
clone-job-datadir-zen-metastoredb-1-0022-ck4b4            0/1     Completed          0          3d23h
clone-job-datadir-zen-metastoredb-2-0023-57s5f            0/1     Completed          0          3d23h
clone-job-elasticsearch-master-backups-0024-5fgt6         0/1     Completed          0          3d23h
clone-job-elasticsearch-master-elasticse-0025-prtx6       0/1     Completed          0          3d23h
clone-job-elasticsearch-master-elasticse-0026-9wtsc       0/1     Completed          0          3d23h
clone-job-elasticsearch-master-elasticse-0027-gpzhv       0/1     Completed          0          3d23h
clone-job-file-api-claim-0028-59vss                       0/1     Completed          0          3d23h
clone-job-iis-db2u-backups-0029-lbggp                     0/1     Completed          0          3d23h
clone-job-kafka-data-kafka-0-0030-qmxq5                   0/1     Completed          0          3d23h
clone-job-solr-data-solr-0-0031-jrm7b                     0/1     Completed          0          3d23h
clone-job-user-home-pvc-0032-8s4j6                        0/1     Completed          0          3d23h
clone-job-wkc-db2u-backups-0033-nscl7                     0/1     Completed          0          3d23h
clone-job-zookeeper-data-zookeeper-0-0034-8v7v2           0/1     Completed          0          3d23h
```

### 5.2 Create POD to access Cloned objects

create the following cpd-clone-backup-pod.yaml script to create pod and mount the backup from CPD clone.
```
vi cpd-clone-backup-pod.yaml
```
cpd-clone-backup-pod.yaml file
```yaml
apiVersion: "v1"
kind: "Pod"
metadata:
  name: "mypod"
spec:
  serviceAccount: clone-service-account
  serviceAccountName: clone-service-account
  containers:
    -
      name: "myfrontend"
      image: "image-registry.openshift-image-registry.svc:5000/zen-cpc/sincrr:latest"
      volumeMounts:
        -
          mountPath: "/backup"
          name: "pvol"
  volumes:
    -
      name: "pvol"
      persistentVolumeClaim:
        claimName: "cpd-claim"
```
```
oc project zen
oc apply -f cpd-clone-backup-pod.yaml
```



check that the pod was created and that the pod has all the backup files
```
oc get pods | grep mypod

oc debug pod/mypod

cd /backup

ls zen1-ppc
```


### 5.3 Change the Storage class

Update the storage class name based on the AccessMode, corresponding the OCS storage class should be mapped in the  file "9-persistentvolumeclaims.json" from cpd-clone volume to local client. After the edit, it should be copied back to the cpd-claim PV before running the CPD Reinstate command.


Change Storge classes based on the Mapping
Find the sections with `ReadWriteOnce` and `ReadWriteMany` and replace the NFS storage class with OCS class. For `ReadWriteMany` is `ocs-storagecluster-cephfs` and for `ReadWriteOnce` is `ocs-storagecluster-ceph-rbd`

/backup/zen1-ppc/specs

ocs-storagecluster-ceph-rbd RW Once
ocs-storagecluster-cephfs RWMany

egrep 'ocs-storagecluster|ReadWriteOnce|ReadWriteMany' 9-persistentvolumeclaims.json

Take the file 9-persistentvolumeclaims.json out of the debug pod to the cluster and delete all the existing PVCs only and create new PVs with OCS storage class
```
oc delete -f 9-persistentvolumeclaims.json
```

egrep "ReadWriteMany|ReadWriteOnce|storageClassName" 9-persistentvolumeclaims.json 
```
egrep "ReadWriteMany|ReadWriteOnce|storageClassName" 9-persistentvolumeclaims.json 
                   "ReadWriteOnce"
        "storageClassName": "storagecluster-ceph-rbd",
          "ReadWriteMany"
        "storageClassName": "ocs-storagecluster-cephfs",
          "ReadWriteMany"
        "storageClassName": "ocs-storagecluster-cephfs",
          "ReadWriteMany"
        "storageClassName": "ocs-storagecluster-cephfs",
          "ReadWriteMany"
        "storageClassName": "ocs-storagecluster-cephfs",
          "ReadWriteMany"
        "storageClassName": "ocs-storagecluster-cephfs",
          "ReadWriteMany"
        "storageClassName": "ocs-storagecluster-cephfs",
          "ReadWriteMany"
        "storageClassName": "ocs-storagecluster-cephfs",
          "ReadWriteOnce"
        "storageClassName": "storagecluster-ceph-rbd",
          "ReadWriteMany"
        "storageClassName": "ocs-storagecluster-cephfs",
          "ReadWriteOnce"
        "storageClassName": "storagecluster-ceph-rbd",
          "ReadWriteOnce"
        "storageClassName": "storagecluster-ceph-rbd",
          "ReadWriteOnce"
        "storageClassName": "storagecluster-ceph-rbd",
          "ReadWriteOnce"
        "storageClassName": "storagecluster-ceph-rbd",
          "ReadWriteOnce"
        "storageClassName": "storagecluster-ceph-rbd",
          "ReadWriteOnce"
        "storageClassName": "storagecluster-ceph-rbd",
          "ReadWriteOnce"
        "storageClassName": "storagecluster-ceph-rbd",
          "ReadWriteOnce"
        "storageClassName": "storagecluster-ceph-rbd",
          "ReadWriteOnce"
        "storageClassName": "storagecluster-ceph-rbd",
          "ReadWriteOnce"
        "storageClassName": "storagecluster-ceph-rbd",
          "ReadWriteOnce"
        "storageClassName": "storagecluster-ceph-rbd",
          "ReadWriteOnce"
        "storageClassName": "storagecluster-ceph-rbd",
          "ReadWriteOnce"
        "storageClassName": "storagecluster-ceph-rbd",
          "ReadWriteMany"
        "storageClassName": "ocs-storagecluster-cephfs",
          "ReadWriteOnce"
        "storageClassName": "storagecluster-ceph-rbd",
          "ReadWriteOnce"
        "storageClassName": "storagecluster-ceph-rbd",
          "ReadWriteOnce"
        "storageClassName": "storagecluster-ceph-rbd",
          "ReadWriteMany"
        "storageClassName": "ocs-storagecluster-cephfs",
          "ReadWriteMany"
        "storageClassName": "ocs-storagecluster-cephfs",
          "ReadWriteOnce"
        "storageClassName": "storagecluster-ceph-rbd",
          "ReadWriteOnce"
        "storageClassName": "storagecluster-ceph-rbd",
          "ReadWriteMany"
        "storageClassName": "ocs-storagecluster-cephfs",
          "ReadWriteMany"
        "storageClassName": "ocs-storagecluster-cephfs",
          "ReadWriteOnce"
        "storageClassName": "storagecluster-ceph-rbd",

```
```
c delete -f 9-persistentvolumeclaims.json
persistentvolumeclaim "0072-iis-dedicatedservices-pvc" deleted
persistentvolumeclaim "0072-iis-en-dedicated-pvc" deleted
persistentvolumeclaim "0072-iis-sampledata-pvc" deleted
persistentvolumeclaim "0073-ug-omag-pvc" deleted
persistentvolumeclaim "c-db2oltp-iis-data" deleted
persistentvolumeclaim "c-db2oltp-iis-meta" deleted
persistentvolumeclaim "c-db2oltp-wkc-data" deleted
persistentvolumeclaim "c-db2oltp-wkc-meta" deleted
persistentvolumeclaim "cassandra-data-cassandra-0" deleted
persistentvolumeclaim "cc-home-pvc" deleted
persistentvolumeclaim "cpd-claim" deleted
persistentvolumeclaim "data-dsx-influxdb-0" deleted
persistentvolumeclaim "data-rabbitmq-ha-0" deleted
persistentvolumeclaim "data-rabbitmq-ha-1" deleted
persistentvolumeclaim "data-rabbitmq-ha-2" deleted
persistentvolumeclaim "data-redis-ha-server-0" deleted
persistentvolumeclaim "data-redis-ha-server-1" deleted
persistentvolumeclaim "data-redis-ha-server-2" deleted
persistentvolumeclaim "database-storage-wdp-couchdb-0" deleted
persistentvolumeclaim "database-storage-wdp-couchdb-1" deleted
persistentvolumeclaim "database-storage-wdp-couchdb-2" deleted
persistentvolumeclaim "datadir-zen-metastoredb-0" deleted
persistentvolumeclaim "datadir-zen-metastoredb-1" deleted
persistentvolumeclaim "datadir-zen-metastoredb-2" deleted
persistentvolumeclaim "elasticsearch-master-backups" deleted
persistentvolumeclaim "elasticsearch-master-elasticsearch-master-0" deleted
persistentvolumeclaim "elasticsearch-master-elasticsearch-master-1" deleted
persistentvolumeclaim "elasticsearch-master-elasticsearch-master-2" deleted
persistentvolumeclaim "file-api-claim" deleted
persistentvolumeclaim "iis-db2u-backups" deleted
persistentvolumeclaim "kafka-data-kafka-0" deleted
persistentvolumeclaim "solr-data-solr-0" deleted
persistentvolumeclaim "user-home-pvc" deleted
persistentvolumeclaim "wkc-db2u-backups" deleted
Error from server (NotFound): persistentvolumeclaims "zookeeper-data-zookeeper-0" not found
```


make sure no jobs or pods using the PVCs are not running. If yer, you need to delete those pods

### update "9-persistentvolumeclaims.json" file with new storage classes and copy it back to the cpd-claim pvc storage location


## 6.0 CPC Reinstate "CPD Namespace"

make a copy of cpc_env.sh and edit the file. Set the following variables to false `SYNCIMAGES=False` `SETKERNELPARAMS=False` `SETNOROOTSQUASH=False`

### 6.1 Reinstate
Run the reinstate command to start deployment restore
```
podman run -d \
   --env-file ./config/cpc_env.sh \
   -e ACTION=REINSTATE \
   -e PROJECT=cpd \
   -e NEWPROJECT=cpd \
   -e BACKUP_DIR=zen1-ppc \
   -e SERVER=$(oc whoami --show-server) \
   -e TOKEN=$(oc whoami -t) \
   clonetool
```
```
podman ps -a
podman logs eloquent_saha
```

If you still have active pods showing up, delete them
```
oc delete po $(oc get pod |grep -i completed|awk '{print $1}')

oc delete po $(oc get pod |grep -i error|awk '{print $1}')
```

Suspend cornjob. Change variable to true `suspend: true`
```
# suspend all cronjobs
oc patch cronjobs $(oc get cronjobs|awk '{print $1}') -p '{"spec":{"suspend" : true}}'

# suspend cronjobs one by one
oc get cronjobs

oc edit conrjobs env-spec-sync-job
```

Delete jobs
```
oc get jobs|grep env-spec

oc delete job <pods_name>
```

Run comand to reinstate again and check the status. Ignore errors (it'll try to create deloyments and statefull sets that are already there)

```
podman ps

podman logs -logs -f thirsty_hellman
```

Delete backup
```
oc get job |grep cpd-claim

oc delete job <job_name>
```


### 6.2 Scale back Services

Scale services back up with scaleup-cpd-services.sh script

```
deploystates="cpd-deployment_replicas.csv"
stsstates="cpd-sts_replicas.csv"

while read d; do
     deployment=$(echo $d | cut -d',' -f 1 | sed 's/"//g')
     scaleto=$(echo $d | cut -d',' -f 2 | sed 's/"//g')
     oc scale deploy $deployment --replicas ${scaleto}
done <${deploystates}

while read s; do
     sts=$(echo $s | cut -d',' -f 1 | sed 's/"//g')
     scaleto=$(echo $s | cut -d',' -f 2 | sed 's/"//g')
     oc scale sts ${sts} --replicas ${scaleto}
done <${stsstates}
```

## 7.0 Validate the cluster
[This section should contain the steps to validate the cluster after the storage swap]

## 8.0 Cleanup
[This section should contain the cleanup after the storage migration]
## References

* [Cloner Tool](https://github.ibm.com/CloudPakCloner/clonetool)


podman run -d \
   --env-file ~/cpc/config/cpc_env.sh \
   -e ACTION=REINSTATE \
   -e PROJECT=cpd \
   -e NEWPROJECT=cpd \
   -e BACKUP_DIR=zen1-ppc \
   -e SERVER=$(oc whoami --show-server) \
   -e TOKEN=$(oc whoami -t) \
   clonetoo
   
   
   
   
   oc patch cronjobs $(oc get cronjobs|awk '{print $1}') -p '{"spec":{"suspend" : true}}'
   
   oc patch cronjobs $(oc get cronjobs|awk '{print $1}') -p '{"spec":{"suspend" : false}}'
   
   
