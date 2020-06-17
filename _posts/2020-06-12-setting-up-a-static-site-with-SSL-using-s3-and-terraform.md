---
title: "Setting up your HTTPS website with S3, Cloudfront, Certificates Manager and GoDaddy for the price of a domain name"
layout: post
---


Amazon's Simple Storage Service (s3) is probably one of the best, most stable services they provide (not throwing shade on the rest, s3 is just that good) and I would love to be able to use its 11 9s of availability to host my site. I would also like to use HTTPS because I'm a good boy scout and managing everything through Terraform because the command line is my jam. Are you on the same boat? Boy do I have a post for you then.
First thing is making sure we have all the tools we need. Make sure you've downloaded Terraform, installed the AWS CLI and set it up correctly. You will also need an AWS account (duh) and a GoDaddy one. If you just want the end result, all the source code is in my Github.
Now that we have everything we need, let's get our hands dirty.
I'll use my personal site as a demo, don't judge it too hard please.
![Cover the pain with memes. Never fails.](assets/images/2020-06-12/gdesign.png)

Cover the pain with memes. Never fails.Our project will have 2 folders in the root: site/ and terraform/. We'll structure the folder for the site in the following way inside the site/ folder: png/, pdf/, css/, js/ ico/ and in the root, there's only the index.html. There's a reason for this, but we'll get into it later. The contents of the site don't really matter for this tutorial, but here's a capture of how my structure looks after everything we've mentioned:
Now that we have our site setup, let's start creating the terraform entry for it. The first thing we'll want to tackle is the S3 bucket, since that's where everything else links back to. The configuration to host a static website in it is pretty straightforward:

