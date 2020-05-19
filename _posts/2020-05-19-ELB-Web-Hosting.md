---
layout: post
title: ELB Web Hosting Cloud Formation Stack
tags: AWS CloudFormation EC2 IaC
categories: Cloud
---

Server deployments in a repetitive and dependable manor can be challenging and time consuming without the use of automation toolsets.  
This article will step through the use of AWS Cloudformation templates, 'Iac' or Infrastructure as Code.  

### Prerequesites / Assumptions

- Following this article requires an AWS Account  
- Components used in this article can incur AWS charges, I recommend enabling billing alerts.  

### What is AWS Cloud Formation?

- Cloud Formation is AWS' own IaC template engine, formated in json or yaml. 

### Basic Overview
![screenshot](https://cloudchopshop.github.io/screenshots/ELB_Overview.png)

### Reference Template

- We will be referencing this json template: [elb_webhosting.json](https://gist.github.com/cloudchopshop/a79859ec876b58212a3f33e5029817f4)  


### Template Components

- Template Version: The Version must be declared 

``` json
    { 
        "AWSTemplateFormatVersion": "2010-09-09",   
```

- Description is self explanitory

``` json
    "Description" : "This template provisions a new VPC, with public and private subnets, loadbalanced EC2 instances hosting Apache",
```

- Mappings: Mappings allow for flexibility, for example this template has the image id for us-east-2, and us-west-2 the template will automatically use one of the images when deployed in either of the named regions

``` json
    "Mappings" : {
        "RegionMap" : {
        
            "us-east-2"        : {"HVM64" : "ami-0d542ef84ec55d71c"},
            "us-west-2"        : {"HVM64" : "ami-01460aa81365561fe"}
        }
    },
```

- Resources: Resources are the declared components, configured for deployment (Note: The below snippet shows a collapsed view)

``` bash
    },
    "Resources": {
        "LBWEBVPC": {               #VPC and properties
        "InternetGateway": {        #Internet Gateway
        "VPCGatewayAttachment": {   #VPC gateway attachment attaches the Internet Gateway to the VPC
        "EIP1": {                   #Elastic IP1, attached to NATGateway1
        "NATGateway1": {            #Nat Gateway for webserver subnet1
        "EIP2": {                   #Elastic IP2, attached to NATGateway2
        "NATGateway2": {            #Nat Gateway for webserver subnet2
        "WebServer1RouteTable": {   #Route table for subnet1
        "WebServer1Route": {        #Route attached to web server1routetable, defines cidr, and nat gateway
        "WebSubnet1": {             #Subnet1 for Web Servers defines cidr block, and avaliability zone
        "WebSubnet1RouteTableAssociation": {    #Subnet Route Table Association attaches the webserver1 route to the webserver1 subnet
        "WebServer2RouteTable": {   #Route table for subnet2
        "WebServer2Route": {        #Route attached to web server2routetable, defines cidr, and nat gateway 
        "WebSubnet2": {             #Subnet2 for Web Servers defines cidr block, and avaliability zone
        "WebSubnet2RouteTableAssociation": {    #Subnet table association attaches webserver2route with websubnet2
        "Public1RouteTable": {      #Public subnet 1 route table
        "Public1Route": {           #Route attached to public 1 routetable, defines cidr, and internet gateway
        "Public1Subnet": {          #Subnet1 for public, defines cidr block, avaliability zone
        "Public1SubnetRouteTableAssociation": {     #Subnet Route Table Association attaches the public route to the public subnet
        "Public2RouteTable": {      #Public subnet 2 route table
        "Public2Route": {           #Route attached to public 2 routetable, defines cidr, and internet gateway
        "Public2Subnet": {          #Subnet2 for public, defines cidr block, avaliability zone
        "Public2SubnetRouteTableAssociation": {     #Subnet Route Table Association attaches the public2 route to the public2 subnet
        "ELBSecurityGroup": {       #ELB Security Group, deines inbound rules
        "ElasticLoadBalancer": {    #Elastic Load Balancer defined listener, health check, subnet, and security group
        "WebScalingGroup": {        #WebScaling Group, defines Launch Config, Loadbalancer, Subnets to deploy the instances, scaling policy and dependencies
        "WebSecurityGroup": {       #Web Security Group, defines inbound rules from the ELBSecurityGroup
        "LaunchConfig": {           #Launch Config, defines the instance, security group, Installation scripts, files, services, and dependencies
         },
```

- Outputs (after the template is deployed this information is displayed) 

``` json
    "Outputs": {
        "URL": {
            "Description": "The URL of the website",
            "Value": {
                "Fn::Join": [
                    "",
                    [
                        "http://",
                        {
                            "Fn::GetAtt": [
                                "ElasticLoadBalancer",
                                "DNSName"
                            ]
                        }
                    ]
                ]
            }
        }
    }
```

### Cloud Formation Template Deployment

- Head to AWS, search for cloudformation from the console.  
    Create Stack  
    Upload a template file select the file provided earlier: [elb_webhosting.json](https://gist.github.com/cloudchopshop/a79859ec876b58212a3f33e5029817f4)  
    View in designer to see the components that will be created (This can be confusing at first, with the overlapping lines)  
    Check the template validity, a check box in the top left  
    Create the Stack! (Cloud Icon)  
    Stack Name: ELB-Stack  
    Stacks can be configured for parameters: This template does not have any  
    Review -> Create Stack  

- Click Refresh from the Resources tab to monitor component deployment  
    If errors occur check the status (This template specifys the us-east-2, and us-west-2 regions, additional AMIs can be added in mappings.)  

- Select the Outputs tab once the stack is complete  
    The URL should resolve to the 'Root Page'

- Test the virtual hosts  

``` bash
curl -H "Host:www.test.com" http://elb-stack-elasticl-1qqo072slb115-439493579.us-east-2.elb.amazonaws.com/
```

- Additional testing  
    Delete or shutdown instances from the Ec2 console (The Web Scaling Group should auto recover)  
    Add your own AMI for different regions  




### Contact

If you found something wrong with this article, have an improvement or other comment drop me an email!  
[mail@cloudchopshop.com](mailto:mail@cloudchopshop.com)  
