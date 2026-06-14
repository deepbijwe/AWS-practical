# 🌐 Static Portfolio Website — AWS S3 + CloudFront + Route 53

> Hosted my personal Cloud DevOps portfolio at **[deepbijwe.in](https://deepbijwe.in)** using AWS S3 for storage, CloudFront as a global CDN, ACM for SSL/TLS, and Route 53 for DNS management.

---

## 🏗️ Architecture

```
User Request
     │
     ▼
Route 53 (DNS)
deepbijwe.in → CloudFront
     │
     ▼
CloudFront Distribution (Global CDN)
• HTTPS enforced
• OAC — private S3 access
• Custom error pages
• All edge locations
     │
     ▼
S3 Bucket (Origin)
deepbijwe.in (ap-south-1)
• index.html
• error.html
• Block public access ON
```

---

## ☁️ AWS Services Used

| Service | Purpose |
|---|---|
| **Amazon S3** | Static file storage (origin) |
| **Amazon CloudFront** | Global CDN, HTTPS, caching |
| **AWS Certificate Manager (ACM)** | Free SSL/TLS certificate |
| **Amazon Route 53** | DNS hosted zone + A/AAAA records |
| **AWS WAF** | Web Application Firewall (DDoS protection) |

---

## 📁 Project Files

```
deepbijwe.in/
├── index.html       # Portfolio homepage (33.4 KB)
└── error.html       # Custom 404 error page (10.2 KB)
```

---

## 🪜 Step-by-Step Implementation

### Step 1 — Route 53: Create Hosted Zone

- Navigated to **Route 53 → Hosted zones → Create hosted zone**
- Domain name: `deepbijwe.in`
- Type: **Public hosted zone**
- AWS auto-generated **4 NS records** and 1 SOA record

```
NS Records generated:
  ns-499.awsdns-62.com
  ns-654.awsdns-17.net
  ns-1939.awsdns-50.co.uk
  ns-1339.awsdns-39.org
```

---

### Step 2 — GoDaddy: Update Nameservers to AWS

- Logged into **GoDaddy → Domain → Nameservers → Edit**
- Selected **"I'll use my own nameservers"**
- Replaced GoDaddy's default nameservers with the 4 Route 53 NS values above
- Clicked **Save** — DNS delegation now points to AWS Route 53

> ⚠️ **DNS propagation** can take a few minutes to 48 hours globally.

---

### Step 3 — S3: Create Bucket & Upload Files

**Bucket configuration:**
- Region: `ap-south-1` (Asia Pacific — Mumbai)
- Bucket name: `deepbijwe.in`
- Bucket type: General purpose
- Namespace: Global namespace
- **Block all public access: ✅ ON** (CloudFront accesses it privately via OAC)
- Versioning: Disabled
- Encryption: SSE-S3 (AWS managed keys)

**Files uploaded:**

| File | Size | Description |
|---|---|---|
| `index.html` | 33.4 KB | Main portfolio page |
| `error.html` | 10.2 KB | Custom 404 error page |

**Static website hosting enabled:**
- Index document: `index.html`
- Error document: `error.html`

---

### Step 4 — ACM: Request SSL Certificate

> ⚠️ **Critical:** CloudFront requires SSL certificates to be in **us-east-1 (N. Virginia)**, not the bucket's region.

**Initial attempt (Mumbai — ap-south-1):**
- Requested certificate for `deepbijwe.in`
- Validation method: DNS validation
- Clicked **"Create records in Route 53"** — CNAME record auto-added
- Certificate issued ✅ but in wrong region (ap-south-1)

**Resolution — requested new certificate in us-east-1:**
- Switched region to **US East (N. Virginia)**
- Requested new certificate covering:
  - `deepbijwe.in`
  - `www.deepbijwe.in`
- DNS validation → Route 53 CNAME auto-created
- Certificate **Issued** ✅ in us-east-1

```
Certificate ARN:
arn:aws:acm:us-east-1:942454901160:certificate/4a31fc02-e2aa-4c5f-95c7-1ca25ed3dc8c

Covered domains:
  deepbijwe.in
  www.deepbijwe.in
```

---

### Step 5 — CloudFront: Create Distribution

Used the **new CloudFront wizard** (5-step flow):

**Step 1 — Get started:**
- Distribution name: `My-Distribution`
- Description: `my-portfolio`
- Distribution type: Single website or app
- Route 53 managed domain: `deepbijwe.in` → verified ✅ *"Domain managed by Route 53"*

**Step 2 — Specify origin:**
- Origin type: **Amazon S3**
- S3 origin: `deepbijwe.in.s3.ap-south-1.amazonaws.com`
- ⚠️ Dismissed the "Use website endpoint" banner — kept bucket endpoint for OAC
- **Allow private S3 bucket access to CloudFront** → ✅ Checked (OAC)
- Origin settings: Use recommended origin settings
- Cache settings: Use recommended cache settings tailored to S3

**Step 3 — Enable security (WAF):**
- WAF: **Do not enable security protections** (skipped to avoid extra cost for portfolio)

**Step 4 — Get TLS certificate:**
- Auto-detected certificate: `deepbijwe.in (4a31fc02...)` ✅
- Covered domains: `deepbijwe.in`, `www.deepbijwe.in`
- Source: Amazon (us-east-1)

**Step 5 — Review and create** → Distribution created ✅

```
Distribution ID:  E12C2C7EU86A4U
Domain:           d2exi5u2kk5gde.cloudfront.net
SSL Certificate:  deepbijwe.in (ACM — us-east-1)
Price class:      Use all edge locations (best performance)
Default root:     index.html
```

---

### Step 6 — S3 Bucket Policy: Allow CloudFront OAC Access

After the distribution was created, CloudFront required a bucket policy update. Applied the following policy to **S3 → deepbijwe.in → Permissions → Bucket Policy**:

```json
{
  "Version": "2008-10-17",
  "Id": "PolicyForCloudFrontPrivateContent",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::deepbijwe.in/*",
      "Condition": {
        "ArnLike": {
          "AWS:SourceArn": "arn:aws:cloudfront::942454901160:distribution/E12C2C7EU86A4U"
        }
      }
    }
  ]
}
```

> This ensures only the specific CloudFront distribution can read from the private S3 bucket — no public bucket access required.

---

### Step 7 — CloudFront: Custom Error Pages

Configured **CloudFront → Error pages** to serve custom `error.html` for S3 errors:

| HTTP Error Code | Response Page | Response Code |
|---|---|---|
| 403 | `/error.html` | 404 |
| 404 | `/error.html` | 404 |

> S3 returns **403** (not 404) when an object key doesn't exist on a private bucket — both rules are needed.

---

### Step 8 — CloudFront: Set Default Root Object

- CloudFront → Distribution → General → **Edit**
- Default root object: `index.html`
- Without this, visiting `https://deepbijwe.in` returns a 403 instead of the homepage

---

### Step 9 — Route 53: Add DNS Records

CloudFront auto-created A and AAAA alias records via the new wizard. Final Route 53 records:

| Record | Type | Alias | Routes To |
|---|---|---|---|
| `deepbijwe.in` | A | Yes | `d2exi5u2kk5gde.cloudfront.net` |
| `deepbijwe.in` | AAAA | Yes | `d2exi5u2kk5gde.cloudfront.net` |
| `deepbijwe.in` | NS | No | AWS nameservers |
| `deepbijwe.in` | SOA | No | AWS SOA |
| `_94890e2...` | CNAME | No | SSL validation record |

---

## 🐛 Errors Faced & Troubleshooting

### ❌ Issue 1: ACM Certificate in Wrong Region

**Problem:** Requested SSL certificate in `ap-south-1` (Mumbai) — same region as S3 bucket. CloudFront could not use it.

**Root Cause:** CloudFront is a global service and **only accepts ACM certificates from us-east-1 (N. Virginia)**.

**Fix:** Switched AWS console region to **US East (N. Virginia)** → requested a new certificate → it was auto-detected in the CloudFront wizard. ✅

---

### ❌ Issue 2: OAC Policy Not Auto-Applied

**Problem:** Could not find where to copy the bucket policy after distribution creation.

**Root Cause:** In the new CloudFront UI, the OAC policy prompt appears under **CloudFront → Origins → Edit origin**, not on the main dashboard.

**Fix:** Navigated to **Origins tab → selected S3 origin → Edit → copied the bucket policy** and pasted it manually into S3 Permissions. ✅

---

### ❌ Issue 3: "Route domains to CloudFront" Button Persisting

**Problem:** CloudFront General tab kept showing the "Route domains to CloudFront" button even after records appeared to be created.

**Root Cause:** Only the root domain A record existed; `www.deepbijwe.in` A/AAAA records were missing.

**Fix:** Manually created `www` A and AAAA alias records in Route 53 pointing to the CloudFront distribution. ✅

---

### ❌ Issue 4: "Use website endpoint" Banner Warning

**Problem:** When selecting the S3 bucket as CloudFront origin, a yellow banner appeared recommending to switch to the S3 website endpoint.

**Root Cause:** S3 had static website hosting enabled, which triggers this recommendation.

**Fix:** Ignored the banner and stayed on the **bucket endpoint** (not website endpoint). The website endpoint bypasses OAC and exposes a public URL — bucket endpoint + OAC is the correct secure approach. ✅

---

## ✅ Final Verification Checklist

| Check | Status |
|---|---|
| S3 bucket created with block public access | ✅ |
| `index.html` + `error.html` uploaded | ✅ |
| Static website hosting enabled | ✅ |
| ACM certificate issued in us-east-1 | ✅ |
| CloudFront distribution deployed | ✅ |
| OAC bucket policy applied | ✅ |
| Default root object set to `index.html` | ✅ |
| Custom error pages (403/404 → error.html) | ✅ |
| Route 53 A + AAAA alias records created | ✅ |
| GoDaddy nameservers updated to Route 53 | ✅ |
| HTTPS enforced (HTTP → HTTPS redirect) | ✅ |
| `https://deepbijwe.in` live | ✅ |
| `https://www.deepbijwe.in` live | ✅ |

---

## 💡 Key Learnings

- ACM certificates for CloudFront **must always be in us-east-1** regardless of where your S3 bucket is
- OAC (Origin Access Control) is the **modern replacement for OAI** — more secure, uses signed requests
- S3 returns **403 not 404** when objects don't exist on private buckets — always configure both error codes in CloudFront
- The new CloudFront wizard (2026) **auto-provisions DNS and TLS**, simplifying the process significantly
- GoDaddy → Route 53 NS delegation hands over full DNS control to AWS for seamless alias routing

---

## 🔗 Links

- 🌐 Live Site: [https://deepbijwe.in](https://deepbijwe.in)
- 💼 LinkedIn: [linkedin.com/in/deep-bijwe](https://linkedin.com/in/deep-bijwe)
- ⬡ GitHub: [github.com/deepbijwe](https://github.com/deepbijwe)
- ☁️ AWS Projects: [github.com/deepbijwe/AWS-projects](https://github.com/deepbijwe/AWS-projects)

---

*Crafted by **Deep Bijwe** — Cloud DevOps Engineer · AWS Certified (CLF-C02) · 2026*