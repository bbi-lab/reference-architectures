# Manual resource creation

This manual procedure should be followed after creating the network environment.

All steps should be performed in the `us-west-2` (Oregon) region except where noted.

## Parameters

This guide does not include passwords or other secrets, but it does assume the following account-specific parameters:

- {aws-account-id}: The ID of your AWS account, typically a 12-digit numeral
- {organization}: The organization name, in a kebab-cased form suitable for use in identifiers
- {project}: The project name, in a kebab-cased form suitable for use in identifiers
- {postgres-password-prod}: The password for the production database's root (`postgres`) user

## 1. EC2 key pair for bastion hosts

AWS console > EC2 > Key pairs > Create key pair

- Key pair
  - **Name:**: `{project}-prod-bastion-1-ec2-root-keypair`
  - **Key pair type:** RSA
  - **Private key format:** .pem
  - Tags
    - *project* = `{project}`

Save the private key when you generate it. It cannot be downloaded again.

This key will provide access (as the root user, `ec2-user`) to the bastion host.

TODO Rename in future to `{project}-bastion-keypair`.

## 2. EC2 security group for bastion hosts

AWS console > EC2 > Security groups > Create security group

- Create security group
  - Basic details
    - **Security group name:** `{project}-bastion-sg`
    - **Description:** `Security group for {project} bastion servers`
    - **VPC:**- {project}-mgmt-vpc
  - Tags
    - *Name* = `{project}-bastion-sg`
    - *project* = `{project}`

## 3. EC2 host (bastion)

AWS console > EC2 > Instances > Launch instances

- Launch an instance
  - Name and tags
    - Tags
      - *Name* = `{project}-bastion-ec2-1` (Apply to resource types: Instances, Volumes, Network interfaces)
      - *project* = `{project}` (Apply to resource types: Instances, Volumes, Network interfaces)
  - Application and OS Images
    - **Quick Start:** Amazon Linux
    - **Amazon Machine Image (AMI):** Amazon Linux 2023 AMI
    - **Architecture:** 64-bit (Arm)
  - **Instance type:** t4g.nano
  - Key pair
    - **Key pair name:** `{project}-prod-bastion-1-ec2-root-keypair`
  - Network settings
    - **VPC:** {project}-mgmt-vpc
    - **Subnet:** {project}-mgmt-public-subnet-1
    - **Auto-assign public IP:** Enable
    - **Firewall (security groups):** Select existing security group
    - **Security group:** {project}-bastion-sg

After creating the bastion host, use SSH to log in as ec2-user, and edit /etc/ssh/sshd_config. Find the lines below and un-comment three of them, as shown:

```
AllowAgentForwarding yes
AllowTcpForwarding yes
#GatewayPorts no
X11Forwarding no
```

## 4. ECR repository for {project}-api Docker images

AWS console > Amazon Elastic Container Registry > Private registry > Repositories > Create repository

- Create repository
  - **Visibility settings**: Private
  - **Repository name:** `{project}-api`

## 5. S3 bucket for application and server logs

AWS console > Amazon S3 > Buckets > Create bucket

- Create bucket
  - General configuration
    - **AWS region:** us-west-2
    - **Bucket type:** General purpose
    - **Bucket name:** `{organization}-{project}-app-logs`
  - **Object Ownership**: ACLs disabled
  - Block Public Access settings for this bucket
    - **Block all public access:** Yes
  - Bucket Versioning
    - **Bucket Versioning:** Disable
  - Tags
    - *project* = `{project}`
  - Default encryption
    - **Encryption type:** Server-side encryption with Amazon S3 managed keys (SSE-S3)
    - **Bucket Key:** Enable
  - Advanced settings
    - **Object Lock:** Disable

## 6. IAM policy granting read access to ECR

AWS console > Identity and Access Management (IAM) > Access management > Policies > Create policy

- Specify permissions
  - **Permissions:**
    ```
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "VisualEditor0",
                "Effect": "Allow",
                "Action": "ecr:GetAuthorizationToken",
                "Resource": "*"
            },
            {
                "Sid": "VisualEditor1",
                "Effect": "Allow",
                "Action": [
                    "ecr:GetDownloadUrlForLayer",
                    "ecr:BatchGetImage",
                    "ecr:BatchCheckLayerAvailability"
                ],
                "Resource": [
                    "arn:aws:ecr:us-west-2:{aws-account-id}:repository/{project}-api"
                ]
            }
        ]
    }
    ```
  - Review and create
    - Policy details
      - **Policy name:** `{project}-api-ecr-read`
      - **Description:** `Permission to read the {project}-api ECR repository`
    - Tags
      - *project* = `{project}`

## 7a. IAM policy granting log write access (production)

AWS console > Identity and Access Management (IAM) > Access management > Policies > Create policy

