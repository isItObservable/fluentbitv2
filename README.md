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
