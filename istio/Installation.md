# Installing Istio into an existing OCP 3.10 Installation

This document describes the steps for installing the Istio Tech Preview release into an existing installation of OCP 3.10.

**Table of Contents**

- [Preparing the OCP 3.10 Installation](#preparing-the-ocp-310-installation)
   - [Updating the Master](#updating-the-master)
   - [Updating the Nodes](#updating-the-nodes)
- [Installing the Istio Operator](#installing-the-istio-operator)
- [Verifying Installation](#verifying-installation)
- [Deploying the Istio Control Plane](#deploying-the-istio-control-plane)
- [Verifying the Istio Control Plane](#verifying-the-istio-control-plane)
- [Removing Istio](#removing-istio)
- [Removing the Istio Operator](#removing-the-istio-operator)

## Preparing the OCP 3.10 Installation

Before Istio can be installed into an OCP 3.10 installation it is necessary to make a number of changes to the master configuration and each of the schedulable nodes.  These changes will enable features required within Istio and also ensure Elasticsearch will function correctly.

### Updating the Master

To enable the automatic injection of the Istio sidecar we first need to modify the master configuration on each master to include support for webhooks and signing of Certificate Signing Requests (CSRs).

Make the following changes on each master within your OCP 3.10 installation.

- Change to the directory containing the master configuration file (master-config.yaml)
- Create a file named master-config.patch with the following contents (also in `master-config.patch`)

```
admissionConfig:
  pluginConfig:
    MutatingAdmissionWebhook:
      configuration:
        apiVersion: v1
        disable: false
        kind: DefaultAdmissionConfig
    ValidatingAdmissionWebhook:
      configuration:
        apiVersion: v1
        disable: false
        kind: DefaultAdmissionConfig
```

- Within the same directory issue the following commands:

```
cp -p master-config.yaml master-config.yaml.prepatch
oc ex config patch master-config.yaml.prepatch -p "$(cat master-config.patch)" > master-config.yaml
systemctl restart atomic-openshift-*
```

### Updating the Nodes

In order to run the Elasticsearch application it is necessary to make a change to the kernel configuration on each node, this change will be handled through the `sysctl` service.

Make the following changes on each node within your OCP 3.10 installation

- Create a file named `/etc/sysctl.d/99-elasticsearch.conf` with the following contents:

`vm.max_map_count = 262144`

- Execute the following command:

```
sysctl vm.max_map_count=262144
```

## Installing the Istio Operator

The Maistra installation process introduces a Kubernetes operator to manage the installation of the Istio control plane within the istio-system namespace.  This operator defines and monitors a custom resource related to the deployment, update and deletion of the Istio control plane.

The following steps will install the Maistra operator into an existing OCP 3.10 installation, these can be executed from any host with access to the cluster.  Please ensure you are logged in as a cluster admin before executing the following

For community images run

```
oc new-project istio-operator
oc new-app -f istio_community_operator_template.yaml --param=OPENSHIFT_ISTIO_MASTER_PUBLIC_URL=<master public url>
```

For product images run

```
oc new-project istio-operator
oc new-app -f istio_product_operator_template.yaml --param=OPENSHIFT_ISTIO_MASTER_PUBLIC_URL=<master public url>
```

## Verifying Installation

The above instructions will create a new deployment within the istio-operator project, executing the operator responsible for managing the state of the Istio control plane through the custom resource.

To verify the operator is installed correctly, locate the pod using the following command

```
oc get pods -n istio-operator
```
Access the logs from the pod with the following command, replacing `<pod name>` with the name of the pod discovered above

```
oc logs -n istio-operator <pod name>
```

and look for output similar to the following

```
time="2018-08-14T20:00:18Z" level=info msg="Go Version: go1.9.7"
time="2018-08-14T20:00:18Z" level=info msg="Go OS/Arch: linux/amd64"
time="2018-08-14T20:00:18Z" level=info msg="operator-sdk Version: 0.0.5+git"
time="2018-08-14T20:00:18Z" level=info msg="Metrics service istio-operator created"
time="2018-08-14T20:00:18Z" level=info msg="Watching resource istio.openshift.com/v1alpha1, kind Installation, namespace istio-operator, resyncPeriod 0"
```

## Deploying the Istio Control Plane

In order to deploy the Istio Control Plane we need to deploy a custom resource such as the following example which demonstrates the configuration options supported by the operator.  The custom resource *must* be deployed into the `istio-operator` namespace and *must* be called `istio-installation`.

```
apiVersion: "istio.openshift.com/v1alpha1"
kind: "Installation"
metadata:
  name: "istio-installation"
spec:
  deployment_type: openshift
  istio:
    authentication: true
    community: false
    prefix: openshift-istio-tech-preview/
    version: 0.1.0
  jaeger:
    prefix: distributed-tracing-tech-preview/
    version: 1.6.0
    elasticsearch_memory: 1Gi
  launcher:
    openshift:
      user: user
      password: password
    github:
      username: username
      token: token
    catalog:
      filter: filter
      branch: branch
      repo: repo
```

The minimal custom resource required to install an Istio Control Plane is as follows

```
apiVersion: "istio.openshift.com/v1alpha1"
kind: "Installation"
metadata:
  name: "istio-installation"
```

## Verifying the Istio Control Plane

The operator will create the `istio-system` namespace and run the installer job, this job will set up the Istio control plane using Ansible playbooks.  The progress of the installation can be followed by either watching the pods or the log output from the `openshift-ansible-istio-installer-job` pod.

To watch the progress of the pods execute the following command:

```
oc get pods -n istio-system -w
```

Once the `openshift-ansible-istio-installer-job` has completed run `oc get pods -n istio-system` and verify you have state similar to the following"

```
NAME                                          READY     STATUS      RESTARTS   AGE
elasticsearch-0                               1/1       Running     0          41s
grafana-6d5c5477-b6twg                        1/1       Running     0          43s
istio-citadel-b9b8d7589-wzh57                 1/1       Running     0          1m
istio-egressgateway-7f987dc785-hfmnh          1/1       Running     0          1m
istio-galley-745db694bb-ngpnd                 1/1       Running     0          1m
istio-ingressgateway-5bd8ffd968-qw8l5         1/1       Running     0          1m
istio-pilot-cf76476d4-2fzjg                   1/2       Running     0          1m
istio-policy-7cd858cc78-n7lll                 2/2       Running     0          1m
istio-sidecar-injector-86c5d87f-qm64q         1/1       Running     0          1m
istio-statsd-prom-bridge-7f44bb5ddb-cv72w     1/1       Running     0          1m
istio-telemetry-f757b89c5-jlfrz               2/2       Running     0          1m
jaeger-agent-5cvh5                            1/1       Running     0          35s
jaeger-collector-d8b97d664-jt9fs              1/1       Running     0          35s
jaeger-query-7745b957bb-5nxgz                 1/1       Running     0          35s
openshift-ansible-istio-installer-job-j9xb8   0/1       Completed   0          1m
prometheus-84bd4b9796-z282c                   1/1       Running     0          1m
```

If you have also chosen to install the Farbic8 launcher you should monitor the containers within the devex project until the following state has been reached:

```
NAME                          READY     STATUS    RESTARTS   AGE
configmapcontroller-1-8rr6w   1/1       Running   0          1m
launcher-backend-2-2wg86      1/1       Running   0          1m
launcher-frontend-2-jxjsd     1/1       Running   0          1m
```

## Removing Istio

The following step will remove Istio from an existing installation, it can be executed from any host with access to the cluster.

```
oc delete -n istio-operator Installation istio-installation
```

## Removing the Istio Operator

The following steps will remove the Maistra operator from an existing OCP 3.10 installation, these can be executed from any host with access to the cluster.  Please ensure you are logged in as a cluster admin before executing the following

For community images run

```
oc process -f istio_community_operator_template.yaml | oc delete -f -
```

For product images run

```
oc process -f istio_product_operator_template.yaml | oc delete -f -
```