- Specify permissions
  - **Permissions:**
    ```
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Sid": "VisualEditor0",
          "Effect": "Allow",
          "Action": [
            "s3:PutObject",
            "s3:GetObject",
            "s3:ListBucket"
          ],
          "Resource": [
            "arn:aws:s3:::{organization}-{project}-app-logs/*",
            "arn:aws:s3:::{organization}-{project}-app-logs"
          ]
        },
        {
          "Effect": "Allow",
          "Action": [
            "logs:CreateLogGroup",
            "logs:CreateLogStream",
            "logs:PutLogEvents",
            "logs:DescribeLogStreams"
          ],
          "Resource": [
            "arn:aws:logs:*:*:*"
          ]
        }
      ]
    }
    ```
  - Review and create
    - Policy details
      - **Policy name:** `{project}-prod-app-logging`
      - **Description:** `Permission to write {project} application logs in production`
    - Tags
      - *project* = `{project}`
      - *environment* = `production`

## 7b. IAM policy granting log write access (staging)

AWS console > Identity and Access Management (IAM) > Access management > Policies > Create policy

- Specify permissions
  - **Permissions:**
    ```
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Sid": "VisualEditor0",
          "Effect": "Allow",
          "Action": [
            "s3:PutObject",
            "s3:GetObject",
            "s3:ListBucket"
          ],
          "Resource": [
            "arn:aws:s3:::{organization}-{project}-app-logs/*",
            "arn:aws:s3:::{organization}-{project}-app-logs"
          ]
        },
        {
          "Effect": "Allow",
          "Action": [
            "logs:CreateLogGroup",
            "logs:CreateLogStream",
            "logs:PutLogEvents",
            "logs:DescribeLogStreams"
          ],
          "Resource": [
            "arn:aws:logs:*:*:*"
          ]
        }
      ]
    }
    ```
  - Review and create
    - Policy details
      - **Policy name:** `{project}-staging-app-logging` <mark>Differs from production</mark>
      - **Description:** `Permission to write {project} application logs for staging environment` <mark>Differs from production</mark>
    - Tags
      - *project* = `{project}`
      - *environment* = `staging` <mark>Differs from production</mark>

## 8. IAM role for API servers (production)

AWS console > Identity and Access Management (IAM) > Access management > Roles > Create role

- Select trusted entity
  - **Trusted entity type:** AWS service
  - **Use case:** EC2
- Add permissions
  - Permissions policies
    - AmazonSSMManagedInstanceCore
    - AWSElasticBeanstalkWebTier
    - {project}-api-ecr-read
    - {project}-prod-app-logging
- Role details
  - **Role name:** {project}-prod-ec2-role
- Add tags
  - *project* = `{project}`
  - *environment* = `production`

## 8. IAM role for API servers (staging)

AWS console > Identity and Access Management (IAM) > Access management > Roles > Create role

- Select trusted entity
  - **Trusted entity type:** AWS service
  - **Use case:** EC2
- Add permissions
  - Permissions policies
    - AmazonSSMManagedInstanceCore
    - AWSElasticBeanstalkWebTier
    - {project}-api-ecr-read
    - {project}-staging-app-logging <mark>Differs from production</mark>
- Role details
  - **Role name:** {project}-staging-ec2-role <mark>Differs from production</mark>
- Add tags
  - *project* = `{project}`
  - *environment* = `staging` <mark>Differs from production</mark>

## 9a. Security group for API servers (production)

AWS console > EC2 > Security Groups > Create security group

- Create security group
  - Basic details
    - **Security group name:** `{project}-prod-api-sg`
    - **Description:** `Security group for {project} production API servers`
    - **VPC:**- {project}-prod-vpc
  - Inbound rules
    - [TODO These will be added later.]
  - Outbound rules
    - [Leave default in place, allowing all outbound traffic.]
  - Tags
    - *project* = `{project}`
    - *environment* = `production`

## 9b. Security group for API servers (staging)

AWS console > EC2 > Security Groups > Create security group

- Create security group
  - Basic details
    - **Security group name:** `{project}-staging-api-sg` <mark>Differs from production</mark>
    - **Description:** `Security group for {project} staging API servers` <mark>Differs from production</mark>
    - **VPC:**- {project}-staging-vpc <mark>Differs from production</mark>
  - Inbound rules
    - [TODO These will be added later.]
  - Outbound rules
    - [Leave default in place, allowing all outbound traffic.]
  - Tags
    - *Name* = `{project}-staging-api-sg` <mark>Differs from production</mark>
    - *project* = `{project}`
    - *environment* = `staging` <mark>Differs from production</mark>

## 9.5b. Security group for worker servers (staging)

AWS console > EC2 > Security Groups > Create security group

- Create security group
  - Basic details
    - **Security group name:** `{project}-staging-worker-sg` <mark>Differs from production</mark>
    - **Description:** `Security group for {project} staging worker servers` <mark>Differs from production</mark>
    - **VPC:**- {project}-staging-vpc <mark>Differs from production</mark>
  - Inbound rules
    - [TODO These will be added later.]
  - Outbound rules
    - [Leave default in place, allowing all outbound traffic.]
  - Tags
    - *Name* = `{project}-staging-api-sg` <mark>Differs from production</mark>
    - *project* = `{project}`
    - *environment* = `staging` <mark>Differs from production</mark>

