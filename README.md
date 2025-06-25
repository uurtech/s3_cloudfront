# Web App Deployment Guide - S3 + CloudFront

🚀 **Complete guide for deploying your web application to AWS S3 with CloudFront CDN**

## 📂 Current Project Structure

```
co/
└── index.html    # Modern Hello World web application
```

## 🎯 What We're Building

You have a beautiful web application (`index.html`) with:
- Modern glass morphism design
- Responsive layout
- Gradient backgrounds
- Ready for production deployment

**Goal:** Deploy to AWS S3 with CloudFront for global distribution and HTTPS.

## 🔧 Prerequisites

### Required Tools
```bash
# Check if you have these installed
node --version    # Optional: for advanced features
aws --version     # Required: AWS CLI
```

### AWS Setup
1. **AWS Account** with appropriate permissions
2. **AWS CLI configured** with your credentials
3. **IAM permissions** for S3 and CloudFront

**Configure AWS CLI:**
```bash
aws configure
# Enter your:
# - AWS Access Key ID
# - AWS Secret Access Key  
# - Default region: us-east-1 (recommended for CloudFront)
# - Output format: json
```

## 🪣 Step 1: Create S3 Bucket

### 1.1 Create Bucket (CLI Method)

```bash
# Replace 'your-unique-app-name' with your desired bucket name
export BUCKET_NAME="your-unique-app-name-$(date +%s)"
echo "Creating bucket: $BUCKET_NAME"

# Create S3 bucket (bucket names must be globally unique)
aws s3 mb s3://$BUCKET_NAME --region us-east-1

# Verify bucket creation
aws s3 ls | grep $BUCKET_NAME
```

### 1.2 Enable Static Website Hosting

```bash
# Enable static website hosting
aws s3 website s3://$BUCKET_NAME \
    --index-document index.html \
    --error-document index.html

# Verify website configuration
aws s3api get-bucket-website --bucket $BUCKET_NAME
```

### 1.3 Set Bucket Policy for Public Access

Create a file called `bucket-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::BUCKET_NAME_PLACEHOLDER/*"
    }
  ]
}
```

Apply the policy:
```bash
# Replace BUCKET_NAME_PLACEHOLDER in the policy file
sed "s/BUCKET_NAME_PLACEHOLDER/$BUCKET_NAME/g" bucket-policy.json > temp-policy.json

# Apply the bucket policy
aws s3api put-bucket-policy \
    --bucket $BUCKET_NAME \
    --policy file://temp-policy.json

# Clean up temp file
rm temp-policy.json

# Verify policy applied
aws s3api get-bucket-policy --bucket $BUCKET_NAME
```

### 1.4 Upload Your Web App

```bash
# Upload index.html to S3
aws s3 cp index.html s3://$BUCKET_NAME/index.html \
    --content-type "text/html" \
    --cache-control "max-age=0,no-cache,no-store,must-revalidate"

# Verify upload
aws s3 ls s3://$BUCKET_NAME/

# Test S3 website endpoint
echo "Your S3 website URL: http://$BUCKET_NAME.s3-website-us-east-1.amazonaws.com"
```

**🧪 Test your S3 website:**
```bash
curl -I http://$BUCKET_NAME.s3-website-us-east-1.amazonaws.com
```

## 🌐 Step 2: Create CloudFront Distribution

### 2.1 Create Distribution Configuration

Create `cloudfront-config.json`:

```json
{
  "CallerReference": "web-app-deployment-2025",
  "Comment": "Web App CloudFront Distribution",
  "DefaultCacheBehavior": {
    "TargetOriginId": "S3-Website-Origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "TrustedSigners": {
      "Enabled": false,
      "Quantity": 0
    },
    "ForwardedValues": {
      "QueryString": false,
      "Cookies": {
        "Forward": "none"
      }
    },
    "MinTTL": 0,
    "DefaultTTL": 86400,
    "MaxTTL": 31536000,
    "Compress": true
  },
  "Origins": {
    "Quantity": 1,
    "Items": [
      {
        "Id": "S3-Website-Origin",
        "DomainName": "BUCKET_NAME_PLACEHOLDER.s3-website-us-east-1.amazonaws.com",
        "CustomOriginConfig": {
          "HTTPPort": 80,
          "HTTPSPort": 443,
          "OriginProtocolPolicy": "http-only"
        }
      }
    ]
  },
  "Enabled": true,
  "CustomErrorResponses": {
    "Quantity": 2,
    "Items": [
      {
        "ErrorCode": 403,
        "ResponsePagePath": "/index.html",
        "ResponseCode": "200",
        "ErrorCachingMinTTL": 300
      },
      {
        "ErrorCode": 404,
        "ResponsePagePath": "/index.html",
        "ResponseCode": "200",
        "ErrorCachingMinTTL": 300
      }
    ]
  },
  "PriceClass": "PriceClass_100"
}
```

