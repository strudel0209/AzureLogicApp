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