## 10a. SSH certificate for api.{project}.org

AWS console > AWS Certificate Manager (ACM) > List certificates > Request

- Request certificaate
  - **Certificate type**: Request a public certificate
- Request public certificate
  - Domain names
    - **Fully qualified domain names:** `api.{project}.org`
  - **Validation method:** DNS validation
  - **Key algorithm:** RSA 2048
  - Tags
    - *project* = `{project}`
    - *environment* = `production`

Return to the list of certificates, and click "Create records in Route 53." Wait for the certificate to be issued.

(If DNS records are not being managed by Route 53, export the CNAME record and import it to the DNS registry.)

## 10b. SSH certificate for api.staging.{project}.org

AWS console > AWS Certificate Manager (ACM) > List certificates > Request

- Request certificaate
  - **Certificate type**: Request a public certificate
- Request public certificate
  - Domain names
    - **Fully qualified domain names:** `api.staging.{project}.org` <mark>Differs from production</mark>
  - **Validation method:** DNS validation
  - **Key algorithm:** RSA 2048
  - Tags
    - *project* = `{project}`
    - *environment* = `staging` <mark>Differs from production</mark>

Return to the list of certificates, and click "Create records in Route 53." Wait for the certificate to be issued.

(If DNS records are not being managed by Route 53, export the CNAME record and import it to the DNS registry.)

## 11a. SSH certificate for {project}.org

Create this certificate in the `us-east-1` (Virginia) region, because it will be used by CloudFront.

- Request certificaate
  - **Certificate type**: Request a public certificate
- Request public certificate
  - Domain names
    - **Fully qualified domain names:** `{project}.org`, `www.{project}.org`
  - **Validation method:** DNS validation
  - **Key algorithm:** RSA 2048
  - Tags
    - *project* = `{project}`
    - *environment* = `production`

Return to the list of certificates, and click "Create records in Route 53." Wait for the certificate to be issued.

(If DNS records are not being managed by Route 53, export the CNAME record and import it to the DNS registry.)

## 11b. SSH certificate for staging.{project}.org

Create this certificate in the `us-east-1` (Virginia) region, because it will be used by CloudFront.

- Request certificaate
  - **Certificate type**: Request a public certificate
- Request public certificate
  - Domain names
    - **Fully qualified domain names:** `staging.{project}.org`, `www.staging.{project}.org` <mark>Differs from production</mark>
  - **Validation method:** DNS validation
  - **Key algorithm:** RSA 2048
  - Tags
    - *project* = `{project}`
    - *environment* = `staging` <mark>Differs from production</mark>

Return to the list of certificates, and click "Create records in Route 53." Wait for the certificate to be issued.

(If DNS records are not being managed by Route 53, export the CNAME record and import it to the DNS registry.)

## 12a. EC2 key pair for API servers (production)

AWS console > EC2 > Key pairs > Create key pair

- Key pair
  - **Name:**: `{project}-prod-eb-root-keypair`
  - **Key pair type:** RSA
  - **Private key format:** .pem
  - Tags
    - *project* = `{project}`
    - *environment* = `production`

This key will provide access (as the root user, `ec2-user`) to the API servers, which are EC2 instances managed by Elastic Beanstalk.

Save the private key when you generate it. It cannot be downloaded again.

## 12b. EC2 key pair for API servers (staging)

AWS console > EC2 > Key pairs > Create key pair

- Key pair
  - **Name:**: `{project}-staging-eb-root-keypair` <mark>Differs from production</mark>
  - **Key pair type:** RSA
  - **Private key format:** .pem
  - Tags
    - *project* = `{project}`
    - *environment* = `staging` <mark>Differs from production</mark>

This key will provide access (as the root user, `ec2-user`) to the API servers, which are EC2 instances managed by Elastic Beanstalk.

Save the private key when you generate it. It cannot be downloaded again.

## 13a. Elastic Beanstalk API environment (production)

AWS console > Elastic Beanstalk > Create environment

- Configure environment
  - **Environment tier:** Web server environment
  - Application information
    - **Application name:** `{project}-prod-api-app`
    - Application tags
      - *project* = `{project}`
      - *environment* = `production`
  - Environment information
    - **Application name:** `{project}-prod-api-env`
    - **Domain:** `{project}-api`.us-west-2.elasticbeanstalk.com
  - Platform
    - **Platform type:** Managed platform
    - **Platform:** Docker
    - **Platform branch:** Docker running on 64bit Amazon Linux 2023
    - **Platform version:** 4.1.1
  - **Application code:** Sample application
  - **Presets:** High availability
- Configure service access
  - **Service role:** aws-elasticbeanstalk-service-role
  - **EC2 key pair:** {project}-prod-eb-root-keypair
  - **EC2 instance profile:** {project}-prod-ec2-role
