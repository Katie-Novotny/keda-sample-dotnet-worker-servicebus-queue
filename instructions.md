
vnet_name=myVnet
subnet_name=myAKSSubnet
az network vnet create \
    --name $vnet_name \
    --address-prefixes 10.0.0.0/8 \
    --subnet-name $subnet_name \
    --subnet-prefix 10.240.0.0/16

virtual_subnet_name=myVirtualNodeSubnet
az network vnet subnet create \
    --vnet-name $vnet_name \
    --name $virtual_subnet_name \
    --address-prefixes 10.241.0.0/16


subnet_id=$(az network vnet subnet show --vnet-name $vnet_name --name $subnet_name --query id -o tsv)
aks_cluster_name=keda-virt-node
az aks create \
          --name $aks_cluster_name \
          --node-count 1 \
          --attach-acr $acr_name \
          --network-plugin azure \
          --vnet-subnet-id $subnet_id \
          --enable-addons monitoring \
          --generate-ssh-keys \
          --enable-cluster-autoscaler \
          --min-count 1 \
          --max-count 3

az aks enable-addons \
    --name $aks_cluster_name \
    --addons virtual-node \
    --subnet-name virtual_subnet_name