
## 4.0 Global pull secret and Image Content Source policy
The following tasks should be performed by the OpenShift cluster admin.
##### Time: 30mins

***
#### 4.1 Global pull secret setup
***


This deployment is using the private registry. So the pull secret should be configured the "PRIVATE REGISTRY". Save the following script to a file named "util-add-pull-secret.sh". 
```
### Create file util-add-pull-secret.sh
# usage" util-add-pull-secret.sh <repository> <user> <password>

# cat util-add-pull-secret.sh
#!/bin/bash

if [ "$#" -lt 3 ]; then
  echo "Usage: $0 <repo-url> <artifactory-user> <API-key>" >&2
  exit 1
fi

# set -x

REPO_URL=$1
REPO_USER=$2
REPO_API_KEY=$3

pull_secret=$(echo -n "$REPO_USER:$REPO_API_KEY" | base64 -w0)
oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | sed -e 's|:{|:{"'$REPO_URL'":{"auth":"'$pull_secret'","email":"not-used"\},|' > /tmp/dockerconfig.json
oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=/tmp/dockerconfig.json
```

***
#### 4.3 Verify the pullsecrets
***
```
oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq

Output (example):

 "auths": {
    "ai.ibmcloudpack.com": {
      "auth": "YWRtaW46YWRtaW5wYXNz",
      "email": "not-used"
    },
```


***
#### 4.4 ImageContentSourcePolicy Setup
***
Note: This will cause the nodes to restart
#### Requires Cluster Admin permission

```
cat <<EOF |oc apply -f -
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: cloud-pak-for-data-mirror
spec:
  repositoryDigestMirrors:
  - mirrors:
    - ${PRIVATE_REGISTRY}
    - ${PRIVATE_REGISTRY}/ibmcom
    source: docker.io/ibmcom
  - mirrors:
    - ${PRIVATE_REGISTRY}
    source: quay.io/opencloudio
  - mirrors:
    - ${PRIVATE_REGISTRY}
    - ${PRIVATE_REGISTRY}/cp
    - ${PRIVATE_REGISTRY}/cp/cpd
    source: cp.icr.io/cp/cpd
  - mirrors:
    - ${PRIVATE_REGISTRY}
    - ${PRIVATE_REGISTRY}/cp
    source: cp.icr.io/cp
  - mirrors:
    - ${PRIVATE_REGISTRY}
    - ${PRIVATE_REGISTRY}/cpopen
    source: icr.io/cpopen
EOF

oc get imageContentSourcePolicy
oc get imageContentSourcePolicy  cloud-pak-for-data-mirror -o yaml
```

Cluster nodes will restart, Wait for all the nodes are ready

```
watch "oc get nodes"

Every 2.0s: oc get nodes                                                                                                                              cxbastion81.fyre.ibm.com: Sat Sep 11 09:02:18 2021

NAME                                  STATUS                     ROLES    AGE   VERSION
master0.cx-norfolks.cp.fyre.ibm.com   Ready                      master   20h   v1.19.0+d670f74
master1.cx-norfolks.cp.fyre.ibm.com   Ready                      master   20h   v1.19.0+d670f74
master2.cx-norfolks.cp.fyre.ibm.com   Ready                      master   20h   v1.19.0+d670f74
worker0.cx-norfolks.cp.fyre.ibm.com   Ready                      worker   20h   v1.19.0+d670f74
worker1.cx-norfolks.cp.fyre.ibm.com   Ready,SchedulingDisabled   worker   20h   v1.19.0+d670f74
worker2.cx-norfolks.cp.fyre.ibm.com   Ready                      worker   20h   v1.19.0+d670f74
```

  
