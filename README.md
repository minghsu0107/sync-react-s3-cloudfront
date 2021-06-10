# React App Deployment with Amazon S3 and CloudFront
This repository demonstrates how to build a CD pipeline that deploys a React application to Amazon S3 and syncing it with Amazon CloudFront, a content delivery network (CDN) managed by AWS.
## Why?
It is quite convenient to serve static website by configuring a bucket for website hosting. However, we can only access it via HTTP protocol. A great way to serve our contents is to place it in Amazon CloudFront. This way, not only can we benefit from HTTPS protection but also the out-of-the-box caching mechanism CloudFront provides for us.
## Prerequisite
1. A S3 bucket with (static website hosting)[https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html] enabled. Be sure that the index document is set to `index.html`.
2. A CloudFront distribution with its origin set to the S3 website endpoint. The endpoint looks like `BUCKET.s3-website.us-east-2.amazonaws.com` where `BUCKET` is a placeholder of your S3 bucket name.
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
    AWS_DEFAULT_REGION:
      from_secret: aws_default_region
    CLOUDFRONT_DISTRIBUTION_ID:
      from_secret: cloudfront_distribution_id
    BUCKET:
      from_secret: bucket
  commands:
  - aws s3 sync ./build/ s3://$BUCKET/ --delete --acl public-read
  - aws cloudfront create-invalidation --distribution-id $CLOUDFRONT_DISTRIBUTION_ID --paths "/*" 
```