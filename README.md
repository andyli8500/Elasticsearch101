# Elasticsearch101

### Create ES domain (CLI)
```bash
aws es create-elasticsearch-domain --domain-name test --elasticsearch-version 5.1 --elasticsearch-cluster-config InstanceType=m4.large.elasticsearch,InstanceCount=5 --ebs-options EBSEnabled=true,VolumeType=gp2,VolumeSize=50
```
### Update the access policies
1. IP-based Policy
```bash
aws es update-elasticsearch-domain-config --domain-name test --access-policies '{  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": "es:*",
      "Resource": "arn:aws:es:ap-southeast-2:<account>:domain/mytestdomain/*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "<ip address>"
        }
      }
    }
  ]
}'

```
2. IAM User and role-based policy
```bash
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<account>:user/admin"
      },
      "Action": "es:*",
      "Resource": "arn:aws:es:ap-southeast-2:<account>:domain/test/*"
    }
  ]
}
```
3. Resource-based (e.g. allow specific account have access to all the domains)
```sh
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "<account>"
        ]
      },
      "Action": [
        "es:*"
      ],
      "Resource": "arn:aws:es:ap-southeast-2:<account>:domain/*"
    }
  ]
}
```

- What happens when you try to access your cluster from an instance running a different role?
  > If the role (different role) is not given the permission to the ES explicitly, it will be denied
  
- What happens when you try to access your cluster from an instance with a different IP address?
  > If IP is not authorized within the access policy, the actions will be denied.
  
### Make backup and restore (Manual)
- Create IAM Role
```sh
ESTEST_MANUAL_SNAPSHOT_ROLENAME=ESTEST_Manual_Snapshot_Role
ESTEST_MANUAL_SNAPSHOT_IAM_POLICY_NAME=ESTEST_Manual_Snapshot_IAM_Policy
ESTEST_MANUAL_SNAPSHOT_S3_BUCKET=es-snapshots
ESTEST_IAM_MANUAL_SNAPSHOT_ROLE_ARN=arn:aws:iam::$ESTEST_ACCOUNT_ID:role/$ESTEST_MANUAL_SNAPSHOT_ROLENAME    

aws iam create-role \
        --role-name "$ESTEST_MANUAL_SNAPSHOT_ROLENAME" \
        --output text \
        --query 'Role.Arn' \
        --assume-role-policy-document '{
              "Version": "2012-10-17",
              "Statement": [{
                  "Effect": "Allow",
                  "Principal": { "Service": "es.amazonaws.com"},
                  "Action": "sts:AssumeRole"
                }
              ]
            }'

cat << EOF > /tmp/iam-policy_for_es_snapshot_to_s3.json
{
    "Version":"2012-10-17",
    "Statement":[{
            "Action":["s3:ListBucket"],
            "Effect":"Allow",
            "Resource":["arn:aws:s3:::$ESTEST_MANUAL_SNAPSHOT_S3_BUCKET"]
        },{
            "Action":[
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject",
                "iam:PassRole"
            ],
            "Effect":"Allow",
            "Resource":["arn:aws:s3:::$ESTEST_MANUAL_SNAPSHOT_S3_BUCKET/*"]
        }
    ]
}
EOF

aws iam put-role-policy \
        --role-name   "$ESTEST_MANUAL_SNAPSHOT_ROLENAME"   \
        --policy-name "$ESTEST_MANUAL_SNAPSHOT_IAM_POLICY_NAME" \
        --policy-document file:///tmp/iam-policy_for_es_snapshot_to_s3.json
```
 - Register Snapshot
 > Note that: only the user has access to the role can register s3 bucket Use the following script to register (CLI does not support signing)
