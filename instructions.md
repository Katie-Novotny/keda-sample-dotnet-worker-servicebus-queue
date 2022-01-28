
vnet_name=newVnet
subnet_name=newAKSSubnet
az network vnet create \
    --name $vnet_name \
    --address-prefixes 10.0.0.0/8 \
    --subnet-name $subnet_name \
    --subnet-prefix 10.240.0.0/16

virtual_subnet_name=newVirtualNodeSubnet
az network vnet subnet create \
    --vnet-name $vnet_name \
    --name $virtual_subnet_name \
    --address-prefixes 10.241.0.0/16


subnet_id=$(az network vnet subnet show --vnet-name $vnet_name --name $subnet_name --query id -o tsv)
aks_cluster_name=vn-take3
acr_name=kedavirtualnode

az identity create  -n vn-take3-user

az aks create \
          --name $aks_cluster_name \
          --node-count 1 \
          --assign-identity $user_resource_id \
          --attach-acr $acr_name \
          --network-plugin azure \
          --vnet-subnet-id $subnet_id \
          --enable-addons monitoring \
          --generate-ssh-keys \
          --enable-cluster-autoscaler \
          --min-count 1 \
          --max-count 3

az aks addon enable \
    --name $aks_cluster_name \
    --addon virtual-node \
    --subnet-name $virtual_subnet_name

az aks disable-addons \
    --name $aks_cluster_name \
    --addons virtual-node 

az aks nodepool list --cluster-name $aks_cluster_name



vnetId=$(az network vnet show --name newVnet --query id -o tsv)

az role assignment create --assignee $principalId --scope $vnetId --role "Network Contributor"

az aks addon show --addon virtual-node  --name $aks_cluster_name

az aks addon update --addon virtual-node --name $aks_cluster_name --subnet-name $virtual_subnet_name
                  
project_name=keda-virtual-node
servicebus_namespace=$project_name
az servicebus namespace create --name $servicebus_namespace --sku basic
queue_name=orders
az servicebus queue create --namespace-name $servicebus_namespace --name $queue_name

az servicebus queue authorization-rule create --namespace-name $servicebus_namespace --queue-name $queue_name --name order-consumer --rights Listen

connection_string=$(az servicebus queue authorization-rule keys list --namespace-name $servicebus_namespace --queue-name $queue_name --name order-consumer --query primaryConnectionString -o tsv)

export connection_string_base64=$(echo -n $connection_string | base64   | tr -d '\n')

kubectl create namespace keda-dotnet-sample

kubectl create secret generic secrets-order-consumer --from-literal=servicebus-connectionstring=$connection_string -n keda-dotnet-sample

kubectl apply -f deploy/connection-string/deploy-app.yaml --namespace keda-dotnet-sample
kubectl delete -f deploy/connection-string/deploy-app.yaml --namespace keda-dotnet-sample

Install KEDA
helm repo update
helm repo add kedacore https://kedacore.github.io/charts
kubectl create namespace keda
helm install keda kedacore/keda --namespace keda

kubectl delete -f deploy/connection-string/deploy-autoscaling.yaml --namespace keda-dotnet-sample
kubectl apply -f deploy/connection-string/deploy-autoscaling.yaml --namespace keda-dotnet-sample

kubectl describe scaledobject order-processor-scaler -n keda-dotnet-sample

kubectl get deployments --namespace keda-dotnet-sample -o wide

az servicebus queue authorization-rule create --namespace-name $servicebus_namespace --queue-name $queue_name --name keda-monitor-send --rights Listen Send

connection_string=$(az servicebus queue authorization-rule keys list --namespace-name $servicebus_namespace --queue-name $queue_name --name keda-monitor-send --query primaryConnectionString -o tsv)



echo -n $connection_string | base64   | tr -d '\n'

echo -n 'Endpoint=sb://keda-virtual-node.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;

kubectl create secret generic secrets-order-autoscaler --from-literal=servicebus=$connection_string -n keda-dotnet-sample


update "src/Keda.Samples.Dotnet.OrderGenerator/Program.cs" with "queue name" and "connection string"

in powershell terminal, run: dotnet run --project .\src\Keda.Samples.Dotnet.OrderGenerator\Keda.Samples.Dotnet.OrderGenerator.csproj

kubectl get secret secrets-order-consumer -o jsonpath='{.data}'


identity_name=$project_name
az identity create --name $identity_name 

app_identity_clientid=$(az identity show --name $identity_name --query clientId -o tsv)
app_identity_resourceid=$(az identity show --name $identity_name --query Id -o tsv)
subscription_id=$(az account show --query id -o tsv)
scope=/subscriptions/$subscription_id/resourceGroups/keda-virtual-node/providers/Microsoft.ServiceBus/namespaces/$servicebus_namespace
az role assignment create --role 'Azure Service Bus Data Receiver' --assignee $app_identity_id --scope $scope

