# Udacity Cloud DevOps Engineer Nanodegree
### Section 'Deploy Infrastructure as Code (IAC)'
#### Project 'Deploy a high-availability web app using CloudFormation'

## Table of Contents

1. [Intro](#intro)
2. [File Structure](#file-structure)
3. [File Details](#file-details)
   1. [network.yml](#networkyml)
   2. [udagram.yml](#udagramyml)
   3. [Other Files](#other-files)
4. [Installation](#installation)
5. [Usage Instructions](#usage-instructions)
6. [Infrastructure Diagram](#infrastructure-diagram)

## Intro

This project simulates the creation of a network and other AWS resources through cloud formation templates. it simulates a realistic scenario where a dummy application is deployed to an apache2 server running on an EC2 instance.


## File structure 

* network.yml : a cloud formation template file to create the network infrastructure
* network-params.json : a cloud formation parameters json file used to create the netwrok resources
* udagram.yml : a coud formation template file to add EC2 instances, a secrity group with inbound and outbound rules, an autoscaling group, a load balancer, a public S3 buckets and other resources
* udagram-params.json : a cloud formation parameters json file used to create the resources in udagram.yml
* run.sh : a bash script to execute the deployment of the resources (see usage instructions)

## File details
### network.yml 

The templates uses several intrinsic functions:
* **!Ref** : references a different resource in the same template or one of the parameters that are then taken from the paramater JSON file
* **!Sub** : substitutes a placeholder in a string with a parameter, for example ``${ProjectName}-public-subnet1`` will become ``udacity-vpc-subents-exercise-public-subnet1`` because the ProjectName is a parameter with value ``udacity-vpc-subents-exercise``
* **!GetAtt** : gets an attribute from another resource, for example a static ip address from an Elastic IP address resource, ``!GetAtt NatGateway1EIP.AllocationId``
* **!Select** : gets an item from a list generated with another intrinsic function, for example ``!Select [1, !GetAZs '']`` to select availability zone 1 from the availability zone available for the default region (region can be specified in the empty string).
* **!Join** : concatenates values with a specific delimiter, for example, to create a list with the IDs of all subnets ``!Join [ ",", [ !Ref PrivateSubnet1, !Ref PrivateSubnet2 ]]``

This template creates the following resources:

* VPC
* Internet Gateway attached with the VPC
* 4 Private subnets in different availability zones to increase availability (2 private and 2 public)
* 2 Nat Gateways - The NAT Gateway is attached to the public subnets and it allows inbound traffic to the private subnets ***if*** initiated by the private subnets themselves. This can be useful to allow access to the internet to download updates for the resources within the VPC.
  The NAT gateways perform Network Address Translation for the private subnets using a public address and that is why they are attached to the public subnets.
* Elastic IP for the NAT gateways. This is required so that the IP does not change and any resources or services that depend on the NAT gateway's IP address won't be affected by changes.
* Route tables

The template also has a 'Parameters' section to reference parameters in a JSON file and an 'Output' section to save information about the resources created:

![image](https://github.com/dedalus94/iac-project/assets/49538048/2524a4f7-d54c-45a0-ae5e-a415cacbaf21)

### udagram.yml

This template creates the following resources:

* SecurityGroup (Type: AWS::EC2::SecurityGroup):
  * This security group is created for the VPC deployed by using the network.yml template
  * the *VPC id* is imported from the Outputs generated by deploying network.yml using the intrinsic function `Fn::ImportValue`.
  * The security group grants inbound access by the Load Balancer Security Group on port 80.
  * The security group grants unrestricted outbound access to the internet. <br> ![image](https://github.com/dedalus94/iac-project/assets/49538048/59e6bcb1-03e5-4ce6-9c20-3f59c0bc82ec)

* LaunchTemplate (Type: AWS::EC2::LaunchTemplate):
  * A launch template that contains some configuration information for the instances launched by the autoscaling group. I have used an Ubuntu 22 AMI and the `UserData` property contains bash code that will install and start apache2 for each instance created.
  * The `SecurityGroupIds` property will attach the previously defined security group to each instance. The storage and the instance type configuration are also defined.
  * `IamInstanceProfile` assigns an Instance Profile role to the instances (see the IAMRole & InstanceProfile resources).

* AutoScalingGroup (Type: AWS::AutoScaling::AutoScalingGroup):
  * > defines an Amazon EC2 Auto Scaling group, which is a collection of Amazon EC2 instances that are treated as a logical grouping for the purposes of automatic scaling and management - [AWS Docs](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-autoscaling-autoscalinggroup.html)
  * The `MinSize` & `MaxSize` properties set the range of machines that will be running
  * The `VPCZoneIdentifier` property is a list of subnet IDs for VPC where instances in the Auto Scaling group can be created. In this case, I am using the private subnets from the network stack.
  * The `TargetGroupARNs` property links the Autoscaling Group to the Load Balancer via the Listener & Listener Rule resources (see the TargetGroup resource)
    
* LoadBalancerSecurityGroup (Type: AWS::EC2::SecurityGroup): A Security group that allows (http) access on port 80 from any IP. The Security group is attached to the Load Balancer resource.
  
* LoadBalancer (Type: AWS::ElasticLoadBalancingV2::LoadBalancer): A Load Balancer deployed on the public subnets to forward traffic evenly across servers.
  
* Listener & Listener Rule: It will listen to load balancer connections on port 80, and forward them to the target group (see the TargetGroup resource)
  
* TargetGroup (Type: AWS::ElasticLoadBalancingV2::TargetGroup): The Load Balancer forwards traffic evenly across a group of servers, but since these servers are part of an autoscaling group they may come and go as demand increases or decreases and therefore they cannot be directly referenced. the target group handles this issue for us and also runs health checks to test the status of the targets.
  
* S3Bucket: This resource deploys a publicly accessible S3 bucket
  
* IAMRole: grants EC2 instances with read/write permissions for an S3 bucket
  
* InstanceProfile: an instance profile is used to pass an IAM role to an EC2 instance

  
### Other files:

- run.sh: a bash script to automate the deployment of stacks. <br>
Arguments:
  
  - EXECUTION_MODE=$1 (only allows 'preview', 'deploy and 'delete)
  - REGION=$2 (AWS region)
  - STACK_NAME=$3 (desired name for the cloud formation stack)
  - TEMPLATE_FILE_NAME=$4 (yaml file with resource definitions - not required in execution mode delete)
  - PARAMETERS_FILE_NAME=$5 (json file with parameters used by resources - not required in execution mode delete)
 
Example usage (execution mode = 'deploy'):
```
bash run.sh deploy us-east-1 udagram-server-stack udagram.yml udagram-params.json 
``` 
Example usage (execution mode = 'preview' - change set is not executed):
```
bash run.sh preview us-east-1 udagram-server-stack udagram.yml udagram-params.json 
```
Example usage (execution mode = 'delete'):
```
bash run.sh delete us-east-1 udagram-server-stack
```



## Installation 
1. Copy the repo locally:
   ```
     git clone https://github.com/dedalus94/iac-project.git
   ```
5. Install and configure AWS CLI (Command Line Interface)

## Usage Instructions 

1. Deploy the resources in order. the network infrastructure first
   ```
   bash run.sh deploy us-east-1 network-stack network.yml network-params.json
   ```
2. And the servers next
   ```
   bash run.sh deploy us-east-1 udagram-server-stack udagram.yml udagram-params.json
   ```
If the creation is completed without errors the result should be visible in the AWS console - CloudFormation > Stacks: 
<br>
![image](https://github.com/dedalus94/iac-project/assets/49538048/6b9b9b4b-01eb-4c8d-b35f-d93973ba1b04)

The Load Balancer DNS is saved as an Output for the second stack:
<br>
![image](https://github.com/dedalus94/iac-project/assets/49538048/c53d62e4-0e68-4294-8b4e-e816799cc91f)

And if the servers are running fine, the welcome page should be displayed at that link:
<br>
![image](https://github.com/dedalus94/iac-project/assets/49538048/bdcd8ef1-3cb1-4406-ad3e-765a0647b11e)



## Infrastructure diagram
<br>

![Untitled Diagram](https://github.com/dedalus94/iac-project/assets/49538048/c47f7dd7-200d-4e11-8399-3ea31adb6f95)


