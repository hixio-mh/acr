# Running ACR Tasks on dedicated agent pools

## Introduction

AgentPool allows you to run tasks in the dedicated machine pool exclusively. Here are some key features.

* You can provision the agent pool into your vnet. The tasks running on the agent pool are able to securely access the resources in the vnet (eg, container registry, key vault, storage).
* You can scale in/out the agent pool on demand.
* You have more machine choices. Current preview release provides 3 tiers, S1 (2 cpu, 3G mem), S2 (4 cpu, 8G mem), and S4 (8 cpu, 16G mem). 
* You can create multiple agent pools to serve different types of workloads. 

AgentPool feature is currently previewed in WestUS2, SouthCentralUS, EastUS2 and EastUS. Please contact acrsup@microsoft.com to enable for your subscription.

## Prerequisites

* Please install Azure CLI __2.3.1__ or above.
* Please prepare a __premium__ container registry in the above preview regions.

## Create and manage an agent pool

* Create an agent pool of tier S2 (4 cpu/instance).

```sh
az acr agentpool create \
    -r mypremiumregistry \
    -n myagentpool \
    --tier S2
```

* The above command will create a new agent pool with 1 instance. You can scale out the agent pool to have more instances or scale in to 0.

```sh
az acr agentpool update \
    -r mypremiumregistry \
    -n myagentpool \
    -c 2
```

## Create an agent pool in a custom vnet

* If you enable the network security rule on the vnet, as a managed service, AgentPool requires unrestricted access to several IP addresses in the Azure data center. To allow communication with these IP addresses, update any existing network security groups or user-defined routes with following outbound network rule. 

| Direction | Protocol | Source         | Source Port | Destination          | Dest Port | Used    |
|-----------|----------|----------------|-------------|----------------------|-----------|---------|
| Outbound  | TCP      | VirtualNetwork | Any         | Storage              | 443       | Default |
| Outbound  | TCP      | VirtualNetwork | Any         | EventHub             | 443       | Default |
| Outbound  | TCP      | VirtualNetwork | Any         | AzureActiveDirectory | 443       | Default |
| Outbound  | TCP      | VirtualNetwork | Any         | AzureMonitor         | 443       | Default |

    [NOTE] If your Tasks require additional resources from public internet, eg, if you run docker build task that needs to pull the base images from DockerHub or restore Nuget package, please add the corresponding rules. 

* Create an agent pool in the vnet.

```sh
subnet=$(az network vnet subnet show \
        -g myvnetresourcegroup \
        --vnet-name myvnetname \
        -n mysubnetname \
        --query id -o tsv)

az acr agentpool create \
    -r mypremiumregistry \
    -n myagentpool \
    --tier S2 \
    --subnet-id $subnet
```

## Schedule runs on the agent pool

* Schedule a quick run on the agent pool.

```sh
az acr build \
    -r mypremiumregistry \
    --agent-pool myagentpool \
    -t myimage:mytag \
    -f Dockerfile \
    https://github.com/Azure-Samples/acr-build-helloworld-node.git
```

* Create a recurring task on the agent pool.

```sh
az acr task create \
    -r mypremiumregistry \
    -n mytask \
    --agent-pool myagentpool \
    -t myimage:mytag \
    -f Dcokerfile \
    -c https://github.com/Azure-Samples/acr-build-helloworld-node.git \
    --commit-trigger-enabled false
    
az acr task run \
    -r mypremiumregistry \
    -n mytask
```

* Query the agent pool queue status (current scheduled runs on the agent pool).

```sh
az acr agentpool show \
    -r mypremiumregistry \
    -n myagentpool \
    --queue-count
```



## Preview limitation

* Windows is not supported.
* For each registry, the default total cpu quota of all agent pools is 16. Please contact acrsup@microsoft.com if you want to bump the quota.