- Set up networking, database, and tags
  - VPC
    - **VPC:** {project}-prod-vpc
  - Instance settings
    - **Public IP address:** [no]
    - **Instance subnets:** {project}-prod-private-subnet-1, {project}-prod-private-subnet-2
  - Database
    - (No database)
  - Tags
    - *project* = `{project}`
    - *environment* = `production`
- Configure instance traffic and scaling
  - Instances
    - Root volume (boot device)
      - **Root volume type:** General Purpose (SSD)
      - **Size:** 25 GB
    - Amazon CloudWatch monitoring
      - **Monitoring interval:** 5 minute
    - Instance metadata service (IMDS)
      - **IMDSv1:** Deactivated
    - EC2 security groups
      - **Security groups:** {project}-prod-api-sg
  - Capacity
    - Auto scaling group
      - **Environment type:** Load balanced
      - Instances
        - **Min:** 1
        - **Max:** 1
        - **Fleet composition:** On-Demand instances
        - **Architecture:** x86_64
        - **Instance types:** t3.small
        - **AMI ID:** ami-0185e1859958a8621 [default]
  - Load balancer network settings
    - **Visibility:** Public
    - **Load balancer subnets:** {project}-prod-public-subnet-1, {project}-prod-public-subnet-2
  - **Load balancer type:** Application load balancer, Dedicated
  - Listeners
    - (For now, leave the default listener on port 80.)
  - Processes
    - (For now, leave the default process on port 80.)
  - Rules
    - (None)
  - Log files access
    - **Store logs:** Enabled
    - **S3 bucket:** {organization}-{project}-app-logs
    - **Prefix:** `prod-api/`
- Configure updates, monitoring, and logging
  - Monitoring
    - Health reporting
      - **System:** Enhanced
  - Managed platform updates
    - **Managed updates:** Activated
    - **Weekly update window:** Sunday at 07:05 UTC
    - **Update level:** Minor and patch
  - Rolling updates and deployment
    - Application deployments
      - **Deployment policy:** Rolling
      - **Batch size type:** Percentage
      - **Deployment batch size:** 30%
    - Configuration updates
      - **Rolling update type:** Rolling based on Time
      - **Batch size:** 1
      - **Minimum capacity:** 0
  - Platform software
    - Container options
      - **Proxy server:** Nginx
      - Instance log streaming to CloudWatch
        - **Log streaming:** Activated
        - **Retention:** 3653
        - **Lifecycle:** Keep logs after terminating environment

## 13b. Elastic Beanstalk API environment (staging)

AWS console > Elastic Beanstalk > Create environment

- Configure environment
  - **Environment tier:** Web server environment
  - Application information
    - **Application name:** `{project}-staging-api-app` <mark>Differs from production</mark>
    - Application tags
      - *project* = `{project}`
      - *environment* = `staging` <mark>Differs from production</mark>
  - Environment information
    - **Application name:** `{project}-staging-api-env` <mark>Differs from production</mark>
    - **Domain:** `{project}-staging-api`.us-west-2.elasticbeanstalk.com <mark>Differs from production</mark>
  - Platform
    - **Platform type:** Managed platform
    - **Platform:** Docker
    - **Platform branch:** Docker running on 64bit Amazon Linux 2023
    - **Platform version:** 4.2.2 <mark>Differs from production</mark>
  - **Application code:** Sample application
  - **Presets:** High availability
- Configure service access
  - **Service role:** aws-elasticbeanstalk-service-role
  - **EC2 key pair:** {project}-staging-eb-root-keypair <mark>Differs from production</mark>
  - **EC2 instance profile:** {project}-staging-ec2-role <mark>Differs from production</mark>
- Set up networking, database, and tags
  - VPC
    - **VPC:** {project}-staging-vpc <mark>Differs from production</mark>
  - Instance settings
    - **Public IP address:** [no]
    - **Instance subnets:** {project}-staging-private-subnet-1, {project}-staging-private-subnet-2 <mark>Differs from production</mark>
  - Database
    - (No database)
  - Tags
    - *project* = `{project}`
    - *environment* = `staging` <mark>Differs from production</mark>
- Configure instance traffic and scaling
  - Instances
    - Root volume (boot device)
      - **Root volume type:** General Purpose (SSD)
      - **Size:** 25 GB
    - Amazon CloudWatch monitoring
      - **Monitoring interval:** 5 minute
    - Instance metadata service (IMDS)
      - **IMDSv1:** Deactivated
    - EC2 security groups
      - **Security groups:** {project}-staging-api-sg <mark>Differs from production</mark>
  - Capacity
    - Auto scaling group
      - **Environment type:** Load balanced
      - Instances
        - **Min:** 1
        - **Max:** 1
        - **Fleet composition:** On-Demand instances
        - **Architecture:** x86_64
        - **Instance types:** t3.small
        - **AMI ID:** ami-027f864c59adb48ec [default] <mark>Differs from production</mark>
  - Load balancer network settings
    - **Visibility:** Public
    - **Load balancer subnets:** {project}-staging-public-subnet-1, {project}-staging-public-subnet-2 <mark>Differs from production</mark>
  - **Load balancer type:** Application load balancer, Dedicated
  - Listeners
    - (For now, leave the default listener on port 80.)
  - Processes
    - (For now, leave the default process on port 80.)
  - Rules
    - (None)
  - Log files access
    - **Store logs:** Enabled
    - **S3 bucket:** {organization}-{project}-app-logs
    - **Prefix:** `staging-api` <mark>Differs from production</mark>
