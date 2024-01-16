# Helios FHIR Server AWS Reference Architecture

## Introduction
This document provides recommended practices and a comprehensive setup instructions for a reference architecture for the Helios FHIR Server on AWS.

## Reference Architecture
This reference architecture consists of the following components:
- An Amazon Virtual Private Cloud (helios-vpc)
- Public and private subnets across two different availability zones for a total of 4 subnets.
- A bastion Linux instance used to provide secure access to Linux instances located in the private and public subnets of your virtual private cloud.  SSH to this instance to perform initial Cassandra cluster installation tasks, and occasional maintenance as required.
- Configuration for using [AxonOps](https://axonops.com/) - a cloud native solution to monitor, maintain and backup your Cassandra cluster.
- A 5-node Cassandra Cluster with nodes spread across two availability zones.
- Helios FHIR Server installed in an Amazon EKS Cluster and configured for auto-scaling.
- IAM Roles and Policies
- Security Groups

![Helios FHIR Server AWS Reference Architecture](hfs-aws.png)

## Installation Prerequisites

- [Install Terraform](https://developer.hashicorp.com/terraform/install) on your local machine.
- Have an [AWS IAM User Account](https://aws.amazon.com/iam) and [create access keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html#Using_CreateAccessKey). In order to successfully provision this reference architecture the user must also be permitted to create the following AWS resources:
  - VPC - AmazonVPCFullAccess
  - EC2 - AmazonEC2FullAccess
  - IAM Role - IAMFullAccess
  - EKS - You will need to add a custom policy named EKS-Full-Access
    - `{
      "Version": "2012-10-17",
      "Statement": [
      {
      "Sid": "eksadministrator",
      "Effect": "Allow",
      "Action": "eks:*",
      "Resource": "*"
      }
      ]
      }`
- Signup for an [AxonOps](https://axonops.com/) account and obtain an Agent Key. 

## Installation Overview
There are two high level steps in this automation:

1. Use Terraform to provision the resources in the `terraform` folder.
2. SSH to the Bastion Linux instance and run Ansible automation to provision Cassandra cluster on the Linux hosts provisioned in step 1.

## Installation Steps

### Preparation ###
1. Clone this repository.
2. At the root of this repository, manually create the following file.
```
./.aws/config
```
3. Add the following content to ./.aws/config
```
export AWS_ACCESS_KEY_ID=<AWS_ACCESS_KEY_ID>
export AWS_SECRET_ACCESS_KEY=<AWS_SECRET_ACCESS_KEY>
export AWS_DEFAULT_REGION=<AWS_DEFAULT_REGION>
export AXONOPS_ORGANIZATION_NAME=<Your organization name you registered with https://axonops.com/>
export AXONOPS_AGENT_KEY=<Your AxonOps Agent Key>
export TF_VAR_AWS_DEFAULT_REGION=us-east-1
export TF_VAR_CASSANDRA_CLUSTER_NAME=helios
export TF_VAR_ENVIRONMENT=Production
```
4. Run `source ./.aws/config`
5. Run `cd terraform`
6. Run `terraform init`

### Configure Terraform Variables ###

The `variables.tf` file contains several variables that you may want to modify such as specific AWS Regions, Zones and other settings.

- Required - Replace the local_ssh_pubkey default value to ~/.ssh/id_rsa.pub or ~/.ssh/id_ed25519.pub (default)  This automation uses your pre-existing public key to access the remote Linux Bastion instance.  If you don't yet have one in the `~/.ssh` folder, create one with this command `ssh-keygen -t ed25519`

### Run Terraform Apply ###

Run `terraform apply`

The provisioning will take a while (15 mins) but the command execution should show the progress.

While this command runs, you may proceed with the next step in another terminal if the Bastion Linux instance has been created and is ready.

### Upload your local .aws/config file and SSH to the Bastion Linux Instance ###

The bastion public IP address can now be found in the AWS EC2 console named `Helios Bastion Server`.

From your local machine, copy your local config file to the Bastion Linux Instance, then ssh to it:
```
cd ../.aws
scp config ubuntu@[bastion ip address]:~/hfs-install/.aws/config
ssh ubuntu@[bastion ip address]
```
Please NOTE:  It may take some time for the setup steps of the Bastion instance to complete.  The steps will be complete when the `/home/ubuntu/.ssh/id_ed25519` is available on the Bastion instance. 

Verify that the .aws/config file is present and correct

`cat hfs-install/.aws/config`

### Execute Cassandra Ansible Setup Scripts fom the Bastion Linux Instance ###
```
cd hfs-install
source .aws/config
cd ansible
bash apache-cassandra-axonops.sh
```
### Start Cassandra ###
Use the following command to start the cluster from the `ansible` folder.
```
bash start-cassandra.sh
```
### Verify that your Cluster is Visible in AxonOps ###
Login to your [AxonOps](https://axonops.com/) account and verify that your Cassandra Cluster is available.  Please be aware, this may take several minutes before all cluster nodes are visible in AxonOps.
### Common EKS Issue ###
If in the AWS EKS Console, you're seeing the following error message:

`Your current IAM principal doesn't have access to Kubernetes objects on this cluster.
This may be due to the current user or role not having Kubernetes RBAC permissions to describe cluster resources or not having an entry in the cluster’s auth config map.`

While logged in to the Bastion Linux instance, run the following command to enable `kubectl` commands.

`aws eks update-kubeconfig --region $TF_VAR_AWS_DEFAULT_REGION --name helios-eks-cluster`

Then, follow the instructions in this [StackOverflow page](https://stackoverflow.com/questions/70787520/your-current-user-or-role-does-not-have-access-to-kubernetes-objects-on-this-eks) to remedy.

### Login To Your Helios FHIR Server Instance! ###
There are three ways to locate the Loadbalancer URL for your cluster.
- In the AWS Console, navigate to EKS > Clusters > helios-eks-cluster, then select Resources, then Service and networking, then Services.  Select helios-fhir-server.
- In the AWS Console, navigate to EC2 > Load balancers, and selected the load balancer.
- On the Bastion Linux instance, execute `kubectl get service -n helios-fhir-server`

Your Load balancer URL will look something like this:

`a83a8d424b4b5456295447ba4c2139f9-350762241.us-east-1.elb.amazonaws.com` (Yours will be different)

Place that URL into a browser, and append `/ui` to access the Administrative User Interface.

Appending `/fhir`to the Load balancer URL is the FHIR Server's root address. 