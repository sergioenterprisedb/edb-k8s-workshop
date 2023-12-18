# Chapter two: a more robust cluster configuration

## Topics

1. A more complete cluster configuration
2. Storage configuration
3. Setting GUC parameters
4. Building the recommended cluster topology
5. Update strategy
6. Resource limits and requests
7. Verifying and changing GUC parameters on the fly

## 1. A more complete cluster configuration

Last chapter, we created a very simple Postgres cluster, consisting of a single Postgres 15 pod. We added a second pod and removed the cluster again. For very simple testing use cases, that might already be interesting capabilities.

For a more robust cluster configuration though, even for non-production use cases, this is probably not enough though. So let's look at a more complete cluster configuration. 

In this chapter, we'll build a cluster that adheres to our recommended cluster topology and storage layout, has a couple of example GUC parameters set, and has an explicitly defined update strategy and a set of resource limits and requests. In the next chapter, we'll look at backing up and restoring data.

We'll use the basic cluster configuration from last chapter and modify it step by step to create a more robust cluster.

## 2. Storage configuration

For production workloads, EDB recommends using a storage class that uses local disks for data and WAL storage. This gives best performance throughput and lowest latency. Note that it is possible to use distributed storage. Just make sure to benchmark your storage and make sure it gives you adequate performance.

For some high performance use cases, we recommend using separate storage for WAL storage and data storage. We can do this by changing the cluser definition to 

```yaml
apiVersion: postgresql.k8s.enterprisedb.io/v1
kind: Cluster
metadata:
  name: more-robust-cluster-edbuserXX
spec:
  instances: 1
  imageName: quay.io/enterprisedb/postgresql:15.2
  storage:
    size: 1Gi
    storageClass: default
  walStorage:
    size: 1Gi
    storageClass: default
```

As you can see, we have added two lines at the end to set up a second persistent volume for walStorage. This volume, like the one for PGDATA, uses the default storage class. If you want a non-default storage class, you can change the `storageClass` field to the name of you desired storage class name.

## 3. Setting GUC parameters

For some workloads, we might want to change some of the GUC parameters. We can do that during cluster creation, as well as on a running cluster. For now, we'll include a GUC parameter in our cluster definition, and verify it later on.

The GUC parameter `work_mem` is one of the most well-known and most often tuning GUC parametes in Postgres. If you are unfamiliar with it: it defines the amount of RAM available per query on a database. It's default setting is 4MiB. If you have queries that perform large sort operations, it might make sense to increase the amount memory available per query by tuning this GUC parameter. It has to be done carefully though, because it might also result in memory contention if increased too far.

Let's say we want to make this 4MiB default explicit in our cluster configuration for now. We can do this by changing the cluster configuration to:

```yaml
apiVersion: postgresql.k8s.enterprisedb.io/v1
kind: Cluster
metadata:
  name: more-robust-cluster-edbuserXX
spec:
  instances: 1
  imageName: quay.io/enterprisedb/postgresql:15.2
  storage:
    size: 1Gi
    storageClass: default
  walStorage:
    size: 1Gi
    storageClass: default
  postgresql:
    parameters:
      work_mem: "4MB"
```

Note the value for `work_mem` is a string!

## 4. Building the recommended cluster topology

The recommanded topology for a highly available cluster is to have a primary, at least one synchronous replica, and a third replica. This topology guarantees a fast failover between a failed primary and the synchronous replica, and the availability for the other replica to get up to date quickly after failover.

