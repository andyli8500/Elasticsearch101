# Elasticsearch101

### Create ES domain (CLI)
```bash
aws es create-elasticsearch-domain --domain-name test --elasticsearch-version 5.1 --elasticsearch-cluster-config InstanceType=m4.large.elasticsearch,InstanceCount=5 --ebs-options EBSEnabled=true,VolumeType=gp2,VolumeSize=50
```
Update the access policies
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
