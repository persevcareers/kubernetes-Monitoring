# Prometheus Monitoring 

## Metric Collection
- **Pull Model**: Prometheus retrieves metrics over HTTP using a pull model.
- **Pushgateway**: Option to push metrics to Prometheus, useful for short-lived Kubernetes jobs & Cronjobs.
  
## Metric Endpoint
- Systems expose metrics on an `/metrics` endpoint for Prometheus to pull.
  
## PromQL
- Flexible query language for querying metrics in the Prometheus dashboard and visualizing them in Prometheus UI and Grafana.
  
## Prometheus Exporters
- Libraries that convert third-party app metrics to Prometheus format.
- Example: Prometheus node exporter for exposing Linux system-level metrics.
  
## TSDB (Time-Series Database)
- Prometheus utilizes TSDB for efficient data storage.
- By default, data is stored locally with options for integrating remote storage to avoid single points of failure.

# Installation ,

## The Kubernetes Prometheus monitoring stack has the following components.

- **Prometheus Server**
- **Alert Manager**
- **Node Exporter**
- **Kube State Metrics**
- **Grafana**

## Prometheus Kubernetes Manifests:
```bash
git clone https://github.com/persevcareers/kubernetes-prometheus
cd prometheus
```

# Create a Namespace & ClusterRole

- **First, we will create a Kubernetes namespace for all our monitoring components. If you don’t create a dedicated namespace, all the Prometheus Kubernetes deployment objects get deployed on the default namespace.**

- **Execute the following command to create a new namespace named `monitoring`:**

```bash
kubectl create namespace monitoring
```
## Prometheus uses Kubernetes APIs to read all the available metrics from Nodes, Pods, Deployments, etc. For this reason, we need to create an RBAC policy with read access to required API groups and bind the policy to the monitoring namespace.

- **In the role, given below, you can see that we have added get, list, and watch permissions to nodes, services endpoints, pods, and ingresses. The role binding is bound to the monitoring namespace. If you have any use case to retrieve metrics from any other object, you need to add that in this cluster role.
Create the role using the following command:**

```bash

kubectl create -f clusterRole.yaml
```
## Create a Config Map To Externalize Prometheus Configurations
## All configurations for Prometheus are part of prometheus.yaml file and all the alert rules for Alertmanager are configured in prometheus.rules.

By externalizing Prometheus configs to a Kubernetes config map, you don’t have to build the Prometheus image whenever you need to add or remove a configuration. You need to update the config map and restart the Prometheus pods to apply the new configuration.

- **Step 1:**
Create a file called config-map.yaml and copy the file contents from this link.

- **Step 2:**
Execute the following command to create the config map in Kubernetes:

```bash
kubectl create -f config-map.yaml
```
It creates two files inside the container.

The prometheus.yaml contains all the configurations to discover pods and services running in the Kubernetes cluster dynamically. We have the following scrape jobs in our Prometheus scrape configuration:

- **kubernetes-apiservers: It gets all the metrics from the API servers.**
- **kubernetes-nodes: It collects all the Kubernetes node metrics.**
- **kubernetes-pods: All the pod metrics get discovered if the pod metadata is annotated with prometheus.io/scrape and prometheus.io/port annotations.**
- **kubernetes-cadvisor: Collects all cAdvisor metrics.**
- **kubernetes-service-endpoints: All the Service endpoints are scrapped if the service metadata is annotated with prometheus.io/scrape and prometheus.io/port annotations. It can be used for black-box monitoring.**
- **The prometheus.rules contains all the alert rules for sending alerts to the Alertmanager.**

##Create a Prometheus Deployment
- **Step 1:**
Create a file named prometheus-deployment.yaml and copy the following contents onto the file. In this configuration, we are mounting the Prometheus config map as a file inside /etc/prometheus as explained in the previous section.

This deployment uses the latest official Prometheus image from the Docker Hub. Also, we are not using any persistent storage volumes for Prometheus storage as it is a basic setup. When setting up Prometheus for production use cases, make sure you add persistent storage to the deployment.

- **Step 2:**
Create a deployment on monitoring namespace using the above file:

```bash
kubectl create -f prometheus-deployment.yaml --namespace=monitoring
```
- **Step 3:**
You can check the created deployment using the following command:

```bash
kubectl get deployments --namespace=monitoring
```

# Connecting To Prometheus Dashboard



## Exposing Prometheus as a Service [NodePort & LoadBalancer]

To access the Prometheus dashboard over an IP or a DNS name, you need to expose it as a Kubernetes service.

1. Create a file named `prometheus-service.yaml` and copy the following contents. This will expose Prometheus on all Kubernetes node IPs on port 30000.

2. Create the service using the following command:

```bash
kubectl create -f prometheus-service.yaml --namespace=monitoring
```

# Setting Up Kube State Metrics

The Kube State Metrics service provides many metrics that are not available by default. It's essential to deploy Kube State Metrics to monitor all your Kubernetes API objects like deployments, pods, jobs, cronjobs, etc.


# What is Kube State Metrics?

Kube State Metrics is a service that communicates with the Kubernetes API server to gather details about all the API objects, such as deployments, pods, daemonsets, StatefulSets, etc.

Primarily, it produces metrics in Prometheus format, ensuring stability aligned with the Kubernetes API. It provides Kubernetes objects and resources metrics that cannot be directly obtained from native Kubernetes monitoring components.

The Kube State Metrics service exposes all the metrics on the `/metrics` URI, allowing Prometheus to scrape all the metrics exposed by Kube State Metrics.

Following are some of the important metrics you can obtain from Kube State Metrics:

- Node status and node capacity (CPU and memory)
- Replica-set compliance (desired/available/unavailable/updated status of replicas per deployment)
- Pod status (waiting, running, ready, etc.)
- Ingress metrics
- PV (Persistent Volume) and PVC (Persistent Volume Claim) metrics
- Daemonset and Statefulset metrics
- Resource requests and limits
- Job and Cronjob metrics


# Kube State Metrics Setup

To set up Kube State Metrics, follow these steps:

1. **Service Account**: Create a service account for Kube State Metrics to operate under.

2. **Cluster Role**: Define a cluster role granting access to all Kubernetes API objects for Kube State Metrics.

3. **Cluster Role Binding**: Bind the service account with the cluster role, ensuring proper access permissions.

4. **Kube State Metrics Deployment**: Deploy the Kube State Metrics service using a Kubernetes deployment.

5. **Service**: Expose the metrics provided by Kube State Metrics through a Kubernetes service.



# Deploying Kube State Metrics Objects

All the above Kube State Metrics objects will be deployed in the `kube-system` namespace.

To deploy the components, follow these steps:

**Step 1**: Clone the Github repo:

```bash
git clone https://github.com/persevcareers/kubernetes-monitoring.git
cd kube-state-metrics
```

 ## Step 2: Create Objects

Create all the objects by pointing to the cloned directory:

```bash
kubectl apply -f kube-state-metrics/
```

 ## Step 3: Check the deployment status using the following command.
```bash
kubectl get deployments kube-state-metrics -n kube-system
```

## Step4: Expose the clusterIP ,
```bash
kubectl expose service <svcname> --port=port--target-port=port --type=NodePort --name=<newsvcname> -n kube-system
kubectl expose service kube-state-metrics --port=8080 --target-port=8080 --type=NodePort --name=kube-state-metrics-nodeport -n kube-system
```
## Kube State Metrics Prometheus Config

All the Kube State Metrics can be obtained from the Kube State service endpoint on the `/metrics` URI.

This configuration can be added as part of the Prometheus job configuration. You need to add the following job configuration to your Prometheus config for Prometheus to scrape all the Kube State Metrics.


## Setting Up Grafana
Using Grafana you can create dashboards from Prometheus metrics to monitor the kubernetes cluster.

``` bash
git clone https://github.com/persevcareers/kubernetes-monitoring.git
cd grafana
```

## Deploy Grafana On Kubernetes

- **Step 1**: Create the configmap using the following command.
``` bash
kubectl create -f grafana-datasource-config.yaml
```

