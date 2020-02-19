---
author:
  name: "Josiah Halme"
date: 2019-11-24
linktitle: Adding CloudFront and ACM SSL
title: Adding CloudFront and ACM SSL
type:
- post
- posts
weight: 10
series:
- Tech
aliases:
- /blog/blog-adding-cloudfront-and-acm-ssl/
---

# Overview

To use SSL on my blog, I need to create an
[Amazon CloudFront](https://aws.amazon.com/cloudfront/) web distribution along
with adding a SSL certificate with
[AWS Certificate Manager (ACM)](https://aws.amazon.com/certificate-manager/).

---

## ACM (AWS Certificate Manager)

Create an Amazon ACM certificate and validate with the appropriate DNS records
that are output. As CloudFront requires the certificate to be in us-east-1, we
need to ensure the certificate is created in that region.

```hcl
resource "aws_acm_certificate" "cert" {
  provider                  = aws.east
  domain_name               = var.domain
  validation_method         = "DNS"
  subject_alternative_names = ["www.${var.domain}"]

  tags = {
    Name        = var.domain
    Environment = "production"
  }

  lifecycle {
    create_before_destroy = true
  }
}

output "domain_name" {
  value = aws_acm_certificate.cert.domain_name
}

output "domain_validation_options" {
  value = aws_acm_certificate.cert.domain_validation_options
}
```

Query the status through the console or CLI

```shell
➜ aws acm describe-certificate
--certificate-arn arn:aws:acm:ap-southeast-2:1234567890:certificate/abcd1234-1234-1234-1234-123456abcdef
--query 'Certificate.DomainValidationOptions[*].{name:DomainName,status:ValidationStatus}'
--output table
-----------------------------------------
|          DescribeCertificate          |
+----------------+----------------------+
|      name      |       status         |
+----------------+----------------------+
|  halme.org     |  SUCCESS             |
|  www.halme.org |  PENDING_VALIDATION  |
+----------------+----------------------+
```

---

## S3

We previously had our S3 bucket configured to host a static website directly,
so we'll need to update our configuration to make it private, allowing
CloudFront access only.

```hcl
resource "aws_s3_bucket" "web" {
  bucket = var.domain
}

data "aws_iam_policy_document" "s3_policy" {
  statement {
    actions   = ["s3:GetObject"]
    resources = ["${aws_s3_bucket.web.arn}/*"]

    principals {
      type        = "AWS"
      identifiers = [aws_cloudfront_origin_access_identity.origin_access_identity.iam_arn]
    }
  }

  statement {
    actions   = ["s3:ListBucket"]
    resources = ["${aws_s3_bucket.web.arn}"]

    principals {
      type        = "AWS"
      identifiers = [aws_cloudfront_origin_access_identity.origin_access_identity.iam_arn]
    }
  }
}

resource "aws_s3_bucket_policy" "web" {
  bucket = aws_s3_bucket.web.id
  policy = data.aws_iam_policy_document.s3_policy.json
}
```

---

## CloudFront

We need to create an [Origin Access Identity](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)

```hcl
resource "aws_cloudfront_origin_access_identity" "origin_access_identity" {
  comment = "${var.domain} cloudfront"
}
```

We will then create a CloudFront distribution

```hcl
locals {
  s3_origin_id = "myS3Origin"
}

resource "aws_cloudfront_distribution" "s3_distribution" {
  origin {
    domain_name = aws_s3_bucket.web.bucket_regional_domain_name
    origin_id   = local.s3_origin_id

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.origin_access_identity.cloudfront_access_identity_path
    }
  }

  enabled             = true
  is_ipv6_enabled     = true
  comment             = "${var.domain} distribution"
  default_root_object = "index.html"

  aliases = ["${var.domain}"]

  default_cache_behavior {
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = local.s3_origin_id

    forwarded_values {
      query_string = false

      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400
  }

  price_class = "PriceClass_200"

  restrictions {
    geo_restriction {
      restriction_type = "whitelist"
      locations        = ["AU", "NZ", "US"]
    }
  }

  tags = {
    Environment = "production"
  }

  viewer_certificate {
    acm_certificate_arn = aws_acm_certificate.cert.arn
    ssl_support_method  = "sni-only"
  }
}

output "cloudfront_domain_name" {
  value = aws_cloudfront_distribution.s3_distribution.domain_name
}
```

Run the terraform plan and it should output you a CloudFront domain (eventually)

```shell
aws_cloudfront_distribution.s3_distribution: Still modifying... [id=ABCDEFGHIJKLM, 24m0s elapsed]
aws_cloudfront_distribution.s3_distribution: Still modifying... [id=ABCDEFGHIJKLM, 24m10s elapsed]
aws_cloudfront_distribution.s3_distribution: Still modifying... [id=ABCDEFGHIJKLM, 24m20s elapsed]
aws_cloudfront_distribution.s3_distribution: Still modifying... [id=ABCDEFGHIJKLM, 24m30s elapsed]
aws_cloudfront_distribution.s3_distribution: Modifications complete after 24m38s [id=ABCDEFGHIJKLM]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

cloudfront_domain_name = example.cloudfront.net
```

---

## Conclusion

Browse to the cloudfront URL and you should have your static website.
It's then a matter of updating your DNS to point to URL.

Next article will look at streamlining and documenting the pipeline for pushing
updates/articles.



