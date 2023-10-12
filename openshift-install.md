# OpenShift install
This document describes setting up an OpenShift cluster on AWS for the EDB Postgres on OpenShift workshop.

## Topics

1. Prepare cluster installation
2. Installing cluster
3. Cloning the GitOps repo
3. Create rolebinding
4. Installing GitOps operator
5. Setting up the workshop
6. Final touches


## Installing the cluster
With the appropriate permissions in AWS (i.e. being a member of EDB-OpenShift-Access in
AWS), installing OpenShift should just need:

```
openshift-install create cluster
```

In the questionnaire that follows, make the following selections:
  
  - the SSH key that you want to use. Your default key (in `~/.ssh/id_rsa.pub`) should be fine. 
  - `aws` as the cloud for deployment
  - region `eu-central-1`
  - base domain: `edb.ocp.wzzrd.com` (we'll set up something more official at a later stage in the development of this workshop)
  - name: your first name? Or at least something we can find you by if something is wrong :)
  - Finally, paste your pull secret. You can get a pull secret by going to the [Red Hat Hybrid Cloud Console](https://console.redhat.com/openshift/create/local) and selecting `Download pull secret`. You only have to do that once. Just save the pull secret between installs.

Installation should start now. It will end by passing you a console URL and a kubeadmin password. Make good note of these.

Go to the console URL, and log into the console as kubeadmin with the password you just received. 

Download the CLI tool (called `oc`, an extended version of `kubectl`) through the question mark icon (top right of the screen) and drop it somewhere in your path. 

Click on the `kube:admin` username in the top right and select `copy login command`. Then, click `Display token`, copy the command and paste it into a terminal. Hit Enter. You are now logged into OpenShift. Any subsequent commands you can execute through either `oc` or `kubectl`. 

## Clone the GitOps repo
Run:

```
git clone https://github.com/maxim-edb/cluster-config
```

To fetch the GitOps repo. Enter the `cluster-config` directory and switch the branch to `aws`:

```
git checkout aws
```

Move into the `prep/gitops` directory.

## Create role binding

In the `prep/gitops` directory, exectute:

```
oc create -f cluster-role-binding.yaml
```

## Installing GitOps operator

Next, in the same directory, run:

```
oc create -f gitops-operator.yaml
```

Wait until the installation of the GitOps operator on OpenShift is completely done. This will take a couple of minutes. Easiest way to figure this out is by running:

```
oc get route -n openshift-gitops
```

When the installations has completed, the output looks like:


## Setting up the workshop

We can set up three types of workshops:

- A full workshop, with everything included
- A small workshop, that only has the EDB operator, a webterminal and minio. Fine for small intro workshops.
- A DBA focused workshop, that installs logging, monitoring (Grafana) and pgadmin
- A developer focused workshop, that includes the small workshop + a Tekton installation (for CI/CD pipeline building)

The easiest way to go, if you don't know which one to pick is either the full workshop or the small one.

In the `cluster/apps` directory, you'll find three directories and several files. 

If you want to run the full workshop, run:

```
oc create -f cluster-configs-apps.yaml
```

For the small workshop, run:

```
oc create -f small/cluster-configs-apps.yaml
```

For the DBA workshop, run:

```
oc create -f dba/cluster-configs-apps.yaml
```

And finally, for the developer workshop, run:

```
oc create -f developer/cluster-configs-apps.yaml
```

## Final touches

Finally, if you are setting up the full workshop or the DBA workshop, we need to create a token for Grafana to query Prometheus with (the dashboard tool (Grafana) needs to be able to reach the metric store (Prometheus)). 

In order to do that, we need to perform three small tasks. First, we log into the GitOps dashboard. You can find the URL by running:

```
oc get route -n openshift-gitops
```

Log into that console with the kubeadmin account (click "Log in using OpenShift"). Click the `cluster-configs-grafana` tile. On the next page (you'll see an error), click `App details` in the top left corner.

Go to the parameters tab. Click `edit`. Add a parameter called `prometheus-token` with `foobar` as its value. Save and close the page.

The error disappears now and you'll see resources being created. When this is done, on your terminal, run:

```
oc create token grafana-edbuser15-tools-sa --duration=8760h -n edbuser15-tools
```

This outputs a looong string. Copy the string and paste it as the value of the `prometheus-token` parameter you just created, instead of `foobar`. 

That's it. You are now done. Go grab yourself a coffee.