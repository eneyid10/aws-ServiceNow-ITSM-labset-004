# Lab 04 — ITSM & Incident Management on AWS
### ServiceNow ITSM + AWS Integration · AWS-Native ITSM Alternative · ITIL 4

![ITSM](https://img.shields.io/badge/ITSM-ServiceNow%20PDI-62D84E?style=flat-square&logo=servicenow&logoColor=white)
![Cloud](https://img.shields.io/badge/Cloud-AWS-FF9900?style=flat-square&logo=amazon-aws&logoColor=white)
![Services](https://img.shields.io/badge/AWS-Incident%20Manager%20%7C%20Service%20Catalog%20%7C%20Change%20Manager-232F3E?style=flat-square&logo=amazon-aws&logoColor=white)
![Cost](https://img.shields.io/badge/Cost-%240%20Free%20Tier%20%2B%20Free%20PDI-22c55e?style=flat-square)
![Duration](https://img.shields.io/badge/Duration-3–5%20Hours-f59e0b?style=flat-square)
![Certs](https://img.shields.io/badge/Aligned-A%2B%20%7C%20Network%2B%20%7C%20ITIL%204-6366f1?style=flat-square)

---

## Overview

This lab documents IT Service Management practice applied to AWS infrastructure — incident management, change control, service catalogue self-service, and operational reporting — using two parallel implementations:

- **Track A — ServiceNow + AWS Integration.** A free ServiceNow Personal Developer Instance managing AWS resources, with CloudWatch alarms automatically opening incidents via EventBridge and SNS, and change requests gating infrastructure deployments.
- **Track B — AWS-Native ITSM.** The same ITIL processes built entirely with AWS Systems Manager Incident Manager, AWS Service Catalog, Systems Manager Change Manager, and AWS Config as the CMDB.

**Why both matter.** ServiceNow is the dominant enterprise ITSM platform — if you work IT support or cloud operations, you will use it or something structurally identical. But cloud teams increasingly run AWS-native ITSM for infrastructure workflows because it lives inside the same IAM boundary as the resources it manages. Understanding both, and being able to articulate when each is appropriate, is a genuine differentiator in a cloud operations interview.

> **Note on scope:** unlike Labs 01–03, this lab has no Azure component to replace. ServiceNow is a SaaS platform, vendor-neutral by design. The AWS adaptation here means *connecting ITSM process to AWS infrastructure* and *building the AWS-native equivalent* — not swapping one cloud provider for another.

---

## Architecture

<img width="1074" height="503" alt="ServiceNowITSM" src="https://github.com/user-attachments/assets/7935a881-43fe-4c29-878a-654570532a13" />

---

## ITIL Process → Platform Mapping

The core value of this lab: seeing the same ITIL process expressed in two platforms.

| ITIL Process | ServiceNow (Track A) | AWS-Native (Track B) |
|---|---|---|
| **Incident Management** | Service Desk → Incidents | AWS Systems Manager **Incident Manager** |
| **Problem Management** | Service Desk → Problems | Incident Manager **Post-Incident Analysis** |
| **Change Enablement** | Change → Changes | Systems Manager **Change Manager** |
| **Service Request** | Service Catalog | **AWS Service Catalog** (CloudFormation-backed) |
| **Configuration Management (CMDB)** | Configuration → CIs | **AWS Config** resource inventory |
| **Knowledge Management** | Knowledge → Articles | Systems Manager **Documents** (runbooks) |
| **Event Management** | Event Management module | **CloudWatch Alarms** + **EventBridge** |
| **SLA Management** | SLA → SLA Definitions | Incident Manager **response plan** severity tiers |

---

## Prerequisites

- AWS account ([aws.amazon.com/free](https://aws.amazon.com/free))
- ServiceNow Personal Developer Instance — free at [developer.servicenow.com](https://developer.servicenow.com), no credit card
- **Labs 01–03 recommended** — the EC2 instances built there become the managed resources in this lab
- IAM permissions for CloudWatch, EventBridge, SNS, Lambda, Systems Manager, Service Catalog, and Config

---

## Key Concepts — Read Before Starting

**What is ITSM?**
IT Service Management — the structured practice of delivering and supporting IT services. It defines *how* work flows: an incident follows an incident workflow, a change requires approval before implementation, a service request draws from a catalogue with predefined fulfilment steps. Without this structure, work falls through the cracks: a server goes down, three people report it separately, nobody knows who owns it.

**Incident vs Problem vs Change**
These three are the backbone of ITIL and come up in every IT interview.
- **Incident** — an unplanned service interruption. Goal: *restore service fast*. "Outlook won't connect."
- **Problem** — the root cause behind one or more incidents. Goal: *eliminate it permanently*. "Corrupted OST files are recurring because of a bad GPO."
- **Change** — a planned modification to infrastructure. Goal: *implement with minimal risk*. "Deploy the patch that fixes the GPO."

**What is a CMDB?**
Configuration Management Database — the authoritative record of every IT asset and how assets relate to one another. When an incident affects a server, the CMDB tells you which applications depend on it and therefore who else is impacted. In AWS, **AWS Config** performs this function natively: it continuously records every resource, its configuration, and its relationships.

**What is an SLA?**
Service Level Agreement — the committed response and resolution time per priority level. A P1 might commit to 15-minute response and 4-hour resolution; a P3 to 8-hour response and 3-day resolution. SLAs turn "we'll get to it" into a measurable commitment.

**What is CloudWatch → EventBridge → SNS?** *(AWS-specific)*
The AWS event pipeline that turns infrastructure conditions into action. **CloudWatch** monitors metrics and fires an alarm when a threshold is breached. **EventBridge** catches that alarm state change and routes it based on rules you define. **SNS** fans the message out to subscribers — email, Lambda, Incident Manager, or an HTTP endpoint. This chain is how an EC2 instance failing a status check automatically becomes a ticket without a human noticing first.

**What is AWS Systems Manager Incident Manager?** *(AWS-specific)*
The AWS-native incident management service. You define **response plans** (severity, engagement contacts, runbooks) and Incident Manager creates incidents automatically from CloudWatch alarms or EventBridge events, pages the on-call contacts, opens a collaboration channel, maintains an automatic timeline of events, and prompts a structured post-incident analysis. It is functionally the AWS answer to PagerDuty plus ServiceNow incident management for infrastructure.

**What is AWS Service Catalog?** *(AWS-specific)*
The AWS-native service request platform. Administrators publish approved, pre-configured infrastructure as **products** (backed by CloudFormation templates) organised into **portfolios**. End users browse the catalogue and provision only what has been approved, within guardrails they cannot exceed. This is the same concept as a ServiceNow catalogue item — but it *actually provisions the resource* rather than opening a fulfilment ticket for a human.

---

## What You Will Learn

| Skill | Real-World Application |
|---|---|
| Create and resolve an Incident | The core daily task of every help desk and cloud ops role |
| Set priority and SLA | Priority drives response time; SLA defines the business commitment |
| Auto-create incidents from CloudWatch alarms | How real cloud operations works — detection triggers the ticket, not a human |
| Build a service catalogue item | Self-service reduces ticket volume for routine requests |
| Create an approval workflow for changes | Core ITIL control preventing uncoordinated production modifications |
| Gate infrastructure changes on approval | Change Manager enforces approval *before* the runbook executes |
| Use AWS Config as a CMDB | Continuous, automatic resource inventory — no manual asset tracking |
| Run reports on volume and MTTR | Metrics drive IT operations decisions at every level |

---

# Track A — ServiceNow ITSM Integrated with AWS

## Step A1 — Provision the ServiceNow Instance

1. Go to [developer.servicenow.com](https://developer.servicenow.com)
2. **Sign Up** → create a free account (email and password only, no credit card)
3. **Request Instance** → select the latest stable release
4. Provisioning takes 10–15 minutes; you receive an instance URL (`dev12345.service-now.com`) and credentials by email

> **Keep the instance active.** ServiceNow hibernates PDIs after 10 days of inactivity and reclaims them after 30. Log in weekly. If reclaimed, a new instance is free — but your work is lost. Export anything you want to keep.

### Platform Navigation

| Module | Location | Purpose |
|---|---|---|
| Incident | Service Desk → Incidents | IT support tickets |
| Problem | Service Desk → Problems | Root cause analysis |
| Change | Change → Changes | Planned infrastructure modifications |
| Service Catalog | Service Catalog → Catalogs | User-facing request portal |
| Reports | Reports → Create New | Analytics and metrics |
| Flow Designer | Process Automation → Flow Designer | Visual workflow builder |

---

## Step A2 — Create and Work an Incident

This scenario uses an AWS infrastructure failure rather than a generic desktop issue — appropriate to a cloud operations context.

### Create the Incident

**Service Desk** → **Incidents** → **New**

| Field | Value |
|---|---|
| Caller | Abel Tuter *(built-in test user)* |
| Category | Infrastructure |
| Subcategory | Server |
| Configuration Item | `lab-splunk-idx` |
| Short description | Splunk indexer unreachable — EC2 status check failed |
| Description | CloudWatch alarm `lab-splunk-idx-StatusCheckFailed` triggered at 09:14 UTC. Splunk Web UI at port 8000 is not responding. EC2 console shows instance state `running` but system status check failing. No log ingestion since 09:12 — Windows forwarder and CloudTrail inputs both queuing. Affects all SOC search and alerting capability. |
| Impact | 2 — Medium |
| Urgency | 1 — High |
| Priority | 2 — High *(auto-calculated from Impact × Urgency)* |
| Assignment Group | Cloud Operations |

Click **Submit**. Note the ticket number (`INC0001234`).

> **Priority is calculated, not chosen.** ServiceNow derives Priority from an Impact × Urgency matrix. Impact = how much of the business is affected. Urgency = how quickly it must be fixed. Understanding this matrix is directly tested in ITIL Foundation and asked in interviews.

### Work the Incident

1. Open the incident → set **State** to `In Progress`
2. Set **Assigned to** = your user
3. Add a **Work Note** (internal, IT staff only):

> Confirmed CloudWatch alarm. EC2 console shows system status check failing — underlying host issue, not OS-level. Instance is Nitro-backed so a stop/start will migrate it to healthy hardware. Verified EBS volume is `in-use` and healthy; no data loss risk. Notified SOC that search is unavailable; log ingestion is queuing at the forwarder and will backfill on recovery. Initiating stop/start now. ETA 5 minutes.

4. Add **Resolution Notes**:

> Performed EC2 stop/start, which migrated the instance to healthy underlying hardware. System status check returned to `passed` at 09:26 UTC. Splunk service auto-started via `enable boot-start`. Verified Web UI reachable on :8000 and both Windows forwarder and CloudTrail inputs resumed — queued events backfilled with no gap in the index. Total outage 12 minutes. **Follow-up:** raising a Problem record to evaluate an Auto Scaling group or CloudWatch auto-recovery action so this self-heals without manual intervention.

5. Set **State** → `Resolved` → then `Closed`

> Notice the resolution creates a **Problem** record. That is the ITIL distinction in practice: the incident restored service, the problem eliminates the recurrence. Interviewers listen for exactly this.

---

## Step A3 — Auto-Create Incidents from CloudWatch Alarms

This is the integration that makes the lab AWS-native rather than a generic ServiceNow walkthrough. A CloudWatch alarm opens a ServiceNow incident with no human in the loop.

### A3a — Create the CloudWatch Alarm

**CloudWatch Console** → **Alarms** → **Create alarm**

| Field | Value |
|---|---|
| Metric | EC2 → Per-Instance Metrics → `StatusCheckFailed` |
| Instance | `lab-splunk-idx` |
| Statistic | Maximum |
| Period | 1 minute |
| Threshold | Static, Greater than `0` |
| Datapoints to alarm | 2 out of 2 |
| Alarm name | `lab-splunk-idx-StatusCheckFailed` |
| Notification | Send to SNS topic `lab-itsm-alerts` *(create new)* |

### A3b — Create the ServiceNow Integration User

In ServiceNow, create a dedicated service account rather than using your admin login:

1. **User Administration** → **Users** → **New**
2. Username `aws_integration`, set a strong password, check **Web service access only**
3. Assign the `itil` role (grants incident create/update, nothing more)

> A dedicated, minimally-privileged service account is the correct pattern for any system-to-system integration. Never wire an automation to a human admin account — you lose attribution and over-grant permissions.

### A3c — Create the Lambda Function

**Lambda Console** → **Create function** → Author from scratch, Python 3.12, name `sn-incident-creator`.

```python
import json
import os
import urllib.request
import base64

SN_INSTANCE = os.environ['SN_INSTANCE']       # e.g. dev12345
SN_USER     = os.environ['SN_USER']           # aws_integration
SN_PASSWORD = os.environ['SN_PASSWORD']       # from Secrets Manager in production

def lambda_handler(event, context):
    # SNS wraps the CloudWatch alarm payload in a JSON string
    message = json.loads(event['Records'][0]['Sns']['Message'])

    alarm_name  = message['AlarmName']
    reason      = message['NewStateReason']
    region      = message['Region']
    instance_id = next(
        (d['value'] for d in message['Trigger']['Dimensions']
         if d['name'] == 'InstanceId'),
        'unknown'
    )

    payload = {
        'short_description': f'CloudWatch alarm: {alarm_name}',
        'description': (
            f'Alarm: {alarm_name}\n'
            f'Instance: {instance_id}\n'
            f'Region: {region}\n'
            f'Reason: {reason}\n\n'
            f'Auto-created by AWS CloudWatch integration.'
        ),
        'category': 'Infrastructure',
        'subcategory': 'Server',
        'impact': '2',
        'urgency': '1',
        'assignment_group': 'Cloud Operations',
        'cmdb_ci': instance_id
    }

    url = f'https://{SN_INSTANCE}.service-now.com/api/now/table/incident'
    creds = base64.b64encode(f'{SN_USER}:{SN_PASSWORD}'.encode()).decode()

    req = urllib.request.Request(
        url,
        data=json.dumps(payload).encode(),
        headers={
            'Content-Type': 'application/json',
            'Accept': 'application/json',
            'Authorization': f'Basic {creds}'
        },
        method='POST'
    )

    with urllib.request.urlopen(req) as response:
        result = json.loads(response.read())
        incident_number = result['result']['number']
        print(f'Created ServiceNow incident {incident_number}')
        return {'statusCode': 200, 'incident': incident_number}
```

**Environment variables:** set `SN_INSTANCE`, `SN_USER`, `SN_PASSWORD`.

> **Production note:** store the password in **AWS Secrets Manager** and fetch it at runtime rather than using a plaintext Lambda environment variable. Environment variables are readable by anyone with `lambda:GetFunctionConfiguration`. The lab uses env vars for simplicity; the production pattern is Secrets Manager with the Lambda execution role granted `secretsmanager:GetSecretValue`.

### A3d — Subscribe Lambda to the SNS Topic

1. **SNS Console** → topic `lab-itsm-alerts` → **Create subscription**
2. Protocol: **AWS Lambda**, Endpoint: `sn-incident-creator`

### A3e — Test the Integration

```bash
# Force the alarm into ALARM state to trigger the pipeline
aws cloudwatch set-alarm-state \
  --alarm-name lab-splunk-idx-StatusCheckFailed \
  --state-value ALARM \
  --state-reason "Integration test — verifying ServiceNow incident creation"
```

Check **Service Desk → Incidents** in ServiceNow. A new incident should appear within seconds. Verify the Lambda execution in CloudWatch Logs if it does not.

---

## Step A4 — Build a Service Catalogue Item

Catalogue items let users self-serve routine requests without opening a ticket for each one.

**Service Catalog** → **Catalogs** → **Service Catalog** → **Maintain Items** → **New**

| Field | Value |
|---|---|
| Name | AWS Development Instance Request |
| Category | Cloud Services |
| Short description | Request a development EC2 instance |
| Description | Request a development EC2 instance for project work. Requests are reviewed within 1 business day. Instances are provisioned via AWS Service Catalog with pre-approved AMIs and instance types, tagged to your cost centre, and automatically stopped nightly at 20:00 UTC to control cost. |
| Fulfillment group | Cloud Operations |

Click **Submit**, then open the **Variables** tab and add:

| Variable Name | Type | Mandatory | Notes |
|---|---|---|---|
| Requester Name | Single Line Text | Yes | |
| Business Justification | Multi Line Text | Yes | |
| Required By Date | Date | Yes | |
| Instance Size | Select Box | Yes | Options: `t3.micro`, `t3.small`, `t3.medium` |
| Operating System | Select Box | Yes | Options: Amazon Linux 2023, Ubuntu 24.04 LTS |
| Environment | Select Box | Yes | Options: Dev, Test *(Prod excluded — requires a Change)* |
| Cost Centre | Single Line Text | Yes | Applied as an AWS resource tag |
| Auto-Shutdown | Checkbox | No | Default checked — nightly stop at 20:00 UTC |

**Save** and **Preview** — the item now appears in the catalogue portal.

> Constraining choices to approved values is the point. The user cannot request a `p4d.24xlarge` GPU instance or provision straight into Production. Guardrails are enforced by the form, not by hoping people read a policy document.

---

## Step A5 — Create an Approval Workflow for Change

Change requests require approval before work begins — a core ITIL control preventing uncoordinated production modifications.

**Change** → **Changes** → **New** → **Normal**

| Field | Value |
|---|---|
| Short description | Apply security patches to EC2 fleet via SSM Patch Manager |
| Category | Software |
| Configuration Item | `lab-ad-dc01`, `lab-splunk-idx` |
| Risk | Low |
| Impact | 2 — Medium |
| Start date | Next Saturday 02:00 UTC |
| End date | Next Saturday 06:00 UTC |
| Description | Monthly security patch deployment to all EC2 instances in the lab VPC. Executed via AWS Systems Manager Patch Manager using the `AWS-RunPatchBaseline` document. Instances require one reboot each. Patches address CVEs rated CVSS 7.0+. Deployment sequenced: Splunk indexer first, domain controller second, to preserve log capture during the DC reboot. |

Under the **Planning** tab:

**Test Plan:**
> Patch baseline validated in a Dev account against identical AMIs. SSM Agent connectivity confirmed on both targets via `aws ssm describe-instance-information`. Patch scan run in `Scan` mode to enumerate missing patches without installing — output reviewed and approved.

**Backout Plan:**
> Create an EBS snapshot of both instances immediately prior to patching. If post-patch validation fails, stop the instance, detach the root volume, create a replacement volume from the snapshot, reattach, and start. Estimated recovery time 15 minutes per instance. Snapshots retained 7 days.

**Implementation Plan:**
> 1. Verify EBS snapshots complete. 2. Execute `AWS-RunPatchBaseline` in `Install` mode against `lab-splunk-idx`. 3. Confirm Splunk service healthy and Web UI reachable. 4. Execute against `lab-ad-dc01`. 5. Confirm AD DS healthy via `Get-ADDomainController`. 6. Confirm log ingestion resumed in Splunk with no gap.

Click **Request Approval** → the change moves to `Pending Approval`. Open the **Approvals** tab and approve as admin → state becomes `Scheduled`.

> A change record with a real backout plan and sequenced implementation steps is what separates a credible portfolio artifact from a checkbox exercise. Hiring managers read the backout plan first — it tells them whether you have actually operated production infrastructure.

---

## Step A6 — Build Reports

**Reports** → **Create New**

**Report 1 — Incident Volume by Priority**

| Field | Value |
|---|---|
| Name | Incident Volume by Priority — Last 30 Days |
| Data source | Incident [incident] |
| Type | Bar Chart |
| Group by | Priority |
| Condition | Created **on or after** 30 days ago |

**Report 2 — Mean Time to Resolution**

| Field | Value |
|---|---|
| Name | MTTR by Assignment Group |
| Data source | Incident [incident] |
| Type | Bar Chart |
| Group by | Assignment Group |
| Aggregation | Average of `Resolve time` |
| Condition | State **is** Closed |

**Report 3 — Open Incidents by Agent**

| Field | Value |
|---|---|
| Name | Open Incidents by Assigned Agent |
| Data source | Incident [incident] |
| Type | Bar Chart |
| Group by | Assigned to |
| Condition | State **is not** Closed |

---

# Track B — AWS-Native ITSM

The same ITIL processes, built entirely inside AWS. This is what cloud-native operations teams increasingly run for infrastructure workflows.

## Step B1 — Incident Management with Systems Manager Incident Manager

### B1a — Create Contacts

**Systems Manager** → **Incident Manager** → **Contacts** → **Create contact**

| Field | Value |
|---|---|
| Name | `cloud-ops-oncall` |
| Contact channel | Email → your address |
| Engagement plan | Stage 1: email immediately (0 min) |

### B1b — Create a Response Plan

**Incident Manager** → **Response plans** → **Create response plan**

| Field | Value |
|---|---|
| Name | `ec2-critical-response` |
| Incident title | `EC2 instance failure — automated` |
| Impact | 2 — High (significant service degradation) |
| Engagements | `cloud-ops-oncall` |
| Runbook | `AWSIncidents-CriticalIncidentRunbook` *(or custom)* |

### B1c — Wire the CloudWatch Alarm to Incident Manager

**CloudWatch** → **Alarms** → select `lab-splunk-idx-StatusCheckFailed` → **Actions** → **Edit**

Under **Notification**, add an action: **Incident Manager response plan** → `ec2-critical-response`.

Now the same alarm that opens a ServiceNow incident in Track A opens a native AWS incident in Track B — with automatic on-call engagement, a collaboration channel, and a running event timeline.

### B1d — Test and Run Post-Incident Analysis

```bash
aws cloudwatch set-alarm-state \
  --alarm-name lab-splunk-idx-StatusCheckFailed \
  --state-value ALARM \
  --state-reason "Incident Manager integration test"
```

Open **Incident Manager** → **Incidents** → your new incident. Work through the timeline, add notes, then resolve it and run **Post-incident analysis** — Incident Manager walks you through a structured template covering timeline accuracy, detection time, mitigation time, and action items.

> That post-incident analysis *is* ITIL Problem Management. Same process, native tooling: the incident restored service, the analysis identifies the root cause and the action items that prevent recurrence.

---

## Step B2 — Service Requests with AWS Service Catalog

### B2a — Create the CloudFormation Product Template

Save as `dev-instance-product.yaml`:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Approved development EC2 instance with enforced guardrails

Parameters:
  InstanceSize:
    Type: String
    Default: t3.micro
    AllowedValues: [t3.micro, t3.small, t3.medium]
    Description: Instance type (constrained to approved sizes)
  Environment:
    Type: String
    Default: Dev
    AllowedValues: [Dev, Test]
    Description: Target environment (Prod requires a Change Request)
  CostCentre:
    Type: String
    Description: Cost centre for chargeback tagging
  RequesterName:
    Type: String
    Description: Name of the requesting user

Resources:
  DevInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceSize
      ImageId: '{{resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64}}'
      Tags:
        - Key: Name
          Value: !Sub 'dev-${RequesterName}-${Environment}'
        - Key: Environment
          Value: !Ref Environment
        - Key: CostCentre
          Value: !Ref CostCentre
        - Key: Requester
          Value: !Ref RequesterName
        - Key: AutoShutdown
          Value: 'true'
        - Key: ManagedBy
          Value: ServiceCatalog

Outputs:
  InstanceId:
    Description: EC2 instance ID
    Value: !Ref DevInstance
  PrivateIp:
    Description: Private IP address
    Value: !GetAtt DevInstance.PrivateIp
```

### B2b — Publish the Product

1. **Service Catalog** → **Portfolios** → **Create portfolio** → name `IT Services`
2. Open the portfolio → **Products** → **Upload new product**

| Field | Value |
|---|---|
| Product name | Development EC2 Instance |
| Owner | Cloud Operations |
| Description | Pre-approved development instance with enforced tagging and size limits |
| Template | Upload `dev-instance-product.yaml` |

3. **Access** tab → grant your IAM user or group access to the portfolio

### B2c — Provision as an End User

**Service Catalog** → **Products** → **Development EC2 Instance** → **Launch product**. Fill the parameters and launch.

> This is the fundamental difference from a ServiceNow catalogue item: Service Catalog *provisions the actual resource* through CloudFormation. There is no human fulfilment step. The guardrails — allowed instance types, mandatory tags, environments excluded — are enforced by the template, so a user cannot exceed them even if they try.

---

## Step B3 — Change Management with Systems Manager Change Manager

### B3a — Configure Change Manager

**Systems Manager** → **Change Manager** → **Settings**
- Set the approval mode and designate approvers (an IAM user, group, or role)

### B3b — Create a Change Template

**Change Manager** → **Templates** → **Create template**

| Field | Value |
|---|---|
| Name | `ec2-patch-deployment` |
| Runbook | `AWS-RunPatchBaseline` |
| Approval required | Yes — 1 approver |
| Change type | Normal |

### B3c — Submit and Approve a Change Request

1. **Change Manager** → **Requests** → **Create request** → select `ec2-patch-deployment`
2. Fill in the scheduled window, targets (`lab-ad-dc01`, `lab-splunk-idx`), and justification
3. Submit → the request enters **Pending approval**
4. Approve as the designated approver
5. The runbook executes automatically at the scheduled time

> **This is the control ServiceNow cannot enforce on its own.** In Track A, ServiceNow records the approval and a human then goes and does the work — nothing technically stops them acting without approval. In Track B, Change Manager *is* the execution path: the runbook physically will not run until the approval is recorded. Approval and execution are the same system. Being able to explain that distinction is a strong interview answer on change control.

---

## Step B4 — CMDB with AWS Config

### B4a — Enable AWS Config

**Config Console** → **Get started**

| Field | Value |
|---|---|
| Resource types | Record all resources supported in this region |
| S3 bucket | `lab-config-<account-id>` *(create new)* |
| IAM role | Create AWS Config service-linked role |

### B4b — Query the Inventory

Config's **Advanced queries** provide SQL-style access to your resource inventory — the CMDB query layer.

```sql
-- All EC2 instances with type, state, and Name tag
SELECT
  resourceId,
  resourceType,
  configuration.instanceType,
  configuration.state.name,
  tags
WHERE
  resourceType = 'AWS::EC2::Instance'
```

```sql
-- Untagged resources — a common compliance finding
SELECT
  resourceId,
  resourceType
WHERE
  resourceType = 'AWS::EC2::Instance'
  AND tags.key NOT IN ('CostCentre')
```

### B4c — Add Compliance Rules

**Config** → **Rules** → **Add rule**. Useful managed rules for this lab:

| Rule | What It Checks |
|---|---|
| `required-tags` | Resources carry mandatory tags (CostCentre, Environment) |
| `ec2-instance-no-public-ip` | No unintended public IPs |
| `incident-response-plan-exists` | An Incident Manager response plan is configured |
| `restricted-ssh` | No security group allows SSH from `0.0.0.0/0` |

> **Why Config beats a manual CMDB:** a traditional CMDB drifts the moment someone provisions a resource without updating it. Config records every resource automatically and continuously, tracks every configuration change over time, and lets you query the state of your estate at any past point. That "as of last Tuesday" capability is invaluable during incident investigation.

---

## Verification

| Check | How to Verify |
|---|---|
| ServiceNow instance active | Instance URL loads and you can log in |
| Incident created and closed | `INC` number exists with work notes and resolution notes populated |
| CloudWatch alarm exists | `aws cloudwatch describe-alarms --alarm-names lab-splunk-idx-StatusCheckFailed` |
| Lambda integration working | Force alarm state; a new incident appears in ServiceNow within 30 seconds |
| Lambda has no errors | CloudWatch Logs → `/aws/lambda/sn-incident-creator` shows a 201 response |
| Catalogue item published | Item appears in the ServiceNow Service Catalog portal preview |
| Change approved | Change record state is `Scheduled` with an approval recorded |
| Reports return data | All three reports render with populated charts |
| Incident Manager response plan | `aws ssm-incidents list-response-plans` returns `ec2-critical-response` |
| Service Catalog product launches | Product provisions successfully; EC2 instance appears with all required tags |
| Change Manager gate works | Runbook does **not** execute until approval is recorded |
| Config recording | `aws configservice get-status` shows the recorder as ON |

---

## Troubleshooting

| Problem | Fix |
|---|---|
| PDI hibernated | Log into developer.servicenow.com and click **Wake Instance**. Takes ~10 minutes. |
| PDI reclaimed | Request a new instance (free). Work is not recoverable — export catalogue items and reports periodically. |
| Lambda returns 401 from ServiceNow | Verify the `aws_integration` user has **Web service access only** checked and holds the `itil` role. Confirm the password env var has no trailing whitespace. |
| Lambda times out | Increase the timeout from the 3-second default to 30 seconds. ServiceNow API calls to a PDI can be slow on cold instances. |
| Lambda cannot reach ServiceNow | If the Lambda is in a VPC it needs a NAT Gateway for outbound internet. For this lab, leave the Lambda outside a VPC. |
| No incident created but Lambda succeeded | Check the ServiceNow REST API response body in CloudWatch Logs. A 201 with an empty result usually means an invalid `assignment_group` value — the group must exist by exact name. |
| CloudWatch alarm never fires | `set-alarm-state` only holds until the next metric evaluation. For testing, that window is sufficient; check Lambda logs immediately. |
| Service Catalog launch fails | Check the CloudFormation stack events for the specific error. Most commonly the launch role lacks `ec2:RunInstances`. |
| Change Manager runbook does not run | Verify the change was approved *and* the scheduled window has arrived. Change Manager will not execute outside the specified window. |
| Config shows no resources | Recording can take 10–15 minutes for initial discovery. Verify the S3 bucket policy allows Config to write. |

---

## Career Relevance

| Role | How This Lab Applies |
|---|---|
| IT Support / Help Desk | Creating and resolving incidents is the core daily task from week one |
| Cloud Operations Engineer | Alarm-to-incident automation, change control, and runbook execution are the job |
| Sysadmin | Change management — logging, approving, and documenting infrastructure changes |
| SRE | Incident Manager, post-incident analysis, and MTTR reporting map directly to SRE practice |
| ITSM Platform Administrator | Catalogue design, workflow building, and integration development |
| Cloud Security Engineer | Config rules as continuous compliance; change control as a security control |

---

## Tools and Services Used

| Tool / Service | Purpose | Cost |
|---|---|---|
| ServiceNow PDI | Full ITSM platform | Free, no expiration (hibernates when idle) |
| AWS CloudWatch | Metric monitoring and alarms | Free tier: 10 alarms |
| Amazon EventBridge | Event routing | Free tier: generous |
| Amazon SNS | Notification fan-out | Free tier: 1M requests |
| AWS Lambda | ServiceNow API integration | Free tier: 1M requests/month |
| SSM Incident Manager | AWS-native incident management | Free tier available |
| AWS Service Catalog | Governed self-service provisioning | No charge for the service itself |
| SSM Change Manager | Approval-gated change execution | Included with Systems Manager |
| AWS Config | Continuous resource inventory (CMDB) | Per configuration item recorded |

---

## Repository Structure

```
lab-04-itsm-aws/
├── README.md                              ← This file
├── lambda/
│   └── sn_incident_creator.py             ← CloudWatch → ServiceNow integration
├── cloudformation/
│   └── dev-instance-product.yaml          ← Service Catalog product template
├── config/
│   ├── advanced-queries.sql               ← CMDB inventory queries
│   └── required-tags-rule.json            ← Config rule definition
├── runbooks/
│   └── ec2-patch-deployment.md            ← Change Manager runbook documentation
└── screenshots/
    ├── incident-with-worknotes.png
    ├── auto-created-incident.png           ← Proof the integration fires
    ├── service-catalog-item.png
    ├── change-approval-workflow.png
    ├── incident-manager-timeline.png
    └── config-cmdb-query.png
```
---

*Part of the AWS Cloud Security & Operations Portfolio Series — hands-on labs documenting real-world cloud infrastructure, identity management, and information security practice on AWS.*
