# Deploy and distribute static website on AWS with S3, CloudFront


```python
# Import libraries
import configparser
import boto3
import json
```


```python
# AWS Access key and secret key should not be exposed 
config = configparser.ConfigParser()
config.read('cloudfront.cfg')
AWS_ACCESS_KEY = config['AWS']['ACCESS_KEY']
SECRET_ACCESS_KEY = config['AWS']['SECRET_ACCESS_KEY']
```


```python
# instantiate s3 client
s3 = boto3.client(
    's3',
    aws_access_key_id=AWS_ACCESS_KEY,
    aws_secret_access_key=SECRET_ACCESS_KEY,
    region_name='us-west-2'
)
```


```python
s3
```




    <botocore.client.S3 at 0x10ed8bdf0>



## 1. Create S3 Bucket to host static website


```python
BUCKET_NAME = config['S3']['BUCKET_NAME']
```


```python
# Create S3 Bucket to host the website  
create_bucket_res = s3.create_bucket(Bucket=BUCKET_NAME,  
                 CreateBucketConfiguration={
        'LocationConstraint': 'us-west-2'
    })
create_bucket_res
```




    {'ResponseMetadata': {'RequestId': 'XDVDDKCMNMFDNMQ3',
      'HostId': 'G27xvJL+KFltEtozzLK2NCRwZcKFOikPuzi1peRRDYs3cDkrkw0giUwYglh7ff1C8PRyYjt9SHs=',
      'HTTPStatusCode': 200,
      'HTTPHeaders': {'x-amz-id-2': 'G27xvJL+KFltEtozzLK2NCRwZcKFOikPuzi1peRRDYs3cDkrkw0giUwYglh7ff1C8PRyYjt9SHs=',
       'x-amz-request-id': 'XDVDDKCMNMFDNMQ3',
       'date': 'Fri, 19 Mar 2021 19:15:09 GMT',
       'location': 'http://udac-static-webapp-demo.s3.amazonaws.com/',
       'content-length': '0',
       'server': 'AmazonS3'},
      'RetryAttempts': 0},
     'Location': 'http://udac-static-webapp-demo.s3.amazonaws.com/'}

