---
title: Terraform IaC DevOps using AWS CodePipeline
description: Create AWS CodePipeline with Multiple Environments Dev and Staging
---
# IaC DevOps using AWS CodePipeline

## 3-Tier Terraform AWS Application
Project Overview: This project demonstrates a production-ready 3-tier web application architecture deployed on AWS using Infrastructure as Code (Terraform). The architecture separates concerns across web, application, and database tiers to achieve scalability, security, and maintainability.  

[![Image](https://stacksimplify.com/course-images/terraform-aws-codepipeline-iac-devops-1.png "Terraform on AWS with IAC DevOps and SRE")](https://stacksimplify.com/course-images/terraform-aws-codepipeline-iac-devops-1.png)

<img width="983" height="737" alt="Codepiline1" src="https://github.com/user-attachments/assets/5b49cf09-454e-4475-856f-18f12e49014b" />

<img width="942" height="429" alt="Codepipeline2" src="https://github.com/user-attachments/assets/a0c0a6ec-a398-4a22-aef3-218aa00f4e01" />

## Business Need
Modern web applications require:

- High availability across multiple availability zones
- Automatic scaling to handle variable traffic loads
- Security isolation between application tiers
- Cost optimization through right-sized resources
- Reproducible deployments across environments

## Architecture Components
Web Tier (Public Subnet)

- Application Load Balancer with SSL termination
- Auto Scaling Group for web servers
- Route 53 DNS management

## Application Tier (Private Subnet)

- Auto Scaling Group for application servers
- NAT Gateway for secure outbound internet access
- Internal load balancing

## Database Tier (Database Subnet)

- RDS Multi-AZ deployment for high availability
- Automated backups and maintenance windows
- Network isolation from public internet

## Key Challenges Encountered
### ALB Target Group Routing 
Problem: Auto Scaling Policy creation failed with "load balancer does not route traffic to target group" error.  
Solution: Configured ALB listeners with explicit forward actions instead of relying on fixed responses. Ensured target groups were properly attached to listeners before creating scaling policies.
### Certificate Validation Issues
Problem: ACM certificate validation failed for domain ownership verification.  
Solution: Implemented automated DNS validation using Route 53 records and ensured proper hosted zone configuration.
### IAM Permission Conflicts
Problem: CodeBuild and CodePipeline lacked sufficient permissions for S3 state management and service interactions.  
Solution: Created granular IAM policies with specific resource ARNs and required actions for each service.
### State Management Complexity
Problem: Terraform state file conflicts and backend configuration issues across environments.  
Solution: Implemented remote state storage with S3 backend and DynamoDB locking to prevent concurrent modifications.

## Step-00: Introduction
1. Terraform Backend with backend-config
2. How to create multiple environments related Pipeline with single TF Config files in Terraform ? 
3. As part of Multiple environments, going to create `dev` and `stag` environments
4. Build IaC DevOps Pipelines using 
- AWS CodeBuild
- AWS CodePipeline
- Github


Step-01:
First thing I need to do is update my main-key\terraform-key.pem file with my actual private key. Make sure the name matches exactly.

Step-02: c1-versions.tf - Setting Up Terraform Backends
I need to configure remote state storage so my Terraform state doesn't just live on my local machine.

Step-02-01 Add backend block as below
t
  # Adding Backend as S3 for Remote State Storage
  backend "s3" { }
Step-02-02: Create file named dev.conf
This file tells Terraform where to store the dev environment state:

t
bucket = "terraform-on-aws-for-ec2"
key    = "iacdevops/dev/terraform.tfstate"
region = "us-east-1" 
dynamodb_table = "iacdevops-dev-tfstate"
Step-02-03: Create file named stag.conf
Same thing for staging environment:

t
bucket = "terraform-on-aws-for-ec2"
key    = "iacdevops/stag/terraform.tfstate"
region = "us-east-1" 
dynamodb_table = "iacdevops-stag-tfstate"
Step-02-04: Create S3 Bucket related folders for both environments for Terraform State Storage
I need to manually create these folders in my S3 bucket:

Go to Services -> S3 -> terraform-on-aws-for-ec2
Create Folder iacdevops
Create Folder iacdevops\dev
Create Folder iacdevops\stag
Step-02-05: Create DynamoDB Tables for Both Environments for Terraform State Locking
This prevents multiple people from running Terraform at the same time and messing things up:

Create Dynamo DB Table for Dev Environment
Table Name: iacdevops-dev-tfstate
Partition key (Primary Key): LockID (Type as String)
Table settings: Use default settings (checked)
Click on Create
Create Dynamo DB Table for Staging Environment
Table Name: iacdevops-stag-tfstate
Partition key (Primary Key): LockID (Type as String)
Table settings: Use default settings (checked)
Click on Create
Step-03: Pipeline Build Out - My Strategy Decision
I had two choices here and needed to decide which approach to take.

Step-03-01: Option-1: Create separate folders per environment
This means I'd have the same Terraform files (c1 to c13) copied into separate folders for each environment:

More work since I'd need to manage environment-specific configs everywhere
Dev - C1 to C13 - About 30 files
QA - C1 to C13 - About 30 files
Stg - C1 to C13 - About 30 files
Prd - C1 to C13 - About 30 files
DR - C1 to C13 - About 30 files
That's close to 150 files I'd have to keep in sync.
Terraform actually recommends this for critical projects where you want complete isolation, but it depends on your situation.
Step-03-02: Option-2: One set of files, multiple environments
Much cleaner - just use the same 30 Terraform files across all environments:

Only 30 files to manage across Dev, QA, Staging, Production and DR
I'm going with this approach and building the pipeline for Dev and Staging
Step-04: Combining my variable files
I'm merging vpc.auto.tfvars and ec2instance.auto.tfvars into environment-specific files like dev.tfvars and stag.tfvars. I'm removing the .auto. part since I want to explicitly pass these files to Terraform.

This way I can run:

t
terraform apply -input=false -var-file=dev.tfvars -auto-approve
Step-04-01: dev.tfvars
t
# Environment
environment = "dev"
# VPC Variables
vpc_name = "myvpc"
vpc_cidr_block = "10.0.0.0/16"
vpc_availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
vpc_public_subnets = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
vpc_private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
vpc_database_subnets= ["10.0.151.0/24", "10.0.152.0/24", "10.0.153.0/24"]
vpc_create_database_subnet_group = true 
vpc_create_database_subnet_route_table = true   
vpc_enable_nat_gateway = true  
vpc_single_nat_gateway = true

# EC2 Instance Variables
instance_type = "t3.micro"
instance_keypair = "terraform-key"
private_instance_count = 2
Step-04-01: stag.tfvars
t
# Environment
environment = "stag"
# VPC Variables
vpc_name = "myvpc"
vpc_cidr_block = "10.0.0.0/16"
vpc_availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
vpc_public_subnets = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
vpc_private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
vpc_database_subnets= ["10.0.151.0/24", "10.0.152.0/24", "10.0.153.0/24"]
vpc_create_database_subnet_group = true 
vpc_create_database_subnet_route_table = true   
vpc_enable_nat_gateway = true  
vpc_single_nat_gateway = true

# EC2 Instance Variables
instance_type = "t3.micro"
instance_keypair = "terraform-key"
private_instance_count = 2
Now I can delete these old files:

vpc.auto.tfvars
ec2instance.auto.tfvars
Step-05: terraform.tfvars
My terraform.tfvars file will only have the generic variables that apply to all environments:

t
# Generic Variables
aws_region = "us-east-1"
business_divsion = "hr"
Step-06: Cleaning up local-exec Provisioners
These don't work in CodeBuild, so I need to remove them.

Step-06-01: c9-nullresource-provisioners.tf
Removing this local provisioner since it won't work in the pipeline:

t
## Local Exec Provisioner:  local-exec provisioner (Creation-Time Provisioner - Triggered during Create Resource)
 provisioner "local-exec" {
    command = "echo VPC created on `date` and VPC ID: ${module.vpc.vpc_id} >> creation-time-vpc-id.txt"
    working_dir = "local-exec-output-files/"
    #on_failure = continue
  }
Also deleting the local-exec-output-files folder.

Step-06-02: c8-elasticip.tf
Same thing here - removing the destroy-time provisioner:

t
## Local Exec Provisioner:  local-exec provisioner (Destroy-Time Provisioner - Triggered during deletion of Resource)
  provisioner "local-exec" {
    command = "echo Destroy time prov `date` >> destroy-time-prov.txt"
    working_dir = "local-exec-output-files/"
    when = destroy
    #on_failure = continue
  }
Step-07: Making everything work with multiple environments
I need to update resource names so they don't conflict between dev and staging.

Step-07-01: c5-03-securitygroup-bastionsg.tf
t
# Before
  name = "public-bastion-sg"  
# After
  name = "${local.name}-public-bastion-sg"
Step-07-02: c5-04-securitygroup-privatesg.tf
t
# Before
  name = "private-sg"
# After
  name = "${local-name}-private-sg"
Step-07-03: c5-05-securitygroup-loadbalancersg.tf
t
# Before
  name = "loadbalancer-sg"
# After
  name = "${local.name}-loadbalancer-sg"
Step-07-04: Setting up DNS names for different environments
Each environment needs its own subdomain.

Step-07-04-01: c12-route53-dnsregistration.tf
Adding a variable for DNS name:

t
# DNS Name Input Variable
variable "dns_name" {
  description = "DNS Name to support multiple environments"
  type = string   
}
Step-07-04-02: c12-route53-dnsregistration.tf
Using that variable in the Route53 record:

t
# DNS Registration 
resource "aws_route53_record" "apps_dns" {
  zone_id = data.aws_route53_zone.mydomain.zone_id 
  name    = var.dns_name 
  type    = "A"
  alias {
    name                   = module.alb.lb_dns_name
    zone_id                = module.alb.lb_zone_id
    evaluate_target_health = true
  }  
}
Step-07-04-03: dev.tfvars
t
# DNS Name
dns_name = "devdemo1.devopsincloud.com"
Step-07-04-04: stag.tfvars
t
# DNS Name
dns_name = "stagedemo1.devopsincloud.com"
Step-07-05: c11-acm-certificatemanager.tf
Updating the SSL certificate to use environment-specific domains:

t
# Before
  subject_alternative_names = [
    "*.devopsincloud.com"
  ]

# After
  subject_alternative_names = [
    #"*.devopsincloud.com"
    var.dns_name  
  ]
Step-07-06: c13-02-autoscaling-launchtemplate-resource.tf
Making the launch template name unique per environment:

t
# Before
  name = "my-launch-template"
# After
  name_prefix = "${local.name}-"
Step-07-07: c13-02-autoscaling-launchtemplate-resource.tf
Same for the instance tags:

t
# Before
  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "myasg"
    }
  }  
