---
author:
  name: "Josiah Halme"
date: 2020-02-17
linktitle: CloudFront, S3 and OAI
title: CloudFront, S3 and OAI
type:
- post
- posts
weight: 10
series:
- Tech
aliases:
- /blog/cloudfront-s3-and-oai/
---

## CloudFront, S3 Static Site hosting and Origin Access Identity
After looking into why certain URL's weren't working, I discovered static site hosting doesn't play nicely with CloudFront and Origin Access Identities.

As a result I removed the OAI and made the bucket public. This simplifies it somewhat, and the content is all publically available regardless.

***

### S3
The update S3 resources,
```hcl
resource "aws_s3_bucket" "web" {
  bucket = var.domain
  acl    = "public-read"

  website {
    index_document = "index.html"
    error_document = "error.html"
  }

}

data "aws_iam_policy_document" "s3_policy" {
  statement {
    sid       = "PublicReadGetObject"
    actions   = ["s3:GetObject"]
    resources = ["${aws_s3_bucket.web.arn}/*"]
    principals {
      type        = "*"
      identifiers = ["*"]
    }
  }
}```


***

### CloudFront
We update the CloudFront distribution to get rid of the Origin Access Identity code.

```hcl
resource "aws_cloudfront_distribution" "s3_distribution" {
  origin {
    domain_name = "${var.domain}.s3-website-${var.region}.amazonaws.com"
    origin_id   = "S3Origin-${var.domain}"
    custom_origin_config {
      http_port                = 80
      https_port               = 443
      origin_keepalive_timeout = 5
      origin_protocol_policy   = "http-only"
      origin_read_timeout      = 30
      origin_ssl_protocols = [
        "TLSv1",
        "TLSv1.1",
        "TLSv1.2",
      ]
    }
  }

  enabled             = true
  is_ipv6_enabled     = true
  comment             = "${var.domain} distribution"
  default_root_object = "index.html"

  aliases = ["${var.domain}", "www.${var.domain}"]

  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    compress         = true
    target_origin_id = "S3Origin-${var.domain}"

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

  price_class = "PriceClass_All"

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  tags = {
    Environment = "production"
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.cert.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.1_2016"
  }
}

output "cloudfront_domain_name" {
  value = aws_cloudfront_distribution.s3_distribution.domain_name
}```

***

### Conclusion
On initial investigation, it looks like you can do pretty rewrites with Lambda, however this seemed like overkill.
At a glance currently, perhaps using the S3 API endpoint for CloudFront may be the way to keep S3 buckets private and use Origin Access Identities.



