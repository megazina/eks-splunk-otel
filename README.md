# How to deploy Splunk OpenTelemetry Collector on EKS cluster
> :construction: This is a **DRAFT (work in progress)**.

This Lab is designed to help you get started with Observability of your EKS clusters. Below you will find instructions to:
- install _OpenTelemetry collector_ on EKS cluster
- collect _metrics_ from k8s environment and send them to Splunk Observability Cloud
- collect _traces_ from applications running in your cluster and send them to Splunk Observability Cloud


## Lab Structure
- Pre-requisites (EKS, Splunk)
- Part 1: Get Metrics from your EKS to Splunk Observability Cloud
  - 1.1 Point kubeconfig to your EKS cluster
  - 1.2 Deploy OTel collector
  - 1.3 Check your metrics
- Part 2: Get Traces from your applications running on EKS to Splunk Observability Cloud
  - 2.1 Deploy a sample JAVA app to your Cluster
- Troubleshooting


## Pre-requisites
### AWS EKS - you will need:

  - [EKS cluster](https://aws.amazon.com/eks/)
  - Administrator access to your EKS cluster and familiarity with your Kubernetes configuration.
  - [Helm 3](https://helm.sh/docs/intro/install/)

  _If you don't have an EKS cluster to practice on, feel free to follow [AWS Getting Started with EKS](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html) guide to create a Cluster & Node and then install [Helm 3](https://helm.sh/docs/intro/install/)._ 

  _There are also plenty of blogs available with instructions, i.e. [How to deploy a k8s cluster on EKS](https://ostechnix.com/deploy-kubernetes-cluster-on-aws-with-amazon-eks/), [How to add worker nodes to your EKS cluster](https://ostechnix.com/add-worker-nodes-to-amazon-eks-cluster/)_

### Splunk - you will need:

  To send Metrics/Traces:
  - Splunk Observability Cloud account and the below values:
    - [Splunk Access Token with INGEST scope](https://docs.splunk.com/Observability/admin/authentication-tokens/org-tokens.html#admin-org-tokens)
    - [Splunk Realm](https://dev.splunk.com/observability/docs/realms_in_endpoints/)


## Part 1: Get Metrics from your EKS to Splunk Observability Cloud

### 1.0. Install _AWS CLI_ and _kubectl_
  If not already done, on your host:
  1. Install and configure [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) v1.18 or later (needs the eks subcommand)
  2. Install [kubectl](https://kubernetes.io/docs/tasks/tools/) (acceptable version for your cluster)

### 1.1. Point _kubeconfig_ to your EKS cluster

  Run below and specify your Region (i.e. ap-southeast-2) and Cluster name (the name you used when creating EKS cluster):

  ```bash
  aws eks update-kubeconfig --region <EKS_CLUSTER_REGION> --name <EKS_CLUSTER_NAME>
  ```
  Note: if your AWS CLI is not configured - you will need to run `aws configure` and specify your `AWS Access Key ID` and `AWS Secret Access Key` first, and then run above command.

  Successful output should look like this:
  `Added new context arn:aws:eks:YourEKSRegion:AcccntNum:cluster/YourClusterName to /Users/YourUserName/.kube/config`

  Check your connection by running:
  ```bash
  kubectl get nodes
  ```

  Successful output should contain the list of nodes you have in the EKS cluster.

  >  :wrench: Troubleshooting
  >
  > If you get an error `‘NoneType’ object is not iterable` - it most likely means that you already have **existing** k8s config.
  >
  >> Do you have an existing k8s config? 
  >>
  >> _Note: Running `aws eks update-kubeconfig --region <EKS_CLUSTER_REGION> --name <EKS_CLUSTER_NAME>` generates a `~/.kube/config`._
  >>
  >> If you already have a ~/.kube/config, there could be a conflict between the file to be generated, and the file that already exists that prevents them from being merged.
  >>
  >> - If you have a ~/.kube/config file, and you _are not actively using it_, running
  `rm ~/.kube/config` and then attempting `aws eks update-kubeconfig --region <EKS_CLUSTER_REGION> --name <EKS_CLUSTER_NAME>` afterwards will likely solve your issue.
  >> - If you _are using_ your `~/.kube/config` file, rename it something else so you could use it later, and then run the eks command again.
  >
  > If you get the `error: exec plugin: invalid apiVersion "client.authentication.k8s.io/v1alpha1"`
  >> You may need to downgrade the kubectl from 1.24 to 1.23 as below:
  >> ```bash
  >> curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.23.6/bin/darwin/amd64/kubectl
  >> chmod +x ./kubectl
  >> sudo mv ./kubectl /usr/local/bin/kubectl
  >> ```

### 1.2 Deploy Splunk OTel collector.

  We are going to deploy OTel collector via Helm chart.
  Collector in this case is running as a Daemonset
  .....
  .....
  .....
  .....


#### Add Helm repo

  ```bash
  helm repo add splunk-otel-collector-chart https://signalfx.github.io/splunk-otel-collector-chart
  ```

#### Create and configure eks-values.yaml file

  1. Create new or clone a yaml file with the sample values `eks-values.yaml`. 
  2. Update the below parameters.

  The following parameter is required for any of the destinations:

  - `clusterName`: identifies your Kubernetes cluster. This value will be associated with every metric and trace as "k8s.cluster.name" attribute.

  To Metrics/Traces to _Splunk Observability Cloud_ the following parameters are required:

  - `splunkObservability.realm`: Splunk realm to send telemetry data to (e.g. us1, eu0, au0, etc.).
  - `splunkObservability.accessToken`: Your Splunk Observability org access token with INGEST scope.

  Make sure below are included as in the provided example: 

  - distribution: eks
  - cloudProvider: aws

#### Deploy helm chart

  1. Create a namespace (it is easier to manage k8s resources when they are logically separated into workspaces, you will just need to remember to add flag -n or -namespace to your helm and kubectl commands)

```bash
kubectl create namespace splunk
```

  Check your work

```bash
kubectl get namespaces
```

  2. Deploy collector helm chart to your new namespace

    Replace `<path-to-your-eks-values.yaml>` with full path to your yaml file (e.g. `/Users/xxx/eks-values.yaml`) and run the below

```bash
helm install my-splunk-otel-collector --values <path-to-your-eks-values.yaml> splunk-otel-collector-chart/splunk-otel-collector --namespace splunk
```

    Check your work

    Run the below command to get the list of pods in all namespaces (flag `-A`), you can also run kubectl `get pods -n splunk` to get the list of pods deployed as a part of otel collector helm chart.

```bash
kubectl get pods -A
```

> :bulb: Info note: What pods do I see? (agent, clusterReceiver)
>
> [Splunk OpenTelemetry Collector for Kubernetes](https://github.com/signalfx/splunk-otel-collector-chart) has the following components and applications:
> - Splunk OpenTelemetry Collector Agent (_agent_) to fetch metrics, traces and logs from a Kubernetes cluster (deployed as a Kubernetes _DaemonSet_)
> - Splunk OpenTelemetry Collector Cluster Receiver (_clusterReceiver_) to fetch metrics from a Kubernetes API (deployed as a Kubernetes 1-replica _Deployment_)
>> - Optional, not used in this lab Splunk OpenTelemetry Collector Gateway (gateway) to forward data through it to reduce load on Kubernetes API and apply additional processing (deployed as a Kubernetes _Deployment_)
>


#### How to upgrade collector config

  Make sure you run helm repo update before you upgrade.
  To upgrade a deployment you can use the same command as for installing but use `upgrade` instead of `install`:

  ```bash
  helm upgrade my-splunk-otel-collector --values <path-to-your-eks-values.yaml> splunk-otel-collector-chart/splunk-otel-collector --namespace splunk
  ```

#### How to uninstall collector

  To uninstall/delete a deployment with name `my-splunk-otel-collector`:

  ```bash
  helm delete my-splunk-otel-collector --namespace splunk
  ```

### Check your Metrics in Splunk Observability Cloud

3 easy ways to start working with your Kubernetes metrics:

1. Use Kubernetes navigator
   Login to Splunk Observability Cloud and navigate to _Infrastructure_ -> _Kubernetes_.

   You should be able to see the numbed of _Nodes_ and _Workloads_.

   Click on the _Nodes_ and explore the metrics down to the Container/Pod level. Similarly, you can explore _Workloads_ related views.

   ![Splunk Observability Kubernetes Navigator](/screenshots/splunk_kubernetes_navigator_view.png "Splunk Observability Kubernetes Navigator")


2. Explore metrics in Metrics Finder
   
   As an example, navigate to _Metric Finder_ (in the menu on the left) and search for `k8s.container.cpu_limit` to see the list of Properties (dimensions and tags) available for this metric.

   ![Splunk Observability Metric Finder](/screenshots/splunk-metric-finder.png "Splunk Observability Metric Finder")
   
   If you want to see what metrics have a specific dimension, try searching for `k8s.cluster.name` to see metrics that contain `k8s.cluster.name` in their name or attributes.


3. Work with Metrics in Dashboards and Charts

    Navigate to _Dashboards_ and search for _kubernetes_, explore the out-of-the-box dashboard.
    
    ![Splunk Observability Kubernetes Dashboards](/screenshots/splunk-kubernetes-ootb-dashboards.png "Splunk Observability Kubernetes Dashboards")


## Part 2: Get Traces from your applications running on EKS to Splunk Observability Cloud