# After
  tag_specifications {
    resource_type = "instance"
    tags = {
      #Name = "myasg"
      Name = local.name
    }
  }
Step-07-08: c13-03-autoscaling-resource.tf
Auto scaling group names:

t
# Before
  name_prefix = "myasg-"
# After
  name_prefix = "${local.name}-"
Step-07-09: c13-06-autoscaling-ttsp.tf
Auto scaling policy names:

t
# Before
  name = "avg-cpu-policy-greater-than-xx"
  name = "alb-target-requests-greater-than-yy"
# After
  name = "${local.name}-avg-cpu-policy-greater-than-xx"
  name = "${local.name}-alb-target-requests-greater-than-yy"
Step-08: Storing AWS credentials securely in Parameter Store
I don't want to hardcode my AWS keys anywhere, so I'm using Parameter Store.

Step-08-01: Create MY_AWS_SECRET_ACCESS_KEY
Go to Services -> Systems Manager -> Application Management -> Parameter Store -> Create Parameter
Name: /CodeBuild/MY_AWS_ACCESS_KEY_ID
Descritpion: My AWS Access Key ID for Terraform CodePipeline Project
Tier: Standard
Type: Secure String
Rest all defaults
Value: ABCXXXXDEFXXXXGHXXX
Step-08-02: Create MY_AWS_SECRET_ACCESS_KEY
Go to Services -> Systems Manager -> Application Management -> Parameter Store -> Create Parameter
Name: /CodeBuild/MY_AWS_SECRET_ACCESS_KEY
Descritpion: My AWS Secret Access Key for Terraform CodePipeline Project
Tier: Standard
Type: Secure String
Rest all defaults
Value: abcdefxjkdklsa55dsjlkdjsakj
Step-09: buildspec-dev.yml
This file tells CodeBuild how to run my Terraform deployment. Here's what each environment variable does:

