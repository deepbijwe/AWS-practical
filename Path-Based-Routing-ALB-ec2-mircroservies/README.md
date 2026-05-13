# Path-Based Routing Using AWS ALB → EC2

## Architecture Overview

```
                          Internet
                             │
                             ▼
                    ┌─────────────────┐
                    │   MY-ALB (ALB)  │
                    │ Internet-Facing │
                    │  HTTP:80        │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
     Path: /sell/*    Path: /cart/*    Default (/)
              │              │              │
              ▼              ▼              ▼
        ┌─────────┐   ┌─────────┐   ┌─────────────┐
        │ sell-tg │   │ cart-tg │   │ frontend-tg │
        └────┬────┘   └────┬────┘   └──────┬──────┘
             │              │              │
       ┌─────┴─────┐  ┌─────┴─────┐  ┌─────┴──────┐
       │           │  │           │  │            │
  sell-ui-1  sell-ui-2 cart-ui-1 cart-ui-2  FE-ec2-1  FE-ec2-2
  (nginx)    (nginx)   (nginx)   (nginx)   (nginx)   (nginx)
  /sell/     /sell/    /cart/    /cart/    /         /
```

**Flow Summary:**
- `ALB DNS /`         → Round-robin between `FE-ec2-1` and `FE-ec2-2`
- `ALB DNS /sell/`    → Round-robin between `Seel-ui-1` and `Seel-ui-2`
- `ALB DNS /cart/`    → Round-robin between `Cart-ec2-1` and `Cart-ec2-2`

---

## Prerequisites

| Item | Value |
|------|-------|
| Region | `ap-south-1` (Mumbai) |
| AMI | Ubuntu 26.04 (`ami-07a00cf47dbbc844c`) |
| Instance Type | `t3.micro` |
| Key Pair | `my-Mumbai.pem` |
| VPC | Default |
| Security Group | Ports **22** (SSH) and **80** (HTTP) open |

---

## Step 1 — Launch Frontend EC2 Instances (×2)

These instances serve the default `/` path.

**Launch Configuration:**

| Field | Value |
|-------|-------|
| Name | `FE-ec2-1` |
| Number of Instances | `2` |
| AMI | Ubuntu |
| Instance Type | `t3.micro` |
| Key Pair | `my-Mumbai.pem` |
| Security Group | Allow ports 22, 80 |

**User Data:**

```bash
#!/bin/bash
apt update -y
apt install nginx -y
echo "<h1> frontend-ui $HOSTNAME</h1>" > /var/www/html/index.html
systemctl start nginx
systemctl enable nginx
```

> AWS automatically names them `FE-ec2-1` and `FE-ec2-2`.

**Verify:** Copy each instance's public IP and open it in a browser. You should see:
```
frontend-ui ip-172-31-xx-xxx
```

---

## Step 2 — Launch Sell UI EC2 Instances (×2)

These instances serve traffic routed to `/sell/*`.

**Launch Configuration:**

| Field | Value |
|-------|-------|
| Name | `Seel-ui-1` |
| Number of Instances | `2` |
| AMI | Ubuntu |
| Instance Type | `t3.micro` |
| Key Pair | `my-Mumbai.pem` |
| Security Group | Allow ports 22, 80 |

**User Data:**

```bash
#!/bin/bash
apt update -y
apt install nginx -y
mkdir -p /var/www/html/sell
echo "<h1> sell-ui $HOSTNAME</h1>" > /var/www/html/sell/index.html
systemctl start nginx
systemctl enable nginx
```

**Verify:** Open `http://<public-ip>/sell/` — you should see:
```
sell-ui ip-172-31-xx-xxx
```

---

## Step 3 — Launch Cart UI EC2 Instances (×2)

These instances serve traffic routed to `/cart/*`.

**Launch Configuration:**

| Field | Value |
|-------|-------|
| Name | `Cart-ec2-1` |
| Number of Instances | `2` |
| AMI | Ubuntu |
| Instance Type | `t3.micro` |
| Key Pair | `my-Mumbai.pem` |
| Security Group | Allow ports 22, 80 |

**User Data:**

```bash
#!/bin/bash
apt update -y
apt install nginx -y
mkdir -p /var/www/html/cart
echo "<h1> cart-ui $HOSTNAME</h1>" > /var/www/html/cart/index.html
systemctl start nginx
systemctl enable nginx
```

**Verify:** Open `http://<public-ip>/cart/` — you should see:
```
cart-ui ip-172-31-xx-xxx
```

> ✅ At this point you should have **6 running instances** — 2 frontend, 2 sell, 2 cart.

---

## Step 4 — Create Target Group: `Frontend-tg`

EC2 → **Target Groups** → **Create target group**

