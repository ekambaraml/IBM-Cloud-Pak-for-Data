# Deploying OpenShift, Cloud Pak for Data on Azure Cloud



## Resource Requirement

### Credentials

Account | Description
------- | ------------
Aad Client ID | Azure active directort id
Aad Client Secret | <secret>
  

### Networking
  
Resource | Count | Description
----------------| ------| -----------
Virtuak Network | 1 | Virtual Private Network
Master Subnet | 1 | [10.0.1.0/24]
Worker Subnet | 1 | [10.0.2.0/24]
Bastion Subnet| 1 | [10.0.3.0/24]
Azure Zone | Single/Multi | Single Zone or Multi Zone deployments
Network Security Group | 3 | Bastion, Master, Worker
IP | 10 | Number of IP Addresses
Storage Account | 1 | NFS Storage

### Machine

Resource | count | Description
---------|-------|------------
 Master  | 3     | RHCOS Nodes
 Worker  | 3+    | RHCOS Nodes
 Bastion | 1     | RHEL
 NFS     | 1 TB  | Azure Disk
 

### DNS Zone
  
