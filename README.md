# AI Agent Deployment on Azure

## 1. Overview

This document provides detailed procedures for IT staff to deploy an AI Agent solution within an Azure environment. It focuses specifically on the technical steps required to deploy the solution using the electronic information provided by the vendor.

### 1.1 Context and Applicability

This deployment approach is specifically designed for environments with restricted external network access, which is common in large enterprises, financial institutions, healthcare organizations, and government agencies. Key considerations addressed by this guide include:

- **Air-gapped or restricted networks**: Many enterprise environments restrict direct access to external resources like GitHub, Docker Hub, or public package repositories for security reasons.
- **Compliance requirements**: Organizations in regulated industries must follow strict protocols for introducing software into their environments, often requiring pre-approved, pre-scanned packages.
- **Change management**: Enterprise IT typically requires all deployments to go through formal change control processes, with software components being reviewed and approved before installation.
- **Security protocols**: The approach allows for thorough security scanning of container images and dependencies prior to deployment.
- **Offline deployment capability**: The procedure enables deployment in environments with no internet access by using pre-packaged container images and dependencies.

This method uses TAR files for container images and pre-packaged Kubernetes manifests, eliminating the need for direct access to public repositories during deployment. The guide outlines how to securely transport these components into restricted environments and deploy them following enterprise security best practices.

## 2. Provided Electronic Information

The vendor will provide the following electronic information for deployment:

| Item | Description | Format |
|------|-------------|--------|
| Docker Images | Pre-built Docker container images for the AI Agent | TAR files |
| Kubernetes Manifests | YAML files for AKS deployment | ZIP archive (k8s-manifests.zip) |
| Documentation | Technical documentation and API references | PDF/Markdown |

The Docker images provided will include:
- agent-ui.tar (Frontend UI)
- agent-apis.tar (Backend API services)
- agent-worker.tar (Background task worker)

### 2.1 Secure Transfer and Verification

Since this deployment approach is designed for environments with strict security policies, the following steps are recommended when receiving the electronic information:

1. **Secure transfer protocol**: Ensure all files are transferred using secure methods (SFTP, encrypted drives, or secure file sharing services approved by your organization).

2. **File integrity verification**: Verify SHA-256 checksums of all received files against those provided by the vendor to ensure files haven't been tampered with during transfer.

3. **Security scanning**: Run all received files through your organization's security scanning tools before proceeding with deployment.

4. **Image verification**: Consider using container scanning tools to verify the security posture of the Docker images before importing them to your container registry.

5. **Approval documentation**: Maintain records of security scans and approvals as required by your organization's change management processes.

## 3. Deployment Prerequisites

Before beginning deployment, ensure the following prerequisites are met:

- [ ] Azure subscription with appropriate permissions to create resources
- [ ] Azure CLI installed and configured
- [ ] Kubernetes CLI (kubectl) installed and configured
- [ ] Access to import Docker images from provided TAR files
- [ ] Network access configured for required Azure services
- [ ] Databricks workspace accessible for data integration (if needed)

### 3.1 Internal Network Requirements

For environments with restricted external access, ensure the following:

- [ ] Workstation preparing the deployment has access to Azure services (direct or via proxy)
- [ ] If using a proxy for Azure CLI access, ensure proper proxy configuration
- [ ] Appropriate firewall rules to allow communication between deployment components
- [ ] Internal DNS resolution for Azure services
- [ ] If completely air-gapped, ensure all required tools and dependencies are pre-installed
- [ ] Proxy/firewall exceptions for the specific Azure endpoints needed during deployment

### 3.2 Security and Compliance Considerations

- [ ] Deployment plan reviewed and approved by security team
- [ ] Required security controls identified and implemented
- [ ] Compliance requirements for data storage and processing identified
- [ ] Role-based access controls (RBAC) plan established
- [ ] Encryption requirements for data at rest and in transit addressed
- [ ] Logging and monitoring requirements identified

### 3.1 Environment Variables

Set the following environment variables for convenience. The values provided are suggestions that should be adjusted according to your organization's naming conventions and requirements:

