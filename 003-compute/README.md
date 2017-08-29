
## Authoring Compute & Database Resource Automation

Take a deep breath - you're well on your way to making a highly automated distributed system, using the most advanced tool on AWS!

Next, we are going to build out the four EC2 and DynamoDB-related Resources. The first one we will fill out is the `LoadBalancer` Resource...:

```javascript
// ... inside "Resources"
		"LoadBalancer": {
			"Type": "AWS::ElasticLoadBalancing::LoadBalancer",
			"Properties": {
				"AccessLoggingPolicy": AccessLoggingPolicy,
				"AppCookieStickinessPolicy": [ AppCookieStickinessPolicy, ... ],
				"AvailabilityZones": [ String, ... ],
				"ConnectionDrainingPolicy": ConnectionDrainingPolicy,
				"ConnectionSettings": ConnectionSettings,
				"CrossZone": Boolean,
				"HealthCheck": HealthCheck,
				"Instances": [ String, ... ],
				"LBCookieStickinessPolicy": [ LBCookieStickinessPolicy, ... ],
				"LoadBalancerName": String,
				"Listeners": [ Listener, ... ],
				"Policies": [ ElasticLoadBalancing Policy, ... ],
				"Scheme": String,
				"SecurityGroups": [ Security Group, ... ],
				"Subnets": [ String, ... ],
				"Tags": [ Resource Tag, ... ]
			}
		},
// ...
```

Hints for filling the `LoadBalancer` out:

 - The application instances will listen on port `8888` for `"HTTP"`-protocol traffic, and web browsers send HTTP traffic to the `LoadBalancer` via port `80`.
 - This will be an `"internet-facing"` ELB.
 - The `InternetToLBSecurityGroup` is the one we will add to the `LoadBalancer` to allow traffic from the web via port `80`.
 - You only need to fill out these `Properties`: `Listeners`, `Scheme`, `SecurityGroups`, and `Subnets`.

Can you figure it out? Check your work...:

```javascript
// ... inside "Resources"
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
// ...
```

That's a tough one, because of the compound `Listeners` Property. Next, let's fill out the `InstancesGroup` Resource. We will also include a new piece on this Resource: `UpdatePolicy`. The `UpdatePolicy` is used on Autoscaling Groups to define how changes should be rolled out across the EC2 Instances that make up the Group, using the `UpdatePolicy.AutoScalingRollingUpdate` sub-resource. Let's take a look at the available fields and fill them out...:

```javascript
// ... inside "Resources"
		"InstancesGroup": {
			"Type": "AWS::AutoScaling::AutoScalingGroup",
			"Properties": {
				"AvailabilityZones": [ String, ... ],
				"Cooldown": String,
				"DesiredCapacity": String,
				"HealthCheckGracePeriod": Integer,
				"HealthCheckType": String,
				"InstanceId": String,
				"LaunchConfigurationName": String,
				"LoadBalancerNames": [ String, ... ],
				"MaxSize": String,
				"MetricsCollection": [ MetricsCollection, ... ],
				"MinSize": String,
				"NotificationConfigurations": [ NotificationConfigurations, ... ],
				"PlacementGroup": String,
				"Tags": [ Auto Scaling Tag, ..., ],
				"TerminationPolicies": [ String, ..., ],
				"VPCZoneIdentifier": [ String, ... ]
			},
			"UpdatePolicy": {
				"AutoScalingRollingUpdate": {
					"MaxBatchSize": Integer,
					"MinInstancesInService": Integer,
					"MinSuccessfulInstancesPercent": Integer,
					"PauseTime": String,
					"SuspendProcesses": [ List of processes ],
					"WaitOnResourceSignals": Boolean
				}
			}
		},
// ...
```

There's a lot going on here, so I'll give more hints than I have been in the previous few iterations.

 - We do **not** fill out the `AvailabilityZones` property, because in our `VPC`, we have a zone assigned to the Subnets.
 - The `Cooldown` tells the Group how long to wait, in seconds, between performing scaling actions. We will just use `60` to assign one minute. We might think about this more in a production deployment, taking into account how quickly our real load scales up and down.
 - We are going to need to tell the Stacks we launch with this Template how many instances we want. We will need to make a numeric **Parameter**, which we will call `NumberOfNodes`.
 - Since for this Lab we will used a static Instance Count, the `NumberOfNodes` Parameter will apply to all of these Properties:
   + `DesiredCapacity`
   + `MaxSize`
   + `MinSize`
 - `HealthCheckGracePeriod` should be `30`, to allow time before beginning the Health Check, to download and install our code to the EC2 instances.
 - Use `"EC2"` for `HealthCheckType`.
 - Use `"MetricsCollection": [{ "Granularity": "1Minute" }],`.
 - The `VPCZoneIdentifier` is poorly named... It corresponds to the Subnet(s) in which to launch the instances. 

 Check your work below - this one is tough.

