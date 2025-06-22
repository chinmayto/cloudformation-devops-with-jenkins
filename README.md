# Provisioning AWS Infrastructure Using Jenkins Pipelines and CloudFormation

In this blog post, we’ll explore how to automate AWS infrastructure provisioning using CloudFormation templates orchestrated by Jenkins pipelines. We'll build a web server environment deployed across three stages: development, staging, and production, following a CI/CD approach.

This post builds upon the previous blog where we detailed setting up a Jenkins server on an EC2 instance and configuring it with essential plugins. We’ll now use that Jenkins server to drive infrastructure-as-code (IaC) workflows using AWS CloudFormation.


### Architecture Overview
The high-level architecture for this solution involves:

- A Jenkins server deployed on an EC2 instance in a public subnet.
- A Git repository containing the CloudFormation templates for creating network and compute resources.
- An S3 bucket to store the templates for nested stack execution.
- A Jenkins pipeline configured to:
- Fetch templates from Git
- Upload them to S3
- Execute CloudFormation stacks in different environments (Dev/Staging/Prod)
- Provide an option to provision or decommission the infrastructure

### Architecture Diagram

![alt text](/images/arechitecture.png)

### Step 1: Create a Jenkins server on EC2 and install required plugins
Please refer to my earlier blog post for this:
https://dev.to/chinmay13/deploying-jenkins-on-ec2-for-cicd-pipelines-k15

### Step 2: Create a Jenkins User with AWS Access
To allow Jenkins to interact with AWS services via the pipeline, create an IAM user with programmatic access:

```aws iam create-user --user-name jenkinsadmivn```

Attach an appropriate IAM policy. For demo purposes, we've used AdministratorAccess (not recommended for production):

```aws iam attach-user-policy --user-name jenkinsadmin --policy-arn arn:aws:iam::aws:policy/AdministratorAccess```

![alt text](/images/jenkins_iam_user.png)

Generate AWS access keys and securely configure them in Jenkins under:

Manage Jenkins → Configure System → AWS Credentials (Environment Variables/With Plugins)

![alt text](/images/jenkins_aws_cred_1.png)

![alt text](/images/jenkins_aws_cred_2.png)

### Step 3: CloudFormation Template Directory Structure

```bash
cloudformation-devops-with-jenkins/
│   Jenkinsfile
└───infrastructure
    │   create-cfn-template-bucket.yaml
    ├───development
    │       compute.yaml
    │       network.yaml
    │       root.yaml
    ├───production
    │       compute.yaml
    │       network.yaml
    │       root.yaml
    └───staging
            compute.yaml
            network.yaml
            root.yaml
```

- `Jenkinsfile` - pipeline defined with various parameters and stages

