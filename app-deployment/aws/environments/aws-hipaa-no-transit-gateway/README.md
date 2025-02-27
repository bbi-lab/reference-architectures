## AWS environment templates

The templates bundled here create an AWS network environment for web applications without a separate management VPC.

The rest of this document may be outdated, and parts are superseded by the instruction in this repository's main [README.md](../README.md).

## HIPAA Reference Architecture on the AWS Cloud—Quick Start

For architectural details, step-by-step instructions, and customization options, see the [deployment guide](https://fwd.aws/vd5pn?).

To post feedback, submit feature ideas, or report bugs, use the **Issues** section of this GitHub repo.

To submit code for this Quick Start, see the [AWS Quick Start Contributor's Kit](https://aws-quickstart.github.io/).

---
## S3 Bucket Setup

To deploy this stack, first create an S3 bucket to store the templates and submodules (see https://aws-quickstart.github.io/option1.html)

The bucket name should match the `TemplateS3BucketName` specified in the entrypoint template (default `{organization}-{project}-aws-templates`) and bucket region should be `TemplateS3BucketRegion` (default `us-east-2`). Add a subfolder matching `TemplateS3KeyPrefix` (default `{project}-network/`). These default values can be changed at time of deployment.

The bucket should have the following structure and contents:
```
{project}-network/
  templates/
    ...
  submodules/
    ...
```

## Deployment

This set of CloudFormation templates is meant to be deployed once. Resource names have been hardcoded to conform with our naming conventions, including a `{project}-` prefix. Even deploying in a separate account will fail because the bucket names must be unique within each region.

The region has been hardcoded as `us-east-2` and Availability Zones 1 and 2 are set to `us-east-2c` and `us-east-2b`, respectively.

Log in to AWS console and go to CloudFormation, makle sure current region in upper right is `us-east-2` (Ohio). Click Create Stack > With new resources.

1. Create Stack:

    Choose "Template is ready" and S3 URL for template source, enter the URL of the entrypoint template from the
    S3 bucket.

    Click Next.

2. Specify stack details:

    Enter a stack name (e.g. `{project}-network`)

    Enter the ARN for the AWS Config Service-linked role (`AWSServiceRoleForConfig`)

    Enter the email address to use with SNS (e.g. a Slack channel email address)

    Confirm the Quick Start S3 bucket name and key prefix match the S3 bucket containing the templates.

    Click Next.

3. Configure stack options:

    Add optional tags (e.g. `Billing Group`: `{project}`)

    Click Next.

4. Review:

    Confirm all parameter values are correct and then click "Submit"

    Monitor the status of the stack. When the status is `CREATE_COMPLETE`, the deployment is ready.


---
## Post-deployment

### Encrypt CloudTrail with KMS key

The current version of this template does not use a KMS key with CloudTrail. To generate a key and apply it to CloudTrail, follow the documentation here:
https://docs.aws.amazon.com/awscloudtrail/latest/userguide/encrypting-cloudtrail-log-files-with-aws-kms.html

To match the naming conventions used in these templates, give the key an alias of `{project}-cloudwatch-kms-key`.

### Update default VPC security groups

Three default VPC security groups are created, one for each VPC included in this stack. To remain consistent with AWS security best practices, these security groups should not default to allowing inbound and outbound traffic, so those rules should be deleted.

Adding names (e.g. `{project}-dev-vpc-default-sg`) to these security groups is also helpful.


## Deleting the stack

Archive or migrate then delete any resources that were created within the environment, outside of CloudFormation.

After deleting the stack, some resources may require manual clean up.

The S3 buckets:
- `{project}-aws-cloudtrail`
- `{project}-app-logs`

Note: server access logging needs to be disabled and the bucket policy removed from  `{project}-aws-cloudtrail` before its contents can be deleted.

Names of deleted S3 buckets may not be immediately available for reuse, resulting in the following error in CloudFormation:
```
A conflicting conditional operation is currently in progress against this resource. Please try again.
```
It could take an hour or more before a deleted bucket name can be reused.


If the CloudFormation status is `DELETE_FAILED`, review the resources associated with the stack to determine which may need to be deleted manually.
