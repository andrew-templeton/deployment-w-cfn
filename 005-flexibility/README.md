
## Finishing It All Up


Before we try launching a Stack with the Template, we need to provide an **Output** for the ELB DNS to visit when the stack finishes creating, and use **Mappings** to make this system work in all Regions.

First, we need an **Output** for what we will call the **ELBDNS** - a URL to visit to see the finished result of the Stack. We want something like `http://$ELB_HOST/index.html`, so we need to use `Fn::Join` to append the prefix and suffix, and `Fn::GetAtt` to get the value of the ELB host/DNS.

```javascript
// ... inside TOP LEVEL of the JSON object.
// ... Outputs is a sibling property of Resources.
	"Outputs": {
		"ELBDNS": {
			"Description": "The DNS for the ELB / our app.",
			"Value": {
				"Fn::Join": [
					"",
					[
						"http://",
						{
							"Fn::GetAtt": [
								"LoadBalancer",
								"DNSName"
							]
						},
						"/index.html"
					]
				]
				
			}
		}
	}
// ...
```

Once we set this up, we need to enable multi-Region support in this system, by adding **Mappings** for the Amazon Linux AMIs we will use when launching the Autoscaling Groups / Launch Configurations, and a **Mapping** for the `NATInstance` AMI.

You can run a web search to find list of Amazon Linux AMIs, but you would have to find the AMIs for each region on your own. Rather than spending time doing that, see the below snippet for the **Mappings** section of the template and put it into your template.

```javascript
// ... inside TOP LEVEL of the JSON object.
// ... Mappings is a sibling property of Resources.
// ... Most people put Mappings above Resources, below Parameters.
	"Mappings": {
		"AWSNATAMIs": {
			"us-east-1": 		{ "AMI": "ami-303b1458" },
			"us-west-2": 		{ "AMI": "ami-69ae8259" },
			"us-west-1": 		{ "AMI": "ami-7da94839" },
			"eu-west-1": 		{ "AMI": "ami-6975eb1e" },
			"eu-central-1": 	{ "AMI": "ami-46073a5b" },
			"ap-southeast-1": 	{ "AMI": "ami-b49dace6" },
			"ap-northeast-1": 	{ "AMI": "ami-03cf3903" },
			"ap-southeast-2": 	{ "AMI": "ami-e7ee9edd" },
			"sa-east-1": 		{ "AMI": "ami-fbfa41e6" }
		},
		"AWSLinuxAMIs": {
			"us-east-1": 		{ "AMI": "ami-1ecae776" },
			"us-west-2": 		{ "AMI": "ami-e7527ed7" },
			"us-west-1": 		{ "AMI": "ami-d114f295" },
			"eu-west-1": 		{ "AMI": "ami-a10897d6" },
			"eu-central-1": 	{ "AMI": "ami-a8221fb5" },
			"ap-northeast-1": 	{ "AMI": "ami-cbf90ecb" },
			"ap-southeast-1": 	{ "AMI": "ami-68d8e93a" },
			"ap-southeast-2": 	{ "AMI": "ami-fd9cecc7" },
			"sa-east-1": 		{ "AMI": "ami-b52890a8" }
		}
	},
```

When composing mappings, we name the mapping as a key of the **Mappings** object. The value of each **Mapping** object is a key-value hash, where the key is some variable piece of data you will use for lookup later - in this case, the **Region** for each AMI. The values of these keys are more key-value hashes, which contain one or more properties you want to use later in the template. In this case, we only need to look up the `AMI` value for each **Regoion**, so we only include this key on each sub-object. The actual `AMI` values are ones I collected for you using the EC2 AMI Marketplace search function. All of these AMIs are free to use.

Beyond defining the **Mappings** above, we need to use them within the template, namely, within the `InstanceConfig` object when defining the AMI to use, and the `NATInstance` - which will use the `AWSLinuxAMIs` and the `AWSNATAMIs` objects, respectively. The function we use is `Fn::FindInMap`, and it's usage is as such:

```javascript
"Fn::FindInMap": [
	"MAPPING_NAME",      // one of our two mappings
	"VARIABLE_DATA_KEY", // { "Ref": "AWS::Region" } in this case
	"SUBKEY_VALUE"       // "AMI" in this case
]
```

Can you figure out how to do this for the two places we need to use it, `NATInstance` and `InstancesConfig`? You can find the places to use it by searching for `?????`, since that is what you used as a placeholder in previous steps. The answers are below...:

```javascript
// ... We adjust template.Resources.NATInstance.Properties.ImageId
"ImageId": {
	"Fn::FindInMap": [
		"AWSNATAMIs",
		{ "Ref": "AWS::Region" },
		"AMI"
	]
},
// ... ELSEWHERE IN TEMPLATE...
// ... We adjust template.Resources.InstancesConfig.Properties.ImageId
"ImageId": {
	"Fn::FindInMap": [
		"AWSLinuxAMIs",
		{ "Ref": "AWS::Region" },
		"AMI"
	]
},
```

That's all we need to do to make this system work in mutiple Regions - next up, we need to enable usage of the template for multiple environments in the same Region. We want to do this to enable, say, Dev, Test, Staging, and Prod environments.

Because the Stack Template creates a new VPC when run, and VPC CIDR blocks cannot intersect, we should make a simple way to change the range of addresses the VPC covers. We can accomplish this by changing the Class B octect the CIDR blocks use. The Class B octet is the second group of numbers separated by `.`'s in a CIDR block. 

To support this, alter the value of the CIDR blocks for `VPC`, `PublicSubnet`, and `PrivateSubnet` to use a numeric **Parameter**. We will call this **Parameter** `VPCClassBOctet`, to match what we are trying to accomplish by using it.