### Check the bucket was scuccessfully created
![S3_bucket_image](https://i.ibb.co/dJ5SHFq/Screen-Shot-2021-03-18-at-9-26-33-PM.png)


## 2. Go to S3 console and upload index.html and other required components 
*Programiccally you can only upload files one-by-one. Upload via console is more efficient and secure if there are many files

![S3_update](https://i.ibb.co/dJ5SHFq/Screen-Shot-2021-03-18-at-9-26-33-PM.png)

## 3. Update the bucket policy so that it can host static website


```python
# this is json bucket policy 
bucket_policy_json = {
    "Version":"2012-10-17",
    "Statement":[
     {
       "Sid":"AddPerm",
       "Effect":"Allow",
       "Principal": "*",
       "Action":["s3:GetObject"],
       "Resource":["arn:aws:s3:::" + BUCKET_NAME + "/*"]
     }
    ]
}

bucket_policy = json.dumps(bucket_policy_json)

bucket_policy_res = s3.put_bucket_policy(Bucket=BUCKET_NAME, Policy=bucket_policy)
bucket_policy_res
```




    {'ResponseMetadata': {'RequestId': 'NGZVZK52DZW9ZQ9E',
      'HostId': 'edl7x7tUVMhSGUxOBGqX0xKCBVc393fPZdfsvYWBhtrzLnGs515p8vJog40Uio3TZKCWB+HbwFk=',
      'HTTPStatusCode': 204,
      'HTTPHeaders': {'x-amz-id-2': 'edl7x7tUVMhSGUxOBGqX0xKCBVc393fPZdfsvYWBhtrzLnGs515p8vJog40Uio3TZKCWB+HbwFk=',
       'x-amz-request-id': 'NGZVZK52DZW9ZQ9E',
       'date': 'Fri, 19 Mar 2021 19:23:24 GMT',
       'server': 'AmazonS3'},
      'RetryAttempts': 0}}




```python
# Define the website configuration
website_configuration = {
    'IndexDocument': {'Suffix': 'index.html'},
}

s3.put_bucket_website(Bucket=BUCKET_NAME,
                      WebsiteConfiguration=website_configuration)
```




    {'ResponseMetadata': {'RequestId': '20QB6C8WXW9G2BJH',
      'HostId': 'FoHOTJPPcMvCLfl9fKQJm2B2ARleDnHi9VP4uS5gD9aBCWwX32C2h898LJHZ927oVMjPFAMMOiY=',
      'HTTPStatusCode': 200,
      'HTTPHeaders': {'x-amz-id-2': 'FoHOTJPPcMvCLfl9fKQJm2B2ARleDnHi9VP4uS5gD9aBCWwX32C2h898LJHZ927oVMjPFAMMOiY=',
       'x-amz-request-id': '20QB6C8WXW9G2BJH',
       'date': 'Fri, 19 Mar 2021 19:25:25 GMT',
       'content-length': '0',
       'server': 'AmazonS3'},
      'RetryAttempts': 0}}



## 4. Use Cloudfront to distribute the static website globally


```python
# Instantiate cloudfront client
cloudfront = boto3.client(
    'cloudfront',
    aws_access_key_id=AWS_ACCESS_KEY,
    aws_secret_access_key=SECRET_ACCESS_KEY
)
```


```python
# Define an origin access identity to later restrict accesses to s3 origin location
origin_identity_response = cloudfront.create_cloud_front_origin_access_identity(
    CloudFrontOriginAccessIdentityConfig={
        'CallerReference': 'hiro-udac',
        'Comment': 'demo'
    }
)

origin_identity_response
```




    {'ResponseMetadata': {'RequestId': '5fc3bea0-ffdc-46bf-b8ff-cdbeda682419',
      'HTTPStatusCode': 201,
      'HTTPHeaders': {'x-amzn-requestid': '5fc3bea0-ffdc-46bf-b8ff-cdbeda682419',
       'etag': 'EWI3UVWK163ZQ',
       'location': 'https://cloudfront.amazonaws.com/2020-05-31/origin-access-identity/cloudfront/E2IH1SFTJD8T91',
       'content-type': 'text/xml',
       'content-length': '445',
       'date': 'Fri, 19 Mar 2021 19:38:58 GMT'},
      'RetryAttempts': 0},
     'Location': 'https://cloudfront.amazonaws.com/2020-05-31/origin-access-identity/cloudfront/E2IH1SFTJD8T91',
     'ETag': 'EWI3UVWK163ZQ',
     'CloudFrontOriginAccessIdentity': {'Id': 'E2IH1SFTJD8T91',
      'S3CanonicalUserId': '66b2eb16d3f5cc05911df75a2222ce40553b7d6d5a523fd30ce0b174271d239043a50d0896571354bc561778b55b82aa',
      'CloudFrontOriginAccessIdentityConfig': {'CallerReference': 'hiro-udac',
       'Comment': 'demo'}}}




```python
# Get origin access identity id from the response
origin_access_identity = origin_identity_response['CloudFrontOriginAccessIdentity']['Id']
origin_access_identity
```




    'E2IH1SFTJD8T91'



# 5. Launch distribution


```python
# Configure the cloudfront distribution
distribution_config = {
    'CallerReference': 'hiro-udac-demo-dist',
    'DefaultRootObject': 'index.html',
    'Origins': {'Quantity': 1,
                'Items': [
                    {
                        'Id':'origin1',
                        'DomainName': BUCKET_NAME + '.s3.us-west-2.amazonaws.com',
                        'OriginPath': '',
                        'S3OriginConfig': {
                            'OriginAccessIdentity': 'origin-access-identity/cloudfront/' + origin_access_identity
                        }
                    }
                ]},
    'DefaultCacheBehavior': {
                    'TargetOriginId':'origin1',
                    'ViewerProtocolPolicy': 'redirect-to-https',
                    'CachePolicyId':'658327ea-f89d-4fab-a63d-7e88639e58f6'
                },
    'Comment': 'hiro-udac-demo',
    'Enabled': True
}
```


```python
# launch distribution 
distribution_response = cloudfront.create_distribution(
    DistributionConfig=distribution_config)

distribution_response
```




    {'ResponseMetadata': {'RequestId': '980b7bff-186a-44e7-ab12-f089a839e1a4',
      'HTTPStatusCode': 201,
      'HTTPHeaders': {'x-amzn-requestid': '980b7bff-186a-44e7-ab12-f089a839e1a4',
       'etag': 'E1I5GF80SNDS6W',
       'location': 'https://cloudfront.amazonaws.com/2020-05-31/distribution/E5B9ZXU3X8MJ',
       'content-type': 'text/xml',
       'content-length': '2936',
       'date': 'Fri, 19 Mar 2021 19:45:22 GMT'},
      'RetryAttempts': 0},
     'Location': 'https://cloudfront.amazonaws.com/2020-05-31/distribution/E5B9ZXU3X8MJ',
     'ETag': 'E1I5GF80SNDS6W',
     'Distribution': {'Id': 'E5B9ZXU3X8MJ',
      'ARN': 'arn:aws:cloudfront::859485984029:distribution/E5B9ZXU3X8MJ',
      'Status': 'InProgress',
      'LastModifiedTime': datetime.datetime(2021, 3, 19, 19, 45, 22, 821000, tzinfo=tzutc()),
      'InProgressInvalidationBatches': 0,
      'DomainName': 'd24eiq5diaxse5.cloudfront.net',
      'ActiveTrustedSigners': {'Enabled': False, 'Quantity': 0},
      'ActiveTrustedKeyGroups': {'Enabled': False, 'Quantity': 0},
      'DistributionConfig': {'CallerReference': 'hiro-udac-demo-dist',
       'Aliases': {'Quantity': 0},
       'DefaultRootObject': 'index.html',
       'Origins': {'Quantity': 1,
        'Items': [{'Id': 'origin1',
          'DomainName': 'udac-static-webapp-demo.s3.us-west-2.amazonaws.com',
          'OriginPath': '',
          'CustomHeaders': {'Quantity': 0},
          'S3OriginConfig': {'OriginAccessIdentity': 'origin-access-identity/cloudfront/E2IH1SFTJD8T91'},
          'ConnectionAttempts': 3,
          'ConnectionTimeout': 10,
          'OriginShield': {'Enabled': False}}]},
       'OriginGroups': {'Quantity': 0},
       'DefaultCacheBehavior': {'TargetOriginId': 'origin1',
        'TrustedSigners': {'Enabled': False, 'Quantity': 0},
        'TrustedKeyGroups': {'Enabled': False, 'Quantity': 0},
        'ViewerProtocolPolicy': 'redirect-to-https',
        'AllowedMethods': {'Quantity': 2,
         'Items': ['HEAD', 'GET'],
         'CachedMethods': {'Quantity': 2, 'Items': ['HEAD', 'GET']}},
        'SmoothStreaming': False,
        'Compress': False,
        'LambdaFunctionAssociations': {'Quantity': 0},
        'FieldLevelEncryptionId': '',
        'CachePolicyId': '658327ea-f89d-4fab-a63d-7e88639e58f6'},
       'CacheBehaviors': {'Quantity': 0},
       'CustomErrorResponses': {'Quantity': 0},
       'Comment': 'hiro-udac-demo',
       'Logging': {'Enabled': False,
        'IncludeCookies': False,
        'Bucket': '',
        'Prefix': ''},
       'PriceClass': 'PriceClass_All',
       'Enabled': True,
       'ViewerCertificate': {'CloudFrontDefaultCertificate': True,
        'MinimumProtocolVersion': 'TLSv1',
        'CertificateSource': 'cloudfront'},
       'Restrictions': {'GeoRestriction': {'RestrictionType': 'none',
         'Quantity': 0}},
       'WebACLId': '',
       'HttpVersion': 'http2',
       'IsIPV6Enabled': True}}}

### Check the distribution is running
![CloudFront_dist](https://i.ibb.co/VYtW80y/Screen-Shot-2021-03-19-at-2-46-23-PM.png)

## View the website


```python
# Copy and Paste the the domain name in your browser and view the website
distribution_response['Distribution']['DomainName']
```




    'd24eiq5diaxse5.cloudfront.net'
    
![website](https://i.ibb.co/m5XZF7B/Screen-Shot-2021-03-19-at-2-59-14-PM.png)


