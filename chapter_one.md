# Chapter one: getting started with a simple database

## Topics

1. Introduction
2. Lab infrastructure
3. The first steps
4. Clean up
10. Some important links

## 1. Introduction
Welcome to the Postgres on Kubernetes workshop based on the cloud-native Postgres
operator. During this workshop, we'll introduce you to all the concepts and technologies
required to run a production-ready Postgres cluster on Kubernetes.

## 2. Lab infrastructure
This lab is meant to run on a ROSA (Red Hat OpenShift on AWS) Kubernetes cluster. The
following URLs are important for this lab:

 - sign-up / username sheet: 
 - OpenShift console: 
 - Postgres cheatsheet: 

The ROSA cluster is made up of three x86_64 EC2 worker nodes. 

## 3. The first steps
Like of any database cluster, the purpose of a Postgres cluster on Kubernetes is to
provide access to important data from applications that connect to it.

In order to make that data available at all times, Postgres clusters are made up of
a single primary (writable) node, and multiple replicas (read-only). When the primary
fails (for whatever reason), the role of the primary can be moved to a replica. The new
primary becomes the single, writable instance in the cluster. This works on traditional
VM or physical machine based infrastructure as well as on Kubernetes.

Building a simple cluster with fail-over capabilities is where we will start our
journey.


Now, log into the OpenShift cluster by going to the URL mentioned in section 2 with your
designated edbuser. In the top of the left-hand navigation menu, make sure it reads
'Administrator'. If it reads 'Developer', click it and switch to the 'Administrator'
perspective.

Click 'Workloads' in the navigation menu, and then on 'Pods'. If you a get permission
error, make sure that the 'Project' menu at the top of the page displays the name of
your edbuser, so 'edbuser09', for example. Do not use the project called
'edbuserXX-tools just yet. We'll use that project later. The list of Pods should be
empty at this point.

In the top right of the screen, click the >_ symbol, and accept the defaults. This will
start a web-based terminal in your browser to interact with the OpenShift Kubernetes
cluster, without you having to install anything on your laptop. 

Starting the terminal will take a minute or so the first time. When it is done, the list
of pods should have a single entry that starts with 'workspace'.

In the terminal, type `kubectl cnp version` or `oc cnp version`. The result of this
command should be similar to
```
Build: {Version:1.20.2 Commit:c42ca1c2 Date:2023-07-28}
```

If not, please contact your instructor.

The `kubectl cnp` command calls a plugin for `kubectl` or `oc` (the Kubernetes and
OpenShift command-line interfaces, respectively) to interact with Postgres clusters on
Kubernetes. We will be using it often during this course. You can minimize the web
terminal with the _ symbol in the top right corner.

From this point on, we'll use `kubectl` in our examples, but feel free to use `oc` and
save a few character on every command. Just type `oc` where we use `kubectl` and all
will work just fine.

In the left hand-menu, switch from the 'Administrator' perspective to the 'Developer'
perspective (by clicking on 'Administrator') and in the Developer perspective menu,
select '+Add'.

On the next page, select 'Import YAML' under 'From Local Machine'.

In the input screen that opens, paste the following code:

```yaml
apiVersion: postgresql.k8s.enterprisedb.io/v1
kind: Cluster
metadata:
  name: my-first-cluster
spec:
  instances: 1
  imageName: quay.io/enterprisedb/postgresql:15.2
  storage:
    size: 1Gi
```

and hit 'Create'. Switch back to the 'Pods' page on the 'Administrator' perspective and
watch your first Postgres cluster on Kubernetes being built! Your cluster is done when
you see an entry called 'my-first-cluster-1' in the Pod list. This is the simplest
cluster you can build with the cloud-native Postgres (CNPG) operator. We'll create more
complex topologies later.

In the web terminal, you can type `kubectl cnp status my-first-cluster`. When your
cluster is ready, the output of that command should show 'Cluster in healthy state' in
green at the top of the output.

## 4. Exploring the cluster
Let's explore what the components of our single node Postgres cluster are. First, let's
look at where the database stores it's data.

In the 'Administrator' perspective, select the 'Storage' menu, and the
'PersistentVolumes'. There should be a 1Gi-sized physical volume listed here called
'XXXX'. This is the physical "disk" that is mounted to our database pod to store data.
This disk is mounted at `/var/lib/postgres/XXX`.

Where the physical volume defines the actual disk (or, more precisely, the block
device), the fact that our cluster owns this block device is defined in
a PhysicalVolumeClaim: a PVC. A PVC defines the link between a Pod and a PV. Note that
the CNPG operator has take care of everything! We defined a single volume of 1Gi for our
database, and that is exactly what we received.

Next, in the web terminal, type `kubectl edit cluster my-first-cluster` and scroll
through the YAML that you'll now see. This YAML defines all aspects of our cluster.
You'll note that it's much longer than the YAML we used to define the cluster. That is
because many parameters were set for our cluster by the CNPG operator with their default
values. 

Look for a section called 'parameters'. These are so-called GUCs, Postgres configuration
parameters, that are also configurable through the operator. Changing or adding
a Postgres configuration parameter in the cluster YAML will set that parameter for the
whole cluster. You won't have to do so much as restart the databases: everything is
fully automated. In fact, changing a parameter will not restart the databases, but just
reload them, which is much faster and unnoticable from an enduser point of view.

Check the PVs and PVCs
Check the cluster YAML after creation
Check the Pods (parent is the cluster object)
Check the services
Check for a route (nope...)
Check the secrets

## 5. Cleanup

Remove your cluster by using the web terminal. Type `kubectl delete cluster
my-first-cluster`. If you have the list of Pods open above your web terminal, you'll see
the `my-first-cluster-1` pod disappear immediately. To check whether the cluster is
really gone, type `kubectl cnp status my-first-cluster`.

## Some important links

Documentation:
- [EDB Postgres for
    Kubernetes](https://www.enterprisedb.com/docs/postgres_for_kubernetes/latest/)
- [Cloud-Native Postgres](https://cloudnative-pg.io/documentation/1.20/)

