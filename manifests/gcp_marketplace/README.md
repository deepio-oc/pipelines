# Kubeflow Pipelines for GKE Marketplace

Kubeflow Pipelines can be installed using either of the following approaches:

* [Using the Google Cloud Platform Console](#using-install-platform-console)

* [Using the command line](#using-install-command-line)


## <a name="using-install-platform-console"></a>Using the Google Cloud Platform Marketplace

Get up and running with a few clicks! Install this Kubeflow Pipelines app to a
Google Kubernetes Engine cluster using Google Cloud Marketplace. Follow the
[on-screen instructions](https://console.cloud.google.com/marketplace/details/google-cloud-ai-platform/kubeflow-pipelines) and [guide](https://github.com/kubeflow/pipelines/blob/master/manifests/gcp_marketplace/guide.md).


## <a name="using-install-command-line"></a>Using the command line

You can use [Google Cloud Shell](https://cloud.google.com/shell/) or a local
workstation to complete these steps.


[![Open in Cloud Shell](http://gstatic.com/cloudssh/images/open-btn.svg)](https://console.cloud.google.com/cloudshell/editor?cloudshell_git_repo=https://github.com/kubeflow/pipelines&cloudshell_open_in_editor=README.md&cloudshell_working_dir=manifests/gcp_marketplace)


### Prerequisites

#### Set up command-line tools

You'll need the following tools in your development environment. If you are
using Cloud Shell, these tools are installed in your environment by default.

-   [gcloud](https://cloud.google.com/sdk/gcloud/)
-   [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/)
-   [docker](https://docs.docker.com/install/)
-   [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

Configure `gcloud` as a Docker credential helper:

```shell
gcloud auth configure-docker
```

You can install Kubeflow Pipelines in an existing GKE cluster or create a new GKE cluster. 

* If you want to **create** a new Google GKE cluster, follow the instructions from the section [Create a GKE cluster](#create-gke-cluster) onwards.

* If you have an **existing** GKE cluster, ensure that the cluster nodes have a minimum 3 node cluster with each node having a minimum of 2 vCPU and running k8s version 1.9 and follow the instructions from section [Install the application resource definition](#install-application-resource-definition) onwards.

#### <a name="create-gke-cluster"></a>Create a GKE cluster

Kubeflow Pipelines requires a minimum 3 node cluster with each node having a minimum of 2 vCPU and k8s version 1.9. Available machine types can be seen [here](https://cloud.google.com/compute/docs/machine-types).

Create a new cluster from the command line:

```shell
export CLUSTER=kubeflow-pipelines-cluster
export ZONE=us-west1-a
export MACHINE_TYPE=n1-standard-2

gcloud container clusters create "$CLUSTER" --zone "$ZONE" --machine-type "$MACHINE_TYPE"
```

Configure `kubectl` to connect to the new cluster:

```shell
gcloud container clusters get-credentials "$CLUSTER" --zone "$ZONE"
```

#### <a name="install-application-resource-definition"></a>Install the application resource definition

An application resource is a collection of individual Kubernetes components,
such as services, stateful sets, deployments, and so on, that you can manage as a group.

To set up your cluster to understand application resources, run the following command:

```shell
kubectl apply -f "https://raw.githubusercontent.com/GoogleCloudPlatform/marketplace-k8s-app-tools/master/crd/app-crd.yaml"
```

You need to run this command once.

The application resource is defined by the Kubernetes
[SIG-apps](https://github.com/kubernetes/community/tree/master/sig-apps)
community. The source code can be found on
[github.com/kubernetes-sigs/application](https://github.com/kubernetes-sigs/application).

#### Prerequisites for using Role-Based Access Control
You must grant your user the ability to create roles in Kubernetes by running the following command. 

```shell
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole cluster-admin \
  --user $(gcloud config get-value account)
```

You need to run this command once.


### Install the Application

#### Run installer script
Set your application instance name and the Kubernetes namespace to deploy:

```shell
# set the application instance name
export APP_INSTANCE_NAME=kubeflow-pipelines-app

# set the Kubernetes namespace the application was originally installed
export NAMESPACE=<namespace>
```

Creat the namespace
```shell
kubectl create namespace $NAMESPACE
```

Download token for your service account which you want to use for calling GCP APIs from the pipelines.
```shell
gcloud iam service-accounts keys create application_default_credentials.json --iam-account [your-service-account]
export SERVICE_ACCOUNT_TOKEN="$(cat application_default_credentials.json | base64 -w 0)"
```

Follow the [instruction](https://github.com/GoogleCloudPlatform/marketplace-k8s-app-tools/blob/master/docs/tool-prerequisites.md#tool-prerequisites) and install mpdev
TODO: The official mpdev won't work because it doesn't have permission to deploy CRD. The latest unofficial build will have right permission. Remove following instruction when change is in prod.
```
BIN_FILE="$HOME/bin/mpdev"
docker run gcr.io/cloud-marketplace-staging/marketplace-k8s-app-tools/k8s/dev:unreleased-pr396 cat /scripts/dev > "$BIN_FILE"
chmod +x "$BIN_FILE"
export MARKETPLACE_TOOLS_TAG=unreleased-pr396
export MARKETPLACE_TOOLS_IMAGE=gcr.io/cloud-marketplace-staging/marketplace-k8s-app-tools/k8s/dev
```

Run the install script

```shell
mpdev scripts/install  --deployer=gcr.io/ml-pipeline/google/pipelines/deployer:0.2   --parameters='{"name": "'$APP_INSTANCE_NAME'", "namespace": "'$NAMESPACE'", "serviceAccountCredential": "'$SERVICE_ACCOUNT_TOKEN'"}'

```

Or if using CloudSQL and GCS,
```
mpdev scripts/install  --deployer=gcr.io/ml-pipeline/google/pipelines/deployer:0.2   --parameters='{"name": "'$APP_INSTANCE_NAME'", "namespace": "'$NAMESPACE'", "serviceAccountCredential": "'$SERVICE_ACCOUNT_TOKEN'", "managedstorage.enabled": true, "managedstorage.cloudsqlInstanceConnectionName": "[your-name]", "managedstorage.dbPassword": "[your-pwd]"}'
```

Watch the deployment come up with

```shell
kubectl get pods -n $NAMESPACE --watch
```

Get public endpoint
```shell
kubectl describe configmap inverse-proxy-config -n $NAMESPACE | grep googleusercontent.com
 
```

# Delete the Application

```shell
kubectl delete applications -n $NAMESPACE $APP_INSTANCE_NAME
```