autoscaler_identity=autoscaler-identity
az identity create --name $autoscaler_identity
autoscaler_identity_clientid=$(az identity show --name $autoscaler_identity --query clientId -o tsv)

kubectl apply -f https://raw.githubusercontent.com/Azure/aad-pod-identity/v1.8.4/deploy/infra/deployment-rbac.yaml

kubectl apply -f deploy/managed-identity/deploy-app-with-managed-identity.yaml --namespace keda-dotnet-sample
kubectl delete -f deploy/managed-identity/deploy-app-with-managed-identity.yaml --namespace keda-dotnet-sample

kubectl apply -f deploy/managed-identity/deploy-autoscaling-infrastructure.yaml --namespace keda-dotnet-sample

helm install keda kedacore/keda --set podIdentity.activeDirectory.identity=app-autoscaler --namespace keda

az role assignment create --role 'Azure Service Bus Data Owner' --assignee <scaler-identity-id> --scope /subscriptions/<subscription-id>/resourceGroups/<resource-group-name>/providers/Microsoft.ServiceBus/namespaces/<namespace-name>

az role assignment create --role 'Azure Service Bus Data Owner' --assignee $autoscaler_identity_clientid --scope $scope

kubectl apply -f deploy/managed-identity/deploy-app-autoscaling.yaml --namespace keda-dotnet-sample

kubectl delete -f deploy/managed-identity/deploy-app-autoscaling.yaml --namespace keda-dotnet-sample

kubectl get po -n keda-dotnet-sample

kubectl logs order-processor-7b84ff997-qsd75 -n keda-dotnet-sample -f

k logs mic-68c7ccd865-6s6bm

k logs nmi-ctb2m


# Output the OIDC issuer URL

az feature register --name EnableOIDCIssuerPreview --namespace Microsoft.ContainerService
# Install the aks-preview extension
az extension add --name aks-preview

# Update the extension to make sure you have the latest version installed
az extension update --name aks-preview

az aks update -n $aks_cluster_name --enable-oidc-issuer

az aks show  --name $aks_cluster_name --query "oidcIssuerProfile.issuerUrl" -o tsv

export AZURE_TENANT_ID="$(az account show -s $subscription_id --query tenantId -otsv)"

helm repo add azure-workload-identity https://azure.github.io/azure-workload-identity/charts
helm repo update
helm install workload-identity-webhook azure-workload-identity/workload-identity-webhook \
   --namespace azure-workload-identity-system \
   --create-namespace \
   --set azureTenantID="${AZURE_TENANT_ID}"


curl -L https://github.com/a8m/envsubst/releases/download/v1.2.0/envsubst-`uname -s`-`uname -m` -o envsubst
chmod +x envsubst
sudo mv envsubst /usr/local/bin

curl -s https://github.com/Azure/azure-workload-identity/releases/download/v0.7.0/azure-wi-webhook.yaml | envsubst | kubectl apply -f -

cat azure-wi-webhook.yaml | envsubst | kubectl apply -f -

go install github.com/Azure/azure-workload-identity/cmd/azwi@v0.7.0


## after creating cluster in Azure Portal

az aks get-credentials --resource-group vn-take2 --name vn-take2
connection_string=$(az servicebus queue authorization-rule keys list --namespace-name $servicebus_namespace --queue-name $queue_name --name order-consumer --query primaryConnectionString -o tsv)
kubectl create namespace keda-dotnet-sample

kubectl create secret generic secrets-order-consumer --from-literal=servicebus-connectionstring=$connection_string -n keda-dotnet-sample

kubectl apply -f deploy/connection-string/deploy-app.yaml --namespace keda-dotnet-sample
kubectl delete -f deploy/connection-string/deploy-app.yaml --namespace keda-dotnet-sample

Install KEDA
helm repo update
helm repo add kedacore https://kedacore.github.io/charts
kubectl create namespace keda
helm install keda kedacore/keda --namespace keda

kubectl delete -f deploy/connection-string/deploy-autoscaling.yaml --namespace keda-dotnet-sample
kubectl apply -f deploy/connection-string/deploy-autoscaling.yaml --namespace keda-dotnet-sample

kubectl describe scaledobject order-processor-scaler -n keda-dotnet-sample

kubectl get deployments --namespace keda-dotnet-sample -o wide

watch kubectl get pods --namespace keda-dotnet-sample -o wide

update "src/Keda.Samples.Dotnet.OrderGenerator/Program.cs" with "queue name" and "connection string"

in powershell terminal, run: dotnet run --project .\src\Keda.Samples.Dotnet.OrderGenerator\Keda.Samples.Dotnet.OrderGenerator.csproj




