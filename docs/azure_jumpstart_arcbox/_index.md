# Jumpstart ArcBox - An Azure Arc Jumpstart project

## Jumpstart ArcBox - Overview

Jumpstart ArcBox is a project that provides an easy to deploy sandbox for all things Azure Arc. ArcBox is designed to be completely self-contained within a single Azure subscription and resource group, which will make it easy for a user to get hands-on with all available Azure Arc technology with nothing more than an available Azure subscription.

![ArcBox architecture diagram](./arch_capi.png)

### Use cases

* Sandbox environment for getting hands-on with Azure Arc technologies
* Accelerator for Proof-of-concepts or pilots
* Training tool for Azure Arc skills development
* Demo environment for customer presentations or events
* Rapid integration testing platform

## Azure Arc capabilities available in ArcBox

### Azure Arc enabled Servers

![ArcBox servers diagram](./servers.png)

ArcBox includes three Azure Arc enabled server resources that are hosted using nested virtualization in Azure. As part of the deployment, a Hyper-V host (ArcBox-Client) is deployed with three guest virtual machines. These machines, _ArcBoxWin_, _ArcBoxUbuntu_, and _ArcBoxSQL_ are connected as Azure Arc enabled servers via the ArcBox automation.

### Azure Arc enabled Kubernetes

![ArcBox Kubernetes diagram](./k8s.png)

ArcBox deploys one single-node Rancher K3s cluster running on an Azure virtual machine. This cluster is then connected to Azure as an Azure Arc enabled Kubernetes resource (_ArcBox-K3s_).

### Azure Arc enabled Data Services

