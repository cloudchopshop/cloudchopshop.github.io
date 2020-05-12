---
layout: post
title: Large Browser Uploads With s3 Multipart
tags: AWS s3 Beanstalk ALB Python
categories: Cloud
---

Moving large files can be challenging, sure there are options like standing up your own sftp server or signing up for 3rd party apps but if you want full control over files, an easy UI for customers, and minimal administrative overhead what are the options? 

This article will step through the processes of publishing a browser based upload utility hosted with AWS beanstalk.  

### - Assumptions & Prerequisites -

These directions / source code are provided as is.\
An AWS account - Not responsible for any unwanted charges, I highly recomend setting up billing alerts.\
Azure AD - Used to secure the admin paths (if you dont plan on using azure you can create a second cognito pool for admins)\
A public DNS namespace\
We will be using Cognito for customer authentication, https and a valid certificate (generated from AWS) are required. 


### What the Customer will see

- Customer Login Page
![screenshot](https://cloudchopshop.github.io/screenshots/S3UP_Cognito_Login.png)

- Upload Portal
![screenshot](https://cloudchopshop.github.io/screenshots/S3UP_Upload_Portal.png)

### Prepare AWS Components

Before getting started its a good idea to confirm your AWS location I will be using us-east-1.

### s3 Bucket

- From the AWS Console search for 's3' and create a new bucket. (note the ARN for later use)

- Navigate to the permissions tab of your new s3 bucket and add a CORS configuration\
    ```
    <?xml version="1.0" encoding="UTF-8"?>
    <CORSConfiguration xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
    <CORSRule>
        <AllowedOrigin>*</AllowedOrigin>
        <AllowedMethod>PUT</AllowedMethod>
        <AllowedMethod>POST</AllowedMethod>
        <AllowedMethod>GET</AllowedMethod>
        <AllowedMethod>HEAD</AllowedMethod>
        <AllowedHeader>*</AllowedHeader>
    </CORSRule>
    </CORSConfiguration> 
    ```


### Cognito User Pool

- Search for Cognito within the AWS Console > Manage User Pools > Create a User Pool > Pool Name: sizeable-file (This will hold customer logins, create multiple pools if not using Azure for Admin AUth)\
- Step through settings
    Attributes: (Keep the defaults, and Tick 'also allow sign in with verified email address')\
    Policies: Set 'Only allow administrators to create users'\
    MFA & Verifications: Default (we arent covering SMS so no need to create the role)\
    Message Customization: Default\
    Tags: Default\
    Devices: Default\
    App Clients: > Add App Client Name: sizeable-file > keep the defaults\
    Triggers: Default\
    Create Pool

- App Integration (After the pool is created, Find App Integrations on the left column)\
    App Client Settings:\
    Enable Identity Providers: Tick 'Cognito User Pool'
    Sign In and sign out URLs: Callback URL(s) https://sizeable-file.cloudchopshop.com/oauth2/idpresponse\
    OAuth 2.0: \
    Allowed OAuth Flows: 'Authorization code grant'\
    Allowed OAuth Scopes: 'openid'\
    Domain Name: sizable-file

![screenshot](https://cloudchopshop.github.io/screenshots/S3UP_Cognito_AppClient.png.png)

- Collect the Pool ARN from the General Settings Page


### IAM User & Policy

- Create a Policy permitting access to s3 bucket and to create a user in cognito 
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "cognito-idp:AdminCreateUser"
            ],
            "Resource": "{cognitp pool arn}"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:PutObject",
                "s3:GetObject",
                "s3:AbortMultipartUpload",
                "s3:ListMultipartUploadParts",
                "s3:ListBucketMultipartUploads"
            ],
            "Resource": [
                "arn:aws:s3:::{bucketname}/*",
                "arn:aws:s3:::{bucketname}"
            ]
        }
    ]
}
```
- Create a user and assign the above policy to that user (be sure to download the csv with the IAM keys)

### Create an SSL certificate to use with your app

- Within the AWS console search for 'AWS Certificate Manager'\
    Provision Certificates > Request a certificate\
    domain name: {fqdn of the app and your public name} 'sizeable-file.cloudchopshop.com'\
    DNS Validation > Next\
    Create a CNAME record for validation: head to your registrar and add the provided CNAME




### Download the App Source Code

- using git\
    `git clone https://github.com/cloudchopshop/s3-multipart-browser-upload.git`

