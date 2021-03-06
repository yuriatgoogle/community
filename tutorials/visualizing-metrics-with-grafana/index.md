# Visualizing GKE and Istio Metrics with Grafana

## Introduction

Stackdriver is a full-featured operations and observability toolkit that includes capabilities specifically targeted at Kubernetes operators, including a rich set of [features](/monitoring/kubernetes-engine/){: track-type="tutorial" track-name="internalLink" track-metadata-position="body" } for Kubernetes observability.  However, users with experience in this area tend to have familiarity with the open-source observability toolkit that includes the ELK [stack](https://www.elastic.co/what-is/elk-stack){: target="external" track-type="tutorial" track-name="externalLink" track-metadata-position="body" } and [Prometheus](prometheus.io){: target="external" track-type="tutorial" track-name="externalLink" track-metadata-position="body" } and often prefer to use [Grafana](grafana.com){: target="external" track-type="tutorial" track-name="externalLink" track-metadata-position="body" } as their visualization layer.

In this tutorial, you will learn how to deploy a sample application and Grafana on a GKE cluster and configure Stackdriver as a backend for Grafana to create dashboards displaying key observability details about the cluster and application running on it.  The sample application is provided [here](https://github.com/GoogleCloudPlatform/microservices-demo){: target="github" track-type="tutorial" track-name="gitHubLink" track-metadata-position="body" }.

The architecture you will be deploying in this tutorial is as follows:

![image](1-architecture.png)

## Costs

This tutorial uses billable components of Google Cloud Platform, including:

-   Kubernetes Engine
-   Stackdriver

Use the [Pricing Calculator](/products/calculator){: track-type="tutorial" track-name="pricingCalculator" track-metadata-position="body" } to generate a cost estimate based on your projected usage.

## Before you begin

1.  Select or create a GCP project.

[GO TO THE MANAGE RESOURCES PAGE]({{console_url}}cloud-resource-manager){: target="console" track-type="tutorial" track-name="consoleLink" track-metadata-position="body" }

1.  Enable billing for your project if you have not selected a billing account during project creation.

[ENABLE BILLING](https://support.google.com/cloud/answer/6293499#enable-billing){: target="support" track-type="tutorial" track-name="supportLink" track-metadata-position="body" }\
When you finish this tutorial, you can avoid continued billing by deleting the resources you created. See [Cleaning up](https://docs.google.com/document/d/1vaelwoytZYoZ5WvyzwdksRTysig0bncoThU9I-T0In0/edit#heading=h.mlrdlgcohh7k){: target="external" track-type="tutorial" track-name="externalLink" track-metadata-position="body" } for more detail.

## Set up your environment

In this section, you set up your environment with the tools you'll be using throughout this tutorial.  You run all the terminal commands in this tutorial from Cloud Shell.

1.  Open Cloud Shell:

> [OPEN CLOUD SHELL]({{console_url}}?cloudshell=true){: target="console" track-type="tutorial" track-name="consoleLink" track-metadata-position="body" }	

1.  Set environment variables:

```
export PROJECT_ID=$(gcloud config list --format 'value(core.project)' 2>/dev/null)
```

1.  Enable the relevant APIs:

```
gcloud services enable \
cloudshell.googleapis.com \
cloudbuild.googleapis.com \
containerregistry.googleapis.com \
container.googleapis.com \
cloudtrace.googleapis.com
```

1.  Download the required files for this tutorial by cloning the sample application git repo.  Make the repo folder your `$WORKDIR` from which you do all the tasks related to this tutorial.  This way you can delete the folder when finished:

```
cd $HOME
git clone https://github.com/GoogleCloudPlatform/microservices-demo
cd $HOME/microservices-demo
WORKDIR=$(pwd)
```

### Install tools

1.  Install [kubectx and kubens](https://github.com/ahmetb/kubectx){: target="github" track-type="tutorial" track-name="gitHubLink" track-metadata-position="body" }.  These tools make it easier to work with multiple Kubernetes clusters, contexts, and namespaces

	`git clone https://github.com/ahmetb/kubectx $WORKDIR/kubectx`

```
export PATH=$PATH:$WORKDIR/kubectx
```

## Deploy application on GKE cluster

In this section, you create a GKE cluster with the Istio on GKE Addon and Stackdriver and deploy the sample application on it.

1.  In Cloud Shell, set the environment variables to be used for cluster creation:

```
#cluster name
export CLUSTER=monitoring-cluster 	

#zone for the cluster
export ZONE=us-central1-a		

#namespace for the application
export APP_NS=hipstershop		

#namespace for Grafana
export MONITORING_NS=grafana		
```

1.  In Cloud Shell, issue this command to create the GKE cluster:

```
gcloud beta container clusters create $CLUSTER \
--zone=$ZONE \
--num-nodes=6 \
--cluster-version=latest \
--enable-stackdriver-kubernetes \
--addons=Istio \
--istio-config=auth=MTLS_PERMISSIVE
```

> **Note** that you are using the PERMISSIVE setting for MTLS configuration for the sake of simplicity. We recommend reviewing the appropriate Istio [documentation](https://istio.io/docs/concepts/security/#mutual-tls-authentication){: target="external" track-type="tutorial" track-name="externalLink" track-metadata-position="body" } to choose the appropriate policy for your deployment.

1.  Issue this command to get cluster credentials:

```
gcloud container clusters get-credentials $CLUSTER --zone=$ZONE
```

1.  Issue these commands to set kubectx context and switch to it:

```
kubectx cluster=gke_${PROJECT_ID}_${ZONE}_${CLUSTER}
kubectx cluster
```

1.  Issue this command to create a dedicated namespace for your application and switch to it:

```
kubectl create namespace $APP_NS
kubens $APP_NS
```

1.  Label the namespace to enable automatic Istio sidecar injection:

```
kubectl label namespace $APP_NS istio-injection=enabled
```

1.  Apply the Istio manifests:

```
kubectl apply -f $WORKDIR/istio-manifests
```

1.  Deploy the sample application:

```
kubectl apply -f $WORKDIR/release/kubernetes-manifests.yaml
```

1.  Confirm that all components of the application have been deployed correctly:

```
kubectl get pods -n $APP_NS

Output is similar to this:
NAME                                     READY   STATUS     RESTARTS   AGE
adservice-5d9dc7989b-8t2ql               0/2     Pending    0          78s
cartservice-7555f749f-dw8mq              1/2     Running    2          80s
checkoutservice-b96bf455b-2wx6n          2/2     Running    0          83s
currencyservice-7f8658c69d-cnn4v         2/2     Running    0          80s
emailservice-6658cb8f7b-tch6p            2/2     Running    0          83s
frontend-65c5cc876c-pnk2l                2/2     Running    0          82s
loadgenerator-778c8489d6-vflz2           0/2     Init:0/2   0          80s
paymentservice-65bcb767c6-5w7p4          2/2     Running    0          81s
productcatalogservice-6c4f68859b-848hg   2/2     Running    0          81s
recommendationservice-5f844c876-m4p4j    2/2     Running    0          82s
redis-cart-65bf66b8fd-hclhm              2/2     Running    0          79s
shippingservice-55bc4768dd-8fgtv         2/2     Running    0          79s
```

> Note that it may take some time for all pods to switch to the 'running' state.

## Deploy Grafana

In this section, you deploy Grafana in a dedicated namespace in your cluster using a Helm chart and template.  [Helm](helm.sh){: target="external" track-type="tutorial" track-name="externalLink" track-metadata-position="body" } is an open-source package manager for Kubernetes.

### Install Grafana

1.  Update the local Helm repository:

```
helm repo update
```

1.  Download Grafana:

```
helm fetch stable/grafana --untar
```

1.  Create a namespace dedicated to Grafana:

```
kubectl create ns $MONITORING_NS
```

1.  Use the Helm chart to create the .yaml file:

```
helm template grafana --namespace $MONITORING_NS --name grafana > $WORKDIR/grafana.yaml
```

1.  Deploy Grafana using the file:

```
kubectl apply -f $WORKDIR/grafana.yaml -n $MONITORING_NS
```

1.  Verify the installation:

```
kubectl get pods -n $MONITORING_NS

Output is similar to this:
NAME                      READY   STATUS    RESTARTS   AGE
grafana-8cf6ddd7b-d5srh   1/1     Running   0          23s
grafana-test              0/1     Error     0          23s
```

### Connect to Grafana

1.  Get the Grafana password and copy the output:

```
kubectl get secret \
    --namespace $MONITORING_NS grafana \
    -o jsonpath="{.data.admin-password}" \
    | base64 --decode ; echo
```

1.  Capture the name of the Grafana pod as a variable:

```
export GRAFANA_POD=$(kubectl get pods --namespace $MONITORING_NS -l "app=grafana,release=grafana" -o jsonpath="{.items[0].metadata.name}")
```

1.  Use port-forwarding to enable access to the Grafana UI:

```
kubectl port-forward $GRAFANA_POD 3000 -n $MONITORING_NS
```

1.  Use the web preview functionality in Cloud Shell to access the UI after changing the port to 3000:

![image](2-preview.png)

1.  At the Grafana login screen, enter **admin** as the username and paste in the password from step 8 above to access Grafana.

## Configure data source and create dashboards

In this section, you configure Grafana to use Stackdriver as the data source and create dashboards that will be used to visualize the health and status of your application.

### Configure Stackdriver data source

1.  In the Grafana UI, click **Add data source.**

1.  Click **Stackdriver.**

1.  Switch **Authentication Type **to** Default GCE Service Account**. Note that this works because Grafana is running on a GKE cluster with default access scopes configured.

1.  Click **Save and Test.**

### Create the dashboard

In this section, you create a dashboard focusing on the [golden signals](https://landing.google.com/sre/sre-book/chapters/monitoring-distributed-systems/){: target="external" track-type="tutorial" track-name="externalLink" track-metadata-position="body" } of monitoring - request rates, errors, and latencies.

#### Create request rates view

1.  Mouse over the + on the left side and select **Create -> Dashboard:**

![image](3-create.png)

1.  Click **Add Query.**
1.  From the **Service** dropdown menu**, **select** Istio**.
1.  From the **Metric** dropdown, select Server Request Count.
1.  Click the + next to Group By and select metric.label.destination_service_name.
1.  From the Aggregation menu, select Sum.
1.  On the left side, click on Visualization.
1.  Under **Axes** -> **Left Y**, click on the **Units** menu and select **Throughput -> requests/sec (rps)**.
1.  On the left side, click on General:

![image](4-query.png)

1.  In the Panel Title field, enter **Request Rates by Service.**
1.  Click the Save Dashboard button at the top right to save your work.
1.  Click the left arrow at the top left to go back.
1.  Click the Dashboard Settings button at the top right.
1.  Under **General**, in the **Name** field, enter **"GKE Services Dashboard"** and click **Save.**

At this point, you should have a dashboard with a single view on it showing request rates for the services in your Istio service mesh.

![image](5-dashboard.png)

#### Create errors view

1.  At the top right, click the Add Panel button.
1.  Click Add Query.
1.  From the Service menu, select Istio.
1.  From the **Metric** dropdown, select Server Request Count.
1.  Click the + next to Group By and select metric.label.destination_service_name.
1.  Click the + next to Filter and select metric.label.response_code.
1.  Select != as the operator and 200 as the value to only count failed requests.

    **Note** that in this example, you're including 4xx errors in your count - often, users choose to exclude these, as they may be caused by issues on the client.

1.  From the Aggregation menu, select Sum.
1.  On the left side, click on Visualization.
1.  Under **Axes** -> **Left Y**, click on the **Units** menu and select **Throughput -> requests/sec (rps)**.
1.  On the left side, click on General:

![image](6-query.png)

1.  In the Panel Title field, enter **Errors by Service.**
1.  At the top right, click the Save Dashboard button.

At this point, your dashboard should contain two panels showing request rates and errors.

![image](7-dashboard.png)

#### Create latencies view

At this point, you have enough information to create the third view showing server latency.  Use the Server Response Latency metric from the Istio service, filter out requests where the metric.label.response_code!=200, and group by metric.label.destination_service_name.  Use 99th percentile as the aggregator (refer to this [publication](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/9c3491f50f97dd01a973173d09dd8590c688eba6.pdf){: target="external" track-type="tutorial" track-name="externalLink" track-metadata-position="body" } by Ben Traynor from Google [SRE](https://landing.google.com/sre/){: target="external" track-type="tutorial" track-name="externalLink" track-metadata-position="body" } to learn more about why 99th percentile latency is the right signal to measure). When you're done, name the panel, save the dashboard, and organize the panels as you like.  Your final result looks like this:

![image](8-dashboard.png)

## Conclusions and Summary

Congratulations on completing this tutorial!  You've now learned how to install Grafana using Helm templates, configure it to use Stackdriver as a data source, and create dashboards to visualize the health of a microservices application.  You have a fully functional monitoring system that will scale with your needs and can be further customized to evolve with your monitoring requirements.

## Cleaning up

To avoid incurring charges to your Google Cloud Platform account for the resources used in this tutorial:

### Delete the project

The easiest way to eliminate billing is to delete the project you created for the tutorial.6\
To delete the project:

1.  In the {{console_name_short}}, go to the Projects page.

    [GO TO THE PROJECTS PAGE]({{console_url}}iam-admin/projects){: target="console" track-type="tutorial" track-name="consoleLink" track-metadata-position="body" }

1.  In the project list, select the project you want to delete and click **Delete**.

![image](9-delete.png)

1.  In the dialog, type the project ID, and then click **Shut down** to delete the project.

## What's next

-   Try out other Google Cloud Platform features for yourself. Have a look at our [tutorials](/docs/tutorials){: track-type="tutorial" track-name="internalLink" track-metadata-position="body" }.