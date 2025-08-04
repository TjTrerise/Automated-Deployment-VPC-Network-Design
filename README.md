# Automated Deployment of AWS Multi-Region VPC Network Architecture with Transit Gateway.


## Project Overview

This project involves designing an automated, robust, multi-region cloud network architecture on AWS. It leverages core networking components like custom VPCs, public/private subnets, NAT Gateways, EC2 and security groups, with a focus on establishing secure and scalable inter-region connectivity using AWS Transit Gateway.

![Architecture Design](/docs/automated-multi-region-vpc.drawio.png)

## Features

- Automated construction of a multi-region cloud network architecture on AWS.
- Leveraging custom core networking components like custom VPCs, public/private subnets, NAT Gateways, EC2 and security groups.
- Focus on establishing a secure and scalable inter-region connectivity using AWS Transit Gateway.
- This project doesn't utilise stack sets as a method to learn YAML. If you want to deploy through a stack set rather than indivudally making two region worth of YAML files that will work too. As it was my first time doing a deployment like this I wanted to do it without the sets. 

## Technologies Used

* CloudFormation
* EC2
* Linux
* Session Manager
* Transit Gateway
* VPC

# Data Types Used

* YAML

## Getting Started

To get started with deploying this AWS Multi-Region VPC Network Architecture, please ensure you meet the following prerequisites.

### Prerequisites 
 
* Fundamental knowledge of networking concepts such as:
 - IPv4.
 - Subnets.
 - Routing.
* An active AWS account with appropriate permissions to create VPCs, subnets, gateways, EC2 instances and security groups.
* Access to the AWS console.
* Able to understand YAML.
* S3 Buckets in both the US Region and EU region.


## Deployment Steps

### Step 1: Create The CloudFormation Service Role For The Project

1. Allow CloudFormation to assume the role.
2. Allow IAM.
3. Allow EC2.
4. Allow S3.

### Step2: Create The IAM Role Your Instances Will Use For Instance Profiles

1. Allow EC2 to assume the role.
2. Add AmazonSSMManagedInstanceCore
3. Add CloudWatchAgentServerPolicy

### Step 3: Create The YAML Files For The VPC

 1. Create the VPC.
 2. Create the internet gateway.
 3. Attach the internet gateway to the VPC.
 4. Create x2 public subnets and enable public IP mapping.
 6. Create x2 private subnets.
 7. Reserve x2 elastic IPs.
 8. Create x2 NAT gateways, allocate the reserved elastic IPs and attach one to each of the public subnets.
 9. Create a public route table.
 10. Create two private route tables.
 11. Associate the public route tables with the public subnets.
 12. Associate each of the private route tables with a seperate private route table.
 13. Create a an external route for the public route table pointing at the internet gateway.
 14. Create two private routes for each private subnet pointing at a seperate NAT gateway from each other.
 15. Export the values for the VPC, both private and public subnets and route tables. 



### Step 4: Create The YAML Files For The Transit Gateway

1. From the VPC stack import the VPC, private subnets and route tables as parameters. 
2. Create the transit gateway, turn off the default route table propagation.
3. Create the transit gateway route table.
4. Create the transit gateway vpc attachment.
5. Associate the transit gateway vpc attachment to the created route table.
6. Create the route to the vpc in the transit gateway route table.
7. Create the routes from the private vpc route tables to the transit gateway vpc attachment.
8. Export the transit gateway and route table. 


### Step 5: Create The YAML File For The Security Group

1. From the VPC stack import the VPC as a parameter
2. Allow inbound traffic for icmp from the US cidr range
3. Allow inbound traffic from any protocol from your that region's subnet.

 ### Step 6: Create The YAML File For The EC2 Instances

1. From the VPC stack import the VPC, private and public subnets.
2. From the security group stack import the security group.
3. Create an instance profile referencing the role that you created at the start of the project.
4. Launch an instance in each public subnet and assign them the security group that you created. Make sure the AMI is region appropriate.

### Step 7: Create The Main YAML File

1. Upload each region's YAML files into the appropriate S3 bucket.
2. Create the VPC stack, link to the vpc.yaml file.
3. Create the transit gateway stack, set the dependency on the VPC stack and add parameters for the vpc, private subnets and route tables. Link to the tgw.yaml file.
4. Create The security group stack, set the dependency on the VPC stack and add parameters for the vpc. Link to the security-groups.yaml file.
5. Create the EC2 stack, set the dependency to the VPC stack and security group stack. Add parameters for the private and public subnets, the vpc and security group. Link to the ec2.yaml file.

### Step 7: Launch the Stacks, Create The Peering Connection & Routes.

1. Create a new stack that initiates the peering connection from region 1 to region 2.
2. Once the stacks create, launch the peering stack. 
2. Manually accept the peering connection and make a copy of the peering attachment ID to add to the secondary region's peering route template. 
3. Create a peering route template for both regions. Associate the peering attachment with the previously created route table and create a route to the corresponding region.

### Step 8: Test Connectivity To Public Internet & Between Private Instances.

* Attempt to connect externally by pinging 1.1.1.1 or 8.8.8.8. 
* Attempt to communicate between private subnet in region 1 and region 2.

## Usage

- You can deploy your applications and services within the private subnets in either region, ensuring they are not directly exposed to the internet while still having controlled outbound internet access via the NAT Gateways.
- The Transit Gateway peering connection enables secure communication between resources deployed in different regions. This is ideal for distributed applications, disaster recovery, or sharing data across regions.
- The architecture inherently supports high availability across services as it is deployed across multiple regions. 
- The Transit Gateway acts as a central hub for network connectivity, simplifying routing and management of inter-VPC and inter-region traffic.
- Individual CloudFormation templates can be used to create other VPCs and resources for other projects or use cases without having to go through the manually configuration. 


## Lessons Learned & Challenges Overcome

This was the hardest project I have taken part in. Learning how to read and use YAML to create a large project such as this. 
  - Dependencies causing templates to fail to provision. It was interesting to see how each of the templates required certain parts in order to continue provisioning. For example launching the transit gateway at the same time as the VPC. 
  - The key features that you can see when you manually create a resource, finding out what the corresponding value is to input in the YAML file was tricky. It felt as if everytime you included every variable you would need, you'd forget about mapping a public IP or where you evne mapped the public IP to begin with.


## Future Enhancements

- Currently the automation is solid but can be improved. Implementing Lambda functions to grab parameters and apply them automatically so you don't need to type them into the templates for the peering connection. Also utilising Lambda would allow the peering connection to be accepted automatically. 
- Managing the infrastructure on a resource like Git so you can version the templates and upgrade, or rollback any changes that cause issues. 
- Deploying the template as a stack set. I know that I could have made it as one and then deployed across both but as it was my first time making a template like this I wanted to manually create both regions templates for learning practice. 

