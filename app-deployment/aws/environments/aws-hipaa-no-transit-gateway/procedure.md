# Manual resource creation

This manual procedure should be followed after creating the network environment.

All steps should be performed in the `us-west-2` (Oregon) region except where noted.

## Parameters

This guide does not include passwords or other secrets, but it does assume the following account-specific parameters:

- {aws-account-id}: The ID of your AWS account, typically a 12-digit numeral
- {organization}: The organization name, in a kebab-cased form suitable for use in identifiers
- {project}: The project name, in a kebab-cased form suitable for use in identifiers
- {postgres-password-prod}: The password for the production database's root (`postgres`) user

## 1. X EC2 key pair for bastion hosts (prod)

AWS console > EC2 > Key pairs > Create key pair

- Key pair
  - **Name::** `{project}-prod-bastion-ec2-user-keypair`
  - **Key pair type:** RSA
  - **Private key format:** .pem
  - Tags
    - *project* = `{project}`
    - *environment* = `prod`

Save the private key when you generate it. It cannot be downloaded again.

This key will provide access (as the root user, `ec2-user`) to the bastion host.

## 2. X EC2 security group for bastion hosts (prod)

AWS console > EC2 > Security groups > Create security group

- Create security group
  - Basic details
    - **Security group name:** `{project}-prod-bastion-sg`
    - **Description:** `Security group for {project} production bastion servers`
    - **VPC:**- {project}-prod-vpc
  - Inbound rules
    - [#1]
      - **Type:** SSH
      - **Source:** Anywhere-IPv4
    - [#2]
      - **Type:** SSH
      - **Source:** Anywhere-IPv6
  - Outbound rules
    - [Leave default in place, allowing all outbound traffic.]
  - Tags
    - *Name* = `{project}-prod-bastion-sg`
    - *project* = `{project}`
    - *environment* = `prod`

## 3. EC2 host (bastion, prod)

AWS console > EC2 > Instances > Launch instances

- Launch an instance
  - Name and tags
    - Tags
      - *Name* = `{project}-prod-bastion` (Apply to resource types: Instances, Volumes, Network interfaces)
      - *project* = `{project}` (Apply to resource types: Instances, Volumes, Network interfaces)
  - Application and OS Images
    - **Quick Start:** Amazon Linux
    - **Amazon Machine Image (AMI):** Amazon Linux 2023 AMI
    - **Architecture:** 64-bit (Arm)
  - **Instance type:** t4g.nano
  - Key pair
    - **Key pair name:** `{project}-prod-bastion-1-ec2-root-keypair`
  - Network settings
    - **VPC:** {project}-prod-vpc
    - **Subnet:** {project}-prod-public-subnet-1
    - **Auto-assign public IP:** Enable
    - **Firewall (security groups):** Select existing security group
    - **Security group:** {project}-prod-bastion-sg

After creating the bastion host, use SSH to log in as ec2-user, and edit /etc/ssh/sshd_config. Find the lines below and un-comment three of them, as shown:

```
AllowAgentForwarding yes
AllowTcpForwarding yes
#GatewayPorts no
X11Forwarding no
```

## 4. X Elastic IP address for the bastion host (prod)

AWS console > EC2 > Elastic IP addresses > Allocate Elastic IP address

- Allocate Elastic IP address
  - Elastic IP address settings
    - **Public IPv4 address pool:** Amazon's pool of IPv4 addresses
  - Tags
    - *project* = `{project}`
    - *environment* = `prod`

Now select the newly-created EIP. Click "Associate Elastic IP address."

- Associate Elastic IP address
  - **Resource type:** Instance
  - **Instance:** [Select the instance ID with name `{project}-prod-bastion-1-ec2-root-keypair`, created above.]
  - **Private IP address:** [Select the sole private IP address listed.]

Click "Associate."

## 5. X Security group for API servers (prod)

AWS console > EC2 > Security Groups > Create security group

- Create security group
  - Basic details
    - **Security group name:** `{project}-prod-api-sg`
    - **Description:** `Security group for {project} production API servers`
    - **VPC:**- {project}-prod-vpc
  - Inbound rules
    - [#1]
      - **Type:** SSH
      - **Source:** Custom: [select the ID of the security group named `{project}-prod-bastion-sg`.]
  - Outbound rules
    - [Leave default in place, allowing all outbound traffic.]
  - Tags
    - *Name* = `{project}-prod-api-sg`
    - *project* = `{project}`
    - *environment* = `prod`

## 6. X ECR repository for {project} Docker images

AWS console > Amazon Elastic Container Registry > Private registry > Repositories > Create repository

- Create repository
  - General settings
    - **Repository name:** `{project}`
    - **Image tag mutability:** Mutable
  - Encryption settings
    - **Encryption configuration:** AES-256

## 7. X RDS subnet group (prod)

AWS console > Amazon RDS > Subnet groups > Create DB subnet group

- Subnet group details
  - **Name:** `{project}-prod-rds-subnetgroup`
  - **Description:** `RDS subnet group for {project} production DB`
  - **VPC:** {project}-prod-vpc
- Add subnets
  - **Availability zones:** us-west-2b, us-west-2c
  - Select Subnet IDs of {project}-prod-private-subnet-1, {project}-prod-private-subnet-2

## 8. X EC2 security group for the database (prod)

AWS console > EC2 > Security groups > Create security group

- Create security group
  - Basic details
    - **Security group name:** `{project}-prod-db-sg`
    - **Description:** `Security group for {project} production DB`
    - **VPC:**- {project}-prod-vpc
  - Inbound rules
    - [#1]
      - **Type:** PostgreSQL
      - **Source:** Custom: [select the ID of the security group named `{project}-prod-api-sg`.]
    - [#2]
      - **Type:** PostgreSQL
      - **Source:** Custom: [select the ID of the security group named `{project}-prod-bastion-sg`.]
  - Outbound rules
    - [Leave default in place, allowing all outbound traffic.]
  - Tags
    - *Name* = `{project}-prod-db-sg`
    - *project* = `{project}`
    - *environment* = `production`

Add inbound rules allowing TCP traffic on port 5432 (PostgreSQL) from two sources:
- The bastion host's IP address, as a CIDR block (i.e. the IP address with the suffix `/32`), with the description `Bastion`
- And the security group `{project}-prod-api-sg`.

## 9. X RDS database (prod)

AWS console > Amazon RDS > Create database

- Create database
  - **Choose a database creation method:** Standard create
  - Engine options
    - **Engine type:** PostgreSQL
    - **Engine Version:** PostgreSQL 16.4-R1
  - **Templates:**** Production
  - **Availability and durability:** Single DB instance
  - Settings
    - **DB instance identifier:** `{project}-prod-db`
    - Credentials Settings
      - **Master username:** postgres
      - **Master password:** {postgres-password-prod}
  - Instance configuration
    - **DB instance class:** db.t3.small
  - Storage
    - **Storage type:** General Purpose SSD (gp3)
    - **Allocated storage:** 30 GiB
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
    - **Availability zone:** us-west-2b
    - **RDS Proxy:** Yes
    - **Certificate authority:** rds-ca-rsa2048-g1
- **Database authentication:** Password and IAM database authentication
- Monitoring
  - **Turn on Performance Insights:** Yes
  - **Retention period for Performance Insights:** 7 days
  - **AWS KMS key:** (default) aws/rds
  - **Turn on DevOps Guru:** No
- Additional configuration
  - Database options
    - **Initial database name:** *[leave blank]*
    - **DB parameter group:** default.postgres16
    - Backup
      - **Enable automated backups:** Yes
      - **Backup retention period:** 7 days
      - **Backup window:** 07:05 UTC
      - **Duration:** 0.5 hours
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
        - **Start day:** Monday
        - **Start time:** 08:05 UTC
        - **Duration:** 0.5 hours
    - Deletion protection
      - **Enable deletion protection:** Yes

## 10. X S3 bucket for application and server logs

AWS console > Amazon S3 > Buckets > Create bucket

- Create bucket
  - General configuration
    - **AWS region:** us-west-2
    - **Bucket type:** General purpose
    - **Bucket name:** `{organization}-{project}-app-logs`
  - **Object Ownership:** ACLs disabled
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

## 11. X IAM policy granting read access to ECR

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
                    "arn:aws:ecr:us-west-2:{aws-account-id}:repository/{project}"
                ]
            }
        ]
    }
    ```
  - Review and create
    - Policy details
      - **Policy name:** `{project}-ecr-read`
      - **Description:** `Permission to read the {project} ECR repository`
    - Tags
      - *project* = `{project}`

## 12. X IAM policy granting log write access (prod)

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
      - *environment* = `prod`

## 13. X IAM role for API servers (prod)

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
  - *environment* = `prod`

## 14. TODO SSH certificate for api.{project}.org

AWS console > AWS Certificate Manager (ACM) > List certificates > Request

- Request certificaate
  - **Certificate type:** Request a public certificate
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

## 15. TODO SSH certificate for {project}.org

Create this certificate in the `us-west-1` (Virginia) region, because it will be used by CloudFront.

- Request certificaate
  - **Certificate type:** Request a public certificate
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

## 16. X EC2 key pair for API servers (prod)

AWS console > EC2 > Key pairs > Create key pair

- Key pair
  - **Name::** `{project}-prod-eb-ec2-user-keypair`
  - **Key pair type:** RSA
  - **Private key format:** .pem
  - Tags
    - *project* = `{project}`
    - *environment* = `production`

This key will provide access (as the root user, `ec2-user`) to the API servers, which are EC2 instances managed by Elastic Beanstalk.

Save the private key when you generate it. It cannot be downloaded again.

## 17. Elastic Beanstalk API environment (prod)

AWS console > Elastic Beanstalk > Create environment

- Configure environment
  - **Environment tier:** Web server environment
  - Application information
    - **Application name:** `{project}-prod-api-app`
    - Application tags
      - *project* = `{project}`
      - *environment* = `prod`
  - Environment information
    - **Application name:** `{project}-prod-api-env`
    - **Domain:** `{project}`.us-west-2.elasticbeanstalk.com [If desired or if the project name is under 4 characters in length, you can choose a different name.]
  - Platform
    - **Platform type:** Managed platform
    - **Platform:** Docker
    - **Platform branch:** Docker running on 64bit Amazon Linux 2023
    - **Platform version:** 4.3.6
  - **Application code:** Sample application
  - **Presets:** High availability
- Configure service access
  - **Service role:** Create and use new service role
  - **Service role name:** aws-elasticbeanstalk-service-role
  - **EC2 key pair:** {project}-prod-eb-ec2-user-keypair
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
      - **Root volume type:** General Purpose 3(SSD)
      - **Size:** 10 GB
      - **IOPS:** 3000 IOPS
      - **Throughput:** 125 MiB/s
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
        - **Architecture:** arm64
        - **Instance types:** t4g.medium
        - **AMI ID:** ami-050670295b3ccce7d [default]
  - Load balancer network settings
    - **Visibility:** Public
    - **Load balancer subnets:** {project}-prod-public-subnet-1, {project}-prod-public-subnet-2
  - **Load balancer type:** Application load balancer, Dedicated
  - Listeners
    - [For now, leave the default listener on port 80.]
  - Processes
    - [For now, leave the default process on port 80.]
  - Rules
    - [None]
  - Log files access
    - **Store logs:** Enabled
    - **S3 bucket:** {organization}-{project}-app-logs
    - **Prefix:** `prod-api`
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

## 15. Route 53 DNS entries (TODO)

## 16. Elastic Beanstalk environment configuration changes (TODO)

## 17a. S3 bucket for the UI (prod)

AWS console > Amazon S3 > Create bucket

- Create bucket
  - General configuration
    - **AWS Region:** us-west-2
    - **Bucket type:** General purpose
    - **Bucket name:** `{organization}-{project}-prod-ui`
  - **Object Ownership:** ACLs disabled
  - Block Public Access settings for this bucket
    - [All disabled, allowing public access]
  - Bucket Versioning
    - **Bucket Versioning:** Disable
  - Tags
    - *project* = `{project}`
    - *environment* = `prod`
  - Default encryption
    - **Encryption type:** Server-side encryption with Amazon S3-managed keys
    - **Bucket Key:** Enable

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

## 18a. CloudFront distribution (prod)

AWS console > CloudFront > Create distribution

- Create distribution
  - Origin
    - **Origin domain:** {organization}-{project}-prod-ui.s3.us-west-2.amazonaws.com
    - **Origin path:** [blank]
    - **Name:** `{organization}-{project}-prod-ui.s3.us-west-2.amazonaws.com`
    - **Origin access:** Public
    - **Enable Origin Shield:** No
  - Default cache behavior
    - **Path pattern:** Default (*)
    - **Compress objects automatically:** Yes
    - Viewer
      - **Viewer protocol policy:** Redirect HTTP to HTTPS
      - **Allowed HTTP methods:** GET, HEAD
      - **Restrict viewer access:** No
    - **Cache key and origin requests:** Cache policy and origin request policy
  - **Web Application Firewall (WAF):** Enable security protections
    - **Use monitor mode:** On
  - Settings
    - **Price class:** Use all edge locations
    - **Alternate domain names (CNAME):** `{project}.org`, `www.{project}.org`
    - **Custom SSL certificate:** {project}.org
    - **Legacy clients support:** Off
    - **Security policy:** TLSv1.2_2021
    - **Supported HTTP versions:** HTTP/2
    - **Default root object:** `index.html`
    - **Standard logging:** Off
    - **IPv6:** On
