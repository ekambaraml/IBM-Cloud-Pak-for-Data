# Persistent volume storage migration for "Cloud Pak for Data"

Cloud Pak for Data(CPD) uses persistent volumes to store application asset and data on Openshift cluster. CPD supports NFS, ODF/OCS, Portworx and other storage provisioners for dynamic Persistent volume creations. This instruction is for migrating the CPD clusters using NFS storage into ODF based storage. 

In this example we will be performing the in-place storage swap within the cluster.

##  Table of Contents
### [1.0 Architecture](#10-architecture-1)
### [2.0 Requirements](#20-requirement-checklist)
### [3.0 Setup Client](#30-setup)
### [4.0 Clone the CPD Instance](#40-cloning-the-cluster)
### [5.0 Change the Storage Class](#50-swap-the-storage-class)
### [6.0 Reinstate CPD Instance](#60-reinstate)
### [7.0 Validate the CPD Instance](#70-validate-the-cluster)
### [8.0 Cleanup](#80-cleanup-1)
### [9.0 References](#90-references-1)

## 1.0 Architecture

Architecture Pattern: Persistent Storage Migration

![NFS-OCS](https://github.ibm.com/CloudPakForDataSWATAssets/InstallPlusPlus/blob/master/nfs-ocs-flow.png)


## 2.0 Requirement Checklist

   * [ ] OpenShift 4.8
   * [ ] CPD 4.0.x
   * [ ] CPD Cloner
   * [ ] User and Permission
   * [ ] Service Account
   * [ ] NFS Storage provisioner
   * [ ] ODF Storage for target 
   * [ ] Storage for cloning the instance
   * [ ] Network access



## 3.0 Setup


### 3.1 Storage Class
This inplace storage migration requires both NFS and OCS storage setup and storage classes.

Check the cluster to verify the storage class:

```
oc get sc | egrep 'nfs|ocs'
managed-nfs-storage           nfs-provisioner/nfs                     Delete          Immediate              false                  21d
ocs-storagecluster-ceph-rbd   openshift-storage.rbd.csi.ceph.com      Delete          Immediate              true                   21d
ocs-storagecluster-ceph-rgw   openshift-storage.ceph.rook.io/bucket   Delete          Immediate              false                  21d
ocs-storagecluster-cephfs     openshift-storage.cephfs.csi.ceph.com   Delete          Immediate              true                   21d
```

Cloud Pak for Data uses two different OCS storage classes based on the Access Mode requirements.. For ReadWriteOnce(RWO) PVs will use the storage class "ocs-storagecluster-ceph-rbd" and ReadWriteMany will use the "ocs-storagecluster-cephfs"
```
ReadWriteOnce(RWO) - ocs-storagecluster-ceph-rbd
ReadWriteMany(RWX) - ocs-storagecluster-cephfs
```

### 3.2 Network Access
Building the CPD cloner tool require access to following URL. But IBM may be able to push this images to quay.io for the client to download..

If you are building the cloner tool, you need access to the following registry and repository.
- mirror.openshift.com
- https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm


### 3.3 Storage for storing PVC backup
CPD deployment assets and persistent volume contents will be backuped into this PVC for restore. This PVC size should be more than the total persistent storages used by the current NFS based CPD Instance.  Create this PVC for backup under the cpd instance namespace, example "zen".  CPC tool requires the PVC name as "cpc-backup-pvc".


```yaml
vi cpc-pvc.yaml


apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
  name: cpc-backup-pvc
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

oc get pvc | grep cpc-backup-pvc
```

change the Reclaim policy to "Retain"
```
oc edit pvc cpc-backup-pvc
```



### 3.4 CPC Project
This project namespace is used for storing the CPC images.
```
# create new namespace for holding the CPD Cloner image
oc new-project zen-cpc
```

### 3.5 User Permission

Storage management requires OpenShift admin permission. Please make sure you are using the ocp admin level user. The following commands can be used for granting cluster admin permission to your account.

Add admin as cluster-admin
```
oc adm policy add-cluster-role-to-user cluster-admin <ocadmin>
```
### 3.6 Service Account(SA)

CPC tool uses "clone-service-account" service account cloning the PVCs. Its required permissions will be added using the below commands.

```sh
export project=zen
export CloneServiceAccount=clone-service-account

oc create serviceaccount  clone-service-account -n zen
oc policy add-role-to-user system:image-puller system:serviceaccount:zen:clone-service-account -n zen-cpc
oc policy add-role-to-group system:image-puller system:serviceaccounts:$project --namespace zen-cpc
oc adm policy add-scc-to-user privileged -z ${CloneServiceAccount} -n ${project}
```


### 3.7 Download the CPC repo..

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


### 3.8 Push the sincrr image to cluster registry

Make the sincrr image that is used by the cloner and reinstate jobs to be available from the image registry. If the customer don't have private registry, then use the OpenShift internal registry.  In this example we are demontrating using the openshift internal registry.


```sh
# images are pushed under the project zen-cpc, this should be created before pushing
REGISTRY=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
podman tag sincrr:latest ${REGISTRY}/zen-cpc/sincrr:latest
podman login ${REGISTRY} -u $(oc whoami) -p $(oc whoami -t) --tls-verify=false
podman push ${REGISTRY}/zen-cpc/sincrr:latest --tls-verify=false
```

Validate the image is transferred into the cluster registry for use
```
oc get images -n zen-cpc | grep sincrr
sha256:829e98415d05b6b21b520b7e5d16a307416af3146e539d6b90a101bfd4dbe694   image-registry.openshift-image-registry.svc:5000/zen-cpc/sincrr@sha256:829e98415d05b6b21b520b7e5d16a307416af3146e539d6b90a101bfd4dbe694
```

### 3.9 Create pod for mounting the Archive PVC 

This test pod for accessing the content of the cpd-backup-pvc and updating it.  This needs to be created when need and deleted after its use to avoid any interference with data clone operations.

vi mypod.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: cpc-backup-pvc
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/backup"
          name: task-pv-storage
```
oc create -f mypod.yaml

* Check the cloned data are in under the mypod:/backup/zen1-ppc folder
```
oc rsh mypod
cd /backup
ls -lRx
```

please delete the pod after verifying it.
```
oc delete pod mypod
```


## 4.0 Cloning the Cluster
The clonetool runs as a docker/podman container and its behaviour is dependent on the parameters passed as environment variables. You can specify all these on the command line via the -e option, but it is more convenient to update the parameters in an environment file.

Update the cpc_env.sh file in the config directory to match your clone/reinstate decisions, such as:

* The clone repository to use: PVC or S3 and their properties
* OpenShift server address
* Whether to use the OpenShift internal registry for the Cloud Pak images 
* Where to pull the sincrr image from



### 4.1 Capture CPD Operators setup (ibm-common-services)

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

Run the script
```
chmod +x capture-operator-status.sh
./capture-operator-status.sh
```

you should see a file called deployment_replicas.csv that contains all deployments and number of active replicas

<details>
  <summary>Click to view deployment_replicas.csv output !</summary>
  
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
</details>

### 4.2 Capture CPD Service deployments and StatefulSets
Save the deployment and StatefullSets

vi capture-deploy-sts-status.sh 
```
deployjson="cpd-deployments.json"
deploymentscsv="cpd-deployment_replicas.csv"

stsjson="cpd-sts.json"
stscsv="cpd-sts_replicas.csv"

echo "Capture deployments state"
oc get deploy -ojson > "$deployjson"
cat "$deployjson" | jq -r '.items[]|[.metadata.name,.spec.replicas]|@csv' > "$deploymentscsv"

echo "Capture sts state"
oc get sts -ojson > "$stsjson"
cat "$stsjson" | jq -r '.items[]|[.metadata.name,.spec.replicas]|@csv' > "$stscsv"
```

Run the scripts
```
oc project zen
chmod +x capture-deploy-sts-status.sh 
./capture-deploy-sts-status.sh 
```

### 4.3 Scaledown CPD Operators

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

<details>
  <summary>Click to view cpd scale down output !</summary>
  
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
</details>

check all deployments are scaled down
```
oc get deploy
```

wait for all pods to go down. It's fine if cert-manager pods remain with 0/1 running state
```
watch "oc get pods"
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



### 4.4 Update the CPC cpc_env.sh

CPC cloning tool uses the cpd_env.sh to for cloning operations. Based the action, the environment varible file parameter values should be updated. In this example, we will be using the PVC to store the cloned objects.

edit file cpc_env.sh. Change the **`CLONEREPOSITORY` to PVC**  and **`CLONEREPO_PVC` to cpd-backup-pvc** is the name of the PVC we created
```
vi cpc_env.sh
```

```
# CLONEREPOSITORY - This can be either S3 or PVC
CLONEREPOSITORY=PVC
CLONEREPO_PVC=cpd-backup-pvc
```


### 4.5 Clone the Data

This step copies all the NFS based PVC content into cpc-backup-pvc. This might take few minutes to few hours based on the amount of data in these PVCs. You can monitor the progress using the podman command.


```
podman run -d \
   --env-file ./cpc_env.sh \
   -e ACTION=CLONE \
   -e PROJECT=zen \
   -e BACKUP_DIR=zen1-ppc \ 
   -e SERVER=$(oc whoami --show-server) \
   -e TOKEN=$(oc whoami -t) \
   clonetool
```

```
example:
podman run -d    --env-file ./cpc_env.sh    -e ACTION=CLONE  -e BACKUP_DIR=zen1-ppc  -e PROJECT=zen  -e SERVER=$(oc whoami --show-server)    -e TOKEN=$(oc whoami -t)    clonetool
```

Check progress
```
podman ps
podman logs --follow <container_id> 

```

Check status

<details>
<summary>Click to expand cpd clone output log ...</summary>>

```
podman logs --follow 55672d347426
Contents of /tmp/clonedata inside the container: DATA
specs
state
[2022-07-29 07:15:33 INFO] Cloud pak cloner build number: 1. Last updated: 2-May-2021
W0729 07:15:33.532400      24 loader.go:223] Config not found: /tmp/kubeconfig
Logged into "https://api.lax-dsc001.cp.fyre.ibm.com:6443" as "kube:admin" using the token provided.

You have access to 69 projects, the list has been suppressed. You can list all projects with ' projects'

Using project "default".
[2022-07-29 07:15:33 INFO] Wanting to clone
[2022-07-29 07:15:34 INFO] Start Time : 2022.07.29-07.15.33
[2022-07-29 07:15:34 INFO] I need to scale up
[2022-07-29 07:15:34 INFO] I need to scale down
Now using project "zen" on server "https://api.lax-dsc001.cp.fyre.ibm.com:6443".
No resources found in zen namespace.
[2022-07-29 07:15:34 INFO] Starting job to clone metadata to PVC
job.batch/clone-metadata created
[2022-07-29 07:15:34 INFO] Waiting for metadata cloning pod to become active (max 5 min)
Metadata pod ready: False
Metadata pod ready: False
Metadata pod ready: True
[2022-07-29 07:15:50 INFO] Finding pod that will copy the metadata to the designated target on 
Checking clone repository
Create backup directory zen1-ppc if it doesn't exist
Checking that we can write a file to the designated PVC
[2022-07-29 07:15:50 INFO] Saving a list of all container images for this namespace so we can check in the target/reinstate cluster and push missing images proactively
[2022-07-29 07:15:51 INFO] Capture deployments state
[2022-07-29 07:15:51 INFO] Capture statefulsets state
configmap/clone-replicas-information created
[2022-07-29 07:15:52 INFO] Uploading metadata to pod clone-metadata-r959j
sending incremental file list
cpc_clone.log.2022.07.29-07.15.33
run_cpc.log.2022.07.29-07.15.33
DATA/
specs/
state/
state/container_image_list.txt
state/container_image_list_tmp.txt
state/deployment_replicas.csv
state/deployments.json
state/statefulset_replicas.csv
state/statefulsets.json

sent 438,652 bytes  received 188 bytes  877,680.00 bytes/sec
total size is 437,820  speedup is 1.00
[2022-07-29 07:15:52 INFO] Telling pod to sync metadata to clone repository PVC
Starting cloning of metadata
Cloning metadata to the designated PVC
Disabling cronjob diagnostics-cronjob
cronjob.batch/diagnostics-cronjob patched
Disabling cronjob usermgmt-ldap-sync-cron-job
cronjob.batch/usermgmt-ldap-sync-cron-job patched (no change)
Disabling cronjob watchdog-alert-monitoring-cronjob
cronjob.batch/watchdog-alert-monitoring-cronjob patched
Disabling cronjob zen-watchdog-cronjob
cronjob.batch/zen-watchdog-cronjob patched
[2022-07-29 07:15:53 INFO] Scaling down
deployment.apps/ibm-nginx scaled
deployment.apps/usermgmt scaled
deployment.apps/zen-audit scaled
deployment.apps/zen-core scaled
deployment.apps/zen-core-api scaled
deployment.apps/zen-data-sorcerer scaled
deployment.apps/zen-watchdog scaled
deployment.apps/zen-watcher scaled
dsx-influxdb
statefulset.apps/dsx-influxdb scaled
zen-metastoredb
statefulset.apps/zen-metastoredb scaled
[2022-07-29 07:15:54 INFO] Waiting for maximum 60 minutes for pods to scale down
[2022-07-29 07:15:54 PROGRESS] Waiting for all pods to be scaled down. Number of pods still active: 4
[2022-07-29 07:15:55 PROGRESS] List of pods still active: 
[2022-07-29 07:16:05 PROGRESS] Waiting for all pods to be scaled down. Number of pods still active: 3
[2022-07-29 07:16:05 PROGRESS] List of pods still active: 
[2022-07-29 07:16:15 PROGRESS] Waiting for all pods to be scaled down. Number of pods still active: 1
[2022-07-29 07:16:15 PROGRESS] List of pods still active: 
[2022-07-29 07:16:26 PROGRESS] Waiting for all pods to be scaled down. Number of pods still active: 1
[2022-07-29 07:16:26 PROGRESS] List of pods still active: 
[2022-07-29 07:16:36 PROGRESS] Waiting for all pods to be scaled down. Number of pods still active: 1
[2022-07-29 07:16:36 PROGRESS] List of pods still active: 
[2022-07-29 07:16:46 PROGRESS] Waiting for all pods to be scaled down. Number of pods still active: 1
[2022-07-29 07:16:46 PROGRESS] List of pods still active: 
[2022-07-29 07:16:57 PROGRESS] Waiting for all pods to be scaled down. Number of pods still active: 1
[2022-07-29 07:16:57 PROGRESS] List of pods still active: 
[2022-07-29 07:17:07 INFO] Exporting manifests
[2022-07-29 07:17:07 INFO] Exporting project to /tmp/clonedata/specs/0-projects.json
[2022-07-29 07:17:07 INFO] Exporting secrets to /tmp/clonedata/specs/1-secrets.json
[2022-07-29 07:17:08 INFO] Exporting serviceaccount to /tmp/clonedata/specs/2-serviceaccounts.json
[2022-07-29 07:17:08 INFO] Exporting scc to /tmp/clonedata/specs/3-securitycontextconstraints.json
[2022-07-29 07:17:08 INFO] Exporting roles to /tmp/clonedata/specs/4-roles.json
[2022-07-29 07:17:08 INFO] Exporting rolebindings to /tmp/clonedata/specs/5-rolebindings.json
[2022-07-29 07:17:08 INFO] Exporting clusterroles to /tmp/clonedata/specs/6-clusterroles.json
[2022-07-29 07:17:09 INFO] Exporting clusterrolebindings to /tmp/clonedata/specs/6b-clusterrolebindings.json
[2022-07-29 07:17:10 INFO] Exporting configmaps to /tmp/clonedata/specs/7-configmaps.json
[2022-07-29 07:17:10 INFO] Exporting pvc to /tmp/clonedata/specs/9-persistentvolumeclaims.json
[2022-07-29 07:17:10 INFO] Exporting deployments to /tmp/clonedata/specs/10-deployments.json
[2022-07-29 07:17:10 INFO] Exporting statefulsets to /tmp/clonedata/specs/11-statefulsets.json
[2022-07-29 07:17:11 INFO] Exporting services to /tmp/clonedata/specs/12-services.json
[2022-07-29 07:17:11 INFO] Exporting jobs to /tmp/clonedata/specs/13-jobs.json
[2022-07-29 07:17:11 INFO] Exporting cronjobs to /tmp/clonedata/specs/14-cronjobs.json
[2022-07-29 07:17:11 INFO] Exporting routes to /tmp/clonedata/specs/18-routes.json
[2022-07-29 07:17:11 INFO] Exporting storageclass to /tmp/clonedata/specs/storageclass.json
[2022-07-29 07:17:12 INFO] Exporting daemonsets to /tmp/clonedata/specs/19-daemonsets.json
[2022-07-29 07:17:12 INFO] Exporting servicemonitor to /tmp/clonedata/specs/20-servicemonitors.json
[2022-07-29 07:17:12 INFO] Exporting networkpolicy to /tmp/clonedata/specs/21-networkpolicy.json
[2022-07-29 07:17:12 INFO] Exporting deploymentconfig to /tmp/clonedata/specs/22-dc.json
[2022-07-29 07:17:12 INFO] Exporting buildconfig to /tmp/clonedata/specs/23-bc.json
[2022-07-29 07:17:12 INFO] Exporting imagestream to /tmp/clonedata/specs/24-is.json
Not exporting: alertmanagerconfigs.monitoring.coreos.com
Not exporting: alertmanagers.monitoring.coreos.com
Not exporting: apirequestcounts.apiserver.openshift.io
Not exporting: apiservers.config.openshift.io
Not exporting: authentications.config.openshift.io
Not exporting: authentications.operator.openshift.io
Not exporting: baremetalhosts.metal3.io
Not exporting: builds.config.openshift.io
Not exporting: catalogsources.operators.coreos.com
Not exporting: certificaterequests.certmanager.k8s.io
Not exporting: certificates.certmanager.k8s.io
Not exporting: challenges.certmanager.k8s.io
Not exporting: cloudcredentials.operator.openshift.io
Not exporting: clusterautoscalers.autoscaling.openshift.io
Not exporting: clustercsidrivers.operator.openshift.io
Not exporting: clusterissuers.certmanager.k8s.io
Not exporting: clusternetworks.network.openshift.io
Not exporting: clusteroperators.config.openshift.io
Not exporting: clusterresourcequotas.quota.openshift.io
Not exporting: clusterserviceversions.operators.coreos.com
Not exporting: clusterversions.config.openshift.io
Not exporting: configs.imageregistry.operator.openshift.io
Not exporting: configs.operator.openshift.io
Not exporting: configs.samples.operator.openshift.io
Not exporting: consoleclidownloads.console.openshift.io
Not exporting: consoleexternalloglinks.console.openshift.io
Not exporting: consolelinks.console.openshift.io
Not exporting: consolenotifications.console.openshift.io
Not exporting: consoleplugins.console.openshift.io
Not exporting: consolequickstarts.console.openshift.io
Not exporting: consoles.config.openshift.io
Not exporting: consoles.operator.openshift.io
Not exporting: consoleyamlsamples.console.openshift.io
Not exporting: containerruntimeconfigs.machineconfiguration.openshift.io
Not exporting: controllerconfigs.machineconfiguration.openshift.io
Not exporting: credentialsrequests.cloudcredential.openshift.io
Not exporting: csisnapshotcontrollers.operator.openshift.io
Not exporting: dnses.config.openshift.io
Not exporting: dnses.operator.openshift.io
Not exporting: dnsrecords.ingress.operator.openshift.io
Not exporting: egressnetworkpolicies.network.openshift.io
Not exporting: egressrouters.network.operator.openshift.io
Not exporting: etcds.operator.openshift.io
Not exporting: featuregates.config.openshift.io
Not exporting: helmchartrepositories.helm.openshift.io
Not exporting: hostsubnets.network.openshift.io
Not exporting: imagecontentsourcepolicies.operator.openshift.io
Not exporting: imagepruners.imageregistry.operator.openshift.io
Not exporting: images.config.openshift.io
Not exporting: infrastructures.config.openshift.io
Not exporting: ingresscontrollers.operator.openshift.io
Not exporting: ingresses.config.openshift.io
Not exporting: installplans.operators.coreos.com
Not exporting: ippools.whereabouts.cni.cncf.io
Not exporting: issuers.certmanager.k8s.io
Not exporting: kubeapiservers.operator.openshift.io
Not exporting: kubecontrollermanagers.operator.openshift.io
Not exporting: kubeletconfigs.machineconfiguration.openshift.io
Not exporting: kubeschedulers.operator.openshift.io
Not exporting: kubestorageversionmigrators.operator.openshift.io
Not exporting: localvolumediscoveries.local.storage.openshift.io
Not exporting: localvolumediscoveryresults.local.storage.openshift.io
Not exporting: localvolumes.local.storage.openshift.io
Not exporting: localvolumesets.local.storage.openshift.io
Not exporting: machineautoscalers.autoscaling.openshift.io
Not exporting: machineconfigpools.machineconfiguration.openshift.io
Not exporting: machineconfigs.machineconfiguration.openshift.io
Not exporting: machinehealthchecks.machine.openshift.io
Not exporting: machines.machine.openshift.io
Not exporting: machinesets.machine.openshift.io
Not exporting: netnamespaces.network.openshift.io
Not exporting: network-attachment-definitions.k8s.cni.cncf.io
Not exporting: networks.config.openshift.io
Not exporting: networks.operator.openshift.io
Not exporting: oauths.config.openshift.io
Not exporting: ocsinitializations.ocs.openshift.io
Not exporting: openshiftapiservers.operator.openshift.io
Not exporting: openshiftcontrollermanagers.operator.openshift.io
Not exporting: operatorconditions.operators.coreos.com
Not exporting: operatorgroups.operators.coreos.com
Not exporting: operatorhubs.config.openshift.io
Not exporting: operatorpkis.network.operator.openshift.io
Not exporting: operators.operators.coreos.com
Not exporting: orders.certmanager.k8s.io
Not exporting: overlappingrangeipreservations.whereabouts.cni.cncf.io
Not exporting: podmonitors.monitoring.coreos.com
Not exporting: podnetworkconnectivitychecks.controlplane.operator.openshift.io
Not exporting: probes.monitoring.coreos.com
Not exporting: profiles.tuned.openshift.io
Not exporting: projects.config.openshift.io
Not exporting: prometheuses.monitoring.coreos.com
Not exporting: prometheusrules.monitoring.coreos.com
Not exporting: provisionings.metal3.io
Not exporting: proxies.config.openshift.io
Not exporting: rangeallocations.security.internal.openshift.io
Not exporting: rolebindingrestrictions.authorization.openshift.io
Not exporting: schedulers.config.openshift.io
Not exporting: securitycontextconstraints.security.openshift.io
Not exporting: servicecas.operator.openshift.io
Not exporting: servicemonitors.monitoring.coreos.com
Not exporting: storageclusters.ocs.openshift.io
Not exporting: storages.operator.openshift.io
Not exporting: storagestates.migration.k8s.io
Not exporting: storageversionmigrations.migration.k8s.io
Not exporting: subscriptions.operators.coreos.com
Not exporting: thanosrulers.monitoring.coreos.com
Not exporting: tuneds.tuned.openshift.io
Not exporting: volumereplicationclasses.replication.storage.openshift.io
Not exporting: volumereplications.replication.storage.openshift.io
Not exporting: volumesnapshotclasses.snapshot.storage.k8s.io
Not exporting: volumesnapshotcontents.snapshot.storage.k8s.io
Not exporting: volumesnapshots.snapshot.storage.k8s.io
[2022-07-29 07:17:35 INFO] Cloning of PVC cpd-claim ignored as it is in the DO-NOT-CLONE list cpd-claim
[2022-07-29 07:17:35 INFO] Clone job clone-job-data-dsx-influxdb-0-0001 not found, let us run the job
[2022-07-29 07:17:35 INFO] Starting data backup for data-dsx-influxdb-0, job name is clone-job-data-dsx-influxdb-0-0001
job.batch/clone-job-data-dsx-influxdb-0-0001 created
[2022-07-29 07:17:35 INFO] Clone job clone-job-datadir-zen-metastoredb-0-0002 not found, let us run the job
[2022-07-29 07:17:36 INFO] Starting data backup for datadir-zen-metastoredb-0, job name is clone-job-datadir-zen-metastoredb-0-0002
job.batch/clone-job-datadir-zen-metastoredb-0-0002 created
[2022-07-29 07:17:36 INFO] Clone job clone-job-datadir-zen-metastoredb-1-0003 not found, let us run the job
[2022-07-29 07:17:36 INFO] Starting data backup for datadir-zen-metastoredb-1, job name is clone-job-datadir-zen-metastoredb-1-0003
job.batch/clone-job-datadir-zen-metastoredb-1-0003 created
[2022-07-29 07:17:36 INFO] Clone job clone-job-datadir-zen-metastoredb-2-0004 not found, let us run the job
[2022-07-29 07:17:36 INFO] Starting data backup for datadir-zen-metastoredb-2, job name is clone-job-datadir-zen-metastoredb-2-0004
job.batch/clone-job-datadir-zen-metastoredb-2-0004 created
[2022-07-29 07:17:37 INFO] Clone job clone-job-user-home-pvc-0005 not found, let us run the job
[2022-07-29 07:17:37 INFO] Starting data backup for user-home-pvc, job name is clone-job-user-home-pvc-0005
job.batch/clone-job-user-home-pvc-0005 created
[2022-07-29 07:17:37 INFO] Waiting for all 5 clone jobs to complete (max 60 minutes)
[2022-07-29 07:17:48 INFO] Number of clone jobs: 5, 0 active, 5 successful, 0 failed, 0 missing
[2022-07-29 07:17:48 INFO] Scaling up
deployment.apps/ibm-nginx scaled
deployment.apps/usermgmt scaled
deployment.apps/zen-audit scaled
deployment.apps/zen-core scaled
deployment.apps/zen-core-api scaled
deployment.apps/zen-data-sorcerer scaled
deployment.apps/zen-watchdog scaled
deployment.apps/zen-watcher scaled
statefulset.apps/dsx-influxdb scaled
statefulset.apps/zen-metastoredb scaled
Enabling cronjob diagnostics-cronjob
cronjob.batch/diagnostics-cronjob patched
Enabling cronjob usermgmt-ldap-sync-cron-job
cronjob.batch/usermgmt-ldap-sync-cron-job patched
Enabling cronjob watchdog-alert-monitoring-cronjob
cronjob.batch/watchdog-alert-monitoring-cronjob patched
Enabling cronjob zen-watchdog-cronjob
cronjob.batch/zen-watchdog-cronjob patched
Error from server (NotFound): routes.route.openshift.io "router-default" not found
[2022-07-29 07:17:50 INFO] End Time : 2022.07.29-07.17.50
[2022-07-29 07:17:50 INFO] Uploading metadata to pod clone-metadata-r959j
sending incremental file list
cpc_clone.log.2022.07.29-07.15.33
specs/0-projects.json
specs/1-secrets.json
specs/10-deployments.json
specs/11-statefulsets.json
specs/12-services.json
specs/13-jobs.json
specs/14-cronjobs.json
specs/18-routes.json
specs/19-daemonsets.json
specs/2-serviceaccounts.json
specs/20-servicemonitors.json
specs/21-networkpolicy.json
specs/22-dc.json
specs/23-bc.json
specs/24-is.json
specs/3-securitycontextconstraints.json
specs/4-roles.json
specs/5-rolebindings.json
specs/6-clusterroles.json
specs/6b-clusterrolebindings.json
specs/7-configmaps.json
specs/9-persistentvolumeclaims.json
specs/cloneddomain.txt
specs/crd-backingstores.noobaa.io.json
specs/crd-bucketclasses.noobaa.io.json
specs/crd-ccs.ccs.cpd.ibm.com.json
specs/crd-cephblockpools.ceph.rook.io.json
specs/crd-cephclients.ceph.rook.io.json
specs/crd-cephclusters.ceph.rook.io.json
specs/crd-cephfilesystemmirrors.ceph.rook.io.json
specs/crd-cephfilesystems.ceph.rook.io.json
specs/crd-cephnfses.ceph.rook.io.json
specs/crd-cephobjectrealms.ceph.rook.io.json
specs/crd-cephobjectstores.ceph.rook.io.json
specs/crd-cephobjectstoreusers.ceph.rook.io.json
specs/crd-cephobjectzonegroups.ceph.rook.io.json
specs/crd-cephobjectzones.ceph.rook.io.json
specs/crd-cephrbdmirrors.ceph.rook.io.json
specs/crd-certificaterequests.cert-manager.io.json
specs/crd-certificates.cert-manager.io.json
specs/crd-certmanagers.operator.ibm.com.json
specs/crd-challenges.acme.cert-manager.io.json
specs/crd-clusterissuers.cert-manager.io.json
specs/crd-commonservices.operator.ibm.com.json
specs/crd-ibmcpds.cpd.ibm.com.json
specs/crd-issuers.cert-manager.io.json
specs/crd-namespacescopes.operator.ibm.com.json
specs/crd-namespacestores.noobaa.io.json
specs/crd-noobaas.noobaa.io.json
specs/crd-objectbucketclaims.objectbucket.io.json
specs/crd-objectbuckets.objectbucket.io.json
specs/crd-operandbindinfos.operator.ibm.com.json
specs/crd-operandconfigs.operator.ibm.com.json
specs/crd-operandregistries.operator.ibm.com.json
specs/crd-operandrequests.operator.ibm.com.json
specs/crd-orders.acme.cert-manager.io.json
specs/crd-podpresets.operator.ibm.com.json
specs/crd-runtimeassemblies.runtimes.ibm.com.json
specs/crd-secretshares.ibmcpcs.ibm.com.json
specs/crd-ug.wkc.cpd.ibm.com.json
specs/crd-wkc.wkc.cpd.ibm.com.json
specs/crd-zenservices.zen.cpd.ibm.com.json
specs/crdnames.tmp
specs/crds.json
specs/crs-backingstores.noobaa.io.json
specs/crs-bucketclasses.noobaa.io.json
specs/crs-ccs.ccs.cpd.ibm.com.json
specs/crs-cephblockpools.ceph.rook.io.json
specs/crs-cephclients.ceph.rook.io.json
specs/crs-cephclusters.ceph.rook.io.json
specs/crs-cephfilesystemmirrors.ceph.rook.io.json
specs/crs-cephfilesystems.ceph.rook.io.json
specs/crs-cephnfses.ceph.rook.io.json
specs/crs-cephobjectrealms.ceph.rook.io.json
specs/crs-cephobjectstores.ceph.rook.io.json
specs/crs-cephobjectstoreusers.ceph.rook.io.json
specs/crs-cephobjectzonegroups.ceph.rook.io.json
specs/crs-cephobjectzones.ceph.rook.io.json
specs/crs-cephrbdmirrors.ceph.rook.io.json
specs/crs-certificaterequests.cert-manager.io.json
specs/crs-certificates.cert-manager.io.json
specs/crs-certmanagers.operator.ibm.com.json
specs/crs-challenges.acme.cert-manager.io.json
specs/crs-clusterissuers.cert-manager.io.json
specs/crs-commonservices.operator.ibm.com.json
specs/crs-ibmcpds.cpd.ibm.com.json
specs/crs-issuers.cert-manager.io.json
specs/crs-namespacescopes.operator.ibm.com.json
specs/crs-namespacestores.noobaa.io.json
specs/crs-noobaas.noobaa.io.json
specs/crs-objectbucketclaims.objectbucket.io.json
specs/crs-objectbuckets.objectbucket.io.json
specs/crs-operandbindinfos.operator.ibm.com.json
specs/crs-operandconfigs.operator.ibm.com.json
specs/crs-operandregistries.operator.ibm.com.json
specs/crs-operandrequests.operator.ibm.com.json
specs/crs-orders.acme.cert-manager.io.json
specs/crs-podpresets.operator.ibm.com.json
specs/crs-runtimeassemblies.runtimes.ibm.com.json
specs/crs-secretshares.ibmcpcs.ibm.com.json
specs/crs-ug.wkc.cpd.ibm.com.json
specs/crs-wkc.wkc.cpd.ibm.com.json
specs/crs-zenservices.zen.cpd.ibm.com.json
specs/storageclass.json

sent 14,038,855 bytes  received 2,034 bytes  28,081,778.00 bytes/sec
total size is 14,463,912  speedup is 1.03
[2022-07-29 07:17:51 INFO] Telling pod to sync metadata to clone repository PVC
Starting cloning of metadata
Cloning metadata to the designated PVC
[2022-07-29 07:17:51 INFO] Signaling metadata pod that processing is complete and to terminate
Metadata is ready, triggering end of metadata pod
job.batch "clone-metadata" deleted
[2022-07-29 07:17:52 INFO] End Time : 2022.07.29-07.17.52

```
</details>

Now the NFS PVCs data are archived to th cpc-backup-pvc and is now ready for storage class swap.

### 4.6 Validation

create a test pod to mount this cpc-backup-pvc and verify the archive contents.

oc create -f mypod.yaml
(Refere section 3.9)
```
oc rsh mypod
cd /backup/zen1-ppc/DATA
ls -lRx
```

It should list archive files of all the nfs pvcs.


## 5.0 Swap the Storage Class

### 5.1 Change the NFS PV reclaim policy to "Retain"

Change all the NFS PVs reclaim policy to "Retain" to prevent the data deletion till the storage migration is complete. 

```
oc get pv|grep -vi ocs|awk '{print $1}'
```

Update the reclaim policy from Delete to Retain
```
oc patch pv $(oc get pv|grep -vi ocs|awk '{print $1}') -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

Check that all the relevan PVs have changed their reclaim policy.  all NFS based CPD PVs are updated with "Retain" as reclaim policy to prevent the PV from deleting.
<details>
  <summary>Check the status , example</summary>
![status](https://github.ibm.com/CloudPakForDataSWATAssets/InstallPlusPlus/blob/master/backuprestore/pc-retain-status.png)

```sh
# oc get pv 
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM       
STORAGECLASS                  REASON   AGE
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

</details>


### 5.2 Create POD to access Cloned objects

oc get pods | greo mypod

if the pod is not running, please create it using the following command
```
oc project zen
oc apply -f mypod.yaml
(Refere section 3.9)
```

check that the pod was created and that the pod has all the backup files
```
oc get pods | grep mypod
oc debug pod/mypod
cd /backup/zen1-ppc/specs
ls 9-persistentvolumeclaims.json
```


### 5.3 Change the Storage class

Update the storage class name based on the AccessMode, corresponding the OCS storage class should be mapped in the  file "9-persistentvolumeclaims.json" from cpd-clone volume to local client. After the edit, it should be copied back to the cpd-claim PV before running the CPD Reinstate command.



Find the sections with `ReadWriteOnce` and `ReadWriteMany` and replace the NFS storage class with OCS class. For `ReadWriteMany` is `ocs-storagecluster-cephfs` and for `ReadWriteOnce` is `ocs-storagecluster-ceph-rbd`

/backup/zen1-ppc/specs

ocs-storagecluster-ceph-rbd RW Once
ocs-storagecluster-cephfs RWMany

copy the file 9-persistentvolumeclaims.json from  of the debug pod to the cluster to local director for chaning the storage classes to OCS based storage classes.
```
oc cp mypod:/backup/zen1-ppc/specs/9-persistentvolumeclaims.json 9-persistentvolumeclaims.json
cp 9-persistentvolumeclaims.json 9-persistentvolumeclaims.json.backup
```


vi 9-persistentvolumeclaims.json

1. Remove cpc-backup-pvc from 9-persistentvolumeclaims.json
2. change the storageClassName to correct OCS storage classname based on the AccessMode of the PV.


After the update, the following command will list the changes storage class values.

egrep "ReadWriteMany|ReadWriteOnce|storageClassName" 9-persistentvolumeclaims.json 

<details>
  <summary>Output</summary>
  
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
</details>


copy the updated "9-persistentvolumeclaims.json" file back to cpd-claim PVC.
```
oc cp 9-persistentvolumeclaims.json  mypod:/backup/zen1-ppc/specs/9-persistentvolumeclaims.json

Validate the file is copied to the location correctly by log into the test pod..

oc rsh mypod
cd /backup/zen1-ppc/specs
egrep "ReadWriteMany|ReadWriteOnce|storageClassName" 9-persistentvolumeclaims.json 
```
above command should list the OCS storage class names

### 5.4 Scaledown the CPD Pods in cpd namespace(zen)


Scaledown all the CPD pods and clone job pods

####  suspend all cronjobs
Suspend cornjob. Change variable to true suspend: true
```
oc patch cronjobs $(oc get cronjobs --no-headers |awk '{print $1}') -p '{"spec":{"suspend" : true}}'
```
#### suspend cronjobs one by one
```
# Check if there is any cronjob pod active and delete it. And check if any running
oc get cronjobs
oc edit conrjobs env-spec-sync-job
```

#### Scaledown CPD pods
vi scaledown-cpd-services.sh 
```
deploystates="cpd-deployment_replicas.csv"
stsstates="cpd-sts_replicas.csv"

while read d; do
     deployment=$(echo $d | cut -d',' -f 1 | sed 's/"//g')
     scaleto=0
     oc scale deploy $deployment --replicas ${scaleto}
done <${deploystates}

while read d; do
     sts=$(echo $d | cut -d',' -f 1 | sed 's/"//g')
     scaleto=0
     oc scale sts $sts --replicas ${scaleto}
done <${stsstates}
```

```
oc project zen
chmod +x scaledown-cpd-services.sh 
./scaledown-cpd-services.sh 
```

### 5.5 Delete all the completed pods
Any pods running or completed have bound with any of the NFS PVCs will interfere with REINSTATE and also prevent the PVC deletion.
```
oc get pod |grep -i completed
oc delete po $(oc get pod --no-headers |grep -i completed|awk '{print $1}')
```
Check any pods in error state
```
oc get pod |grep -i error
oc delete po $(oc get pod no-headers |grep -i error|awk '{print $1}')
```
Verify all pods are down
```
oc get pods
No resources found in zen namespace.
```

### 5.6 Delete NFS based PVCs



Next delete the NFS PVCs using the folloing command.
```
oc delete -f 9-persistentvolumeclaims.json
```

<details>
  <summary>output</summary>
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
</details>



### 5.7 Recreate OCS Based PVCs

Once all the PVCs with NFS are deleted, Recreate the PVCs using the file with following command. (do not use oc apply)
```
oc create -f  9-persistentvolumeclaims.json 
```

list the newly created OCS PVC
```
oc get pvc
```


### 5.8 Resize OCS PVC volume size
NFS PVCs sizes may not be updated correctly, But in OCS you need to ensure that, correct PVC size reflected updated before reinstate. Patch or edit those PVCs with right sizes.

```
Update the OCS PVCs size, if needed before reinstate
```


## 6.0 Reinstate

cpc reinstate extract the PVC contents from cpd-backup-pv into respective PVs of the services. This will take few minutes to few hours based on the data size.


## 6.1 Upgrade Configuration

make a copy of cpc_env.sh and edit the file. Set the following variables to false `SYNCIMAGES=False` `SETKERNELPARAMS=False` `SETNOROOTSQUASH=False`. 

```
cp cpc_env.sh cpc_env_reinstate.sh
vi cpc_env_reinstate.sh
```

Edit the file "cpc_env_reinstate.sh" and set the following variables to false SYNCIMAGES=False SETKERNELPARAMS=False SETNOROOTSQUASH=False


### 6.2 Reinstate


```
podman run -d \
   --env-file ./cpc_env_reinstate.sh \
   -e ACTION=REINSTATE \
   -e PROJECT=zen \
   -e NEWPROJECT=zen \
   -e BACKUP_DIR=zen1-ppc \
   -e SERVER=$(oc whoami --show-server) \
   -e TOKEN=$(oc whoami -t) \
   clonetool
```


Reinstate cpd services in the Zen namespace with OCS Storage with existing data

Sample command
```
podman run -d    --env-file ./cpc_env_reinstate.sh    -e ACTION=REINSTATE    -e PROJECT=zen    -e NEWPROJECT=zen    -e BACKUP_DIR=zen1-ppc    -e SERVER=$(oc whoami --show-server)    -e TOKEN=$(oc whoami -t)    clonetool
```

Monitor the logs
```
podman ps -a
podman logs --follow eloquent_saha
```


### 6.3 Scale up Services
On successfull restore, all the CPD pods should be back up again and ready to use. CPD Reinstate should bring up the services, if not use the following to Scaleup command to bring up cpd Services.

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

```
oc project zen
chmod +x scaleup-cpd-services.sh
./scaleup-cpd-services.sh 

# check status by
oc get deploy,sts
```


## 7.0 Validate the cluster
[This section should contain the steps to validate the cluster after the storage swap]

Log into the cluster and verify all the connection and projects details are intact and able to access them.  This complete the migration of storage from NFS to OCS.

## 8.0 Cleanup
[This section should contain the cleanup after the storage migration]

### 8.1 Delete old PVs
oc get pv |grep "Released"

<details>
  <summary>Results</summary>
pvc-0200dcc9-0e4d-465d-8a7b-e514fd29aea3   10Gi       RWO            Retain           Released   zen/data-redis-ha-server-0                                    managed-nfs-storage                    2d19h
pvc-165235c1-b0de-482d-8abf-f79a71ba58d4   10Gi       RWO            Retain           Released   zen/datadir-zen-metastoredb-2                                 managed-nfs-storage                    2d19h
pvc-3608e4e3-a183-4aa0-b6b8-61adb54b5091   30Gi       RWO            Retain           Released   zen/database-storage-wdp-couchdb-2                            managed-nfs-storage                    2d19h
pvc-378eb21e-08d2-4301-8e1b-74e142cc36ab   10Gi       RWO            Retain           Released   zen/data-redis-ha-server-1                                    managed-nfs-storage                    2d19h
pvc-38ba7486-68b4-4645-94bb-cb24fe27491d   10Gi       RWO            Retain           Released   zen/datadir-zen-metastoredb-1                                 managed-nfs-storage                    2d19h
pvc-3b16b913-9e77-4e92-b71a-f2f67b45d038   30Gi       RWO            Retain           Released   zen/elasticsearch-master-elasticsearch-master-1               managed-nfs-storage                    2d19h
pvc-44b65609-b422-4be7-8aee-00f40565341c   20Gi       RWO            Retain           Released   zen/0072-iis-dedicatedservices-pvc                            managed-nfs-storage                    2d19h
pvc-4f212a66-064f-4ed1-8289-d846bdc5f232   10Gi       RWO            Retain           Released   zen/datadir-zen-metastoredb-0                                 managed-nfs-storage                    2d19h
pvc-6479b3db-28fe-4760-bca3-a28ee16823c5   50Gi       RWX            Retain           Released   zen/iis-db2u-backups                                          managed-nfs-storage                    2d19h
pvc-6e64caa2-ecc5-4ce3-9a20-5b892fd76d49   100Mi      RWX            Retain           Released   zen/0072-iis-sampledata-pvc                                   managed-nfs-storage                    2d19h
pvc-707a12d1-5692-4cda-9a01-e0764d75fec2   10Gi       RWO            Retain           Released   zen/data-rabbitmq-ha-1                                        managed-nfs-storage                    2d19h
pvc-7363ed3d-6daf-4842-8876-71b60af9f1cd   10Gi       RWO            Retain           Released   zen/data-rabbitmq-ha-2                                        managed-nfs-storage                    2d19h
pvc-80befefe-f911-4077-bb7d-ba443c1882dc   30Gi       RWX            Retain           Released   zen/elasticsearch-master-backups                              managed-nfs-storage                    2d19h
pvc-83bbe8a2-7ac7-4530-aaae-0294d8dd9865   40Gi       RWX            Retain           Released   zen/c-db2oltp-wkc-data                                        managed-nfs-storage                    2d19h
pvc-846943df-279b-4b8f-bcb1-8b272be8d4ef   30Gi       RWO            Retain           Released   zen/database-storage-wdp-couchdb-1                            managed-nfs-storage                    2d19h
pvc-8ded8a86-4075-4a38-9993-ffd320158647   30Gi       RWO            Retain           Released   zen/elasticsearch-master-elasticsearch-master-0               managed-nfs-storage                    2d19h
pvc-8fe445da-3b8e-4dd8-8882-2b98cf22927f   40Gi       RWX            Retain           Released   zen/0072-iis-en-dedicated-pvc                                 managed-nfs-storage                    2d19h
pvc-a1d3d4f6-ee50-4426-ba2f-7148e2330530   5Gi        RWO            Retain           Released   zen/zookeeper-data-zookeeper-0                                managed-nfs-storage                    2d19h
pvc-abe65f55-5228-4f66-8620-420a6ace9412   10Gi       RWO            Retain           Released   zen/data-dsx-influxdb-0                                       managed-nfs-storage                    2d19h
pvc-ac7d3af3-34a1-497f-9cb8-917a733c601d   50Gi       RWX            Retain           Released   zen/cc-home-pvc                                               managed-nfs-storage                    2d19h
pvc-b509b7b8-2be7-40bf-af72-77cc4d9fb06b   20Gi       RWX            Retain           Released   zen/c-db2oltp-iis-meta                                        managed-nfs-storage                    2d19h
pvc-b7f9480e-2d68-4d62-80bb-13a4edf1d2e3   10Gi       RWO            Retain           Released   zen/data-rabbitmq-ha-0                                        managed-nfs-storage                    2d19h
pvc-b7fb6502-c3a3-4b8f-a501-7027494e8758   90Gi       RWO            Retain           Released   zen/cassandra-data-cassandra-0                                managed-nfs-storage                    2d19h
pvc-bc5ce6ed-05f8-4751-8314-1265ce523264   40Gi       RWX            Retain           Released   zen/wkc-db2u-backups                                          managed-nfs-storage                    2d19h
pvc-c1fb1e90-7802-4209-be66-5c3c519c2a81   40Gi       RWX            Retain           Released   zen/c-db2oltp-iis-data                                        managed-nfs-storage                    2d19h
pvc-c81fcbcf-bf39-4eb6-8243-0ec58f861a9e   30Gi       RWO            Retain           Released   zen/elasticsearch-master-elasticsearch-master-2               managed-nfs-storage                    2d19h
pvc-d76428c2-155d-4c23-8e06-c85b4fbab4d8   5Gi        RWX            Retain           Released   zen/0073-ug-omag-pvc                                          managed-nfs-storage                    2d19h
pvc-e1f2e7e3-7cd3-47a6-b110-2f9c92e924f1   100Gi      RWO            Retain           Released   zen/kafka-data-kafka-0                                        managed-nfs-storage                    2d19h
pvc-ed44ccde-fb8d-44c3-82da-57381b956f2b   20Gi       RWX            Retain           Released   zen/c-db2oltp-wkc-meta                                        managed-nfs-storage                    2d19h
pvc-f147c06f-f663-4b32-8724-808166947fda   10Gi       RWX            Retain           Released   zen/user-home-pvc                                             managed-nfs-storage                    2d19h
pvc-f3683fbd-c8e8-4d0b-a508-c63c7e62f53e   30Gi       RWO            Retain           Released   zen/database-storage-wdp-couchdb-0                            managed-nfs-storage                    2d19h
pvc-f460ef07-0f66-4532-ba6e-378ffcfcc09e   100Gi      RWX            Retain           Released   zen/file-api-claim                                            managed-nfs-storage                    2d19h
pvc-f7632474-fea6-4f99-8848-687a07116858   30Gi       RWO            Retain           Released   zen/solr-data-solr-0                                          managed-nfs-storage                    2d19h
pvc-fad87bb4-06b8-4875-920e-bd33553b79a9   10Gi       RWO            Retain           Released   zen/data-redis-ha-server-2                                    managed-nfs-storage                    2d19h
```
</details>

### 8.2 Delete all clone/reinstate jobs

oc get jobs -l clonejob=reinstate
oc delete jobs $(oc get jobs -l clonejob=reinstate --no-headers|awk '{ print $1 }')

oc get jobs -l clonejob=clone
oc delete jobs $(oc get jobs -l clonejob=clone --no-headers|awk '{ print $1 }')

oc get pods | egrep 'clone|reinstate'
oc delete pods $(oc get pods | egrep 'clone|reinstate' |awk '{ print $1 }')


### 8.3 Delete mypod
oc delete pod mypod

### 8.4 Delete cpc-backup-pvc

oc delete pvc cpc-backup-pvc
oc delete pv $(oc get pv | grep cpc-backup-pvc|awk '{ print $1 }')


## 9.0 References

* [Cloner Tool](https://github.ibm.com/CloudPakCloner/clonetool)

