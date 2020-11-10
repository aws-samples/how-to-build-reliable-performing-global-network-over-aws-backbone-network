# How to Build your reliable, performing global network over AWS Backbone Network

Author: Vijay Menon | AWS Principal Solutions Architect

# License

This sample code is made available  under the MIT-0 license. See the LICENSE file.

# Overview

In blogpost on [Building a global network using AWS Transit Gateway
Inter-Region
peering](https://aws.amazon.com/blogs/networking-and-content-delivery/building-a-global-network-using-aws-transit-gateway-inter-region-peering),
my colleague Nicola Arnoldi described how you can architect a global
network that relies on AWS Transit Gateway (TGW), AWS Direct Connect
(DX) and AWS Accelerated Site-to-Site VPN (S2S VPN) to build a modern,
secure, scalable and cost-effective global network on top of the AWS
backbone network. In this sample I give you a step by step
walkthrough on how to deploy a similar network. At the end of this bwalkthrough
you will have a working example global network on top of the AWS
backbone network. Below is the diagram of the example network we
implement in this walkthrough.

![](/GlobalNetworkBlog.png)

<p align="center">Figure 1: Global Network</p>

As shown in figure 1, we have 3 remote locations/offices. These are
simulated by Amazon Virtual Private Cloud (VPC), spread across 3 regions
(N. Virginia, Ireland and India) and I am calling it 'office VPC'
throughout this post. We also have 3 other VPCs in these regions with
Bastion Host in it, simulating AWS workload in the region. I am calling
it 'workload VPC' throughout this post. All offices connect to their
respective region via AWS Transit Gateway (TGW) using Accelerated
Site-to-Site VPN connection. I am using Cisco CSR 1000v AMI from market
place to simulate office network VPN headend. TGWs in each region is
connected to each other in a mesh topology using TGW cross region
peering.

Before we start, let's do some planning on IP addressing scheme and
Border Gateway Protocol (BGP) Autonomous System Numbers (ASNs) for our
network. We will be using the supernet of 10.0.0.0/8. Following is the information for individual regions.

**India: 10.0.0.0/16**   
```
  TGW-ASN                  64512   
                        
  Workload VPC CIDR        10.0.0.0/20 
  PublicSubnet1            10.0.0.0/24 
  PublicSubnet2            10.0.1.0/24
  PrivateSubnet1           10.0.2.0/24
  PrivateSubnet2           10.0.3.0/24
                            
  Office VPC CIDR          10.0.16.0/20
  Office VPC ASN           65000
  PublicSubnet1            10.0.16.0/24
  PublicSubnet2            10.0.17.0/24
  PrivateSubnet1           10.0.18.0/24
  PrivateSubnet2           10.0.19.0/24

```
**US: 10.2.0.0/16**   
```
  TGW-ASN               64514
                         
  Workload VPC CIDR     10.2.0.0/20
  PublicSubnet1         10.2.0.0/24
  PublicSubnet2         10.2.1.0/24
  PrivateSubnet1        10.2.2.0/24
  PrivateSubnet2        10.2.3.0/24
                         
  Office VPC CIDR       10.2.16.0/20
  Office VPC ASN        65002
  PublicSubnet1         10.2.16.0/24
  PublicSubnet2         10.2.17.0/24
  PrivateSubnet1        10.2.18.0/24
  PrivateSubnet2        10.2.19.0/24
```                        

**Ireland: 10.1.0.0/16**   
```
  TGW-ASN                    64513
                              
  Workload VPC CIDR          10.1.0.0/20
  PublicSubnet1              10.1.0.0/24
  PublicSubnet2              10.1.1.0/24
  PrivateSubnet1             10.1.2.0/24
  PrivateSubnet2             10.1.3.0/24
                              
  Office VPC CIDR            10.1.16.0/20
  Office VPC ASN             65001
  PublicSubnet1              10.1.16.0/24
  PublicSubnet2              10.1.17.0/24
  PrivateSubnet1             10.1.18.0/24
  PrivateSubnet2             10.1.19.0/24
```
  
### Prerequisites:

Readers of this walkthrough should be familiar with Networking Concepts
and Border Gateway Protocol (BGP) along with the following AWS services:

-   [AWS CloudFormation](https://aws.amazon.com/cloudformation/)
-   [AWS Networking and Content Delivery
    services](https://aws.amazon.com/products/networking/) like Amazon
    VPC (VPC), AWS Transit Gateway (TGW), AWS Virtual Private Network
    (VPN), Accelerated Site-to-Site VPN and AWS Global Accelerator.
-   AWS VPN connection options using third party software VPN appliance.
    I will be using [Cisco CSR 1000v
    BYOL](https://aws.amazon.com/marketplace/pp/Cisco-Systems-Inc-Cisco-Cloud-Services-Router-CSR-/B00NF48FI2)
    from AWS Marketplace in this walkthrough.

For this walkthrough, you should have the following:

1.  [AWS
    account](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/)
2.  AWS CLI
    [installed](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)
    and
    [configured](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html)
    on the workstation.
3.  Credentials configured in AWS CLI should have the required IAM
    permissions to spin up and modify the resources mentioned in this
    post.

### Walkthrough:

Let's start with creating TGW in each region. Use the following CLI
commands to do so. These TGWs will have Default Route Table Association
and Propagation enabled. Note down the TGW-id (TransitGatewayId) and TGW
route-table-id (AssociationDefaultRouteTableId) for each region from the
output of the command. We will be using these throughout this blogpost.

***India (ap-south-1):***

```
    aws ec2 create-transit-gateway \
    --region=ap-south-1 \
    --description IN-TGW \
    --options AmazonSideAsn=64512,AutoAcceptSharedAttachments=enable,DefaultRouteTableAssociation=enable,DefaultRouteTablePropagation=enable,VpnEcmpSupport=enable,DnsSupport=enable \
    --tag-specifications 'ResourceType=transit-gateway,Tags=[{Key=Name,Value=IN-BBBlogTGW}]'
```

***Ireland (eu-west-1):***

```
    aws ec2 create-transit-gateway \
    --region=eu-west-1 \
    --description EUW-TGW \
    --options AmazonSideAsn=64514,AutoAcceptSharedAttachments=enable,DefaultRouteTableAssociation=enable,DefaultRouteTablePropagation=enable,VpnEcmpSupport=enable,DnsSupport=enable \
    --tag-specifications 'ResourceType=transit-gateway,Tags=[{Key=Name,Value=EU-BBBlogTGW}]'
```

***US (us-east-1):***

```
    aws ec2 create-transit-gateway \
    --region=us-east-1 \
    --description USE-TGW \
    --options AmazonSideAsn=64513,AutoAcceptSharedAttachments=enable,DefaultRouteTableAssociation=enable,DefaultRouteTablePropagation=enable,VpnEcmpSupport=enable,DnsSupport=enable \
    --tag-specifications 'ResourceType=transit-gateway,Tags=[{Key=Name,Value=US-BBBlogTGW}]'
```

Let's peer these TGW in a mesh configuration using the following
commands. Note down the TGW Attachment id (TransitGatewayAttachmentId)
for each peering attachment from output of the commands. We will be
using these throughout this blogpost. Replace transit-gateway-id and
peer-transit-gateway-id with the one you noted in previous section. Also
replace peer-account-id with your AWS account number.

***India -- US TGW peering:***

```
    aws ec2 create-transit-gateway-peering-attachment \
    --region=ap-south-1 \
    --transit-gateway-id tgw-04a7731405afaa427 \
    --peer-transit-gateway-id tgw-0829024ccc35f0f92 \
    --peer-account-id 12345678901 \
    --peer-region us-east-1
```

***Accept India - US TGW peering attachment in us-east-1 region:***

```
    aws ec2 accept-transit-gateway-peering-attachment \
    --transit-gateway-attachment-id tgw-attach-051c92810dd244558 \
    --region us-east-1
```

***India -- Ireland TGW peering:***

```
    aws ec2 create-transit-gateway-peering-attachment \
    --region=ap-south-1 \
    --transit-gateway-id tgw-04a7731405afaa427 \
    --peer-transit-gateway-id tgw-0d2c4d145c4ec9b6b \
    --peer-account-id 12345678901 \
    --peer-region eu-west-1
```

***Accept India - Ireland TGW peering attachment in eu-west-1 region:***

```
    aws ec2 accept-transit-gateway-peering-attachment \
    --transit-gateway-attachment-id tgw-attach-080a85994533a0fb2 \
    --region eu-west-1
```

***US -- Ireland TGW peering:***

```
    aws ec2 create-transit-gateway-peering-attachment \
    --region=us-east-1 \
    --transit-gateway-id tgw-0829024ccc35f0f92 \
    --peer-transit-gateway-id tgw-0d2c4d145c4ec9b6b \
    --peer-account-id 12345678901 \
    --peer-region eu-west-1
```

***Accept US - Ireland TGW peering attachment in eu-west-1 region:***

```
    aws ec2 accept-transit-gateway-peering-attachment \
    --transit-gateway-attachment-id tgw-attach-0792088037e0287b3 \
    --region eu-west-1
```

Let's create key pairs to be used by bastion host in workload VPC and by
Cisco CSR 1000v EC2 instances in office VPC.

***India:***

```
    aws ec2 create-key-pair \
    --region ap-south-1 \
    --key-name ap-south-1-key \
    --query 'KeyMaterial' \
    --output text > ap-south-1-key.pem
```

***Make the key file read only.***

```
    chmod 400 ap-south-1-key.pem
```

***Ireland:***

```
    aws ec2 create-key-pair \
    --region eu-west-1 \
    --key-name eu-west-1-key \
    --query 'KeyMaterial' \
    --output text > eu-west-1-key.pem
```

***Make the key file read only.***

```
    chmod 400 eu-west-1-key.pem
```

***US:***

```
    aws ec2 create-key-pair \
    --region us-east-1 \
    --key-name us-east-1-key \
    --query 'KeyMaterial' \
    --output text > us-east-1-key.pem
```

***Make the key file read only.***

```
    chmod 400 us-east-1-key.pem
```

Now let's create the workload VPCs in each region and attach them to
their respective TGWs. For that download the AWS CloudFormation template
from
[here](/vpc.yaml)
to a folder in your workstation and then run the following commands from
the same folder.

***India:***

```
    aws cloudformation create-stack \
    --stack-name bbblog-ap-south-1 \
    --template-body file://vpc.yaml \
    --region ap-south-1 \
    --parameters \
    ParameterKey=VPCCIDR,ParameterValue=10.0.0.0/20 \
    ParameterKey=PublicSubnet1,ParameterValue=10.0.0.0/24 \
    ParameterKey=PublicSubnet2,ParameterValue=10.0.1.0/24 \
    ParameterKey=PrivateSubnet1,ParameterValue=10.0.2.0/24 \
    ParameterKey=PrivateSubnet2,ParameterValue=10.0.3.0/24 \
    ParameterKey=BastionHostAMIid,ParameterValue=ami-0ebc1ac48dfd14136 \
    ParameterKey=BastionHostSSHKey,ParameterValue=ap-south-1-key \
    ParameterKey=TagPrefix,ParameterValue=bbblog-ap-south-1
```

Note the VPC-ID, Route-Table-id of PublicRouteTable, PrivateRouteTable1
and PribateRouteTable2 and Subnet-ID of PrivateSubnet1 and
PrivateSubnet2 from the output section of AWS CloudFormation stack
bbblog-ap-south-1. We will be using these in subsequent steps in this
walkthrough. You can use the following command to find these ids:

```
    aws cloudformation describe-stacks \
    --stack-name bbblog-ap-south-1 \
    --region ap-south-1
```

Attach VPC to TGW. Replace transit-gateway-id, vpc-id, subnet-ids with
the ones for ap-south-1 you noted earlier:

```
    aws ec2 create-transit-gateway-vpc-attachment \
    --region ap-south-1 \
    --transit-gateway-id tgw-04a7731405afaa427 \
    --vpc-id vpc-0a18c4dd5299fddc4 \
    --subnet-ids "subnet-0d06feb37b845c1f4" "subnet-099c2cbb769ee95df"
```

***Ireland:***

```
    aws cloudformation create-stack \
    --stack-name bbblog-eu-west-1 \
    --template-body file://vpc.yaml \
    --region eu-west-1 \
    --parameters \
    ParameterKey=VPCCIDR,ParameterValue=10.1.0.0/20 \
    ParameterKey=PublicSubnet1,ParameterValue=10.1.0.0/24 \
    ParameterKey=PublicSubnet2,ParameterValue=10.1.1.0/24 \
    ParameterKey=PrivateSubnet1,ParameterValue=10.1.2.0/24 \
    ParameterKey=PrivateSubnet2,ParameterValue=10.1.3.0/24 \
    ParameterKey=BastionHostAMIid,ParameterValue=ami-07d9160fa81ccffb5 \
    ParameterKey=BastionHostSSHKey,ParameterValue=eu-west-1-key \
    ParameterKey=TagPrefix,ParameterValue=bbblog-eu-west-1
```

Note the VPC-ID, Route-Table-id of PublicRouteTable, PrivateRouteTable1
and PribateRouteTable2 and Subnet-ID of PrivateSubnet1 and
PrivateSubnet2 from the output section of above AWS CloudFormation
stack. We will be using these in subsequent steps in this walkthrough. You
can use the following command to find these ids:

```
    aws cloudformation describe-stacks \
    --stack-name bbblog-eu-west-1 \
    --region eu-west-1
```

Attach VPC to TGW. Replace transit-gateway-id, vpc-id, subnet-ids with
the ones for eu-west-1 you noted earlier:

```
    aws ec2 create-transit-gateway-vpc-attachment \
    --region eu-west-1 \
    --transit-gateway-id tgw-0d2c4d145c4ec9b6b \
    --vpc-id vpc-01ab98eb177c86a7d \
    --subnet-ids "subnet-0b9be1993310f735f" "subnet-0e85519da11698fe6"
```

***US:***

```
    aws cloudformation create-stack \
    --stack-name bbblog-us-east-1 \
    --template-body file://vpc.yaml \
    --region us-east-1 \
    --parameters \
    ParameterKey=VPCCIDR,ParameterValue=10.2.0.0/20 \
    ParameterKey=PublicSubnet1,ParameterValue=10.2.0.0/24 \
    ParameterKey=PublicSubnet2,ParameterValue=10.2.1.0/24 \
    ParameterKey=PrivateSubnet1,ParameterValue=10.2.2.0/24 \
    ParameterKey=PrivateSubnet2,ParameterValue=10.2.3.0/24 \
    ParameterKey=BastionHostAMIid,ParameterValue=ami-02354e95b39ca8dec \
    ParameterKey=BastionHostSSHKey,ParameterValue=us-east-1-key \
    ParameterKey=TagPrefix,ParameterValue=bbblog-us-east-1
```

Note the VPC-ID, Route-Table-id of PublicRouteTable, PrivateRouteTable1
and PribateRouteTable2 and Subnet-ID of PrivateSubnet1 and
PrivateSubnet2 from the output section of above AWS CloudFormation
stack. We will be using these in subsequent steps in this walkthrough. You
can use the following command to find these ids:

```
    aws cloudformation describe-stacks \
    --stack-name bbblog-us-east-1 \
    --region us-east-1
```

Attach VPC to TGW. Replace transit-gateway-id, vpc-id, subnet-ids with
the ones for us-east-1 you noted earlier:

```
    aws ec2 create-transit-gateway-vpc-attachment \
    --region us-east-1 \
    --transit-gateway-id tgw-0829024ccc35f0f92 \
    --vpc-id vpc-0201afb2014d51d0c \
    --subnet-ids "subnet-0704cbdcb186db2dc" "subnet-020365e02db5cd884"
```

Let's configure routes for cross region TGW peering. To do that you will
need TGW route table id and cross region TGW attachment id. You can get
it from the output of following command for each region. Replace
transit-gateway-route-table-id and transit-gateway-attachment-id with
the one for peering attachment between the two regions.

```
    aws ec2 describe-transit-gateways --region <region-code>    
```

and 

```
    aws ec2 describe-transit-gateway-peering-attachments --region <region-code>    
```

***Add routes for us-east-1 CIDR in ap-south-1***

```
    aws ec2 create-transit-gateway-route \
    --region ap-south-1 \
    --destination-cidr-block 10.2.0.0/16 \
    --transit-gateway-route-table-id tgw-rtb-0b6326ad43128db8a \
    --transit-gateway-attachment-id tgw-attach-051c92810dd244558
```

***Add routes for eu-west-1 CIDR in ap-south-1***

```
    aws ec2 create-transit-gateway-route \
    --region ap-south-1 \
    --destination-cidr-block 10.1.0.0/16 \
    --transit-gateway-route-table-id tgw-rtb-0b6326ad43128db8a \
    --transit-gateway-attachment-id tgw-attach-080a85994533a0fb2
```

***Add routes for us-east-1 CIDR in eu-west-1***

```
    aws ec2 create-transit-gateway-route \
    --region eu-west-1 \
    --destination-cidr-block 10.2.0.0/16 \
    --transit-gateway-route-table-id tgw-rtb-02d69e2e6585fe372 \
    --transit-gateway-attachment-id tgw-attach-0792088037e0287b3
```

***Add routes for ap-south-1 CIDR in eu-west-1***

```
    aws ec2 create-transit-gateway-route \
    --region eu-west-1 \
    --destination-cidr-block 10.0.0.0/16 \
    --transit-gateway-route-table-id tgw-rtb-02d69e2e6585fe372 \
    --transit-gateway-attachment-id tgw-attach-080a85994533a0fb2
```

***Add routes for ap-south-1 CIDR in us-east-1***

```
    aws ec2 create-transit-gateway-route \
    --region us-east-1 \
    --destination-cidr-block 10.0.0.0/16 \
    --transit-gateway-route-table-id tgw-rtb-006945ff8a967bafc \
    --transit-gateway-attachment-id tgw-attach-051c92810dd244558
```

***Add routes for eu-west-1 CIDR in us-east-1***

```
    aws ec2 create-transit-gateway-route \
    --region us-east-1 \
    --destination-cidr-block 10.1.0.0/16 \
    --transit-gateway-route-table-id tgw-rtb-006945ff8a967bafc \
    --transit-gateway-attachment-id tgw-attach-0792088037e0287b3
```

Traffic from workload VPCs to other regions will have the target of TGW
attachment. In this case any traffic form workload VPCs towards supernet
of 10.0.0.0/8 will have the target as TGW attached to the VPC. Execute
the follwoing commands to add these routes in workload VPCs route
tables. Replace the value of route-table-id with that of each subnet in
the workload VPC. You can find it in output section of CloudFormation
template in each region. Replace transit-gateway-id with the one you
captured in the beginning of the post for each region.

***India:***

```
    aws ec2 create-route \
    --route-table-id rtb-0f1b26b868a3cab15
    --destination-cidr-block 10.0.0.0/8 \
    --transit-gateway-id tgw-04a7731405afaa427 \
    --region ap-south-1
```

```
    aws ec2 create-route \
    --route-table-id rtb-0a1fd81e4d19e40ce \
    --destination-cidr-block 10.0.0.0/8 \
    --transit-gateway-id tgw-04a7731405afaa427 \
    --region ap-south-1
```

    aws ec2 create-route \
    --route-table-id rtb-02707ae43b291bd07 \
    --destination-cidr-block 10.0.0.0/8 \
    --transit-gateway-id tgw-04a7731405afaa427 \
    --region ap-south-1

***Ireland:***

    aws ec2 create-route \
    --route-table-id rtb-045714e55b7aa89bf \
    --destination-cidr-block 10.0.0.0/8 \
    --transit-gateway-id tgw-0d2c4d145c4ec9b6b \
    --region eu-west-1

     aws ec2 create-route \
     --route-table-id rtb-02a36e0b56635281c \
     --destination-cidr-block 10.0.0.0/8 \
     --transit-gateway-id tgw-0d2c4d145c4ec9b6b \
     --region eu-west-1

     aws ec2 create-route \
     --route-table-id rtb-0d41b75f263a6e209
     --destination-cidr-block 10.0.0.0/8 \
     --transit-gateway-id tgw-0d2c4d145c4ec9b6b \
     --region eu-west-1

***US:***

    aws ec2 create-route \
    --route-table-id rtb-0708c0bed0543f10e
    --destination-cidr-block 10.0.0.0/8 \
    --transit-gateway-id tgw-0829024ccc35f0f92 \
    --region us-east-1

    aws ec2 create-route \
    --route-table-id rtb-05046823347a013fd
    --destination-cidr-block 10.0.0.0/8 \
    --transit-gateway-id tgw-0829024ccc35f0f92 \
    --region us-east-1

    aws ec2 create-route \
    --route-table-id rtb-0f70c4f9da87cfc19
    --destination-cidr-block 10.0.0.0/8 \
    --transit-gateway-id tgw-0829024ccc35f0f92 \
    --region us-east-1

Now let's create the office networks (office VPC) in each region
simulated by VPCs and attach them to their respective TGWs via S2S
Accelerated VPNs. For that download the AWS CloudFormation template from
[here](/remoteSiteVpc.yaml)
to a folder in your workstation and then run the following commands from
the same folder.

***India:***

    aws cloudformation create-stack \
    --stack-name bbblog-CSR-ap-south-1 \
    --template-body file://remoteSiteVpccopy.yaml \
    --region ap-south-1 \
    --parameters \
    ParameterKey=VPCCIDR,ParameterValue=10.0.16.0/20 \
    ParameterKey=PublicSubnet1,ParameterValue=10.0.16.0/24 \
    ParameterKey=PublicSubnet2,ParameterValue=10.0.17.0/24 \
    ParameterKey=PrivateSubnet1,ParameterValue=10.0.18.0/24 \
    ParameterKey=PrivateSubnet2,ParameterValue=10.0.19.0/24 \
    ParameterKey=CSRAMIid,ParameterValue=ami-0126515024e181b2e \
    ParameterKey=CSRSSHKey,ParameterValue=ap-south-1-key \
    ParameterKey=CSRBGPASN,ParameterValue=65000 \
    ParameterKey=TagPrefix,ParameterValue=bbblog-ap-south-1

To attach this VPC to TGW via S2S Accelerated VPN follow the steps
below:

-   Open the Amazon VPC console at <https://console.aws.amazon.com/vpc/>
    in ap-south-1
-   On the navigation pane, choose Transit Gateway Attachments.
-   Choose Create Transit Gateway Attachment.
-   For Transit Gateway ID, choose the transit gateway you created
    earlier in ap-south-1 region.
-   For Attachment type, choose VPN.
-   For Customer Gateway, choose Existing, and then select the gateway
    you created in ap-south-1 region earlier.
-   For Routing options, choose Dynamic.
-   Check Enable acceleration
-   For Tunnel Options do the following:
    -   For Inside IP CIDR for Tunnel 1 enter: 169.254.10.0/30
    -   For Pre-Shared Key for Tunnel 1 enter: BBBlogPreSharedKey1
    -   For Inside IP CIDR for Tunnel 2 enter: 169.254.10.4/30
    -   For Pre-shared key for Tunnel 2 enter: BBBlogPreSharedKey2
-   Choose Create attachment and then click on Close.

Next, click on the Resource ID of newly created TGW attachment. In the
resulting screen click on \"Download Configuration\" and select the
Vendor, Platform and Software for the customer gateway you created
earlier section and Download the sample configuration. You can modify
the configuration (like IPSec parameters for Phase 1 and 2 etc.) as per
your policies and standards. Cisco CSR 1000v configuration I used can be
found
[here](/ap-south-1-CSR-Config).

***Ireland:***

    aws cloudformation create-stack \
    --stack-name bbblog-CSR-eu-west-1 \
    --template-body file://remoteSiteVpc.yaml \
    --region eu-west-1 \
    --parameters \
    ParameterKey=VPCCIDR,ParameterValue=10.1.16.0/20 \
    ParameterKey=PublicSubnet1,ParameterValue=10.1.16.0/24 \
    ParameterKey=PublicSubnet2,ParameterValue=10.1.17.0/24 \
    ParameterKey=PrivateSubnet1,ParameterValue=10.1.18.0/24 \
    ParameterKey=PrivateSubnet2,ParameterValue=10.1.19.0/24 \
    ParameterKey=CSRAMIid,ParameterValue=ami-028ff7f30065ad8b3 \
    ParameterKey=CSRSSHKey,ParameterValue=eu-west-1-key \
    ParameterKey=CSRBGPASN,ParameterValue=65001 \
    ParameterKey=TagPrefix,ParameterValue=bbblog-eu-west-1

To attach this VPC to TGW via S2S Accelerated VPN follow the steps
below:

-   Open the Amazon VPC console at <https://console.aws.amazon.com/vpc/>
-   On the navigation pane, choose Transit Gateway Attachments.
-   Choose Create Transit Gateway Attachment.
-   For Transit Gateway ID, choose the transit gateway you created
    earlier in eu-west-1 region.
-   For Attachment type, choose VPN.
-   For Customer Gateway, choose Existing, and then select the gateway
    you created in eu-west-1 region earlier.
-   For Routing options, choose Dynamic.
-   Check Enable acceleration
-   For Tunnel Options do the following:
    -   For Inside IP CIDR for Tunnel 1 enter: 169.254.10.8/30
    -   For Pre-Shared Key for Tunnel 1 enter: BBBlogPreSharedKey1
    -   For Inside IP CIDR for Tunnel 2 enter: 169.254.10.12/30
    -   For Pre-shared key for Tunnel 2 enter: BBBlogPreSharedKey2
-   Choose Create attachment and then click on Close.

Next, click on the Resource ID of newly created TGW attachment. In the
resulting screen click on \"Download Configuration\" and select the
Vendor, Platform and Software for the customer gateway you created
earlier section and Download the sample configuration. You can modify
the configuration (like IPSec parameters for Phase 1 and 2 etc.) as per
your policies and standards. Cisco CSR 1000v configuration I used can be
found
[here](/eu-west-1-CSR-Config).

***US:***

    aws cloudformation create-stack \
    --stack-name bbblog-CSR-us-east-1 \
    --template-body file://remoteSiteVpc.yaml \
    --region us-east-1 \
    --parameters \
    ParameterKey=VPCCIDR,ParameterValue=10.2.16.0/20 \
    ParameterKey=PublicSubnet1,ParameterValue=10.2.16.0/24 \
    ParameterKey=PublicSubnet2,ParameterValue=10.2.17.0/24 \
    ParameterKey=PrivateSubnet1,ParameterValue=10.2.18.0/24 \
    ParameterKey=PrivateSubnet2,ParameterValue=10.2.19.0/24 \
    ParameterKey=CSRAMIid,ParameterValue=ami-0d8a2f539abbd5763 \
    ParameterKey=CSRSSHKey,ParameterValue=us-east-1-key \
    ParameterKey=CSRBGPASN,ParameterValue=65002 \
    ParameterKey=TagPrefix,ParameterValue=bbblog-us-east-1

To attach this VPC to TGW via S2S Accelerated VPN follow the steps
below:

-   Open the Amazon VPC console at <https://console.aws.amazon.com/vpc/>
-   On the navigation pane, choose Transit Gateway Attachments.
-   Choose Create Transit Gateway Attachment.
-   For Transit Gateway ID, choose the transit gateway you created
    earlier in us-east-1 region.
-   For Attachment type, choose VPN.
-   For Customer Gateway, choose Existing, and then select the gateway
    you created in us-east-1 region earlier.
-   For Routing options, choose Dynamic.
-   Check Enable acceleration
-   For Tunnel Options do the following:
    -   For Inside IP CIDR for Tunnel 1 enter: 169.254.10.16/30
    -   For Pre-Shared Key for Tunnel 1 enter: BBBlogPreSharedKey1
    -   For Inside IP CIDR for Tunnel 2 enter: 169.254.10.20/30
    -   For Pre-shared key for Tunnel 2 enter: BBBlogPreSharedKey2
-   Choose Create attachment and then click on Close.

Next click on the Resource ID of newly created TGW attachment. In the
resulting screen click on \"Download Configuration\" and select the
Vendor, Platform and Software for the customer gateway you created
earlier section and Download the sample configuration. You can modify
the configuration (like IPSec parameters for Phase 1 and 2 etc.) as per
your policies and standards. Cisco CSR 1000v configuration I used can be
found
[here](/us-east-1-CSR-Config).

### Testing end to end connectivity over Amazon Backbone:

Let's start by examining the route tables and BGP state in the Cisco CSR
1000v routers. We should see it receiving prefixes of other regions
CIDRs as well as same region workload VPC. Log in to the CSR instance in
each region by running the following command and then run Cisco show
commands as shown subsequently.

```
    $ ssh -i "ap-south-1-key.pem" ec2-user\@ec2-3-7-123-18.ap-south-1.compute.amazonaws.com
    
    ip-10-0-16-133#show ip bgp summary
    BGP router identifier 169.254.10.2, local AS number 65000
    BGP table version is 18, main routing table version 18
    5 network entries using 1240 bytes of memory
    8 path entries using 1152 bytes of memory
    2/2 BGP path/bestpath attribute entries using 576 bytes of memory
    1 BGP AS-PATH entries using 24 bytes of memory
    0 BGP route-map cache entries using 0 bytes of memory
    0 BGP filter-list cache entries using 0 bytes of memory
    BGP using 2992 total bytes of memory
    BGP activity 6/1 prefixes, 15/7 paths, scan interval 60 secs
    5 networks peaked at 10:47:15 Aug 10 2020 UTC (19:53:26.075 ago).
    Neighbor V AS MsgRcvd MsgSent TblVer InQ OutQ Up/Down State/PfxRcd
    169.254.10.1 4 64512 1255 1322 18 0 0 03:28:36 3
    169.254.10.5 4 64512 7127 7492 18 0 0 19:47:31 3
    
    ip-10-0-16-133#show ip bgp
    BGP table version is 18, local router ID is 169.254.10.2
    Status codes: s suppressed, d damped, h history, \* valid, \    best, i -
    internal,
    r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
    x best-external, a additional-path, c RIB-compressed,
    t secondary path, L long-lived-stale,
    Origin codes: i - IGP, e - EGP, ? - incomplete
    RPKI validation codes: V valid, I invalid, N Not found
    Network Next Hop Metric LocPrf Weight Path
    
    * 10.0.0.0/20 169.254.10.1 100 0 64512 i
    *    169.254.10.5 100 0 64512 i
    *    10.0.16.0/20 0.0.0.0 0 32768 i
    * 10.1.0.0/16 169.254.10.1 100 0 64512 i
    *    169.254.10.5 100 0 64512 i
    * 10.2.0.0/16 169.254.10.1 100 0 64512 i
    *    169.254.10.5 100 0 64512 i
    *    169.254.10.0/30 0.0.0.0 0 32768 i
    
    ip-10-0-16-133#show ip route
    Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B -
    BGP
    D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
    N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
    E1 - OSPF external type 1, E2 - OSPF external type 2, m - OMP
    n - NAT, Ni - NAT inside, No - NAT outside, Nd - NAT DIA
    i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
    ia - IS-IS inter area, * - candidate default, U - per-user static route
    H - NHRP, G - NHRP registered, g - NHRP registration summary
    o - ODR, P - periodic downloaded static route, l - LISP
    a - application route
    + - replicated route, % - next hop override, p - overrides from PfR
    
    Gateway of last resort is 10.0.16.1 to network 0.0.0.0
    S* 0.0.0.0/0 [1/0] via 10.0.16.1, GigabitEthernet1
    10.0.0.0/8 is variably subnetted, 6 subnets, 4 masks
    B 10.0.0.0/20 [20/100] via 169.254.10.5, 11:55:39
    S 10.0.16.0/20 is directly connected, GigabitEthernet1
    C 10.0.16.0/24 is directly connected, GigabitEthernet1
    L 10.0.16.133/32 is directly connected, GigabitEthernet1
    B 10.1.0.0/16 [20/100] via 169.254.10.5, 11:55:39
    B 10.2.0.0/16 [20/100] via 169.254.10.5, 11:55:39
    169.254.0.0/16 is variably subnetted, 4 subnets, 2 masks
    C 169.254.10.0/30 is directly connected, Tunnel1
    L 169.254.10.2/32 is directly connected, Tunnel1
    C 169.254.10.4/30 is directly connected, Tunnel2
    L 169.254.10.6/32 is directly connected, Tunnel2

```

To test the connectivity, ping private IP addresses of bastion hosts and
CSR 1000v instances in other regions. Below are the ping outputs showing
successful connectivity from India office VPC. You can perform similar
tests and run similar show commands from Ireland and US office VPCs too.

```
    ip-10-0-16-133#ping 10.0.0.22 source GigabitEthernet 1
    Type escape sequence to abort.
    Sending 5, 100-byte ICMP Echos to 10.0.0.22, timeout is 2 seconds:
    Packet sent with a source address of 10.0.16.133
    !!!!!
    Success rate is 100 percent (5/5), round-trip min/avg/max = 2/2/3 ms
```

```
    ip-10-0-16-133#ping 10.2.16.173 source GigabitEthernet 1
    Type escape sequence to abort.
    Sending 5, 100-byte ICMP Echos to 10.2.16.173, timeout is 2 seconds:
    Packet sent with a source address of 10.0.16.133
    !!!!!
    Success rate is 100 percent (5/5), round-trip min/avg/max = 190/191/193ms
```

```
    ip-10-0-16-133#ping 10.2.0.117 source GigabitEthernet 1
    Type escape sequence to abort.
    Sending 5, 100-byte ICMP Echos to 10.2.0.117, timeout is 2 seconds:
    Packet sent with a source address of 10.0.16.133
    !!!!!
    Success rate is 100 percent (5/5), round-trip min/avg/max = 189/189/191ms
```

```
    ip-10-0-16-133#ping 10.1.16.129 source GigabitEthernet 1
    Type escape sequence to abort.
    Sending 5, 100-byte ICMP Echos to 10.1.16.129, timeout is 2 seconds:
    Packet sent with a source address of 10.0.16.133
    !!!!!
    Success rate is 100 percent (5/5), round-trip min/avg/max = 128/128/129ms
```

```
    ip-10-0-16-133#ping 10.1.0.166 source GigabitEthernet 1
    Type escape sequence to abort.
    Sending 5, 100-byte ICMP Echos to 10.1.0.166, timeout is 2 seconds:
    Packet sent with a source address of 10.0.16.133
    !!!!!
    Success rate is 100 percent (5/5), round-trip min/avg/max = 135/135/136ms
```

### Clean-Up:

To avoid additional charges, ensure you do the following:

-   Delete all TGW Peerings and VPC attachments you created for this
    walkthrough by following the steps mentioned in this
    [link](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-vpc-attachments.html).

-   Delete all VPN attachments you created for this walkthrough by
    following the steps mentioned in this
    [link](https://docs.aws.amazon.com/vpn/latest/s2svpn/delete-vpn.html).

-   Delete CloudFormation templates bbblog-ap-south-1,
    bbblog-CSR-ap-south-1, bbblog-eu-west-1, bbblog-CSR-eu-west-1,
    bbblog-us-east-1, bbblog-CSR-us-east-1.

-   Delete all the three transit gateways you created for this walkthrough
    by following the steps mentioned in this
    [link](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-transit-gateways.html#delete-tgw).

-   Delete all the EC2 key-pairs you created for this walkthrough by
    following the steps mentioned in this
    [link](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#delete-key-pair).

### Conclusion:

In this walkthrough, you created a sample global network connecting offices and
AWS VPCs spread across multiple geographies using AWS networking
constructs and services. You can modify this sample to fit your
networking requirement, add more sites and regions and build you own
global network using AWS backbone network.