TERRAFORM_VERSION: I'm using 0.15.3 (latest at the time)
TF_COMMAND: Usually "apply" to create resources, "destroy" to tear them down
AWS credentials: Pulled securely from Parameter Store
yaml
version: 0.2

env:
  variables:
    TERRAFORM_VERSION: "0.15.3"
    TF_COMMAND: "apply"
    #TF_COMMAND: "destroy"
  parameter-store:
    AWS_ACCESS_KEY_ID: "/CodeBuild/MY_AWS_ACCESS_KEY_ID"
    AWS_SECRET_ACCESS_KEY: "/CodeBuild/MY_AWS_SECRET_ACCESS_KEY"

phases:
  install:
    runtime-versions:
      python: 3.7
    on-failure: ABORT       
    commands:
      - tf_version=$TERRAFORM_VERSION
      - wget https://releases.hashicorp.com/terraform/"$TERRAFORM_VERSION"/terraform_"$TERRAFORM_VERSION"_linux_amd64.zip
      - unzip terraform_"$TERRAFORM_VERSION"_linux_amd64.zip
      - mv terraform /usr/local/bin/
  pre_build:
    on-failure: ABORT     
    commands:
      - echo terraform execution started on `date`            
  build:
    on-failure: ABORT   
    commands:
    # Project-1: AWS VPC, ASG, ALB, Route53, ACM, Security Groups and SNS 
      - cd "$CODEBUILD_SRC_DIR/terraform-manifests"
      - ls -lrt "$CODEBUILD_SRC_DIR/terraform-manifests"
      - terraform --version
      - terraform init -input=false --backend-config=dev.conf
      - terraform validate
      - terraform plan -lock=false -input=false -var-file=dev.tfvars           
      - terraform $TF_COMMAND -input=false -var-file=dev.tfvars -auto-approve  
  post_build:
    on-failure: CONTINUE   
    commands:
      - echo terraform execution completed on `date`
