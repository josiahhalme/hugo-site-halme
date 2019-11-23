---
author:
  name: "Josiah Halme"
date: 2019-11-23
linktitle: A Serverless Blog with Hugo
title: A Serverless Blog with Hugo
type:
- post
- posts
weight: 10
series:
- Tech
aliases:
- /blog/a-serverless-blog-with-hugo/
---

## AWS + Hugo + Terraform

On a whim I decided to add a blog to my static website, which consisted simply of links to my socials (linkedin/github/email).
I thought this would be a good opportunity to dip my toes into serverless, which through various articles led me to **Hugo**. It also gives me a good excuse to practice my writing in **Markdown**.


### Hugo

Following the **Hugo** quick start guide located [here](https://gohugo.io/getting-started/quick-start/), I followed these steps to get a basic site up and running.

1. Install Hugo with homebrew

  ```shell
  brew install hugo
  ```

2. Create a new site with Hugo

  ```shell
  hugo new site halme
  ```

3. Create a git repo for our site

  ```shell
  git init
  ```

4. Find a nice theme at [themes.gohugo.io](https://themes.gohugo.io/) and add it as a submodule

  ```shell
  git submodule add https://github.com/rhazdon/hugo-theme-hello-friend-ng.git themes/hello-friend-ng
  ```

5. Update the Hugo `config.toml` file with the appropriate theme config

  ```shell
  baseURL = "/"
  title   = "Josiah Halme"
  theme = "hello-friend-ng"
  ```

6. Create a new post

 ```shell
 hugo new posts/first-post.md
 ```

7. Start a local Hugo server to test, browse to `http://localhost:1313/`

  ```shell
  ➜  hugo server -t hello-friend-ng
                     | EN
  +------------------+----+
    Pages            | 12
    Paginator pages  |  0
    Non-page files   |  0
    Static files     | 12
    Processed images |  0
    Aliases          |  4
    Sitemaps         |  1
    Cleaned          |  0

  Total in 35 ms
  Watching for changes in   /Users/josiah/hugo/halme/{archetypes,content,data,layouts,static,themes}
  Watching for config changes in /Users/josiah/hugo/halme/config.toml
  Environment: "development"
  Serving pages from memory
  Running in Fast Render Mode. For full rebuilds on change: hugo server --  disableFastRender
  Web Server is available at //localhost:1313/ (bind address 127.0.0.1)
  Press Ctrl+C to stop
  ```

8. Create the static website by running `hugo` which will then deploy to the `public` directory

  ```shell
  ➜ hugo

                     | EN
  +------------------+----+
    Pages            | 12
    Paginator pages  |  0
    Non-page files   |  0
    Static files     | 12
    Processed images |  0
    Aliases          |  4
    Sitemaps         |  1
    Cleaned          |  0

  Total in 38 ms
  ```

9. Add a Hugo deployment target in `config.toml` (S3 bucket in our case)
  ```shell
  [deployment]
  order = [".jpg$", ".gif$"]

  [[deployment.targets]]
  # An arbitrary name for this target.
  name = "mydeployment"

  # S3; see https://gocloud.dev/howto/blob/#s3
  # For S3-compatible endpoints, see https://gocloud.dev/howto/blob/#s3-compatible
  URL = "s3://mybucketname?region=ap-southeast-2"

  [[deployment.matchers]]
  #  Cache static assets for 20 years.
  pattern = "^.+\\.(js|css|svg|ttf)$"
  cacheControl = "max-age=630720000, no-transform, public"
  gzip = true

  [[deployment.matchers]]
  pattern = "^.+\\.(png|jpg)$"
  cacheControl = "max-age=630720000, no-transform, public"
  gzip = false

  [[deployment.matchers]]
  pattern = "^.+\\.(html|xml|json)$"
  gzip = true
  ```

10. Deploy to our S3 Bucket, in this case we've added our AWS IAM credentials (access_id and access_key) to `~/.aws/credentials`

  ```shell
  ➜ hugo deploy
  Deploying to target "mydeployment" (s3://halme.org?region=ap-southeast-2)
  Identified 53 file(s) to upload, totaling 1.6 MB, and 0 file(s) to delete.
  Success!
  ```
11. We have now deployed to our S3 bucket. By default this means nothing, and we have nothing to point our DNS to yet.

### Terraform
For the purpose of this blog, I'll just be starting off with static website hosting in S3 as outlined [here](https://docs.aws.amazon.com/AmazonS3/latest/dev/WebsiteHosting.html).
I'll be managing it with Terraform as I intend on using CloudFront at a later point.

1. Bootstrap a simple Terraform repo
```shell
terraform-web-halme
├── aws.tf
├── policy.json
├── s3.tf
├── state.tf
├── terraform.tfvars
└── variables.tf
```

  aws.tf

  ```hcl
  # Define AWS terraform provider, along with required region,
  # account_id and provider version

  provider "aws" {
    region              = var.region
    allowed_account_ids = [var.account_id]
    version             = "~> 2.0"
  }
  ```

  policy.json
  ```hcl
  # Define S3 policy to allow static website hosting

  {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Sid": "PublicReadGetObject",
              "Effect": "Allow",
              "Principal": "*",
              "Action": "s3:GetObject",
              "Resource": "arn:aws:s3:::halme.org/*"
          }
      ]
  }
  ```

  s3.tf
  ```hcl
  # S3 Bucket configuration for static hosting. Use ACL defined
  # earlier, and add basic website settings.

  resource "aws_s3_bucket" "web" {
    bucket = var.domain
    acl    = "public-read"
    policy = "${file("policy.json")}"

    website {
      index_document = "index.html"
      error_document = "error.html"

      routing_rules = <<EOF
  [{
      "Condition": {
          "KeyPrefixEquals": "docs/"
      },
      "Redirect": {
          "ReplaceKeyPrefixWith": "documents/"
      }
  }]
  EOF
    }
  }
  ```

  state.tf
  ```hcl
  # Create an S3 bucket for the terraform state
  resource "aws_s3_bucket" "terraform_state" {
    bucket        = "web-${var.name}-terraform"
    force_destroy = true

    versioning {
      enabled = true
    }
  }

  # Create a dynamodb table for the terraform state lock
  resource "aws_dynamodb_table" "terraform_statelock" {
    name           = "web-${var.name}-terraform-lock"
    read_capacity  = 1
    write_capacity = 1
    hash_key       = "LockID"

    attribute {
      name = "LockID"
      type = "S"
    }
  }

  # These settings must be hard coded, use S3 backend for state.
  terraform {
    required_version = ">= 0.12.0"

    backend "s3" {
      bucket         = "web-halme-terraform"
      key            = "terraform"
      region         = "ap-southeast-2"
      encrypt        = "true"
      dynamodb_table = "web-halme-terraform-lock"
    }
  }
  ```

  terraform.tfvars
  ```hcl
  account_id = "1234567890"
  domain     = "example.tld"
  name       = "example"
  ```

  variables.tf
  ```hcl
  variable "account_id" {
  }

  variable "region" {
    default = "ap-southeast-2"
  }

  variable "domain" {
  }

  variable "name" {
  }
  ```

2. Run a terraform `plan` and `apply` to create the AWS infrastructure.
3. More to come....
