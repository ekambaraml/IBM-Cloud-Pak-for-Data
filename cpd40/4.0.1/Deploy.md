
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