```javascript
// ... inside "Parameters"
		"NumberOfNodes": {
			"Type": "Number",
			"Description": "The number of EC2 nodes the service should have.",
			"Default": 2
		},
// ...
```

 ```javascript
// ... inside "Resources"
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
// ...
```

After defining the `InstancesGroup`, we need to define the `InstancesConfig`, which corresponds to an AWS EC2 Autoscaling Launch Configuration we might configure in the AWS Console...:

```javascript
// ... inside "Resources"
		"InstancesConfig": {
			"Type": "AWS::AutoScaling::LaunchConfiguration",
			"Properties": {
				"AssociatePublicIpAddress": Boolean,
				"BlockDeviceMappings": [ BlockDeviceMapping, ... ],
				"ClassicLinkVPCId": String,
				"ClassicLinkVPCSecurityGroups": [ String, ... ],
				"EbsOptimized": Boolean,
				"IamInstanceProfile": String,
				"ImageId": String,
				"InstanceId": String,
				"InstanceMonitoring": Boolean,
				"InstanceType": String,
				"KernelId": String,
				"KeyName": String,
				"PlacementTenancy": String,
				"RamDiskId": String,
				"SecurityGroups": [ SecurityGroup, ... ],
				"SpotPrice": String,
				"UserData": String
			}
		},
// ...
```

 - Our instances should not have public IPs - they will be in `PrivateSubnet`, as we just defined in `InstancesGroup`.
 - Everything up to `IamInstanceProfile` is optional and unused.
 - You can figure `IamInstanceProfile` out.
 - Use `"??????"` for `ImageId`, as we will be using a **Mapping** in a later step to pick the correct Amazon Linux AMI for each Region the Template is used in.
 - We do want the Instances to be monitored.
 - Can you create a `String` **Parameter** named `NodeSize`, and use this value for the `InstanceType` setting here?
 - The `LBToInstancesSecurityGroup` will be the Group which allows access to the instances from the `LoadBalancer` ELB we already configured.
 - `UserData` is a script that uses the built-in `Fn::Base64` encoding function CloudFormation provides. The script runs on instance launch. This one depends on the application code, so you should just copy it from the correct answer below. Generally speaking, it installes Git, Node.js, and NPM, then launches the application, providing the DynamoDB Table name and the right port to listen on (8888).
 - UserData also includes a `RunNumber` **Parameter**, which we should also add in the `Parameters` block. This value is used to force an update on the Auto-Scaling Group's Instances if we need to update the code by re-downloading from GitHub. It is able to force this update by requiring a change within the `UserData` script, which forces the `InstancesConfig` to refresh and apply the `UpdatePolicy` we configured above.

```javascript
// ... inside "Parameters"
		"NodeSize": {
			"Type": "String",
			"Description": "The type of node to use for your system",
			"Default": "m3.medium"
		},
		"RunNumber": {
			"Type": "Number",
			"Description": "Used to force the ASG's nodes' UserData to update.",
			"Default": 1
		}
// ...
```

 ```javascript
// ... inside "Resources"
		"InstancesConfig": {
			"DependsOn": [
				"DynamoAccessInstanceProfile",
				"LBToInstancesSecurityGroup"
			],
			"Type": "AWS::AutoScaling::LaunchConfiguration",
			"Properties": {
				"AssociatePublicIpAddress": false,
				"IamInstanceProfile": { "Ref": "DynamoAccessInstanceProfile" },
				"ImageId": "ami-f0091d91",
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
// ...
```


For our last Resource in this step, we need to define a DynamoDB table, with a `HashKey` of `id` - this is what the application requires. Since there are application requirements, jsut copy and paste this value into your template. If you would like to learn more about how to create DynamoDB Tables, reference the documentation for `AWS::DynamoDB::Table` type Resources in the CloudFormation documentation, or refer to the DynamoDB Table API documentation in AWS.

To allow for configurable DynamoDB Table throughput performance, this snippet also makes use of yet another numeric parameter, `TableThroughput`, the definition for which is provided below: 

```javascript
// ... inside "Parameters"
		"TableThroughput": {
			"Type": "Number",
			"Description": "The number of queries per second the table should support.",
			"Default": 10
		}
// ...
```

```javascript
// ... inside "Resources"
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
// ...
```

Good job! Your template now has nearly complete networking, compute, and database configurations. The last section of Resources we need to define is the access section, which defines the IAM Resources and Security Groups for this application. The access step is a lot shorter, so move on to the **Next Step** now!

If you would like to check your work thus far, refer to the complete work-in-progress template snippet below, which contains all you should have written thus far.

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

		"DynamoAccessRole": {},
		"DynamoAccessInstanceProfile": {},

		"LBToInstancesSecurityGroup": {},
		"InternetToLBSecurityGroup": {},
		"InstancesToNATSecurityGroup": {}
	}
}
```

## [NEXT](../004-access/README.md)