Step-10: buildspec-stag.yml
Pretty much the same as dev, but uses staging config files:

yaml
version: 0.2

env:
  variables:
    TERRAFORM_VERSION: "0.15.3"
    TF_COMMAND: "apply"
    #TF_COMMAND: "destroy"
  parameter-store:
    AWS_ACCESS_KEY_ID: "/CodeBuild/MY_AWS_ACCESS_KEY_ID"
    AWS_SECRET_ACCESS_KEY: "/CodeBuild/MY_AWS_SECRET_ACCESS_KEY"

phases:
  install:
    runtime-versions:
      python: 3.7
    on-failure: ABORT       
    commands:
      - tf_version=$TERRAFORM_VERSION
      - wget https://releases.hashicorp.com/terraform/"$TERRAFORM_VERSION"/terraform_"$TERRAFORM_VERSION"_linux_amd64.zip
      - unzip terraform_"$TERRAFORM_VERSION"_linux_amd64.zip
      - mv terraform /usr/local/bin/
  pre_build:
    on-failure: ABORT     
    commands:
      - echo terraform execution started on `date`            
  build:
    on-failure: ABORT   
    commands:
    # Project-1: AWS VPC, ASG, ALB, Route53, ACM, Security Groups and SNS 
      - cd "$CODEBUILD_SRC_DIR/terraform-manifests"
      - ls -lrt "$CODEBUILD_SRC_DIR/terraform-manifests"
      - terraform --version
      - terraform init -input=false --backend-config=stag.conf
      - terraform validate
      - terraform plan -lock=false -input=false -var-file=stag.tfvars           
      - terraform $TF_COMMAND -input=false -var-file=stag.tfvars -auto-approve  
  post_build:
    on-failure: CONTINUE   
    commands:
      - echo terraform execution completed on `date`
Step-11: Setting up my Github Repository
Time to get all this code into version control.

Step-11-01: Create New Github Repository
Go to github.com and login with your credentials
URL: https://github.com/olabadmus (my git repo url)
Click on Repositories Tab
Click on New to create a new repository
Repository Name: terraform-iacdevops-with-aws-codepipeline
Description: Implement Terraform IAC DevOps for AWS Project with AWS CodePipeline
Repository Type: Private
Choose License: Apache License 2.0
Click on Create Repository
Click on Code and Copy Repo link
Step-11-02: Clone Remote Repo and Copy all related files
t
# Change Directory
cd demo-repos

