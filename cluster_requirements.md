# Cluster requirements

This document describes setting up a shared ROSA cluster for the EDB Postgres on OpenShift workshop.

## Topics

1. Prepare cluster installation
2. Installing cluster
3. Set up `dedicated-admin` for the instructor
4. Create user accounts for participants
5. Installing the EDB operator
6. Preparing object storage for backups


## Prepare cluster installation
- scaling for users?
- setting up AWS credentials
- getting enough quota (test!)
- getting membership of Red Hat org
- linking RH account
- Point to Red Hat docs on ROSA?

## Installing the cluster
is the `--mode=auto` option enough?

## Installing the operator
To install the EDB Postgres for Kubernetes (EPG4K) operator, log into the cluster with the account with `dedicated-admin` privileges and navigate to "OperatorHub" through the Administrator menu.

In the OperatorHub type `edb` in the box that reads `Filter by keyword`. Click the tile of the EPG4K operator, and select `Install` at the top of the screen.

Accept the defaults on the next page by clicking `Install` again. Navigate to "Installed Operators" through the Administrator menu, and wait for the Status of the EPG4K operator to become `Succeeded`.

You have now installed the EPG4K operator.