```
from boto.connection import AWSAuthConnection

class ESConnection(AWSAuthConnection):

    def __init__(self, region, **kwargs):
        super(ESConnection, self).__init__(**kwargs)
        self._set_auth_region_name(region)
        self._set_auth_service_name("es")

    def _required_auth_capability(self):
        return ['hmac-v4']

if __name__ == "__main__":

    client = ESConnection(
            region='ap-southeast-2',
            host='search-<>-<>.ap-southeast-2.es.amazonaws.com',
            aws_access_key_id=os.environ['AWS_ACCESS_KEY_ID'],
            aws_secret_access_key=os.environ['AWS_SECRET_ACCESS_KEY'],
            is_secure=False)
    print 'Registering Snapshot Repository'
    resp = client.make_request(method='POST',
            path='/_snapshot/weblogs-index-backups',
            data='{"type": "s3","settings": { "bucket": "<bucketname>","region": "ap-southeast-2","role_arn": "arn:aws:iam::<account>:role/<rolename>"}}')
    body = resp.read()
    print body
 ```
 
 - Make a snapshot
 ```
 curl -XPUT 'http://<Elasticsearch_domain_endpoint>/_snapshot/snapshot_repository/snapshot_name'
 ```
 
 - Make restore
 ```
 curl -XPOST 'http://<Elasticsearch_domain_endpoint>/_snapshot/snapshot_repository/snapshot_name/_restore'
 ```
 
 ### Reverse Proxy
 ##### Use proxy to simplify request signing [Github resources](https://github.com/abutaha/aws-es-proxy)
 Run the following to start the proxy.
 ```sh
 ./aws-es-proxy -endpoint https://test-es-somerandomvalue.eu-west-1.es.amazonaws.com
 ```
 The above proxy listen at port 9200, to change to port 8080:
 ```sh
 ./aws-es-proxy -listen 8080 -endpoint https://test-es-somerandomvalue.eu-west-1.es.amazonaws.com
 or
 ./aws-es-proxy -listen 127.0.0.1:8080 -endpoint ...
 ```
 
 ##### Set IAM to allow or deny access to different users
 In your access policy of Elasticsearch Service,
 ```json
 {
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/<user>"
      },
      "Action": "es:*",
      "Resource": "arn:aws:es:us-west-2:123456789012:domain/mydomain/*"
    }
  ]
}
 ```
 
### Automate data upload to Elasticsearch using Firehose
1. Create a Firehose delivery stream. Reuse the Firehose template, but including the following:
 ```json
    ...
    {
      "Sid": "",
      "Effect": "Allow",
      "Action": [
        "es:DescribeElasticsearchDomain",
        "es:DescribeElasticsearchDomains",
        "es:DescribeElasticsearchDomainConfig",
        "es:ESHttpPost",
        "es:ESHttpPut"
      ],
      "Resource": [
        "arn:aws:es:us-east-1:563402962900:domain/esdomain",
        "arn:aws:es:us-east-1:563402962900:domain/esdomain/*"
      ]
    },
    {
      "Sid": "",
      "Effect": "Allow",
      "Action": [
        "es:ESHttpGet"
      ],
      "Resource": [
        "arn:aws:es:us-east-1:563402962900:domain/esdomain/_all/_settings",
        "arn:aws:es:us-east-1:563402962900:domain/esdomain/_cluster/stats",
        "arn:aws:es:us-east-1:563402962900:domain/esdomain/movies*/_mapping/string",
        "arn:aws:es:us-east-1:563402962900:domain/esdomain/_nodes",
        "arn:aws:es:us-east-1:563402962900:domain/esdomain/_nodes/stats",
        "arn:aws:es:us-east-1:563402962900:domain/esdomain/_nodes/*/stats",
        "arn:aws:es:us-east-1:563402962900:domain/esdomain/_stats",
        "arn:aws:es:us-east-1:563402962900:domain/esdomain/movies*/_stats"
      ]
    }
    ...
```

2. Use AWS SDK write data to Firehose:
```python
import boto3
import sys
import json
import decimal


def put_record(client, fname):
    with open(fname) as fin:
        raw_data = json.load(fin)

        for i, record in enumerate(raw_data):
            print('Processing', i)
            movie = json.dumps(record, ensure_ascii=False)
            client.put_record(DeliveryStreamName='es-deliver', Record={'Data': movie})
            

def main():
    client = boto3.client('firehose', region_name='us-east-1')
    fname = sys.argv[1]

    put_record(client, fname)


if __name__ == '__main__':
    main()

```

### Logstash
##### Install Logstash in EC2 instance
- Install Java 8
```sh
cd ~
wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "http://download.oracle.com/otn-pub/java/jdk/8u60-b27/jdk-8u60-linux-x64.rpm"
```
Then
```sh
sudo yum localinstall jdk-8u60-linux-x64.rpm
```
Set `JAVA_HOME`
```sh
export JAVA_HOME=/usr/java/jdk1.8.9_60
```
- install `rake` and `bundler`
```
gem install rake
gem install bundler
```















