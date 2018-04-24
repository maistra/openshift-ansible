# Installing Istio into an existing OCP 3.9 installation

This document describes the steps for installing the Istio Tech Preview release into an existing installation of OCP 3.9.

**Table of Contents**

- [Preparing the OCP 3.9 Installation](#preparing-the-ocp-39-installation)
   - [Updating the Master](#updating-the-master)
   - [Updating the Nodes](#updating-the-nodes)
   - [Restart OCP 3.9 installation](#restart-ocp-39-installation)
- [Installing Istio](#installing-istio)
- [Verifying Installation](#verifying-installation)


## Preparing the OCP 3.9 Installation

Before Istio can be installed into an OCP 3.9 installation it is necessary to make a number of changes to the master configuration and each of the schedulable nodes.  These changes will enable features required within Istio and also ensure Elasticsearch will function correctly.

### Updating the Master

To enable the automatic injection of the Istio sidecar we first need to modify the master configuration to include support for webhooks and signing of Certificate Siging Requests (CSRs).

Edit the master configuration (master-config.yaml) and make the following changes

  - locate the admissionConfig section and add the following
(indentation should be the same as GenericAdmissionWebhook)

```
    MutatingAdmissionWebhook:
      configuration:
        apiVersion: v1
        disable: false
        kind: DefaultAdmissionConfig
```

  - locate the kubernetesMasterConfig and add the following
(indentation should be the same as admissionConfig)

```
  controllerArguments:
    cluster-signing-cert-file:
    - ca.crt
    cluster-signing-key-file:
    - ca.key
```

### Updating the Nodes
In order to run the Elasticsearch application it is necessary to make a change to the kernel configuration on each node, this change will be handled through the sysctl service.

Make the following changes on each node within your OCP 3.9 installation

- Create a file named /etc/sysctl.d/99-elasticsearch.conf with the following contents

`vm.max_map_count = 262144`

### Restart OCP 3.9 installation

Once the above changes have been applied restart your master/nodes.

## Installing Istio

The following steps will install Istio into an existing OCP 3.9 installation

- Create an inventory file named /var/lib/origin/openshift.local.config/istio.inventory, with the following contents (also in `istio.inventory`)

```
[OSEv3:children]
masters

[OSEv3:vars]
openshift_release=v3.9.0
openshift_istio_jaeger_image_version=0.6

# origin or openshift-enterprise
openshift_deployment_type=origin

# The following are needed only if you are not using the default settings
#openshift_istio_image_prefix=openshiftistio/
#openshift_istio_image_version=0.7.1
#openshift_istio_namespace=istio-system

# true if the upstream community should be installed
openshift_istio_install_community=false

# true if mTLS should be enabled
openshift_istio_install_auth=false

# true if the fabric8 launcher should be installed
openshift_istio_install_launcher=false

# The public endpoint for accessing the master
openshift_istio_master_public_url=https://127.0.0.1:8443

# Uncomment the following to override the name of the openshift user used by the launcher, this defaults to developer
# launcher_openshift_user=
# Uncomment the following to override the password of the openshift user used by the launcher, this defaults to developer
# launcher_openshift_pwd=
# Uncomment the following to override the GitHub username used by the launcher, this is empty by default
# launcher_github_username=
# Uncomment the following to override the GitHub token used by the launcher, this is empty by default
# launcher_github_token=
# Uncomment the following to override the GitHub catalog repository used by the launcher, this defaults to https://github.com/snowdrop/launcher-booster-catalog.git
# launcher_catalog_git_repo=
# Uncomment the following to override the GitHub catalog branch used by the launcher, this defaults to istio
# launcher_catalog_git_branch=

[masters]
127.0.0.1 ansible_connection=local
```

- Modify the contents of the istio.inventory file to reflect your master URL configuration and whether mTLS should be enabled
- Upload the `istio_installer_job.yaml` template to the master node and execute the following commands

```
oc login —config=/etc/origin/master/admin.kubeconfig
oc new-project istio-system
oc create sa openshift-ansible
oc adm policy add-scc-to-user privileged -z openshift-ansible
oc create -f istio_installer_job.yaml
```

## Verifying Installation
The above instructions will create a job within the istio-system project to install Istio using ansible playbooks, the progress of the installation can be followed by either watching the pods or the log output from the `openshift-ansible-istio-job` pod.

To watch the progress of the pods execute the following command

```
oc get pods -n istio-system -w
```

Once the `openshift-ansible-istio-job` has completed run `oc get pods -n istio-system` and verify you have state similar to the following

```
NAME                                      READY     STATUS      RESTARTS   AGE
elasticsearch-0                           1/1       Running     0          18s
elasticsearch-1                           1/1       Running     0          3s
grafana-6f4fd4986f-tzkzl                  1/1       Running     0          25s
istio-ca-ddb878d84-mknp6                  1/1       Running     0          43s
istio-ingress-76b5496c58-4v9s8            1/1       Running     0          43s
istio-mixer-56f49dc667-4nl2v              3/3       Running     0          44s
istio-mixer-validator-65c7fccc64-rj8dd    1/1       Running     0          43s
istio-pilot-76dd785958-tb2fn              2/2       Running     0          44s
istio-sidecar-injector-599d8c454c-7pv7b   1/1       Running     0          35s
jaeger-agent-rfkkk                        1/1       Running     0          1s
jaeger-collector-b86c6bf8d-p5g7p          1/1       Running     0          2s
jaeger-query-6c8c85454-kzz82              1/1       Running     0          2s
openshift-ansible-istio-job-zg6nv         0/1       Completed   0          1m
prometheus-cf8456855-n4qd9                1/1       Running     0          24s
```

If you have also chosen to install the Farbic8 launcher then you should monitor the containers within the devex project until the following state has been reached

```
NAME                          READY     STATUS    RESTARTS   AGE
configmapcontroller-1-8rr6w   1/1       Running   0          1m
launcher-backend-2-2wg86      1/1       Running   0          1m
launcher-frontend-2-jxjsd     1/1       Running   0          1m
```



