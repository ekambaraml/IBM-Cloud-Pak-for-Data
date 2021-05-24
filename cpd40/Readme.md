# Cloud Pak for Data 4.0 



#### 1.0 Getting CPD Entitlements


#### 2.0 Downloading

- Download Installers


  - Download cloudctl
    ```
    wget https://github.com/IBM/cloud-pak-cli/releases/download/v3.7.1/cloudctl-linux-amd64.tar.gz
    tar xvf cloudctl-linux-amd64.tar.gz
    chmod +x cloudctl-linux-amd64
    cp cloudctl-linux-amd64 /usr/local/bin/cloudctl
    ```

  - Download openshift client
    ```
    wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.6.20/openshift-client-linux-4.6.20.tar.gz
    tar xvf openshift-client-linux-4.6.20.tar.gz
    cp oc /usr/bin
    # oc version
    Client Version: 4.6.20
    Server Version: 4.6.20

    cp kubectl /usr/bin
    ```


- Download Cloud Pak for Data 4.0 RC1

  - CPD Lite
    ```
    wget https://ibm-open-platform.ibm.com/repos/cpd/v4.0/rc1/case/ibm-zen/1.1.0-279/ibm-zen-1.1.0-279.tgz
    ```
  - Common Core Services
    ```
    wget https://ibm-open-platform.ibm.com/repos/cpd/v4.0/rc1/case/ibm-ccs/1.0.0-637/ibm-ccs-1.0.0-637.tgz
    ```
  - DB2aaservice
    ```
    wget https://ibm-open-platform.ibm.com/repos/cpd/v4.0/rc1/case/ibm-db2aaservice/4.0.0-1151.344/ibm-db2aaservice-4.0.0-1151.344.tgz
    ```
  - DB2U Operator
    ```
    wget https://ibm-open-platform.ibm.com/repos/cpd/v4.0/rc1/case/ibm-db2uoperator/4.0.0-3506-2130/ibm-db2uoperator-4.0.0-3506-2130.tgz
    ```
  - IIS
    ```
    wget https://ibm-open-platform.ibm.com/repos/cpd/v4.0/rc1/case/ibm-iis/4.0.0-268/ibm-iis-4.0.0-268.tgz
    ```
  - WKC
    ```
    wget https://ibm-open-platform.ibm.com/repos/cpd/v4.0/rc1/case/ibm-wkc/4.0.0-311/ibm-wkc-4.0.0-311.tgz
    ```

#### 3.0 Registry Mirrors

<b> Check the pullsecrets has been created</b>
```
oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | grep us.icr.io

pull_secret=$(echo -n "iamapikey:<Entitlement Key>" | base64 -w0)

oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | sed -e 's|:{|:{"us.icr.io/4_0_rc1":{"auth":"'$pull_secret'"\},|' > /tmp/dockerconfig.json
   
oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=/tmp/dockerconfig.json
```
   
```
  
  
  
  
#### 4.0 Cluster Pre-Requisites

#### 5.0 Installing control plane

#### 6.0 Installing WKC

