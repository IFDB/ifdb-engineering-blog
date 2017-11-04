---
title: "Quick static websites on AWS S3 and CloudFront with the AWS CLI"
author: Matt Dorn
date: 2017-10-25
tags:
  - aws
  - cloudfront
  - s3
  - hugo
thumbnailImagePosition: top
thumbnailImage: /img/clouds.jpg
coverImage: /img/clouds.jpg
metaAlignment: center
---

In this post, I'll explain how to easily get an infinitely scalable, superfast static website up and running using [S3](https://aws.amazon.com/s3/) and fronted by the AWS [Cloudfront](https://aws.amazon.com/cloudfront/) CDN, using (almost!) nothing but the [AWS Command Line Interface](https://aws.amazon.com/cli/) (CLI).

<!--more-->

<!-- TOC -->

We'll use the [Hugo](http://gohugo.io/) static site generator for our example, but the AWS portions can be used for any static files you want to deploy.  Also note I'm working on macOS, and in addition to the AWS CLI, I assume you have the `git` and [`jq`](https://stedolan.github.io/jq/) commands installed, though you can work through the info without them.  I also assumed you've set up your configuration files for use with the AWS CLI.

As a preliminary measure, first we'll set up some environment variables for our bucket `ifdb-testing`:

    export TMP_BUCKET_NAME=ifdb-testing
    export TMP_BUCKET_REGION=us-east-1
    export TMP_BUCKET_URL=http://$TMP_BUCKET_NAME.s3-website-$TMP_BUCKET_REGION.amazonaws.com/

## S3 bucket

First things first: we want to create and configure our S3 bucket that will hold our site's static files.

Now we create the bucket:

    aws s3api create-bucket --bucket $TMP_BUCKET_NAME --region $TMP_BUCKET_REGION

To configure the bucket, we're concerned with two things: the bucket policy to manage permissions to the bucket, and the configuration for the bucket's "static website hosting" property.

Since we're deploying a publicly accessible website, the policy is straightforward.  Create a JSON configuration file called `/tmp/policy.json` locally like so:

    echo '{
      "Version": "2012-10-17",
      "Statement": [
        {
          "Sid": "Allow Public Access to All Objects",
          "Effect": "Allow",
          "Principal": "*",
          "Action": "s3:GetObject",
          "Resource": "arn:aws:s3:::'$TMP_BUCKET_NAME'/*"
        }
      ]
    }' > /tmp/policy.json

The website configuration needs to instruct S3 to serve an `index.html` file whenever the user navigates to a URL like `https://example.com/posts/2017/11/hello-world`, since Hugo and most other static site generator tools place an `index.html` file in a folder called `/posts/2017/11/hello-world` that represents your content.  Hence we generate a config as follows:

    echo '{
      "IndexDocument": {
        "Suffix": "index.html"
      },
      "ErrorDocument": {
        "Key": "404.html"
      },
      "RoutingRules": [
        {
          "Redirect": {
            "ReplaceKeyWith": "index.html"
          },
          "Condition": {
            "KeyPrefixEquals": "/"
          }
        }
      ]
    }' > /tmp/website.json

You push these configuration changes with the CLI as follows:

    aws s3api put-bucket-policy --bucket $TMP_BUCKET_NAME --policy file:///tmp/policy.json
    aws s3api put-bucket-website --bucket $TMP_BUCKET_NAME -- --website-configuration file:///tmp/website.json

## Static website

Now we'll generate the files for our static website.  Initial setup for Hugo is based on the Hugo [quick start documentation](https://gohugo.io/getting-started/quick-start/), and I won't go into details here.  Suffice it to say you'll end up with a set of static files in your Hugo project's `public/` which are the files you'll be deploying to S3.

Run through the following commands to get your site up and running:

    hugo new site ifdb_testing
    cd ifdb_testing
    git init
    git submodule add https://github.com/budparr/gohugo-theme-ananke.git themes/ananke
    echo 'baseURL = "/"
    languageCode = "en-us"
    title = "IFDB Testing"
    theme = "ananke"' >> config.toml
    hugo new posts/my-first-post.md
    echo "Hello world" > posts/my-first-post.md

Run the `hugo server` command, and navigate your browser to http://localhost:1313 to make sure you have a website.  Once you've confirmed, run `hugo` to generate the site, send the files to S3, and confirm your site is available at the static website config URL:

    hugo
    cd public
    aws s3 cp . s3://$TMP_BUCKET_NAME --recursive --include "*"
    # on Mac, you can automatically open website in default browser
    open $TMP_BUCKET_URL

## Cloudfront distribution