ArcBox deploys one single-node Rancher K3s cluster (_ArcBox-CAPI-MGMT_), which is then transformed to a [Cluster API](https://cluster-api.sigs.k8s.io/user/concepts.html) management cluster with the Azure CAPZ provider, and a workload cluster is deployed onto the management cluster. The Azure Arc enabled data services and data controller are deployed onto this workload cluster via a PowerShell script that runs when first logging into ArcBox-Client virtual machine.

![ArcBox data services diagram](./dataservices2.png)

### Hybrid Unified Operations

ArcBox deploys several management and operations services that work with ArcBox's Azure Arc resources. These resources include an an Azure Automation account, an Azure Log Analytics workspace with the Update Management solution, Azure Policy assignments for deploying Log Analytics agents on Windows and Linux, Azure Policy assignment for adding tags to resources, and a storage account used for staging resources needed for the deployment automation.

![ArcBox unified operations diagram](./unifiedops.png)

## Automation flow

![Deployment flow diagram](./deploymentflow.png)

ArcBox uses an advanced automation flow to deploy and configure all necessary resources with minimal user interaction. The above diagram provides a high-level overview of the deployment flow. A high-level summary of the deployment is:

* User deploys the primary ARM template (azuredeploy.json). This template contains several nested templates that will run simultaneously.
  * ClientVM ARM template - deploys the Client Windows VM. This is the Hyper-V host VM where all user interactions with the environment are made from.
  * CAPI ARM template - deploys an Ubuntu Linux VM which will have Rancher (K3s) installed and transformed into a Cluster API management cluster via the Azure CAPZ provider.
  * Rancher K3s template - deploys an Ubuntu Linux VM which will have Rancher (K3s) installed on it and connected as an Azure Arc enabled Kubernetes cluster
  * Storage account template - used for staging files in automation scripts
  * Management artifacts template - deploys Azure Log Analytics workspace and solutions and Azure Policy artifacts
* User remotes into Client Windows VM, which automatically kicks off multiple scripts that:
  * Deploy and configure three (3) nested virtual machines in Hyper-V
    * Windows VM - onboarded as Azure Arc enabled Server
    * Ubuntu VM - onboarded as Azure Arc enabled Server
    * Windows VM running SQL Server - onboarded as Azure Arc enabled SQL Server (as well as Azure Arc enabled Server)
  * Deploy and configure Azure Arc enabled data services on the CAPI workload cluster including a data controller, a SQL MI instance, and a PostgreSQL Hyperscale cluster. After deployment, Azure Data Studio opens automatically with connection entries for each database instance. Data services deployed by the script are:
    * Data controller
    * SQL MI instance
    * Postgres instance

## ArcBox Azure Region Compatibility

ArcBox must be deployed to one of the following regions. Deploying ArcBox outside of these regions may result in unexpected results or deployment errors.

* East US
* East US 2
* West US 2
* North Europe
* West Europe
* UK South
* Southeast Asia
* Australia East

## Prerequisites

* ArcBox requires 52 vCPUs when deploying with default parameters such as VM series/size. Ensure you have sufficient vCPU quota available in your Azure subscription.

* [Install or update Azure CLI to version 2.15.0 and above](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest). Use the below command to check your current installed version.

  ```shell
  az --version
  ```

* Login to AZ CLI using the ```az login``` command.

* Create Azure service principal (SP)

    To deploy ArcBox an Azure service principal assigned with the "Contributor" role is required. To create it login to your Azure account run the below command (this can also be done in [Azure Cloud Shell](https://shell.azure.com/)).

    ```shell
    az login
    az ad sp create-for-rbac -n "<Unique SP Name>" --role contributor
    ```

    For example:

    ```shell
    az ad sp create-for-rbac -n "http://AzureArcBox" --role contributor
    ```

    Output should look like this:

    ```json
    {
    "appId": "XXXXXXXXXXXXXXXXXXXXXXXXXXXX",
    "displayName": "AzureArcBox",
    "name": "http://AzureArcBox",
    "password": "XXXXXXXXXXXXXXXXXXXXXXXXXXXX",
    "tenant": "XXXXXXXXXXXXXXXXXXXXXXXXXXXX"
    }
    ```

    > **Note: The Jumpstart scenarios are designed with as much ease of use in-mind and adhering to security-related best practices whenever possible. It is optional but highly recommended to scope the service principal to a specific [Azure subscription and resource group](https://docs.microsoft.com/en-us/cli/azure/ad/sp?view=azure-cli-latest) as well considering using a [less privileged service principal account](https://docs.microsoft.com/en-us/azure/role-based-access-control/best-practices)**

* [Generate SSH Key](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/) (or use existing ssh key)

## Deployment Option 1: Azure Portal

* Click the <a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fmicrosoft%2Fazure_arc%2Fmain%2Fazure_jumpstart_arcbox%2Fazuredeploy.json" target="_blank"><img src="https://aka.ms/deploytoazurebutton"/></a> button and enter values for the the ARM template parameters.

  ![Screenshot showing Azure Portal deployment of ArcBox](./portaldeploy.png)

  ![Screenshot showing Azure Portal deployment of ArcBox](./portaldeployinprogress.png)

  ![Screenshot showing Azure Portal deployment of ArcBox](./portaldeploymentcomplete.png)

## Deployment Option 2: Azure CLI

* Clone the Azure Arc Jumpstart repository

    ```shell
    git clone https://github.com/microsoft/azure_arc.git
    ```

* Edit the [azuredeploy.parameters.json](../../azure_jumpstart_arcbox/azuredeploy.parameters.json) ARM template parameters file and supply some values for your environment.

  * *sshRSAPublicKey* - Your SSH public key
  * *spnClientId* - Your Azure service principal id
  * *spnClientSecret* - Your Azure service principal secret
  * *spnTenantId* - Your Azure tenant id
  * *windowsAdminUsername* - Client Windows VM Administrator name
  * *windowsAdminPassword* - Client Windows VM Administrator password
  * *myIpAddress* - Your local IP address. This is used to allow remote RDP and SSH connections to the Client Windows VM and K3s Rancher VM.
  * *logAnalyticsWorkspaceName* - Unique name for the ArcBox log analytics workspace

    ![Screenshot showing example parameters](./parameters.png)

* Now you will deploy the ARM template. Navigate to the local cloned [deployment folder](../../azure_jumpstart_arcbox) and run the below command:

  ```shell
  az group create --name <Name of the Azure resource group> --location <Azure Region>
  az deployment group create \
  --resource-group <Name of the Azure resource group> \
  --template-file azuredeploy.json \
  --parameters azuredeploy.parameters.json 
  ```

  ![Screenshot showing az group create](./azgroupcreate.png)

  ![Screenshot showing az deployment group create](./azdeploy.png)

* After deployment, you should see the ArcBox resources inside your resource group.

  ![Screenshot showing az deployment group create](./deployedresources.png)

* Open a remote desktop connection into _ArcBox-Client_. Upon logging in, multiple automated scripts will open and start running. These scripts usually take 10-20 minutes to finish and once completed the script windows will close. At this point, the deployment is complete.

  ![Screenshot showing ArcBox-Client](./automation5.png)

  ![Screenshot showing ArcBox resources in Azure Portal](./rgarc.png)

## Using ArcBox

After deployment is complete, its time to start exploring ArcBox. Most interactions with ArcBox will take place either from Azure itself (Azure Portal, CLI or similar) or from inside the ArcBox-Client virtual machine. When remoted into the client VM, here are some things to try:

* Open Hyper-V and access the Azure Arc enabled servers
  * Username: arcdemo
  * Password: ArcDemo123!!

  ![Screenshot showing ArcBox Client VM with Hyper-V](./hypervterminal.png)

* Use the included [kubectx](https://github.com/ahmetb/kubectx) tool to switch Kubernetes contexts between the Rancher K3s and AKS clusters.

  ```shell
  kubectx
  kubectx arcbox-capi
  kubectl get nodes
  kubectl get pods -n arcdatactrl
  kubectx arcboxk3s
  kubectl get nodes
  ```

  ![Screenshot showing usage of kubectx](./kubectx.png)

* Login to the Azure Arc data controller with [Azdata CLI](https://docs.microsoft.com/en-us/sql/azdata/reference/reference-azdata-arc?view=sql-server-ver15) and explore its functionality.
  * Azdata username: arcdemo
  * Azdata password: ArcPassword123!!
  * Namespace: arcdatactrl
  
  ```shell
  azdata login --username arcdemo --namespace arcdatactrl
  azdata arc dc status show
  azdata arc sql endpoint list
  azdata arc postgres endpoint list
  ```

  ![Screenshot showing Azdata CLI usage](./azdatausage.png)

* Open Azure Data Studio and explore the SQL MI and PostgreSQL Hyperscale instances.

  ![Screenshot showing Azure Data Studio usage](./azdatastudio.png)

### Included tools

| Tool                                                 | Location      |
| ---------------------------------------------------- | ------------- |
| Azure Data Studio with Arc and PostgreSQL extensions | ArcBox-Client |
| kubectl, kubectx, helm                               | ArcBox-Client |
| Chocolatey                                           | ArcBox-Client |
| Visual Studio Code                                   | ArcBox-Client |
| Putty                                                | ArcBox-Client |
| 7zip                                                 | ArcBox-Client |
| Terraform                                            | ArcBox-Client |
| Git                                                  | ArcBox-Client |

### Next steps
  
ArcBox is a sandbox that can be used for a large variety of use cases, such as an environment for testing and training or kickstarter for proof of concept projects. Ultimately, you are free to do whatever you wish with ArcBox. Some suggested next steps for you to try in your ArcBox are:

* Login to the Azure Arc data controller using azdata and explore the functionality provided by the data controller
* Deploy sample databases to the PostgreSQL Hyperscale instance or to the SQL Managed Instance
* Use the included kubectx to switch contexts between the two Kubernetes clusters
* Deploy GitOps configurations with Azure Arc enabled Kubernetes
* Build policy initiatives that apply to your Azure Arc enabled resources
* Write and test custom policies that apply to your Azure Arc enabled resources
* Incorporate your own tooling and automation into the existing automation framework
* Build a certificate/secret/key management strategy with your Azure Arc resources

Do you have an interesting use case to share? Submit an issue on GitHub with your idea and we will consider it for future releases!

## Clean up the deployment

To clean up your deployment, simply delete the resource group using Azure CLI or Azure Portal.

```shell
az group delete -n <name of your resource group>
```

![Screenshot showing az group delete](./azdelete.png)

![Screenshot showing group delete from Azure Portal](./portaldelete.png)

## Known issues

* Azure Arc enabled SQL Server assessment report not always visible in Azure Portal
* Currently, Azure Arc enabled data services are deployed in **indirectly connected** mode.
* The [_custom-location_](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/custom-locations) feature required for Azure Arc enabled data services directly connected mode currently cannot be installed using a service principal and will present an "Insufficient privileges" error as part of the data services logon script runtime that can be safely ignored for now.

    ![Screenshot showing custom location "Insufficient privileges" error](./customlocationerror.png)
