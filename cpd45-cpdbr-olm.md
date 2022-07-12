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
The Cloud Pak for Data OADP backup and restore utility uses the following components:
* [ ] OADP/Velero and its default plug-ins
* [ ] Custom Velero plug-in cpdbr-velero-plugin
* [ ] cpd-cli oadp command-line interface. This CLI is part of the cpd-cli utility.



```
./oc version
Client Version: 4.8.35
Server Version: 4.8.43
Kubernetes Version: v1.21.11+6b3cbdd
```

```
jq --version
  wget -O jq https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux64
  chmod +x ./jq
  cp jq /usr/bin
```


#### Download cpd-cli commandline utility
```
wget https://github.com/IBM/cpd-cli/releases/download/v11.0.0/cpd-cli-linux-EE-11.0.0.tgz
tar xvf cpd-cli-linux-EE-11.0.0.tgz
cd cpd-cli-linux-EE-11.0.0-20
cpd-cli
	Version: 11.0
	Build Date: 2022-06-08T17:28:23
	Build Number: 20
	CPD Release Version: 4.5.0
  
```

#### Install cpdbr-velero-plugin

```
oc get route -n openshift-image-registry

IMAGE_REGISTRY=`oc get route -n openshift-image-registry | grep image-registry | awk '{print $2}'`
echo $IMAGE_REGISTRY
NAMESPACE=oadp-operator
echo $NAMESPACE
CPU_ARCH=`uname -m`
echo $CPU_ARCH
BUILD_NUM=1
echo $BUILD_NUM


# Pull cpdbr-velero-plugin image from IBM Cloud Container Registry
podman pull icr.io/cpopen/cpd/cpdbr-velero-plugin:4.0.0-beta1-${BUILD_NUM}-${CPU_ARCH}
# Push image to internal registry
podman login -u kubeadmin -p $(oc whoami -t) $IMAGE_REGISTRY --tls-verify=false
podman tag icr.io/cpopen/cpd/cpdbr-velero-plugin:4.0.0-beta1-${BUILD_NUM}-${CPU_ARCH} $IMAGE_REGISTRY/$NAMESPACE/cpdbr-velero-plugin:4.0.0-beta1-${BUILD_NUM}-${CPU_ARCH}
podman push $IMAGE_REGISTRY/$NAMESPACE/cpdbr-velero-plugin:4.0.0-beta1-${BUILD_NUM}-${CPU_ARCH} --tls-verify=false
```

#### Install OADp 1.0 on cluster with internet access
```
oc annotate namespace oadp-operator openshift.io/node-selector="" --overwrite

vi credentials-velero

[default]
aws_access_key_id = minio
aws_secret_access_key = minio123


oc create secret generic cloud-credentials \
--namespace oadp-operator \
--from-file cloud=./credentials-velero
```

```
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: dpa-sample
spec:
  configuration:
    velero:
      customPlugins:
      - image: image-registry.openshift-image-registry.svc:5000/oadp-operator/cpdbr-velero-plugin:4.0.0-beta1-1-x86_64
        name: cpdbr-velero-plugin
      defaultPlugins:
      - aws
      - openshift
      - csi
      podConfig:
        resourceAllocations:
          limits:
            cpu: "1"
            memory: 1Gi
          requests:
            cpu: 500m
            memory: 256Mi
    restic:
      enable: true
      timeout: 2h
      podConfig:
        resourceAllocations:
          limits:
            cpu: "1"
            memory: 4Gi
          requests:
            cpu: 500m
            memory: 256Mi
  backupImages: false
  backupLocations:
    - velero:
        provider: aws
        default: true
        objectStorage:
          bucket: velero
          prefix: cpdbackup
        config:
          region: minio
          s3ForcePathStyle: "true"
          s3Url: http://minio-velero.apps.mycluster.cp.com
        credential:
          name: cloud-credentials
          key: cloud
```

```
oc get po -n oadp-operator
```


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


