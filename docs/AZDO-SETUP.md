# Multi-Environment Azure DevOps Setup

This document outlines the steps to set up a multi-environment workflow to deploy infrastructure and services to Azure using Azure Pipelines, taking the solution from proof of concept to production-ready.

> [!NOTE]
> Note that additional steps not currently covered in this guide may be required when working with the Zero Trust Architecture Deployment to handle deploying to a network-isolated environment.

# Assumptions:

- This example assumes you have an Azure DevOps Organization and Project already set up.
- This example deploys the infrastructure in the same pipeline as all of the services.
- This example deploys three environments: dev, test, and prod. You may modify the number and names of environments as needed.
- This example uses [`azd pipeline config`](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/configure-devops-pipeline?tabs=azdo) to rapidly set up Azure Pipelines and federated identity configuration for enhanced security.
- All below commands are run as a one-time setup on a local machine by an admin who has access to the Azure DevOps Project and Azure tenant.
- This example does not cover configuring any naming conventions.
- The original remote versions of the [orchestrator](https://github.com/Azure/gpt-rag-orchestrator), [frontend](https://github.com/Azure/gpt-rag-frontend), and [ingestion](https://github.com/Azure/gpt-rag-ingestion) repositories are used; in a real scenario, you would fork these repositories and use your forked versions. This would require updating the repository URLs in the `scripts/fetchComponents.*` files.
- Bicep is the IaC language used in this example.

# Decisions required:

- Service Principals that will be used for each environment
- Decisions on which Azure DevOps Repo, Azure subscription, and Azure location to use

# Prerequisites:

- Assumes that the infrastructure creation already completed using GPT-RAG-V2.
- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows?tabs=azure-cli) with [Azure DevOps extension](https://learn.microsoft.com/en-us/azure/devops/cli/?view=azure-devops)
- [Azure Developer CLI](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/install-azd?tabs=winget-windows%2Cbrew-mac%2Cscript-linux&pivots=os-windows)
- [PowerShell 7](https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell?view=powershell-7.4)
- [Git](https://git-scm.com/downloads)
- Azure DevOps organization
- Bash shell (e.g., Git Bash)
- Personnel with the following access levels:
  - In Azure: Either Owner role or Contributor + User Access Administrator roles within the Azure subscription, which provides the ability to create and assign roles to a Service Principal
  - In Azure DevOps: Ability create and manage [Service Connections](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints?view=azure-devops), contribute to repository, create and manage pipelines, and Administrator access on [Default agent pool](https://learn.microsoft.com/en-us/azure/devops/pipelines/policies/permissions?view=azure-devops#set-agent-pool-security-in-azure-pipelines)



# Steps:

> [!NOTE]
> 1. All commands below are to be run in a AZ CLI / Azure Powershell.
> 2. This guide aims to provide automated/programmatic steps for pipeline setup where possible. Manual setup is also possible, but not covered extensively in this guide. Please read more about manual pipeline setup [here](https://github.com/Azure/azure-dev/blob/main/cli/azd/docs/manual-pipeline-config.md).

## 1. Create environments & Service Principals

### Setup

`cd` to the root of the repo. Before creating environments, you need to define the environment names. Note that these environment names are reused as the Azure DevOps environment names and service connection names later.

```bash
$dev_env='<dev-env-name>' # Example: dev
$test_env='<test-env-name>' # Example: test
$prod_env='<prod-env-name>' # Example: prod
```

Next, define the names of the Service Principals that will be used for each environment. You will need the name in later steps.

```bash
$dev_principal_name='<dev-sp-name>'
$test_principal_name='<test-sp-name>'
$prod_principal_name='<prod-sp-name>'
```

Then, set some additional variables that will be used when setting up the environments, pipelines, and credentials:

```bash
$org='<your-org-name>'
$project='<your-project-name>'

$repo='<your-repo-name>'
$pipeline_name= "Azure Dev Deploy ('$repo')"
$subscription_id=$(az account show --query "id" --output tsv)
$rg_location='<your-resource group-location>'
```
Get service connection 

##### Dev
```bash
$service_connection_id=$(az devops service-endpoint list --query "[?name=='$dev_env'].id" --output tsv)
$service_connection_name=$(az devops service-endpoint list --query "[?name=='$dev_env'].name" --output tsv)
```

##### Test
```bash
$service_connection_id=$(az devops service-endpoint list --query "[?name=='$test_env'].id" --output tsv)
$service_connection_name=$(az devops service-endpoint list --query "[?name=='$test_env'].name" --output tsv)
```

##### Production
```bash
$service_connection_id=$(az devops service-endpoint list --query "[?name=='$prod_env'].id" --output tsv)
$service_connection_name=$(az devops service-endpoint list --query "[?name=='$prod_env'].name" --output tsv)
```


Login to Azure and configure the default organization and project:

```bash
az login
az devops configure --defaults organization=https://dev.azure.com/$org project=$project
```


**Setup:** Set up the pipeline, and service principal:

```bash
az pipelines create --name $pipeline_name --description "Pipeline for project: '$repo'" --repository $repo --branch main --repository-type tfsgit --yml-path /.azdo/pipelines/azure-dev.yml --service-connection SERVICE_CONNECTION --skip-first-run true 
```


```bash
azd env select $dev_env
```

## 2. Set up Azure DevOps Environments

### Environment setup

Run `az devops` CLI commands to create the environments:

```bash
echo "{\"name\": \"$dev_env\"}" > azdoenv.json
az devops invoke --area distributedtask --resource environments --route-parameters project=$project --api-version 7.1 --http-method POST --in-file ./azdoenv.json

echo "{\"name\": \"$test_env\"}" > azdoenv.json
az devops invoke --area distributedtask --resource environments --route-parameters project=$project --api-version 7.1 --http-method POST --in-file ./azdoenv.json

echo "{\"name\": \"$prod_env\"}" > azdoenv.json
az devops invoke --area distributedtask --resource environments --route-parameters project=$project --api-version 7.1 --http-method POST --in-file ./azdoenv.json

rm azdoenv.json # clean up temp file
```

> [!TIP]
> After environments are created, consider setting up deployment protection rules for each environment. See [this article](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/approvals?view=azure-devops&tabs=check-pass) for more.

### Variable setup

Once the pipeline YML file is committed to the repository and the environments are set up, the `AZURE_ENV_NAME` pipeline variable needs to be deleted. This value is passed in at the environment level in the pipeline YML file. If you do not delete this pipeline variable, the pipeline will erroneously deploy to the same environment in every stage.

To do this in the Azure DevOps portal, navigate to the pipeline, edit the pipeline, open the variables menu, and delete the `AZURE_ENV_NAME` pipeline variable.

You may alternately run the below command to delete the variable; ensure you replace the pipeline ID with the correct ID. You can find the pipeline ID by navigating to the pipeline in the Azure DevOps portal and looking at the URL. This value is also printed out after running `azd pipeline config`, in the "Link to view your pipeline status".

```bash
az pipelines variable delete --name 'AZURE_ENV_NAME' --pipeline-id <pipeline-id>

$pipelineInfo=$(az pipelines show --name $pipeline_name)
$Pipelineid=$(echo "$pipelineInfo" | python -c "import sys, json; print(json.load(sys.stdin)['id'])")

az pipelines variable create --name 'AZURE_LOCATION' --value $rg_location --pipeline-id $Pipelineid
az pipelines variable create --name 'AZURE_SERVICE_CONNECTION' --value $service_connection_name --pipeline-id $Pipelineid
az pipelines variable create --name 'AZURE_SUBSCRIPTION_ID' --value $subscription_id --pipeline-id $Pipelineid
```

## 3. Modify the workflow files as needed for deployment

> [!IMPORTANT]
> - The environment names are defined as variables within the below described `azure-dev.yml` file, **which need to be edited to match the environment names you created.** In this example, the environment name is also used as the service connection name. If you used different names for the environment name and service connection name, you will **also need to update the service connection parameter passed in each stage**.
> - The `trigger` in the `azure-dev.yml` file is set to `none` to prevent the pipeline from running automatically. You can change this to `main` or `master` to trigger the pipeline on a push to the main branch.

- The following files in the `.azdo/pipelines` folder are used to deploy the infrastructure and services to Azure:
  - `azure-dev.yml`
    - This is the main file that triggers the deployment workflow. The environment names are passed as inputs to the deploy job.
  - `deploy-template.yml`
    - This is a template file invoked by `azure-dev.yml` that is used to deploy the infrastructure and services to Azure.