# Execute Git Clone
git clone https://github.com/olabadmus/terraform-iacdevops-with-aws-codepipeline.git

# Verify Git Status
git status

# Git Commit
git commit -am "First Commit"

# Push files to Remote Repository
git push

# Verify same on Remote Repository
https://github.com/olabadmus/terraform-iacdevops-with-aws-codepipeline.git
Step-12: Verify if AWS Connector for GitHub already installed on your Github
I need to check if I already have this set up:

Go to below url and verify
URL: https://github.com/settings/installations
Step-13: Create Github Connection from AWS Developer Tools
This connects AWS to my GitHub repo:

Go to Services -> CodePipeline -> Create Pipeline
In Developer Tools -> Click on Settings -> Connections -> Create Connection
Select Provider: Github
Connection Name: terraform-iacdevops-aws-cp-con1
Click on Connect to Github
GitHub Apps: Click on Install new app
It should redirect to github page Install AWS Connector for GitHub
Only select repositories: terraform-iacdevops-with-aws-codepipeline
Click on Install
Click on Connect
Verify Connection Status: It should be in Available state
Go to below url and verify
URL: https://github.com/settings/installations
You should see Install AWS Connector for GitHub app installed
Step-14: Create AWS CodePipeline
Now I'm building the actual pipeline that will deploy my infrastructure.

Go to Services -> CodePipeline -> Create Pipeline
Pipeline settings
Pipeline Name: tf-iacdevops-aws-cp1
Service role: New Service Role
rest all defaults
Artifact store: Default Location
Encryption Key: Default AWS Managed Key
Click Next
Source Stage
Source Provider: Github (Version 2)
Connection: terraform-iacdevops-aws-cp-con1
Repository name: terraform-iacdevops-with-aws-codepipeline
Branch name: main
Change detection options: leave to defaults as checked
Output artifact format: leave to defaults as CodePipeline default
Add Build Stage
Build Provider: AWS CodeBuild
Region: N.Virginia
Project Name: Click on Create Project
Project Name: codebuild-tf-iacdevops-aws-cp1
Description: CodeBuild Project for Dev Stage of IAC DevOps Terraform Demo
Environment image: Managed Image
Operating System: Amazon Linux 2
Runtimes: Standard
Image: latest available today (aws/codebuild/amazonlinux2-x86_64-standard:3.0)
Environment Type: Linux
Service Role: New (leave to defaults including Role Name)
Build specifications: use a buildspec file
Buildspec name - optional: buildspec-dev.yml (Ensure that this file is present in root folder of your github repository)
Rest all leave to defaults
Click on Continue to CodePipeline
Project Name: This value should be auto-populated with codebuild-tf-iacdevops-aws-cp1
Build Type: Single Build
Click Next
Add Deploy Stage
Click on Skip Deploy Stage
Review Stage
Click on Create Pipeline
Step-15: Testing the Pipeline (and expecting it to fail)
Verify Source Stage: Should pass
Verify Build Stage: should fail with error
Verify Build Stage logs by clicking on details in pipeline screen
I expect to see this error because the CodeBuild role can't read from Parameter Store yet:

log
[Container] 2021/05/11 06:24:06 Waiting for agent ping
[Container] 2021/05/11 06:24:09 Waiting for DOWNLOAD_SOURCE
[Container] 2021/05/11 06:24:09 Phase is DOWNLOAD_SOURCE
[Container] 2021/05/11 06:24:09 CODEBUILD_SRC_DIR=/codebuild/output/src851708532/src
[Container] 2021/05/11 06:24:09 YAML location is /codebuild/output/src851708532/src/buildspec-dev.yml
[Container] 2021/05/11 06:24:09 Processing environment variables
[Container] 2021/05/11 06:24:09 Decrypting parameter store environment variables
[Container] 2021/05/11 06:24:09 Phase complete: DOWNLOAD_SOURCE State: FAILED
[Container] 2021/05/11 06:24:09 Phase context status code: Decrypted Variables Error Message: AccessDeniedException: User: arn:aws:sts::180789647333:assumed-role/codebuild-codebuild-tf-iacdevops-aws-cp1-service-role/AWSCodeBuild-97595edc-1db1-4070-97a0-71fa862f0993 is not authorized to perform: ssm:GetParameters on resource: arn:aws:ssm:us-east-1:180789647333:parameter/CodeBuild/MY_AWS_ACCESS_KEY_ID
Step-16: Fixing the IAM permissions issue
The CodeBuild role needs permission to read from Parameter Store.