### 2.2 Create CloudFront Distribution

```bash
# Replace placeholder in config file
sed "s/BUCKET_NAME_PLACEHOLDER/$BUCKET_NAME/g" cloudfront-config.json > temp-cloudfront-config.json

# Create CloudFront distribution
aws cloudfront create-distribution \
    --distribution-config file://temp-cloudfront-config.json > distribution-output.json

# Extract distribution ID and domain
export DISTRIBUTION_ID=$(cat distribution-output.json | grep '"Id"' | head -1 | cut -d'"' -f4)
export CLOUDFRONT_DOMAIN=$(cat distribution-output.json | grep '"DomainName"' | head -1 | cut -d'"' -f4)

echo "Distribution ID: $DISTRIBUTION_ID"
echo "CloudFront Domain: $CLOUDFRONT_DOMAIN"

# Clean up temp file
rm temp-cloudfront-config.json
```

### 2.3 Wait for Distribution to Deploy

```bash
# Check distribution status
aws cloudfront get-distribution --id $DISTRIBUTION_ID | grep '"Status"'

# Wait for deployment (this can take 10-15 minutes)
echo "Waiting for CloudFront distribution to deploy..."
aws cloudfront wait distribution-deployed --id $DISTRIBUTION_ID

echo "✅ Distribution deployed successfully!"
```

## 🚀 Step 3: Test Your Deployment

### 3.1 Access Your App

```bash
echo "🌍 Your app is now live at:"
echo "https://$CLOUDFRONT_DOMAIN"

# Test the deployment
curl -I https://$CLOUDFRONT_DOMAIN
```

### 3.2 Performance Verification

```bash
# Test global CDN performance
echo "Testing from different locations..."

# Test HTTPS redirect
curl -I http://$CLOUDFRONT_DOMAIN

# Test compression
curl -H "Accept-Encoding: gzip" -I https://$CLOUDFRONT_DOMAIN
```

## 🔄 Step 4: Update & Redeploy

### 4.1 Update Your App

When you make changes to `index.html`:

```bash
# Upload updated file
aws s3 cp index.html s3://$BUCKET_NAME/index.html \
    --content-type "text/html" \
    --cache-control "max-age=0,no-cache,no-store,must-revalidate"

# Invalidate CloudFront cache to see changes immediately
aws cloudfront create-invalidation \
    --distribution-id $DISTRIBUTION_ID \
    --paths "/*"

echo "🔄 Cache invalidation initiated. Changes will be live in 1-2 minutes."
```

### 4.2 Automated Deployment Script

Create `deploy.sh`:

```bash
#!/bin/bash
set -e

# Configuration (update these values)
BUCKET_NAME="your-unique-app-name"
DISTRIBUTION_ID="your-distribution-id"

# Colors for pretty output
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m'

echo -e "${YELLOW}🚀 Starting deployment...${NC}"

# Upload to S3
echo -e "${YELLOW}📦 Uploading to S3...${NC}"
aws s3 cp index.html s3://$BUCKET_NAME/index.html \
    --content-type "text/html" \
    --cache-control "max-age=0,no-cache,no-store,must-revalidate"

if [ $? -eq 0 ]; then
    echo -e "${GREEN}✅ Upload successful!${NC}"
else
    echo -e "${RED}❌ Upload failed!${NC}"
    exit 1
fi

# Invalidate CloudFront cache
echo -e "${YELLOW}🔄 Invalidating CloudFront cache...${NC}"
INVALIDATION_ID=$(aws cloudfront create-invalidation \
    --distribution-id $DISTRIBUTION_ID \
    --paths "/*" \
    --query 'Invalidation.Id' \
    --output text)

echo -e "${GREEN}✅ Invalidation created: $INVALIDATION_ID${NC}"
echo -e "${GREEN}🌍 Your app will be updated in 1-2 minutes${NC}"
echo -e "${GREEN}🔗 Live URL: https://$(aws cloudfront get-distribution --id $DISTRIBUTION_ID --query 'Distribution.DomainName' --output text)${NC}"
```

