## Deploying Cloud Pak for Data on AWS Gov Cloud

Topics

- Deploying OpenShift 4.6 on AWS gov cloud
- Deploying Cloud Pak for Data 3.5

### Deploying OpenShift 4.6
https://docs.openshift.com/container-platform/4.6/installing/installing_aws/installing-aws-government-region.html

#### Requirements

* AWS Government Region
```
        us-gov-west-1
        us-gov-east-1
```
* AWS account access
```
         * access_key_id = "xxxxxxxxxxxxxxxxxxxxxxx"
         * secret_access_key = "xxxxxxxxxxxxxxxxxxxxxxx"
```
#### Private cluster
 
To create a private cluster on Amazon Web Services (AWS), you must provide an existing private VPC and subnets to host the cluster. The installation program must also be able to resolve the DNS records that the cluster requires. The installation program configures the Ingress Operator and API server for access from only the private network.

- Public subnets
- Public load balancers, which support public ingress
- A public Route 53 zone that matches the baseDomain for the cluster
        

#### Setup Bastion Host

Make sure this machine has internet access
* 8core/32GB/120GB. 
* If you want to use this as NFS server, add additional 1TB disk



## Deploying Cloud Pak for Data

###  2.1 Prepare the OpenShift cluster for CPD
*   Registry
*   Storage Classes
*   Node Settings
*   Download the Cloud Pak for Data Installer
*   Get the CPD entitlement API key


### Installing Cloud Pak for Data and Services


* Installing CPD Control Plane (lite)

        
        ./cpd-cli adm --assembly lite -n zen --arch x86_64 -r ./repo.yaml --accept-all-licenses --apply


        Install assembly dry-run
        ./cpd-cli install \
        --repo ./repo.yaml \
        --assembly lite \
        --namespace zen \
        --storageclass aws-efs \
        --transfer-image-to $(oc registry info)/zen \
        --cluster-pull-prefix $( oc registry info --internal)/zen \
        --target-registry-username kubeadmin \
        --target-registry-password $(oc whoami -t) \
        --insecure-skip-tls-verify \
        --latest-dependency \
        --dry-run


        Installing assembly
        ./cpd-cli install \
        --repo ./repo.yaml \
        --assembly lite \
        --namespace zen \
        --storageclass aws-efs \
        --transfer-image-to $(oc registry info)/zen \
        --cluster-pull-prefix $( oc registry info --internal)/zen \
        --target-registry-username kubeadmin \
        --target-registry-password $(oc whoami -t) \
        --insecure-skip-tls-verify \
        --latest-dependency 


* Installing Watson Studio (wsl)

        ./cpd-cli adm --assembly wsl -n zen --arch x86_64 -r ./repo.yaml --accept-all-licenses --apply


        ./cpd-cli install \
        --repo ./repo.yaml \
        --assembly wsl \
        --namespace zen \
        --storageclass aws-efs \
        --transfer-image-to $(oc registry info)/zen \
        --cluster-pull-prefix $( oc registry info --internal)/zen \
        --target-registry-username kubeadmin \
        --target-registry-password $(oc whoami -t) \
        --insecure-skip-tls-verify \
        --latest-dependency \
        --dry-run


        ./cpd-cli install \
        --repo ./repo.yaml \
        --assembly wsl \
        --namespace zen \
        --storageclass aws-efs \
        --transfer-image-to $(oc registry info)/zen \
        --cluster-pull-prefix $( oc registry info --internal)/zen \
        --target-registry-username kubeadmin \
        --target-registry-password $(oc whoami -t) \
        --insecure-skip-tls-verify \
        --latest-dependency 

   ![image](https://user-images.githubusercontent.com/26153008/113579340-c8941a80-95e9-11eb-9366-fbf81f467f47.png)
   
   On Successfull Deployment, users should be able to access the CPD Web Console
   ![image](https://user-images.githubusercontent.com/26153008/113579721-4f48f780-95ea-11eb-914e-e4eeb388263b.png)


* Other CPD services & assembly names




### I

