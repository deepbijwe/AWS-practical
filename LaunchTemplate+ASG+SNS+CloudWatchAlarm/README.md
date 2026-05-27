# 🔦 EC2 Auto Scaling with CloudWatch Alarm & SNS Notification

**Region:** Asia Pacific (Mumbai) — `ap-south-1`
**Account:** 942454901160

---

## Architecture Overview

The lab demonstrates end-to-end auto scaling driven by CPU load:

1. A **Launch Template** defines the EC2 configuration (AMI, instance type, key pair, security group).
2. An **Auto Scaling Group (ASG)** spans three availability zones and maintains a desired capacity of 2 instances (min 1 / max 4).
3. A **Target Tracking Policy** keeps average CPU utilization at 15%.
4. A **CloudWatch Alarm** fires when CPU ≥ 15% and triggers both the scale-out action and an SNS email alert.
5. When CPU drops back below the target, the ASG automatically **scales in** to the minimum capacity.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Auto Scaling Group: My-ASG-test                 │
│  Desired: 2 · Min: 1 · Max: 4 · VPC: default                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │ ap-south-1a  │  │ ap-south-1b  │  │ ap-south-1c  │               │
│  │  t3.small    │  │  t3.small    │  │  t3.small    │               │
│  └──────────────┘  └──────────────┘  └──────────────┘               │
└─────────────┬───────────────────────────────────────────────────────┘
              │ CPU metric (CloudWatch)
              ▼
   ┌──────────────────────┐       ┌─────────────────────┐
   │  CloudWatch Alarm    │─────▶│  SNS Topic           │──▶ Email
   │  CPU ≥ 15% trigger   │       │  MY-SNS-TEST        │
   └──────┬───────────────┘       └─────────────────────┘
          │ scale-out / scale-in
          ▼
   ┌──────────────────────┐
   │  Target Tracking     │
   │  Policy (15% CPU)    │
   └──────────────────────┘
```

---

## Step 1 — Create a Launch Template

Navigate to **EC2 → Launch Templates → Create launch template**.

| Setting | Value |
|---|---|
| Launch template name | `My-EC2-Template1` |
| Template version description | `v1` |
| AMI | Ubuntu Server 26.04 LTS (HVM) · `ami-07a00cf47dbbc844c` |
| Architecture | 64-bit (x86) |
| Instance type | `t3.small` |
| Key pair | `my-mumbai` |
| Subnet | Default VPC · `subnet-0481dc1786be08eb4` (ap-south-1a) |
| Security group | Create new — `MyWebServerGroup` |

**Inbound security group rules:**

| Rule | Protocol | Port | Source |
|---|---|---|---|
| SSH | TCP | 22 | 0.0.0.0/0 |
| HTTP | TCP | 80 | 0.0.0.0/0 |

> **Note:** The AWS console warns that `0.0.0.0/0` allows all IP addresses. For production, restrict SSH to your known IP range.

Click **Create launch template**. The template `lt-0f7fb977af54a98c8` is created with default version 1.

---

## Step 2 — Create an Auto Scaling Group

Navigate to **EC2 → Auto Scaling Groups → Create Auto Scaling group**.

### Step 2.1 — Choose launch template

| Setting | Value |
|---|---|
| Auto Scaling group name | `My-ASG-test` |
| Launch template | `My-EC2-Template1` (Version: Default 1) |

### Step 2.2 — Choose instance launch options

| Setting | Value |
|---|---|
| VPC | `vpc-06e69e0c7769f41da` (172.31.0.0/16 · Default) |
| Availability Zones | `ap-south-1a`, `ap-south-1b`, `ap-south-1c` |

### Step 2.3 — Integrate with other services (optional)

| Setting | Value |
|---|---|
| Load balancer | No load balancer |
| VPC Lattice | No VPC Lattice service |

### Step 2.4 — Configure group size and scaling

| Setting | Value |
|---|---|
| Desired capacity | `2` |
| Min desired capacity | `1` |
| Max desired capacity | `4` |
| Automatic scaling | No scaling policies (added separately below) |

### Step 2.5 — Add notifications (optional)

| Setting | Value |
|---|---|
| SNS topic | `MY-SNS-TEST` |
| Recipient | `deepbijwe90@gmail.com` |
| Event types | Launch, Terminate, Replace root volume, Fail to launch, Fail to terminate, Fail to replace root volume |

Click **Create Auto Scaling group**. The ASG status shows **Updating capacity** while the 2 desired instances launch across the selected availability zones.

**Verification:** In EC2 → Instances, confirm two `t3.small` instances are in the `Running` state.

---

## Step 3 — Create a Dynamic Scaling Policy

Navigate to **EC2 → Auto Scaling Groups → My-ASG-test → Automatic scaling → Create dynamic scaling policy**.

| Setting | Value |
|---|---|
| Policy type | Target tracking scaling |
| Scaling policy name | `Target Tracking Policy` |
| Metric type | Average CPU utilization |
| Target value | `15` |
| Instance warmup | `30` seconds |

Click **Create**. The ASG will now automatically add or remove instances to keep average CPU at 15%.

---

## Step 4 — Create an SNS Topic

Navigate to **SNS → Topics → Create topic**.

| Setting | Value |
|---|---|
| Type | Standard |
| Name | `MY-SNS-TEST` |

After creation, add a subscription:

| Setting | Value |
|---|---|
| Protocol | Email |
| Endpoint | `deepbijwe90@gmail.com` |

Confirm the subscription by clicking the link in the confirmation email sent by AWS.

---

## Step 5 — Create a CloudWatch Alarm

Navigate to **CloudWatch → Alarms → Create alarm**.

### Select metric

1. Click **Select metric** → **EC2** → **By Auto Scaling Group**.
2. Find the metric for `My-ASG-test` with metric name **CPUUtilization** and select it.

### Configure metric and conditions

| Setting | Value |
|---|---|
| Metric name | `CPUUtilization` |
| Auto Scaling group name | `My-ASG-test` |
| Statistic | Average |
| Period | 5 minutes |
| Threshold type | Static |
| Condition | Greater than or equal to (`>=`) |
| Threshold value | `15` |

### Configure actions

| Setting | Value |
|---|---|
| Alarm state trigger | In alarm |
| Send notification to | `MY-SNS-TEST` (existing topic) |

### Name and create

| Setting | Value |
|---|---|
| Alarm name | `test-alarm-my` |

Click **Create alarm**. The alarm starts in **OK** state until CPU crosses 15%.

---

## Step 6 — Stress Test & Observe Auto Scaling

SSH into one of the running instances and install the `stress` utility.

```bash
apt update && apt install stress -y
```

Run the stress command to push CPU above 15%:

```bash
# Light initial test (1 CPU, 60 seconds)
stress --cpu 1 --timeout 60

