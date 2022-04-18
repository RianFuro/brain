Static file hosting is handily accomplished by putting the files into an S3 bucket, then pointing a CloudFront endpoint to it. CloudFront will then take care of all the nasty details of hosting a webserver for you, like providing a CDN and handling throttling.

This is a minimal CloudFormation template to get startet:
```yaml
AppBucket:  
  Type: AWS::S3::Bucket  
  DeletionPolicy: Retain  
  Properties:  
    BucketEncryption:  
      ServerSideEncryptionConfiguration:  
        - ServerSideEncryptionByDefault:  
            SSEAlgorithm: AES256  
  
AppBucketPolicy:  
  Type: AWS::S3::BucketPolicy  
  Properties:  
    Bucket: !Ref AppBucket  
    PolicyDocument:  
      Version: '2012-10-17'  
      Statement:  
        - Action:  
            - s3:GetObject  
          Effect: Allow  
          Resource: !Sub '${AppBucket.Arn}/*'  
          Principal:  
            CanonicalUser: !GetAtt CloudFrontOriginAccessIdentity.S3CanonicalUserId  
  
AppDistribution:  
  Type: AWS::CloudFront::Distribution  
  Properties:  
    DistributionConfig:  
      DefaultCacheBehavior:  
        Compress: true  
        DefaultTTL: 86400  
        ForwardedValues:  
          QueryString: true  
        MaxTTL: 31536000  
        TargetOriginId: !Sub 'S3-${AWS::StackName}-root'  
        ViewerProtocolPolicy: 'redirect-to-https'  
        ResponseHeadersPolicyId: !Ref ResponseHeadersPolicy  
      CustomErrorResponses:  
        - ErrorCachingMinTTL: 60  
          ErrorCode: 404  
          ResponseCode: 200  
          ResponsePagePath: '/index.html'  
        - ErrorCachingMinTTL: 60  
          ErrorCode: 403  
          ResponseCode: 200  
          ResponsePagePath: '/index.html'  
      Enabled: true  
      HttpVersion: 'http2'  
      DefaultRootObject: 'index.html'  
      IPV6Enabled: true  
      Origins:  
        - DomainName: !GetAtt AppBucket.DomainName  
          Id: !Sub 'S3-${AWS::StackName}-root'  
          S3OriginConfig:  
            OriginAccessIdentity:  
              !Sub 'origin-access-identity/cloudfront/${CloudFrontOriginAccessIdentity}'  
      PriceClass: 'PriceClass_100'  
      ViewerCertificate:  
        CloudFrontDefaultCertificate: 'true'  
  
CloudFrontOriginAccessIdentity:  
  Type: AWS::CloudFront::CloudFrontOriginAccessIdentity  
  Properties:  
    CloudFrontOriginAccessIdentityConfig:  
      Comment: !Sub 'CloudFront OAI for TODO'  
  
ResponseHeadersPolicy:  
  Type: AWS::CloudFront::ResponseHeadersPolicy  
  Properties:  
    ResponseHeadersPolicyConfig:  
      Name: !Sub "${AWS::StackName}-static-site-security-headers"  
      SecurityHeadersConfig:  
        StrictTransportSecurity:  
          AccessControlMaxAgeSec: 63072000  
          IncludeSubdomains: true  
          Override: true  
          Preload: true  
        ContentSecurityPolicy:  
          ContentSecurityPolicy: "default-src 'none'; img-src 'self'; script-src 'self'; style-src 'self'; font-src 'self'; object-src 'none'"  
          Override: true  
        ContentTypeOptions:  
          Override: true  
        FrameOptions:  
          FrameOption: DENY  
          Override: true  
        ReferrerPolicy:  
          ReferrerPolicy: "same-origin"  
          Override: true  
        XSSProtection:  
          ModeBlock: true  
          Override: true  
          Protection: true
```

Let's break it down by resources:

## AppBucket
This is the bucket where the files are stored in. The encryption here is optional, I left it in if you need it.

## AppBucketPolicy
This policy allows CloudFront exclusive `GetObject`-access to the bucket. Note that since we do not explicitly deny other access patterns, an IAM policy can still allow a user access.

## AppDistribution
This is the CloudFront distribution. The most important parts here are the `Origins` and `CustomErrorResponses` keys.
`Origins` defines the upstream S3 bucket to serve the content from. Additionally it attaches a OriginalAccessIdentity to the connection, ensuring that the bucket's contents can be accessed from CloudFront, but not directly from the bucket.
`CustomErrorResponses` define alternative actions when a URL does not resolve to an object in the bucket. This is important for SPAs, as arbitrary paths need to resolve back to `index.html`

Additionally, this configuration defines the caching behaviour of the CDN as well as setting the default CloudFront viewer certificate for easy https access. The latter needs to be replaced if access by a custom domain is needed.

## ResponseHeadersPolicy
The last piece is the response headers policy. It defines default headers on the response served by CloudFront, so stuff like content security policy headers goes here.