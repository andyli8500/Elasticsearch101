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
 
 
