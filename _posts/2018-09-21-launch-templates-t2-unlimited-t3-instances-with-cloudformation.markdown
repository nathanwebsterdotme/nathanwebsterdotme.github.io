---
title:  "Launch Templates, T2 Unlimited and T3 Instances with CloudFormation"
date:   2018-09-21 14:37:00
categories: [devops]
tags: [cloudformation, ec2]
---

AWS Launch Templates were [announced fairly quietly](https://aws.amazon.com/about-aws/whats-new/2017/11/introducing-launch-templates-for-amazon-ec2-instances/) in late November 2017 (I know I missed the announcement!) but it seems that Launch Templates are going to be quite a key piece to the EC2 puzzle going forwards, possibly replacing Launch Configurations completely.  A reason I think this is that (at the time of writing) the new configuration options for T3 instances (CPU Credit Options) are only supported in Launch Templates and not Launch Configurations.

I've recently completed a small piece of work for a client to replace all of the T2 instances they were running with T3 instances, which meant having to use Launch Templates with their AutoScalingGroups.  This blog post will show how to do this in CloudFormation, as this is what my client is using.

We'll cover a simple CloudFormation template that deploys a simple ASG of EC2 instances, first with Launch Configurations then with Launch Templates.  (Obviously you would parameterize a lot of the hard-coded variables in a real scenario!)


ASG with Launch Configuration:
JSON:

```json
{
	"Description": "A simple ASG + Launch Configuration Example",
	"AWSTemplateFormatVersion": "2010-09-09",
	"Resources":{
		"LaunchConfiguration": {
			"Type": "AWS::AutoScaling::LaunchConfiguration",
			"Properties": {
				"ImageId": "ami-1234567890",
				"InstanceType": "t2.small",
				"KeyName": "test_key",
				"SecurityGroups": [ { "sg-1234567890" } ]
			}
		},
		"AutoScalingGroup": {
			"Type": "AWS::AutoScaling::AutoScalingGroup",
			"DependsOn": "LaunchConfiguration",
			"Properties": {
				"MinSize": 1,
				"MaxSize": 3,
				"DesiredCapacity": 1,
				"LaunchConfigurationName": { "Ref": "LaunchConfiguration" },
				"VPCZoneIdentifier": "subnet-1234567890"
			}
		}
	}
}
```

YAML:

```yaml
---
Description: "A simple ASG + Launch Configuration Example"
AWSTemplateFormatVersion: "2010-09-09"
Resources:
  LaunchConfiguration:
    Type: "AWS::AutoScaling::LaunchConfiguration"
    Properties:
      ImageId: "ami-123456789"
      InstanceType: "t2.small"
      KeyName: "test_key"
      SecurityGroups:
        - "sg-1234567890"
  AutoScalingGroup:
    Type: "AWS::AutoScaling::AutoScalingGroup"
    DependsOn: "LaunchConfiguration"
    Properties:
      MinSize: "1"
      MaxSize: "3"
      DesiredCapacity: "1"
      LaunchConfigurationName:
        - Ref: "LaunchConfiguration"
      VPCZoneIdentifier:
        - "subnet-1234567890"
```

Below are examples of a like-for-like switch of the above code, replacing the `LaunchConfiguration` resource with a `LaunchTemplate`:

JSON:

```json
{
	"Description": "A simple ASG + Launch Template Example",
	"AWSTemplateFormatVersion": "2010-09-09",
	"Resources":{
		"LaunchTemplate": {
			"Type": "AWS::EC2::LaunchTemplate",
			"Properties": {
				"LaunchTemplateName": "example_name",
				"LaunchTemplateData": {
					"ImageId": "ami-1234567890",
					"InstanceType": "t2.small",
					"KeyName": "test_key",
					"SecurityGroups": [ { "sg-1234567890" } ]
				}
			}
		},
		"AutoScalingGroup": {
			"Type": "AWS::AutoScaling::AutoScalingGroup",
			"DependsOn": "LaunchTemplate",
			"Properties": {
				"MinSize": 1,
				"MaxSize": 3,
				"DesiredCapacity": 1,
				"LaunchTemplate": {
					"LaunchTemplateId": { "Ref": "LaunchTemplate" },
					"Version": { "Fn::GetAtt" : [ "LaunchTemplate", "LatestVersionNumber" ] }
				},
				"VPCZoneIdentifier": "subnet-1234567890"
			}
		}
	}
}
```

YAML:

```yaml
---
Description: "A simple ASG + Launch Configuration Example"
AWSTemplateFormatVersion: "2010-09-09"
Resources:
  LaunchTemplate:
    Type: "AWS::EC2::LaunchTemplate"
    Properties:
      LaunchTemplateName: "example_name"
      LaunchTemplateData:
        ImageId: "ami-123456789"
        InstanceType: "t2.small"
        KeyName: "test_key"
        SecurityGroups:
          - "sg-1234567890"
  AutoScalingGroup:
    Type: "AWS::AutoScaling::AutoScalingGroup"
    DependsOn: "LaunchTemplate"
    Properties:
      MinSize: "1"
      MaxSize: "3"
      DesiredCapacity: "1"
      LaunchTemplate:
        LaunchTemplateId:
          Ref: "LaunchTemplate"
        Version:
          Fn::GetAtt:
            [ "LaunchTemplate", "LatestVersionNumber" ]
      VPCZoneIdentifier:
        - "subnet-1234567890"
```

The LaunchTemplate resource is slightly different to the LaunchConfiguration one, wrapping up all of the configuration in a `LaunchTemplateData` sub-object.  We can also provide an *optional* Launch Template name, which has to be unique, or we can omit this and let CloudFormation name the resource for us.

Another difference is how we use the Launch Template in the AutoScalingGroup resource. Not only do we have to provide either the `LaunchTemplateName` or `LaunchTemplateId` but also a `Version`.  This is an additional feature of Launch Templates over Launch Configurations - you can now create many different versions of the same Launch Template, and also define a 'default' version if you so wish.
In our CloudFormation template above, I've set the latest version of the Launch Template to be deployed by the ASG by using the Launch Template return value `LatestVersionNumber`.

All good so far...Now to enable support for T3 instances!

On the same day that Launch Templates were announced, a [new CPU mode was announced for burstable T2 instances](https://aws.amazon.com/blogs/aws/new-t2-unlimited-going-beyond-the-burst-with-high-performance/).  This is called "Unlimted" mode, and is optional for T2 instances, and the only mode supported by T3 instances.  With our new Launch Templates, we can enable unlimited mode quite easily by adding the following to the `LaunchTemplateData`.

```json
{
	"LaunchTemplateData": {
		"CreditSpecification": {"CpuCredits": "unlimited" }
	}
}
```

```yaml
  LaunchTemplateData:
    CreditSpecification:
      CpuCredits: "unlimited"

```

A good way for you to introduce this functionality in to your template whilst maintaining backwards compatibility is to parameterize the `CpuCredits` value, setting a default value to ``"standard"``.  That way, existing templates will continue to work as normal, and the value can be overwritten when you want to use `"unlimited"` mode with T2 instances or are deploying T3 instances.  Our templates now looks as follows:

```json
{
	"Description": "A simple ASG + Launch Template Example",
	"AWSTemplateFormatVersion": "2010-09-09",
	"Parameters":{
		"CpuCredits": {
			"Description": "CpuCredit option for T2/T3 instances",
			"Type": "String",
			"AllowedValues": ["standard", "unlimited"],
			"Default": "standard"
		}
	},
	"Resources":{
		"LaunchTemplate": {
			"Type": "AWS::EC2::LaunchTemplate",
			"Properties": {
				"LaunchTemplateName": "example_name",
				"LaunchTemplateData": {
					"CreditSpecification": {
						"CpuCredits": { "Ref": "CpuCredits" }
					},
					"ImageId": "ami-1234567890",
					"InstanceType": "t2.small",
					"KeyName": "test_key",
					"SecurityGroups": [ { "sg-1234567890" } ]
				}
			}
		},
		"AutoScalingGroup": {
			"Type": "AWS::AutoScaling::AutoScalingGroup",
			"DependsOn": "LaunchTemplate",
			"Properties": {
				"MinSize": 1,
				"MaxSize": 3,
				"DesiredCapacity": 1,
				"LaunchTemplate": {
					"LaunchTemplateId": { "Ref": "LaunchTemplate" },
					"Version": { "Fn::GetAtt" : [ "LaunchTemplate", "LatestVersionNumber" ] }
				},
				"VPCZoneIdentifier": "subnet-1234567890"
			}
		}
	}
}
```

YAML:

```yaml
---
Description: "A simple ASG + Launch Configuration Example"
AWSTemplateFormatVersion: "2010-09-09"
Parameters:
  CpuCredits:
    Description: "CpuCredit option for T2/T3 instances"
    Type: String
    AllowedValues:
      - "standard"
      - "unlimited"
    Default: "standard"
Resources:
  LaunchTemplate:
    Type: "AWS::EC2::LaunchTemplate"
    Properties:
      LaunchTemplateName: "example_name"
      LaunchTemplateData:
        CreditSpecification:
          CpuCredits:
            Ref: CpuCredits
        ImageId: "ami-123456789"
        InstanceType: "t2.small"
        KeyName: "test_key"
        SecurityGroups:
          - "sg-1234567890"
  AutoScalingGroup:
    Type: "AWS::AutoScaling::AutoScalingGroup"
    DependsOn: "LaunchTemplate"
    Properties:
      MinSize: "1"
      MaxSize: "3"
      DesiredCapacity: "1"
      LaunchTemplate:
        LaunchTemplateId:
          Ref: "LaunchTemplate"
        Version:
          Fn::GetAtt:
            [ "LaunchTemplate", "LatestVersionNumber" ]
      VPCZoneIdentifier:
        - "subnet-1234567890"
```

Hopefully that helps you get started with Launch Templates, T2 Unlimited and T3 instances!