```groovy
pipeline {
    agent any

    parameters {
            booleanParam(name: 'DEVELOPMENT', defaultValue: false, description: 'Check to deploy to Development environment')
            booleanParam(name: 'STAGING', defaultValue: false, description: 'Check to deploy to Staging environment')
            booleanParam(name: 'PRODUCTION', defaultValue: false, description: 'Check to deploy to Production environment')
            booleanParam(name: 'DEVELOPMENT_ROLLBACK', defaultValue: false, description: 'Check to rollback for Development environment')
            booleanParam(name: 'STAGING_ROLLBACK', defaultValue: false, description: 'Check to rollback for Staging environment')
            booleanParam(name: 'PRODUCTION_ROLLBACK', defaultValue: false, description: 'Check to rollback for Production environment')

    }

    stages {
        stage('Clone Repository') {
            steps {
                // Clean workspace before cloning (optional)
                deleteDir()

                // Clone the Git repository
                git branch: 'main',
                    url: 'https://github.com/chinmayto/cloudformation-devops-with-jenkins.git'

                sh "ls -lart"
            }
        }

        stage('Create CFN Template S3 Bucket') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-crendentails-jenkinsadmin']]) {
                    dir('infrastructure') {
                        sh "aws cloudformation deploy --stack-name cfn-s3bucket --template-file create-cfn-template-bucket.yaml --region 'us-east-1' --capabilities CAPABILITY_IAM"
                        sh "aws s3 cp ../ s3://ct-cfn-files-for-stack/ --recursive"
                    }
                }
            }
        }

        stage('Development Deployment') {
            steps {
                script {
                    if (params.DEVELOPMENT) {
                       withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-crendentails-jenkinsadmin']]){
                            dir('infrastructure/development/') {
                                sh 'echo "=================Development Deployment=================="'
                                sh "aws cloudformation deploy --stack-name DeployDevelopmentStack --template-file root.yaml --region 'us-east-1' --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM"
                            }
                        }
                    }
                }
            }
        }

        stage('Staging Deployment') {
            steps {
                script {
                    if (params.STAGING) {
                       withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-crendentails-jenkinsadmin']]){
                            dir('infrastructure/staging/') {
                                sh 'echo "=================Staging Deployment=================="'
                                sh "aws cloudformation deploy --stack-name DeployStagingStack --template-file root.yaml --region 'us-east-1' --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM"
                            }
                        }
                    }
                }
            }
        }

        stage('Production Deployment') {
            steps {
                script {
                    if (params.PRODUCTION) {
                       withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-crendentails-jenkinsadmin']]){
                            dir('infrastructure/production/') {
                                sh 'echo "=================Production Deployment=================="'
                                sh "aws cloudformation deploy --stack-name DeployProductionStack --template-file root.yaml --region 'us-east-1' --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM"
                            }
                        }
                    }
                }
            }
        }

        stage('Development Deployment Rollback') {
            steps {
                script {
                    if (params.DEVELOPMENT_ROLLBACK) {
                       withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-crendentails-jenkinsadmin']]){
                            dir('infrastructure/development/') {
                                sh 'echo "=================Development Deployment Rollback=================="'
                                sh "aws cloudformation delete-stack --stack-name DeployDevelopmentStack --region 'us-east-1'"
                                sh "aws cloudformation wait stack-delete-complete --stack-name DeployDevelopmentStack --region 'us-east-1'"
                            }
                        }
                    }
                }
            }
        }

        stage('Staging Deployment Rollback') {
            steps {
                script {
                    if (params.STAGING_ROLLBACK) {
                       withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-crendentails-jenkinsadmin']]){
                            dir('infrastructure/staging/') {
                                sh 'echo "=================Staging Deployment Rollback=================="'
                                sh "aws cloudformation delete-stack --stack-name DeployStagingStack --region 'us-east-1'"
                                sh "aws cloudformation wait stack-delete-complete --stack-name DeployStagingStack --region 'us-east-1'"
                            }
                        }
                    }
                }
            }
        }

        stage('Production Deployment Rollback') {
            steps {
                script {
                    if (params.PRODUCTION_ROLLBACK) {
                       withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-crendentails-jenkinsadmin']]){
                            dir('infrastructure/production/') {
                                sh 'echo "=================Production Deployment Rollback=================="'
                                sh "aws cloudformation delete-stack --stack-name DeployProductionStack --region 'us-east-1'"
                                sh "aws cloudformation wait stack-delete-complete --stack-name DeployProductionStack --region 'us-east-1'"
                            }
                        }
                    }
                }
            }
        }
    }
}
```

When deploying CloudFormation templates that create or modify AWS Identity and Access Management (IAM) resources, such as roles, users, or policies, you must explicitly acknowledge that by providing special capabilities during the stack creation or update process.

There are two such capabilities:
1. `CAPABILITY_IAM` - This is required when your template creates or modifies IAM resources without specifying custom names. For example, if you're creating a role with a generated logical ID and letting AWS name it automatically, use this flag.
2. `CAPABILITY_NAMED_IAM` - This is required when your template creates or modifies IAM resources with explicitly defined names using the RoleName, UserName, or GroupName properties. CloudFormation needs your explicit consent since named IAM resources can persist across stack deletions and impact other stacks or services.

- `create-cfn-template-bucket.yaml` - will create bucket for cloudformation templates before provisioning

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'CFN for creating an S3 bucket for nested stack templates and applying a policy for access from CloudFormation and EC2 instances'

Parameters:
  S3BucketName:
    Type: String
    Default: 'ct-cfn-files-for-stack'

Resources:
#####################################
# S3 bucket for CFN nested stack templates
#####################################
  PipelineArtifactStoreS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref S3BucketName

  PipelineArtifactStoreS3BucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref PipelineArtifactStoreS3Bucket
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Sid: AllowS3AccessForJenkinsService
            Principal:
              Service:
                - cloudformation.amazonaws.com
                - ec2.amazonaws.com
            Effect: Allow
            Action: 
              - s3:GetObject
              - s3:GetObjectVersion
              - s3:PutObject
              - s3:ListBucket
            Resource: 
              - !Sub 'arn:${AWS::Partition}:s3:::${S3BucketName}/*'
              - !Sub 'arn:${AWS::Partition}:s3:::${S3BucketName}'