```bash
# Resource Group and Location
export RESOURCE_GROUP=ai-agent-rg
export LOCATION=eastus

# Azure Container Registry
export ACR_NAME=aiagentacr

# AKS Cluster
export AKS_CLUSTER=ai-agent-aks
export AKS_NODE_SIZE=Standard_D8s_v3
export AKS_NODE_COUNT=3

# Azure API Management
export APIM_NAME=ai-agent-apim

# Azure OpenAI
export OPENAI_NAME=ai-agent-openai
export OPENAI_ENDPOINT=https://${OPENAI_NAME}.openai.azure.com/

# PostgreSQL Database
export DB_SERVER=ai-agent-postgres
export DB_USERNAME=agentadmin
export DB_PASSWORD=$(openssl rand -base64 16)
export DB_NAME=agentdb
export VECTOR_DB_NAME=vectordb

# Virtual Network
export VNET_NAME=ai-agent-vnet
export SUBNET_AKS=aks-subnet
export SUBNET_APIM=apim-subnet

# Databricks (if needed)
export DATABRICKS_WORKSPACE=ai-agent-databricks
```

These variables will be used throughout the deployment process to ensure consistency and simplify commands.

## 4. Detailed Deployment Procedure

### 4.0 Deployment Overview

The deployment process will follow these steps:

1. Create Azure Container Registry (ACR)
2. Import Docker images to ACR
3. Create Azure OpenAI service
4. Set up Azure Database for PostgreSQL
5. Create AKS cluster
6. Configure required secrets
7. Deploy application to AKS
8. Configure API Management
9. Configure Databricks integration (if needed)

#### 4.0.1 Why This Approach Works for Restricted Environments

This deployment approach offers several advantages for restricted environments:

- **Self-contained resources**: All container images are pre-built and provided as TAR files, removing dependencies on public container registries.
- **Controlled deployment**: The Kubernetes manifests are pre-packaged, eliminating the need to clone public repositories.
- **Isolation capabilities**: All resources are deployed within a private network with minimal external dependencies.
- **Customizable permissions**: The deployment uses Azure's native RBAC mechanisms to implement principle of least privilege.
- **Audit-friendly**: All deployment steps can be logged and audited for compliance purposes.

First, extract the Kubernetes manifests:
```bash
mkdir -p kubernetes
unzip -j k8s-manifests.zip -d ./kubernetes
```

### 4.1 Azure Container Registry (ACR) Setup

1. **Create a new Resource Group** (if not already exists):
   ```bash
   az group create --name $RESOURCE_GROUP --location $LOCATION
   ```

2. **Create Azure Container Registry**:
   ```bash
   az acr create \
     --resource-group $RESOURCE_GROUP \
     --name $ACR_NAME \
     --sku Premium \
     --admin-enabled true
   ```

3. **Get ACR credentials**:
   ```bash
   ACR_USERNAME=$(az acr credential show --name $ACR_NAME --query "username" -o tsv)
   ACR_PASSWORD=$(az acr credential show --name $ACR_NAME --query "passwords[0].value" -o tsv)
   
   echo "ACR Username: $ACR_USERNAME"
   echo "ACR Password: $ACR_PASSWORD"
   ```

4. **Log in to ACR**:
   ```bash
   az acr login --name $ACR_NAME
   ```

### 4.2 Docker Image Import to ACR

1. **Import Docker image tar files to ACR**:
   ```bash
   # Load the Docker images from tar files
   docker load -i agent-ui.tar
   docker load -i agent-apis.tar
   docker load -i agent-worker.tar
   
   # Tag the images for ACR (use exact same names as in tar files)
   docker tag vendor/agent-ui:latest ${ACR_NAME}.azurecr.io/agent-ui:latest
   docker tag vendor/agent-apis:latest ${ACR_NAME}.azurecr.io/agent-apis:latest
   docker tag vendor/agent-worker:latest ${ACR_NAME}.azurecr.io/agent-worker:latest
   
   # Push the images to ACR
   docker push ${ACR_NAME}.azurecr.io/agent-ui:latest
   docker push ${ACR_NAME}.azurecr.io/agent-apis:latest
   docker push ${ACR_NAME}.azurecr.io/agent-worker:latest
   ```

2. **Verify image import**:
   ```bash
   az acr repository list --name $ACR_NAME --output table
   
   # Verify tags for each repository
   az acr repository show-tags --name $ACR_NAME --repository agent-ui --output table
   az acr repository show-tags --name $ACR_NAME --repository agent-apis --output table
   az acr repository show-tags --name $ACR_NAME --repository agent-worker --output table
   ```

