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
 

