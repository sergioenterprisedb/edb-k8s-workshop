# OpenShift install
This document describes setting up an OpenShift cluster on AWS for the EDB Postgres on OpenShift workshop.

## Topics

1. Installing cluster manually
2. Installing the cluster automatically
3. Cloning the GitOps repo
4. Create rolebinding
5. Installing GitOps operator
6. Setting up the workshop
7. Final touches


## Installing the cluster manually
With the appropriate permissions in AWS (i.e. being a member of EDB-OpenShift-Access in
AWS), installing OpenShift should just need:

```
openshift-install create cluster
```

```
!! Do not remove the directory you are running the installer in,
!! nor any of its artifacts until after you delete the cluster again
```

In the questionnaire that follows, make the following selections:
  
  - the SSH key that you want to use. Your default key (in `~/.ssh/id_rsa.pub`) should be fine. 
  - `aws` as the cloud for deployment
  - region `eu-central-1`
  - base domain: `edb.ocp.wzzrd.com` (Sergio, Borys, this needs to change to something
      like `workshop.edb.com`!)
  - name: use your first name
  - Finally, paste your pull secret. You can get a pull secret by going to the [Red Hat Hybrid Cloud Console](https://console.redhat.com/openshift/create/local) and selecting `Download pull secret`. You only have to do that once. Just save the pull secret between installs.

Installation should start now. It will end by passing you a console URL and a kubeadmin password. Make good note of these.

Go to the console URL, and log into the console as kubeadmin with the password you just received. 

Download the CLI tool (called `oc`, an extended version of `kubectl`) through the question mark icon (top right of the screen, click it, select `Command line tools`), untar it, and drop the `oc` binary somewhere in your path. 

Click on the `kube:admin` username in the top right and select `copy login command`. Then, click `Display token`, copy the command and paste it into a terminal. Hit Enter. You are now logged into OpenShift. Any subsequent commands you can execute through either `oc` or `kubectl`. 

----

Alternatively to copying the login command, you can run the following command in the in the directory that you started the `openshift-install` command from: 
```
export KUBECONFIG=PATH_TO_YOUR_INSTALL_DIRECTORY/auth/kubeconfig
```

The exact & correct command for this export will be in the `openshift-install` output. Just copy and paste it. This sets up your commandline tools (oc and kubectl) to connect to the cluster with kubeadmin privileges.


## Installing the cluster automatically

Use this file below, fill in the appropriate fields, and save it as
`install-config.yaml.orig` on your system. When you want to create a cluster, copy it to
an empty directory, renaming it to `install-config.yaml`. This way, the questionnaire
will be skipped during install:

```yaml
additionalTrustBundlePolicy: Proxyonly
apiVersion: v1
baseDomain: INSERT EDB OWNED DOMAIN HERE
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform: {}
  replicas: 3
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform: {}
  replicas: 3
metadata:
  creationTimestamp: null
  name: CHANGE THIS TO YOUR NAME
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.0.0.0/16
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  aws:
    region: CHANGE THIS TO THE REGION YOU WANT TO RUN IN, e.g. eu-central-1
publish: External
pullSecret: PASTE PULL SECRET HERE
sshKey: |
  PASTE YOUR SSH PUBLIC KEY HERE
```

From the directory this file is in, just run `openshift-install create cluster`. No more
questionnaire.

## Clone the GitOps repo
Run:

```
git clone git@github.com:maxim-edb/edb-k8-workshop.git
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

When the installations has completed, the output looks something like:

```bash
oc get route -n openshift-gitops
NAME                      HOST/PORT                                                               PATH   SERVICES                  PORT    TERMINATION            WILDCARD
kam                       kam-openshift-gitops.apps.sergio.workshop.edb.com                              kam                       8443    passthrough/None       None
openshift-gitops-server   openshift-gitops-server-openshift-gitops.apps.sergio.workshop.edb.com          openshift-gitops-server   https   passthrough/Redirect   None
```

Make note of the `openshift-gitops-server` route. Copy the HOST/PORT part into your browser and nagivate to it.

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

Log into that console with the kubeadmin account (click "Log in via OpenShift"). Click the `cluster-configs-grafana` tile. On the next page (you'll see an error), click `App details` in the top left corner.

Go to the parameters tab. Click `edit`. Add an external variable called `prometheus-token` with `foobar` as its value. Save and close the page.

The error will disappear now, and you'll see resources being created. When this is done, on your terminal, run:

```
oc create token grafana-edbuser15-tools-sa --duration=8760h -n edbuser15-tools
```

Note: when `Sync status` is `Synced`, all resources have been successfully created.

This outputs a looong string. Copy the string and paste it as the value of the `prometheus-token` parameter you just created, instead of `foobar`. 

That's it. You are now done. Go grab yourself a coffee. 

## Scaling

If you want to scale beyond the three default worker nodes (you probably do), go to
Compute -> MachineSets and for each line, click the three dot menu on the right, select
"Edit Machine count" and insert the desired number of worker nodes. Each line on this
screen represents an availability zone, so if you scale one line, you should probably
scale all lines.

You can follow progress on the Compute -> Machines page. Your new worker nodes are ready
when they are listed on Compute -> Nodes with role "worker" and Status "Ready".

You can scale worker nodes both up and down this way.

## Deleting

From the same directory from where you ran `openshift-install` to install the cluster (this is why you should not remove it after installing!!), run: 

```
openshift-install destroy cluster
```

Wait for five minutes. Do not shut down your machine until this is done, as the installer runs from your machine.