- Download the zip file\
    https://github.com/cloudchopshop/s3-multipart-browser-upload.git

### Prepare the files for aws beanstalk
- beanstalk accepts files from an s3 bucket or from a local machine in a zip file,
    the zip file must have application source files at the root (github download zip, is a zip file with a folder containing the source - this will not work for beanstalk)


### Deploy the App to AWS Beanstalk 
- From the AWS Console search for 'beanstalk' and create a new application\
    Application Name: {Name}\
    Tags: Project {Name}\
    Platform: Python\
    Branch: Python 3.6 running 64bit Amazon Linux\
    Version: 2.9.10 (Newer versions will likely work 2.9.10 is what was used at time of writing)\
    Application Code: Upload your code\
    Source code origin: choose the zip created from the 'prepare files step' 

- Configure more options (we need to add a loadbalancer for path routing)\
    Presets: Custom configuration\
    Edit the loadbalancer\
    Add listener: Port: 443 Protocol: https SSL Certificate: (This should auto populate the dropdown once aws validates the requested certificate)\
    Save Loadbalancer Settings\
    Create App


### Add the CNAME Record
- Create a CNAME with your registrar\
    Once the deployment completes with a green check mark collect the generated dns name (Go to environment link)\
    create a CNAME using the loadbalancer dns name

### Register Azure ClientApp