Step-16-01: Get IAM Service Role used by CodeBuild Project
First I need to find the role name:

Go to CodeBuild -> codebuild-tf-iacdevops-aws-cp1 -> Edit -> Environment
Make a note of Service Role ARN
t
# CodeBuild Service Role ARN 
arn:aws:iam::180789647333:role/service-role/codebuild-codebuild-tf-iacdevops-aws-cp1-service-role
Step-16-02: Create IAM Policy with Systems Manager Get Parameter Read Permission
Go to Services -> IAM -> Policies -> Create Policy
Service: Systems Manager
Actions: Get Parameters (Under Read)
Resources: All
Click Next Tags
Click Next Review
Policy name: systems-manger-get-parameter-access
Policy Description: Read Parameters from Parameter Store in AWS Systems Manager Service
Click on Create Policy
Step-16-03: Associate this Policy to IAM Role
Go to Services -> IAM -> Roles -> Search for codebuild-codebuild-tf-iacdevops-aws-cp1-service-role
Attach the policy named systems-manger-get-parameter-access
Step-17: Running the Pipeline Again
Go to Services -> CodePipeline -> tf-iacdevops-aws-cp1
Click on Release Change
Verify Source Stage:
Should pass
Verify Build Stage:
Verify Build Stage logs by clicking on details in pipeline screen
Verify Cloudwatch -> Log Groups logs too (Logs saved in CloudWatch for additional reference)
Step-18: Checking that everything worked
Once the pipeline runs successfully, I can verify all my resources got created:
0. Confirm SNS Subscription in your email

Verify EC2 Instances
Verify Launch Templates (High Level)
Verify Autoscaling Group (High Level)
Verify Load Balancer
Verify Load Balancer Target Group - Health Checks
Access and Test
t
# Access and Test
https://devdemo5.olatundecloud.com
https://devdemo5.olatundecloud.com/app1/index.html
https://devdemo5.olatundecloud.com/app1/metadata.html
Step-19: Adding manual approval before staging deployment
I want to approve changes before they go to staging:

Go to Services -> AWS CodePipeline -> tf-iacdevops-aws-cp1 -> Edit
Add Stage
Name: Email-Approval
Add Action Group
Action Name: Email-Approval
Action Provider: Manual Approval
SNS Topic: Select SNS Topic from drop down
Comments: Approve to deploy to staging environment
Step-20: Adding the Staging Environment Deploy Stage
Go to Services -> AWS CodePipeline -> tf-iacdevops-aws-cp1 -> Edit
Add Stage
Name: Stage-Deploy
Add Action Group
Action Name: Stage-Deploy
Region: US East (N.Virginia)
Action Provider: AWS CodeBuild
Input Artifacts: Source Artifact
Project Name: Click on Create Project
Project Name: stage-deploy-tf-iacdevops-aws-cp1
Description: CodeBuild Project for Staging Environment of IAC DevOps Terraform Demo
Environment image: Managed Image
Operating System: Amazon Linux 2
Runtimes: Standard
Image: latest available today (aws/codebuild/amazonlinux2-x86_64-standard:3.0)
Environment Type: Linux
Service Role: New (leave to defaults including Role Name)
Build specifications: use a buildspec file
Buildspec name - optional: buildspec-stag.yml (Ensure that this file is present in root folder of your github repository)
Rest all leave to defaults
Click on Continue to CodePipeline
Project Name: This value should be auto-populated with stage-deploy-tf-iacdevops-aws-cp1
Build Type: Single Build
Click on Done
Click on Save
Step-21: Update the IAM Role
The new staging CodeBuild project needs the same Parameter Store permissions:

Update the IAM Role created as part of this stage-deploy-tf-iacdevops-aws-cp1 CodeBuild project by adding the policy systems-manger-get-parameter-access1
Step-22: Running the Full Pipeline
Now I can test the complete dev -> approval -> staging flow:

Go to Services -> AWS CodePipeline -> tf-iacdevops-aws-cp1
Click on Release Change
Verify Source Stage
Verify Build Stage (Dev Environment - Dev Deploy phase)
Verify Manual Approval Stage - Approve the change
Verify Stage Deploy Stage
Verify build logs
Step-23: Verify Staging Environment
Same checks as dev, but for staging:
0. Confirm SNS Subscription in your email

Verify EC2 Instances
Verify Launch Templates (High Level)
Verify Autoscaling Group (High Level)
Verify Load Balancer
Verify Load Balancer Target Group - Health Checks
Access and Test
t
# Access and Test
https://stagedemo5.olatundecloud.com
https://stagedemo5.olatundecloud.comapp1/index.html
https://stagedemo5.olatundecloud.com/app1/metadata.html
Step-24: Testing the pipeline with a real change
Let me make a change and see the entire pipeline work end-to-end.

Step-24-01: c13-03-autoscaling-resource.tf
I'll increase the minimum EC2 instances to test scaling:

t
# Before
  desired_capacity = 2
  max_size = 10
  min_size = 2
# After
  desired_capacity = 4
  max_size = 10
  min_size = 4
Step-24-02: Commit Changes via Git Repo
t
# Verify Changes
git status

# Commit Changes to Local Repository
git add .
git commit -am "ASG Min Size from 2 to 4"

# Push changes to Remote Repository
git push
Step-24-03: Review Build Logs
Go to Services -> CodePipeline -> tf-iacdevops-aws-cp1
Verify Dev Deploy Logs
Approve at Manual Approval stage
Verify Stage Deploy Logs
Step-24-04: Verify EC2 Instances
Check that the scaling actually happened:

Go to Services -> EC2 Instances
Newly created instances should be visible.
hr-dev: 4 EC2 Instances
hr-stag: 4 EC2 Instances
Step-25: Clean up - Destroy Resources
When I'm done testing, I can tear everything down by changing TF_COMMAND to "destroy".

Step-25-01: Update buildspec-dev.yml
t
# Before
    TF_COMMAND: "apply"
    #TF_COMMAND: "destroy"
# After
    #TF_COMMAND: "apply"
    TF_COMMAND: "destroy"
Step-25-02: Update buildspec-stag.yml
t
# Before
    TF_COMMAND: "apply"
    #TF_COMMAND: "destroy"
# After
    #TF_COMMAND: "apply"
    TF_COMMAND: "destroy"
Step-25-03: Commit Changes via Git Repo
t
# Verify Changes
git status

# Commit Changes to Local Repository
git add .
git commit -am "Destroy Resources"

# Push changes to Remote Repository
git push
Step-25-03: Review Build Logs
Go to Services -> CodePipeline -> tf-iacdevops-aws-cp1
Verify Dev Deploy Logs
Approve at Manual Approval stage
Verify Stage Deploy Logs
Step-26: Reset everything back to working state
After testing destroy, I want to get back to a working demo state.

Step-26-01: c13-03-autoscaling-resource.tf
Back to the original scaling settings:

t
# Before
  desired_capacity = 4
  max_size = 10
  min_size = 4
# After
  desired_capacity = 2
  max_size = 10
  min_size = 2
Step-26-02: buildspec-dev.yml and buildspec-stag.yml
Back to apply mode:

t
# Before
    #TF_COMMAND: "apply"
    TF_COMMAND: "destroy"   
# After
    TF_COMMAND: "apply"
    #TF_COMMAND: "destroy"
Step-26

Continue

Step-26-03: Commit Changes via Git Repo
Getting everything back to a clean demo state:

t
# Verify Changes
git status

# Commit Changes to Local Repository
git add .
git commit -am "Fixed all the changes back to demo state"

# Push changes to Remote Repository
git push





## References
- [1:Backend configuration Dynamic](https://www.terraform.io/docs/cli/commands/init.html)
- [2:Backend configuration Dynamic](https://www.terraform.io/docs/language/settings/backends/configuration.html#partial-configuration)
- [AWS CodeBuild Builspe file reference](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html#build-spec.env)