- Configure updates, monitoring, and logging
  - Monitoring
    - Health reporting
      - **System:** Enhanced
  - Managed platform updates
    - **Managed updates:** Activated
    - **Weekly update window:** Sunday at 07:05 UTC
    - **Update level:** Minor and patch
  - Rolling updates and deployment
    - Application deployments
      - **Deployment policy:** Rolling
      - **Batch size type:** Percentage
      - **Deployment batch size:** 30%
    - Configuration updates
      - **Rolling update type:** Rolling based on Time
      - **Batch size:** 1
      - **Minimum capacity:** 0
  - Platform software
    - Container options
      - **Proxy server:** Nginx
      - Instance log streaming to CloudWatch
        - **Log streaming:** Activated
        - **Retention:** 7 <mark>Differs from production</mark>
        - **Lifecycle:** Keep logs after terminating environment

## 13.1b Amazon SQS worker dead-letter queue (staging)

AWS console > Amazon SQS > Queues > Create queue

- Create queue
  - Details
    - **Type**: Standard
    - **Name**: `{project}-staging-dead-queue` <mark>Differs from production</mark>
  - Configuration
    - **Visibility timeout**: 15 minutes
    - **Message retention period**: 14 days
    - **Delivery delay**: 0 seconds
    - **Maximum message size**: 256 KB
    - **Receive message wait time**: 0 seconds
  - Encryption
    - **Server-side encryption**: Enabled
    - **Encryption key type**: Amazon SQS key (SSE-SQS)
  - Access policy
    - **Choose method**: Advanced
    - **[JSON policy]**: <mark>Differs from production</mark>
```
{
  "Version": "2012-10-17",
  "Id": "__default_policy_ID",
  "Statement": [
    {
      "Sid": "__owner_statement",
      "Effect": "Allow",
      "Principal": {
        "AWS": "{aws-account-id}"
      },
      "Action": [
        "SQS:*"
      ],
      "Resource": "arn:aws:sqs:us-west-2:{aws-account-id}:{project}-staging-queue"
    }
  ]
}
```
  - **Redrive allow policy**: Disabled
  - **Dead-letter queue**: Disabled
  - Tags
    - *project* = `{project}`
    - *environment* = `staging` <mark>Differs from production</mark>

## 13.2b Amazon SQS worker queue (staging)

AWS console > Amazon SQS > Queues > Create queue

- Create queue
  - Details
    - **Type**: Standard
    - **Name**: `{project}-staging-queue` <mark>Differs from production</mark>
  - Configuration
    - **Visibility timeout**: 15 minutes
    - **Message retention period**: 14 days
    - **Delivery delay**: 0 seconds
    - **Maximum message size**: 256 KB
    - **Receive message wait time**: 0 seconds
  - Encryption
    - **Server-side encryption**: Enabled
    - **Encryption key type**: Amazon SQS key (SSE-SQS)
  - Access policy
    - **Choose method**: Advanced
    - **[JSON policy]**: <mark>Differs from production</mark>
```
{
  "Version": "2012-10-17",
  "Id": "__default_policy_ID",
  "Statement": [
    {
      "Sid": "__owner_statement",
      "Effect": "Allow",
      "Principal": {
        "AWS": "{aws-account-id}"
      },
      "Action": [
        "SQS:*"
      ],
      "Resource": "arn:aws:sqs:us-west-2:{aws-account-id}:{project}-staging-queue"
    }
  ]
}
```
  - **Redrive allow policy**: Disabled
  - **Dead-letter queue**: Disabled
  - Tags
    - *project* = `{project}`
    - *environment* = `staging` <mark>Differs from production</mark>

## 14a. Elastic Beanstalk worker environment (production)

## 14b. Elastic Beanstalk worker environment (staging)

AWS console > Elastic Beanstalk > Create environment

- Configure environment
  - **Environment tier:** Worker environment
  - Application information
    - **Application name:** `{project}-staging-worker-app` <mark>Differs from production</mark>
    - Application tags
      - *project* = `{project}`
      - *environment* = `staging` <mark>Differs from production</mark>
  - Environment information
    - **Application name:** `{project}-staging-worker-env` <mark>Differs from production</mark>
  - Platform
    - **Platform type:** Managed platform
    - **Platform:** Docker
    - **Platform branch:** Docker running on 64bit Amazon Linux 2023
    - **Platform version:** 4.3.0
  - **Application code:** Sample application
  - **Presets:** Single instance
- Configure service access
  - **Service role:** aws-elasticbeanstalk-service-role
  - **EC2 key pair:** {project}-staging-eb-root-keypair <mark>Differs from production</mark>
  - **EC2 instance profile:** {project}-staging-ec2-role <mark>Differs from production</mark>
