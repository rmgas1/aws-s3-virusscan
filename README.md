# Antivirus for Amazon S3

## Update 2024-08-12

* Update to use "Amazon Linux 2023 version 2023.5.20240805" a.k.a. al2023-ami-2023.5.20240805.0-kernel-6.1-x86_64 (ami-02346a771f34de8ac).
* Add rsyslog to view journalctl -t s3-virusscan and other logs via CloudWatch.
* Update s3-virusscan scripts to use ruby3.2
* Update s3-virusscan gems to latest and removed workarounds for old gems.
* Change s3-virusscan from SysV init script to systemd service file.
* Update services to use systemd (sshd in ssh-access outstanding).
* Add BounceServices to ensure relevant items are enabled, and running when the box comes up.

### How to install

1. Enable "Default Host Management Configuration" for the account, or as you see fit.
2. Make sure you have "vpc-2azs", "vpc-3azs", or "vpc-4azs" stack setup.
3. Create a new stack by uploading the template file from this repository.
4. Set template variables as required for your use case.
5. Update the security group to restrict ssh inbound access.
6. Buy me a coffee :-)

## Update 2024-08-01

Updating in this repository as the original was archived by the owner on Oct 3, 2023.

* Update to use "Amazon Linux 2023 AMI 2023.5.20240722.0 x86_64 HVM kernel-6.1" (al2023-ami-2023.5.20240722.0-kernel-6.1-x86_64)
* Add new Regions

## Description

This template creates a malware scanner cluster for S3 buckets. Connect as many S3 buckets as you like.

> [bucketAV - Antivirus for Amazon S3](https://bucketav.com/) with additional features is available at [AWS Marketplace](https://aws.amazon.com/marketplace/pp/B07XFR781T).

## Features

* Uses ClamAV to scan newly added files on S3 buckets
* Updates ClamAV database every 3 hours automatically
* Scales EC2 instance workers to distribute the workload
* Publishes a message to SNS in case of a finding
* Can optionally delete compromised files automatically
* Logs to CloudWatch Logs

### Additional Commercial Features by bucketAV

* Reporting capabilities
* Dashboard
* Scan buckets at regular intervals / initial bucket scan
* Quarantine infected files
* Enhanced security features (e.g., IMDSv2)
* Regular Security updates
* Multi-Account support
* AWS Integrations:
    * CloudWatch Integration (Metrics and Dashboard)
    * Security Hub Integration
    * SSM OpsCenter Integration
* S3 -> SNS fan-out support
* Support

[bucketAV - Antivirus for Amazon S3](https://bucketav.com/) with additional features is available at [AWS Marketplace](https://aws.amazon.com/marketplace/pp/B07XFR781T).

## How does it work

A picture is worth a thousand words:

![Architecture](./docs/architecture.png?raw=true "Architecture")

1. A SQS queue is used to decouple scan jobs from the ClamAV workers. Each S3 bucket can fire events to that SQS queue in case of new objects. This feature of S3 is called [S3 Event Notifications](http://docs.aws.amazon.com/AmazonS3/latest/dev/NotificationHowTo.html).
1. The SQS queue is consumed by a fleet of EC2 instances running in an Auto Scaling Group. If the number of outstanding scan jobs reaches a threshold a new ClamAV worker is automatically added. If the queue is mostly empty workers are removed.
1. The ClamAV workers run a simple ruby script that executes the [clamscan](http://linux.die.net/man/1/clamscan) command. In the background the virus db is updated every three hours.
1. If `clamscan` finds a virus the file is directly deleted (you can configure that) and a SNS notification is published.

## Installation

### Create the CloudFormation Stack
1. This templates depends on one of our [`vpc-*azs.yaml`](https://templates.cloudonaut.io/en/stable/vpc/) templates. [Launch Stack](https://console.aws.amazon.com/cloudformation/home#/stacks/create/review?templateURL=https://s3-eu-west-1.amazonaws.com/widdix-aws-cf-templates-releases-eu-west-1/stable/vpc/vpc-2azs.yaml&stackName=vpc)
1. [Launch Stack](https://console.aws.amazon.com/cloudformation/home#/stacks/create/review?templateURL=https://s3-eu-west-1.amazonaws.com/widdix-aws-s3-virusscan/template.yaml&stackName=s3-virusscan&param_ParentVPCStack=vpc)
1. Click **Next** to proceed with the next step of the wizard.
1. Specify a name and all parameters for the stack.
1. Click **Next** to proceed with the next step of the wizard.
1. Click **Next** to skip the **Options** step of the wizard.
1. Check the **I acknowledge that this template might cause AWS CloudFormation to create IAM resources.** checkbox.
1. Click **Create** to start the creation of the stack.
1. Wait until the stack reaches the state **CREATE_COMPLETE**

### Configure the buckets
Configure the buckets you want to connect to as shown in the next figure:

![Configure Event Notifications 1](./docs/configure_event_notifications1.png?raw=true "Configure Event Notifications 1")

![Configure Event Notifications 2](./docs/configure_event_notifications2.png?raw=true "Configure Event Notifications 2")

**Make sure you select the *-ScanQueue-* NOT the *-ScanQueueDLQ-*!**

### Configure E-Mail subscription
If you like to receive emails if a virus was found you must subscribe to the SNS topic as shown in the next two figures:

![Subscribe Topic: Step 1](./docs/subscribe_topic_step1.png?raw=true "Subscribe Topic: Step 1")

![Subscribe Topic: Step 2](./docs/subscribe_topic_step2.png?raw=true "Subscribe Topic: Step 2")

You will receive a confirmation email.

> [bucketAV - Antivirus for Amazon S3](https://bucketav.com/) with additional features is available at [AWS Marketplace](https://aws.amazon.com/marketplace/pp/B07XFR781T).

## Troubleshooting

1. Go to [CloudWatch Logs in the AWS Management Console](https://console.aws.amazon.com/cloudwatch/home#logs:)
2. Click on the log group of the s3-virusscan
3. Click on the blue **Search Log Group** button
4. Search for `"s3-virusscan["`

## Known issues / limitations

* It was [reported](https://github.com/widdix/aws-s3-virusscan/issues/12) that the solution does not run on a t2.micro or smaller. Use at least a t2.small instance.
* An initial scan may also be useful but is not performed at the moment. This could be implemented with a Lambda function that pushes every key to SQS.
