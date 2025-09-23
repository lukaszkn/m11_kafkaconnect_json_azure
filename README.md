# KafkaConnect [json]

## Prerequisites

Before proceeding, ensure you have the following tools installed:

- Rancher Desktop – Required for running Kubernetes locally (alternative to Docker Desktop). Please keep it running.
- Azure CLI (az) – Used to interact with Azure services and manage resources.
- Terraform – Infrastructure as Code (IaC) tool for provisioning Azure resources.
- Helm - Helps you manage Kubernetes applications. Helm Charts help you define, install, and upgrade them.

📘 Follow the full setup instructions for [Windows environment setup](./setup-windows.md)<br>
🍎 Follow the full setup instructions for [MacOS environment setup](./setup-macos.md)<br>
🐧 Follow the full setup instructions for [Ubuntu 24.10 environment setup](./setup-ubuntu.md)

📌 **Important Guidelines**
Please read the instructions carefully before proceeding. Follow these guidelines to avoid mistakes:

- If you see `<SOME_TEXT_HERE>`, you need to **replace this text and the brackets** with the appropriate value as described in the instructions.
- Follow the steps in order to ensure proper setup.
- Pay attention to **bolded notes**, warnings, or important highlights throughout the document.
- Clean Up Azure Resources Before Proceeding. Since you are using a **free-tier** Azure account, it’s crucial to clean up any leftover resources from previous lessons or deployments before proceeding. Free-tier accounts 
have strict resource quotas, and exceeding these limits may cause deployment failures.

## 1. Create a Storage Account in Azure for Terraform State

Terraform requires a remote backend to store its state file. Follow these steps:

### **Option 1: Using Azure CLI [Recomended]**

1. **Authenticate with Azure CLI**

Run the following command to authenticate:

```bash
az login
```

💡 **Notes**:
- This will open a browser for authentication.
- If you have **multiple subscriptions**, you will be prompted to **choose one**.
- If you only have **one subscription**, it will be selected by default.
- **Please read the output carefully** to ensure you are using the correct subscription.

2. **Create a Resource Group:**  

📌 **Important! Naming Rules for Azure Resources**

<details>
  <summary>👇<strong> Before proceeding, carefully review the naming rules to avoid deployment failures.</strong> 👇 [⬇️⬇️⬇️ Expand to see Naming Rules ⬇️⬇️⬇️]</summary>

### 📝 **Naming Rules for Azure Resources**
Before creating any resources, ensure that you follow **Azure's naming conventions** to avoid errors.

- **Resource names must follow specific character limits and allowed symbols** depending on the resource type.
- **Using unsupported special characters can cause deployment failures.**
- **Storage accounts, resource groups, and other Azure resources have different rules.**

🔹 **Common Rules Across Most Resources**:
- **Allowed characters:** Only **letters (A-Z, a-z)**, **numbers (0-9)**.
- **Case Sensitivity:** Most names are **lowercase only** (e.g., storage accounts).
- **Length Restrictions:** Vary by resource type (e.g., Storage accounts: **3-24 characters**).
- **No special symbols:** Avoid characters like `@`, `#`, `$`, `%`, `&`, `*`, `!`, etc.
- **Hyphens and underscores:** Some resources support them, but rules differ.

📖 **For complete naming rules, refer to the official documentation:**  
🔗 [Azure Naming Rules and Restrictions](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/resource-name-rules)

</details>

To create a Resource Group name run the command:

```bash
az group create --name <RESOURCE_GROUP_NAME> --location <AZURE_REGION>
```  

3. **Create a Storage Account:**  

⚠️  **Storage Account name, are globally unique, so you must choose a name that no other Azure user has already taken.** 

```bash
az storage account create --name <STORAGE_ACCOUNT_NAME> --resource-group <RESOURCE_GROUP_NAME> --location <AZURE_REGION> --sku Standard_LRS
```  

4. **Create a Storage Container:**  

```bash
az storage container create --name <CONTAINER_NAME> --account-name <STORAGE_ACCOUNT_NAME>
```  

### **Option 2: Using Azure Portal (Web UI)**