### 4.3 Azure OpenAI Service Setup

1. **Create Azure OpenAI service**:
   ```bash
   # Create Azure OpenAI service
   az cognitiveservices account create \
     --name $OPENAI_NAME \
     --resource-group $RESOURCE_GROUP \
     --location $LOCATION \
     --kind OpenAI \
     --sku s0 \
     --custom-domain $OPENAI_NAME

   # Wait for the service to be created
   echo "Waiting for Azure OpenAI service to be ready..."
   sleep 60
   ```

2. **Deploy models to Azure OpenAI service**:
   ```bash
   # Deploy GPT-4 Turbo model
   az cognitiveservices account deployment create \
     --name $OPENAI_NAME \
     --resource-group $RESOURCE_GROUP \
     --deployment-name gpt-4 \
     --model-name gpt-4 \
     --model-version "1106" \
     --model-format OpenAI \
     --sku Standard \
     --capacity 1

   # Deploy embedding model
   az cognitiveservices account deployment create \
     --name $OPENAI_NAME \
     --resource-group $RESOURCE_GROUP \
     --deployment-name text-embedding-3-large \
     --model-name text-embedding-3-large \
     --model-version "1" \
     --model-format OpenAI \
     --sku Standard \
     --capacity 1
   ```

3. **Get Azure OpenAI API key**:
   ```bash
   OPENAI_API_KEY=$(az cognitiveservices account keys list \
     --name $OPENAI_NAME \
     --resource-group $RESOURCE_GROUP \
     --query key1 -o tsv)

   echo "Azure OpenAI API Key: $OPENAI_API_KEY"
   ```

### 4.4 Azure Database for PostgreSQL Setup

1. **Create Virtual Network and Subnets**:
   ```bash
   # Create VNET and subnets if they don't exist
   az network vnet create \
     --resource-group $RESOURCE_GROUP \
     --name $VNET_NAME \
     --address-prefix 10.0.0.0/16 \
     --subnet-name $SUBNET_AKS \
     --subnet-prefix 10.0.0.0/24
   
   # Create subnet for PostgreSQL
   az network vnet subnet create \
     --resource-group $RESOURCE_GROUP \
     --vnet-name $VNET_NAME \
     --name postgres-subnet \
     --address-prefix 10.0.1.0/24 \
     --delegations Microsoft.DBforPostgreSQL/flexibleServers
   
   # Get the subnet IDs for later use
   POSTGRES_SUBNET_ID=$(az network vnet subnet show \
     --resource-group $RESOURCE_GROUP \
     --vnet-name $VNET_NAME \
     --name postgres-subnet \
     --query id -o tsv)
   ```

2. **Create PostgreSQL flexible server with private access**:
   ```bash
   # Create PostgreSQL server in the same region as AKS
   # This improves performance and reduces latency
   export POSTGRES_REGION=$LOCATION
   
   # Create a private DNS zone for PostgreSQL
   az network private-dns zone create \
     --resource-group $RESOURCE_GROUP \
     --name "${DB_SERVER}.private.postgres.database.azure.com"
   
   # Link the private DNS zone to the VNET
   az network private-dns link vnet create \
     --resource-group $RESOURCE_GROUP \
     --zone-name "${DB_SERVER}.private.postgres.database.azure.com" \
     --name MyDNSLinkToVNet \
     --virtual-network $VNET_NAME \
     --registration-enabled false
   
   # Create PostgreSQL server with private access
   az postgres flexible-server create \
     --resource-group $RESOURCE_GROUP \
     --name $DB_SERVER \
     --location $POSTGRES_REGION \
     --admin-user $DB_USERNAME \
     --admin-password $DB_PASSWORD \
     --sku-name Standard_D4s_v3 \
     --storage-size 16384 \
     --version 14 \
     --vnet $VNET_NAME \
     --subnet postgres-subnet \
     --private-dns-zone "${DB_SERVER}.private.postgres.database.azure.com"
   
   # Create databases
   az postgres flexible-server db create \
     --resource-group $RESOURCE_GROUP \
     --server-name $DB_SERVER \
     --database-name $DB_NAME
   
   az postgres flexible-server db create \
     --resource-group $RESOURCE_GROUP \
     --server-name $DB_SERVER \
     --database-name $VECTOR_DB_NAME
   ```

