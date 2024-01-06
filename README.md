# Udacity Cloud DevOps Engineer Nanodegree
### Section 'Deploy Infrastructure as Code (IAC)'
#### Project 'Deploy a high-availability web app using CloudFormation'

## Intro

This project simulates the creation of a network and other AWS resources through cloud formation templates.

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
* 2 Nat Gateways - The NAT Gateway is attached to the public subnets and it allows inbound traffic to the private subnets *if* initiated by the private subnets. This can be useful for example to download updates for the resources within the VPC from the internet.
  The NAT gateways perform Network Address Translation for the private subnets using a public address and that is why they are attached to the public subnets.
* Elastic IP for the NAT gateways, this is required so that the IP does not change and any resources or services that depend on the NAT gateway's IP address won't be affected by changes.
* Route tables

The template also has a 'Parameters' section to reference parameters in a JSON file and an 'Output' section to save information about the resources created:

![image](https://github.com/dedalus94/cloud-formation-IAC-scripts/assets/49538048/059b35df-b697-4460-b53b-79beadafd855)

Other files:

- parameters2.json : new parameters file
  
- create.sh : bash script to create a stack through AWS CLI using a cloud formation template and a parameter file (takes 3 parameters: cloud stack name, template file and parameter file)
  
- update.sh : bash script to update a cloud formation stack, take the same parameters as the previous bash file

### A map of more complex VPC is also available on AWS with traffic rules highlighted:

Public traffic
![image](https://github.com/dedalus94/cloud-formation-IAC-scripts/assets/49538048/ffad20fd-4788-4482-b4fe-73b3d53b19ee)


Private traffic: 
![image](https://github.com/dedalus94/cloud-formation-IAC-scripts/assets/49538048/20d7f098-ae1f-4f11-a70c-e4a3c7f703cf)

## Installation 

The files can be run through a terminal and they require AWS CLI to be installed and configured.


## Infrastructure diagram

![image](https://github.com/dedalus94/cloud-formation-IAC-scripts/assets/49538048/f00ec782-c634-4b69-8330-cb466971ce07)


