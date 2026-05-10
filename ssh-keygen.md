#  🔑🔐 🔑 AWS Practical — SSH Key-Based User Access on Ubuntu EC2

> **Aim:** Provide local user-only access to an AWS Ubuntu instance using a public key (`.pub`) file, allowing SSH login with only user-level (non-root) privileges.

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Step 1 — Generate SSH Key Pair (Local Machine)](#step-1--generate-ssh-key-pair-local-machine)
4. [Step 2 — Launch Ubuntu EC2 Instance on AWS](#step-2--launch-ubuntu-ec2-instance-on-aws)
5. [Step 3 — Connect to the Instance](#step-3--connect-to-the-instance)
6. [Step 4 — Create a New User on the Server](#step-4--create-a-new-user-on-the-server)
7. [Step 5 — Set Correct Permissions](#step-5--set-correct-permissions)
8. [Step 6 — Copy Public Key to Server](#step-6--copy-public-key-to-server)
9. [Step 7 — SSH into the New User's Shell](#step-7--ssh-into-the-new-users-shell)
10. [Conclusion](#conclusion)

---

## 🧭 Overview

This practical demonstrates how to:
- Generate an SSH key pair on your local machine
- Create a non-root user on an AWS EC2 Ubuntu instance
- Grant that user SSH access **only** via a public key
- Restrict access to user-level privileges (no root/sudo access)

[SSH Key Flow Diagram]

```
Local Machine                        AWS EC2 (Ubuntu)
─────────────                        ────────────────
 key  (private key)  ──── SSH ────►  ~/.ssh/authorized_keys
 key.pub (public key) ──► copy ────► /home/username/.ssh/authorized_keys
```

---

## ✅ Prerequisites

| Requirement | Details |
|-------------|---------|
| Local OS | Windows / macOS / Linux |
| AWS Account | Active with EC2 access |
| Terminal | Command Prompt, PowerShell, Terminal, or MobaXterm |
| AWS EC2 | Ubuntu instance (t2.micro or higher) |

---

## Step 1 — Generate SSH Key Pair (Local Machine)

Open your **local terminal** (Windows CMD / PowerShell / macOS Terminal) and run:

```bash
ssh-keygen
```

You will be prompted to enter a filename. For example, name it `john`:

```
Generating public/private ed25519 key pair.
Enter file in which to save the key: john
Enter passphrase (empty for no passphrase): 
```

After successful execution, two files are created:

```
john        ← Private Key (keep this secret, never share!)
john.pub    ← Public Key  (this goes to the server)
```

> ⚠️ **Never share your private key (`john`) with anyone.**

---

## Step 2 — Launch Ubuntu EC2 Instance on AWS

1. Log in to the **AWS Management Console**
2. Navigate to **EC2 → Launch Instance**
3. Choose **Ubuntu Server** as the AMI
4. Select your instance type (e.g., `t2.micro`)
5. Under **Security Group**, add an **Inbound Rule**:

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| SSH  | TCP      | 22   | Anywhere (0.0.0.0/0) |

6. **Launch the instance** and note the **Public IP address**

---

## Step 3 — Connect to the Instance

Connect to your EC2 instance using the **default ubuntu user** and your **AWS key pair**:

```bash
# Using terminal
ssh -i your-aws-key.pem ubuntu@<your-public-ip>

# Or use MobaXterm (Windows) — 3rd party SSH GUI tool
```

> 💡 **MobaXterm** is a popular third-party tool for SSH on Windows with a built-in terminal and SFTP browser.

---

## Step 4 — Create a New User on the Server

Once connected, switch to root and create a new user:

```bash
# Switch to root
sudo -i

# Create a new user (replace 'john' with your desired username)
adduser john
```

Now set up the SSH directory for the new user. You can use either approach:

### Option A — Using Relative Path (from root)

```bash
mkdir /home/john/.ssh
touch /home/john/.ssh/authorized_keys
```

### Option B — Using Absolute Path (switching to user)

```bash
su - john        # Switch to the new user
mkdir .ssh
touch .ssh/authorized_keys
```

---

## Step 5 — Set Correct Permissions

SSH requires **strict permissions** on the `.ssh` directory and `authorized_keys` file. Incorrect permissions will cause SSH to reject the key.

![Permissions Structure](images/ssh-permissions-diagram.svg)

### If using Relative Path (as root):

```bash
chmod 700 /home/john/.ssh
chmod 600 /home/john/.ssh/authorized_keys
chown -R john:john /home/john
```

### If using Absolute Path (as the user):

```bash
chmod 700 .ssh
chmod 600 .ssh/authorized_keys
chown -R john:john ~
```

| Path | Permission | Meaning |
|------|-----------|---------|
| `.ssh/` | `700` | Owner can read/write/execute; no one else |
| `authorized_keys` | `600` | Owner can read/write; no one else |

---

## Step 6 — Copy Public Key to Server

### On your Local Machine:

Open `john.pub` in Notepad (Windows) or any text editor and **copy the entire content**. It looks like:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJ4MSB5vmjSC6MOhzfQKCIf3XwGvW7bk+aG0l2J6weh7 user@VICTUS
```

### On the Server:

Paste the copied public key into the `authorized_keys` file:

```bash
# Open the file with nano or vim
nano /home/john/.ssh/authorized_keys

# Paste the public key content, then save and exit
```

Verify the key was added correctly:

```bash
cat /home/john/.ssh/authorized_keys
```

**Expected Output:**
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJ4MSB5vmjSC6MOhzfQKCIf3XwGvW7bk+aG0l2J6weh7 user@VICTUS
```

---

## Step 7 — SSH into the New User's Shell

Back on your **local machine**, use the private key to log in as the new user:

```bash
# Syntax
ssh -i <private-key-file> <username>@<public-ip>

# Example
ssh -i john john@16.112.64.140
```

When prompted, type `yes` to accept the host fingerprint.

```
The authenticity of host '16.112.64.140' can't be established.
Are you sure you want to continue connecting (yes/no)? yes

Welcome to Ubuntu 22.04 LTS
john@ip-172-xx-xx-xx:~$     ← You are now in john's home directory!
```

> 🎉 **Congratulations! You have successfully logged in as a non-root user using SSH key authentication.**

---

## 📁 File & Permission Summary

```
/home/john/
├── .ssh/                    (chmod 700, owned by john)
│   └── authorized_keys      (chmod 600, owned by john)
                              ↑ contains john.pub content
```

---

## 🏁 Conclusion

This practical demonstrates how **enterprises** can provide secure, restricted SSH access to employees:

- ✅ Employees get **user-level shell access only** — no root/sudo privileges
- ✅ Authentication is done via **SSH key pair** — no passwords needed
- ✅ The **private key never leaves the user's machine** — more secure than passwords
- ✅ Access can be **revoked instantly** by removing the public key from `authorized_keys`
- ✅ Each user is **isolated** in their own home directory

This is a standard DevOps and cloud security practice used widely in production environments.

---

## 🔧 Commands Quick Reference

```bash
# Local Machine
ssh-keygen                                  # Generate key pair
ssh -i john john@<public-ip>               # SSH into user shell

# Server (as root)
adduser john                                # Create user
mkdir /home/john/.ssh                      # Create .ssh directory
touch /home/john/.ssh/authorized_keys      # Create authorized_keys file
chmod 700 /home/john/.ssh                  # Set directory permission
chmod 600 /home/john/.ssh/authorized_keys  # Set file permission
chown -R john:john /home/john              # Set ownership
cat /home/john/.ssh/authorized_keys        # Verify key
```

---

*Practical completed as part of AWS & Linux administration learning.*