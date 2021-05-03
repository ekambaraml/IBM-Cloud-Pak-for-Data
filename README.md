# IBM-Cloud-Pak-for-Data
IBM Cloud Pak for Data and its tutorials


# 1. OpenShift 4.x

## 2. Requirements
   ### 2.1  Azure permissions
      - IPI Requirements
      
          # az account list
          [
          {
            "cloudName": "AzureCloud",
            "id": "<subscription>", 
            "isDefault": false,
            "name": "Pay-As-You-Go",
            "state": "Enabled",
            "tenantId": "<tenant-id>",
            "user": {
              "name": "admin@example.com",
              "type": "user"
            }
          ]
  
      
      - UPO Requirements
      
      
## 3. Infrastructure
    A resource group
    An Azure virtual network
    One or more network security groups that contain the required OpenShift Container Platform ports
    A storage account
    A service principal
    Two load balancers
    Two or more DNS entries for the routers and for the OpenShift Container Platform web console
    Three Availability Sets
    Three master instances
    Three infrastructure instances
    One or more application instances