In order to achieve this topology, we want to create a cluster with 3 instances and set the `minSyncReplicas` and `maxSyncReplicas` parameters. These settings make sure we have at least a certain number of replicas (`minSyncReplicas`) and what the quorum is (`minSyncReplicas`). You can read more on this in the [official documentation](https://www.enterprisedb.com/docs/postgres_for_kubernetes/latest/replication/#synchronous-replication).

After adding these two parameters, our cluster configuration looks like the one below. Note that we set 1 for both `minSyncReplicas` and `maxSyncReplicas`, because for our use case, we are satisfied by having a single synchronous replica.

```yaml
apiVersion: postgresql.k8s.enterprisedb.io/v1
kind: Cluster
metadata:
  name: more-robust-cluster-edbuserXX
spec:
  instances: 3
  imageName: quay.io/enterprisedb/postgresql:15.2
  minSyncReplicas: 1
  maxSyncReplicas: 1
  storage:
    size: 1Gi
    storageClass: default
  walStorage:
    size: 1Gi
    storageClass: default
  postgresql:
    parameters:
      work_mem: "4MB"
```

## 5. Update strategy

The EDB Postgres for Kubernetes operator can update out clusters without our intervention, if we let it, but we can also keep full control over the update process.

The two parameters that define the behavior of the operator in this case, are `primaryUpdateStrategy` and `primaryUpdateMethod`. Let's go over them both.

The first parameter, `primaryUpdateStrategy`, tells the operator whether or not to  update the primary without human intervention or not. Note that this means that when the cluster definition was changed, and a new version of Postgres was set in the `imageName` parameter, the operator will *always* update the replicas to the new version of Postgres without human intervention! The operator updates the cluster by updating the replicas first, and then, either automatically or not, updates the primary.

The `primaryUpdateStrategy` parameter can be either `supervised` or `unsupervised`. The `unsupervised` parameter will automatically update the primary after the replicas have been updated, the `unsupervised` parameter wait for an administrator to manually perform a switchover or restart, after which the former primary will be updated.

The second parameter is called `primaryUpdateMethod`, and it has two valid options: `switchover` and `restart`. This parameter defines how to update the primary pod. As the member pods of our cluster cannot be updated in place, they have to be restarted with a new image. If the `primaryUpdateMethod` is set to `switchover`, the operator will promote the best replica to primary, and then replace the former primary by a new container based on the new container image. If it is `restart`, the operator will not perform a switchover, but rather it will stop the primary and then start a new container based on the new container image in its place. 

Which of these methods you should choose, depends on your application and the RPO and RTO requirements around that application. We recommend reading the [official documentation](https://www.enterprisedb.com/docs/postgres_for_kubernetes/latest/rolling_update/) on this topic to decide for yourself.

For our example, we'll make the defaults (which are `unsupervised` and `restart`) explicit, by changing our cluster definition to:


```yaml
apiVersion: postgresql.k8s.enterprisedb.io/v1
kind: Cluster
metadata:
  name: more-robust-cluster-edbuserXX
spec:
  instances: 3
  imageName: quay.io/enterprisedb/postgresql:15.2
  minSyncReplicas: 1
  maxSyncReplicas: 1
  primaryUpdateStrategy: unsupervised
  primaryUpdateMethod: restart
  storage:
    size: 1Gi
    storageClass: default
  walStorage:
    size: 1Gi
    storageClass: default
  postgresql:
    parameters:
      work_mem: "4MB"
```

## 6. Resource limits and requests

For each cluster we create - or any Deployment-like object on Kubernetes, really - we can optionally set resource requests and limits. This means we can explicitly define the amount of memory and cpu cycles we want our cluster to request, and the amount of memory and cpu cycles our cluster will be limited to.

If we do not set any limits or resources, the cluster can ask for whatever the Kubernetes cluster node has avaiable, which - for obvious reasons - might not be what you want or expect. This behavior is called for a cluster to be "Best Effort". Quality of service (QoS) classes like "Best Effort" are visible when you ask for a cluster's status with `kubectl cnp status more-robust-cluster-edbuserXX`.

If we set requested resources only, a workload QoS class will be "Burstable", meaning it will request a certain amount of resources, but can grab more if needed. The Kubernetes scheduler looks at the requested resources when selecting a node for a pod to run on.

For Postgres database workloads, EDB recommends making sure pods get the "Guaranteed" QoS class. A pod gets that class when you define both resource requests and limits and set those to the *same value*:

```yaml
apiVersion: postgresql.k8s.enterprisedb.io/v1
kind: Cluster
metadata:
  name: more-robust-cluster-edbuserXX
spec:
  instances: 3
  imageName: quay.io/enterprisedb/postgresql:15.2
  minSyncReplicas: 1
  maxSyncReplicas: 1
  primaryUpdateStrategy: unsupervised
  primaryUpdateMethod: restart
  storage:
    size: 1Gi
    storageClass: default
  walStorage:
    size: 1Gi
    storageClass: default
  postgresql:
    parameters:
      work_mem: "4MB"
  resources:
    requests:
      memory: "1024Mi"
      cpu: 1
    limits:
      memory: "1024Mi"
      cpu: 1
```

As you can see, this sets the request and limit for both CPUs and memory to the same level, giving this new cluster the "Guaranteed" QoS class. This is what we want! Mind that changing either the requests or the limits of an existing cluster will restart all pods in that cluster. This is a limitation of Kubernetes, not the EDB Postgres for Kubernetes operator.

## 7. Verifying and changing GUC parameters on the fly

Ok, let's create this cluster and verify how changing GUC parameters works. Apply the cluster YAML by clicking the "⊕" sign in the top right corner of the screen and pasting the above YAML into the text field. Make sure you change the name of the cluster by making it reflect your username (e.g. `edbuser15` instead of `edbuserXX`)!

After a few minutes, the list of pods in your `edbuserXX` namespace should look like the screenshot below.



Open the web terminal again (if you haven't got it open already), and have it display the cluster configuration by typing `kubectl cnp psql more-robust-cluster-edbuserXX -- -U postgres`. In the psql session you are now in, type `show work_mem`. This should output "4MB".

Get the time when the primary pod was started, by typing the following into your web terminal: `kubectl describe pod more-robust-cluster-edbuserXX | grep -A 1 "State:.*Running"`. This shows a timestamp for when the primary pod of your cluster was started. 

Ok, now we'll change the `work_mem` setting. We'll see that the operator executed a `pg_ctl reload` under the hood and not a complete restart of the cluster. 

In your web terminal, type `kubectl edit cluster more-robust-cluster-edbuserXX`. Find the `work_mem` setting, and change it to 6MB from 4MB. Save the editor.

Run `kubectl cnp psql more-robust-cluster-edbuserXX -- -U postgres` again, and run `show work_mem`. This should now show "6MB". Finally, run `kubectl describe pod more-robust-cluster-edbuserXX | grep -A 1 "State:.*Running"` again, and compare the output to the output from executing this same command earlier. It should show the same output, as the pod was not restarted to apply the new GUC parameter: the operator was clever enough to only execute a `pg_ctl reload`. Nice!