# Heavy load to sustain the alarm (8 CPUs, I/O, memory, 500 seconds)
stress --cpu 8 --io 2 --vm 2 --vm-bytes 256M --hdd 1 --timeout 500
```

### What to observe

| Event | What happens |
|---|---|
| CPU ≥ 15% sustained | CloudWatch alarm transitions **OK → IN ALARM** |
| Alarm fires | ASG launches new EC2 instances (up to max 4) |
| SNS notification sent | Email arrives: `ALARM: "test-alarm-my" in Asia Pacific (Mumbai)` |
| CPU < 15% sustained | Alarm transitions **IN ALARM → OK** |
| Instances terminate | ASG scales in back to min capacity (1) |

**Observed email notification details:**

```
Alarm Name:        test-alarm-my
State Change:      OK -> ALARM
Reason:            Threshold Crossed: 1 out of the last 1 datapoints
                   [15.029...] was greater than or equal to 15.0
Timestamp:         Wednesday 27 May, 2026 03:55:49 UTC
AWS Account:       942454901160
Alarm ARN:         arn:aws:cloudwatch:ap-south-1:942454901160:alarm:test-alarm-my
```

**Scale-out observed:** EC2 instances jumped from 2 to 5 running instances across `ap-south-1a` and `ap-south-1b`.

**Scale-in observed:** Once the stress commands were stopped and CPU dropped below 15%, all excess instances transitioned to **Terminated**, leaving only the minimum 1 instance running.

---

## Resource Summary

| Resource | Name / ID |
|---|---|
| Launch Template | `My-EC2-Template1` (`lt-0f7fb977af54a98c8`) |
| Auto Scaling Group | `My-ASG-test` |
| ASG ARN | `arn:aws:autoscaling:ap-south-1:942454901160:autoScalingGroup:1dd6fb5e-a18c-4cc5-a219-a7de89ef7116:autoScalingGroupName/My-ASG-test` |
| Dynamic Scaling Policy | `Target Tracking Policy` (CPU target: 15%) |
| SNS Topic | `MY-SNS-TEST` |
| CloudWatch Alarm | `test-alarm-my` |
| AMI | Ubuntu 26.04 LTS · `ami-07a00cf47dbbc844c` |
| Instance Type | `t3.small` |
| Region | `ap-south-1` (Asia Pacific · Mumbai) |

---

## Key Concepts

**Launch Template** — A reusable configuration blueprint (AMI, instance type, key pair, security group, storage) used by the ASG to launch identical instances.

**Auto Scaling Group** — Manages the fleet of EC2 instances. The ASG ensures the number of running instances stays between the configured minimum and maximum, and it distributes instances across availability zones for high availability.

**Target Tracking Policy** — A scaling policy that automatically adjusts instance count to maintain a specified metric target (here: average CPU = 15%). Scale-out happens when CPU exceeds the target; scale-in when CPU is sustainably below it.

**CloudWatch Alarm** — Monitors a CloudWatch metric and changes state (`OK` / `IN ALARM` / `INSUFFICIENT DATA`) when the metric crosses a defined threshold. Alarms can trigger ASG scaling actions and SNS notifications.

**SNS (Simple Notification Service)** — A managed pub/sub messaging service. Subscribers (email, SMS, Lambda, etc.) receive messages whenever a notification is published to a topic — in this case, when the CloudWatch alarm fires.