- Head to the [Azure Active Directory Admin Center](https://aad.portal.azure.com) > All Services > App Registrations > New Registration
- Register an Application\
    Name: sizeable-file (The name is only relevant to Azure, admins will be assigned to this app)\
    Supported account types: Default - Accounts in this organizational directory only\
    Redirect URI: `https://{your app domain}/oauth2/idpresponse` (The loadbalancer CNAME + /oauth2/idpresponse)\
                  `https://{ALB DNS NAME}.us-east-1.elb.amazonaws.com/oauth2/idpresponse` (The loadbalancer DNS Name + /oauth2/idpresponse) 

- API Permissions Grant Admin Consent for read profile                  
- Generate a Secret > Go to the newly registered app > certificates & secrets > New Client Secret (make sure you record the value)
- Record the Tenant / App Info (Go to App Overview > Endpoints)\
    Issuer: `https://login.microsoftonline.com/{tenant ID}/v2.0`\
    Token endpoint:`https://login.microsoftonline.com/{tenant ID}/oauth2/v2.0/token`\
    User info endpoint:`https://graph.microsoft.com/oidc/userinfo`\
    Authorization endpoint:`https://login.microsoftonline.com/{tenant ID}/oauth2/authorize`\
    ClientId: {client ID} (found on the registered app overview)\
    Client Secret: {client secret} (generated in the step above)


### Configure the Load Balancer Rules

- From the AWS Console, navigate to the EC2 dashboard and then select loadbalancers\
    select the loadbalancer with the DNS name of the beanstalk deployment\
    select listeners\
    'view/edit rules' port:80 http > delete the default action of forward, and add a redirect to https 443\
    ![screenshot](https://cloudchopshop.github.io/screenshots/S3UP_http_to_https.png)

    'view/edit rules' port:443 https > delete the default action of forward, and set a fixed response 401\
    ![screenshot](https://cloudchopshop.github.io/screenshots/S3UP_401_unrouted.png)
 
- Setup the Root User Paths\
    Add path rule for { / }\
    from the https:443 listener view/edit rules > Add\
    Insert rule > Path is /\
    Then (Action) > Authenticate > Amazon Cognito\
    Cognito user pool: > (the dropdown should show the previously deployed userpool)\
    App Client: (the dropdown should show the previously deployed app client from the userpool)\
    Add action to forward to the loadbalancer (the dropdown should auto populate)\
    ![screenshot](https://cloudchopshop.github.io/screenshots/S3UP_Cognito_Path.png)


- Setup the Root Admin Paths\
    Add path rule for { /admin* }\
    from the https:443 listener view/edit rules > Add\
    Insert rule > Path is /admin*\
    Then (Action) > Authenticate > OIDC\
    Populate the fields from the data collected from the azure app  registration\
    Session Cookie Name: AzureADAuthSessionCookie (This must be set or it will overlap with the user/customer login)\
    Session Timeout: 300\
    Add Action > Forward to > loadbalancer\
    ![screenshot](https://cloudchopshop.github.io/screenshots/S3UP_AzureAD_Path.png)

- Setup Shared User Cookie Paths\
    Add http header: Cookie Value: AWSELBAuthSessionCookie-0=*\
    Add path rule for /s3/api/v1.0 or /static*\
    Then (Action) > Authenticate > Amazon Cognito\
    Cognito user pool: > (the dropdown should show the previously deployed userpool)\
    App Client: (the dropdown should show the previously deployed app client from the userpool)\
    Add action to forward to the loadbalancer (the dropdown should auto populate)\
    ![screenshot](https://cloudchopshop.github.io/screenshots/S3UP_Cognito_Cookie.png)    

- Setup Shared Admin Cookie Paths\
    Add http header: Cookie Value: AzureADAuthSessionCookie-0=*\
    Add path rule for /s3/api/v1.0 or /static/*\
    Then (Action) > Authenticate > OIDC\
    Populate the fields from the data collected from the azure app  registration\
    Session Cookie Name: AzureADAuthSessionCookie (This must be set or it will overlap with the user/customer login)\
    Session Timeout: 300\
    Add Action > Forward to > loadbalancer\
    ![screenshot](https://cloudchopshop.github.io/screenshots/S3UP_Cognito_Cookie.png)  

### Add Outbound Rule for Loadbalancer
- For the loadbalancer to handle authentication it needs to communicate out to cognito and Azure AD\
    Navigate to EC2 > Loadbalancer > Select our Loadbalancer\
    On the Description tab scroll down to Security (Note the name of the Security Group before clicking)\
    Select the Security group in use > add an outbound rule 443 out 0.0.0.0/0\
    ![screenshot](https://cloudchopshop.github.io/screenshots/S3UP_LB_SEC.png)  

### Add Environmental Variables

Now with secure path routing enabled we can add the environment variables

- Head back to the beanstalk application, from the overview page the left column has a 'configuration' link\
    Click Edit on the software Category

- Environmental Properties (Case Sensitive / no quotes)\
    S3_KEY (Your IAM ID)\
    S3_SECRET_ACCESS_KEY (IAM SECRET KEY)\
    S3_BUCKET (Your Bucket Name)\
    PREFIX (The root folder within the bucket to store uploads)\
    Cognito_PoolId (The Cognito Pool Id not ARN Region Specific) 

- Once Applied the environment will reboot\
![screenshot](https://cloudchopshop.github.io/screenshots/S3UP_ENV.png) 

### Login

- After the application reloads with a green check mark its time to login to the admin portal\
    https://sizeable-file.cloudchopshop.com/admin\
    Authenticate with Azure AD\
    Generate an upload endpoint:\
    ![screenshot](https://cloudchopshop.github.io/screenshots/S3UP_Admin_Gen_Dir.png) 

    Capture Customer Temporary Credentials:\
    ![screenshot](https://cloudchopshop.github.io/screenshots/S3UP_Admin_Cust_Creds.png)

    Browse Customer Uploaded Files:\
    ![screenshot](https://cloudchopshop.github.io/screenshots/S3UP_Admin_View_Files.png)


### Contact

If you found something wrong with this article, have an improvement or other comment drop me an email!\
[mail@cloudchopshop.com](mailto:mail@cloudchopshop.com)