### 4.5 AKS Cluster Creation

1. **Create AKS cluster**:
   ```bash
   az aks create \
     --resource-group $RESOURCE_GROUP \
     --name $AKS_CLUSTER \
     --node-count $AKS_NODE_COUNT \
     --node-vm-size $AKS_NODE_SIZE \
     --enable-managed-identity \
     --attach-acr $ACR_NAME
   ```

2. **Get credentials for the AKS cluster**:
   ```bash
   az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER
   ```

3. **Verify AKS connection**:
   ```bash
   kubectl get nodes
   ```

### 4.6 Configuration Setup

1. **Create AKS namespace**:
   ```bash
   kubectl create namespace ai-agent
   ```

2. **Create secrets for sensitive configuration**:
   ```bash
   # Create secret for database credentials
   # For private VNET-integrated Flexible Server
   POSTGRES_HOST="${DB_SERVER}.private.postgres.database.azure.com"
   
   kubectl create secret generic db-credentials \
     --namespace ai-agent \
     --from-literal=username=$DB_USERNAME \
     --from-literal=password=$DB_PASSWORD \
     --from-literal=host=$POSTGRES_HOST \
     --from-literal=database=$DB_NAME
   
   kubectl create secret generic vector-db-credentials \
     --namespace ai-agent \
     --from-literal=username=$DB_USERNAME \
     --from-literal=password=$DB_PASSWORD \
     --from-literal=host=$POSTGRES_HOST \
     --from-literal=database=$VECTOR_DB_NAME
   
   # Create secret for Azure OpenAI credentials
   kubectl create secret generic openai-credentials \
     --namespace ai-agent \
     --from-literal=api-key=$OPENAI_API_KEY \
     --from-literal=endpoint=$OPENAI_ENDPOINT \
     --from-literal=model-name=gpt-4 \
     --from-literal=model-version=1106 \
     --from-literal=embedding-model-name=text-embedding-3-large \
     --from-literal=embedding-model-version=1
   ```

### 4.7 AKS Deployment

1. **Apply Kubernetes manifests**:
   ```bash
   # Apply all Kubernetes manifests at once
   kubectl apply -f kubernetes/ -n ai-agent
   ```

2. **Verify deployment status**:
   ```bash
   # Check if pods are running
   kubectl get pods -n ai-agent
   
   # Check if services are created
   kubectl get services -n ai-agent
   ```

### 4.8 External Communication Configuration

1. **Create Azure API Management instance** (if not already exists):
   ```bash
   az apim create \
     --name $APIM_NAME \
     --resource-group $RESOURCE_GROUP \
     --publisher-name "Your Organization" \
     --publisher-email "it-admin@yourcompany.com" \
     --sku-name Developer
   ```

2. **Configure Azure API Management for external communication**:
   ```bash
   # Get the service IP address
   SERVICE_IP=$(kubectl get service agent-service -n ai-agent -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
   
   # Create an API
   az apim api create \
     --resource-group $RESOURCE_GROUP \
     --service-name $APIM_NAME \
     --api-id agent-api \
     --display-name "AI Agent API" \
     --path agent \
     --protocols https \
     --service-url "http://${SERVICE_IP}"
   ```

### 4.9 Databricks Integration (Optional)

1. **Configure Databricks connection**:
   ```bash
   # Create an Azure AD service principal for Databricks access
   SP_PASSWORD=$(az ad sp create-for-rbac \
     --name "agent-databricks-access" \
     --role "Contributor" \
     --scopes /subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RESOURCE_GROUP \
     --query password -o tsv)
   
   SP_CLIENT_ID=$(az ad sp list --display-name "agent-databricks-access" --query [0].appId -o tsv)
   SP_TENANT_ID=$(az account show --query tenantId -o tsv)
   
   # Create secret in AKS for Databricks credentials
   kubectl create secret generic databricks-credentials \
     --namespace ai-agent \
     --from-literal=client-id=$SP_CLIENT_ID \
     --from-literal=client-secret=$SP_PASSWORD \
     --from-literal=tenant-id=$SP_TENANT_ID
   ```