Can you figure out how to make the Parameter and modify the CIDR blocks in these three Resources on your own?

```javascript
// ... inside "Parameters"
		"VPCClassBOctet": {
			"Type": "Number",
			"Description": "The Class B block to use for the VPC (0-255).",
			"MaxValue": 255,
			"MinValue": 0,
			"Default": 0
		},
// ...
```

```javascript
// ... inside "Resources"
		"VPC": {
			"Type": "AWS::EC2::VPC",
			"Properties": {
				"CidrBlock": { "Fn::Join": [ "", [
					"10.",
					{ "Ref": "VPCClassBOctet" },
					".0.0/16"
				]]}
			}
		},
		"PublicSubnet": {
			"DependsOn": [
				"VPC"
			],
			"Type": "AWS::EC2::Subnet",
			"Properties": {
				"AvailabilityZone": { "Ref": "VPCAZ" },
				"CidrBlock": { "Fn::Join": [ "", [
					"10.",
					{ "Ref": "VPCClassBOctet" },
					".0.0/24"
				]]},
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
				"CidrBlock": { "Fn::Join": [ "", [
					"10.",
					{ "Ref": "VPCClassBOctet" },
					".1.0/24"
				]]},
				"VpcId": { "Ref": "VPC" }
			}
		},
// ...
```

Alright! Our Stack Template is now complete, and meets all of the requirements set forth by the business!

Check your work against the template below, or just save the below template to disk, and prepare to use it in the **Next Step**, during which we will launch this Stack!

## [NEXT](../006-test/README.md)

```
{
	"Description": "My Cloud Academy Labs 2-Tier Scalable API Template",
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
		"TableThroughput": {
			"Type": "Number",
			"Description": "The number of queries per second the table should support.",
			"Default": 10
		},
		"VPCClassBOctet": {
			"Type": "Number",
			"Description": "The Class B block to use for the VPC (0-255).",
			"MaxValue": 255,
			"MinValue": 0,
			"Default": 0
		},
		"RunNumber": {
			"Type": "Number",
			"Description": "Used to force the ASG's nodes' UserData to update.",
			"Default": 1
		}
	},
	"Mappings": {
		"AWSNATAMIs": {
			"us-east-1": 		{ "AMI": "ami-303b1458" },
			"us-west-2": 		{ "AMI": "ami-69ae8259" },
			"us-west-1": 		{ "AMI": "ami-7da94839" },
			"eu-west-1": 		{ "AMI": "ami-6975eb1e" },
			"eu-central-1": 	{ "AMI": "ami-46073a5b" },
			"ap-southeast-1": 	{ "AMI": "ami-b49dace6" },
			"ap-northeast-1": 	{ "AMI": "ami-03cf3903" },
			"ap-southeast-2": 	{ "AMI": "ami-e7ee9edd" },
			"sa-east-1": 		{ "AMI": "ami-fbfa41e6" }
		},
		"AWSLinuxAMIs": {
			"us-east-1": 		{ "AMI": "ami-1ecae776" },
			"us-west-2": 		{ "AMI": "ami-e7527ed7" },
			"us-west-1": 		{ "AMI": "ami-d114f295" },
			"eu-west-1": 		{ "AMI": "ami-a10897d6" },
			"eu-central-1": 	{ "AMI": "ami-a8221fb5" },
			"ap-northeast-1": 	{ "AMI": "ami-cbf90ecb" },
			"ap-southeast-1": 	{ "AMI": "ami-68d8e93a" },
			"ap-southeast-2": 	{ "AMI": "ami-fd9cecc7" },
			"sa-east-1": 		{ "AMI": "ami-b52890a8" }
		}
	},
	"Resources": {
		"VPC": {
			"Type": "AWS::EC2::VPC",
			"Properties": {
				"CidrBlock": { "Fn::Join": [ "", [
					"10.",
					{ "Ref": "VPCClassBOctet" },
					".0.0/16"
				]]}
			}
		},
		"PublicSubnet": {
			"DependsOn": [
				"VPC"
			],
			"Type": "AWS::EC2::Subnet",
			"Properties": {
				"AvailabilityZone": { "Ref": "VPCAZ" },
				"CidrBlock": { "Fn::Join": [ "", [
					"10.",
					{ "Ref": "VPCClassBOctet" },
					".0.0/24"
				]]},
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
				"CidrBlock": { "Fn::Join": [ "", [
					"10.",
					{ "Ref": "VPCClassBOctet" },
					".1.0/24"
				]]},
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
				"ImageId": {
					"Fn::FindInMap": [
						"AWSNATAMIs",
						{ "Ref": "AWS::Region" },
						"AMI"
					]
				},
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
				"PrivateSubnet",
				"RouteToNat"
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
					"PauseTime": "PT1M01S",
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
				"ImageId": {
					"Fn::FindInMap": [
						"AWSLinuxAMIs",
						{ "Ref": "AWS::Region" },
						"AMI"
					]
				},
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
				},
				{
					"FromPort": 22,
					"IpProtocol": "TCP",
					"CidrIp": "0.0.0.0/0",
					"ToPort": 22
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
				}, {
					"FromPort": 22,
					"IpProtocol": "TCP",
					"CidrIp": "0.0.0.0/0",
					"ToPort": 22
				}],
				"VpcId": { "Ref": "VPC" }
			}
		}
	},
	"Outputs": {
		"ELBDNS": {
			"Description": "The DNS for the ELB / our app.",
			"Value": {
				"Fn::Join": [
					"",
					[
						"http://",
						{
							"Fn::GetAtt": [
								"LoadBalancer",
								"DNSName"
							]
						},
						"/index.html"
					]
				]
				
			}
		}
	}
}
```


## All Done, Yay!

