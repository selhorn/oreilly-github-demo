### Template Parameters ###
One you have launched the CFT from the marketplace, you need to complete the template by entering the required parameter values. The following table can help you gather the information you need before beginning the template.  


| Parameter | Required | Description |
| --- | --- | --- |
| deploymentName | x | Name the template uses to create BIG-IP and AWS object names |
| vpc | x | AWS VPC where you want to deploy the BIG-IP VEs |
| availabilityZones | x | Availability Zones where you want to deploy the BIG-IP VEs (we recommend at least 2) |
| subnets | x | Public or External Subnet for the Availability Zones |
| bigipSecurityGroup | x | AWS Security Group for the BIG-IP VEs |
| bigipElasticLoadBalancer | x | AWS Elastic Load Balancer group for the BIG-IP VEs |
| sshKey | x | EC2 KeyPair to enable SSH access to the BIG-IP instance |
| instanceType | x | AWS Instance Type (the default is m3.2xlarge) |
| throughput | x | Maximum amount of throughput for the BIG-IP VEs (the default is 1000Mbps) |
| adminUsername | x | BIG-IP Admin Username for clustering. Note that the user name can contain only alphanumeric characters, periods ( . ), underscores ( _ ), or hyphens ( - ). Note also that the user name cannot be any of the following: adm, apache, bin, daemon, guest, lp, mail, manager, mysql, named, nobody, ntp, operator, partition, password, pcap, postfix, radvd, root, rpc, rpm, sshd, syscheck, tomcat, uucp, or vcsa. |
| managementGuiPort | x | Port of BIG-IP management Configuration utility (the default is 8443) |
| timezone | x | Olson timezone string from /usr/share/zoneinfo (the default is UTC) |
| ntpServer | x | NTP server for this implementation (Default 0.pool.ntp.org) |
| scalingMinSize | x | Minimum number of BIG-IP instances (1-8) to be available in the Auto Scaling Group (the default is 1) |
| scalingMaxSize | x | Maximum number of BIG-IP instances (2-8) that can be created in the Auto Scaling Group (the default is 3) |
| scaleDownBytesThreshold | x | Incoming Bytes Threshold to begin scaling down BIG-IP Instances (the default is 10000)<sup>1</sup> |
| scaleUpBytesThreshold | x | Incoming Bytes Threshold to begin scaling up BIG-IP Instances (the default is 35000)<sup>1</sup> |
| notificationEmail |  | Valid email address to send Auto Scaling Event Notifications |
| virtualServicePort | x | Port on BIG-IP (the default is 80) |
| applicationPort | x | Application Pool Member Port on BIG-IP (the default is 80) |
| appInternalDnsName | x | DNS name for the application pool |
| [policyLevel](#security-blocking-levels-) | x | WAF Policy Level to protect the application (the default is high) |
| application |  | Application Tag (the default is f5app) |
| environment |  | Environment Name Tag (the default is f5env) |
| group |  | Group Tag (the default is f5group) |
| owner |  | Owner Tag (the default is f5owner) |
| costcenter |  | Cost Center Tag (the default is f5costcenter) |
<br>


<sup>1</sup> Note about the Scaling Up/Down Thresholds:
The default values are set artificially low for testing. We recommend adjusting as a ratio to utility size (optional).
For example, 80% of Throughput:

Scale Up Bytes Threshold = <br>
25 Mbps   = 3276800 bytes   * .80 =   2621440<br>
200 Mbps  = 26214400 bytes  * .80 =  20971520<br>
1000 Mbps = 131072000 bytes * .80 = 104857600<br>
5000 Mbps = 655360000 bytes * .80 = 524288000