{% highlight terraform %}
resource "aws_s3_bucket" "bucket" {
  bucket = "abaltra.me"
  acl    = "public-read"
  policy = <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::abaltra.me/*"
            ]
        }
    ]
}
EOF

  website {
    index_document = "index.html"
    error_document = "error.html"
  }
}

resource "aws_s3_bucket_object" "indexes" {
  for_each = fileset("../site/", "*")

  content_type = "text.html"
  bucket = aws_s3_bucket.bucket.id
  key    = each.value
  source = "../site/${each.value}"
  # etag makes the file update when it changes; see https://stackoverflow.com/questions/56107258/terraform-upload-file-to-s3-on-every-apply
  etag   = filemd5("../site/${each.value}")
}

resource "aws_s3_bucket_object" "js" {
  for_each = fileset("../site/js/", "*")

  content_type = "text/javascript"
  bucket = aws_s3_bucket.bucket.id
  key    = "js/${each.value}"
  source = "../site/js/${each.value}"
  # etag makes the file update when it changes; see https://stackoverflow.com/questions/56107258/terraform-upload-file-to-s3-on-every-apply
  etag   = filemd5("../site/js/${each.value}")
}

resource "aws_s3_bucket_object" "css" {
  for_each = fileset("../site/css/", "*")

  content_type = "text/css"
  bucket = aws_s3_bucket.bucket.id
  key    = "css/${each.value}"
  source = "../site/css/${each.value}"
  # etag makes the file update when it changes; see https://stackoverflow.com/questions/56107258/terraform-upload-file-to-s3-on-every-apply
  etag   = filemd5("../site/css/${each.value}")
}

resource "aws_s3_bucket_object" "img" {
  for_each = fileset("../site/img/", "*")

  content_type = "img/png"
  bucket = aws_s3_bucket.bucket.id
  key    = "img/${each.value}"
  source = "../site/img/${each.value}"
  # etag makes the file update when it changes; see https://stackoverflow.com/questions/56107258/terraform-upload-file-to-s3-on-every-apply
  etag   = filemd5("../site/img/${each.value}")
}

resource "aws_s3_bucket_object" "ico" {
  for_each = fileset("../site/ico/", "*")

  content_type = "image/ico"
  bucket = aws_s3_bucket.bucket.id
  key    = "ico/${each.value}"
  source = "../site/ico/${each.value}"
  # etag makes the file update when it changes; see https://stackoverflow.com/questions/56107258/terraform-upload-file-to-s3-on-every-apply
  etag   = filemd5("../site/ico/${each.value}")
}

resource "aws_s3_bucket_object" "static" {
  for_each = fileset("../site/pdf/", "*")

  content_type = "application/pdf"
  bucket = aws_s3_bucket.bucket.id
  key    = "static/${each.value}"
  source = "../site/pdf/${each.value}"
  # etag makes the file update when it changes; see https://stackoverflow.com/questions/56107258/terraform-upload-file-to-s3-on-every-apply
  etag   = filemd5("../site/pdf/${each.value}")
}
{% endhighlight %}


If you look at lines 29 forward, that's all to be able to upload files using terraform. Starting with version 0.12 you can use the for_each operator, which will let us handle file uploads easily along with our infrastructure code. And in here comes one of the caveats. Remember the file structure that basically had a folder per file type? Here's the reason: Terraform can't infer the content_type of files when it's uploading, so it assumes everything's an octet-stream. This can be a problem because browsers won't display your data, but will rather try to download it which is rarely what we want our website to do. Treating each upload like this, while more verbose, lets us handle everything with the same tool and in the same process.
Now that the website is up, you can actually visit it immediately. Log into the AWS console and go to S3. You should see a bucket with the name mywebsite.com.
It's abaltra.me because I used this process for my personal website, but the process is the same for other domains.Click on the Endpoint link, and you should see a hosted version of your site. Step 1 complete!
Getting a certificate and a domain
To provide HTTPS support, you'll need a certificate. Custom certificates can get pretty expensive, but what if I told you we can get away with doing it for free? Using Amazon's Certificate Manager, we can use one of Amazon's certificates and get all the benefits of SSL while staying as cheap as possible. Also, we get to do it inside of Terraform, pretty cool right?
ADD THE GODADDY SECTION
The code for our certificate is pretty simple:

{% highlight terraform %}
resource "aws_acm_certificate" "cert" {
  domain_name       = "www.abaltra.me"

  subject_alternative_names = [
    "www.abaltra.me",
    "abaltra.me"
  ]

  validation_method = "DNS"

  tags = {
    Environment = "test"
  }

  lifecycle {
    create_before_destroy = true
  }
}
{% endhighlight %}



And with the certificate created and validated, we can then setup our Cloudfront distribution:

{% highlight terraform %}
resource "aws_cloudfront_distribution" "s3_distribution" {
    origin {
        domain_name = aws_s3_bucket.bucket.website_endpoint
        origin_id = "S3-abaltra.me"
    
        // The origin must be http even if it's on S3 for redirects to work properly
        // so the website_endpoint is used and http-only as S3 doesn't support https for this
        custom_origin_config {
        http_port = 80
        https_port = 443
        origin_protocol_policy = "http-only"
        origin_ssl_protocols = ["TLSv1.2"]
        }
    }

    default_root_object = "index.html"
    
    aliases = ["www.abaltra.me", "abaltra.me"]
    
    enabled = true
    is_ipv6_enabled = true
    default_root_object = "index.html"

    
    default_cache_behavior {
        allowed_methods = ["GET", "HEAD"]
        cached_methods = ["GET", "HEAD"]
        target_origin_id = "S3-abaltra.me"
    
        forwarded_values {
        cookies {
            forward = "none"
        }
        query_string = false
        }
    
        viewer_protocol_policy = "redirect-to-https"
        min_ttl = 0
        max_ttl = 86400
        default_ttl = 3600
        compress = true
    }
    
    viewer_certificate {
        acm_certificate_arn = aws_acm_certificate.cert.arn
        ssl_support_method = "sni-only"
        minimum_protocol_version = "TLSv1.1_2016"
    }
    
    restrictions {
        geo_restriction {
        restriction_type = "none"
        }
    }    
}
{% endhighlight %}

Once all of the terraform has run, just visit your website in the domain name you bought, or if you want to go straight to the Cloudfront distribution, you can go to AWS and get the url it's using in the Domain Name column. Remember, you can view your website straight from the S3 link, through the Cloudfront Domain Name or through your domain name. Knowing this might can help you eliminate layers of possible problems and narrow down the level you need to work on to solve any issues
Finally, how much will all of this cost me? Well, for me it looks like this:
USD$3.5 domain name (USD$19 a year after the first year)
USD$0 infrastructure.

Wait, what? Yeah, big round $0 (at least as long as the the website doesn't get a ton of traffic, but at that point, I think we could give AWS a little money, right?)
In case you don't believe me, here's the proof:
Big fat 0 right thereAWS' infrastructure is fantastic, and with terraform and all the flexibility it offers there really is no reason to not use it (ok, the reason would be "there's no Terraform provider for X, but there is one for most of AWS) and keep all your infrastructure nice and version controlled right alongside your source code.
Go build something fun!