# Auto scaling the BIG-IP VE Web Application Firewall in AWS
[![Slack Status](https://f5cloudsolutions.herokuapp.com/badge.svg)](https://f5cloudsolutions.herokuapp.com)

**Contents**             

 - [Introduction](#introduction) 
 - [Prerequisites](#prerequisites)
 - [Quick Start] (#quick-start-)
 - [Getting Help](#help)
 - [Additional BIG-IP VE Deployment and Configuration Details](#additional-big-ip-ve-deployment-and-configuration-details)
 - [Security](security.md)


## Introduction
This solution implements auto scaling of BIG-IP Virtual Edition (VE) Web Application Firewall (WAF) systems in Amazon Web Services. The BIG-IP VEs have the <a href="https://f5.com/products/big-ip/local-traffic-manager-ltm">Local Traffic Manager</a> (LTM) and <a href="https://f5.com/products/big-ip/application-security-manager-asm">Application Security Manager</a> (ASM) modules enabled to provide advanced traffic management and web application security functionality.  As traffic increases or decreases, the number of BIG-IP VE WAF instances automatically increases or decreases accordingly. 


## Prerequisites
The following are prerequisites for this solution:
 - The appropriate permission in AWS to launch CloudFormation templates. This template creates Auto Scale Groups, S3 Buckets, Instances, and IAM Instance Profiles, so the account you are using must have permission to create these objects.
 - An existing AWS VPC with a public subnet, a classic Elastic load balancer (ELB) in front of the BIG-IP VE(s), and a DNS name for the application pool (which can be also be the DNS name of an ELB if using one behind the BIG-IP(s)). 
   - The classic ELB in front of the BIG-IP VEs must be pre-configured to perform SSL offload for the BIG-IP WAF auto scale tier.  See [ELB configuration](#elb) for an example of the ELB configuration.
 - Access to **Best** BIG-IP images in the Amazon region within which you are working.
 - Accepted the EULA for all Images in the AWS marketplace. If you have not accepted the EULAs, you may experience an unspecified error when trying to launch the configuration. 
 - Key pair for SSH access to BIG-IP VE (you can create or import the key pair in AWS)
 - An AWS Security Group with the following inbound rules:
    - Port 22 for SSH access to the BIG-IP VE *(source = Intra-VPC and/or mgmt networks)*
    - Port 8443 (or other port) for accessing the BIG-IP web-based Configuration utility *(source = Intra-VPC and/or mgmt networks)*. See the [Configuration Utility note](#note) for an important note about accessing the Configuration utility.
    - Port 4353 and 6123-6128 for cluster communication *(source = Intra-VPC or the public subnet of the peer)* 
    - Port 80 accessing your applications via the BIG-IP virtual server *(source = any)*
 
 
## Quick Start 
This readme file describes launching from the AWS Marketplace.  If you are using another method, see https://github.com/F5Networks/f5-aws-cloudformation/tree/master/supported/solutions/autoscale.

For the Marketplace, 
- From the **For Region** list, select your Region 
- From the **Delivery Methods** list, select **Auto Scale Cluster Deployment using AWS CFT**
- Click **Continue**
- Launch the CloudFormation template.

[Parameters](parameters.md)


### Logging into the BIG-IP device

Once you have completed the template and the BIG-IP system instantiates, use the following guidance to access the BIG-IP VE.

  - Find the IP address of the BIG-IP VE<br> You can find the BIG-IP IP address (Instance IP or Public IP) on the Instances tab of the Auto Scale group created by the template (From the AWS console, click **EC2 > Auto Scaling > Auto Scaling Groups > Instances tab > Instance ID**)

  - SSH to the Instance or Public IP address using the following syntax:<br>``` >ssh -i ~/.ssh/<YOUR-PRIVATE-SSH-KEY-HERE> admin@<INSTANCE IP> ```
  - Create a custom-admin user using the following syntax:<br> ``` #tmsh create auth user my-custom-admin partition-access add { all-partitions { role admin } } prompt-for-password```
 

  - Once you have accessed the BIG-IP using SSH, you can log into the BIG-IP web-based Configuration utility using: **https://(IP Address of the instance):8443**

<a name="note"></a>
<br>
**Note**: If you want access to the BIG-IP web-based Configuration utility, you must first SSH into the BIG-IP VE using the SSH key you provided in the template as described in this section.  You can then create a user account with admin-level permissions on the BIG-IP VE to allow access to the Configuration utility if necessary.  In this deployment, the BIG-IP Configuration utility port is 8443 by default.


You can now configure the BIG-IP VE as applicable for your configuration.  See the BIG-IP documentation for details (https://support.f5.com/csp/tech-documents)




### Help <a name="help"></a>
Because this template has been created and fully tested by F5 Networks, it is supported by F5. This means you can get assistance if necessary from F5 Technical Support.

We encourage you to use our [Slack channel](https://f5cloudsolutions.herokuapp.com) for discussion and assistance on F5 CloudFormation templates.  This channel is typically monitored Monday-Friday 9-5 PST by F5 employees who will offer best-effort support. 

---
---

## Additional BIG-IP VE deployment and configuration details 

All BIG-IP VE instances deploy with a single interface (NIC) attached to a public subnet. This single interface processes both management and data plane traffic. The LTM and ASM provide advanced traffic management and security functionality. The CloudFormation template collects some initial deployment input parameters and creates an auto scale group of BIG-IP VEs. The instances parameters and configurations are defined by the Auto Scale group's *launch configuration*. The launch configuration is used to:

  - Set the BIG-IP system information: hostname, NTP, DNS settings, and so on.
  - Provision the WAF module: BIG-IP Application Security Manager (ASM)
  - Join the auto scale cluster
  - Deploy integration with EC2 Auto Scale and CloudWatch services for scaling of the BIG-IP tier.
  - Create an initial HTTP virtual server with a basic Web Application Firewall policy (Low, Medium, High)
    - See the [Security Blocking Levels](##security-blocking-levels-) section for a description of the blocking levels for the Web Application Firewall presented in the template.

The CloudFormation template uses the default **Best 1000Mbps** image available in the AWS marketplace to license these modules (you can choose 1000, 200, or 25 Mbps). Once the first instance is deployed, it becomes the cluster primary and all subsequent instances launched will join a cluster primary to pull the latest configuration from the cluster. In this respect, you can make changes to the running configuration of this cluster and not have to manage the lifecycle of the configuration strictly through the Launch Configuration.  

#### Configuration Example <a name="config"></a>

The following is a simple configuration diagram deployment. 

![Configuration example](images/config-diagram-autoscale-waf.png)

#### Detailed clustering information
This solution creates a clustered system with "AutoSync" enabled, so any change is immediately propagated throughout the cluster. Each cluster member instance reports "Active" and "Actively" processes traffic.  Although Autosync is enabled and technically you can make changes to any existing clustered member, for consistency we recommend you make any changes to the original, primary instance.

We also recommended you launch the cluster with one member instance to start. This instance registers itself as the primary and sets "Scale In" protection to Enabled. 

When the first auto scale instance is launched, a Device Group called "autoscale-group" is automatically created. 

Whenever a new instance is launched, it joins the cluster. If those instances are scaled down, they are removed from the cluster and the original instance remains. The cluster membership is updated once every 10 minutes and sends metrics every 60 seconds using [iCall](https://devcentral.f5.com/icall).

This deployment creates an initial BIG-IP configuration using an [iApp](https://devcentral.f5.com/iapps) that includes a basic virtual service (listening on 0.0.0.0:80) with a WAF policy.   

After the first instance is launched, you can log in and customize the configuration.


### How this solution works

The CloudFormation Template creates and leverages several AWS resources to support Auto Scale, including:

  - S3 Bucket<br>
    The S3 bucket acts as persistent storage for the cluster database. It contains:
      - *Records*<br>
      Records, named for the instance ID contain metadata for each instance. For example  ```{"privateIp":"10.0.11.190","mgmtIp":"10.0.11.190","hostname":"ip-10-0-11-190.ec2.internal","isMaster":true}```<br>
      Newly launched instances query this bucket for an existing cluster and use information found here to join the cluster. If it it the first member launched in the group, it creates a record with "is Master":true. Otherwise, it enters itself as "isMaster":false.
      - *Auto-generated credentials for clustering*<br>
      The S3 bucket also contains auto-generated credentials for clustering, for example 
      ```
      {"username":"custom-admin","password":"J#\"?}$YDgb8c=L>>P8#FzmS$WB9EYzx3<"}
      ```
  - IAM Role<br>
  The IAM Role is used to create Instance Profile. The instance profile allows the auto scaled BIG-IP instances to access / update the S3 Bucket, query the Auto Scale Group, and upload metrics to Cloudwatch.
  
  - SNS Topic<br>
  The SNS topic is used to notify the admin via email of Scale Up / Down events.
  - Cloudwatch Alarms<br>
  These alarms are used to trigger scale Up / Down events.
  - Auto Scale Group<br>
  By default, the number of auto scaled instances is set to 1 and the Max is set to 8. We recommend you launch the solution with 1 instance to start. 




### Removing the deployment

To remove the deployment, there are few objects that need to be removed manually before you can delete the stack associated with the clustered auto scale BIG-IP CloudFormation template.  

1. Remove Scale-In Protection 
You must first remove "Instance Teardown" Protection on one of the auto scaled instances launched. From the AWS Console, click **EC2 > Auto Scaling Groups >** *Select your BIG-IP auto scale Group* **> Instances Tab >** *Select the Instance with "Scale-In" in the "Protected From" Tab*.  Right Click or select the Actions Tab and then click **Instance Protection > Remove Scale-In Protection**

2. Delete the S3 Bucket
This deployment creates an S3 bucket using the following naming convention: *(your-deploymentName)-autoscale-bigip-s3bucket-52byo83nzxlu*.  To delete the S3 bucket, go to the S3 page in the AWS Console, and then click **Bucket > Empty Bucket**.


3. Delete the CloudFormation stack
In the AWS Console, navigate to the CloudFormation page, select the stack created with the f5-autoscale-bigip.template and delete the stack by right-clicking or (selecting the Actions tab) and then clicking **Delete Stack**.



---

## Example ELB configuration <a name="elb"></a>
The following is an example ELB configuration that could be used in this implementation. For specific instructions on configuring an ELB, see http://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/load-balancer-getting-started.html.

```json
    "bigipElasticLoadBalancer": {
      "Type": "AWS::ElasticLoadBalancing::LoadBalancer",
      "DependsOn": "internetGatewayAttachment",
      "Properties": {
        "LoadBalancerName": {
          "Fn::Join": [
            "",
            [ 
              { "Ref" : "deploymentName" },
              "-BigipElb"
            ]
          ]
        },
        "HealthCheck": {
          "HealthyThreshold": "2",
          "Interval": "10",
          "Target": "HTTP:80/",
          "Timeout": "5",
          "UnhealthyThreshold": "10"
        },
        "subnets" : [ 
            { "Ref": "az1ExternalSubnet" },
            { "Ref": "az2ExternalSubnet" }
        ],
        "CrossZone" : true,
        "Listeners" : [ {
            "LoadBalancerPort" : "443",
            "InstancePort" : "80",
            "Protocol" : "HTTPS",
            "InstanceProtocol" : "HTTP",
            "SSLCertificateId" : { "Ref" : "certificateId" },
            "PolicyNames" : [
                "ELBSecurityPolicy-2016-08",
                "MyAppCookieStickinessPolicy"
            ]
        } ],
        "Policies" : [
            {
                "PolicyName" : "MyAppCookieStickinessPolicy",
                "PolicyType" : "AppCookieStickinessPolicyType",
                "Attributes" : [
                    { "Name" : "CookieName", "Value" : "MyCookie"}
                ]
            }
        ],
        "SecurityGroups": [
          {
            "Ref": "bigipSecurityGroup"
          }
        ],
        "Tags": [
          {
            "Key": "application",
            "Value": {
              "Ref": "AWS::StackId"
            }
          }
        ]
      }
    },

```

## Filing Issues
If you find an issue, we would love to hear about it. 
You have a choice when it comes to filing issues:
  - Use the **Issues** link on the GitHub menu bar in this repository for items such as enhancement or feature requests and non-urgent bug fixes. Tell us as much as you can about what you found and how you found it.
  - Contact F5 Technical support via your typical method for more time sensitive changes and other issues requiring immediate support.




## Copyright

Copyright 2014-2017 F5 Networks Inc.


## License


Apache V2.0
~~~~~~~~~~~
Licensed under the Apache License, Version 2.0 (the "License"); you may not use
this file except in compliance with the License. You may obtain a copy of the
License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and limitations
under the License.

Contributor License Agreement
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Individuals or business entities who contribute to this project must have
completed and submitted the `F5 Contributor License Agreement`
