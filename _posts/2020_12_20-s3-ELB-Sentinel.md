---
layout: post
title: Ship s3 ELB logs to Azure Sentinel
tags: Azure AWS s3 lambda ELB Python Sentinel
categories: Cloud
---

If you've started down the Azure Sentinel path, and you have existing AWS application you may find this article useful.  

This article will go over the basic steps to ship ELB logs from AWS for Sentinel processing using s3, python, and lambda.  


### Ship s3 ELB logs to Azure Sentinel

## Prereqsuisites  
- Active Azure Sentinel Workspace
- AWS APP using Elastic Load Balancer  
- ELB logs ship to s3 [Export Log Data to Amazon S3 Using the Console](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/S3ExportTasksConsole.html) (Recommend using a dedicated s3 bucket for logging)
- Lambda s3 ship to Sentinel source
[s3_ELB_to_Sentinel.py](https://gist.github.com/cloudchopshop/e352606a0e6efeddcc14e78280220ddd#file-s3_elb_to_sentinel-py)  


#### Lambda Function

- Use the above source for a new lambda function. This lambda function will need a role with permission to the s3 logging bucket.  
- Add a trigger, choose s3 complete the configuration choosting the s3 logging bucket, for event type choose all object create events, finish by adding the trigger
- if you have not done so already import the source code. 
- Environmental Variables - customer_id (this is your sentinel workspace id), shared_key (this is the shared key for the workspace)
- Once new logs are shipped to s3 the lambda function should be triggered, sending ELB logs to Azure Sentinel
![screenshot](https://cloudchopshop.github.io/screenshots/ELB_Ship_Sentinel_Lambda.png)  
![screenshot](https://cloudchopshop.github.io/screenshots/ELB_Ship_Sentinel_ENV.png)  


#### Azure Sentinel

![screenshot](https://cloudchopshop.github.io/screenshots/ELB_Ship_Sentinel.png) 


#### Troubleshooting 

If logs are not making their way to sentinel use the monitoring tab from within the lambda function to look for errors. 


### Contact

If you found something wrong with this article, have an improvement or other comment drop me an email!  
[mail@cloudchopshop.com](mailto:mail@cloudchopshop.com)
