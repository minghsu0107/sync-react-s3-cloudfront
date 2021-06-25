# React App Deployment with Amazon S3 and CloudFront
This repository demonstrates how to build a CD pipeline that deploys a React application to Amazon S3 and syncs it with Amazon CloudFront, a content delivery network (CDN) managed by AWS.
## Why?
It is quite convenient to configure a S3 bucket for static website hosting. However, we can only access it via HTTP protocol. A great way to host our contents is to place it to Amazon CloudFront. This way, not only can we benefit from HTTPS protection but also the out-of-the-box caching mechanism CloudFront provides for us.
## Prerequisite
1. A S3 bucket with [static website hosting](https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html) enabled. Be sure that the index document is set to `index.html`.
2. A CloudFront distribution with its origin set to the S3 website endpoint. The endpoint looks like `BUCKET.s3-website.us-east-2.amazonaws.com` where `BUCKET` is a placeholder of your S3 bucket name.
3. The IAM user must be attached with proper S3 access policies and the following CloudFront policies:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": "s3:ListAllMyBuckets",
            "Effect": "Allow",
            "Resource": "arn:aws:s3:::BUCKET"
        },
        {
            "Action": [
                "cloudfront:CreateInvalidation",
                "cloudfront:GetDistribution",
                "cloudfront:GetStreamingDistribution",
                "cloudfront:GetDistributionConfig",
                "cloudfront:GetInvalidation",
                "cloudfront:ListInvalidations",
                "cloudfront:ListStreamingDistributions",
                "cloudfront:ListDistributions"
            ],
            "Effect": "Allow",
            "Resource": "*"
        }
    ]
}
```
Where BUCKET is a placeholder of your bucket name.
## CD Pipeline
We are using [Drone CI](https://www.drone.io) as our automation tool. There are two main steps in the pipeline, which are building React applcation and deploying to AWS.

1. **Building React application** - In this step, we install all dependencies and create a minified bundle to `build/` folder.

```yaml
- name: build-react-app
  image: node:15.14
  commands:
  - npm install && npm run build
```

2. **Deploy to AWS** - In this step, we upload the minified bundle to S3 and invalidate the CloudFront cache in order to see the updated website instantly.

```yaml
- name: sync-react-app
  image: amazon/aws-cli:2.1.29
  environment:
    AWS_ACCESS_KEY_ID: 
      from_secret: aws_access_key_id
    AWS_SECRET_ACCESS_KEY:
      from_secret: aws_secret_access_key
    CLOUDFRONT_DISTRIBUTION_ID:
      from_secret: cloudfront_distribution_id
    BUCKET:
      from_secret: bucket
  commands:
  - aws s3 sync ./build/ s3://$BUCKET/ --delete --acl public-read
  - aws cloudfront create-invalidation --distribution-id $CLOUDFRONT_DISTRIBUTION_ID --paths "/*" 
```
