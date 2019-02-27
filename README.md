# static-website-configuration

## About
This repository is a serverless configuration that sets up a cloudfront distribution, an S3 bucket, and a route53 record to easily deploy static websites to AWS.

## Pre-requisites
Before using this repository, you must have:
- [serverless installed](https://serverless.com/framework/docs/providers/aws/guide/quick-start/)
- An AWS account with proper credentials to be able to deploy cloudformation templates, and resources.
- You must either have your credentials stored in the default .aws/credentials file, or otherwise passed in to your serverless deploy command so that the serverless framework can create resources on behalf of your account. See [this article for examples of how to set up your credentials to work with serverless](https://serverless.com/framework/docs/providers/aws/guide/credentials/)
- A domain with a hosted zone in Route53
- An SSL certificate for that domain in ACM

## Set up

This serverless configuration creates a cloudfront distribution, which can take up to 40 minutes.
Double check that your variables are correct prior to deployment otherwise you will wait 40 minutes for it to deploy, just to get an error, and it will take another 20ish minutes for it to delete the cloudfront distribution... You've been warned.

```sh
git clone https://github.com/nikita-skobov/static-website-configuration.git
cd static-website-configuration
```

1. Find your certificate ID from ACM. The certificate ID is the UUID that comes after the / in the ACM ARN.
```sh
# An ARN might look something like:
arn:aws:acm:us-east-1:210375816012:certificate/00925a3a-1d59-46b8-ad34-4c36b1256064
# so the certificate ID is:
00925a3a-1d59-46b8-ad34-4c36b1256064
```
2. If you want to use both yourwebsite.com and www.yourwebsite.com make sure to uncomment these lines at the bottom of `serverless.yml`
```yml
resources:
  Resources:
    WebAppRecord: ${file(./serverlessConfig/resources.yml):WebAppRecord}
    WebAppS3Bucket: ${file(./serverlessConfig/resources.yml):WebAppS3Bucket}
    WebAppS3BucketPolicy: ${file(./serverlessConfig/resources.yml):WebAppS3BucketPolicy}
    WebAppDistribution: ${file(./serverlessConfig/resources.yml):WebAppDistribution}

    # # Uncomment these if you want to redirect www.{your alias}.com to {your alias}.com
    # WebRedirectS3Bucket: ${file(./serverlessConfig/resources.yml):WebRedirectS3Bucket}
    # RedirectCloudFrontDistribution: ${file(./serverlessConfig/resources.yml):RedirectCloudFrontDistribution}
    # WWWWebAppRecord: ${file(./serverlessConfig/resources.yml):WWWWebAppRecord}
```
3. Pick a secret user-agent string. This configuration sets up the S3 website to only allow requests that include a specific user-agent string, and it sets that string as a custom cloudfront distribution header. This way, users can only access your website through your domain, and not the S3 website URL. Please note that this method relies on security by obfuscation which is obviously not very secure. AWS has something called Origin Access Identity which lets you restrict access so that only a specific cloudfront distribution can access your S3 buckets [however, this functionality does not work when the S3 bucket is configured as a website](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html) If you are reading this from the future, and there has been a change to how you can restrict access to S3 buckets, please submit an issue.
4. Now, you are ready to deploy:

An example deployment command might look like:
```sh
# the name of your s3 bucket doesnt really matter. It just needs to be unique
sls deploy --bucket thenameofyours3bucket --uasecret someuasecret --certid 00925a3a-1d59-46b8-ad34-4c36b1256064 --alias mywebsite.com --hzname mywebsite.com
```

## What's next?

Once your deployment is done, you will need to put an index.html file into the root of your bucket. The way that S3 with website configuration works is you specify an index file, and an error file. When a user makes a request to your S3 website, S3 looks at the requested path, which maps to a folder within S3. S3 checks that folder for an index.html file, and returns that document. Here are a few examples:

Your S3 structure:
```
- index.html
- script.js
- style.css
- image.png
- about/
  - index.html
  - script.js
- contact/
  - index.html
  - script.js
```

- User makes request to https://yourwebsite.com
  - S3 looks at the root of your bucket, and sees that you have an index.html file, so it returns that
- User makes request to https://yourwebsite.com/about
  - S3 sees you have an about folder, and returns the index.html file within the about folder
- User makes request to https://yourwebsite.com/somethingelse
  - There is no somethingelse folder, so S3 will return an error. In this configuration, I have specified index.html as both the index file, and the error file. So S3 will return the index.html file at the root of the bucket. If you want to have a seperate error file, go to `serverlessConfig/resources.yml` and change these lines as needed:

```yml
WebAppS3Bucket:
  Type: AWS::S3::Bucket
  Properties:
    BucketName: ${self:custom.cloudfront-app.bucketName}
    WebsiteConfiguration:
      IndexDocument: index.html
      ErrorDocument: index.html
```