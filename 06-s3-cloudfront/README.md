# 06 — S3 Static Website Hosting & CloudFront CDN

> Host a static website on S3 with public access, custom bucket policy, and CloudFront as a CDN for global content delivery.

---

## Architecture

```
User (anywhere in the world)
         │
         ▼
  CloudFront Edge Location (nearest)
         │
         ▼
   S3 Bucket (Origin)
   Static Website Files
   index.html / assets
```

---

## What Was Built

- S3 bucket with static website hosting enabled
- Files and folders uploaded to the bucket
- Bucket policy configured for public read access
- CloudFront distribution pointing to S3 as origin
- Static website accessible via CloudFront URL globally

---

## Step-by-Step Implementation

### Step 1: Create S3 Bucket
```
Bucket name: your-static-website-bucket
Region: ap-northeast-1 (Tokyo)
Block Public Access: DISABLED (required for public website)
```
> ⚠️ You must disable "Block all public access" — this is not enough alone. A bucket policy is also required.

### Step 2: Upload Website Files
```
Upload: index.html, CSS, JS, images, folders
Storage class: Standard (default)
```

### Step 3: Enable Static Website Hosting
```
Bucket → Properties → Static website hosting → Enable
Index document: index.html
Error document: error.html (optional)
```

### Step 4: Add Bucket Policy (Public Read)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::your-static-website-bucket/*"
    }
  ]
}
```
> ⚠️ Common mistake: Disabling block public access without adding this policy still results in Access Denied.

### Step 5: Set Up CloudFront
```
Origin domain: S3 website endpoint (not S3 REST endpoint)
Viewer protocol policy: Redirect HTTP to HTTPS
Cache policy: CachingOptimized
Do NOT enable security protections (WAF) on free tier
```

### Step 6: Access the Website
```
S3 Website URL:    http://bucket-name.s3-website-region.amazonaws.com
CloudFront URL:    https://xxxxxxxxxxxx.cloudfront.net
```

---

## Key Concepts

### S3 REST Endpoint vs Website Endpoint
```
REST Endpoint (default):
  bucket-name.s3.amazonaws.com
  → Returns XML for directories, not index.html

Website Endpoint:
  bucket-name.s3-website-region.amazonaws.com
  → Returns index.html for directory requests ✅

Use Website Endpoint as CloudFront origin for static sites.
```

### Why CloudFront Over Direct S3 Access?
| Feature | S3 Direct | CloudFront |
|---|---|---|
| Latency | Higher (single region) | Lower (edge locations) |
| HTTPS | Not native on website endpoint | ✅ Native |
| Custom domain | Complex | Simple via ACM |
| DDoS protection | None | AWS Shield Standard |
| Cost for high traffic | Higher | Lower (caching) |

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| Block public access enabled | Access Denied | Disable in bucket settings |
| No bucket policy | Access Denied (even with block disabled) | Add public read policy |
| Using REST endpoint in CloudFront | Returns XML, not HTML | Use S3 website endpoint as origin |
| Security protections enabled | Extra cost | Disable WAF on non-production |

---

## Production Relevance
```
→ Frontend SPAs (React/Vue) deployed to S3 + CloudFront
→ CloudFront + ACM for custom domain HTTPS
→ Origin Access Control (OAC) to keep S3 private
   (CloudFront accesses S3 privately — no public bucket needed)
→ CloudFront cache invalidation after deployments:
   aws cloudfront create-invalidation --distribution-id XXXXX --paths "/*"
```

---

**Author:** Vinayaka K — DevOps Engineer | [LinkedIn](https://www.linkedin.com/in/vinayak-k)
