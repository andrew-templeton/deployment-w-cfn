

## Scalable DevOps: Using CloudFormation for Whole VPC Stacks


Whether running on AWS, another cloud provider, or other infrastructure, while many companies practice code deployment automation for business logic and applications, far fewer practice **full-stack deployment automation**. Most businesses recognize that the automation of code deployment offers major operational benefits and time savings. This Lab provides a complete walkthrough for the design and development of **full-stack deployment automation** on AWS, using **AWS CloudFormation**.

Some of the key benefits of learning stack deployment automation:
 - Ensure true parity bewteen Development, Test, Staging, and Production systems
 - Recoverability during disasters with rapid re-deployment of whole clouds
 - Ease of multi-region deployment consistency
 - Enable full-cloud integration testing

During this lab, you will learn how to take business requirements for a cloud system on AWS, design a stack for the requirements, and implement total automation for deployment of the stack by authoring a complete **AWS CloudFormation Template** for a stack, including:

 - A VPC with private and public subnets
 - A NAT instance for access to the internet from a private subnet
 - Route tables and Network ACLs for the VPC
 - An Elastic Load Balancer and Autoscaling Group of EC2 Instances running code
 - Code deployment automation
 - Security Groups for the EC2 Instances and Elastic Load Balancer
 - DynamoDB Table to persist data, and IAM Roles to allow EC2 Instance access


## [ENTER LAB](./001-planning/README.md)