2. **Update deployment with Databricks configuration**:
   ```bash
   # Update the deployment to use Databricks configuration
   kubectl set env deployment/agent-backend \
     --namespace ai-agent \
     DATABRICKS_WORKSPACE_URL=https://${DATABRICKS_WORKSPACE}.azuredatabricks.net \
     DATABRICKS_CLIENT_ID=$SP_CLIENT_ID
   ```

## 5. Deployment Verification

### 5.1 System Component Verification

| Component | Verification Command | Expected Result |
|-----------|----------------------|----------------|
| Pods | `kubectl get pods -n ai-agent` | All pods showing `Running` status |
| Services | `kubectl get services -n ai-agent` | All services have assigned ClusterIPs |
| Persistent Volumes | `kubectl get pv,pvc -n ai-agent` | PVs and PVCs show `Bound` status |

### 5.2 Functional Verification

1. **Database Connection Test**:
   ```bash
   # Forward port to database pod
   kubectl port-forward -n ai-agent svc/agent-database 5432:5432
   
   # In another terminal
   PGPASSWORD=$DB_PASSWORD psql -h localhost -U $DB_USERNAME -d $DB_NAME -c "SELECT version();"
   ```
   Expected result: PostgreSQL version information

2. **API Connection Test**:
   ```bash
   # Forward port to backend service
   kubectl port-forward -n ai-agent svc/agent-backend 8000:8000
   
   # In another terminal
   curl http://localhost:8000/api/health
   ```
   Expected result: `{"status":"healthy"}`

3. **Azure OpenAI Connection Test**:
   ```bash
   # Forward port to backend service
   kubectl port-forward -n ai-agent svc/agent-backend 8000:8000
   
   # In another terminal
   curl http://localhost:8000/api/llm/health
   ```
   Expected result: `{"status":"connected","provider":"azure_openai"}`

## 6. Troubleshooting

### 6.1 Common Issues and Resolutions

| Issue | Cause | Resolution |
|-------|-------|------------|
| Pod crash looping | Missing configuration or secrets | Check pod logs: `kubectl logs -n ai-agent <pod-name>` |
| Database connection failure | Incorrect credentials or network issues | Verify secrets and network connectivity |
| Image pull failures | ACR access issues | Check image names and AKS-ACR connectivity |
| API Management errors | Incorrect backend configuration | Verify backend service URL and authentication settings |
| OpenAI connection errors | API key issues or endpoint misconfiguration | Verify OpenAI credentials and endpoint configuration |

### 6.2 Collecting Diagnostic Information

In case of deployment issues, collect the following diagnostic information:

1. **Pod logs**:
   ```bash
   kubectl logs -n ai-agent <pod-name> > pod-logs.txt
   ```

2. **Deployment status**:
   ```bash
   kubectl describe deployment -n ai-agent <deployment-name> > deployment-status.txt
   ```

3. **Service configuration**:
   ```bash
   kubectl get svc -n ai-agent -o yaml > services.yaml
   ```

4. **Environment variables**:
   ```bash
   kubectl exec -n ai-agent <pod-name> -- env > pod-env.txt
   ```

## 7. Resource Cleanup

When the testing is complete, use the following script to clean up all resources deployed in Azure:

```bash
#!/bin/bash

# Set environment variables
RESOURCE_GROUP=ai-agent-rg
AKS_CLUSTER=ai-agent-aks
APIM_NAME=ai-agent-apim
OPENAI_NAME=ai-agent-openai
DB_SERVER=ai-agent-postgres
ACR_NAME=aiagentacr
DATABRICKS_WORKSPACE=ai-agent-databricks

# Delete resources in reverse order
echo "Cleaning up resources..."

# Delete AKS cluster first (this will delete all Kubernetes resources)
echo "Deleting AKS cluster..."
az aks delete --name $AKS_CLUSTER --resource-group $RESOURCE_GROUP --yes

# Delete API Management
echo "Deleting API Management..."
az apim delete --name $APIM_NAME --resource-group $RESOURCE_GROUP --yes

# Delete PostgreSQL server
echo "Deleting PostgreSQL server..."
az postgres flexible-server delete --name $DB_SERVER --resource-group $RESOURCE_GROUP --yes

# Delete Azure OpenAI
echo "Deleting Azure OpenAI service..."
az cognitiveservices account delete --name $OPENAI_NAME --resource-group $RESOURCE_GROUP

# Delete ACR
echo "Deleting Container Registry..."
az acr delete --name $ACR_NAME --resource-group $RESOURCE_GROUP --yes

# Delete Databricks workspace (if created)
echo "Deleting Databricks workspace..."
az databricks workspace delete --name $DATABRICKS_WORKSPACE --resource-group $RESOURCE_GROUP --yes

# Finally, delete the resource group
echo "Deleting resource group..."
az group delete --name $RESOURCE_GROUP --yes

echo "All resources have been cleaned up."
```

