
## Authoring Access Resource Automation

Now that we have our networking, compute, and database Resources defined, we need to finish out the IAM Resources and Security Groups for the stack, in order to allow access to DynamoDB and HTTP traffic to the ELB, respectively.

First, let's build out the IAM Role and IAM Instance Profile which allow access to DynamoDB...:

```javascript
// ... inside "Resources"
		"DynamoAccessRole": {
			"Type": "AWS::IAM::Role",
			"Properties": {
				"AssumeRolePolicyDocument": { JSON },
				"ManagedPolicyArns": [ String, ... ],
				"Path": String,
				"Policies": [ Policies, ... ]
			}
		},
		"DynamoAccessInstanceProfile": {
			"Type": "AWS::IAM::InstanceProfile",
			"Properties": {
				"Path": String,
				"Roles": [ IAM Roles ]
			}
		},
// ...
```

This one can be a done a few ways.
 - The `DynamoAccessInstanceProfile` is simple - use a `Path` of `"/"`.
 - The element in `Roles` should refer to `DynamoAccessRole`.
 - Write a normal IAM policy document to allow access to DynamoDB resources.


There is a lot of flexibility here, I used the simplest Policy I could, albeit without Resource level restrictions you may elect to include...:

```javascript
// ... inside "Resources"
		"DynamoAccessRole": {
			"Type": "AWS::IAM::Role",
			"Properties": {
				"AssumeRolePolicyDocument": {
					"Version": "2012-10-17",
					"Statement": [{
						"Effect": "Allow",
						"Principal": { "Service": [ "ec2.amazonaws.com" ]},
						"Action": [ "sts:AssumeRole" ]
					}]
				},
				"Path": "/",
				"Policies": [{
					"PolicyName": "AllowTodoTableAccess",
					"PolicyDocument": {
						"Version": "2012-10-17",
						"Statement": [{
							"Action": [ "dynamodb:*" ],
							"Effect": "Allow",
							"Resource": [ "*" ]
						}]
					}
				}]
			}
		},
		"DynamoAccessInstanceProfile": {
			"DependsOn": [
				"DynamoAccessRole"
			],
			"Type": "AWS::IAM::InstanceProfile",
			"Properties": {
				"Path": "/",
				"Roles": [{ "Ref": "DynamoAccessRole" }]
			}
		},
// ...
```

After building out the IAM Resources, all that remains is the set of the Security Groups we planned for out system: `LBToInstancesSecurityGroup`, `InternetToLBSecurityGroup`, `InstancesToNATSecurityGroup`.

```javascript
// ... inside "Resources"
		"LBToInstancesSecurityGroup": {
			"Type": "AWS::EC2::SecurityGroup",
			"Properties": {
				"GroupDescription": String,
				"SecurityGroupEgress": [ Security Group Rule, ... ],
				"SecurityGroupIngress": [ Security Group Rule, ... ],
				"Tags": [ Resource Tag, ... ],
				"VpcId": String
			}
		},
		"InternetToLBSecurityGroup": {
			"Type": "AWS::EC2::SecurityGroup",
			"Properties": {
				"GroupDescription": String,
				"SecurityGroupEgress": [ Security Group Rule, ... ],
				"SecurityGroupIngress": [ Security Group Rule, ... ],
				"Tags": [ Resource Tag, ... ],
				"VpcId": String
			}
		},
		"InstancesToNATSecurityGroup": {
			"Type": "AWS::EC2::SecurityGroup",
			"Properties": {
				"GroupDescription": String,
				"SecurityGroupEgress": [ Security Group Rule, ... ],
				"SecurityGroupIngress": [ Security Group Rule, ... ],
				"Tags": [ Resource Tag, ... ],
				"VpcId": String
			}
		}
// ...
```


There are compound field values here. We need to allow: 
 - All inbound TCP traffic over port `8888` from InternetToLBSecurityGroup to `LBToInstancesSecurityGroup`
 - All outbound traffic from `LBToInstancesSecurityGroup` to anywhere
 - All inbound TCP over port `80` from anywhere to `InternetToLBSecurityGroup`
 - All outbound traffic from `InstancesToNATSecurityGroup` to anywhere
 - All inbound traffic from `LBToInstancesSecurityGroup` to `InstancesToNATSecurityGroup`

