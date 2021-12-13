# Intro

This repo contains instructions for installing KEDA, configuring two use cases around HTTP based scaling using Prometheus queries and Queue count scaling using Azure Service Bus. The configuration assumes the application will be deployed into the keda-demo namespace, but feel free to change to match your scenario.

## (Optional) PREREQ: Create AKS Cluster

Create an AKS Cluster or choose to re-use an existing cluster. The following steps will create a cluster.

### Set Envrionment variables for deployment
```
export RG_NAME=KEDA_DEMO
export REGION=eastus
export CLUSTER_NAME=KEDA-AKS
export VNET=AKS_VNET
export SUBNET=AKS_SUBNET
```

### Create RG & Cluster
```
az group create --name ${RG_NAME} --location ${REGION}

az network vnet create -g ${RG_NAME} -n ${VNET} --address-prefix 10.0.0.0/16 \
    --subnet-name ${SUBNET} --subnet-prefix 10.0.0.0/24 --location ${REGION}

az aks create --resource-group ${RG_NAME} --name ${CLUSTER_NAME} --node-count 1 --enable-addons monitoring --generate-ssh-keys \
--nodepool-name np1  --location ${REGION} --enable-managed-identity

az aks get-credentials -g keda_demo -n keda-aks
```

## (Optional) PREREQ: Deploy workload to use

This guide assumes you have a workload that can be tested. If not, the following links can be used to deploy a sample workload into the keda-demo namespace.

1. Install Istio: https://istio.io/latest/docs/setup/install/
1. Deploy sample application: https://istio.io/latest/docs/setup/getting-started/#bookinfo
1. Deploy ingress: https://istio.io/latest/docs/setup/getting-started/#ip
1. Install an application that can read messages off a queue. This YAML will deploy an application that reads messages off a queue (Update env. variables): https://github.com/kedacore/sample-dotnet-worker-servicebus-queue/blob/main/deploy/connection-string/deploy-app.yaml


## Install Keda to the cluster

Keda can be installed using YAML or Helm chart, the below commands use a HELM  to install Keda components to the keda namespace. For other install options, check: https://keda.sh/docs/2.2/deploy/

1. Execute the following to deploy Keda to the Keda namespace using Helm:

```
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
kubectl create namespace keda
helm install keda kedacore/keda --version 2.5 --namespace keda
```

1. (Optional) You can observe the keda components that have been installed:
```
kubectl get pods -n keda
kubectl get crd | grep -i keda
```


## Configure Keda ScaledOjbect

This repository has a keda-config folder with 3 different ScaledObject YAML files:
- **keda-http-prometh.yaml** - This is an example of scaling by metrics retreived from a prometheus query.
- **keda-http-prometh2.yaml** - This is similar to the above YAML file with the differences being multiple triggers have been added including CPU and memory.
- **keda-servicebus-constr.yaml** - This is an example of a service bus  trigger scaling a deployment. It has a Secret and Auth object.

### Configure and Deploy Http based scaling
Update config/keda-http-prometh.yaml
1. Change the deployment name value in **ScaleTargetRef** to match the desired deployment you are targeting to be scaled.
1. In the Prometheus Trigger Section, Update the following fields:
   - **serverAddress**: This is the service hostname/IP address of your prometheus serve.
   - **query**: Update this to match the query of your desired service and validate it returns RPS.
2. If neccessary, update the namespace to where the target deployment lives.
3. (Optional) Adjust any scaling parameters in the YAML as you see fit. The existing configuration will scale one additional replica for every increment of 10 which the Prometheus RPS query returns.
4. Deploy the configuration:
```
kubectl apply -f keda-config/keda-http-prometh.yaml
```
5. (Optional) Observe the keda scaled object deployed along with the HPA that Keda manages.
```
kubectl get scaledobjects -n keda-demo
kubectl get hpa -n keda-demo
```


### Configure and Deploy Azure service bus queue based scaling
You will need an azure service bus queue for this example.

1. From the Azure Portal navigate to your Azure Service Bus, Go to your desired queue and click the "Shared Access Policies" blade. Add a "manage" connection string for the particular queue. Select the newly created policy and copy the Primary Connection String. A Manage connection string is required to retrieve Service Bus Metrics.
2. Convert this connection string to base64: Ex:
```
 echo ConnectionString | base64
```
3. Edit the keda-servicebus-constr.yaml and update the following:
   - ScaledObject: **QueueName**: Update to match the queue name.
   - Secret: **servicebus-connectionstring**: Your base64 service bus connection string.
4. Optionally update any scaling configuration.
4. Deploy the ScaledObject for service bus deployment
```
kubectl apply -f keda-config/keda-servicebus-constr.yaml
```



## (Optional) Load Testing the http workload with K6
1. Install K6 
  ```
  brew install k6
  ```
2. Inspect the k6config/script.js file
3. Update the hostname to match your applications endpoint.
4. Optionally update any other configuration as needed. This file has virtual user count of 5 users and will perform 30 requests per second for duration of 1 minute.
5. Execute K6 load testing
 ``` 
 k6 run ./k6config/script.js 
 ```
4. Observe pods autoscaling:
```
 watch -n .1 kubectl get pods -n keda-demo 
 ```


## (Optional) Load Testing with client app to generate queue messages

1. You can use a sample application to process queue messages: https://github.com/kedacore/sample-dotnet-worker-servicebus-queue/tree/main/src/Keda.Samples.Dotnet.OrderProcessor
   
## (Optional) Load Testing with Azure Load Testing service

TBD


## Keda Trobuleshooting

1. Get the pod name of the keda operator and retrieve the logs.
```
kubectl get pods -n keda 
kubectl logs -n keda pod/{OperatorPodName}
```