Make it executable and run:
```bash
chmod +x deploy.sh
# Update the variables in deploy.sh first, then:
./deploy.sh
```

## 🔧 Advanced Configuration

### Custom Domain (Optional)

If you want to use your own domain:

1. **Request SSL Certificate:**
```bash
aws acm request-certificate \
    --domain-name yourdomain.com \
    --domain-name www.yourdomain.com \
    --validation-method DNS \
    --region us-east-1
```

2. **Update CloudFront distribution** to use your domain and certificate

3. **Update DNS records** to point to CloudFront

### Performance Optimization

Add these optimizations to your HTML:

```html
<!-- Add to your index.html <head> section -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="dns-prefetch" href="//fonts.googleapis.com">

<!-- Add cache control meta tags -->
<meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate">
<meta http-equiv="Pragma" content="no-cache">
<meta http-equiv="Expires" content="0">
```

## 🔍 Troubleshooting

### Common Issues

**1. "Access Denied" Error:**
```bash
# Check bucket policy
aws s3api get-bucket-policy --bucket $BUCKET_NAME

# Make sure bucket is publicly readable
aws s3api put-bucket-acl --bucket $BUCKET_NAME --acl public-read
```

**2. CloudFront Shows Old Content:**
```bash
# Create cache invalidation
aws cloudfront create-invalidation \
    --distribution-id $DISTRIBUTION_ID \
    --paths "/*"
```

**3. HTTPS Not Working:**
```bash
# Check CloudFront configuration
aws cloudfront get-distribution-config --id $DISTRIBUTION_ID | grep -A5 -B5 "ViewerProtocolPolicy"
```

**4. Check Deployment Status:**
```bash
# S3 website endpoint
curl -I http://$BUCKET_NAME.s3-website-us-east-1.amazonaws.com

# CloudFront endpoint  
curl -I https://$CLOUDFRONT_DOMAIN

# Check distribution status
aws cloudfront get-distribution --id $DISTRIBUTION_ID --query 'Distribution.Status'
```

## 💰 Cost Estimation

**Monthly costs for a small website:**
- **S3 Storage:** ~$0.01-0.10 (depending on file size)
- **S3 Requests:** ~$0.01-0.05
- **CloudFront:** ~$0.50-2.00 (first 1TB free tier)
- **Data Transfer:** ~$0.09/GB after free tier

**Total estimated cost:** $1-5/month for typical small websites

## 🔒 Security Best Practices

1. **Enable CloudFront security headers:**
   - Add security headers to your HTML
   - Configure CloudFront response headers

2. **Regular updates:**
   - Keep AWS CLI updated
   - Monitor AWS security bulletins

3. **Access logging:**
```bash
# Enable CloudFront access logs
aws cloudfront get-distribution-config --id $DISTRIBUTION_ID
```

## 📋 Quick Reference Commands

```bash
# Save these commands for quick access
export BUCKET_NAME="your-bucket-name"
export DISTRIBUTION_ID="your-distribution-id"

# Quick deploy
aws s3 cp index.html s3://$BUCKET_NAME/ && aws cloudfront create-invalidation --distribution-id $DISTRIBUTION_ID --paths "/*"

# Check status
aws cloudfront get-distribution --id $DISTRIBUTION_ID --query 'Distribution.Status'

# Get live URL
echo "https://$(aws cloudfront get-distribution --id $DISTRIBUTION_ID --query 'Distribution.DomainName' --output text)"
```

---

## ✅ Success Checklist

- [ ] AWS CLI configured
- [ ] S3 bucket created and configured
- [ ] Website hosting enabled
- [ ] Files uploaded to S3
- [ ] CloudFront distribution created
- [ ] Distribution deployed (Status: Deployed)
- [ ] HTTPS working
- [ ] Cache invalidation tested
- [ ] Deployment script created

**🎉 Congratulations! Your web app is now deployed globally with AWS S3 + CloudFront!**

---

**Last Updated:** 2025-01-19  
**Project:** Web App Deployment Guide 