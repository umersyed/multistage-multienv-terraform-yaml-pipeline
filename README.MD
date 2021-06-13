https://medium.com/microsoftazure/creating-a-single-secure-azure-devops-yaml-pipeline-to-provision-multiple-environments-using-620900aae18

## Purpose
This repository has artifacts supporting the blog post "Creating a single Azure DevOps yaml pipeline to provision multiple environments using terraform, with manual judgement between the terraform plan and terraform apply steps".

Creating a single secure Azure DevOps yaml pipeline to provision multiple environments using terraform, with manual approval between the terraform steps
Maninderjit (Mani) Bindra
Maninderjit (Mani) Bindra
Following
Jan 8, 2020 · 13 min read





In a recent project I was involved in, we had the following requirements:
We needed to create a single Azure DevOps (AzDO) yaml pipeline, which used terraform and could provision infrastructure for different environments (like development, testing and production). This means that the person executing the pipeline can pass in a parameter for the environment name (dev, tst, or prd) and the pipeline would provision / update the infrastructure for that environment.
All secrets needed to be in Azure key vault.
The service principals created for AzDO and terraform need to be given minimal access.
There was a need for manual approval between the terraform plan and terraform apply steps so that the terraform plan output can be verified prior to the infrastructure changes being applied.
Contextual view of the requirements

In this post I am sharing the approach we took to meet these requirements.
Solution Overview

The approach we took had the following key points
A naming convention for the resources was agreed upon. The naming convention used in this post is a simplified version of that, where most resources are prefixed with the environment name, which is dev for development, tst for testing, and prd for production.
Similar naming convention is followed for service connections to the different subscriptions created in AzDO.
In the pipeline the only parameter passed is the environment (dev, tst ,etc) and based on that the AzDO yaml pipeline provisions resources in the appropriate environment. This will become clear in the Pipeline steps section.
For enable manual approval between the terraform plan and apply steps we have used the multistage pipeline with the environments property. This will be explained in the Enabling Manual approval section
Sample git repository for this post
The sample git repository is https://github.com/maniSbindra/multienv-multistage-terraform-yaml-pipeline . The file / folder structure is as follows:

At the root folder we have the multi environment provisioning yaml file “azure-pipelines-multi-environment.yml” and the multi environment, multi stage provisioning, with manual approval file “azure-pipelines-multi-env-multi-stage-with-approval”.
The tf-modules folder contains the sub modules used by the main module. In the sample we just have a single Azure Kubernetes Service (AKS) module under this folder.
The tf-infra-provision folder contains the main module. In our sample the main module includes the aks sub module (modules.tf). There is a sub folder called tf-vars, which has the environment specific tfvars files (dev.tfvars, tst.tfvars etc). Terraform variables which have secret values are fetched from the key vault by the AzDO pipeline. The not secret values for the environment are fetched from the tfvars files. The file below is the dev.tfvars
location            = "westus"
aks_resource_group_name = "dev-provisioning-rg"
aks_name            = "dev-mani-blogpost-aks"
aks_dns_name        = "devmaniblogpost"
kubernetes_version  = "1.13.10"
cluster_size        = "1"
vm_size             = "Standard_D1_v2"
environment         = "dev"
# Following are injected from the pipeline
# aks_sp_id           = ""
# aks_sp_secret       = ""
Setup of Pipeline dependencies
In this section we setup the dev environment dependencies (in the dev subscription) which are needed prior to our sample AzDO pipeline being executed for the dev environment. These dependencies include resource groups, key vault / key vault secrets, and the service principals (SPs) needed by Azure DevOps and Terraform.
The scripts below explicitly use the prefix “dev”. You can make minor tweaks to these scripts to create generic environment agnostic scripts for provisioning of the pipeline dependencies
Create the key vault resource group and the key vault
az group create -n dev-pipeline-dependencies-rg -l westus
az keyvault create -n dev-pipeline-secrets-kv -g dev-pipeline-dependencies-rg -l westus
Make a note of the key vault resource id which will be in the format “/subscriptions/XXXXXXXX-XX86–47XX-X8Xf-XXXXXXXXXX/resourceGroups/dev-pipeline-dependencies-rg/providers/Microsoft.KeyVault/vaults/dev-pipeline-secrets-kv”. We will need this in a subsequent step to give the AzDO dev subscription SP access to this key vault
Create storage account and container where terraform will store the state file for the environment
az group create -n dev-terraform-backend-rg -l westus
# Create storage account
az storage account create --resource-group dev-terraform-backend-rg --name devterraformbackendsa --sku Standard_LRS --encryption-services blob
# Get storage account key
ACCOUNT_KEY=$(az storage account keys list --resource-group dev-terraform-backend-rg --account-name devterraformbackendsa --query [0].value -o tsv)
# Create blob container
az storage container create --name terraform-backend-files --account-name devterraformbackendsa --account-key $ACCOUNT_KEY
Make a note of the storage account id which will in the format “/subscriptions/XXXXXXXX-XX86–47XX-X8Xf-XXXXXXXXXX/resourceGroups/dev-terraform-backend-rg/providers/Microsoft.Storage/storageAccounts/devterraformbackendsa” . This will be needed when providing the terraform service principal access to the storage account container.
Add the storage account key as a secret in the key vault. This will be read in the AzDO pipeline.
az keyvault secret set --vault-name dev-pipeline-secrets-kv --name "tf-backend-sa-access-key" --value "$ACCOUNT_KEY"
Create a resource group where terraform will provision the resources
az group create -n dev-provisioning-rg -l westus
Create terraform service principal with required access and add corresponding secrets to key vault
You will need jq installed to execute some of the commands below
The example below just gives the terraform service principal minimal access, you can provide access as per your requirements. When using minimal access as described in this post, if you do not have the “Microsoft.Kusto” provider registered, you may get an error as described in the github issue https://github.com/terraform-providers/terraform-provider-azurerm/issues/4488 . To work around this you can register the provider by executing the az command “az provider register -n Microsoft.Kusto”
# Create terraform service principal
TF_SP=$(az ad sp create-for-rbac -n dev-tf-sp --role contributor --scopes "/subscriptions/XXXXXXXX-XX86-47XX-X8Xf-XXXXXXXXXX/resourceGroups/dev-terraform-backend-rg/providers/Microsoft.Storage/storageAccounts/devterraformbackendsa" "/subscriptions/XXXXXXXX-XX86-47XX-X8Xf-XXXXXXXXXX/resourceGroups/dev-provisioning-rg"  )
# Client ID of the service principal
TF_CLIENT_ID=$(echo $TF_SP | jq '.appId' | sed 's/"//g')
# Client secret of the service principal
TF_CLIENT_SECRET=$(echo $TF_SP | jq '.password' | sed 's/"//g')
# Set your tenant ID
TF_TENANT_ID="your-tenant-id"
# Set your subscription ID
TF_SUBSCRIPTION="your-subcription-id"
# Add the values as secrets to key vault
az keyvault secret set --vault-name dev-pipeline-secrets-kv --name "tf-sp-id" --value "$TF_CLIENT_ID"
az keyvault secret set --vault-name dev-pipeline-secrets-kv --name "tf-sp-secret" --value "$TF_CLIENT_SECRET"
az keyvault secret set --vault-name dev-pipeline-secrets-kv --name "tf-tenant-id" --value "$TF_TENANT_ID"
az keyvault secret set --vault-name dev-pipeline-secrets-kv --name "tf-subscription-id" --value "$TF_SUBSCRIPTION"
Create AKS service principal and add corresponding secrets to key vault
Note this step is only needed because the terraform template being used is provisioning an AKS cluster. This step can be skipped if your template does not need this service principal
AKS_SP=$(az ad sp create-for-rbac -n dev-aks-sp --skip-assignment)
AKS_CLIENT_ID=$(echo $AKS_SP | jq '.appId' | sed 's/"//g')
AKS_CLIENT_SECRET=$(echo $AKS_SP | jq '.password' | sed 's/"//g')
az keyvault secret set --vault-name dev-pipeline-secrets-kv --name "aks-sp-id" --value "$AKS_CLIENT_ID"
az keyvault secret set --vault-name dev-pipeline-secrets-kv --name "aks-sp-secret" --value "$AKS_CLIENT_SECRET"
Create a SP for AzDO with access to key vault secrets.
This SP will be used by the AzDO pipeline. We will also create an AzDO service connection using this SP
AzDO_SP=$(az ad sp create-for-rbac -n dev-azdo-sp --skip-assignment)
AzDO_CLIENT_ID=$(echo $AzDO_SP | jq '.appId' | sed 's/"//g')
AzDO_CLIENT_SECRET=$(echo $AzDO_SP | jq '.password' | sed 's/"//g')
DEV_SUBSCRIPTION_ID="your-subscription"
DEV_SUBSCRIPTION_NAME="your-subscription-name"
TENANT_ID="your-tenant-id"
Now we give the AzDO SP access to get pipeline secrets
Make sure to replace the resource Id of the key vault in the scope argument by your key vault resource id.
az role assignment create --assignee $AzDO_CLIENT_ID --scope "/subscriptions/XXXXXXXX-XX86-47XX-X8Xf-XXXXXXXXXX/resourceGroups/dev-pipeline-dependencies-rg/providers/Microsoft.KeyVault/vaults/dev-pipeline-secrets-kv" --role "reader"
az keyvault set-policy --name dev-pipeline-secrets-kv --spn $AzDO_CLIENT_ID --subscription $DEV_SUBSCRIPTION_ID --secret-permissions get
Next we need to create an AzDO service connection using the above service principal. To do this from the command like we will need the az azure devops extension. We can also create the service conection using the AzDO portal as shown here. If adding the service connection using the command below, make a note of the SP password value ($AzDO_CLIENT_SECRET), as the command will prompt you for this.
az devops service-endpoint azurerm create --azure-rm-service-principal-id $AzDO_CLIENT_ID --azure-rm-subscription-id $DEV_SUBSCRIPTION_ID --azure-rm-subscription-name $DEV_SUBSCRIPTION_NAME --azure-rm-tenant-id $TENANT_ID --name dev-sp --organization "https://dev.azure.com/your-org" --project "your-project"
The name of the service connection is important as we need to have the environment prefix in it. In the case above the service connection is for the dev environment.
Once the service connection is created you will be able to see it and update it in the AzDO portal

