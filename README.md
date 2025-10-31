# 🌩️ GitOps Project — Azure AKS + Application Gateway + ArgoCD

This repository demonstrates a GitOps setup with Azure AKS, Application Gateway Ingress Controller (AGIC), and ArgoCD, deploying a multi-arch 2048 game app.

## ⚙️ Prerequisites

Ubuntu VM (or cloud VM with internet access)

Azure CLI installed

curl, wget, git installed

## ⚙️ 1. Install Azure CLI

Refer: [How to install the Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
az --version
```

---

## 🔐 2. Login to Azure

```bash
az login
```

---

## ☸️ 3. Create Resource Group and AKS Cluster

```bash
az group create -n myResourceGroup -l eastus

az aks create -g myResourceGroup -n myAksCluster \
--node-count 2 \
--enable-cluster-autoscaler \
--min-count 2 --max-count 6 \
--node-vm-size Standard_B2ms \
--generate-ssh-keys \
--kubernetes-version 1.33.0
```

---

## 🧰 4. Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
kubectl version --client
```

---

## 🔗 5. Connect to Cluster

```bash
az aks get-credentials -g myResourceGroup -n myAksCluster
kubectl get nodes
```
---

## 🌐 6. Enable Application Gateway Ingress Controller (AGIC)

```bash
# Get the node resource group for your AKS cluster
NODE_RG=$(az aks show -g myResourceGroup -n myAksCluster --query "nodeResourceGroup" -o tsv)

# Get the name of the Application Gateway in that resource group
APPGW_NAME=$(az network application-gateway list -g $NODE_RG --query "[0].name" -o tsv)

# Get the VNet name
VNET_NAME=$(az network vnet list -g $NODE_RG --query "[0].name" -o tsv)

# Get the subnet ID for aks-appgateway
SUBNET_ID=$(az network vnet subnet show \
  --resource-group $NODE_RG \
  --vnet-name $VNET_NAME \
  --name aks-appgateway \
  --query "id" -o tsv)

# Enable AGIC using the existing subnet
az aks enable-addons \
  --addons ingress-appgw \
  --resource-group myResourceGroup \
  --name myAksCluster \
  --appgw-subnet-id $SUBNET_ID

# Check that the AGIC pod is running
kubectl get pods -n kube-system | grep appgw

⚠️ Verify AGIC Identity Role Assignment

# Get AGIC client ID
AGIC_CLIENT_ID=$(az aks show \
  --resource-group myResourceGroup \
  --name myAksCluster \
  --query "addonProfiles.ingressApplicationGateway.identity.clientId" -o tsv)

# List role assignments for AGIC on the Application Gateway
az role assignment list \
  --assignee $AGIC_CLIENT_ID \
  --scope /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/$NODE_RG/providers/Microsoft.Network/applicationGateways/$APPGW_NAME \
  --query "[].roleDefinitionName" -o tsv

# If the Network Contributor role is missing, assign it
az role assignment create \
  --assignee $AGIC_CLIENT_ID \
  --role "Contributor" \
  --scope /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/$NODE_RG/providers/Microsoft.Network/applicationGateways/$APPGW_NAME

```

---

## 🚀 7. Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```
---
## 🧭 8. Install ArgoCD with Helm

```bash
kubectl create namespace argocd

helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd --namespace argocd
```
Expose ArgoCD server externally:
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"LoadBalancer"}}'
kubectl get svc -n argocd
```

---

## 🔑 9. Get ArgoCD Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
Expose ArgoCD server externally:
```bash
http://<ARGOCD-EXTERNAL-IP>
```
---

## 📦 10. Clone and Deploy Your App via GitOps

```bash
git clone https://github.com/gitbloodprince/20148.git
cd 20148
```
Apply manifests manually (optional — ArgoCD will sync automatically):
```bash
kubectl apply -f app-deployment.yml
kubectl apply -f app-ingress.yml
```
Check deployment::
```bash
kubectl get pods -n store-ns
kubectl get svc -n store-ns
kubectl get ingress -n store-ns
```
---

## 🌍 11. Access Your Application
Get the external IP of the ingress:
```bash
kubectl get ingress store-ns-ingress -n store-ns
```
Visit:
```bash
http://<EXTERNAL-IP>
```
Optional: If you have a domain, create an A record pointing to the external IP.

---

## 12. Cleanup Resources

```bash
kubectl delete -f app-deployment.yml
kubectl delete -f app-ingress.yml
az aks delete -g 20148-argoCD -n myAksCluster --yes --no-wait
az group delete -n 20148-argoCD --yes --no-wait
```

