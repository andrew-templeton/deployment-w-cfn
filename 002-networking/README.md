
## Authoring Networking Resource Automation

Now that we have a clear plan conceptually for the **Resources** we need to define in our **CloudFormation Template**, we need to begin filling the **Resources** out in the correct format for actual use. Each **CloudFormation Resource Type** has a distinct format...:

 - **Type**, a `String` which tells CloudFormation which kind of Resource to make (always required).
 - **Properties**, a JSON-formatted object which defines the way to build the Resource (almost always required). This varies by Resource **Type**.
 - **DependsOn**, an `Array` of `String`s (`Array<String>`), where the `String` elements are names of the other resources of the Template to create before the resource. The name(s) are one or more of the keys of the objects in the **Resource** object.
 - There are a couple, much less common fields, which we will address per-Resource-Type when they come up.

The AWS documentation for CloudFormation development contains a Template Reference: Resource Types section, which we will use to look up the names of **Type** fields for our Resources and the format for the **Properties** objects.

While you work on this Lab, you need to have the [Template Reference: Resource Types](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html) open in another tab or window. Please open the link in the previous sentence, then proceed.

While working through this Lab, you may want to try finding the correct Resource Types on your own, by using the above link, or, just follow along - I will choose the correct Resource Types for our planned Resources and paste the format each follows each step of the way.

Alright - first, we will fill out the Resource definitions for the planned `VPC`, `PublicSubnet`, and `PrivateSubnet`, simply by pasting the sample snippets from the Resource Types documentation pages - after I do this below, we will then fill out the correct values for each field, and remove Optional properties we do not need

Here's the first three Resource Types filled out: 

```javascript
// ... inside "Resources"
		"VPC": {
			"Type": "AWS::EC2::VPC",
			"Properties": {
				"CidrBlock": String,
				"EnableDnsSupport": Boolean,
				"EnableDnsHostnames": Boolean,
				"InstanceTenancy": String,
				"Tags": [ Resource Tag, ... ]
			}
		},
		"PublicSubnet": {
			"Type": "AWS::EC2::Subnet",
			"Properties": {
				"AvailabilityZone": String,
				"CidrBlock": String,
				"MapPublicIpOnLaunch": Boolean,
				"Tags": [ Resource Tag, ... ],
				"VpcId": { "Ref": String }
			}
		},
		"PrivateSubnet": {
			"Type": "AWS::EC2::Subnet",
			"Properties": {
				"AvailabilityZone": String,
				"CidrBlock": String,
				"MapPublicIpOnLaunch": Boolean,
				"Tags": [ Resource Tag, ... ],
				"VpcId": { "Ref": String }
			}
		},
// ...
```

 - The `{ "Ref": "?????" } ` syntax tells CloudFormation to either look up the ID or Name of another Resource in the Resources block, or the Value of a Parameter from the Parameters block - thus, for the `VpcId` fields above, we will use the value `{ "Ref": "VPC" }`.
 - The `PublicSubnet` should `MapPublicIpOnLaunch`, to enable Internet communication.
 - We can just set convenient values for `CidrBlock` values, like the VPC wizard does in the Console.
 - All values besides `CidrBlock` on the `VPC` are fine as default.
 - `AvailabilityZone` values on the `Subnet` Resources requires a Parameter, since this value needs to change across Regions.
 - Both `Subnet` resources must `DependsOn` the `VPC`.


 ```javascript
// ... inside "Parameters"

		"VPCAZ": {
			"Type": "AWS::EC2::AvailabilityZone::Name",
			"Description": "The Availability Zone to use for the simple VPC."
		}

// ... inside "Resources"
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
// ...
```

Now, let's fill out the `GatewayToInternet` and the `NATInstance`...:

```javascript
// ... inside "Resources"
		"GatewayToInternet": {
			"Type": "AWS::EC2::InternetGateway",
			"Properties": {
				"Tags": [ Resource Tag, ... ]
			}
		},
		"NATInstance": {
			"Type": "AWS::EC2::Instance",
			"Properties": {
				"AvailabilityZone": String,
				"BlockDeviceMappings": [ EC2 Block Device Mapping, ... ],
				"DisableApiTermination": Boolean,
				"EbsOptimized": Boolean,
				"IamInstanceProfile": String,
				"ImageId": String,
				"InstanceInitiatedShutdownBehavior": String,
				"InstanceType": String,
				"KernelId": String,
				"KeyName": String,
				"Monitoring": Boolean,
				"NetworkInterfaces": [ EC2 Network Interface, ... ],
				"PlacementGroupName": String,
				"PrivateIpAddress": String,
				"RamdiskId": String,
				"SecurityGroupIds": [ String, ... ],
				"SecurityGroups": [ String, ... ],
				"SourceDestCheck": Boolean,
				"SsmAssociations": [ SSMAssociation, ... ]
				"SubnetId": String,
				"Tags": [ Resource Tag, ... ],
				"Tenancy": String,
				"UserData": String,
				"Volumes": [ EC2 MountPoint, ... ],
				"AdditionalInfo": String
			}
		},
// ...
```

 - We will skip `Tags` on all resources in the Lab, to save space and time. In production deployments, these are extremely helpful for Billing and Cost Management.
 - `AvailabilityZone` should be `{ "Ref": "VPCAZ" }`
 - `ImageId` should be an EC2 AMI for a NAT. We will define this using a **Mapping** combined with the `{ "Ref": "AWS::Region" }`, a built-in **Pseudo-Parameter**, later, so for now, enter `"???????"`. You could also search for one in the Console.
 - Set `InstanceType` to `"m3.medium"`, this is large enough.
 - `SecurityGroupIds` should contain `{ "Ref": "InstancesToNATSecurityGroup" }`, to allow this traffic.
 - `SourceDestCheck` must be `false`, to allow the `NATInstance` to forward traffic.
 - `SubnetId` should be `{ "Ref": "PublicSubnet" }`, so the `NATInstance` can access the Internet.
 - Make `NATInstance` have `"DependsOn": ["GatewayAttachmentToVPC", "InstancesToNATSecurityGroup", "PublicSubnet"]` - the Security Group and Subnet are obvious, the `"GatewayAttachmentToVPC"` less so - the `NATInstance` **must** be connected to the Internet to work, so it `DependsOn` this attachment.
 - All other `NATInstance` values are optional / will be left as default.

```javascript
// ... inside "Resources"
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
				"ImageId": "???????",
				"InstanceType": "m3.medium",
				"SecurityGroupIds": [
					{ "Ref": "InstancesToNATSecurityGroup" }
				],
				"SourceDestCheck": false,
				"SubnetId": { "Ref": "PublicSubnet" }
			}
		},
// ...
```

Great! Let's keep on filling out Resources, continuing with: `RoutesForPublicSubnet`, `RoutesForPrivateSubnet`, `GenericNACL`, and `GatewayAttachmentToVPC`...:


```javascript
// ... inside "Resources"
		"RoutesForPublicSubnet": {
			"Type": "AWS::EC2::RouteTable",
			"Properties": {
				"VpcId": String,
				"Tags": [ Resource Tag, ... ]
			}
		},
		"RoutesForPrivateSubnet": {
			"Type": "AWS::EC2::RouteTable",
			"Properties": {
				"VpcId": String,
				"Tags": [ Resource Tag, ... ]
			}
		},
		"GenericNACL": {
			"Type": "AWS::EC2::NetworkAcl",
			"Properties": {
				"Tags": [ Resource Tag, ... ],
				"VpcId": String
			}
		},

		"GatewayAttachmentToVPC": {
			"Type": "AWS::EC2::VPCGatewayAttachment",
			"Properties": {
				"InternetGatewayId": String,
				"VpcId": String,
				"VpnGatewayId": String
			}
		},
// ...
```

 - Discard all `Tags`
 - You should fill `VpcId` out based on what you learned from earlier resources, and also know how to refer to the `GatewayToInternet` Resource for the `InternetGatewayId` field.
 - Discard `VpnGatewayId`, we do not use this unless we are making a VPN.
 - The `DependsOn` values should be clear - anything we are using `Ref` to in a Resource needs to be included in `DependsOn`. From here forward in the Lab, I will not point this out - you should try to get it right each time!


```javascript
// ... inside "Resources"
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
// ...
```

Next, we find the appropriate types and copy the formats in for `RouteToGateway`, `RouteToNat`, `NACLInboundEntry`, and `NACLOutboundEntry`...:


```javascript
// ... inside "Resources"
		"RouteToGateway": {
			"Type": "AWS::EC2::Route",
			"Properties": {
				"DestinationCidrBlock": String,
				"GatewayId": String,
				"InstanceId": String,
				"NetworkInterfaceId": String,
				"RouteTableId": String,
				"VpcPeeringConnectionId": String
			}
		},
		"RouteToNat": {
			"Type": "AWS::EC2::Route",
			"Properties": {
				"DestinationCidrBlock": String,
				"GatewayId": String,
				"InstanceId": String,
				"NetworkInterfaceId": String,
				"RouteTableId": String,
				"VpcPeeringConnectionId": String
			}
		},
		"NACLInboundEntry": {
			"Type": "AWS::EC2::NetworkAclEntry",
			"Properties": {
				"CidrBlock": String,
				"Egress": Boolean,
				"Icmp": EC2 ICMP,
				"NetworkAclId": String,
				"PortRange": EC2 PortRange,
				"Protocol": Integer,
				"RuleAction": String,
				"RuleNumber": Integer
			}
		},
		"NACLOutboundEntry": {
			"Type": "AWS::EC2::NetworkAclEntry",
			"Properties": {
				"CidrBlock": String,
				"Egress": Boolean,
				"Icmp": EC2 ICMP,
				"NetworkAclId": String,
				"PortRange": EC2 PortRange,
				"Protocol": Integer,
				"RuleAction": String,
				"RuleNumber": Integer
			}
		},
// ...
```

I'm going to be giving less hints from here on out! When we see values like `EC2 PortRange`, we can find the right formats for these Sub-Properties within the documentation for their respective fields on the pages describing each Resource Type. With this in mind, here are some hints for these four resources - try to figure the rest our yourself!

 - The `RouteToGateway` will need `GatewayAttachmentToVPC` as a `DependsOn` element, since the Route will not work without being connected to the Internet.
 - The `CidrBlock`-type resources will need to be `0.0.0.0/0`, to match all routes and destinations. The `VPC` includes implicit routes for `local` addresses, which will take precedence over this wildcard CIDR range, so we can safely use this value without breaking communication within the `VPC`.
 - The top and bottom ports to select all possible ports are `"0"` and `"65535"`
 - `Protocol` and `RuleNumber` will require some brief digging in the documentation for those fields to get correct.


Did you figure it out? Compare your four resources to the right settings below...:


```javascript
// ... inside "Resources"
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
// ...
```


How'd you do?? Only one more set of four Resources to go before we finish with the networking Resources of this Stack Template: `NACLBindingForPublicSubnet`, `NACLBindingForPrivateSubnet`, `RoutesBindingForPublicSubnet`, and `RoutesBindingForPrivateSubnet`. All of these Resources are short...

```javascript
// ... inside "Resources"
		"NACLBindingForPublicSubnet": {
			"Type": "AWS::EC2::SubnetNetworkAclAssociation",
			"Properties": {
				"SubnetId": String ,
				"NetworkAclId": String
			}
		},
		"NACLBindingForPrivateSubnet": {
			"Type": "AWS::EC2::SubnetNetworkAclAssociation",
			"Properties": {
				"SubnetId": String ,
				"NetworkAclId": String
			}
		},
		"RoutesBindingForPublicSubnet": {
			"Type": "AWS::EC2::SubnetRouteTableAssociation",
			"Properties": {
				"RouteTableId": String,
				"SubnetId": String,
			}
		},
		"RoutesBindingForPrivateSubnet": {
			"Type": "AWS::EC2::SubnetRouteTableAssociation",
			"Properties": {
				"RouteTableId": String,
				"SubnetId": String,
			}
		},
// ...
```

If you have been paying close attention to how to use `DependsOn` and `Ref`, you should be able to fill this section out yourself...:


```javascript
// ... inside "Resources"
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
// ...
```


WHEW! The networking portion of our stack is done for now! The networking portion is the longest and most difficult part - good job so far! You're already written significantly more AWS CloudFormation Stack Template automation than most DevOps Engineers ever do.

You can proceed to the Lab's **Next Step** now. The entire networking portion of the Stack Template is all together below - feel free to compare what you have built so far to the snippet below.


```javascript
{
	"Description": "My Cloud Academy Labs 2-Tier Scalable API Template",
	"Parameters": {
		"VPCAZ": {
			"Type": "AWS::EC2::AvailabilityZone::Name",
			"Description": "The Availability Zone to use for the simple VPC."
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


		"LoadBalancer": {},
		"InstancesGroup": {},
		"InstancesConfig": {},
		"DynamoTableForTodos": {},

		"DynamoAccessRole": {},
		"DynamoAccessInstanceProfile": {},

		"LBToInstancesSecurityGroup": {},
		"InternetToLBSecurityGroup": {},
		"InstancesToNATSecurityGroup": {}
	}
}
```


## [NEXT](../003-compute/README.md)