Note that since in our example the AzDO service principal does not have access to the subscription or resource group, it just has access to the key vault, the “verify connection” may fail.
Pipeline Steps
If you have never created an AzDO yaml pipeline you can have a look at https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema?view=azure-devops&tabs=schema .
Now Let us first go through the single stage pipeline without any manual approval between the terraform plan and apply. This pipeline accepts a parameter called environment, and provisions resources using terraform in that environment. Let us look at the pipeline:
variables:
  InfraProvisioningResoureGroupName: $(environment)-provisioning-rg
  tfBackendStorageAccountName: $(environment)terraformbackendsa
  tfBackendStorageContainerName: terraform-backend-files
  tfBackendFileName: $(environment)-tf-state-file
  tfvarsFile: $(environment).tfvars
pool:
  vmImage: 'ubuntu-latest'
steps:
#PARAMETER VALIDATION
- script: |
    set +e
    if [ -z $(environment) ]; then
    echo "target environment not specified";
    exit 1;
    fi
    echo "environment is:" $(environment)
displayName: 'Verify that the environment parameter has been supplied to pipeline'
#KEY VAULT TASK
- task: AzureKeyVault@1
  inputs:
    azureSubscription: '$(environment)-sp'
    KeyVaultName: '$(environment)-pipeline-secrets-kv'
    SecretsFilter: 'tf-sp-id,tf-sp-secret,tf-tenant-id,tf-subscription-id,tf-backend-sa-access-key,aks-sp-id,aks-sp-secret'
  displayName: 'Get key vault secrets as pipeline variables'
# INSTALLING REQUIRED VERSION OF 
- task: ms-devlabs.custom-terraform-tasks.custom-terraform-installer-task.TerraformInstaller@0
  displayName: 'Install Terraform 0.12.3'
# AZURE CLI TASK
- task: AzureCLI@1
  inputs:
    azureSubscription: '$(environment)-sp'
    scriptLocation: 'inlineScript'
    inlineScript: 'terraform version'
  displayName: "Terraform Version"
# AZ LOGIN USING TERRAFORM SERVICE PRINCIPAL
- script: |
    az login --service-principal -u $(tf-sp-id) -p $(tf-sp-secret) --tenant $(tf-tenant-id)
    cd $(System.DefaultWorkingDirectory)/tf-infra-provision
# TERRAFORM INIT    
    echo '#######Terraform Init########'
    terraform init -backend-config="storage_account_name=$(tfBackendStorageAccountName)" -backend-config="container_name=$(tfBackendStorageContainerName)" -backend-config="access_key=$(tf-backend-sa-access-key)" -backend-config="key=$(tfBackendFileName)"
# TERRAFORM PLAN    
    echo '#######Terraform Plan########'
    terraform plan -var-file=./tf-vars/$(tfvarsFile) -var="client_id=$(tf-sp-id)" -var="client_secret=$(tf-sp-secret)" -var="tenant_id=$(tf-tenant-id)" -var="subscription_id=$(tf-subscription-id)" -var="aks_sp_id=$(aks-sp-id)" -var="aks_sp_secret=$(aks-sp-secret)" -out="out.plan"
