# Is it Observable
<p align="center"><img src="/image/logo.png" width="40%" alt="Is It observable Logo" /></p>

## Episode : Fluentbit the Telemetry Agent
This repository contains the files utilized during the tutorial presented in the dedicated IsItObservable episode related to Fluentbit v2.
<p align="center"><img src="/image/fluentbit.png" width="40%" alt="Fluentbit Logo" /></p>

What you will learn
* How to use the [Fluentbit v2](https://fluentbit.io/)

This repository showcase the usage of Fluentbit  with :
* The Otel-demo
* The OpenTelemetry Operator
* Nginx ingress controller
* Dynatrace
* Coulple of Prometheus exporters provided by the Prometheus Operator


We will send all Telemetry data produced by the Otel-demo to Dynatrace.

## Prerequisite
The following tools need to be install on your machine :
- jq
- kubectl
- git
- gcloud ( if you are using GKE)
- Helm


## Deployment Steps in GCP

You will first need a Kubernetes cluster with 2 Nodes.
You can either deploy on Minikube or K3s or follow the instructions to create GKE cluster:
### 1.Create a Google Cloud Platform Project
```shell
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
gcloud services enable monitoring.googleapis.com \
    cloudtrace.googleapis.com \
    clouddebugger.googleapis.com \
    cloudprofiler.googleapis.com \
    --project ${PROJECT_ID}
```
### 2.Create a GKE cluster
```shell
ZONE=europe-west3-a
NAME=isitobservable-fluentbitv2
gcloud container clusters create "${NAME}" --zone ${ZONE} --machine-type=e2-standard-2 --num-nodes=3 
```


## Getting started
### Dynatrace Tenant
#### 1. Dynatrace Tenant - start a trial
If you don't have any Dyntrace tenant , then i suggest to create a trial using the following link : [Dynatrace Trial](https://bit.ly/3KxWDvY)
Once you have your Tenant save the Dynatrace tenant hostname in the variable `DT_TENANT_URL` (for example : dedededfrf.live.dynatrace.com)
```
DT_TENANT_HOSTNAME=<YOUR TENANT Host>
```

#### 2. Create the Dynatrace API Tokens
Create a Dynatrace token with the following scope ( left menu Acces Token):
* ingest metrics
* ingest OpenTelemetry traces
* ingest logs
<p align="center"><img src="/image/data_ingest.png" width="40%" alt="data token" /></p>
Save the value of the token . We will use it later to store in a k8S secret

```
DATA_INGEST_TOKEN=<YOUR TOKEN VALUE>
```
### 3.Clone the Github Repository
```shell
https://github.com/isItObservable/fluentbitv2
cd fluentbitv2
```
### 4.Deploy most of the components
The application will deploy the otel demo v1.2.1
```shell
chmod 777 deployment.sh
./deployment.sh  --clustername "${NAME}" --dthost "${DT_TENANT_URL}" --dttoken "${DATA_INGEST_TOKEN}"
```
### 5.Configure Fluentbit

Edit the fluentbit daemonset to expose the otlphttp port
```shell
kubectl edit ds fluent-bit -n fluentbit
```
add the followin port int he ports section:
```yaml
- containerPort: 4318
  name: otlphttp
  protocol: TCP
```
now edit the fluent-bit service to add the otlphttp port:
```shell
kubectl edit svc fluent-bit -n fluentbit
```
Add the new port :
```yaml
- name: otlphttp
  port: 4318
  protocol: TCP
  targetPort: otlphttp
```
### 5. Fluentbit pipeline

#### 1. Logs
The current pipeline deployed is already configured to collect logs, transform the logs and send it to dynatrace.

Let's have a look at this pipeline
```shell
cat fluentbit/fluent-bit_initial.yaml
```

#### 2. Add Expect after renaming the kubernetes metada

To validate our pipeline step we can utilze the `Expect` filter.
In our case we want to make sure that the k8s.pod.name key exists in our stream :
```yaml
    [FILTER]
       Name          expect
       Match         kube.*
       key_exists    k8s.pod.name
       key_val_is_not_null k8s.namespace.name
       action warn
```

Let's modify our current pipeline  by adding our expect step after :
```yaml
[FILTER]
  Name modify
  Match kube.*
  ...
```
To edit our current pipeline :
```shell
vi fluentbit/fluent-bit_initial.yaml
```
After applying our changes , let's apply the new version of the pipeline and restart the fluentbit agents :
```shell
kubectl apply -f fluentbit/fluent-bit_initial.yaml -n fluentbit
kubectl  rollout restart ds fluent-bit -n fluentbit
```


#### 3. Let's add metrics

##### Collect host metrics with the node exporter plugin
Fluentbit provides a Node exporter within the fluentbit agents.
let's use it to collect metrics in our current pipeline :
```yaml
[INPUT]
 name node_exporter_metrics
 tag  otel.node
 scrape_interval 2
```

Now that we have a input plugin collecting metrics we need to also add output plugins for our metrics:
```yaml
[OUTPUT]
name            prometheus_exporter
match           otel.*
host            0.0.0.0
port            2021
add_label      k8s.cluster.name CLUSTER_NAME_TO_REPLACE
```

Let's modify our current pipeline and look at the port 2021 of our fluentbit agent
```shell
vi fluentbit/fluent-bit_initial.yaml
```
And update the pipeline of our agents :
```shell
kubectl apply -f fluentbit/fluent-bit_initial.yaml -n fluentbit
kubectl  rollout restart ds fluent-bit -n fluentbit
```

Now let's have a look a the metrics produced by our agent:
```shell
kubectl get pods -n fluenbit
```
select one of the pod and apply the following command: 
```shell
kubectl port-forward <fluentbit pod id> -n fluentbit 2021:2021
```
open you browser and opent the page http://localhost:2021/metrics


#### 4. Let's scrape Prometheus metrics
In the cluster the Prometheus operator has been deployed.
It means that we can collect the metrics produced by the kubestate metrics exporter.
```yaml
    [INPUT]
      name prometheus_scrape
      host prometheus-kube-state-metrics.default.svc
      port 8080
      tag otel.metrics
      metrics_path /metrics
      scrape_interval 10s
```
Let's modify our current pipeline and look at the port 2021 of our fluentbit agent
```shell
vi fluentbit/fluent-bit_initial.yaml
```
And update the pipeline of our agents :
```shell
kubectl apply -f fluentbit/fluent-bit_initial.yaml -n fluentbit
kubectl  rollout restart ds fluent-bit -n fluentbit
```
#### 5. Let's add the fluentbit metrics
```yaml
  [INPUT]
    name fluentbit_metrics
    tag  otel.fluent
    scrape_interval 2
```
Let's modify our current pipeline and look at the port 2021 of our fluentbit agent
```shell
vi fluentbit/fluent-bit_initial.yaml
```
And update the pipeline of our agents :
```shell
kubectl apply -f fluentbit/fluent-bit_initial.yaml -n fluentbit
kubectl  rollout restart ds fluent-bit -n fluentbit
```

#### 6. Let's add OpenTelemetry
```yaml
     [INPUT]
       name opentelemetry
       listen 0.0.0.0
       port 4318
       tag otel.otel
```
and add the output openTelemetry to send metrics and traces using the output OpenTelemtry:
```yaml
[OUTPUT]
  Name opentelemetry
  Host  ${DT_TENANT_URL}
  Port  443
  Match otel.*
  Metrics_uri  /api/v2/otlp/v1/metrics
  Traces_uri  /api/v2/otlp/v1/traces
  Logs_uri   /api/v2/otlp/v1/logs
  Log_response_payload True
  Tls On
  Tls.verify Off
  header Authorization Api-Token ${DATA_INGEST_TOKEN}
  header Content-type application/x-protobuf
```
Let's modify our current pipeline and look at the port 2021 of our fluentbit agent
```shell
vi fluentbit/fluent-bit_initial.yaml
```
And update the pipeline of our agents :
```shell
kubectl apply -f fluentbit/fluent-bit_initial.yaml -n fluentbit
kubectl  rollout restart ds fluent-bit -n fluentbit
```