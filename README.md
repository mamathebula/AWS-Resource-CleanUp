# AWS Resource Cleanup Stack

Automated CloudFormation stack that deploys a Lambda function to clean up unused AWS resources on a schedule. Sends email notifications before deleting resources and publishes metrics to CloudWatch.

## What It Cleans Up

| Resource | Behavior |
|----------|----------|
| CloudFormation Stacks | Deleted after configurable time (default 8 hours) |
| EC2 Instances | Terminated after configurable time (pending, running, stopping, stopped) |
| S3 Buckets | Emptied and deleted after configurable time (handles versioned objects) |
| EBS Volumes | Deleted immediately if unattached (`available` status) |
| EBS Snapshots | Deleted if older than 7 days and not in use by AMIs |
| Elastic IPs | Released immediately if unassociated |

## How Protection Works

Resources are protected from deletion if any of these apply:

- Resource name or `Name` tag contains the exclude string (default: `do-not-delete`)
- CloudFormation stack has termination protection enabled
- The cleanup stack itself is always protected (matches keywords like `resourcecleanup`, `cleaner`, `cleanup-stack`)

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `MinRunningTimeHours` | `8` | Hours before stacks, instances, and buckets are deleted (1–168) |
| `NotificationTimeHours` | `4` | Hours before deletion to send a warning email (1–24) |
| `ExcludeString` | `do-not-delete` | Resources with this string in their name/tags are protected |
| `NotificationEmail` | *(required)* | Email address for deletion notifications |
| `ScheduleExpression` | `rate(2 hours)` | EventBridge schedule for how often cleanup runs |
| `LambdaTimeout` | `900` | Lambda timeout in seconds (60–900) |

## Deployment

### Deploy via AWS Console

1. Go to CloudFormation → Create Stack → Upload a template file
2. Upload `aws-resource-cleanup-stack-updated.yaml`
3. Fill in the parameters (at minimum, provide your email)
4. Acknowledge IAM resource creation
5. Create stack
6. Confirm the SNS subscription email you receive

### Deploy via CLI

```bash
aws cloudformation create-stack \
  --stack-name aws-resource-cleanup \
  --template-body file://aws-resource-cleanup-stack-updated.yaml \
  --parameters \
    ParameterKey=NotificationEmail,ParameterValue=your@email.com \
    ParameterKey=MinRunningTimeHours,ParameterValue=8 \
    ParameterKey=ExcludeString,ParameterValue=do-not-delete \
  --capabilities CAPABILITY_NAMED_IAM
```

## What Gets Created

- Lambda function (`ResourceCleanupFunction`) — Python 3.9, 512 MB memory
- IAM role with permissions scoped to the resource types it manages
- SNS topic (`CFT-Cleaner-Notifications`) with email subscription
- EventBridge rule to trigger the Lambda on schedule

## Notifications

The stack sends two types of email notifications:

1. **Warning** — sent during the notification window before a stack is deleted, with details on how to protect it
2. **Summary** — sent after each run if any resources were deleted, listing counts and names

Duplicate warnings are prevented by tagging stacks with `CleanupWarningNotified=true`.

## CloudWatch Metrics

Metrics are published to the `ResourceCleaner` namespace:

- `DeletedStacks`, `ProtectedStacks`, `StackNotifications`
- `TerminatedInstances`, `ProtectedInstances`
- `DeletedVolumes`, `ProtectedVolumes`, `VolumeAgeDistribution`
- `DeletedSnapshots`, `ProtectedSnapshots`, `SnapshotAgeDistribution`
- `ReleasedElasticIPs`, `ProtectedElasticIPs`
- `DeletedS3Buckets`, `ProtectedS3Buckets`
- `FailedDeletions` (with `ResourceType` dimension)

## IAM Permissions

The Lambda role has broad permissions across many AWS services to handle CloudFormation stack deletion (stacks can contain any resource type). Key service permissions include:

EC2, CloudFormation, S3, SNS, CloudWatch, Lambda, API Gateway, DynamoDB, EventBridge, CloudFront, Step Functions, Cloud9, SES, RDS, EKS, ECR, SQS, IAM, Route53, Secrets Manager, Auto Scaling, ELB, ECS, ElastiCache, EFS, MSK, KMS, SSM, Cognito, Image Builder

## Tips

- Set `ExcludeString` to something your team uses consistently, like `prod` or `permanent`
- Start with a longer `MinRunningTimeHours` (e.g., 24) and reduce once you're comfortable
- Check CloudWatch Logs at `/aws/lambda/ResourceCleanupFunction` for detailed execution logs
- The `FORCE_DELETE` environment variable is set to `True` by default — instances are terminated regardless of age when this is enabled