# TERRAFORM APPLY    
    echo '#######Terraform Apply########'
    terraform apply out.plan
    
  displayName: 'Terraform Init, Plan and Apply '
Setting Parameter for pipeline with Default value
To add a parameter to the pipeline we click on variable, and then add the variable “environment” with a default value of “dev”. Check the option to enable user to override this value.

Variables Section
Most variables for resource group and storage account etc have the prefix as the variable for environment (which is passed when the pipeline is executed). This is in alignment with the naming convention of the resources. Similarly the subscription names in the tasks are prefixed with the variable for environment name, which is in accordance with the naming convention of the service connections (dev-sp, tst-sp etc)
Parameter Validation
The pipeline next validates if the environment name parameter has been passed or not. If environment name variable is not passed the pipeline throws an error and exits
Key Vault task
Next we use the Keyvault task to fetch the required secrets from the key vault. All fetched secrets can be used as variables in the pipeline. For instance a secret “tf-sp-id” in the secret filter section of this task can be used in the pipeline as $(tf-sp-id). This makes accessing the secrets convinient
az login using Terraform SP
Next we use the az cli task and perform an az login using the terraform SP secrets fetched from the key vault
Terraform init, plan and apply
In the terraform init we initialize terraform to connect to the state file in azure blob storage, All required secrets are available post the key vault task. The terraform plan next creates an output file which is then consumed by the apply stage. The environment specific secrets and tfvars files are passed to the terraform plan and apply steps
Enabling Manual Approval
At the time of writing this post AzDO multistage yaml pipelines were in preview. You can have a look at these docs regarding how to enable preview features. We used the environment feature (with environment checks) of multistage yaml pipeline to implement this requirement. Let us look at the enhanced pipeline with multiple stages and approval added.
variables:
  InfraProvisioningResoureGroupName: $(environment)-provisioning-rg
  tfBackendStorageAccountName: $(environment)terraformbackendsa
  tfBackendStorageContainerName: terraform-backend-files
  tfBackendFileName: $(environment)-tf-state-file
  tfvarsFile: $(environment).tfvars
