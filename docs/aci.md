# Azure Container Instances and serverless containers with Virtual Kubelet
Azure Container Instances are top level resource in Azure so you don't have to create (and pay for) any VM, just create container directly and pay by second. In this demo we will deploy Microsoft SQL Server in Linux container.

- [Azure Container Instances and serverless containers with Virtual Kubelet](#azure-container-instances-and-serverless-containers-with-virtual-kubelet)
    - [Create resource group](#create-resource-group)
    - [Run SQL on Linux container](#run-sql-on-linux-container)
    - [Connect to SQL](#connect-to-sql)
    - [Running tasks in Azure Container Instance](#running-tasks-in-azure-container-instance)
    - [Delete container](#delete-container)
- [Virtual Kubelet - orchestrating containers without actual nodes](#virtual-kubelet---orchestrating-containers-without-actual-nodes)
    - [Using Virtual Kubelet with Azure Container Instances](#using-virtual-kubelet-with-azure-container-instances)
        - [Create resource gorup](#create-resource-gorup)
        - [Assign RBAC to default service account](#assign-rbac-to-default-service-account)
        - [Deploy ACI Connector](#deploy-aci-connector)
        - [Deploy pod to ACI](#deploy-pod-to-aci)
    - [Using Virtual Kubelet for IoT devices](#using-virtual-kubelet-for-iot-devices)
    - [Clean up](#clean-up)

**This section has not yet been updated for AKS**


## Create resource group
```
az group create -n aci-group -l westeurope
```

## Run SQL on Linux container
```
az container create -n mssql \
    -g aci-group \
    --cpu 2 \
    --memory 4 \
    --ip-address public \
    --port 1433 \
    -l westeurope \
    --image microsoft/mssql-server-linux \
    -e 'ACCEPT_EULA=Y' 'SA_PASSWORD=my(!)Password' 

export sqlip=$(az container show -n mssql -g aci-group --query ipAddress.ip -o tsv)
watch az container logs -n mssql -g aci-group
```

## Connect to SQL
```
sqlcmd -S $sqlip -U sa -P 'my(!)Password' -Q 'select name from sys.databases'
```

## Running tasks in Azure Container Instance
ACI can be used to execute tasks packaged in container, for example transform some data, prepare calculation or do maintenance. Key is to have this task as entry process and configure ACI to not restart process when it exits with 0 code. Instead when task is over Azure will delete this container so you stop paying for it.

Let's use simplistic alpine container and our "task" will be simulated by sleeping for 20 seconds. Instead of default restart policy (Always, so it keeps restarting process) we configure this to OnFailure, so when our process returns 0, container will be terminated.

```
az container create -n containertask \
    -g aci-group \
    --cpu 1 \
    --memory 1 \
    -l westeurope \
    --image alpine \
    --command-line 'sleep 20' \
    --restart-policy OnFailure
    
az container show -n containertask -g aci-group --query containers[].instanceView.currentState
```

## Delete container
```
az container delete -n mssql -g aci-group -y
az container delete -n containertask -g aci-group -y
az group delete -n aci-group -y
```

# Virtual Kubelet - orchestrating containers without actual nodes
Kubernetes comes with scheduler and managed nodes to deploy Pods however there are situations when you want to leverage Kubernetes orchestration capabilities without deploying to managed nodes. Two examples are Azure Container Instances (deployment of containers directly to cloud without need for any underlaying nodes) and IoT devices (run containers in IoT device, for example using Azure IoT edge to deploy Azure Functions, Azure Stream Analytics or Machine Learning).

Virtual Kubelet is virtual node that simulates unlimited pool of cloud containers (ACI) or pool of IoT devices (Azure IoT Hub). Please note this technology is in alpha stage.

## Using Virtual Kubelet with Azure Container Instances

As show in first example Azure Container Instance can be used as top level resources. Azure can run containers directly without need to do so in VMs. There is experimental connector available so that Azure behaves like Kubernetes node with infinite capacity. In this demo we will install this connector and schedule pod to run on this infinite node reprezentaion of Azure.

### Create resource gorup
```
az group create -n aci-connect -l eastus
```

### Assign RBAC to default service account
This step is only for RBAC enabled cluster, no AKS

```
kubectl create -f clusterRoleBindingService.yaml
```

### Deploy ACI Connector
TO BE UPDATED FOR AKS

```
kubectl create -f aciConnector.yaml
kubectl get nodes
```

### Deploy pod to ACI
```
kubectl create -f podACI.yaml
kubectl get pods -o wide
az container list -g aci-connect -o table
```
Connect to IP on port 8080

## Using Virtual Kubelet for IoT devices

TBD

## Clean up
```
kubectl delete -f podACI.yaml
kubectl delete -f aciConnector.yaml
kubectl delete -f clusterRoleBindingService.yaml
```