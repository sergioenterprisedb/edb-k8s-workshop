# Cluster requirements

This document describes setting up a shared ROSA cluster for the EDB Postgres on
OpenShift workshop.

## Topics

1. Prepare cluster installation
2. Installing cluster
3. Set up `dedicated-admin` for the instructor
4. Create user accounts for participants
5. Installing the EDB operator
7. Installing the Grafana operator
8. Preparing object storage for backups

Separate workshops exist for using Postgres in a Tekton pipeline, using GitOps to
manage Postgres clusters on Kubernetes, and using centralized logs to keep track of
Postgres clusters.


## Prepare cluster installation
- scaling for users?
- setting up AWS credentials
- getting enough quota (test!)
- getting membership of Red Hat org
- linking RH account
- Point to Red Hat docs on ROSA?

## Logging into AWS
In order to log into AWS, run the following command. This will create a local set of
credentials that we can use for a couple of hours to manage infrastucture on AWS:

```
aws sso login --profile ocp
```

The above command assumes the existence of an AWS profile in `~/.aws/config` called
`ocp`. If you named your profile differently, change the above commandline accordingly.


## Installing the cluster
With the appropriate permissions in AWS (i.e. being a member of EDB-OpenShift-Access in
AWS), installing ROSA should just need:

```
rosa create cluster --cluster-name=YOUR_NAME --region=YOUR_REGION --sts --mode=auto
```

If you set up a separate profile for EDB-OpenShift-Access the command would be as
follows (assuming you profile is called `ocp`):

```
rosa create cluster --cluster-name=YOUR_NAME --region=YOUR_REGION --sts --mode=auto
--profile ocp
```

## Set up `dedicated-admin` for the instructor

From that same terminal - still assuming your profile name is ocp - run the following
command to create a `dedicated-admin` account for the instructor. This account has
(almost) cluster-admin level privileges and can be used to configure anything on the
cluster we need today.

```
rosa create admin --cluster YOUR_NAME --profile ocp
```

## Installing the EDB operator
To install the EDB Postgres for Kubernetes (EPG4K) operator, log into the cluster with
the account with `dedicated-admin` privileges and navigate to "OperatorHub" through the
Administrator menu.

In the OperatorHub type `edb` in the box that reads `Filter by keyword`. Click the tile
of the EPG4K operator, and select `Install` at the top of the screen.

Accept the defaults on the next page by clicking `Install` again. Navigate to "Installed
Operators" through the Administrator menu, and wait for the Status of the EPG4K operator
to become `Succeeded`.

You have now installed the EPG4K operator. It is enabled for all namespaces on the
cluster, so the workshop participants don't have to worry about this bit anymore.

## Installing the Grafana operator
Because we'll use Grafana dashboards during the workshop, we'll deploy the Grafana
operator as well. Search for `grafana` in the OperatorHub, and - following the same
procedure as before - install the operator in all namespaces.

## Create an S3 bucket
In the AWS console, create an S3 bucket in the same region as your ROSA cluster. The
bucket does not require public access. Create a user for your S3 bucket, and give the
user an inline policy with the following contents. Make sure to replace BUCKET_NAME with
the name of your bucket.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetBucketLocation",
                "s3:ListAllMyBuckets"
            ],
            "Resource": "arn:aws:s3:::*"
        },
        {
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::BUCKET_NAME",
                "arn:aws:s3:::BUCKET_NAME/*"
            ]
        }
    ]
}
```

Create an access key (through the security credentials menu of your new user) and make
a note of both the key and the secret. We'll need this to create secrets later on, that
in turn will be used to make database backups to S3.

## Automate it all!
Clone `https://github.com/maxim-edb/cluster-config.git` and switch to the `dev` branch.
Move into the `prep/gitops` directory. Apply all three YAML files in that directory.

The `gitops-operator` YAML installs the ArgoCD operator into your cluster. The
`cluster-role-binding` YAML makes the ArgoCD service account capable of managing the
cluster and the `cluster-configs-apps` file set up other GitOps apps to manage different
aspects of the cluster, like the EDB operator.

Applying the `cluster-configs-apps.yaml` from the `prep/gitops` directory will set up
the full workshop. Setting up smaller workshops or with a focus group in mind
(developers or dbas) is work in progress.
