# AWS Incident Response & Automated EC2 Isolation Lab

> **Platform:** AWS SimuLearn | Skill Builder  
> **Focus:** Cloud Security, Incident Response Automation, SOC Workflows


<img width="1188" height="658" alt="IncidentResponse" src="https://github.com/user-attachments/assets/edfe6b8c-0990-403e-8e9d-c79d55a65e26" />





## What This Is About

This was a hands on lab from AWS SimuLearn focused on incident response in a cloud environment. The core idea is straightforward when a brute force attack hits your application server and starts throwing a flood of HTTP 401 errors, the system should detect it and automatically isolate the compromised instance. No manual intervention, no waiting for someone to notice.

I got to wire up the full pipeline myself  from setting up log ingestion to configuring alarms, connecting notification services, and watching a Lambda function strip an EC2 instance's access in real time. That last part was genuinely satisfying to see.



## The Problem This Solves

In a traditional setup, someone on the security team would need to notice the spike, investigate, then manually revoke access and update security groups. That takes time  and in a real attack scenario, time matters.

What this lab demonstrates is how you can automate that entire response chain. The moment the system detects abnormal 401 activity, it handles containment by itself. The instance gets isolated  IAM role removed, security group replaced with one that has zero ingress/egress rules  and it stays running so forensic analysis is still possible.

This is exactly how modern cloud SOC (Security Operations Center) workflows are designed. Detection → Notification → Automated Response, all without a human in the loop for the critical containment step.



## Services Used

### 🔔 Amazon SNS (Simple Notification Service)
A notification service that can send messages to multiple subscribers. You publish a message to a "topic" and any subscribers to that topic receive it. Here it acts as the glue between CloudWatch (detection) and Lambda (response). When the alarm fires, it publishes to the SNS topic  and the Lambda function subscribed to that topic gets invoked automatically.

### 📊 Amazon CloudWatch
AWS's monitoring and logging service. It collects logs and metrics from your resources. In this project it does two things  ingests application logs from the EC2 instance via the CloudWatch Agent, and then evaluates those logs against an alarm condition to detect the attack.

### ⚡ AWS Lambda
Serverless compute  you write a function, define what triggers it, and AWS handles running it. No servers to manage. Here, one Lambda simulates the attacker (sends 40 bad login attempts in a loop), and another one does the actual incident response work (isolates the EC2 instance).

### 🖥️ Amazon EC2 (Elastic Compute Cloud)
The virtual server running the Flask web application. This is the "victim" in the scenario  the instance that gets targeted by the bruteforce attack and eventually isolated.

### 🔧 AWS Systems Manager (SSM)
A management service that lets you access and manage EC2 instances without needing SSH keys or opening inbound ports. I used two of its features: **Fleet Manager** to view and manage the instance, and **Session Manager** to open a terminal session directly from the AWS console.

### 🗂️ AWS SSM Parameter Store
Secure storage for configuration data. After configuring the CloudWatch agent, the config gets saved here so it can be referenced and reused  useful for deploying the same agent config across multiple instances at scale.

### 🔑 AWS IAM (Identity and Access Management)
Controls permissions in AWS. The EC2 instance had an IAM instance profile (a role attached to the instance) that gave it certain permissions. Part of the isolation response is removing this role  so even if someone was still connected, they'd have no AWS permissions to do anything.



## How the Whole Thing Works

Here's the flow from start to finish:

### Step 1  Set Up the SNS Topic
First thing was creating an SNS topic called `UnauthorizedExceptionNotification`. Then subscribing an email address to it using the email protocol  AWS sends a confirmation email and the subscription only activates after you click the confirm link. This SNS topic was later used by both the email alerts and the isolation Lambda function.

### Step 2 – Review the EC2 Instance
Checked the AppServer instance in EC2 and confirmed the IAM role and security group were already attached and configured. I also copied the Instance ID because it was later used in the CloudWatch metric and Lambda workflow.

### Step 3  Install CloudWatch Agent via SSM
Instead of SSHing in, I used **SSM Fleet Manager** → selected the AppServer → Node Actions → Execute Run Command. Selected the `AWSConfigureAWSPackage` document, set the name to `amazon-cloudwatch-agent`, targeted the instance manually, then ran it. The output confirmed the package installed successfully.

