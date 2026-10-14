# Web Services without Servers: S3 Static Website Hosting

Amazon S3 allows you to host a static website without the need for an EC2 instance or a web server like Nginx or Apache. This is an incredibly cost-effective and highly available solution for documentation sites, simple landing pages, or modern Single Page Applications (SPAs) built with React, Vue, or Angular. In a professional DevOps environment, S3 website hosting is typically integrated with CloudFront for global delivery and SSL/TLS security. This chapter provide a definitive exploration of the `aws_s3_bucket_website_configuration` resource and the technical patterns for deploying public-facing web content using Terraform.

## 1. The `aws_s3_bucket_website_configuration` Resource

Enabling a website on a bucket is done through a specific configuration resource that defines the "Index" and "Error" pages.

```hcl
resource "aws_s3_bucket_website_configuration" "static_site" {
  bucket = aws_s3_bucket.site_bucket.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "error.html"
  }

  routing_rule {
    condition {
      key_prefix_equals = "docs/"
    }
    redirect {
      replace_key_prefix_with = "documents/"
    }
  }
}
```

## 2. Global Uniqueness and DNS Mapping

When you enable a website on a bucket, AWS provides a unique URL in the format:
`<bucket-name>.s3-website-<region>.amazonaws.com`.

- **Technical Constraint:** If you want to use your own domain name (e.g., `www.example.com`), your S3 bucket name **MUST** exactly match the domain name you intend to use.
- **Example:** To host `www.example.com`, your Terraform resource should be `bucket = "www.example.com"`.

## 3. Security and Public Access Requirement

For a website to be visible to the public, the objects in the S3 bucket must be readable by anyone. This requires two specific configuration steps in Terraform.

### 3.1 Disabling Public Access Block
You must explicitly allow public policies on the bucket.

```hcl
resource "aws_s3_bucket_public_access_block" "site_access" {
  bucket = aws_s3_bucket.site_bucket.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}
```

### 3.2 Applying a Public Read Policy
You must then add a Bucket Policy that grants `s3:GetObject` to the `*` principal.

```hcl
resource "aws_s3_bucket_policy" "allow_public_access" {
  bucket     = aws_s3_bucket.site_bucket.id
  depends_on = [aws_s3_bucket_public_access_block.site_access]

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "PublicReadGetObject"
        Effect    = "Allow"
        Principal = "*"
        Action    = "s3:GetObject"
        Resource  = "${aws_s3_bucket.site_bucket.arn}/*"
      }
    ]
  })
}
```

## 4. Why Use CloudFront? (The Production Pattern)

While you can access the S3 website URL directly, the raw S3 website endpoint does not support HTTPS. In a production environment, you should always use **Amazon CloudFront** as the frontend for your S3 website.

**Benefits of the CloudFront + S3 Pattern:**
1.  **SSL/TLS:** Use your own ACM certificates for HTTPS access.
2.  **Performance:** Cache your content in 400+ Edge locations globally.
3.  **Security:** Disable public access to your S3 bucket entirely and use an **Origin Access Control (OAC)** to ensure only CloudFront can read your files.
4.  **Cost:** CloudFront data transfer out to the internet is often cheaper or has a larger free tier than S3 directly.

## 5. Handling SPAs (Single Page Applications)

Modern SPAs (like React) use "Client-Side Routing." If a user refreshes the page on `example.com/login`, S3 will return a 404 because there is no physical file named `login`. 
- **The Solution:** Configure your S3 website's `error_document` to point to `index.html`. This tells S3 "if you can't find a file, just serve the main app and let the JavaScript handle the routing."

## 6. Advanced Redirects and Routing Rules

The `routing_rule` block in the website configuration allows you to perform basic URL manipulation at the edge.
- **Prefix Changes:** Redirect `/archive/*` to `/new-site/*`.
- **Error Redirects:** Redirect specific HTTP error codes (like 404) to a different domain altogether.

## 7. Operational Best Practices: Versioning

Even for a simple website, enable **S3 Versioning**. If a malicious actor or a buggy deployment script deletes your `index.html`, you can restore your site in seconds by simply reverting to the previous version.

## 8. Summary

S3 Static Website Hosting is the simplest way to deploy web content at global scale. By understanding the requirement for public access blocks, the necessity of matching bucket names to DNS records, and the strategic advantages of CloudFront integration, you can build a web architecture that is both extremely inexpensive and virtually indestructible. Terraform's ability to orchestrate the bucket, its security policy, and its website routing rules ensures that your web presence is consistently provisioned and easily manageable.