- Modify worker
  - Queue
    - **Worker queue**: [URL of queue named {project}-staging-queue] <mark>Differs from production</mark>
  - Messages
    - **HTTP path**: `/worker`
    - **MIME type**: application/json
    - **HTTP connections**: 50
    - **Visibility timeout**: 900 seconds
    - **Error visibility timeout**: 30 seconds
  - Advanced options
    - **Max retries**: 10
    - **Connection timeout**: 5
    - **Inactivity timeout**: 299
    - **Retention period**: 345600
- Set up networking, database, and tags
  - VPC
    - **VPC:** {project}-staging-vpc <mark>Differs from production</mark>
  - Instance settings
    - **Public IP address:** [no]
    - **Instance subnets:** {project}-staging-private-subnet-1, {project}-staging-private-subnet-2 <mark>Differs from production</mark>
  - Database
    - (No database)
  - Tags
    - *project* = `{project}`
    - *environment* = `staging` <mark>Differs from production</mark>
- Configure instance traffic and scaling
  - Instances
    - Root volume (boot device)
      - **Root volume type:** General Purpose (SSD)
      - **Size:** 25 GB
    - Amazon CloudWatch monitoring
      - **Monitoring interval:** 5 minute
    - Instance metadata service (IMDS)
      - **IMDSv1:** Deactivated
    - EC2 security groups
      - **Security groups:** {project}-staging-worker-sg <mark>Differs from production</mark>
  - Capacity
    - Auto scaling group
      - **Environment type:** Single instance
      - Instances
        - **Fleet composition:** On-Demand instances
        - **Architecture:** x86_64
        - **Instance types:** t3.small
        - **AMI ID:** ami-0407c266f70e40fb4 [default]
- Configure updates, monitoring, and logging
  - Monitoring
    - Health reporting
      - **System:** Enhanced
  - Managed platform updates
    - **Managed updates:** Activated
    - **Weekly update window:** Sunday at 07:05 UTC
    - **Update level:** Minor and patch
  - Rolling updates and deployment
    - Application deployments
      - **Deployment policy:** All at once
      - **Batch size type:** Percentage
    - Configuration updates
      - **Rolling update type:** Deactivated
    - Deployment preferences
      - **Ignore health check**: True
      - **Health threshold**: Ok
      - **Command timeout**: 600 seconds
  - Platform software
    - Container options
      - **Proxy server:** Nginx
      - Instance log streaming to CloudWatch
        - **Log streaming:** Activated
        - **Retention:** 7 <mark>Differs from production</mark>
        - **Lifecycle:** Keep logs after terminating environment

## 15. Route 53 DNS entries (TODO)

## 16. Elastic Beanstalk environment configuration changes (TODO)

## 17a. S3 bucket for the UI (production)

AWS console > Amazon S3 > Create bucket

- Create bucket
  - General configuration
    - **AWS Region**: us-west-2
    - **Bucket type**: General purpose
    - **Bucket name**: `{organization}-{project}-prod-ui`
  - **Object Ownership**: ACLs disabled
  - Block Public Access settings for this bucket
    - [All disabled, allowing public access]
  - Bucket Versioning
    - **Bucket Versioning**: Disable
  - Tags
    - *project* = `{project}`
    - *environment* = `prod`
  - Default encryption
    - **Encryption type**: Server-side encryption with Amazon S3-managed keys
    - **Bucket Key**: Enable

When prompted, acknowledge the consequences of allowing public access.