```

Each environment has three templates:
- `network.yml` — defines VPC, subnets, routing, etc.
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Networking Resources for the Environment

Parameters:
  Environment:
    Type: String

Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-vpc"

  MyInternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-igw"

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref MyVPC
      InternetGatewayId: !Ref MyInternetGateway

  MyPublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 10.0.1.0/24
      MapPublicIpOnLaunch: true
      AvailabilityZone: !Select [ 0, !GetAZs "us-east-1" ]
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-public-subnet"

  MyRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MyVPC
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-rtb"

  MyRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref MyRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref MyInternetGateway

  MySubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref MyPublicSubnet
      RouteTableId: !Ref MyRouteTable

  InstanceSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow SSH and HTTP
      VpcId: !Ref MyVPC
      GroupName: !Sub "${Environment}-sg"
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-instance-sg"

Outputs:
  EnvironmentName:
    Description: Environment of this deployment
    Value: !Ref Environment

  VPCId:
    Description: VPC ID
    Value: !Ref MyVPC

  MyPublicSubnet:
    Description: Public Subnet ID
    Value: !Ref MyPublicSubnet
    Export:
      Name: !Sub "${Environment}-public-subnet"

  InstanceSecurityGroup:
    Description: Security Group
    Value: !Ref InstanceSecurityGroup
    Export:
      Name: !Sub "${Environment}-instance-sg"
```
- `compute.yml` — provisions EC2 instance (web server) and related security groups
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: EC2 Auto Scaling Group with Launch Template and IAM Role for the Environment

Parameters:
  Environment:
    Type: String

  InstanceType:
    Description: EC2 instance type
    Type: String
    Default: t3.micro
    AllowedValues:
      - t2.micro
      - t2.small
      - t3.micro
      - t3.small
    ConstraintDescription: Must be a valid EC2 instance type.

  MyPublicSubnet:
    Type: String

  InstanceSecurityGroup:
    Type: String

Resources:
  EC2Role:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub "${Environment}-ec2-role"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: [ec2.amazonaws.com]
            Action: ['sts:AssumeRole']
      Path: "/"
      Policies:
        - PolicyName: !Sub "${Environment}-ec2-policy"
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action:
                  - ec2:Describe*
                  - ssm:*
                  - cloudwatch:PutMetricData
                Resource: "*"

  IAMInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      InstanceProfileName: !Sub "${Environment}-ec2-profile"
      Roles: [ !Ref EC2Role ]

  LaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateName: !Sub "${Environment}-launch-template"
      LaunchTemplateData:
        InstanceType: !Ref InstanceType
        KeyName: WorkshopKeyPair
        IamInstanceProfile:
          Name: !Ref IAMInstanceProfile
        ImageId: "{{resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64}}"
        SecurityGroupIds:
          - !Ref InstanceSecurityGroup
        UserData:
          Fn::Base64: 
            !Sub |
              #!/bin/bash
              yum update -y
              yum install -y httpd.x86_64
              systemctl start httpd.service
              systemctl enable httpd.service

              TOKEN=$(curl --request PUT "http://169.254.169.254/latest/api/token" --header "X-aws-ec2-metadata-token-ttl-seconds: 3600")

              instanceId=$(curl -s http://169.254.169.254/latest/meta-data/instance-id --header "X-aws-ec2-metadata-token: $TOKEN")
              instanceAZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone --header "X-aws-ec2-metadata-token: $TOKEN")
              privHostName=$(curl -s http://169.254.169.254/latest/meta-data/local-hostname --header "X-aws-ec2-metadata-token: $TOKEN")
              privIPv4=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4 --header "X-aws-ec2-metadata-token: $TOKEN")
              
              echo "<font face = "Verdana" size = "5">"                               > /var/www/html/index.html
              echo "<center><h1>AWS Linux VM - ${Environment} Environment </h1></center>"   >> /var/www/html/index.html
              echo "<center> <b>EC2 Instance Metadata</b> </center>"                  >> /var/www/html/index.html
              echo "<center> <b>Instance ID:</b> $instanceId </center>"               >> /var/www/html/index.html
              echo "<center> <b>AWS Availablity Zone:</b> $instanceAZ </center>"      >> /var/www/html/index.html
              echo "<center> <b>Private Hostname:</b> $privHostName </center>"        >> /var/www/html/index.html
              echo "<center> <b>Private IPv4:</b> $privIPv4 </center>"                >> /var/www/html/index.html
              echo "</font>"                                                          >> /var/www/html/index.html

  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      AutoScalingGroupName: !Sub "${Environment}-asg"
      VPCZoneIdentifier:
        - !Ref MyPublicSubnet
      LaunchTemplate:
        LaunchTemplateId: !Ref LaunchTemplate
        Version: !GetAtt LaunchTemplate.LatestVersionNumber
      MinSize: 1
      MaxSize: 3
      DesiredCapacity: 3
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-ec2-instance"
          PropagateAtLaunch: true