- **Step 2**: Create a file named deployment.yaml
  
``` bash
kubectl create -f deployment.yaml
```

- **Step 3**:Create the service.
``` bash
kubectl create -f service.yaml
```

## Use the following default username and password to log in. Once you log in with default credentials, it will prompt you to change the default password.

User: admin
Pass: admin
Now you should be able to access the Grafana dashboard using any node IP on port 32000. 

# Setting Up Alertmanager

Alertmanager handles all the alerting mechanisms for Prometheus metrics. There are many integrations available to receive alerts from the Alertmanager (Slack, email, API endpoints, etc).

# Setting Up Alertmanager

Alertmanager handles all the alerting mechanisms for Prometheus metrics. There are many integrations available to receive alerts from the Alertmanager (Slack, email, API endpoints, etc).

## Alert Manager Setup

Alert Manager setup involves the following key configurations:

1. A config map for AlertManager configuration
2. A config Map for AlertManager alert templates
3. Alert Manager Kubernetes Deployment
4. Alert Manager service to access the web UI.

First, clone the repository containing the necessary configurations:

``` bash
git clone https://github.com/persevcareers/kubernetes-monitoring.git
cd alert-manager
```

# Config Map for Alert Manager Configuration

Alert Manager reads its configuration from a config.yaml file. It contains the configuration of alert template path, email, and other alert receiving configurations.

# Let’s create the config map using kubectl.
```bash
kubectl create -f AlertManagerConfigmap.yaml
```

# Config Map for Alert Template

##We need alert templates for all the receivers we use (email, Slack, etc). Alert manager will dynamically substitute the values and deliver alerts to the receivers based on the template. You can customize these templates based on your needs.


## Create the configmap using kubectl.
``` bash
kubectl create -f AlertTemplateConfigMap.yaml
```
## Create a Deployment
In this deployment, we will mount the two config maps we created.

## Create the alert manager deployment using kubectl.
```bash
kubectl create -f deployment.yaml
```
## Create the service using kubectl.
```bash
kubectl create -f Service.yaml
```
Now, you will be able to access Alert Manager on Node Port 31000.

By default, most of the Kubernetes clusters expose the metric server metrics (Cluster level metrics from the summary API) and Cadvisor (Container level metrics). It does not provide detailed node-level metrics.

To get all the kubernetes node-level system metrics, you need to have a node-exporter running in all the kubernetes nodes. It collects all the Linux system metrics and exposes them via /metrics endpoint on port 9100

Similarly, you need to install Kube state metrics to get all the metrics related to kubernetes objects.


## Kubernetes Manifests

The Kubernetes manifest used in this guide is present in the Github repository. Clone the repo to your local system.
```bash
git clone https://github.com/persevcareers/kubernetes-monitoring.git
cd node-exporter
```

#Deploy the daemonset using the kubectl command.
```bash
kubectl create -f daemonset.yaml
```
#List the daemonset in the monitoring namespace and make sure it is in the available state.
```bash
kubectl get daemonset -n monitoring
``
#check the service’s endpoints and see if it is pointing to all the daemonset pods.
```bash
kubectl get endpoints -n monitoring
```

##Node-exporter Prometheus Config
We have the node-exporter daemonset running on port 9100 and a service pointing to all the node-exporter pods.

You need to add a scrape config to the Prometheus config file to discover all the node-exporter pods.

Let’s take a look at the Prometheus scrape config required to scrape the node-exporter metrics.
```bash
      - job_name: 'node-exporter'
        kubernetes_sd_configs:
          - role: endpoints
        relabel_configs:
        - source_labels: [__meta_kubernetes_endpoints_name]
          regex: 'node-exporter'
          action: keep
```

In this config, we mention the role as endpoints to scrape the endpoints with the name node-exporter.

See Prometheus config map (https://github.com/persevcareers/kubernetes-monitoring/blob/main/prometheus/config-map.yaml)file we have created for the Kubernetes monitoring . It includes all the scrape configs for kubernetes components.

Once you add the scrape config to Prometheus, you will see the node-exporter targets in Prometheus, as shown below.
