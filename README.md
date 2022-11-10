# Is it Observable
<p align="center"><img src="/image/logo.png" width="40%" alt="Is It observable Logo" /></p>

## Episode : Cilium 
This repository contains the files utilized during the tutorial presented in the dedicated IsItObservable episode related to Cilium and Hubble.
<p align="center"><img src="/image/logo_cilium.png" width="40%" alt="TraceTest Logo" /></p>

What you will learn
* How to deploy Cilium & hubble
* How to ingest the Cilium/hubble metrics in Dynatrace using the OpenTeletry Collector
* Try the Hubble-otel project
* Create a CiliumNetworkPolicy

This repository showcase the usage of the Cilium  with :
* the Otel-demo
* The OpenTelemetry Operator
* Hubble
* the Istio integration  
* Prometheus
* Dynatrace

if you don't have a Dynatrace Tenant, you can start a [trial](https://bit.ly/3KxWDvY)

## Prerequisite
The following tools need to be install on your machine :
- jq
- kubectl
- git
- gcloud ( if you are using GKE)
- Helm


## Deployment Steps in GCP

You will first need a Kubernetes cluster with 4 Nodes.
### 1.Create a Google Cloud Platform Project
```
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
gcloud services enable monitoring.googleapis.com \
    cloudtrace.googleapis.com \
    clouddebugger.googleapis.com \
    cloudprofiler.googleapis.com \
    --project ${PROJECT_ID}
```
### 2.Create a GKE cluster
```
ZONE=europe-west3-a
NAME=isitobservable-cilium
gcloud container clusters create "${NAME}" \
 --node-taints node.cilium.io/agent-not-ready=true:NoExecute \
 --zone ${ZONE} --machine-type=e2-standard-2 --num-nodes=4
```

### 3.Clone the Github Repository
```
git clone https://github.com/isItObservable/cilium-hubble
cd cilium-hubble
```
### 4.Install Cilium & Hubble
#### 1. Install Cilium CLI
```
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/master/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```
#### 2. Deploy the Cilium 
```
cilium install --config bpf-lb-sock-hostns-only=true  --kube-proxy-replacement=strict  --helm-set  prometheus.enabled=true   --helm-set hubble.enabled=true  --helm-set operator.prometheus.enabled=true   --helm-set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,http} --helm-set-string extraConfig.enable-envoy-config=true"
```
#### 3. Check the status of cilium
```
cilium status
```
You should have the following output:
<p align="center"><img src="/image/cilium_status_1.png" width="40%" alt="TraceTest Logo" /></p>

#### 4. Enable Hubble
```
cilium hubble enable --ui
```
#### 5. Check the status of Cilium
```
cilium status
```
You should have the following output:
<p align="center"><img src="/image/cilium_status_2.png" width="40%" alt="TraceTest Logo" /></p>

#### 6. Install the hubble CLI
```
export HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
HUBBLE_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then HUBBLE_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-${HUBBLE_ARCH}.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-${HUBBLE_ARCH}.tar.gz /usr/local/bin
rm hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
```
#### 7. Enable a port-forward to the hubble relay ( for the hubble CLI)
```
cilium hubble port-forward&
```
#### 8. Check the status of Hubble
```
hubble status
```
#### 9. deploy Hubble services
```
kubectl apply -f cilium/ciliumagent-metric_svc.yaml
```

### 4.Install Cilium Istio

#### 1. Install the istioCTL from cilium
```
curl -L https://github.com/cilium/istio/releases/download/1.10.6-1/cilium-istioctl-1.10.6-1-linux-amd64.tar.gz | tar xz
```
#### 2. Deploy Istio
```
./cilium-istioctl install -y
```
#### 3. Label the application namespace
```
kubectl create ns otel-demo
kubectl label namespace otel-demo istio-injection=enabled
```
### 5.Prometheus

#### 1. Deploy Prometheus without Grafana
 ```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack  
```

### 6. Install the OpenTelemetry Operator

####  1. Deploy the cert-manager
```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.yaml
```

Wait for pod webhook started
```
kubectl wait pod -l app.kubernetes.io/component=webhook -n cert-manager --for=condition=Ready --timeout=2m
```

#### 2. Deploy the opentelemetry operator
```
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
```

### 6. Update ingress rules
#### 1. Get the external ip adress
Since we are using Ingress controller to route the traffic , we will need to get the public ip adress of our ingress.
With the public ip , we would be able to update the deployment of the ingress for :
* otel-demo application
* grafana 
* hubble ui
```
IP=$(kubectl get svc istio-ingressgateway -n istio-system -ojson | jq -j '.status.loadBalancer.ingress[].ip')
```
#### 2. Update the manifest files
update the following files to update the ingress definitions :
```
sed -i "s,IP_TO_REPLACE,$IP," istio/otel-demo-gateway.yaml
```