Outputs:
  EnvironmentName:
    Description: Environment of this deployment
    Value: !Ref Environment

```
- `root.yaml` — it use these templates via nested stacks and has defined parameters (development environment for example)
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Main Stack for the specified environment'

Parameters:
  TemplateUrlBase: 
    Type: String
    Default: 'https://ct-cfn-files-for-stack.s3.us-east-1.amazonaws.com/infrastructure'
  Environment:
    Description: List of available environments
    Type: String
    AllowedValues:
      - development
      - staging
      - production
    Default: development

Resources:
  NetworkStack:
    DeletionPolicy: Delete
    UpdateReplacePolicy: Retain
    Type: 'AWS::CloudFormation::Stack'
    Properties:
      TemplateURL: !Sub '${TemplateUrlBase}/${Environment}/network.yaml'
      Parameters:
        Environment: !Ref Environment

  ComputeStack:
    DeletionPolicy: Delete
    UpdateReplacePolicy: Retain
    Type: 'AWS::CloudFormation::Stack'
    Properties:
      TemplateURL: !Sub '${TemplateUrlBase}/${Environment}/compute.yaml'
      Parameters:
        Environment: !Ref Environment
        MyPublicSubnet: !GetAtt NetworkStack.Outputs.MyPublicSubnet
        InstanceSecurityGroup: !GetAtt NetworkStack.Outputs.InstanceSecurityGroup
```

### Step 3: Creation and initial Jenkins Pipeline Run

Since the pipeline needs to upload templates to S3 before CloudFormation can use them, the first run will:
- Clone the repo
- Create an S3 bucket (if not existing)
- Upload templates to it

Tip: On the first run, parameters like environment selection or action (provision/decommission) won’t show up unless configured via a seed job using JobDSL plugin.

![alt text](/images/create_pipeline_1.png)

![alt text](/images/create_pipeline_2.png)

![alt text](/images/build_initial_1.png)

![alt text](/images/build_initial_2.png)

### Step 4: Rerun with Parameters to Provision Infrastructure

Run the pipeline again (Build with parameter option), this time selecting appropriate parameters. Watch Jenkins orchestrate the creation of AWS resources!

![alt text](/images/build_with_params_1.png)

![alt text](/images/build_with_params_2.png)

![alt text](/images/build_with_params_3.png)

![alt text](/images/build_with_params_4.png)

Pipeline Completion deploying all 3 environments:
![alt text](/images/build_complete_1.png)

![alt text](/images/build_complete_2.png)

![alt text](/images/build_complete_3.png)

![alt text](/images/build_complete_4.png)

### Step 5: Decommission Infrastructure
The same pipeline also supports deletion of the provisioned stack using the rollback paramters.

You’ll see the CloudFormation stack being deleted and AWS resources getting removed from the environment.
![alt text](/images/build_rollback_1.png)

![alt text](/images/build_rollback_2.png)

![alt text](/images/build_rollback_3.png)

### Conclusion
By leveraging CloudFormation, Jenkins, and Git, we created an automated, parameter-driven deployment pipeline to manage infrastructure across multiple AWS environments. This approach promotes IaC, CI/CD practices, and minimizes manual errors.

### References
Github Repo for Jenkin Server: https://github.com/chinmayto/terraform-aws-jenkins-server
Github Repo for Cloudformation with Jenkins: https://github.com/chinmayto/cloudformation-devops-with-jenkins
Creating Jenkins Pipelines: https://www.jenkins.io/doc/book/pipeline/