stages: 
  - stage: terraform_plan
    jobs:
    - deployment: Terraform_Plan
      displayName: Terraform_Plan
      pool:
        vmImage: 'Ubuntu-16.04'
      # creates an environment if it doesn't exist
      environment: 'terraform_plan'
      strategy:
        runOnce:
          deploy:
            steps:
            - checkout: self
            - script: |
                set +e
                if [ -z $(environment) ]; then
                echo "target environment not specified";
                exit 1;
                fi
                echo "environment is:" $(environment)
              displayName: 'Verify that the environment parameter has been supplied to pipeline'
            - task: AzureKeyVault@1
              inputs:
                azureSubscription: '$(environment)-sp'
                KeyVaultName: '$(environment)-pipeline-secrets-kv'
                SecretsFilter: 'tf-sp-id,tf-sp-secret,tf-tenant-id,tf-subscription-id,tf-backend-sa-access-key,aks-sp-id,aks-sp-secret'
              displayName: 'Get key vault secrets as pipeline variables'
            - task: ms-devlabs.custom-terraform-tasks.custom-terraform-installer-task.TerraformInstaller@0
              displayName: 'Install Terraform 0.12.3'
            - task: AzureCLI@1
              inputs:
                azureSubscription: '$(environment)-sp'
                scriptLocation: 'inlineScript'
                inlineScript: 'terraform version'
              displayName: "Terraform Version"
            - script: |
                az login --service-principal -u $(tf-sp-id) -p $(tf-sp-secret) --tenant $(tf-tenant-id)
                cd $(System.DefaultWorkingDirectory)/tf-infra-provision
                
                echo '#######Terraform Init########'
                terraform init -backend-config="storage_account_name=$(tfBackendStorageAccountName)" -backend-config="container_name=$(tfBackendStorageContainerName)" -backend-config="access_key=$(tf-backend-sa-access-key)" -backend-config="key=$(tfBackendFileName)"
                
                echo '#######Terraform Plan########'
                terraform plan -var-file=./tf-vars/$(tfvarsFile) -var="client_id=$(tf-sp-id)" -var="client_secret=$(tf-sp-secret)" -var="tenant_id=$(tf-tenant-id)" -var="subscription_id=$(tf-subscription-id)" -var="aks_sp_id=$(aks-sp-id)" -var="aks_sp_secret=$(aks-sp-secret)" -out="out.plan"
  - stage: terraform_apply
    jobs:
    - deployment: Terraform_Apply
      displayName: terraform_apply
      pool:
        vmImage: 'Ubuntu-16.04'
      # creates an environment if it doesn't exist  
      environment: 'terraform_apply'
      strategy:
        runOnce:
          deploy:
            steps:
            - checkout: self
            - script: |
                set +e
                if [ -z $(environment) ]; then
                echo "target environment not specified";
                exit 1;
                fi
                echo "environment is:" $(environment)
              displayName: 'Verify that the environment parameter has been supplied to pipeline'
            - task: AzureKeyVault@1
              inputs:
                azureSubscription: '$(environment)-sp'
                KeyVaultName: '$(environment)-pipeline-secrets-kv'
                SecretsFilter: 'tf-sp-id,tf-sp-secret,tf-tenant-id,tf-subscription-id,tf-backend-sa-access-key,aks-sp-id,aks-sp-secret'
              displayName: 'Get key vault secrets as pipeline variables'
            - task: ms-devlabs.custom-terraform-tasks.custom-terraform-installer-task.TerraformInstaller@0
              displayName: 'Install Terraform 0.12.3'
            - task: AzureCLI@1
              inputs:
                azureSubscription: '$(environment)-sp'
                scriptLocation: 'inlineScript'
                inlineScript: 'terraform version'
              displayName: "Terraform Version"
            - script: |
                az login --service-principal -u $(tf-sp-id) -p $(tf-sp-secret) --tenant $(tf-tenant-id)
                cd $(System.DefaultWorkingDirectory)/tf-infra-provision
                
                echo '#######Terraform Init########'
                terraform init -backend-config="storage_account_name=$(tfBackendStorageAccountName)" -backend-config="container_name=$(tfBackendStorageContainerName)" -backend-config="access_key=$(tf-backend-sa-access-key)" -backend-config="key=$(tfBackendFileName)"
                
                echo '#######Terraform Plan########'
                terraform plan -var-file=./tf-vars/$(tfvarsFile) -var="client_id=$(tf-sp-id)" -var="client_secret=$(tf-sp-secret)" -var="tenant_id=$(tf-tenant-id)" -var="subscription_id=$(tf-subscription-id)" -var="aks_sp_id=$(aks-sp-id)" -var="aks_sp_secret=$(aks-sp-secret)" -out="out.plan"
                echo '#######Terraform Apply########'
                terraform apply out.plan
As you will see above we now have 2 stages in the pipeline. The first stage is for terraform plan, and the second one for terraform apply. Steps in both stages are almost similar to the pipeline we had earlier. Some of the key differences have been highlighted. They are :
The first stage has a “environment” field added in the yaml and its value is “terraform_plan”. In the second stage the value of this field is “terraform_apply”. At the time when we were working on this project we found that to add manual approval in a multistage yaml pipeline, an easy way was by adding the environment field in the stage where manual approval is needed, and then adding manual approval check (which we will see later) to the environment.
In the first stage we do only the terraform plan step.
The second stage which should be executed after the approval post stage 1 has happened. In this stage we execute the terraform plan again, followed by terraform apply. Since we are executing the terraform plan and apply again post approval, there is a risk that if some manual infrastructure changes take place while the approval happens, then the approved terraform plan and applied terraform plan may be different. After evaulating other options, this option was selected.
Adding Manual approval check to the terraform_apply
In AzDO click on environments. If your multistage pipeline has executed once you will already see the terraform_plan and terraform_apply environments. If not you can create these two environments. The name should match the names in the environment fields of the pipeline yaml.

Next we click on the “terraform_apply” environment, and add a manual approval check as shown below

After you click on check you can add the approver and instructions to approver

That is it we have added the manual approval between the terraform plan and terrorm apply.
Let us look at the pipeline in action.
Execution
Click on “Run Pipeline”, and you will be asked to enter the environment

After we run the pipeline, the terraform plan stage will be executed and then the second stage will get executed only after manual approval. The screen shot below shows the pipeline is blocked for approval before the terraform apply stage

The approver can then validate the terraform plan

If the terraform plan is as expected the approver can approve and the terraform apply stage will get executed

The screenshot below shows the state of the pipeline after terraform apply has hapenned.

The following link gives more detail on adding approval checks for environments : https://docs.microsoft.com/en-us/azure/devops/pipelines/process/approvals?view=azure-devops
Microsoft Azure