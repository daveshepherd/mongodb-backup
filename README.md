# Prerequisites

## Create Mongo backup user:

```
db.createUser({
  user: "backup-user",
  pwd: "{password}",
  roles: [ "backup" ]
});
```

## Create S3 user

In IAM:

1. Create Policy
 * Name: same as bucket name
 * JSON: as below, replacing resource name with you bucket name
 
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject"
            ],
            "Resource": "arn:aws:s3:::your_bucket_name/*",
            "Condition": {
                "ForAnyValue:IpAddress": {
                    "aws:SourceIp": [
                        "YOUR_1st_IP_HERE/32",
                        "YOUR_2nd_IP_HERE/32"
                    ]
                }
            }
        }
    ]
}
```
2. Create Group with matching name and attach the new policy
3. Create User with matching name, programmatic access, the new group, and note down access key ID and secret access key

## Create S3 bucket

In S3 management UI:

1. 'Create Bucket'
2. Name and region:
 * Bucket name: {cluster}-mongodb-backups, e.g. instance-mongodb-backups
 * Region: EU (London)
3. Set properties:
 * Versioning: enabled
 * Tags: tennant
4. Set permissions
 * Set this up later
5. Review and Create Bucket

Also create a retension lifecycle to delete old files, in pre-prod.

## Create encryption key

In IAM create and encryption key in the same region as the bucket granting usage to the user created above.

# Configuration

```
MONGODB_HOST={comma-separated-list-of-hosts}
MONGODB_REPLICASET={optional-replica-set}
MONGODB_PORT=27017
MONGODB_DB={optional-db-to-backup/restore}
MONGODB_USER=backup-user
MONGODB_PASS={from-mongo-user-created-above}
BUCKET={s3-backet-created-above}
GPG_PHRASE={phrase}
ACCESS_KEY={from-s3-user-created-above}
SECRET_KEY={from-s3-user-created-above}
```

# Restore a database

Set MONGODB_DB to the name of a specific database or "all" to restore all DBs in the backup location. Set BUCKET to the name of the S3 bucket, POLICY_CYCLE to defined the sub dir in the bucket and BACKUP_NAME (e.g. 2017.06.29.140428) to specify the backup to restore and use /restore.sh as the CMD.

Test restore locally to a Docker mongo instance:

Create a local mongo database and create a user for the restore process:

```
docker run -d -p 27017:27017 mongo:3.4
mongo 'mongodb://localhost/admin' --quiet --eval 'db.createUser({user: "restore-user",pwd: "password",roles: [ "dbAdmin" ]});'
```

Make sure you have an appropriate .env file, e.g.:

```
MONGODB_HOST=10.10.10.1
MONGODB_PORT=27017
MONGODB_USER=restore-user
MONGODB_PASS=password
BUCKET={bucket/cluster-to-restore}
GPG_PHRASE={phrase}
ACCESS_KEY={aws-access-key}
SECRET_KEY={aws-secret-key}
MONGODB_DB=all
BACKUP_NAME=2018.08.24.034719_Friday-24-August
POLICY_CYCLE=weekly
```

Where:
* MONGODB_HOST is your laptop's IP (localhost won't work because the restore is running inside Docker)
* MONGODB_DB is the database to restore
* BACKUP_NAME is the backup iteration in S3 to restore, this should be the date/time that the backup was taken
* POLICY_CYCLE is the directory where you find the backup file.

Run the public image, passing the /restore.sh command (specify mongo version as appropriate):

```
docker run -it --rm -v /tmp:/tmp --env-file=.env quay.io/wealthwizards/mongodb-backup:3.4 /restore.sh
```

To restore to a mongo cluster ensure the mongo host is a list of names/IPs and the replica set is defined in addition to
the above:

```
MONGODB_HOST=10.0.0.1,10.0.0.2
MONGODB_REPLICASET=my-replica-set
```