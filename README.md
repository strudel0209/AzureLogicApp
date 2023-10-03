# LogicApp-Terraform-Deploy

This project provides examples on how to use Terraform and Azure DevOps to create standard Logic App. 

# Terraform

## LAstandard.tf

Find the *Terraform/LAstandard.tf* and check the terraform code.
In this code you can see that we will be creating the following resources on Azure:

1. The resource group which your Logic App resources will de deployed to.
2. The storage account hosting the Logic App standard.
3. The app service plan.
4. The application insights and the associated log analytics workspace.
5. The standard Logic App

# Azure Pipelines

## Create a project

- To create a pipeline, first we will need to create a new project in Azure DevOps
- Once the project is created, we can go to the project-Repos, and import the code from Github or git push code from existing repository. 

## Create a service connection

- Before we create a pipeline, to deploy the resources to Azure, we will need to create a service connection, by clicking the project settings.
- Then go to the service connections, create service connection. Choose Azure Resource Manager->Service principal
- Select the subscription and resource group you would like to connect, give the connection a name, and save. Now we are ready to use this service connection in the pipeline.
- If the subscription doesn't appear in the drop-down, you will need to create an Azure App Registration and add the Service Principal manually by entering: SubscriptionId / Name, TenantId,Service Principal Id and Service Principal Key (Client secret in Azure portal)

## Create a pipeline

Now we can go back to the project and create a pipeline. 

- Select "New pipeline"->"Azure Repos Git"-> "Existing Azure Pipelines YAML file"  from the project repo. 
- Choose the *"logic-app-pipeline-infra.yml"* file in the repository. This pipeline puts CI and CD together, including two stages, to create the infrastructures for Logic App:

### Stage 1: Build

This stage will build and publish the artifact from path *"Terraform/LAstandard.tf"*

```terraform
stages:
- stage: Build
  displayName: 'Build Artifact'
  jobs:
  - job: logic_app_build
    displayName: 'Build and publish terraformcode'
    steps:
    - task: CopyFiles@2
      displayName: 'Copy Terraform files to artifacts'
      inputs:
        SourceFolder: Terraform
        TargetFolder: '$(build.artifactstagingdirectory)/Terraform'

    - task: PublishBuildArtifacts@1
      displayName: 'Publish Artifact'
      inputs:
        PathtoPublish: '$(Build.ArtifactStagingDirectory)'
        ArtifactName: 'drop'
```

### Stage 2: Deploy

This stage has the following steps:

1. An Azure CLI task to create the storage for terraform. This storage account is different than the Logic App hosting storage.  By default, Terraform stores state locally in a file named *terraform.tfstate*. With remote state, Terraform writes the state data to a remote data store. This task is commented out. If you already have a storage account to save the terraform files, it can be skipped.

```terraform
- task: AzureCLI@1
      displayName: 'Azure CLI to create storage account for terraform'
      inputs:
        azureSubscription: '47324483-dabd-4295-aedf-829cc6cf68ab'
        scriptLocation: inlineScript
        inlineScript: |
         #this will create Azure resource group
         call az group create --location westeurope --name $(terraformstoragerg)      
         call az storage account create --name $(terraformstorageaccount) --resource-group $(terraformstoragerg) --location westeurope --sku Standard_LRS       
         call az storage container create --name terraform --account-name $(terraformstorageaccount)
         call az storage account keys list -g $(terraformstoragerg) -n $(terraformstorageaccount)
```

2.  We will use the values specified in the variables:

```terraform
variables:
#the storage account information for the terraform state data, replace with your own resource
  terraformstoragerg: 'tfdeploy'
  terraformstorageaccount: 'tfdeploy'
  storagekey: 'PipelineWillGetThisValueRuntime'
  terraformstorageaccountcontainer: 'terraform'
  ```

3.  Download the artifact

