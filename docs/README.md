# EDB Postgres on OpenShift workshop

## What is this?

This workshop is meant to introduce you to running Postgres databases on OpenShift through the EDB Postgres for Kubernetes operator.

This is not an introductory workshop on Postgres. It is purely focused on running Postgres on OpenShift, how the EPG4K operator can help you do that, and what the possibilities are.

If you are setting up this workshop as an EDB employee, or workshop instructor, please refer to the [cluster requirements section](cluster_requirements.md) for cluster preparation instructions.

## Workshop outline

1. Concepts of Postgres on Kubernetes & requirements
2. Creating your first Postgres cluster on Kubernetes
    1. Creating a single node cluster
    2. Using the `kubectl cnp` plugin to view cluster status
    3. Setting up replication / scaling out
    4. Switching over the primary to one of the replicas
    5. Configuring WAL storage
    6. Using the `kubectl cnp` plugin to connect to the database
3. Configuring automatic backups
    1. Ingesting data into your cluster
	2. Creating backup configuration
	3. Backing up your database
	4. Creating a backup schedule
	5. Restoring an old copy of the database
4. Performing a Postgres minor update
    1. Inspecting available Postgres images
    2. Chosing an update policy
    3. Performing a minor version rolling update
5. Advanced configuration
    1. Implementing a GUC change
    2. Cluster hibernation
    3. Recovering from a failed replica
    4. Recovering from a failed primary
