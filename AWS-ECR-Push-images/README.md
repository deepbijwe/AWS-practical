# 🐳 Pushing Docker Images to AWS ECR from an EC2 Instance

This guide walks through the complete process of setting up an EC2 instance with the right IAM permissions and pushing a Docker image to Amazon Elastic Container Registry (ECR).

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Step 1 – Launch an EC2 Instance](#step-1--launch-an-ec2-instance)
4. [Step 2 – Create an IAM Policy for ECR](#step-2--create-an-iam-policy-for-ecr)
5. [Step 3 – Create an IAM Role and Attach the Policy](#step-3--create-an-iam-role-and-attach-the-policy)
6. [Step 4 – Attach the IAM Role to the EC2 Instance](#step-4--attach-the-iam-role-to-the-ec2-instance)
7. [Step 5 – Install AWS CLI and Docker on the EC2 Instance](#step-5--install-aws-cli-and-docker-on-the-ec2-instance)
8. [Step 6 – Pull a Docker Image from Docker Hub](#step-6--pull-a-docker-image-from-docker-hub)
9. [Step 7 – Authenticate Docker with ECR](#step-7--authenticate-docker-with-ecr)
10. [Step 8 – Tag and Push the Image to ECR](#step-8--tag-and-push-the-image-to-ecr)
11. [Verify the Push](#verify-the-push)

---

## Architecture Overview

```
Docker Hub  ──pull──►  EC2 Instance  ──push──►  Amazon ECR
                            │
                       IAM Role (ECR Full Access)
```

**AWS Services used:**
- **EC2** – Virtual machine to run Docker commands
- **IAM** – Policy and Role to grant EC2 access to ECR
- **ECR** – Private container registry to store Docker images

---

## Prerequisites

- An active AWS account
- An EC2 instance running Ubuntu (or any Linux distro)
- SSH access to the EC2 instance
- An ECR private repository already created

---

## Step 1 – Launch an EC2 Instance

Launch or use an existing EC2 instance. In this guide, the instance is named **test-ECR** (`t3.micro`, `us-east-1d`).

The instance should be in the **Running** state with all status checks passing before proceeding.

![EC2 Instance Running]

**Key instance details recorded:**
- **Instance ID:** `i-07951fg10dae919d8`
- **Public IP:** `13.216.244.153`
- **Private IP:** `172.31.32.134`
- **Region:** `us-east-1`

---

## Step 2 – Create an IAM Policy for ECR

You need a policy that grants the EC2 instance full access to ECR. Navigate to **IAM → Policies → Create policy**.

### 2a. Browse existing ECR policies (optional)

Search for `ecr` in the IAM Policies console to see what AWS-managed policies exist.



### 2b. Create a custom policy

In the policy editor, select **Elastic Container Registry** as the service and grant **All actions (`ecr:*`)**.



The policy grants the following access levels:
- **List** – 4/4 actions
- **Read** – 21/21 actions
- **Write** – 29/29 actions
- **Permissions management** – 4/4 actions
- **Tagging** – 2/2 actions

Name the policy **`new-ECR`** and create it.

### 2c. Policy created successfully


The new policy (`new-ECR`) has been created with **Full access** to Elastic Container Registry on **All resources**.

- **Type:** Customer managed
- **ARN:** `arn:aws:iam::942454901160:policy/new-ECR`

---

## Step 3 – Create an IAM Role and Attach the Policy

Navigate to **IAM → Roles → Create role**.

### 3a. Select trusted entity

Select **AWS service** as the trusted entity type and choose **EC2** as the use case. This allows the EC2 instance to assume the role and call AWS services on your behalf.


### 3b. Attach the ECR policy

Search for `new` in the permissions policies search box and select the **`new-ECR`** policy you just created.


### 3c. Role created successfully

Name the role **`ECR-ROLE_EC2`** and finish creation.



**Role details:**
- **Role name:** `ECR-ROLE_EC2`
- **ARN:** `arn:aws:iam::942454901160:role/ECR-ROLE_EC2`
- **Instance profile ARN:** `arn:aws:iam::942454901160:instance-profile/ECR-ROLE_EC2`

---

## Step 4 – Attach the IAM Role to the EC2 Instance

Go to **EC2 → Instances**, select your instance, then navigate to **Actions → Security → Modify IAM role**.

Select **`ECR-ROLE_EC2`** from the dropdown and click **Update IAM role**.



After updating, the role is successfully attached. The instance now has the permissions defined in the `new-ECR` policy. The final role has 2 permission policies attached:

| Policy | Type |
|---|---|
| `AmazonElasticContainerRegistryPublicFullAccess` | AWS managed |
| `new-ECR` | Customer managed |



---

## Step 5 – Install AWS CLI and Docker on the EC2 Instance

SSH into your EC2 instance and install the required tools.

```bash
# SSH into the instance
ssh -i your-key.pem ubuntu@<your-public-ip>

# Switch to root
sudo -i

# Update package lists
apt update

# Install AWS CLI
apt install -y awscli

# Install Docker
apt install -y docker.io

# Start and enable Docker
systemctl start docker
systemctl enable docker
```

Verify installations:

```bash
aws --version
docker --version
```

---

## Step 6 – Pull a Docker Image from Docker Hub

Pull the **nginx** image from Docker Hub as the base image to push to ECR.

```bash
# Pull the nginx image
docker pull nginx

# Verify the image is downloaded
docker images
```

[Docker Pull nginx]

The nginx image is downloaded successfully:
- **Image:** `nginx:latest`
- **Image ID:** `6e23479198b9`
- **Size:** 240 MB (65.8 MB compressed)

---

## Step 7 – Authenticate Docker with ECR

Before pushing, authenticate your Docker client with your ECR registry using the AWS CLI. Since the EC2 instance has the IAM role attached, no explicit credentials are needed.

```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  942454901160.dkr.ecr.us-east-1.amazonaws.com
```

Expected output:
```
Login Succeeded
```

> 💡 The IAM role attached to the EC2 instance automatically provides the credentials — no `aws configure` needed.

---

## Step 8 – Tag and Push the Image to ECR

### 8a. View push commands from ECR console

In the ECR console, open your repository and click **View push commands** to see the exact commands for your repo.

![ECR Push Commands](images/11-ecr-push-commands.png)

### 8b. Tag the image

Tag the local `nginx:latest` image with your ECR repository URI:

```bash
# Tag for ECR (latest)
docker tag nginx:latest 942454901160.dkr.ecr.us-east-1.amazonaws.com/cloudblitz/deepbijwe:latest

# Tag with a version label
docker tag nginx:latest 942454901160.dkr.ecr.us-east-1.amazonaws.com/cloudblitz/deepbijwe:ubuntu-images.v2
```

### 8c. Push the image

```bash
# Push the image to ECR
docker push 942454901160.dkr.ecr.us-east-1.amazonaws.com/cloudblitz/deepbijwe:ubuntu-images.v2
```


The push completes successfully. You'll see a confirmation like:

```
ubuntu-images.v2: digest: sha256:cdb5fd928fced577cfecf12c8966e830fcdf42ee481fb0b91904eeddc2fe5eff size: 424
```

> ⚠️ **Note:** If you see a `403 Forbidden` error on the first push attempt, re-run the `docker login` command to refresh the token, then push again.

---

## Verify the Push

Navigate to **ECR → Private registry → Repositories → cloudblitz/deepbijwe → Images** to confirm the images are present.


Both images are now stored in ECR:

| Image Tag | Size | Created At |
|---|---|---|
| `ubuntu-images.v1`, `ubuntu-images.v2` | 29.74 MB | April 25, 2026 |
| `latest` | 62.95 MB | April 25, 2026 |

---

## 🧹 Cleanup (Optional)

To avoid incurring charges, clean up the resources when done:

```bash
# Terminate EC2 instance
# (from EC2 console → Instance state → Terminate)

# Delete ECR images
aws ecr batch-delete-image \
  --repository-name cloudblitz/deepbijwe \
  --image-ids imageTag=latest \
  --region us-east-1

# Delete ECR repository
aws ecr delete-repository \
  --repository-name cloudblitz/deepbijwe \
  --force \
  --region us-east-1
```

Also delete the IAM role (`ECR-ROLE_EC2`) and policy (`new-ECR`) from the IAM console.

---

## 📝 Summary

| Step | Action |
|---|---|
| 1 | Launched EC2 instance (`test-ECR`, `t3.micro`) |
| 2 | Created IAM policy `new-ECR` with full ECR access |
| 3 | Created IAM role `ECR-ROLE_EC2` with EC2 trusted entity |
| 4 | Attached IAM role to the EC2 instance |
| 5 | Installed `awscli` and `docker.io` on the instance |
| 6 | Pulled `nginx:latest` from Docker Hub |
| 7 | Authenticated Docker with ECR using `aws ecr get-login-password` |
| 8 | Tagged and pushed the image to ECR repository |

---

## 🔗 References

- [Amazon ECR Documentation](https://docs.aws.amazon.com/ecr/latest/userguide/what-is-ecr.html)
- [IAM Roles for EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html)
- [Docker CLI Reference](https://docs.docker.com/engine/reference/commandline/cli/)
- [AWS CLI ECR Commands](https://docs.aws.amazon.com/cli/latest/reference/ecr/index.html)
