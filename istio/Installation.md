# Installing Istio into an existing OCP 3.11 Installation

This document describes the steps for installing the Istio Tech Preview release into an existing installation of OCP 3.11.

**Table of Contents**

- [Preparing the OCP 3.11 Installation](#preparing-the-ocp-311-installation)
   - [Updating the Master](#updating-the-master)
   - [Updating the Nodes](#updating-the-nodes)
- [Installing the Istio Operator](#installing-the-istio-operator)
- [Verifying Installation](#verifying-installation)
- [Deploying the Istio Control Plane](#deploying-the-istio-control-plane)
- [Verifying the Istio Control Plane](#verifying-the-istio-control-plane)
- [Removing Istio](#removing-istio)
- [Removing the Istio Operator](#removing-the-istio-operator)

## Preparing the OCP 3.11 Installation

Before Istio can be installed into an OCP 3.11 installation it is necessary to make a number of changes to the master configuration and each of the schedulable nodes.  These changes will enable features required within Istio and also ensure Elasticsearch will function correctly.

### Updating the Master

To enable the automatic injection of the Istio sidecar we first need to modify the master configuration on each master to include support for webhooks and signing of Certificate Signing Requests (CSRs).

Make the following changes on each master within your OCP 3.11 installation.

- Change to the directory containing the master configuration file (master-config.yaml)
- Create a file named master-config.patch with the following contents (also in `master-config.patch`)

```
admissionConfig:
  pluginConfig:
    MutatingAdmissionWebhook:
      configuration:
        apiVersion: apiserver.config.k8s.io/v1alpha1
        kubeConfigFile: /dev/null
        kind: WebhookAdmission
    ValidatingAdmissionWebhook:
      configuration:
        apiVersion: apiserver.config.k8s.io/v1alpha1
        kubeConfigFile: /dev/null
        kind: WebhookAdmission
```

- Within the same directory issue the following commands:

```
cp -p master-config.yaml master-config.yaml.prepatch
oc ex config patch master-config.yaml.prepatch -p "$(cat master-config.patch)" > master-config.yaml
/usr/local/bin/master-restart api && /usr/local/bin/master-restart controllers
```

### Updating the Nodes

In order to run the Elasticsearch application it is necessary to make a change to the kernel configuration on each node, this change will be handled through the `sysctl` service.

Make the following changes on each node within your OCP 3.11 installation

- Create a file named `/etc/sysctl.d/99-elasticsearch.conf` with the following contents:

`vm.max_map_count = 262144`

- Execute the following command:

```
sysctl vm.max_map_count=262144
```

## Installing the Istio Operator

The Maistra installation process introduces a Kubernetes operator to manage the installation of the Istio control plane within the istio-system namespace.  This operator defines and monitors a custom resource related to the deployment, update and deletion of the Istio control plane.

The following steps will install the Maistra operator into an existing OCP 3.11 installation, these can be executed from any host with access to the cluster.  Please ensure you are logged in as a cluster admin before executing the following

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

To verify the operator is installed correctly, access the logs from the operator pod using the following command

```
oc logs -n istio-operator $(oc -n istio-operator get pods -l name=istio-operator --output=jsonpath={.items..metadata.name})
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
    version: 0.4.0
  jaeger:
    prefix: distributed-tracing-tech-preview/
    version: 1.7.0
    elasticsearch_memory: 1Gi
  kiali:
    username: username
    password: password
    prefix: kiali/
    version: v0.9.1
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
elasticsearch-0                               1/1       Running     0          32s
grafana-6d5c5477-q8pd7                        1/1       Running     0          37s
istio-citadel-5c94d7584f-jtjtr                1/1       Running     0          1m
istio-egressgateway-d67b6466-lq57z            1/1       Running     0          1m
istio-galley-d4c9cdcf6-g29cc                  1/1       Running     0          1m
istio-ingressgateway-7fd59bd49c-4vfwt         1/1       Running     0          1m
istio-pilot-5c988db7dd-2p746                  2/2       Running     0          1m
istio-policy-576bd648fd-mtcln                 2/2       Running     0          1m
istio-sidecar-injector-6695744d96-kqd9q       1/1       Running     0          1m
istio-statsd-prom-bridge-7f44bb5ddb-vvtmm     1/1       Running     0          1m
istio-telemetry-c777d7595-w6dps               2/2       Running     0          1m
jaeger-agent-7s4vb                            1/1       Running     0          26s
jaeger-collector-588589c867-6bhn4             1/1       Running     1          28s
jaeger-query-5fb87fb4d9-m6kb4                 1/1       Running     1          26s
kiali-779bcc566f-qqt65                        1/1       Running     0          16s
openshift-ansible-istio-installer-job-hmn8t   0/1       Completed   0          2m
prometheus-84bd4b9796-xbrgx                   1/1       Running     0          1m
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

The following steps will remove the Maistra operator from an existing OCP 3.11 installation, these can be executed from any host with access to the cluster.  Please ensure you are logged in as a cluster admin before executing the following

For community images run

```
oc process -f istio_community_operator_template.yaml | oc delete -f -
```

For product images run

```
oc process -f istio_product_operator_template.yaml | oc delete -f -
```
