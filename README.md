# Overview

<br>

* This document describes the installation of SyncTree.
* The services used for installation are:
  * EKS(Elastic Kubernetes Service) and Managed Node Group
  * Elasticache for Redis
  * Amazon Aurora
  * EC2 for bastion server
* Configure EKS on a new cluster(Recomended)
* Approximately 20 minutes to deploy infrastructure with CloudFormation.
* It is recommended that you start with the specifications below when you first start.
  * EKS Worker Nodes : t3.medium (Minimum 50 GB disk)
  * Amazon Aurora : db.t3.medium
  * Elasticache for Redis : cache.t3.micro
  * EC2 for bastion server : t3.micro
  * Bastion server image must be Amazon Linux 2 AMI (HVM)
  * (The install scripts was written based on Amazon Linux 2 AMI)
* You can proceed with cloudformation by referring to the attached sample parameter values.

<br>

> Note.
> <br>Depending on the traffic, the specification may need to be changed.
> <br>Therefore, it is recommended to start with a minimum specification.

<br>

# Architectural Diagram

<br>

Three Availability Zone(AZ)
![architectrual_diagram](https://user-images.githubusercontent.com/103020388/190327215-1285374e-08e1-42dc-87db-6725e8ad19e9.png)

<br>

Two Availability Zone(AZ)
![architectural_diagram_2az](https://user-images.githubusercontent.com/103020388/205843996-fdf14021-a769-4a79-a6c9-915d6ebb383b.png)

<br>

# Caution

<br>

The tasks described in this document should only be performed during the installation phase and should never be performed in the production environment

<br>


# Prerequisites

<br>

* A Key Pair must be configured in the region you plan to deploy your EKS Cluster.

<br>

# Create CloudFormation Stack

<br>

Move to CloudFormation page and press Create stack button.
![create_stack](https://user-images.githubusercontent.com/103020388/190582150-a23f674a-62aa-4af2-a4ff-02f131f1ca3b.png)


<br>

Option. 1) upload provided template.

<br>

Enter parameter values for installation. If you are not sure, please refer to the link below.
<br>
(https://github.com/ntuple-synctree/synctree/blob/main/CloudFormation_Sample_Parameter.txt)
![enter_value](https://user-images.githubusercontent.com/103020388/190582350-92713d5a-046f-469f-a563-a80b8bb759a8.png)

<br>

Press next button at the bottom.
![stack_next](https://user-images.githubusercontent.com/103020388/190582420-774abc29-4af5-4664-8f24-cb65717909ee.png)

<br>

Press next button.
![advanced_option](https://user-images.githubusercontent.com/103020388/190586172-2103f5aa-d790-425c-8b59-669ec481599a.png)

<br>

Check the check box and press Create stack button.
![create_stack_2](https://user-images.githubusercontent.com/103020388/190582496-11ece7db-9e51-47fb-b59d-da751b3a1d18.png)

<br>

The installation takes about 20 to 30 minutes.
![start_stack](https://user-images.githubusercontent.com/103020388/190585458-2744b452-815f-4e85-bc78-3546dcd88f40.png)

<br>

# Connect Bastion Server For Installation

<br>

> Note.
> <br>Set the inbound setting of the bastion server security group for SSH connection.

<br>

* Connect to bastion server for SyncTree installation, After the cloudformation task is finished.
* Check End mark in task.log file like below:
```
tail -f task.log
```
Example output:
```
install cmctl completed
install kubectl completed
install eksctl completed
install mysql-client completed
install redis-cli completed
create env.sh completed
create cmd.txt completed
End
```

<br>

# Check Tools for Installation

<br>

Make sure you have these tools installed to install SyncTree solutions. These tools are installed automatically during cloud information installation. The list includes:

* redis-cli
* mysql-client
* jq
* eksctl
* kubectl
* cmctl
* docker

<br>

Check redis-cli installation.
```
redis-cli --version
```
Example output:
```
redis-cli 6.2.6
```

<br>

Check mysql-client installation.

```
mysql --version
```
Example outpu:
```
mysql  Ver 15.1 Distrib 5.5.68-MariaDB, for Linux (x86_64) using readline 5.1
```

<br>

Check jq installation.

```
jq --version
```
Example output:
```
jq-1.5
```

<br>

Check eksctl installation.
```
eksctl version
```
Example output:
```
0.92.0
```

<br>

Check kubectl installation.

```
kubectl version
```
Example output:
  * Note. You can ignore the error message at this point.
```
Client Version: version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.5", GitCommit:"c285e781331a3785a7f436042c65c5641ce8a9e9", GitTreeState:"clean", BuildDate:"2022-03-16T15:58:47Z", GoVersion:"go1.17.8", Compiler:"gc", Platform:"linux/amd64"}
```

<br>

Check cmctl installation.

```
cmctl --help
```
Example output:
```
cmctl is a CLI tool manage and configure cert-manager resources for Kubernetes

Usage: cmctl [command]

Available Commands:
  approve      Approve a CertificateRequest
  check        Check cert-manager components
  completion   Generate completion scripts for the cert-manager CLI
```

<br>

Check docker installation.
```
sudo systemctl start docker
sudo systemctl status docker
```
Example output:
```
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-05-01 05:18:14 UTC; 17s ago
     Docs: https://docs.docker.com
  Process: 8449 ExecStartPre=/usr/libexec/docker/docker-setup-runtimes.sh (code=exited, status=0/SUCCESS)
  Process: 8439 ExecStartPre=/bin/mkdir -p /run/docker (code=exited, status=0/SUCCESS)
 Main PID: 8454 (dockerd)

```

<br>
  
# Configuration and Credential File Settings




Check the settings with the command below

```
aws sts get-caller-identity
```
Example output:
```
{
    "Account": "Your Account ID",
    "UserId": "Your UserID",
    "Arn": "arn:aws:iam::Your Account ID:user/username"
}
```

<br>

This is the output if it fails.
```
Unable to locate credentials. You can configure credentials by running "aws configure".
```

<br>

Run the command below to create and save environment variables to a file for future use.

```
export AWS_ACCOUNTID=`aws sts get-caller-identity | jq '.Account' | sed 's/\"//g'`
sed -i "/AWS_ACCOUNTID/d" ~/env.sh
echo "export AWS_ACCOUNTID=$AWS_ACCOUNTID" >> ~/env.sh

TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

AWS_REGION=$(curl -s -H "X-aws-ec2-metadata-token:$TOKEN" http://169.254.169.254/latest/dynamic/instance-identity/document | jq -r .region)

export AWS_REGION
sed -i "/AWS_REGION/d" ~/env.sh
echo "export AWS_REGION=$AWS_REGION" >> ~/env.sh
```

<br>

Make sure it's saved to a env.sh file.

```
cat ~/env.sh | grep -e "AWS_ACCOUNTID" -e "AWS_REGION"
```
Example output:
```
export AWS_ACCOUNTID=026134571450
export AWS_REGION=ap-northeast-2
```

<br>

# Set Environment Variables

<br>

Run the command below to export the variables required for the operation. Various endpoint, port, and user information are stored in env.sh and are automatically created when Bastion EC2 is created. In order to proceed with the next steps, make sure to run and then proceed with the next step.

<br>

Run the command below.
```
source ~/env.sh
```

<br>

# Check AWS Infrastructure Status

<br>

Check that the aws infrastructure required to install SyncTree solution is installed successfully. The list includes:

<br>

* Elasticache
* Aurora Database
* Kubernetes Worker Nodes

<br>

## Check Elasticache

<br>

If the PONG value is output by running the command below, proceed to the next step.

```
redis-cli -h $ELASTICACHE_ENDPOINT -p $ELASTICACHE_PORT -c ping
```
Example output:
```
PONG
```

<br>

## Check Aurora Database

<br>

If version information is printed by running the two commands below, proceed to the next step.

<br>

> Note. 
> <br>SyncTree uses two Aurora Databases for data and log information.

<br>

Check if the version information is output by running the command below.
<br>

```
mysql -h $AURORA_DATA_ENDPOINT -u $AURORA_DATA_USERNAME -p$AURORA_DATA_PASSWORD -e "select version()"
```
Example output:
```
+-----------+
| version() |
+-----------+
| 5.7.12    |
+-----------+
```

<br>

Check if the version information is output by running the command below.
<br>
```
mysql -h $AURORA_LOG_ENDPOINT -u $AURORA_LOG_USERNAME -p$AURORA_LOG_PASSWORD -e "select version()"
```
Example output:
```
+-----------+
| version() |
+-----------+
| 5.7.12    |
+-----------+
```

<br>

## Check Kubernetes Worker Nodes

<br>

Run the command below to download the kubeconfig file.
<br>

```
aws eks update-kubeconfig --region=$AWS_REGION --name=$EKS_CLUSTER_NAME
```
Example output:
```
Added new context arn:aws:eks:ap-northeast-2:052908120200:cluster/eks-synctree to /home/ec2-user/.kube/config
```

<br>

Run the command below to check that all STATUS in the node are ready.
```
kubectl get nodes
```
Example output:
```
NAME                                            STATUS   ROLES    AGE   VERSION
ip-10-0-1-41.ap-northeast-2.compute.internal    Ready    <none>   94m   v1.21.5-eks-9017834
ip-10-0-1-70.ap-northeast-2.compute.internal    Ready    <none>   93m   v1.21.5-eks-9017834
ip-10-0-5-151.ap-northeast-2.compute.internal   Ready    <none>   94m   v1.21.5-eks-9017834
```

<br>

# Download the Installation Script

<br>

Run the command below to download the script required for the installation.

```
git clone https://github.com/synctreeTech/install-synctree.git
```


<br>


# Create Database Schema

<br>

Move to cloned directory

```
cd install-synctree
```

Set execute permission
<br>
```
chmod +x ./set-studio-password.sh
```

<br>

Set account information to log in to SyncTree Studio. Set password within ' ' as in Example.
```
./set-studio-password.sh your-user-name your-user-email your-user-password
```
Example:
```
./set-studio-password.sh root root@test.com 'root1234!'
```


<br>

Check that user information is saved by running the command below.
```
cat ./sql/03.DataDB_Account_Data_Insert.sql | grep -e "SET @name" -e "SET @email" -e "SET @passphrase"
```
Example output:
```
SET @name = 'root';
SET @email = 'root@test.com';
SET @passphrase = 'MyPassword1!';
```

<br>

Set environment variables for future operations. Copy and run the command below.

```
unset STUDIO_USER_PASSWORD
source ~/env.sh
```

<br>

Check environment variable settings.

```
env | grep -e "STUDIO_USER_PASSWORD"
```
Example output:
```
STUDIO_USER_PASSWORD=root1234!
```

<br>

Copy and run the command below for Log Database setup.
```
cp ./templates/04.DataDB_Shard_Info_Data_Insert.sql ./sql/
sed -i "s/YOUR-AURORA-LOG-ENDPOINT/$AURORA_LOG_ENDPOINT/g" ./sql/04.DataDB_Shard_Info_Data_Insert.sql
sed -i "s/YOUR-AURORA-LOG-PORT/$AURORA_LOG_PORT/g" ./sql/04.DataDB_Shard_Info_Data_Insert.sql
```

<br>

Check the dns address and port of the two @connection_string sections below.
```
cat ./sql/04.DataDB_Shard_Info_Data_Insert.sql
```
Example output:
```
-- log db ip and port
SET @connection_string = 'synctree-log.cluster-cpaezbezw5u4.ap-northeast-2.rds.amazonaws.com,3306';
... (omission) ...

USE synctree_portal;

-- log db ip and port
SET @connection_string = 'synctree-log.cluster-cpaezbezw5u4.ap-northeast-2.rds.amazonaws.com,3306';
... (omission) ...
```

<br>

Check to see if it matches the AURORA_LOG_ENDPOINT, AURORA_LOG_PORT information on env.sh.
```
cat ~/env.sh | grep -e "AURORA_LOG_ENDPOINT" -e "AURORA_LOG_PORT"
```
Example output:
```
... (omission) ...
export AURORA_LOG_ENDPOINT=synctree-log.cluster-cpaezbezw5u4.ap-northeast-2.rds.amazonaws.com
export AURORA_LOG_PORT=3306
... (omission) ...
```
<br>

> Note.
> <br>Refer to image below.  
![ref_shard_info](https://user-images.githubusercontent.com/103020388/163495271-045cd201-ace3-4c20-a87e-99b9ed403c3c.png)

<br>

Make sure the current directory is the install-synctree location.
```
pwd
```
Example output:
```
/home/ec2-user/install-synctree
```

<br>

Run the command below to create a schema for the database for storing synctree data.
```
mysql -h $AURORA_DATA_ENDPOINT -u $AURORA_DATA_USERNAME -p$AURORA_DATA_PASSWORD < ./sql/00.DataDB_User_Create.sql
mysql -h $AURORA_DATA_ENDPOINT -u admin -p$ADMIN_PASSWORD < ./sql/01.DataDB_synctree_script.sql
mysql -h $AURORA_DATA_ENDPOINT -u admin -p$ADMIN_PASSWORD < ./sql/02.DataDB_Default_Insert.sql
mysql -h $AURORA_DATA_ENDPOINT -u admin -p$ADMIN_PASSWORD < ./sql/03.DataDB_Account_Data_Insert.sql
mysql -h $AURORA_DATA_ENDPOINT -u admin -p$ADMIN_PASSWORD < ./sql/04.DataDB_Shard_Info_Data_Insert.sql
```

<br>

Check the database creation by running the command below.

```
mysql -h $AURORA_DATA_ENDPOINT -u admin -p$ADMIN_PASSWORD -e "show databases;"
```

<br>

As shown below, check that a total of 6 databases have been created: synctree_agent, synctree_auth, synctree_marketplace, synctree_plan, synctree_portal, synctree_studio.

```
+----------------------+
| Database             |
+----------------------+
| information_schema   |
| mysql                |
| performance_schema   |
| synctree_agent       |
| synctree_auth        |
| synctree_marketplace |
| synctree_plan        |
| synctree_portal      |
| synctree_studio      |
| sys                  |
+----------------------+
```

<br>

Run the command below to create a schema for the database for storing synctree log.
```
mysql -h $AURORA_LOG_ENDPOINT -u $AURORA_LOG_USERNAME -p$AURORA_LOG_PASSWORD < ./sql/00_1.LogDB_User_Create.sql
mysql -h $AURORA_LOG_ENDPOINT -u $AURORA_LOG_USERNAME -p$AURORA_LOG_PASSWORD < ./sql/01_1.LogDB_synctree_script.sql
mysql -h $AURORA_LOG_ENDPOINT -u $AURORA_LOG_USERNAME -p$AURORA_LOG_PASSWORD < ./sql/02_1.LogDB_Default_Insert.sql  
```

<br>

Check the database creation by running the command below.

```
mysql -h $AURORA_LOG_ENDPOINT -u $AURORA_LOG_USERNAME -p$AURORA_LOG_PASSWORD -e "show databases;"
```

<br>

Check that the synctree_studio_logdb database has been created as shown below.

```
+-----------------------+
| Database              |
+-----------------------+
| information_schema    |
| mysql                 |
| performance_schema    |
| synctree_studio_logdb |
| sys                   |
+-----------------------+
```

<br>

Set execute permission
<br>
```
chmod +x ./get-portal-domain-key.sh
```

<br>

Extract the PORTAL_DOMAIN_KEY value by running the command below.
```
./get-portal-domain-key.sh
```
Example output:
```
PORTAL DOMAIN KEY : 960E644741A1
```

<br>

Run the command below to set environment variables.
```
source ~/env.sh
```

<br>

Run the command below to check whether the value is extracted as shown in the example output.

```
env | grep PORTAL_DOMAIN_KEY
```
Example output:
```
PORTAL_DOMAIN_KEY=9603D64DEF21
```
<br>

# Create an IAM OIDC Provider for Your Cluster

<br>

Your cluster has an OpenID Connect issuer URL associated with it. To use IAM roles for service accounts, an IAM OIDC provider must exist for your cluster.

<br>

Determine whether you have an existing IAM OIDC provider for your cluster.

```
aws eks describe-cluster --name $EKS_CLUSTER_NAME --query "cluster.identity.oidc.issuer" --output text
```
Example output:
```
https://oidc.eks.region-code.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE
```

<br>

List the IAM OIDC providers in your account. Replace EXAMPLED539D4633E53DE1B71EXAMPLE with the value returned from the previous command.
```
aws iam list-open-id-connect-providers | grep EXAMPLED539D4633E53DE1B71EXAMPLE
```
Example output:
```
"Arn": "arn:aws:iam::111122223333:oidc-provider/oidc.eks.region-code.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE"
```

If output is returned from the previous command, then you already have a provider for your cluster. If no output is returned, then you must create an IAM OIDC provider.

<br>

Create an IAM OIDC identity provider for your cluster with the following command.
```
eksctl utils associate-iam-oidc-provider --cluster $EKS_CLUSTER_NAME --approve
```
Example output:
```
2022-04-09 03:48:05 [ℹ]  eksctl version 0.92.0
2022-04-09 03:48:05 [ℹ]  using region ap-northeast-2
2022-04-09 03:48:06 [ℹ]  will create IAM Open ID Connect provider for cluster "eks-synctree" in "ap-northeast-2"
2022-04-09 03:48:07 [✔]  created IAM Open ID Connect provider for cluster "eks-synctree" in "ap-northeast-2"
```

<br>

# Create IAM Policy for AWS Load Balancer Controller

<br>

Create an IAM Policy to install the AWS Load Balancer Controller.

<br>

Go to the installation script folder.
```
cd /home/ec2-user/install-synctree
```

<br>

Run the command below to create a policy.
```
aws iam create-policy \
--policy-name AWSLoadBalancerControllerIAMPolicy \
--policy-document file://./json/alb_iam_policy.json
```
Example output:
```
{
    "Policy": {
        "PolicyName": "AWSLoadBalancerControllerIAMPolicy",
        "PermissionsBoundaryUsageCount": 0,
        "CreateDate": "2022-04-09T03:54:20Z",
        "AttachmentCount": 0,
        "IsAttachable": true,
        "PolicyId": "ANPAQYUMR3CEKSYRT7DVK",
        "DefaultVersionId": "v1",
        "Path": "/",
        "Arn": "arn:aws:iam::052908120200:policy/AWSLoadBalancerControllerIAMPolicy",
        "UpdateDate": "2022-04-09T03:54:20Z"
    }
}
```

<br>

# Create IAM Role for AWS Load Balancer Controller

<br>

Create an IAM Role to install the AWS Load Balancer Controller.

<br>

Go to the installation script folder.
```
cd /home/ec2-user/install-synctree
```

<br>

Run the command below to create an IAM Role.

```
eksctl create iamserviceaccount \
--cluster=$EKS_CLUSTER_NAME \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::$AWS_ACCOUNTID:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--approve
```

Example output:
```
2022-04-09 06:25:36 [ℹ]  eksctl version 0.92.0
2022-04-09 06:25:36 [ℹ]  using region ap-northeast-2
2022-04-09 06:25:37 [ℹ]  1 iamserviceaccount (kube-system/aws-load-balancer-controller) was included (based on the include/exclude rules)
2022-04-09 06:25:37 [!]  metadata of serviceaccounts that exist in Kubernetes will be updated, as --override-existing-serviceaccounts was set
2022-04-09 06:25:37 [ℹ]  1 task: {
    2 sequential sub-tasks: {
        create IAM role for serviceaccount "kube-system/aws-load-balancer-controller",
        create serviceaccount "kube-system/aws-load-balancer-controller",
    } }2022-04-09 06:25:37 [ℹ]  building iamserviceaccount stack "eksctl-eks-synctree-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2022-04-09 06:25:37 [ℹ]  deploying stack "eksctl-eks-synctree-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2022-04-09 06:25:37 [ℹ]  waiting for CloudFormation stack "eksctl-eks-synctree-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2022-04-09 06:25:56 [ℹ]  waiting for CloudFormation stack "eksctl-eks-synctree-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2022-04-09 06:26:15 [ℹ]  waiting for CloudFormation stack "eksctl-eks-synctree-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2022-04-09 06:26:15 [ℹ]  created serviceaccount "kube-system/aws-load-balancer-controller"
```

> Note.
> <br>If the operation fails and you try again, you can delete the stack from cloudformation and have to redo it. Refer to image below.
> <br>
> 
> ![cloudformation_eksctl_stack](https://user-images.githubusercontent.com/103020388/163666593-dcda800c-f546-47a3-8f5c-3fa14c1ab2f4.png)

<br>

# Install Cert-Manager

<br>

Install cert-manager to install AWS Load Balancer Controller.

<br>

Go to the installation folder.
```
cd /home/ec2-user/install-synctree/
```

<br>

Install cert-manager.
```
kubectl apply -f ./yaml/cert-manager.yaml
```

<br>

Run the command below to check that the three pods are running, such as Example output.
```
kubectl get pods -n cert-manager
```
Example output:
```
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-7c6f78c46d-g6s7z              1/1     Running   0          27s
cert-manager-cainjector-668d9c86df-j7tsw   1/1     Running   0          27s
cert-manager-webhook-764b556954-jm4nk      1/1     Running   0          27s

```

<br>

Run the command below to check if the cert-manager api status is ready.
```
cmctl check api
```
This is the output if the cert-manager api is ready.
```
The cert-manager API is ready
```
This is the output if the cert-manager api is not ready.
```
Not ready: Internal error occurred: failed calling webhook "webhook.cert-manager.io": Post "https://cert-manager-webhook.cert-manager.svc:443/mutate?timeout=10s": no endpoints available for service "cert-manager-webhook"
...
Not ready: the cert-manager webhook CA bundle is not injected yet
```

<br>

Run the command below to update the CRD.
```
kubectl apply -k ./yaml/
```
Example output:
```
customresourcedefinition.apiextensions.k8s.io/ingressclassparams.elbv2.k8s.aws created
customresourcedefinition.apiextensions.k8s.io/targetgroupbindings.elbv2.k8s.aws created
```
<br>

Run the command below and check whether ingressclassparams.elbv2.k8s.aws and targetgroupbindings.elbv2.k8s.aws are created as in the example output.
```
kubectl get crd
```
Example output:
```
NAME                                         CREATED AT
certificaterequests.cert-manager.io          2022-04-09T06:49:04Z
certificates.cert-manager.io                 2022-04-09T06:49:04Z
challenges.acme.cert-manager.io              2022-04-09T06:49:04Z
clusterissuers.cert-manager.io               2022-04-09T06:49:05Z
eniconfigs.crd.k8s.amazonaws.com             2022-04-08T21:11:56Z
ingressclassparams.elbv2.k8s.aws             2022-04-09T06:52:20Z
issuers.cert-manager.io                      2022-04-09T06:49:06Z
orders.acme.cert-manager.io                  2022-04-09T06:49:06Z
securitygrouppolicies.vpcresources.k8s.aws   2022-04-08T21:12:00Z
targetgroupbindings.elbv2.k8s.aws            2022-04-09T06:52:20Z
```
<br>

# Install AWS Load Balancer Controller

<br>

Refer to link below for installation.
<br>
https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html

<br>

Go to installation folder and set environment variables.
```
cd /home/ec2-user/install-synctree/
source ~/env.sh
```

<br>

Create additional policies.
```
aws iam create-policy \
--policy-name AWSLoadBalancerControllerAdditionalIAMPolicy \
--policy-document file://./json/alb_iam_policy_v1_to_v2_additional.json
```
Example output:
```
{
    "Policy": {
        "PolicyName": "AWSLoadBalancerControllerAdditionalIAMPolicy",
        "PermissionsBoundaryUsageCount": 0,
        "CreateDate": "2022-04-09T06:01:18Z",
        "AttachmentCount": 0,
        "IsAttachable": true,
        "PolicyId": "ANPAQYUMR3CEIS66OH6AQ",
        "DefaultVersionId": "v1",
        "Path": "/",
        "Arn": "arn:aws:iam::your-accountid:policy/AWSLoadBalancerControllerAdditionalIAMPolicy",
        "UpdateDate": "2022-04-09T06:01:18Z"
    }
}

```

<br>

Run the command below to add a policy to the IAM Role.

```
output=`aws iam list-roles | grep "eksctl-$EKS_CLUSTER_NAME-addon-iamservice" | grep $AWS_ACCOUNTID | sed 's/\"//g' | sed 's/,//g'`
eks_iam_service_role_arn=${output#*role/}

sed -i "/EKS_IAM_SERVICE_ROLE_ARN/d" ~/env.sh
echo "export EKS_IAM_SERVICE_ROLE_ARN=$eks_iam_service_role_arn" >> ~/env.sh

aws iam attach-role-policy \
--role-name $eks_iam_service_role_arn \
--policy-arn arn:aws:iam::$AWS_ACCOUNTID:policy/AWSLoadBalancerControllerAdditionalIAMPolicy

source ~/env.sh
```

<br>

Run the command below and check if two policies are output in PolicyName: 
<br>
* AWSLoadBalancer ControllerAdditionalIAMPolicy
* AWSLoadBalancer ControllerIAMPolicy.
```
aws iam list-attached-role-policies --role-name $eks_iam_service_role_arn
```
Example output:
```
{
    "AttachedPolicies": [
        {
            "PolicyName": "AWSLoadBalancerControllerAdditionalIAMPolicy",
            "PolicyArn": "arn:aws:iam::052908120200:policy/AWSLoadBalancerControllerAdditionalIAMPolicy"
        },
        {
            "PolicyName": "AWSLoadBalancerControllerIAMPolicy",
            "PolicyArn": "arn:aws:iam::052908120200:policy/AWSLoadBalancerControllerIAMPolicy"
        }
    ]
}
```

<br>

> Note.  
<br>The --role-name above can be seen in the CloudFormation stack, click on the part starting with eksctl below. When executing the above command, it is automatically extracted and executed, so please refer to this part only.
![cloudformation_eksctl1](https://user-images.githubusercontent.com/103020388/163307745-a1e6c0a2-a083-44e2-936f-28e48bf8eb3d.png)  
<br>Use the Physical ID in the Resources tab
![cloudformation_eksctl2](https://user-images.githubusercontent.com/103020388/163296374-426a0bdd-6a79-44be-b95c-d2c0b738dbe6.png)


<br>


Copy yaml file and rename cluster by running command below.
```
cp ./templates/v2_4_0_full.yaml ./yaml/
sed -Ei "s|your-cluster-name|$EKS_CLUSTER_NAME|" ./yaml/v2_4_0_full.yaml
```

<br>

Check if --cluster-name is set by running the command below.
```
cat ./yaml/v2_4_0_full.yaml | grep cluster-name
```
Example output:
```
        - --cluster-name=eks-synctree
```

<br>

Install AWS Load Balancer Controller.
```
kubectl apply -f ./yaml/v2_4_0_full.yaml
```


<br>

Run the command below to check whether the aws-load-balancer-controller pods are running.
```
kubectl get pods -n kube-system | grep aws-load-balancer
```
Example output:
```
aws-load-balancer-controller-5f7cb8b897-m8zrh   1/1     Running   0          67s
aws-load-balancer-controller-5f7cb8b897-vg2dq   1/1     Running   0          67s
aws-load-balancer-controller-5f7cb8b897-wbtz8   1/1     Running   0          9s
```

<br>

# Installing SyncTree Pods

<br>

Run the command below to create secret.
```
kubectl create secret docker-registry private-repo --docker-server=repo.synctreestudio.com --docker-username=<provided-account> --docker-password=<provided-password> --namespace=synctree
```
Example:
```
kubectl create secret docker-registry private-repo --docker-server=repo.synctreestudio.com --docker-username=ntp-mtr --docker-password=<provided-password> --namespace=synctree
```

<br>

Go to installation folder and set environment variable.
```
cd /home/ec2-user/install-synctree/
source ~/env.sh
```

<br>

Run the command below to update the settings required for SyncTree installation.

```
cp ./templates/credentials ./synctree/deploy/
sed -i "s/YOUR-ELASTICACHE-ENDPOINT/$ELASTICACHE_ENDPOINT/" ./synctree/deploy/credentials
sed -i "s/YOUR-ELASTICACHE-PORT/$ELASTICACHE_PORT/" ./synctree/deploy/credentials
sed -i "s/YOUR-AURORA-DATA-ENDPOINT/$AURORA_DATA_ENDPOINT/" ./synctree/deploy/credentials
sed -i "s/YOUR-AURORA-DATA-PORT/$AURORA_DATA_PORT/" ./synctree/deploy/credentials
sed -i "s/YOUR-ADMIN-PASSWORD/$ADMIN_PASSWORD/" ./synctree/deploy/credentials
sed -i "s/YOUR-PORTAL-DOMAIN-KEY/$PORTAL_DOMAIN_KEY/g" ./synctree/deploy/credentials

cp ./templates/php.ini ./synctree/deploy/
sed -i "s/YOUR-ELASTICACHE_ENDPOINT/$ELASTICACHE_ENDPOINT/" ./synctree/deploy/php.ini
sed -i "s/YOUR-ELASTICACHE_PORT/$ELASTICACHE_PORT/" ./synctree/deploy/php.ini

echo "AWS_ACCOUNTID=`aws sts get-caller-identity | jq '.Account' | sed 's/\"//g'`" >> ./synctree/deploy/aws.txt
```
<br>

Run the command below to install SyncTree pods.

```
cd /home/ec2-user/install-synctree/synctree/
chmod +x ./deploy-synctree.sh
./deploy-synctree.sh

```

<br>

Run the command below to check if STATUS is Running.
```
kubectl get pods -n synctree
```
Example output:
```
NAME                                    READY   STATUS    RESTARTS   AGE
synctree-portal-67b5d5cfd6-fhl8j        1/1     Running   0          41h
synctree-studio-7bf86c4569-xv5b2        1/1     Running   0          41h
synctree-testing-5cb4887677-sbkwt       1/1     Running   0          41h
synctree-tool-6456744d4c-8lnxc          1/1     Running   0          41h
synctree-tool-proxy-5b98d8b74-m6g46     1/1     Running   0          41h
```

<br>

# Create Ingress

<br>

Create ingress.
```
cd /home/ec2-user/install-synctree/synctree/alb
chmod +x ./deploy-ingress.sh
./deploy-ingress.sh
```
Example output:
```
ingress.networking.k8s.io/synctree-ingress created
```

<br>

Run the command below to check whether synctree-ingress has been created.
```
kubectl get ingress -n synctree
```
Example output:
```
NAME               CLASS    HOSTS                                                                             ADDRESS                                                                        PORTS   AGE
synctree-ingress   <none>   studio.your-domain.com,testing.your-domain.com,tool.your-domain.com + 3 more...   k8s-synctree-synctree-929256406b-1043780048.ap-northeast-2.elb.amazonaws.com   80      42m
```

<br>

A load balancer is automatically created when creating an ingress.

<br>

# Check Load Balancer Creation

Go to the EC2 > Load Balancer page to check the Load Balancer creation.

* Check if State is Active
* Copy DNS name

![load_balancer](https://user-images.githubusercontent.com/103020388/163297801-c40be846-5264-48b6-aa71-9a1df9d6f9f3.png)

<br>

# Connect SyncTree Studio

<br>

Search the DNS Name of the Load Balancer with the nslookup command to check the IP address.

```
$ nslookup
> k8s-synctree-synctree-929256406b-766064353.ap-northeast-2.elb.amazonaws.com
Server:         10.0.0.2
Address:        10.0.0.2#53

Non-authoritative answer:
Name:   k8s-synctree-synctree-929256406b-766064353.ap-northeast-2.elb.amazonaws.com
Address: 54.180.39.83
Name:   k8s-synctree-synctree-929256406b-766064353.ap-northeast-2.elb.amazonaws.com
Address: 15.165.65.214
Name:   k8s-synctree-synctree-929256406b-766064353.ap-northeast-2.elb.amazonaws.com
Address: 52.78.171.161

```
<br>

Select any one of the three IPs and set the window host file as follows.

```
54.180.39.83	studio.your-domain.com
54.180.39.83	tool.your-domain.com
54.180.39.83	api.your-domain.com
54.180.39.83	portal.your-domain.com
```

<br>

Check SyncTree Studio connection.

* http://studio.your-domain.com 
* Enter your SyncTree Studio account information, you can check it with the command below.
    ```
    cat ~/env.sh | grep -e "STUDIO_USER_EMAIL" -e "STUDIO_USER_PASSWORD"
    ```

![synctree_studio_http1](https://user-images.githubusercontent.com/103020388/163300584-b7277f11-5cfb-4b38-a9a3-7d11c4717076.png)

<br>

SyncTree Studio connection screen

![synctree_studio_http2](https://user-images.githubusercontent.com/103020388/163300891-1c78034e-6327-411c-8ee0-1bd26d9cd43a.png)

<br>

# What to Do After You've Completed Up to Here

<br>

Up to now, HTTP has been used for installation verification. For HTTPS, the following must be done:

<br>

* Register a Domain via AWS Route 53.
* Creating a public hosted zone.
* Generate a certificate via AWS Certificate Manager.
* Set certificate arn and domain name in SyncTree ingress.yaml.
* Configure Amazon Route 53 as your DNS service.

<br>

# Register a New Domain

<br>

The domain must be registered to use HTTPS. Register your domain via AWS Route53.

<br>

Refer to link below.
<br>
https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/domain-register.html

<br>

# Working with Hosted Zones

<br>

If you have registered a domain, you must create a Hosted zone in Route 53.

<br>

Refer to link below.
<br>
https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-configuring.html

<br>

Refer to image below.

![route53_hosted](https://user-images.githubusercontent.com/103020388/163301796-de26fd14-84b9-4ffe-acb5-e04a93df3db4.png)

<br>

# Generate Certificate

<br>

You must obtain a certificate to use HTTPS.

<br>

Generate Certificates with AWS Certificate Manager. Refer to link below.
<br>
https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html

<br>

Refer to image below.
<br>

![certificate_manager](https://user-images.githubusercontent.com/103020388/163666198-d624b751-dfdf-47bc-a8ce-f13ed433a8d2.png)

<br>

# Modify ingress.yaml

<br>

You have to modify and deploy ingress.yaml to use HTTPS.

<br>

Go to installation folder.
```
cd /home/ec2-user/install-synctree
```

<br>

Set execute permission
<br>
```
chmod +x ./set-cert-domain.sh
```
<br>

Run set-cert-domain.sh.
* Pass Certificate ARN and Domain Name as argument
* Set the Certificate ARN to the value you checked during the certificate generation step
```
./set-cert-domain.sh YOUR-CERTIFICATE-ARN YOUR-DOMAIN-NAME
```
Example
```
./set-cert-domain.sh arn:aws:acm:ap-northeast-2:052908120200:certificate/709966d8-7cb8-4d30-919d-e643e9b979f0 mydomain.com
```

<br>

> Note.  
Refer to the image below for the Certificate ARN value.
![certificate_manager](https://user-images.githubusercontent.com/103020388/163302283-80b73518-25b0-4c55-9a93-27b7b2979dc3.png)

<br>

Check the Certificate ARN and Domain Name settings.
```
cat ./synctree/alb/ingress.yaml
```

* Check your alb.ingress.kubernetes.io/certificate-arn
* The studio, tool, api, and portal parts are fixed, and only the mydomain.com part is changed to your domain.
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: "synctree-ingress"
  annotations:
    ... (omission) ...
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-northeast-2:013817514782:certificate/2e7d0g9b-134b-4253-c3e4-57f0b2e5615f
spec:
  rules:
  - host: studio.mydomain.com
    ... (omission) ...
  - host: tool.mydomain.com
    ... (omission) ...
  - host: api.mydomain.com
    ... (omission) ...
  - host: portal.mydomain.com
    ... (omission) ...
```

<br>

Check the domain name settings in synctree credentials
```
cat ./synctree/deploy/credentials
```

* Make sure that the domain name of a,b,c,d in endpoint section and the domain name of portal section are changed.
* synctree-testing and synctree-auth must be used as fixed values.
```
[end-point]
synctree-studio="https://studio.mp.synctreestudio.com"
synctree-tool="https://tool.mp.synctreestudio.com/plan/entrance"
synctree-tool-proxy="https://api.mp.synctreestudio.com{base-path}"
synctree-portal="https://portal.mp.synctreestudio.com"
synctree-portal-admin="https://portal-admin.mp.synctreestudio.com"
synctree-testing="http://synctree-testing.synctree.svc/plan/entrance"
synctree-auth="http://synctree-tool.synctree.svc/auth"

[portal]
domain_key["portal.mp.synctreestudio.com"]="97E02FD29B81"
domain_key["portal-admin.mp.synctreestudio.com"]="97E02FD29B81"

```

<br>

Apply ingress.yaml
```
cd /home/ec2-user/install-synctree/synctree/alb
./deploy-ingress.sh
```
Example output:
```
ingress.networking.k8s.io/synctree-ingress configured
```

<br>

Check HTTPS 443 rules of Load Balancer.

![alb_rule1](https://user-images.githubusercontent.com/103020388/192662907-b1b8bc5c-303e-4cc3-8610-b5ceb6670652.png)


<br>

Make sure it is set to the domain name you want to use after studio, tool, api and portal.
![alb_rule2](https://user-images.githubusercontent.com/103020388/163304236-02fb4bf9-df98-4709-bdda-ff00fb573410.png)

<br>

Restart SyncTree pods.

```
cd /home/ec2-user/install-synctree/synctree
chmod +x ./delete-synctree.sh
./delete-synctree.sh
./deploy-synctree.sh

```

<br>

Run the command below to check if STATUS is Running.
```
kubectl get pods -n synctree
```
Example output:
```
NAME                                    READY   STATUS    RESTARTS   AGE
synctree-portal-67b5d5cfd6-fhl8j        1/1     Running   0          41h
synctree-studio-7bf86c4569-xv5b2        1/1     Running   0          41h
synctree-testing-5cb4887677-sbkwt       1/1     Running   0          41h
synctree-tool-6456744d4c-8lnxc          1/1     Running   0          41h
synctree-tool-proxy-5b98d8b74-m6g46     1/1     Running   0          41h
```

<br>

# Configuring Amazon Route 53 as Your DNS Service

<br>

Copy the DNS Name of the Load Balancer  

![load_balancer](https://user-images.githubusercontent.com/103020388/163297801-c40be846-5264-48b6-aa71-9a1df9d6f9f3.png)

<br>

Set your Load Balancer's DNS name in the records for the domain you want to use. Refer to image below.

![route53_dns](https://user-images.githubusercontent.com/103020388/163304748-8045fc93-7801-4f02-a922-29032e5ab425.png)

<br>

# Connect SyncTree Studio via HTTPS

<br>

Check HTTPS connectivity to the domain you set up.
<br>
Example:
https://studio.your-domain.com

![synctree_studio_https_cert](https://user-images.githubusercontent.com/103020388/163305210-9963d090-fe83-43e5-bf5c-1c5af8763812.png)

<br>

# Configure Cluster Autoscaler

<br>

Refer to the link below for EKS Worker Node Autoscale configuration.

https://awskrug.github.io/eks-workshop/scaling/deploy_ca/
<br>
https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html
