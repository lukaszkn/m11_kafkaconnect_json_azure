# KafkaConnect [json]

## [Screenshots from execution are emdedded in this README (stored in `screenshots` folder)](screenshots/)

## 1. Create a Storage Account in Azure for Terraform State

Terraform requires a remote backend to store its state file. Follow these steps:

### **Option 1: Using Azure CLI [Recomended]**

1. **Authenticate with Azure CLI**

Run the following command to authenticate:

```bash
az login
```

2. **Create a Resource Group:**  

To create a Resource Group name run the command:

```bash
az group create --name <RESOURCE_GROUP_NAME> --location <AZURE_REGION>
```  

3. **Create a Storage Account:**  

```bash
az storage account create --name <STORAGE_ACCOUNT_NAME> --resource-group <RESOURCE_GROUP_NAME> --location <AZURE_REGION> --sku Standard_LRS
```  

4. **Create a Storage Container:**  

```bash
az storage container create --name <CONTAINER_NAME> --account-name <STORAGE_ACCOUNT_NAME>
```  

## 2. Get Your Azure Subscription ID

### **Option 1: Using Azure Portal (Web UI)**

1. **Go to [Azure Portal](https://portal.azure.com/)**  
2. Click on **Subscriptions** in the left-hand menu.  
3. You will see a list of your subscriptions.  
4. Choose the subscription you want to use and copy the **Subscription ID**.  

### **Option 2: Using Azure CLI**

- Retrieve it using Azure CLI:  

```bash
az account show --query id --output tsv
```

## 3. Update Terraform Configuration

Navigate into folder `terraform`. Modify `main.tf` and replace placeholders with your actual values.

- Get a Storage Account Key (`<STORAGE_ACCOUNT_KEY>`):

```bash
az storage account keys list --resource-group <RESOURCE_GROUP_NAME> --account-name <STORAGE_ACCOUNT_NAME> --query "[0].value"
```

- **Edit the backend block in `main.tf` :**

```hcl
  terraform {
    backend "azurerm" {
      resource_group_name  = "<RESOURCE_GROUP_NAME>"
      storage_account_name = "<STORAGE_ACCOUNT_NAME>"
      container_name       = "<CONTAINER_NAME>"
      key                  = "<STORAGE_ACCOUNT_KEY>"
    }
  }
  provider "azurerm" {
    features {}
    subscription_id = "<SUBSCRIPTION_ID>"
  }
```  

## 4. Deploy Infrastructure with Terraform

To start the deployment using Terraform scripts, you need to navigate to the `terraform` folder.

```bash
cd terraform/
```

Run the following Terraform commands:

```bash
terraform init
```  

```bash
terraform plan -out terraform.plan
```  

```bash
terraform apply terraform.plan
```  

- To see the resource group that was created by terraform (`<RESOURCE_GROUP_NAME_CREATED_BY_TERRAFORM>`) run the command:

```bash
terraform output resource_group_name
```

<img src='screenshots/Screenshot 2025-09-25 at 12.06.54.png' width='850'>

## 5. Verify Resource Deployment in Azure

After Terraform completes, verify that resources were created:

1. **Go to the [Azure Portal](https://portal.azure.com/)**  
2. Navigate to **Resource Groups** → **Find `<RESOURCE_GROUP_NAME_CREATED_BY_TERRAFORM>`**  
3. Check that the resources (Storage Account, Databricks, etc.) are created.  

Alternatively, check via CLI:

```bash
az resource list --resource-group <RESOURCE_GROUP_NAME_CREATED_BY_TERRAFORM> --output table
```

## 6. Retrieve kubeconfig.yaml and Set It as Default

1. Extract `kubeconfig.yaml` from the directory `/terraform`:

First we need to recieve the current `<AKS_NAME>`,run a command:

```bash
terraform output -raw aks_name
```

Then we need to get `<RESOURCE_GROUP_NAME_CREATED_BY_TERRAFORM>`, run a commands:

```bash
terraform output resource_group_name
```

2. Set `kubeconfig.yaml` as Default for kubectl in Current Terminal Session:

Before run change the placeholders:

```bash
az aks get-credentials --resource-group <RESOURCE_GROUP_NAME_CREATED_BY_TERRAFORM> --name <AKS_NAME>
```

3. Switch to the project kubernetes namespace:

```bash
kubectl config set-context --current --namespace confluent
```

Verify Kubernetes Cluster Connectivity:

```bash
kubectl get nodes
```

<img src='screenshots/Screenshot 2025-09-25 at 12.12.59.png' width='850'>

4. Install Confluent for Kubernetes

- Add the Confluent for Kubernetes Helm repository:

```bash
helm repo add confluentinc https://packages.confluent.io/helm
helm repo update
 ```

- Install Confluent for Kubernetes:

```bash
helm upgrade --install confluent-operator confluentinc/confluent-for-kubernetes
```

<img src='screenshots/Screenshot 2025-09-25 at 12.14.17.png' width='850'>

## 7. Configure and Use Azure Container Registry (ACR)

Azure Container Registry (ACR) is used to store container images before deploying them to AKS.

1. Get the `<ACR_NAME>` run the following command:

```bash
terraform output acr_login_server
```

2. Authenticate with ACR.

Change the `<ACR_NAME>` with the output from the step `1`

```bash
az acr login --name <ACR_NAME>
```

## 8. Build and push `azure-connector` into ACR

1. Go into folder `connectors/`, modify the Dockerfile if required.
2. Build the Docker image: 

```bash
docker build --platform linux/amd64 -t <ACR_NAME>/azure-connector:latest .
```

3. Push Docker Image to ACR:

```bash
docker push <ACR_NAME>/azure-connector
```

<img src='screenshots/Screenshot 2025-09-25 at 12.23.48.png' width='850'>

4. Verify Image in ACR:

```bash
az acr repository list --name <ACR_NAME> --output table
```

## 9.  Install Confluent Platform

- Go into `root` folder. Modify the file `confluent-platform.yaml` and replace the placeholder with actual value:

```yaml
    image:
    ⦙ application: <ACR_NAME>/azure-connector:latest
    ⦙ init: confluentinc/confluent-init-container:2.10.0
    dependencies:
```

- Install all Confluent Platform components:

```bash
kubectl apply -f confluent-platform.yaml
```

- Install a sample producer app and topic:

```bash
kubectl apply -f producer-app-data.yaml
```

- Check that everything is deployed (all pods should be in the `Running` state and have a ready status of `1/1`):
It will take approximately **15–20 minutes** to set up all resources.

```bash
kubectl get pods -o wide 
```

<img src='screenshots/Screenshot 2025-09-25 at 12.32.11.png' width='850'>

<img src='screenshots/Screenshot 2025-09-25 at 14.31.06.png' width='850'>

### View Control Center

- Set up port forwarding to Control Center web UI from local machine:

```bash
kubectl port-forward controlcenter-0 9021:9021 &>/dev/null &
```

- Browse to Control Center: [http://localhost:9021](http://localhost:9021)

<img src='screenshots/Screenshot 2025-09-25 at 13.19.38.png' width='850'>

## 10. Create a kafka topic

The topic should have at least 3 partitions because the azure blob storage has 3 partitions. Name the new topic: `expedia`.

- Create a connection for kafka:

```bash
kubectl port-forward connect-0 8083:8083 &>/dev/null &
```

- execute below command to create Kafka topic with a name `expedia`

```bash
kubectl exec kafka-0 -c kafka -- bash -c "/usr/bin/kafka-topics --create --topic expedia --replication-factor 3 --partitions 3 --bootstrap-server kafka:9092"
```

<img src='screenshots/Screenshot 2025-09-25 at 13.18.31.png' width='850'>

## 11. Upload the data files into Azure Conatainers

1. Log in to [Azure Portal](https://portal.azure.com/)
2. Go to your STORAGE account => Data Storage => Containers

- to get actual STORAGE account name, you can run a command from the terraform folder:

```bash
terraform output storage_account_name
```

3. Choose the conatiner `data`  
4. You should see the upload button
5. Upload the `data` files here.
6. The folder structure should be like:

```bash
    <YOUR_DIR_NAME_WHERE_IS_TOPIC_LOCATED>
    └── topics
        └── <TOPIC_NAME>
            ├── partition=0
            │   └── example+0+0000000000.avro
            ├── partition=1
            │   └── example+1+0000000000.avro
            └── partition=2
                └── example+2+0000000000.avro
```

<img src='screenshots/Screenshot 2025-09-25 at 14.20.53.png' width='850'>

## 12. Prepare the `azure connector` configuration file

Modify the file `/terraform/azure-source-cc.json` (example file located in folder `connectors`)

- Modify Kafka Connect to read data from storage container into Kafka topic (`expedia`)
- Use this tutorial [Azure Blob Storage Source Connector for Confluent Platform](https://docs.confluent.io/kafka-connect-azure-blob-storage-source/current/index.html) & [Google Cloud Storage Source Connector for Confluent Platform](https://docs.confluent.io/kafka-connectors/gcs-source/current)
- Before uploading data into Kafka topic, please, mask time from the date field using MaskField transformer like: 2015-08-18 12:37:10 -> 0000-00-00 00:00:00. Use this tutorial [Kafka Connect MaskField](https://docs.confluent.io/kafka-connectors/transforms/current/maskfield.html)
- placeholders in the file must be updated and also add parameters for MaskField

```yaml
    "azblob.account.key": "<redacted>",
    "azblob.account.name": "stdevwesteurope2ch6",
    "azblob.container.name": "data",
    "bootstrap.servers": "kafka:9071",
    "connector.class": "io.confluent.connect.azure.blob.storage.AzureBlobStorageSourceConnector",
    "format.class": "io.confluent.connect.azure.blob.storage.format.avro.AvroFormat",
    "tasks.max": "2",

    "topics": "expedia",
    "topics.dir": "m11kafkaconnect/topics",

    "transforms": "MaskTime",
    "transforms.MaskTime.type": "org.apache.kafka.connect.transforms.MaskField$Value",
    "transforms.MaskTime.fields": "date_time",
    "transforms.MaskTime.replacement": "0000-00-00 00:00:00"
```

## 13. Upload the connector file through the API

- go into folder `terraform`, and run a command:

```bash
curl -s -X POST -H "Content-Type:application/json" --data @azure-source-cc.json http://localhost:8083/connectors
```

<img src='screenshots/Screenshot 2025-09-25 at 14.31.55.png' width='850'>

## 14. Verify the messages in Kafka

- Browse to Control Center: [http://localhost:9021](http://localhost:9021)
- Go into Cluster => Topics
- Choose your topic name
- In the `messages` tab you should able to see incoming messages

<img src='screenshots/Screenshot 2025-09-25 at 14.33.50.png' width='850'>

## 15. Destroy Infrastructure (Required Step)

After completing all steps, **destroy the infrastructure** to clean up all deployed resources.

To remove all deployed resources, run:

- Clean Kubernetes Resources run from the git `root` folder:

```bash
kubectl delete -f producer-app-data.yaml
```

```bash
kubectl delete -f confluent-platform.yaml
```

```bash
helm uninstall confluent-operator
```

<img src='screenshots/Screenshot 2025-09-25 at 14.35.31.png' width='850'>

- To remove all Azure deployed resources, run from the `terraform` folder:

```bash
terraform destroy
```

<img src='screenshots/Screenshot 2025-09-25 at 14.43.35.png' width='850'>