```terraform
- task: DownloadPipelineArtifact@2
      inputs:
        artifact: 'drop'
        path: '$(Build.ArtifactStagingDirectory)'
        
```

4.  A PowerShell script to get the storage key

```terraform
 - task: AzurePowerShell@3
      displayName: 'Azure PowerShell script to get the storage key'
      inputs:
        azureSubscription: 'ADO-R2F' #replace this with your own service connection name
        ScriptType: InlineScript
        Inline: |
         # using this script to fetch storage key required in terraform file
        
         $key=(Get-AzureRmStorageAccountKey -ResourceGroupName $(terraformstoragerg) -AccountName $(terraformstorageaccount)).Value[0]
        
         Write-Host  "##vso[task.setvariable variable=storagekey]$key"
        azurePowerShellVersion: LatestVersion
```

5.  Replace the token in the *LAstandard.tf* file, started with "__"  in the terraform code. For example "__storagekey__". This task will replace the token value with the values got from the PowerShell task.

```terraform
 - task: qetza.replacetokens.replacetokens-task.replacetokens@5
      displayName: 'Replace tokens in terraform file'
      inputs:
        targetFiles: '$(Build.ArtifactStagingDirectory)/Terraform/*.tf'
        tokenPattern: custom
        tokenPrefix: '__'
        tokenSuffix: '__'
```

6.  Install the latest version of Terraform.

```terraform
 - task: ms-devlabs.custom-terraform-tasks.custom-terraform-installer-task.TerraformInstaller@0
      displayName: 'Install Terraform latest'
```

7.  Terraform init->plan->apply to initialize, plan, and deploy the Logic App using Terraform.

```terraform
 - task: ms-devlabs.custom-terraform-tasks.custom-terraform-release-task.TerraformTaskV2@2
      displayName: 'Terraform : init'
      inputs:
        workingDirectory: '$(Build.ArtifactStagingDirectory)/Terraform'
        backendServiceArm: 'ADO-R2F' #replace this with your own service connection name
        backendAzureRmResourceGroupName: '$(terraformstoragerg)'
        backendAzureRmStorageAccountName: '$(terraformstorageaccount)'
        backendAzureRmContainerName: '$(terraformstorageaccountcontainer)'
        backendAzureRmKey: terraform.tfstate

task: ms-devlabs.custom-terraform-tasks.custom-terraform-release-task.TerraformTaskV2@2
      displayName: 'Terraform : plan'
      inputs:
        command: plan
        workingDirectory: '$(Build.ArtifactStagingDirectory)/Terraform'
        environmentServiceNameAzureRM: 'ADO-R2F' #replace this with your own service connection name

    - task: ms-devlabs.custom-terraform-tasks.custom-terraform-release-task.TerraformTaskV2@2
      displayName: 'Terraform :  apply -auto-approve'
      inputs:
        command: apply
        workingDirectory: '$(Build.ArtifactStagingDirectory)/Terraform'
        commandOptions: '-auto-approve'
        environmentServiceNameAzureRM: 'ADO-R2F' #replace this with your own service connection name
        backendServiceArm: 
        backendAzureRmResourceGroupName: '$(terraformstoragerg)'
        backendAzureRmStorageAccountName: '$(terraformstorageaccount)'
        backendAzureRmContainerName: '$(terraformstorageaccountcontainer)'
        backendAzureRmKey: '$(storagekey)'
```

### Run the pipeline

Once the yml file is added, you can replace the variable values with your own storage account information, and the *environmentServiceNameAzureRM* with your service connection name.  After the yml file is edited and saved, you can start to run the pipeline.

1. First stage is to build the artifact, the artifact will be published to pipeline.
2. Second stage is to deploy the Logic App infrastructure.
    -   In the Terraform plan step you can review the output to verify what are the resources going to be added, changed, or destroyed
    -   In the terraform apply step the resources will be created.
    -   After the pipeline run successfully, you can view the resources which just deployed from the Azure portal.