Now we'll create the Cloudfront distro that will serve your files from S3.  To accomplish this from the command line, you'll need to [create a distribution configuration file](http://docs.aws.amazon.com/cli/latest/reference/cloudfront/create-distribution.html).  Running the command below will do the trick to give you a configuration that will work, resulting in a file at `/tmp/distconfig.json`:

    echo '{
      "CallerReference": "'$TMP_BUCKET_NAME'-'`date +%s`'",
      "Aliases": {
        "Quantity": 0
      },
      "DefaultRootObject": "index.html",
      "Origins": {
        "Quantity": 1,
        "Items": [
          {
            "Id": "S3-'$TMP_BUCKET_NAME'",
            "DomainName": "'$TMP_BUCKET_NAME'.s3.amazonaws.com",
            "S3OriginConfig": {
              "OriginAccessIdentity": ""
            }
          }
        ]
      },
      "DefaultCacheBehavior": {
        "TargetOriginId": "S3-'$TMP_BUCKET_NAME'",
        "ForwardedValues": {
          "QueryString": true,
          "Cookies": {
            "Forward": "none"
          }
        },
        "TrustedSigners": {
          "Enabled": false,
          "Quantity": 0
        },
        "ViewerProtocolPolicy": "allow-all",
        "MinTTL": 3600
      },
      "CacheBehaviors": {
        "Quantity": 0
      },
      "Comment": "",
      "Logging": {
        "Enabled": false,
        "IncludeCookies": false,
        "Bucket": "",
        "Prefix": ""
      },
      "PriceClass": "PriceClass_All",
      "Enabled": true,
      "Aliases": {
        "Items": [
          "testing.ifdb.com"
        ], 
        "Quantity": 1
      }
    }' > /tmp/distconfig.json

Two things to note here. First, Note the `Aliases` key.  This refers to the website's domain name, which you'll set up in your site's DNS service (e.g. AWS Route 53, Namecheap, etc.)  Second, if you've set up an SSL certificate with AWS (beyond the scope of this post), you can instruct your distribution to use it by adding to the `DistributionConfig` key in the configuration an entry like the following:

    "ViewerCertificate": {
        "SSLSupportMethod": "sni-only", 
        "MinimumProtocolVersion": "TLSv1.1_2016", 
        "IAMCertificateId": "[YOUR_CERT_ID]", 
        "Certificate": "[YOUR_CERT_ID]", 
        "CertificateSource": "iam"
    }, 

And, to redirect all requests to HTTPS, in the `DistributionConfig.DefaultCacheBehavior` key add:

    "ViewerProtocolPolicy": "redirect-to-https", 

See the [documentation](http://docs.aws.amazon.com/cli/latest/reference/cloudfront/index.html#cli-aws-cloudfront) for more details.

Now we can create the distribution with the CLI:

    aws cloudfront create-distribution --distribution-config file:///tmp/distconfig.json > /tmp/distconfig_result.json

The resulting JSON document a couple of key pieces of information, only one of which we'll concern ourselves with here, the domain name:

    cat /tmp/distconfig_result.json | jq .Distribution.DomainName

That value will look something like `d329sw4e8bbqcz.cloudfront.net`.  If you want to update anything in the config, you can do so by getting the ID and ETag of the newly created distro:

    cat /tmp/distconfig_result.json | jq .Distribution.Id
    cat /tmp/distconfig_result.json | jq .ETag

Make updates to the configuration in a new JSON file called `/tmp/distconfig_update.json`, and submit the update:

    aws cloudfront update-distribution --id [YOUR DISTRO ID] --distribution-config file:///tmp/distconfig_update.json --if-match [YOUR DISTRO ETAG]

There is in fact one update we need to make.  If you take a look at the `Origins.Items[0].DomainName` value in our distro configuration, you'll notice that in our example it was set to `ifdb-testing.s3.amazonaws.com`.  This is our bucket's "domain" but it is *not* the domain for the bucket's website config, which is the one we need because of the required `index.html` rules we mentioned above.  Visiting any page except the root URL will result in an `Access Denied` error.  Unfortunately if you try to change the origin value appropriately  in the CLI, AWS chokes:

    An error occurred (InvalidArgument) when calling the UpdateDistribution operation: The parameter Origin DomainName does not refer to a valid S3 bucket.

You didn't really think we were going to be able to do all of this without going into the AWS web console, did you?

Find your new distribution in the Cloudwatch section of the console, and click on the "Origins" tab.  Edit the one existing origin record, and set the `Origin Domain Name` value to the S3 website config domain name, which in our case is `ifdb-testing.s3-website-us-east-1.amazonaws.com`.  Save it, and wait for the Cloudfront distro to update.  After a minute or two, your site should be accessible at the `Distribution.DomainName` value we got from the configuration result.

To make it accessible at your custom domain name, just add a CNAME record through your DNS provider, and make the `Distribution.DomainName` value the target for your hostname.