Can you figure it out by reading the documentation for the `AWS::EC2::SecurityGroup` Resource Properties field definitions? Check your work below: 

```javascript
// ... inside "Resources"
		"LBToInstancesSecurityGroup": {
			"DependsOn": [
				"InternetToLBSecurityGroup"
			],
			"Type": "AWS::EC2::SecurityGroup",
			"Properties": {
				"GroupDescription": "Allows LB to hit instances and instances to NAT",
				"SecurityGroupEgress": [{
					"CidrIp": "0.0.0.0/0",
					"FromPort": 0,
					"IpProtocol": "-1",
					"ToPort": 65535
				}],
				"SecurityGroupIngress": [{
					"FromPort": 8888,
					"IpProtocol": "TCP",
					"SourceSecurityGroupId": { "Ref": "InternetToLBSecurityGroup" },
					"ToPort": 8888
				}],
				"VpcId": { "Ref": "VPC" }
			}
		},
		"InternetToLBSecurityGroup": {
			"Type": "AWS::EC2::SecurityGroup",
			"Properties": {
				"GroupDescription": "Allows the internet to hit the LB.",
				"SecurityGroupIngress": [{
					"CidrIp": "0.0.0.0/0",
					"FromPort": 80,
					"IpProtocol": "TCP",
					"ToPort": 80
				}],
				"VpcId": { "Ref": "VPC" }
			}
		},
		"InstancesToNATSecurityGroup": {
			"DependsOn": [
				"LBToInstancesSecurityGroup"
			],
			"Type": "AWS::EC2::SecurityGroup",
			"Properties": {
				"GroupDescription": "Allows the instance to hit NAT, and NAT outbound.",
				"SecurityGroupEgress": [{
					"CidrIp": "0.0.0.0/0",
					"FromPort": 0,
					"IpProtocol": "-1",
					"ToPort": 65535
				}],
				"SecurityGroupIngress": [{
					"FromPort": 0,
					"IpProtocol": "-1",
					"SourceSecurityGroupId": { "Ref": "LBToInstancesSecurityGroup" },
					"ToPort": 65535
				}],
				"VpcId": { "Ref": "VPC" }
			}
		}
// ...
```


That's the end of the IAM and Security Group Resources for our Stack Template, and the last Resource group we had to define! Next, we need to finish out some final touches to enable Template use across all Regions, using **Mappings**. Hit **Next Step** now, or use the full Template definition below to check your work.


## [NEXT](../005-flexibility/README.md)

