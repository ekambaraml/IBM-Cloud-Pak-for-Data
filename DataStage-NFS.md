# Passing Data Files to DataStage

In legacy DataStage environment, it is a common practice to use NFS mount to pass data files such as CSV and XML between DataStage and source/target systems.

Since DataStage in Cloud Pak for Data is container based thus immutable in nature, you cannot just get in to the DataStage pod and mount NFS volume unless the pod is started as a privilege mode which is not the case for us.

You have two options. One is to use Persistent Volume that is created using the target NFS, and the other one is to mount the specific target NFS volume in the pod.

Using Persistent Volume would be the easiest. However, when you create a PVC, OpenShift creates a sub directory in the directory you specified for PV to avoid conflict between PVC's. For example:

```bash
/SHARE/zen-0072-iis-en-dedicated-pvc-pvc-824ad47d-2979-4437-adc1-7cdc1fae80db
```

In this example, '`/SHARE`' is the mount path you would specify in the PV specification, and the rest is appended by OpenShift. If the external system allows changing the shared directory to this directory, this may be a good option. You should create a dedicated PV for the staging space instead of using the shared storage CP4D uses as default. Be aware if you re-create the PV, the sub directory path will change.

This note will not go over the steps to setup the PV using NFS as there are many articles available on the internet.

From here on, this note will walk you through the steps to setup NFS mount in the DataStage pods without using PV. The steps are based on OpenShift 4.3.8 on bare metal with CoreOS. Depending on the cloud provider and the OpenShift version, procedure may be different.

***CAUTION**: This procedure involves changing security context constraint for the DataStage pod service account which may cause a security concern. Consult with DataStage offering manager before you proceed.*

## Mounting NFS Volume on Worker Nodes

We will mount the NFS volume on the worker nodes and then mount it on the pod using **hostPath**. It may sound odd but this was the only way I was able to mount the specific NFS export path on the pod without granting privilege mode to the pod. The volume declaration in the pod spec like below is supposed to work but it did not. It may be due to the software issue with OpenShift. The future version may work.

```bash
spec:
  volumes:
    - name: nfs-volume
      nfs:
        server: 10.108.211.244
        path: /share
```

First, we will mount the NFS volume on the worker nodes. In CoreOS, you use a mount file instead of running **mount** command or setting up `/etc/fstab` file.

We will mount the NFS volume at `/var/mnt/nfs` in our example. The mount file has a naming rule, that is, it requires the directory names concatenated with dash followed by "`.mount`" at the end.

```bash
<dir>-<dir>.mount
```

In our case, we need to concatenate "var", "mnt" and "nfs" with dash and the file needs to be created in `/etc/systemd/system`. So, the full path will be:

```bash
/etc/systemd/system/var-mnt-nfs.mount
```

Log in to the worker node and declare the mount details in the mount file. An example is shown below.

```bash
[Unit]
Description = Mount NFS share

[Mount]
What=10.176.115.194:/share
Where=/var/mnt/nfs
Type=nfs

[Install]
WantedBy = multi-user.target
```

If not already created the `/var/mnt/nfs` directory, create it now. Then runt the **systemctl** command to enable the mount.

```bash
systemctl enable --now /etc/systemd/system/var-mnt-nfs.mount
```

## Allowing Access from Pod to Remote NFS

By default, SELinux does not allow writing from a pod to a remote NFS server. Run this command to allow write access by pods.

```bash
setsebool -P virt_use_nfs on
```

Repeat these steps on all worker nodes.

## Adding Security Context Constraint

The DataStage pods run with the service account **wkc-iis-sa**. This service account does not have a permission to mount local directories. Local hosts are protected. You need to add the **hostmount-anyuid** SCC to this service account. This command needs to be performed by the cluster administrator.

```bash
oc adm policy add-scc-to-user hostmount-anyuid system:serviceaccount:zen:wkc-iis-sa
```

## Updating StatefulSet Specification

Use Open OpenShift Web Console or the **oc** command to update the DataStage StatefulSet. You need to update two of them, **is-en-conductor** and **ds-engine-compute**. If you use Web Console, search those StatefulSets. If you use the **oc** command, run the command below to edit the StatefulSet.

```bash
oc edit sts <is-en-conductor or ds-engine-compute>
```

Find the **volumeMounts** element and declare the mount location. The mount path and the volume name can be anything.

```bash
volumeMounts:
  - mountPath: /mnt/dedicated_vol/Engine
    name: engine-dedicated-volume
  - mountPath: /opt/ia/custom
    name: engine-dedicated-volume
    subPath: is-en-conductor-0/ia/custom
  - mountPath: /share      ## <-- here
    name: nfs-data-share   ## <-- here
```

Find the **volumes** element and add the volume declaration. See the element starting with "`- name: nfs-data-share`"

```bash
volumes:
  - name: engine-dedicated-volume
    persistentVolumeClaim:
      claimName: 0072-iis-en-dedicated-pvc
  - name: nfs-data-share    ## <- here and below
    hostPath:
      path: /var/mnt/nfs
      type: Directory
```

Next, find the **securityContext** element. You might find it more than one. Add **supplementalGroups** to the one that is outermost. This allows the file access with the group ID's specified for this element. This **supplementalGroups** keyword is meant to allow the group ID other than what the pod run-time user has. If the run-time user's group has enough permission to access the NFS volume, it should not require this element but I needed to use this otherwise got permission error. You may not need this. Try and see the pod events.

```bash
terminationGracePeriodSeconds: 30
securityContext:
  supplementalGroups: [1000]  ## <-- here
  fsGroup: 1000
```

## Restarting Pods

Once StatefulSet is updated, OpenShift automatically restarts the pods. Check the age of the pods to make sure pods are all restarted.

Now, you are ready to go.