### Step 4  Configure the CloudWatch Agent
Used Session Manager to open a terminal session directly inside the instance  no SSH, no key pairs, no open ports needed. Ran the config wizard:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

Walked through the wizard to configure it to ship `/home/ssm-user/record.log` to a CloudWatch log group, with 7day retention. Then saved that config to SSM Parameter Store and started the agent:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-a fetch-config -s -m ec2 \
-c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json
```

### Step 5  Create a Metric Filter on 401 Errors
In CloudWatch Logs, opened the `record.log` log group, searched for `401` events, then created a metric filter called `Unauthorized401`. Mapped it to:

 **Namespace:** `LabApplications`
 **Metric name:** the EC2 Instance ID (so the alarm knows which instance triggered it)
 **Value:** 1 per hit, unit: Count

### Step 6  Simulate the Attack
Before setting up the alarm, ran the `labFunctionTrafficGenerator` Lambda with a test event called `Generate401Traffic`. This function hammered the login endpoint 40 times with wrong credentials. Checked the logs  all 401s, exactly as expected.

### Step 7  Create the CloudWatch Alarm
In CloudWatch → Alarms → Create Alarm:
 Selected the `LabApplications` metric (the one tied to the instance ID)
 Statistic: **Sum**, Period: **30 seconds**
 Condition: **Greater than 20**
 Missing data treatment: **Treat as good** (so baseline silence doesn't trigger false alarms)
 Action: Notify the `UnauthorizedExceptionNotification` SNS topic when in ALARM state

Named it `Unauthorized401ErrorBreach`. After creation, it starts in `Insufficient data`, then moves to `OK` after a couple of minutes once it has baseline data.

### Step 8  Wire Up the Isolator Lambda
Went to the `labFunctionisolator` Lambda → Add Trigger → SNS → selected `UnauthorizedExceptionNotification`. Now this function fires automatically whenever that SNS topic receives a message.

### Step 9  Trigger the Full Pipeline
Ran the Traffic Generator Lambda again. This time:

1. 40 bad login attempts → 40 401 errors logged
2. CloudWatch sees Sum > 20 in 30 seconds → Alarm fires
3. Alarm publishes to SNS topic
4. SNS invokes the isolator Lambda
5. Lambda removes the IAM role from the instance and attaches `Isolated_SG` (a security group with no ingress or egress rules)

### Step 10  Verify Isolation
Back in EC2 → AppServer → Security tab:
- IAM Role: Removed
- Security Group: `Isolated_SG`

The instance is still running  but it's completely cut off. No inbound connections, no outbound traffic, no AWS permissions. Ready for forensics.



## What I Actually Learned

**SSM was one of the most interesting parts of this lab for me.** Before this, I always thought SSH was the normal way to access Linux servers. Using Session Manager to open a terminal directly from the AWS console without SSH keys or open ports felt much simpler and more secure.

**Log ingestion is the foundation of everything.** None of the detection or automation works if the logs aren't flowing. Setting up the CloudWatch agent properly, choosing the correct log path, and saving the configuration to Parameter Store turned out to be more important than I expected. 

**Metric filters are more powerful than they look.** The trick of using the EC2 instance ID as the metric name to carry context through the alarm → SNS → Lambda chain was something I wouldn't have thought of. It's simple but effective  the Lambda doesn't need to do any extra lookup to know which instance triggered the alarm.

**Automated response.** Before this lab, I mostly thought incident response was a manual process. Seeing AWS automatically isolate the EC2 instance after the alarm triggered helped me understand how cloud environments can respond much faster using automation.



## Why This Lab Was Useful

This lab covers a pattern that shows up constantly in cloud security: **detect → notify → respond**. The specific trigger here is HTTP 401s, but the same architecture works for GuardDuty findings, CloudTrail anomalies, unusual data transfer volumes  really anything you can express as a CloudWatch metric or log pattern.

Building this gave me a clearer picture of how automated incident response fits into a cloud SOC, and why the traditional "wait for a human" model doesn't scale in cloud environments where manual investigation and response would take too much time.



*Completed as part of the AWS SimuLearn Incident Response lab on [Skill Builder](https://skillbuilder.aws)*