Save this script as `cleanup.sh`, make it executable with `chmod +x cleanup.sh`, and run it when you're ready to clean up all resources:

```bash
./cleanup.sh
```

## Appendix A: Environment-Specific Configuration Parameters

The table below shows the recommended environment variables with suggested values for your environment:

| Parameter | Description | Suggested Value |
|-----------|-------------|----------------|
| `RESOURCE_GROUP` | Azure resource group | `ai-agent-rg` |
| `LOCATION` | Azure region | `eastus` |
| `ACR_NAME` | Name of ACR registry | `aiagentacr` |
| `AKS_CLUSTER` | Name of AKS cluster | `ai-agent-aks` |
| `AKS_NODE_SIZE` | AKS node VM size | `Standard_D8s_v3` |
| `AKS_NODE_COUNT` | Number of AKS nodes | `3` |
| `APIM_NAME` | Azure API Management name | `ai-agent-apim` |
| `OPENAI_NAME` | Azure OpenAI service name | `ai-agent-openai` |
| `VNET_NAME` | Virtual network name | `ai-agent-vnet` |
| `SUBNET_AKS` | AKS subnet name | `aks-subnet` |
| `SUBNET_APIM` | API Management subnet name | `apim-subnet` |
| `DATABRICKS_WORKSPACE` | Databricks workspace name | `ai-agent-databricks` |

## Appendix B: Required Network Configurations

| Source | Destination | Port/Protocol | Purpose |
|--------|-------------|---------------|---------|
| AKS cluster | ACR | 443/TCP | Image pulling |
| AKS cluster | Azure OpenAI | 443/TCP | LLM API access |
| AKS cluster | PostgreSQL | 5432/TCP | Database access (via VNET) |
| AKS cluster | Databricks | 443/TCP | Data access (if needed) |
| Internal network | API Management | 443/TCP | API access |
| API Management | AKS service | 80,443/TCP | Backend service access |

## Appendix C: Working in Air-Gapped Environments

For completely air-gapped environments with no internet access, additional preparations are necessary:

### C.1 Pre-Deployment Package Preparation

Before entering the air-gapped environment, prepare the following on an internet-connected system:

1. **Complete Azure CLI package**:
   - Download the Azure CLI installer appropriate for your deployment workstation
   - Download any required extensions

2. **Kubernetes tools**:
   - Download the kubectl binary for your deployment workstation
   - Package any required Kubernetes tools (helm, etc.)

3. **Docker engine**:
   - Download Docker Desktop or Docker Engine installer

4. **Documentation**:
   - Download complete Azure documentation for relevant services
   - Save copies of Kubernetes documentation

### C.2 Modified Deployment Process

In air-gapped environments:

1. Install all pre-downloaded tools on the deployment workstation
2. Use private/internal Azure endpoints for all Azure service communication
3. Ensure all required container images are imported to ACR using the provided TAR files
4. Modify Kubernetes manifests to use only the internal ACR for image references
5. Implement additional validation steps at each deployment stage
6. Document any deviations from standard procedure for audit purposes

### C.3 Alternative LLM Provider Options

If Azure OpenAI is not accessible in your environment, the deployment can be adapted to use:

1. **Self-hosted LLM options**:
   - Configure the application to use locally deployed open-source LLMs
   - Adjust hardware requirements accordingly (typically requiring GPU resources)

2. **Alternative cloud provider LLMs**:
   - If another cloud provider's LLM services are approved in your environment
   - Requires additional configuration and possibly code changes

3. **On-premises AI infrastructure**:
   - For organizations with existing AI infrastructure investments
   - May require custom integration work