Choose the bucket and navigate to Permissions. Add a bucket policy:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::{organization}-{project}-prod-ui/*"
        }
    ]
}
```

## 17b. S3 bucket for the UI (staging)

AWS console > Amazon S3 > Create bucket

- Create bucket
  - General configuration
    - **AWS Region**: us-west-2
    - **Bucket type**: General purpose
    - **Bucket name**: `{organization}-{project}-staging-ui` <mark>Differs from production</mark>
  - **Object Ownership**: ACLs disabled
  - Block Public Access settings for this bucket
    - [All disabled, allowing public access]
  - Bucket Versioning
    - **Bucket Versioning**: Disable
  - Tags
    - *project* = `{project}`
    - *environment* = `staging` <mark>Differs from production</mark>
  - Default encryption
    - **Encryption type**: Server-side encryption with Amazon S3-managed keys
    - **Bucket Key**: Enable

When prompted, acknowledge the consequences of allowing public access.

Choose the bucket and navigate to Permissions. Add a bucket policy:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::{organization}-{project}-staging-ui/*"
        }
    ]
}
```

## 18a. CloudFront distribution (production)

AWS console > CloudFront > Create distribution

- Create distribution
  - Origin
    - **Origin domain**: {organization}-{project}-prod-ui.s3.us-west-2.amazonaws.com
    - **Origin path**: [blank]
    - **Name**: `{organization}-{project}-prod-ui.s3.us-west-2.amazonaws.com`
    - **Origin access**: Public
    - **Enable Origin Shield**: No
  - Default cache behavior
    - **Path pattern**: Default (*)
    - **Compress objects automatically**: Yes
    - Viewer
      - **Viewer protocol policy**: Redirect HTTP to HTTPS
      - **Allowed HTTP methods**: GET, HEAD
      - **Restrict viewer access**: No
    - **Cache key and origin requests**: Cache policy and origin request policy
  - **Web Application Firewall (WAF)**: Enable security protections
    - **Use monitor mode**: On
  - Settings
    - **Price class**: Use all edge locations
    - **Alternate domain names (CNAME)**: `{project}.org`, `www.{project}.org`
    - **Custom SSL certificate**: {project}.org
    - **Legacy clients support**: Off
    - **Security policy**: TLSv1.2_2021
    - **Supported HTTP versions**: HTTP/2
    - **Default root object**: `index.html`
    - **Standard logging**: Off
    - **IPv6**: On

## 18b. CloudFront distribution (staging)

AWS console > CloudFront > Create distribution

- Create distribution
  - Origin
    - **Origin domain**: {organization}-{project}-staging-ui.s3.us-west-2.amazonaws.com <mark>Differs from production</mark>
    - **Origin path**: [blank]
    - **Name**: `{organization}-{project}-staging-ui.s3.us-west-2.amazonaws.com` <mark>Differs from production</mark>
    - **Origin access**: Public
    - **Enable Origin Shield**: No
  - Default cache behavior
    - **Path pattern**: Default (*)
    - **Compress objects automatically**: Yes
    - Viewer
      - **Viewer protocol policy**: Redirect HTTP to HTTPS
      - **Allowed HTTP methods**: GET, HEAD
      - **Restrict viewer access**: No
    - **Cache key and origin requests**: Cache policy and origin request policy
  - **Web Application Firewall (WAF)**: Do not enable security protections <mark>Differs from production</mark>
  - Settings
    - **Price class**: Use all edge locations
    - **Alternate domain names (CNAME)**: `staging.{project}.org`, `www.staging.{project}.org` <mark>Differs from production</mark>
    - **Custom SSL certificate**: staging.{project}.org <mark>Differs from production</mark>
    - **Legacy clients support**: Off
    - **Security policy**: TLSv1.2_2021
    - **Supported HTTP versions**: HTTP/2
    - **Default root object**: `index.html`
    - **Standard logging**: Off
    - **IPv6**: On

## 19a. RDS subnet group (production)

AWS console > Amazon RDS > Subnet groups > Create DB subnet group

- Subnet group details
  - **Name:** `{project}-prod-rds-subnetgroup`
  - **Description:** `RDS subnet group for {project} production DB`
  - **VPC:** {project}-prod-vpc
- Add subnets
  - **Availability zones:** us-west-2b, us-west-2c
  - Select Subnet IDs of {project}-prod-private-subnet-1, {project}-prod-private-subnet-2

## 19b. RDS subnet group (staging)

AWS console > Amazon RDS > Subnet groups > Create DB subnet group

- Subnet group details
  - **Name:** `{project}-staging-rds-subnetgroup` <mark>Differs from production</mark>
  - **Description:** `RDS subnet group for {project} staging DB` <mark>Differs from production</mark>
  - **VPC:** {project}-staging-vpc <mark>Differs from production</mark>
- Add subnets
  - **Availability zones:** us-west-2b, us-west-2c
  - Select Subnet IDs of {project}-staging-private-subnet-1, {project}-staging-private-subnet-2 <mark>Differs from production</mark>

## 20a. EC2 security group for the database (production)

AWS console > EC2 > Security groups > Create security group

- Create security group
  - Basic details
    - **Security group name:** `{project}-prod-db-sg`
    - **Description:** `Security group for {project} production DB`
    - **VPC:**- {project}-prod-vpc
  - Tags
    - *Name* = `{project}-prod-db-sg`
    - *project* = `{project}`
    - *environment* = `production`

Add inbound rules allowing TCP traffic on port 5432 (PostgreSQL) from two sources:
- The bastion host's IP address, as a CIDR block (i.e. the IP address with the suffix `/32`), with the description `Bastion`
- And the security group `{project}-prod-api-sg`.

## 20b. EC2 security group for the database (staging)

AWS console > EC2 > Security groups > Create security group

- Create security group
  - Basic details
    - **Security group name:** `{project}-staging-db-sg` <mark>Differs from production</mark>
    - **Description:** `Security group for {project} staging DB` <mark>Differs from production</mark>
    - **VPC:**- {project}-staging-vpc <mark>Differs from production</mark>
  - Tags
    - *Name* = `{project}-staging-db-sg`
    - *project* = `{project}`
    - *environment* = `staging` <mark>Differs from production</mark>

Add inbound rules allowing TCP traffic on port 5432 (PostgreSQL) from two sources:
- The bastion host's IP address, as a CIDR block (i.e. the IP address with the suffix `/32`), with the description `Bastion`
- And the security group `{project}-staging-api-sg`. <mark>Differs from production</mark>

## 21a. RDS database (production)

AWS console > Amazon RDS > Create database

- Create database
  - **Choose a database creation method:** Standard create
  - Engine options
    - **Engine type:** PostgreSQL
    - **Engine Version:** PostgreSQL 15.4-R3
  - **Templates**:** Production
  - **Availability and durability:** Single DB instance
  - Settings
    - **DB instance identifier:** `{project}-prod-db`
    - Credentials Settings
      - **Master username:** postgres
      - **Master password:** {postgres-password-prod}
  - Instance configuration
    - **DB instance class:** db.m6g.large
  - Storage
    - **Storage type:** General Purpose SSD (gp3)
    - **Allocated storage:** 200 GiB
  - Storage autoscaling
    - **Enable storage autoscaling:** Yes
    - **Maximum storage threshold:** 1000 GiB
  - Connectivity
    - **Compute resource:** Don't connect to an EC2 compute resource
    - **Network type:** IPv4
    - **Virtual private cloud (VPC):** {project}-prod-vpc
    - **DB subnet group:** {project}-prod-rds-subnetgroup
    - **Public access:** No
   - **VPC security group:** {project}-prod-db-sg
    - **Availability zone:** us-west-2c
    - **RDS Proxy:** Yes
    - **Certificate authority:** rds-ca-2019
- **Database authentication:** Password and IAM database authentication
- Monitoring
  - **Turn on Performance Insights:** Yes
  - **Retention period for Performance Insights:** 7 days
  - **AWS KMS key:** (default) aws/rds
  - **Turn on DevOps Guru:** No
- Additional configuration
  - Database options
    - **Initial database name:** *[leave blank]*
    - **DB parameter group:** default.postgres15
    - Backup
      - **Enable automated backups:** Yes
      - **Backup retention period:** 7 days
      - **Backup window:** No preference
      - **Copy tags to snapshots:** Yes
      - Backup replication
        - **Enable replication in another AWS Region:** No
    - Encryption
      - **Enable encryption:** Yes
      - **AWS KMS key:** aws/rds
    - **Log exports:** *[none]*
    - Maintenance
      - **Enable auto minor version upgrade:** Yes
      - **Maintenance window:** Choose a window
        - **Start day:** Saturday
        - **Start time:** 07:05 UTC
        - **Duration:** 0.5 hours
    - Deletion protection
      - **Enable deletion protection:** Yes

## 21b. RDS database (staging)

AWS console > Amazon RDS > Create database

- Create database
  - **Choose a database creation method:** Standard create
  - Engine options
    - **Engine type:** PostgreSQL
    - **Engine Version:** PostgreSQL 15.4-R3
  - **Templates**:** Production
  - **Availability and durability:** Single DB instance
  - Settings
    - **DB instance identifier:** `{project}-staging-db` <mark>Differs from production</mark>
    - Credentials Settings
      - **Master username:** postgres
      - **Master password:** *postgres-password-staging* <mark>Differs from production</mark>
  - Instance configuration
    - **DB instance class:** db.t3.micro <mark>Differs from production</mark>
  - Storage
    - **Storage type:** General Purpose SSD (gp3)
    - **Allocated storage:** 20 GiB <mark>Differs from production</mark>
  - Storage
  - Storage autoscaling
    - **Enable storage autoscaling:** No <mark>Differs from production</mark>
  - Connectivity
    - **Compute resource:** Don't connect to an EC2 compute resource
    - **Network type:** IPv4
    - **Virtual private cloud (VPC):** {project}-staging-vpc <mark>Differs from production</mark>
  - Connectivity
    - **DB subnet group:** {project}-staging-rds-subnetgroup <mark>Differs from production</mark>
  - Connectivity
    - **Public access:** No
   - **VPC security group:** {project}-staging-db-sg <mark>Differs from production</mark>
  - Connectivity
    - **Availability zone:** us-west-2c
    - **RDS Proxy:** No <mark>Differs from production</mark>
    - **Certificate authority:** rds-ca-rsa2048-1 <mark>Differs from production</mark>
- **Database authentication:** Password and IAM database authentication
- Monitoring
  - **Turn on Performance Insights:** Yes
  - **Retention period for Performance Insights:** 7 days
  - **AWS KMS key:** (default) aws/rds
  - **Turn on DevOps Guru:** No
- Additional configuration
  - Database options
    - **Initial database name:** *[leave blank]*
    - **DB parameter group:** default.postgres15
    - Backup
      - **Enable automated backups:** No <mark>Differs from production</mark>
    - Encryption
      - **Enable encryption:** Yes
      - **AWS KMS key:** aws/rds
    - **Log exports:** *[none]*
    - Maintenance
      - **Enable auto minor version upgrade:** Yes
      - **Maintenance window:** Choose a window
        - **Start day:** Saturday
        - **Start time:** 07:05 UTC
        - **Duration:** 0.5 hours
    - Deletion protection
      - **Enable deletion protection:** Yes
