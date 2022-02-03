# Leveraging Node Pools for Cluster Autoscaling in AKS

## Important References:
1. [AKS Core Concepts like Node Pools](https://docs.microsoft.com/en-us/azure/aks/concepts-clusters-workloads)
2. [Using the Cluster Autoscaler in AKS](https://docs.microsoft.com/en-us/azure/aks/cluster-autoscaler)
3. [Creating and Managing Multiple AKS Node Pools](https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools)
4. [Creating AKS Spot Node Pools](https://docs.microsoft.com/en-us/azure/aks/spot-node-pool)
5. [Advanced Cluster Scheduling on AKS](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-advanced-scheduler)
6. [az aks nodepool cli reference](https://docs.microsoft.com/en-us/cli/azure/aks/nodepool?view=azure-cli-latest#az_aks_nodepool_update)

## Prerequisites:
1. Install az cli
2. Install kubectl cli
3. Clone the repo locally

## Demo Steps:

1. Setup a `.env` file in the root of the directory with the following values - we will use these as environment variables for the following commands:

```
# Azure resoure group settings
RG_NAME=demo-aks-cluster                  # Resource group name
LOCATION=eastus                           # az account list-locations --query '[].name'

# AKS deployment settings
AKS_NAME=demo-aks                         # AKS Cluster Name, ie: 'demo-aks'
AKS_SYSTEM_NODE_COUNT=2                   # Initial Node Count in Cluster
```

2. Run the following in your terminal to set the env vars:

```
set -a
source .env
set +a
```

3. Create a resource group and AKS cluster with a default system node pool:

```
# Create a resource group
az group create --name $RG_NAME --location $LOCATION

# Create a basic AKS cluster
# enable monitoring to view in log analytics with container insights
az aks create \
    --resource-group $RG_NAME \
    --name $AKS_NAME \
    --node-count $AKS_SYSTEM_NODE_COUNT \
    --enable-addons monitoring

# Obtain Credentials
az aks get-credentials --resource-group $RG_NAME --name $AKS_NAME
```

4. View the system node pool that is provisioned:

```
# View system node pool (default)
az aks nodepool list --resource-group $RG_NAME --cluster-name $AKS_NAME -o table
```

>Note: View the mode there and you should see 'System' reflecting that this is the system node pool

5. Add a user node pool that mocks the scenario of creating a GPU-Enabled node pool for specific workloads (we will use a non-GPU vm-size for demo purposes, but conceptually the same):

```
# Add a user node pool
# mark this node pool as our gpu node pool
# use node taints and labels for advanced scheduling 
### notice the vm size is not actually a gpu-enabled node (demo purpose only)
az aks nodepool add \
    --resource-group $RG_NAME \
    --cluster-name $AKS_NAME \
    --name gpunodepool \
    --node-count 1 \
    --node-taints sku=gpu:NoSchedule \
    --node-vm-size Standard_DS2_v2 \
    --labels sku=gpu
```

>Note: The specified node taint will allow these nodes to only run pods with the appropriate tolerations. Additionally, the labels will allow for pods to specify these in their node selector.

6. Run the following command to now see the two node pools:

```
az aks nodepool list --resource-group $RG_NAME --cluster-name $AKS_NAME -o table
```

>Note: View the mode there and you should see 'User' reflecting that this is a user node pool

7. Enable Cluster Autoscaling for the GPU-Node Pool (we will enable the min count to go all the way to 0, so that we only provision these nodes when workloads require them):

```
# enable cluster autoscaling
# notice scaling to 0
az aks nodepool update \
--resource-group $RG_NAME \
--cluster-name $AKS_NAME \
--name gpunodepool \
--enable-cluster-autoscaler \
--min-count 0 \
--max-count 5
```

8. To make it easier to view scaling activities, enable the following cluster profile settings (these should be determined based on workload/scaling requirements in reality):

```
# specify cluster profile
az aks update \
    --resource-group $RG_NAME \
    --name $AKS_NAME \
    --cluster-autoscaler-profile scale-down-delay-after-add=1m scale-down-unneeded-time=1m ok-total-unready-count=10 max-total-unready-percentage=100
```

9. Apply the example template in the repo which will provision 5 instances of a pod that requires more resources than immediately available, which will kick off the cluster autoscaling.

```
# deploy the sample workload
kubectl apply -f template.yaml

# view pods scheduling on the different nodes
kubectl get pods -o wide
```

>Note: Notice how the pods are only being scheduled on the GPU Nodes - this is done using the labels and node selectors created on the node pool and within the deployment template.

10. View the events in AKS to see the nodes spinning up to support the deployment. Use the following two commands to see these activities occurring:

```
# view pods scheduling
kubectl get pods -o wide

# view scaling of node pool
kubectl get events --sort-by=lastTimestamp --watch
```

11. Delete all the resources for the workload and then watch to see that the GPU node pool then scales down all the way to 0:

```
# delete resources
kubectl delete deployment,svc nginx

# view scaling
kubectl get events --sort-by=lastTimestamp --watch

# view count of nodes for the node pool
az aks nodepool list --resource-group $RG_NAME --cluster-name $AKS_NAME -o table

# view nodes
kubectl get nodes
```

12. If you redeploy the workload, you will see the GPU nodes spin up again:

```
# reapply the workload to see the gpu nodes get deployed
kubectl apply -f template.yaml

# view pods scheduling on the different nodes
kubectl get pods -o wide

# view scaling
kubectl get events --sort-by=lastTimestamp --watch
```

## Clean Up Resources:
1. To delete everything provisioned, delete the Resource Group:

```
az group delete --resource-group $RG_NAME
```
