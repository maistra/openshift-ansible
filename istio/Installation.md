# Installing Istio into an existing OCP 3.9 installation

This document describes the steps for installing the Istio Tech Preview release into an existing installation of OCP 3.9.

**Table of Contents**

- [Preparing the OCP 3.9 Installation](#preparing-the-ocp-39-installation)
   - [Updating the Master](#updating-the-master)
   - [Updating the Nodes](#updating-the-nodes)
- [Installing Istio](#installing-istio)
- [Verifying Installation](#verifying-installation)
- [Removing Istio](#removing-istio)

## Preparing the OCP 3.9 Installation

Before Istio can be installed into an OCP 3.9 installation it is necessary to make a number of changes to the master configuration and each of the schedulable nodes.  These changes will enable features required within Istio and also ensure Elasticsearch will function correctly.

### Updating the Master

To enable the automatic injection of the Istio sidecar we first need to modify the master configuration on each master to include support for webhooks and signing of Certificate Siging Requests (CSRs).

Make the following changes on each master within your OCP 3.9 installation.

- Change to the directory containing the master configuration file (master-config.yaml)
- create a file named master-config.patch with the following contents (also in `master-config.patch`)

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

- within the same directory issue the following commands

```
cp -p master-config.yaml master-config.yaml.prepatch
oc ex config patch master-config.yaml.prepatch -p "$(cat master-config.patch)" > master-config.yaml
systemctl restart atomic-openshift-master*
```

### Updating the Nodes
In order to run the Elasticsearch application it is necessary to make a change to the kernel configuration on each node, this change will be handled through the sysctl service.

Make the following changes on each node within your OCP 3.9 installation

- Create a file named /etc/sysctl.d/99-elasticsearch.conf with the following contents

`vm.max_map_count = 262144`

- execte the foolowing command

```
sysctl vm.max_map_count=262144
```

## Installing Istio

The following steps will install Istio into an existing OCP 3.9 installation, they can be executed from any host with access to the cluster

```
oc new-project istio-system
oc create sa openshift-ansible
oc adm policy add-scc-to-user privileged -z openshift-ansible
oc adm policy add-cluster-role-to-user cluster-admin -z openshift-ansible
oc new-app istio_installer_template.yaml --param=OPENSHIFT_ISTIO_MASTER_PUBLIC_URL=<master public url> --param=OPENSHIFT_ISTIO_KIALI_USERNAME=<username> --param=OPENSHIFT_ISTIO_KIALI_PASSWORD=<password>
```

## Verifying Installation
The above instructions will create a job within the istio-system project to install Istio using ansible playbooks, the progress of the installation can be followed by either watching the pods or the log output from the `openshift-ansible-istio-installer-job` pod.

To watch the progress of the pods execute the following command

```
oc get pods -n istio-system -w
```

Once the `openshift-ansible-istio-installer-job` has completed run `oc get pods -n istio-system` and verify you have state similar to the following

```
NAME                                        READY     STATUS      RESTARTS   AGE
elasticsearch-0                             1/1       Running     0          1m
grafana-6bb556d859-hslg4                    1/1       Running     0          1m
istio-citadel-5f59bd46f8-6f89t              1/1       Running     0          1m
istio-egressgateway-66c558586b-9rkr4        1/1       Running     0          1m
istio-galley-5d4b48cfb-tslrh                1/1       Running     0          1m
istio-ingress-58649fdc6b-v6vdv              1/1       Running     0          1m
istio-ingressgateway-6bbb647b64-wjnfc       1/1       Running     0          1m
istio-pilot-bf7d7fd97-9kfhl                 2/2       Running     0          1m
istio-policy-8677b55fd4-ppsjk               2/2       Running     0          1m
istio-sidecar-injector-7c4b5fc547-qqsl5     1/1       Running     0          1m
istio-statsd-prom-bridge-6dbb7dcc7f-h6zg8   1/1       Running     0          1m
istio-telemetry-8c8f9c5c6-ljs75             2/2       Running     0          1m
jaeger-agent-69c7f                          1/1       Running     0          1m
jaeger-collector-68fd846775-m5lkf           1/1       Running     0          1m
jaeger-query-58f4655965-nkhg4               1/1       Running     0          1m
kiali-54f98bf9d5-b6wz4                      1/1       Running     0          1m
openshift-ansible-istio-job-5mlvs           0/1       Completed   0          2m
prometheus-586d95b8d9-dmwx5                 1/1       Running     0          1m
```

If you have also chosen to install the Farbic8 launcher then you should monitor the containers within the devex project until the following state has been reached

```
NAME                          READY     STATUS    RESTARTS   AGE
configmapcontroller-1-8rr6w   1/1       Running   0          1m
launcher-backend-2-2wg86      1/1       Running   0          1m
launcher-frontend-2-jxjsd     1/1       Running   0          1m
```

## Removing Istio

The following step will remove Istio from an existing installation, it can be executed from any host with access to the cluster.

```
oc new-app istio_removal_template.yaml
```