### 7. Dynatrace
#### 1. Dynatrace Tenant - start a trial
If you don't have any Dyntrace tenant , then i suggest to create a trial using the following link : [Dynatrace Trial](https://bit.ly/3KxWDvY)
Once you have your Tenant save the Dynatrace (including https) tenant URL in the variable `DT_TENANT_URL` (for example : https://dedededfrf.live.dynatrace.com)
```
DT_TENANT_URL=<YOUR TENANT URL>
```

##### Generate Token to ingest data
Create a Dynatrace token with the following scope:
* ingest metrics
* ingest OpenTelemetry traces
<p align="center"><img src="/image/data_ingest.png" width="40%" alt="data token" /></p>
Save the value of the token . We will use it later to store in a k8S secret

```
DATA_INGEST_TOKEN=<YOUR TOKEN VALUE>
```

### 8. Configure the OpenTelemetry Collector


#### Udpate the openTelemetry manifest file
```
CLUSTERID=$(kubectl get namespace kube-system -o jsonpath='{.metadata.uid}')
CLUSTERNAME="YOUR OWN NAME"
sed -i "s,CLUSTER_ID_TO_REPLACE,$CLUSTERID," kubernetes-manifests/openTelemetry-sidecar.yaml
sed -i "s,CLUSTER_NAME_TO_REPLACE,$CLUSTERNAME," kubernetes-manifests/openTelemetry-sidecar.yaml
sed -i "s,DT_TOKEN,$DATA_INGEST_TOKEN," kubernetes-manifests/openTelemetry-manifest.yaml
sed -i "s,DT_TENANT_URL,$DT_TENANT_URL," kubernetes-manifests/openTelemetry-manifest.yaml
sed -i "s,CLUSTER_ID_TO_REPLACE,$CLUSTERID," kubernetes-manifests/openTelemetry-sidecar_hubble.yaml
sed -i "s,CLUSTER_NAME_TO_REPLACE,$CLUSTERNAME," kubernetes-manifests/openTelemetry-sidecar_hubble.yaml
```
#### Deploy the OpenTelemetry Collector
```
kubectl create ns otel
kubectl apply -f kubernetes-manifests/rbac.yaml -n otel
kubectl apply -f kubernetes-manifests/openTelemetry-manifest.yaml -n otel
```
### 7. Deploy the demo application
#### Deploy otel-demo application
```
VERSION=v1.0.0
sed -i "s,VERSION_TO_REPLACE,$VERSION," kubernetes-manifests/K8sdemo.yaml
kubectl apply -f kubernetes-manifests/openTelemetry-sidecar.yaml -n otel-demo
kubectl apply -f K8sdemo.yaml -n otel-demo
```
#### Deploy the ingress rules
```
kubectl apply -f istio/otel-demo-gateway.yaml
```

### 8. look at hubble traffic

Open hubble.IP_TO_REPLACE.nip.io
<p align="center"><img src="/image/hubble.png" width="40%" alt="hubble" /></p>

### 9. Hubble OTEL
Cilium provide an experimental Collector recevier "hubble"
let's modify the current traffic to:
- produce openTelemtry measurements from otel-demo 
- the signals will be sent to a new collector deployed in the kube-system namespace
- the new collector will forward the signals to our collector located in otel namespace.

#### 1. Deploy the new collector
```
kubectl apply -f kubernetes-manifests/openTelemetry-hubble.yaml
```
#### 2. Deploy the new side car collector in otel-demo
```
kubectl delete -f kubernetes-manifests/K8sdemo.yaml -n otel-demo
kubectl apply -f kubernetes-manifests/openTelemetry-sidecar_hubble.yaml
kubectl apply -f kubernetes-manifests/K8sdemo.yaml -n otel-demo
```

#### 3. Let's open Hubble
Open Hubble and look at the new traffic of the otel namespace.
We can see incoming traffic from kube-system and outgoing traffic to internet.
<p align="center"><img src="/image/hubb_otel.png" width="40%" alt="hubble" /></p>

#### 4. Deploy CiliumNetworkPolicy
```
kubectl apply -f cilium/CiliumNetworkPolicy_collector.yaml
```
#### 5.  Let's open Hubble

Open hubble.IP_TO_REPLACE.nip.io
<p align="center"><img src="/image/hubble.png" width="40%" alt="hubble" /></p>
