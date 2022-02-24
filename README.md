# Testing the fileIntegrity operator

## TOC

- [Testing the fileIntegrity operator](#testing-the-fileintegrity-operator)
  - [TOC](#toc)
  - [Introduction](#introduction)
  - [Installing File Integrity Operator](#installing-file-integrity-operator)
  - [Deploying the File Integrity Operator](#deploying-the-file-integrity-operator)
  - [Retrieving the File Integrity Operator Results](#retrieving-the-file-integrity-operator-results)
  - [Reviewing the AIDE configuration](#reviewing-the-aide-configuration)
  - [Simulating a failure](#simulating-a-failure)
  - [Reinitializing the AIDE database](#reinitializing-the-aide-database)
  - [References](#references)

## Introduction

The [File Integrity Operator](https://docs.openshift.com/container-platform/4.7/security/file_integrity_operator/file-integrity-operator-installation.html) is used to watch for changed files on any node within an OpenShift cluster. Once deployed and configured, it will watch a set of pre-configured locations and report if any files are modified in any way that were not approved. This operator works in sync with MachineConfig so if you update a file through MachineConfig, once the files are updated, the File Integrity Operator will update its database of signatures to ensure that the approved changes do not trigger an alert. The File Integrity Operator is based on the OpenSource project [AIDE](https://aide.github.io/).

We will use Operator Hub to install an instance of the File Integrity Operator on our cluster and we will then test it by modifying files in multiple ways to see how changes are reported.

## Installing File Integrity Operator

Using the OpenShift Console:

1. Log into your cluster
2. Select Operators->Operator Hub
3. Search for File Integrity Operator and Select it.
4. Select Install and leave all defaults, and click install.

From the command line:

1. Login to your cluster with the oc command
2. Apply the yaml located in the install directory
   `oc apply -f install/`
3. Verify that the install worked

```shell
$ oc get csv -n openshift-file-integrity
NAME                              DISPLAY                   VERSION   REPLACES   PHASE
file-integrity-operator.v0.1.13   File Integrity Operator   0.1.13               Succeeded
$ oc get deploy -n openshift-file-integrity
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
file-integrity-operator   1/1     1            1           41h
```

## Deploying the File Integrity Operator

Using the OpenShift Console:

1. Log into your cluster
2. Select Operators->Installed Operators
3. Ensure that the Project Dropdown has "All Projects" selected
4. Search for File Integrity Operator and Select it.
5. Select "FileIntegrity"
6. Select "Create FileIntegrity"
7. Enter a name for your FileIntegrity instance. Use "cluster-fileintegrity", as the name. This will be used for the remainder of this demo.
8. Leave all other options as default, and click Apply

From the command line:

1. Login to your cluster with the oc command
2. Apply the fio-cr.yaml to create a default instance
   `oc create -f fio-cr.yml`
3. Validate that the default config is running
   `oc get fileintegrities/cluster-fileintegrity -n openshift-file-integrity -o jsonpath="{ .status.phase }"`

The status will be one of three phases:

- Pending - This is the inital state, while the FileIntegrity instance starts
- Active - This is the primary state and indicates that the AIDE daemon is active
- Initializing - This state indicates that the AIDE database is being initialized

Once the status is "Active", you can move onto the next steps.

## Retrieving the File Integrity Operator Results

The first round of data collection will take some time. The amount of time it takes for the scan to run will be hardware dependent. Once complete we can view the status of every node:

```shell
$ oc get fileintegritynodestatuses -n openshift-file-integrity
 NAME                                            NODE                       STATUS
cluster-fileintegrity-ocp47-jzgl9-master-0       ocp47-jzgl9-master-0       Succeeded
cluster-fileintegrity-ocp47-jzgl9-master-1       ocp47-jzgl9-master-1       Succeeded
cluster-fileintegrity-ocp47-jzgl9-master-2       ocp47-jzgl9-master-2       Succeeded
cluster-fileintegrity-ocp47-jzgl9-worker-6svd6   ocp47-jzgl9-worker-6svd6   Succeeded
cluster-fileintegrity-ocp47-jzgl9-worker-jt5ng   ocp47-jzgl9-worker-jt5ng   Succeeded
cluster-fileintegrity-ocp47-jzgl9-worker-wrddr   ocp47-jzgl9-worker-wrddr   Succeeded
cluster-fileintegrity-ocp47-jzgl9-worker-x5btd   ocp47-jzgl9-worker-x5btd   Succeeded
```

The important information here is the STATUS. You will see one of three states:

- Succeeded - No files have been changed
- Failed - Files have been changed
- Errored - The AIDE application had an internal failure

## Reviewing the AIDE configuration

We can take a look at the AIDE configuration file by reviewing the ConfigMap used by the operator:

oc describe cm/cluster-fileintegrity -n openshift-file-integrity

```shell
aide.conf:
----
@@define DBDIR /hostroot/etc/kubernetes
@@define LOGDIR /hostroot/etc/kubernetes
database=file:@@{DBDIR}/aide.db.gz
database_out=file:@@{DBDIR}/aide.db.gz
gzip_dbout=yes
verbose=5
report_url=file:@@{LOGDIR}/aide.log
report_url=stdout
PERMS = p+u+g+acl+selinux+xattrs
CONTENT_EX = sha512+ftype+p+u+g+n+acl+selinux+xattrs

/hostroot/boot/        CONTENT_EX
/hostroot/root/\..* PERMS
/hostroot/root/   CONTENT_EX
!/hostroot/usr/src/
!/hostroot/usr/tmp/

/hostroot/usr/    CONTENT_EX

# OpenShift specific excludes
!/hostroot/opt/
!/hostroot/var
!/hostroot/etc/NetworkManager/system-connections/
!/hostroot/etc/mtab$
!/hostroot/etc/.*~
!/hostroot/etc/kubernetes/static-pod-resources
!/hostroot/etc/kubernetes/aide.*
!/hostroot/etc/kubernetes/manifests
!/hostroot/etc/docker/certs.d
!/hostroot/etc/selinux/targeted
!/hostroot/etc/openvswitch/conf.db

# Catch everything else in /etc
/hostroot/etc/    CONTENT_EX
```

## Simulating a failure

We will edit a file to simulate a failure. Select one of the nodes that is being monitored and connect to it using the `oc debug` command.

```shell
$ oc debug node/<node>
echo "# someone was here" >> /host/etc/resolv.conf
chmod 666 /host/etc/resolv.conf
exit
```

You will now need to wait for the tool to find the change. The AIDE scan can be resource intensive. By default, the scan will occur every 15 minutes.

```shell
$ oc get fileintegritynodestatuses -n openshift-file-integrity
 NAME                                            NODE                       STATUS
cluster-fileintegrity-ocp47-jzgl9-master-0       ocp47-jzgl9-master-0       Succeeded
cluster-fileintegrity-ocp47-jzgl9-master-1       ocp47-jzgl9-master-1       Succeeded
cluster-fileintegrity-ocp47-jzgl9-master-2       ocp47-jzgl9-master-2       Succeeded
cluster-fileintegrity-ocp47-jzgl9-worker-6svd6   ocp47-jzgl9-worker-6svd6   Failed
cluster-fileintegrity-ocp47-jzgl9-worker-jt5ng   ocp47-jzgl9-worker-jt5ng   Succeeded
cluster-fileintegrity-ocp47-jzgl9-worker-wrddr   ocp47-jzgl9-worker-wrddr   Succeeded
cluster-fileintegrity-ocp47-jzgl9-worker-x5btd   ocp47-jzgl9-worker-x5btd   Succeeded
```

Note that in the output above you can now see that the node we edited shows that the STATUS is "Failed". Let's check and see what the results are.

```shell
$ oc get fileintegritynodestatuses.fileintegrity.openshift.io/cluster-fileintegrity-ocp47-jzgl9-worker-6svd6 -n openshift-file-integrity -ojsonpath='{.results}' | jq -r
[
  {
    "condition": "Succeeded",
    "lastProbeTime": "2021-06-09T14:57:56Z"
  },
  {
    "condition": "Failed",
    "filesChanged": 1,
    "lastProbeTime": "2021-06-09T15:28:20Z",
    "resultConfigMapName": "aide-ds-cluster-fileintegrity-ocp47-jzgl9-worker-6svd6-failed",
    "resultConfigMapNamespace": "openshift-file-integrity"
  }
]
```

The detailed information we are looking for is in the ConfigMapName listed above. So lets get the details:

```shell
$ oc describe cm aide-ds-cluster-fileintegrity-ocp47-jzgl9-worker-6svd6-failed -n openshift-file-integrity
Name:         aide-ds-cluster-fileintegrity-ocp47-jzgl9-worker-6svd6-failed
Namespace:    openshift-file-integrity
...
====
integritylog:
----
Start timestamp: 2021-06-09 15:43:20 +0000 (AIDE 0.16)
AIDE found differences between database and filesystem!!

Summary:
  Total number of entries:  33901
  Added entries:              0
  Removed entries:            0
  Changed entries:            1

---------------------------------------------------
Changed entries:
---------------------------------------------------

f   ...    .C... : /hostroot/etc/resolv.conf
End timestamp: 2021-06-09 15:43:32 +0000 (run time: 0m 12s)

Events:  <none>
```

With the information above we can now identify the file that was changed. You can then review the file and see what has changed, and deal with it as appropriate. Note that our change was a simple contents change, and ACL change, AIDE will also pick up changes for permissions, ownership, SELinux settings and much more. For a full list of things that it can detect see the man page for AIDE, or check here for a cached web version: [aide.conf(5)](https://linux.die.net/man/5/aide.conf)

## Reinitializing the AIDE database

Lets say that your configuration was changed, but it was done intentionally, and you do not want to be alerted to the change any more. What do you do to reset the AIDE database so that it knows this change was intentional. Using the oc command we will annotate the fileintegrity and mark it for "re-init", which will re-scan all the hosts that are managed by the fileintegrity instance, and update the database to have the new signatures.

```shell
$ oc annotate fileintegrities/cluster-fileintegrity file-integrity.openshift.io/re-init= -n openshift-file-integrity
```

## References

- [AIDE Home Page](https://aide.github.io/)
- [OpenShift File Integrity Operator Install](https://docs.openshift.com/container-platform/4.7/security/file_integrity_operator/file-integrity-operator-installation.html)