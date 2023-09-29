# EDB Postgres on OpenShift workshop

## What is this?

This workshop is meant to introduce you to running Postgres databases on OpenShift through the EDB Postgres for Kubernetes operator.

This is not an introductory workshop on Postgres. It is purely focused on running Postgres on OpenShift, how the EPG4K operator can help you do that, and what the possibilities are.

If you are setting up this workshop as an EDB employee, or workshop instructor, please refer to the [cluster requirements section](cluster_requirements.md) for cluster preparation instructions.

## Workshop outline

# Chapter one: getting started with a simple database
# Chapter two: creating a replicated cluster

1. Concepts of Postgres on Kubernetes & requirements
2. Installing the operator on OpenShift
3. Creating your first Postgres cluster on Kubernetes
4. Configuring additional options:  
	1. Creating backup configuration
	2. Backing up your database
	3. Creating a backup schedule
	4. Configuring the database itself
5. Performing a Postgres minor update
6. Configuring RBAC
7. Installing and using the `kubectl cnp` plugin
8. Creating and configuring the PodMonitor object
9. Forcing a failover by killing a pod
10. Create a second cluster based on an existing backup