```javascript
{
	"Description": "My Learn The Cloud Labs 2-Tier Scalable API Template",
	"Parameters": {
		"VPCAZ": {
			"Type": "AWS::EC2::AvailabilityZone::Name",
			"Description": "The Availability Zone to use for the simple VPC."
		},
		"NumberOfNodes": {
			"Type": "Number",
			"Description": "The number of EC2 nodes the service should have.",
			"Default": 2
		},
		"NodeSize": {
			"Type": "String",
			"Description": "The type of node to use for your system",
			"Default": "m3.medium"
		},
		"RunNumber": {
			"Type": "Number",
			"Description": "Used to force the ASG's nodes' UserData to update.",
			"Default": 1
		},
		"TableThroughput": {
			"Type": "Number",
			"Description": "The number of queries per second the table should support.",
			"Default": 10
		}
	},
	"Resources": {
		"VPC": {
			"Type": "AWS::EC2::VPC",
			"Properties": {
				"CidrBlock": "10.0.0.0/16"
			}
		},
		"PublicSubnet": {
			"DependsOn": [
				"VPC"
			],
			"Type": "AWS::EC2::Subnet",
			"Properties": {
				"AvailabilityZone": { "Ref": "VPCAZ" },
				"CidrBlock": "10.0.0.0/24",
				"MapPublicIpOnLaunch" : true,
				"VpcId": { "Ref": "VPC" }
			}
		},
		"PrivateSubnet": {
			"DependsOn": [
				"VPC"
			],
			"Type": "AWS::EC2::Subnet",
			"Properties": {
				"AvailabilityZone": { "Ref": "VPCAZ" },
				"CidrBlock": "10.0.1.0/24",
				"VpcId": { "Ref": "VPC" }
			}
		},

		"GatewayToInternet": {
			"Type": "AWS::EC2::InternetGateway"
		},
		"NATInstance": {
			"DependsOn": [
				"GatewayAttachmentToVPC",
				"InstancesToNATSecurityGroup",
				"PublicSubnet"
			],
			"Type": "AWS::EC2::Instance",
			"Properties": {
				"AvailabilityZone": { "Ref": "VPCAZ" },
				"ImageId": "??????",
				"InstanceType": "m3.medium",
				"SecurityGroupIds": [
					{ "Ref": "InstancesToNATSecurityGroup" }
				],
				"SourceDestCheck": false,
				"SubnetId": { "Ref": "PublicSubnet" }
			}
		},
		"RoutesForPublicSubnet": {
			"DependsOn": [
				"VPC"
			],
			"Type": "AWS::EC2::RouteTable",
			"Properties": {
				"VpcId": { "Ref": "VPC" }
			}
		},
		"RoutesForPrivateSubnet": {
			"DependsOn": [
				"VPC"
			],
			"Type": "AWS::EC2::RouteTable",
			"Properties": {
				"VpcId": { "Ref": "VPC" }
			}
		},
		"GenericNACL": {
			"DependsOn": [
				"VPC"
			],
			"Type": "AWS::EC2::NetworkAcl",
			"Properties": {
				"VpcId": { "Ref": "VPC" }
			}
		},

		"GatewayAttachmentToVPC": {
			"DependsOn": [
				"GatewayToInternet",
				"VPC"
			],
			"Type": "AWS::EC2::VPCGatewayAttachment",
			"Properties": {
				"InternetGatewayId": { "Ref": "GatewayToInternet" },
				"VpcId": { "Ref": "VPC" }
			}
		},
		"RouteToGateway": {
			"DependsOn": [
				"GatewayToInternet",
				"GatewayAttachmentToVPC",
				"RoutesForPublicSubnet"
			],
			"Type": "AWS::EC2::Route",
			"Properties": {
				"DestinationCidrBlock": "0.0.0.0/0",
				"GatewayId": { "Ref": "GatewayToInternet" },
				"RouteTableId": { "Ref": "RoutesForPublicSubnet" }
			}
		},
		"RouteToNat": {
			"DependsOn": [
				"NATInstance",
				"RoutesForPrivateSubnet"
			],
			"Type": "AWS::EC2::Route",
			"Properties": {
				"DestinationCidrBlock": "0.0.0.0/0",
				"InstanceId": { "Ref": "NATInstance" },
				"RouteTableId": { "Ref": "RoutesForPrivateSubnet" }
			}
		},
		"NACLInboundEntry": {
			"DependsOn": [
				"GenericNACL"
			],
			"Type": "AWS::EC2::NetworkAclEntry",
			"Properties": {
				"CidrBlock": "0.0.0.0/0",
				"Egress": false,
				"NetworkAclId": { "Ref": "GenericNACL" },
				"PortRange": {
					"From": "0",
					"To": "65535"
				},
				"Protocol": "6",
				"RuleAction": "allow",
				"RuleNumber": 100
			}
		},
		"NACLOutboundEntry": {
			"DependsOn": [
				"GenericNACL"
			],
			"Type": "AWS::EC2::NetworkAclEntry",
			"Properties": {
				"CidrBlock": "0.0.0.0/0",
				"Egress": true,
				"NetworkAclId": { "Ref": "GenericNACL" },
				"PortRange": {
					"From": "0",
					"To": "65535"
				},
				"Protocol": "6",
				"RuleAction": "allow",
				"RuleNumber": 100
			}
		},
		"NACLBindingForPublicSubnet": {
			"DependsOn": [
				"PublicSubnet",
				"GenericNACL"
			],
			"Type": "AWS::EC2::SubnetNetworkAclAssociation",
			"Properties": {
				"SubnetId": { "Ref": "PublicSubnet" },
				"NetworkAclId": { "Ref": "GenericNACL" }
			}
		},
		"NACLBindingForPrivateSubnet": {
			"DependsOn": [
				"PrivateSubnet",
				"GenericNACL"
			],
			"Type": "AWS::EC2::SubnetNetworkAclAssociation",
			"Properties": {
				"SubnetId": { "Ref": "PrivateSubnet" },
				"NetworkAclId": { "Ref": "GenericNACL" }
			}
		},
		"RoutesBindingForPublicSubnet": {
			"DependsOn": [
				"RoutesForPublicSubnet",
				"PublicSubnet"
			],
			"Type": "AWS::EC2::SubnetRouteTableAssociation",
			"Properties": {
				"RouteTableId": { "Ref": "RoutesForPublicSubnet" },
				"SubnetId": { "Ref": "PublicSubnet" }
			}
		},
		"RoutesBindingForPrivateSubnet": {
			"DependsOn": [
				"RoutesForPrivateSubnet",
				"PrivateSubnet"
			],
			"Type": "AWS::EC2::SubnetRouteTableAssociation",
			"Properties": {
				"RouteTableId": { "Ref": "RoutesForPrivateSubnet" },
				"SubnetId": { "Ref": "PrivateSubnet" }
			}
		},


		"LoadBalancer": {
			"DependsOn": [
				"InternetToLBSecurityGroup",
				"PublicSubnet"
			],
			"Type": "AWS::ElasticLoadBalancing::LoadBalancer",
			"Properties": {
				"Listeners": [{
					"InstancePort": 8888,
					"InstanceProtocol": "HTTP",
					"LoadBalancerPort": 80,
					"Protocol": "HTTP"
				}],
				"Scheme": "internet-facing",
				"SecurityGroups": [{ "Ref": "InternetToLBSecurityGroup" }],
				"Subnets": [{ "Ref": "PublicSubnet" }]
			}
		},
		"InstancesGroup": {
			"DependsOn": [
				"InstancesConfig",
				"LoadBalancer",
				"PrivateSubnet"
			],
			"Type": "AWS::AutoScaling::AutoScalingGroup",
			"Properties": {
				"Cooldown": 60,
				"DesiredCapacity": { "Ref": "NumberOfNodes" },
				"HealthCheckGracePeriod": 30,
				"HealthCheckType": "EC2",
				"LaunchConfigurationName": { "Ref": "InstancesConfig" },
				"LoadBalancerNames": [{ "Ref": "LoadBalancer" }],
				"MaxSize": { "Ref": "NumberOfNodes" },
				"MetricsCollection": [{ "Granularity": "1Minute" }],
				"MinSize": { "Ref": "NumberOfNodes" },
				"VPCZoneIdentifier": [{ "Ref": "PrivateSubnet" }]
			},
			"UpdatePolicy": {
				"AutoScalingRollingUpdate": {
					"MaxBatchSize": 1,
					"MinInstancesInService": 1,
					"PauseTime": 30,
					"WaitOnResourceSignals": false
				}
			}
		},
		"InstancesConfig": {
			"DependsOn": [
				"DynamoAccessInstanceProfile",
				"LBToInstancesSecurityGroup"
			],
			"Type": "AWS::AutoScaling::LaunchConfiguration",
			"Properties": {
				"AssociatePublicIpAddress": false,
				"IamInstanceProfile": { "Ref": "DynamoAccessInstanceProfile" },
				"ImageId": "??????",
				"InstanceMonitoring": true,
				"InstanceType": { "Ref": "NodeSize" },
				"SecurityGroups": [{ "Ref": "LBToInstancesSecurityGroup" }],
				"UserData": { "Fn::Base64": { "Fn::Join": [ "\n", [
					"#!/bin/bash",
					{ "Fn::Join": [ "", [
						"echo 'Running number ",
						{ "Ref": "RunNumber" },
						"' > /tmp/runnum.log"
					]]},
					"yum install --enablerepo=epel -y git nodejs npm",
					"npm install -g forever",
					"git clone https://github.com/andrew-templeton/dynamo-demo",
					{ "Fn::Join": [ "", [
						"export AWS_REGION=", { "Ref": "AWS::Region" }
					]]},
					"cd dynamo-demo",
					{ "Fn::Join": [ "", [
						"forever start server.js --port 8888 --table ",
						{ "Ref": "DynamoTableForTodos" } 
					]]}
				]]}}
			}
		},
		"DynamoTableForTodos": {
			"Type": "AWS::DynamoDB::Table",
			"Properties": {
				"AttributeDefinitions": [{
					"AttributeName": "id",
					"AttributeType": "S"
				}],
				"KeySchema": [{
					"AttributeName": "id",
					"KeyType": "HASH"
				}],
				"ProvisionedThroughput": {
					"ReadCapacityUnits": { "Ref": "TableThroughput" },
					"WriteCapacityUnits": { "Ref": "TableThroughput" }
				}
			}
		},

		"DynamoAccessRole": {
			"Type": "AWS::IAM::Role",
			"Properties": {
				"AssumeRolePolicyDocument": {
					"Version": "2012-10-17",
					"Statement": [{
						"Effect": "Allow",
						"Principal": { "Service": [ "ec2.amazonaws.com" ]},
						"Action": [ "sts:AssumeRole" ]
					}]
				},
				"Path": "/",
				"Policies": [{
					"PolicyName": "AllowTodoTableAccess",
					"PolicyDocument": {
						"Version": "2012-10-17",
						"Statement": [{
							"Action": [ "dynamodb:*" ],
							"Effect": "Allow",
							"Resource": [ "*" ]
						}]
					}
				}]
			}
		},
		"DynamoAccessInstanceProfile": {
			"DependsOn": [
				"DynamoAccessRole"
			],
			"Type": "AWS::IAM::InstanceProfile",
			"Properties": {
				"Path": "/",
				"Roles": [{ "Ref": "DynamoAccessRole" }]
			}
		},

		"LBToInstancesSecurityGroup": {
			"DependsOn": [
				"InternetToLBSecurityGroup"
			],
			"Type": "AWS::EC2::SecurityGroup",
			"Properties": {
				"GroupDescription": "Allows LB to hit instances and instances to NAT",
				"SecurityGroupEgress": [{
					"CidrIp": "0.0.0.0/0",
					"FromPort": 0,
					"IpProtocol": "-1",
					"ToPort": 65535
				}],
				"SecurityGroupIngress": [{
					"FromPort": 8888,
					"IpProtocol": "TCP",
					"SourceSecurityGroupId": { "Ref": "InternetToLBSecurityGroup" },
					"ToPort": 8888
				}],
				"VpcId": { "Ref": "VPC" }
			}
		},
		"InternetToLBSecurityGroup": {
			"Type": "AWS::EC2::SecurityGroup",
			"Properties": {
				"GroupDescription": "Allows the internet to hit the LB.",
				"SecurityGroupIngress": [{
					"CidrIp": "0.0.0.0/0",
					"FromPort": 80,
					"IpProtocol": "TCP",
					"ToPort": 80
				}],
				"VpcId": { "Ref": "VPC" }
			}
		},
		"InstancesToNATSecurityGroup": {
			"DependsOn": [
				"LBToInstancesSecurityGroup"
			],
			"Type": "AWS::EC2::SecurityGroup",
			"Properties": {
				"GroupDescription": "Allows the instance to hit NAT, and NAT outbound.",
				"SecurityGroupEgress": [{
					"CidrIp": "0.0.0.0/0",
					"FromPort": 0,
					"IpProtocol": "-1",
					"ToPort": 65535
				}],
				"SecurityGroupIngress": [{
					"FromPort": 0,
					"IpProtocol": "-1",
					"SourceSecurityGroupId": { "Ref": "LBToInstancesSecurityGroup" },
					"ToPort": 65535
				}],
				"VpcId": { "Ref": "VPC" }
			}
		}
	}
}
```