1. **Log in to [Azure Portal](https://portal.azure.com/)**  
2. Navigate to **Resource Groups** and click **Create**.  
3. Enter a **Resource Group Name**, select a **Region**, and click **Review + Create**.  
4. Once the Resource Group is created, go to **Storage Accounts** and click **Create**.  
5. Fill in the required details:  
   - **Storage Account Name**  
   - **Resource Group** (Select the one you just created)  
   - **Region** (Choose your preferred region)  
   - **Performance**: Standard  
   - **Redundancy**: Locally Redundant Storage (LRS)  
6. Click **Review + Create** and then **Create**.  
7. Once created, go to the **Storage Account** → **Data Storage** → **Containers** → Click **+ Container**.  
8. Name it `tfstate` (as example) and set **Access Level** to **Private**.  
9. To get `<STORAGE_ACCOUNT_KEY>`: Navigate to your **Storage Account** →**Security & Networking** → **Access Keys**. Press `show` button on `key1`

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
2. ⚠️ To build the Docker image, choose the correct command based on your CPU architecture: 

    <details>
    <summary><code>Linux</code>, <code>Windows</code>, <code>&lt;Intel-based macOS&gt;</code> (<i>click to expand</i>)</summary>

    ```bash
    docker build -t <ACR_NAME>/azure-connector:latest .
    ```

    </details>
    <details>
    <summary><code>macOS</code> with <code>M1/M2/M3</code> <code>&lt;ARM-based&gt;</code>  (<i>click to expand</i>)</summary>

    ```bash
    docker build --platform linux/amd64 -t <ACR_NAME>/azure-connector:latest .
    ```

    </details>

3. Push Docker Image to ACR:

```bash
docker push <ACR_NAME>/azure-connector
```

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

### View Control Center

- Set up port forwarding to Control Center web UI from local machine:

    <details>
    <summary><code>Linux</code>, <code>MacOS</code> (<i>click to expand</i>)</summary>

    ```bash
    kubectl port-forward controlcenter-0 9021:9021 &>/dev/null &
    ```

    </details>
    <details>
    <summary><code>Windows - [powershell]</code></code>  (<i>click to expand</i>)</summary>

    ```bash
    Start-Process powershell -WindowStyle Hidden -ArgumentList 'kubectl port-forward controlcenter-0 9021:9021 *> $null'
    ```

    </details>

- Browse to Control Center: [http://localhost:9021](http://localhost:9021)

## 10. Create a kafka topic

The topic should have at least 3 partitions because the azure blob storage has 3 partitions. Name the new topic: `expedia`.

- Create a connection for kafka:

    <details>
    <summary><code>Linux</code>, <code>MacOS</code> (<i>click to expand</i>)</summary>

    ```bash
    kubectl port-forward connect-0 8083:8083 &>/dev/null &
    ```

    </details>
    <details>
    <summary><code>Windows - [powershell]</code></code>  (<i>click to expand</i>)</summary>

    ```bash
    Start-Process powershell -WindowStyle Hidden -ArgumentList 'kubectl port-forward connect-0 8083:8083 *> $null'
    ```

    </details>

- execute below command to create Kafka topic with a name `expedia`

```bash
kubectl exec kafka-0 -c kafka -- bash -c "/usr/bin/kafka-topics --create --topic expedia --replication-factor 3 --partitions 3 --bootstrap-server kafka:9092"
```

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

## 12. Prepare the `azure connector` configuration file

Modify the file `/terraform/azure-source-cc.json` (example file located in folder `connectors`)

- Modify Kafka Connect to read data from storage container into Kafka topic (`expedia`)
- Use this tutorial [Azure Blob Storage Source Connector for Confluent Platform](https://docs.confluent.io/kafka-connect-azure-blob-storage-source/current/index.html) & [Google Cloud Storage Source Connector for Confluent Platform](https://docs.confluent.io/kafka-connectors/gcs-source/current)
- Before uploading data into Kafka topic, please, mask time from the date field using MaskField transformer like: 2015-08-18 12:37:10 -> 0000-00-00 00:00:00. Use this tutorial [Kafka Connect MaskField](https://docs.confluent.io/kafka-connectors/transforms/current/maskfield.html)
- placeholders in the file must be updated and also add parameters for MaskField

```yaml
    "bootstrap.servers"                 = "kafka:9071"
    "topics"                            = "PUT_YOUR_TOPIC_NAME_HERE"
    "topics.dir"                        = "PUT_YOUR_DIR_NAME_WHERE_IS_TOPIC_LOCATED"
    // please add your MaskField configs here
```

## 13. Upload the connector file through the API

- go into folder `terraform`, and run a command depends on your OS:

    <details>
    <summary><code>Linux</code>, <code>MacOS</code> (<i>click to expand</i>)</summary>

    ```bash
    curl -s -X POST -H "Content-Type:application/json" --data @azure-source-cc.json http://localhost:8083/connectors
    ```

    </details>
    <details>
    <summary><code>Windows - [powershell]</code>  (<i>click to expand</i>)</summary>

    ```bash
    Remove-item alias:curl
    ```

    then:

    ```bash
    curl -s -X POST -H "Content-Type:application/json" --data @azure-source-cc.json http://localhost:8083/connectors
    ```

    </details>

## 14. Verify the messages in Kafka

- Browse to Control Center: [http://localhost:9021](http://localhost:9021)
- Go into Cluster => Topics
- Choose your topic name
- In the `messages` tab you should able to see incoming messages

## 15. Destroy Infrastructure (Required Step)

After completing all steps, **destroy the infrastructure** to clean up all deployed resources.

⚠️ **Warning:** This action is **irreversible**. Running the command below will **delete all infrastructure components** created in previous steps.

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

- To remove all Azure deployed resources, run from the `terraform` folder:

```bash
terraform destroy
```