| Field | Value |
|-------|-------|
| Target type | Instances |
| Name | `Frontend-tg` |
| Protocol | HTTP |
| Port | 80 |
| VPC | Default |

1. In the **Register targets** step, select `FE-ec2-1` and `FE-ec2-2`.
2. Click **Include as pending below**.
3. Click **Create target group**.

---

## Step 5 — Create Target Group: `sell-tg`

Repeat the same process:

| Field | Value |
|-------|-------|
| Name | `sell-tg` |
| Protocol | HTTP / Port 80 |
| Targets | `Seel-ui-1`, `Seel-ui-2` |

---

## Step 6 — Create Target Group: `cart-tg`

| Field | Value |
|-------|-------|
| Name | `cart-tg` |
| Protocol | HTTP / Port 80 |
| Targets | `Cart-ec2-1`, `Cart-ec2-2` |

---

## Step 7 — Create the Application Load Balancer

EC2 → **Load Balancers** → **Create load balancer** → **Application Load Balancer**

**Basic Configuration:**

| Field | Value |
|-------|-------|
| Name | `MY-alb` |
| Scheme | Internet-facing |
| IP Address Type | IPv4 |
| Availability Zones | `ap-south-1a` and `ap-south-1b` |

**Listener Configuration:**

| Field | Value |
|-------|-------|
| Protocol | HTTP |
| Port | 80 |
| Default Routing Action | Forward to target group |
| Default Target Group | `Frontend-tg` |

Click **Create load balancer** and wait until the state shows **Active**.

> ✅ **Verify with Resource Map:** In the ALB details page, open the **Resource map** tab. You should see the listener, all 3 target groups, and all 6 instances mapped out.

---

## Step 8 — Add Listener Rule for `/sell/*`

Navigate to: **MY-alb** → **HTTP:80 listener** → **Add rule**

**Step 1 — Add rule conditions:**

| Field | Value |
|-------|-------|
| Condition type | Path |
| Match pattern type | Value matching |
| Path condition value | `/sell/*` |

**Step 2 — Set action:**

| Field | Value |
|-------|-------|
| Routing action | Forward to target group |
| Target group | `sell-tg` |
| Priority | `1` |

Click **Save** → **Review and create** → **Create**.

---

## Step 9 — Add Listener Rule for `/cart/*`

Navigate to: **MY-alb** → **HTTP:80 listener** → **Add rule**

**Step 1 — Add rule conditions:**

| Field | Value |
|-------|-------|
| Condition type | Path |
| Match pattern type | Value matching |
| Path condition value | `/cart/*` |

**Step 2 — Set action:**

| Field | Value |
|-------|-------|
| Routing action | Forward to target group |
| Target group | `cart-tg` |
| Priority | `2` |

Click **Save** → **Review and create** → **Create**.

---

## Step 10 — Verify Listener Rules

Go to **MY-alb → HTTP:80 listener → Rules tab**.

You should see 3 rules in this order:

| Priority | Condition | Action |
|----------|-----------|--------|
| 1 | Path = `/sell/*` | Forward → `sell-tg` |
| 2 | Path = `/cart/*` | Forward → `cart-tg` |
| Default (Last) | If no other rule applies | Forward → `Frontend-tg` |

---

## Step 11 — Test Path-Based Routing

Copy the **DNS name** of `MY-alb` from the Load Balancers page (e.g., `my-alb-1403036916.ap-south-1.elb.amazonaws.com`).

| URL | Expected Response | Served By |
|-----|-------------------|-----------|
| `http://<ALB-DNS>/` | `frontend-ui ip-172-31-xx-xxx` | `FE-ec2-1` or `FE-ec2-2` |
| `http://<ALB-DNS>/sell/` | `sell-ui ip-172-31-xx-xxx` | `Seel-ui-1` or `Seel-ui-2` |
| `http://<ALB-DNS>/cart/` | `cart-ui ip-172-31-xx-xxx` | `Cart-ec2-1` or `Cart-ec2-2` |

> 🔄 **Refresh the page multiple times** — the hostname in the response will alternate between the two instances in each target group, demonstrating **round-robin load balancing**.

---

## Summary

You have successfully implemented **path-based routing** on an AWS Application Load Balancer:

- **6 EC2 instances** running nginx across 3 logical service groups.
- **3 Target Groups** (`Frontend-tg`, `sell-tg`, `cart-tg`) grouping instances by service.
- **1 ALB** (`MY-alb`) with **3 listener rules** routing HTTP traffic by URL path.
- Traffic is distributed in **round-robin** between the 2 instances in each target group.

This pattern is foundational for microservices architectures where a single load balancer routes to multiple backend services based on the request path.