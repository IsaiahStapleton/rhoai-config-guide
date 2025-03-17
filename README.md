# Guide to Configuring RHOAI for Model Deployment

## Purpose

The purpose of this guide is to offer the simplest steps for configuring Red Hat OpenShift AI (RHOAI) for model deployment.
RHOAI does not come properly configured for GPU use and model deployment out of the box after installing the operator.
This guide will give you the steps needed to do so, as well as offer insight into why each step is necessary for model deployment. This guide focuses on preparing for deploying models using the vLLM ServingRuntime for KServe.

Required Operators:
- ***Node Feature Discovery (NFD) Operator:*** Detects and labels nodes based on hardware capabilities for proper AI workload scheduling.
- ***NVIDIA GPU Operator:*** Automates deployment of GPU drivers, CUDA libraries, and dependencies for AI workloads.
- ***OpenShift Service Mesh Operator:*** Provides Istio for managing secure communication between model-serving components.
- ***Red Hat OpenShift Serverless Operator:*** Provides Knative Serving for scalable and event-driven AI model deployment.
- ***Red Hat Authorino Operator:*** Provides authentication and authorization for secure access to AI model endpoints.
- ***Red Hat OpenShift AI Operator:*** Manages and deploys AI components and services within OpenShift.

Environment used for:
- Deploying the ***Granite model*** on RHOAI, using ***A100 NVIDIA GPU***, served using ***vLLM ServingRuntime for KServe***
- Gathering metrics and doing benchmarking on the deployed granite models 


## 1. Enabling GPU Support

### 1.1 Install the Node Feature Discovery (NFD) Operator 

***The NFD Operator is used to detect and label nodes based on their hardware capabilities. This is important that we can properly schedule AI workloads or deploy models on GPU nodes.***

Apply the Namespace, OperatorGroup, and Subscription object

`oc apply -f manifests/01/nfd-operator.yaml`

Verify the operator is installed and running before moving on to the next step.

`oc get pods -n openshift-nfd -w`

### 1.2 Apply the NFD Instance Object 

***After installing the NFD Operator, you must apply an NFD instance object to deploy the NFD DaemonSet, which scans nodes for hardware features and labels them accordingly.*** 

`oc apply -f manifests/01/nfd-instance.yaml`

### 1.3 Install the NVIDIA GPU Operator 

***The NVIDIA GPU Operator is essential for RHOAI since it is used to automate the deployment of GPU drivers, CUDA libraries, and other dependencies required for AI workloads and model serving.***

Apply the Namespace, OperatorGroup, and Subscription object

`oc apply -f manifests/01/nvidia-gpu-operator.yaml`

Verify the operator is installed and running before moving on to the next step.

### 1.4 Apply NVIDIA GPU ClusterPolicy 

***The NVIDIA GPU Operator itself only installs the operator framework, but doesn't automaticaly deploy the necessary components. We need to apply the ClusterPolicy as that is what enables and configures GPU support within the cluster.***

Create the ClusterPolicy

`oc get csv -n nvidia-gpu-operator -l operators.coreos.com/gpu-operator-certified.nvidia-gpu-operator -ojsonpath='{.items[0].metadata.annotations.alm-examples}' | jq '.[0]' > scratch/nvidia-gpu-clusterpolicy.json`

Apply the ClusterPolicy

`oc apply -f scratch/nvidia-gpu-clusterpolicy.json`

## 2. Install RHOAI KServe Dependencies

***KServe provides scalable and efficient model serving capabilities, enabling deployment, inference, and monitoring of AI models within OpenShift.***

### 2.1 Install the OpenShift Service Mesh Operator

***The OpenShift Service Mesh Operator is required for KServe because KServe relies on Istio for managing communication between model serving components***

Apply the Subscription object to install the operator

`oc apply -f manifests/02/servicemesh-subscription.yaml`

### 2.2 Install the Red Hat OpenShift Serverless Operator

***The Red Hat OpenShift Serverless Operator is required because it provides Knative Serving, which enables serverless capabilities that assist in model deployment.***

`oc apply -f manifests/02/serverless-operator.yaml`

### 2.2 Install the Red Hat Authorino Operator

***The Red Hat Authorino Operator is required because it provides authentication for API requests, ensuring secure access to AI model endpoints.***

`oc apply -f manifests/02/authorino-subscription.yaml`

## 3. Install the Red Hat OpenShift AI Operator

Apply the Namespace, OperatorGroup, and Subscription object

`oc apply -f manifests/03/rhoai-operator.yaml`


### 3.1 Install RHOAI Components

Wait for the RHOAI operator to be installed before proceeding with this step. (***Provide steps for checking***)

`oc apply -f manifests/03/rhoai-operator-dsc.yaml`

