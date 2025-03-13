# Guide to Configuring RHOAI for Model Deployment

## Purpose

The purpose of this guide is to offer the simplest steps for configuring Red Hat OpenShift AI (RHOAI) for model deployment.
RHOAI does not come properly configured for GPU use and model deployment out of the box after installing the operator.
This guide will give you the steps needed to do so, as well as offer insight into why each step is necessary for model deployment. This guide focuses on preparing for deploying models using the vLLM ServingRuntime for KServe.

My environment at time of demo:
- Openshift {4.18.1}
- RHOAI {version}
- Other Operator versions


Environment used for:
- Deploying the ***Granite model*** on RHOAI, using ***A100 NVIDIA GPU*** {other specs}, served using ***vLLM ServingRuntime for KServe***
- Gathering metrics and doing benchmarking on the deployed granite models 

Observability:
- Prometheus: To gather metrics 
- Grafana: To visualize metrics
- NVIDIA DCGM: Export metrics related to GPU (Temperature, Utilization, etc.)
- vLLM Serving Runtime: Exporter metrics related to model performance (Throughput, Latency, Time to First Token, etc.)

## 1. Enabling GPU Support

### 1.1 Install the Node Feature Discovery (NFD) Operator 

Apply the Namespace, OperatorGroup, and Subscription object

`oc apply -f manifests/01/nfd-operator.yaml`


### 1.2 Install the NVIDIA GPU Operator 

Apply the Namespace, OperatorGroup, and Subscription object

`oc apply -f manifests/01/nvidia-gpu-operator.yaml`


*** TODO: Instructions for Creating clusterpolicy 

## 2. Install RHOAI Kserve Dependencies

### 2.1 Install the OpenShift Service Mesh Operator

Apply the Subscription object to install the operator

`oc apply -f manifests/02/servicemesh-subscription.yaml`

### 2.2 Install the Red Hat OpenShift Serverless Operator

`oc apply -f manifests/02/serverless-operator.yaml`

### 2.2 Install the Red Hat Authorino Operator

`oc apply -f manifests/02/authorino-subscription.yaml`

## 3. Install the Red Hat OpenShift AI Operator

Apply the Namespace, OperatorGroup, and Subscription object

`oc apply -f manifests/03/rhoai-operator.yaml`


### 3.1 Install RHOAI Components

Wait for the RHOAI operator to be installed before proceeding with this step. (***Provide steps for checking***)

`oc apply -f manifests/03/rhoai-operator-dsc.yaml`


***TODO: Provide context/explanation for each step***
