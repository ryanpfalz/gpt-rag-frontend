# Multi-Environment GitHub Setup - Frontend

This document outlines the steps to set up a multi-environment workflow to deploy the frontend service to Azure using GitHub Actions, taking the solution from proof of concept to production-ready.

> [!IMPORTANT]
>
> - To follow this guide, the infrastructure creation step must already be completed using the guidance in the [GPT-RAG](https://github.com/Azure/GPT-RAG) repository. This repository does not cover infrastructure creation, but rather the deployment of a service to the infrastructure.
>
> - This guide additionally assumes that Service Principals have been created for each environment using the guidance in the [GPT-RAG](https://github.com/Azure/GPT-RAG) repository.

# Decisions required:

- Whether to use federated identity or client secret for authentication

# Prerequisites:

- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows?tabs=azure-cli)
- [Azure Developer CLI](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/install-azd?tabs=winget-windows%2Cbrew-mac%2Cscript-linux&pivots=os-windows)
- [GitHub CLI](https://cli.github.com/)
- [PowerShell 7](https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell?view=powershell-7.4)
- [Git](https://git-scm.com/downloads)
- Bash shell (e.g., Git Bash)
- GitHub organization with the [ability to provision environments](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment) (e.g., GitHub Enterprise)
- Personnel with the following access levels:
  - In Azure: Either Owner role or Contributor + User Access Administrator roles within the Azure subscription, which provides the ability to create and assign roles to a Service Principal
  - In GitHub: Repository owner access, which provides the ability to create environments and variables/secrets

# Steps:

> [!NOTE]
>
> 1. All commands below are to be run in a Bash shell.

## 1. Initial setup

### Setup

`cd` to the root of the repo. Before creating environments, you need to define the environment names. Note that these environment names are reused as the GitHub environment names later. **These are the same environment names that were used when setting up the infrastructure in the [GPT-RAG](https://github.com/Azure/GPT-RAG) repository.**

```bash
dev_env='<dev-env-name>' # Example: dev
test_env='<test-env-name>' # Example: test
prod_env='<prod-env-name>' # Example: prod
```

Next, define the names of the Service Principals that will be used for each environment. You will need the name in later steps. **These are the same Service Principal names that were used when setting up the infrastructure in the [GPT-RAG](https://github.com/Azure/GPT-RAG) repository - the per-environment service principals are reused here.** Using separate Service Principals for each service is not covered in this guide.

```bash
dev_principal_name='<dev-sp-name>'
test_principal_name='<test-sp-name>'
prod_principal_name='<prod-sp-name>'
```

Login to Azure:

```bash
az login
```

## 2. Set up GitHub Environments

### Environment setup

Set up initial variables with your GitHub organization and repository names (the repository is the service repository, not the [GPT-RAG](https://github.com/Azure/GPT-RAG) repository):

```bash
org='<your-org-name>'
repo='<your-repo-name>'
```

Run GitHub CLI commands to create the environments:

```bash
gh auth login

gh api --method PUT -H "Accept: application/vnd.github+json" repos/$org/$repo/environments/$dev_env
gh api --method PUT -H "Accept: application/vnd.github+json" repos/$org/$repo/environments/$test_env
gh api --method PUT -H "Accept: application/vnd.github+json" repos/$org/$repo/environments/$prod_env
```

### Variables setup

Get the client IDs of the Service Principals. Ensure you previously ran `az login`.

```bash
dev_client_id=$(az ad sp list --display-name $dev_principal_name --query "[].appId" --output tsv)
test_client_id=$(az ad sp list --display-name $test_principal_name --query "[].appId" --output tsv)
prod_client_id=$(az ad sp list --display-name $prod_principal_name --query "[].appId" --output tsv)
```

> [!NOTE]
>
> _Alternative approach to get the client IDs in the above steps:_
> In the event that there are multiple Service Principals containing the same name, the `az ad sp list` command executed above may not pull the correct ID. You may execute an alternate command to manually review the list of Service Principals by name and ID. The command to do this is exemplified below for the dev environment.
>
> ```bash
> az ad sp list --display-name $dev_principal_name --query "[].{DisplayName:displayName, > AppId:appId}" --output table # return results in a table format
> dev_client_id='<guid>' # manually assign the correct client ID
> ```
>
> Also note you may get the client IDs from the Azure Portal.

Initialize additional variables that will be set at the GitHub repository level. Note the `RG_LOCATION` variable needs to be the location of the resource group where the infrastructure was created with the [GPT-RAG](https://github.com/Azure/GPT-RAG) repository.

```bash
TENANT_ID=$(az ad sp show --id $dev_client_id --query "appOwnerOrganizationId" --output tsv)
SUBSCRIPTION_ID=$(az account show --query "id" --output tsv)
RG_LOCATION='<your-resource group-location>'
```

Set these values as variables at the environment level:

```bash
gh variable set AZURE_CLIENT_ID -b $dev_client_id -e $dev_env
gh variable set AZURE_CLIENT_ID -b $test_client_id -e $test_env
gh variable set AZURE_CLIENT_ID -b $prod_client_id -e $prod_env
```

Set these values as variables at the repository level:

```bash
gh variable set AZURE_LOCATION -b $RG_LOCATION
gh variable set AZURE_SUBSCRIPTION_ID -b $SUBSCRIPTION_ID
gh variable set AZURE_TENANT_ID -b $TENANT_ID
```

> [!TIP]
> After environments are created, consider setting up deployment protection rules for each environment. See [this article](https://docs.github.com/en/actions/administering-github-actions/managing-environments-for-deployment#deployment-protection-rules) for more.

> [!NOTE]
> If you want to manage and authenticate with a client secret rather than using federated identity, you would need to create a secret for each Service Principal, store it as an environment secret in GitHub, and modify the workflow to use the secret for authentication. This is not covered in this example. If you choose to use a client secret, you may skip 3.

## 3. Configure Azure Federated credentials to use newly set up GitHub environments

Run the following commands to create a federated credential within the Service Principals you created. These commands will enable authentication between GitHub and Azure for each environment.

```bash
issuer="https://token.actions.githubusercontent.com"
audiences="api://AzureADTokenExchange"

echo '{"name": "'"${org}-${repo}-${dev_env}"'", "issuer": "'"${issuer}"'", "subject": "repo:'"$org"'/'"$repo"':environment:'"$dev_env"'", "description": "'"${dev_env}"' environment", "audiences": ["'"${audiences}"'"]}' > federated_id.json
az ad app federated-credential create --id $dev_client_id --parameters ./federated_id.json

echo '{"name": "'"${org}-${repo}-${test_env}"'", "issuer": "'"${issuer}"'", "subject": "repo:'"$org"'/'"$repo"':environment:'"$test_env"'", "description": "'"${test_env}"' environment", "audiences": ["'"${audiences}"'"]}' > federated_id.json
az ad app federated-credential create --id $test_client_id --parameters ./federated_id.json

echo '{"name": "'"${org}-${repo}-${prod_env}"'", "issuer": "'"${issuer}"'", "subject": "repo:'"$org"'/'"$repo"':environment:'"$prod_env"'", "description": "'"${prod_env}"' environment", "audiences": ["'"${audiences}"'"]}' > federated_id.json
az ad app federated-credential create --id $prod_client_id --parameters ./federated_id.json

rm federated_id.json # clean up temp file
```

> [!NOTE]
> You may delete the existing/unmodified federated credentials created by Azure Developer CLI in the Service Principals.

## 4. Set up self-hosted runner for deploying to network isolated infrastructure (if applicable)

If you opted to provision network isolated infrastructure with the [GPT-RAG](https://github.com/Azure/GPT-RAG) repository, you will need to set up a self-hosted runner to deploy the services.

> [!IMPORTANT]
>
> - When you opt to deploy network isolated infrastructure with [GPT-RAG](https://github.com/Azure/GPT-RAG), a virtual machine is created in the network isolated environment. This guide uses this virtual machine as the self-hosted runner for deploying services, since the VM is already integrated into the virtual network and the majority of tooling for building the services are already installed on it. While self-hosted runners can be set up in other ways - e.g., on Kubernetes - this guide does not cover other setup approaches.
> - This guide assumes the self-hosted runner will be reused by all of the services (i.e., by multiple repositories). [Multiple repositories can share the same runner when set up at the organization level](https://github.blog/enterprise-software/ci-cd/github-actions-community-momentum-enterprise-capabilities-and-developer-improvements/#share-self-hosted-runners-across-an-organization).

### Setting up the VM as a self-hosted runner

1. Log into the virtual machine with the user **gptrag** and authenticate with the password stored in the Key Vault, similar to the figure below:
   <img src="../media/readme-keyvault-login.png" alt="Key Vault Login" width="1024">

> [!TIP]
> If you get Error "You do not have access to List secrets for this resource", you need to be added as a Key Vault Secrets User in the Key Vault that starts with `bastionkv`.

2. Upon login to Windows, install [Powershell](https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell-on-windows?view=powershell-7.4#installing-the-msi-package) on the VM - the other prerequisite tools are already installed on the VM.

3. Follow the instructions in the [GitHub documentation](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/adding-self-hosted-runners#adding-a-self-hosted-runner-to-an-organization) install the self-hosted runner tooling on the VM. Ensure the runner is created at the organization level.

> [!CAUTION]
> When prompted "Would you like to run the runner as service?" select **No**. If you don't, the runner will experience [known `azd` issues](https://github.com/Azure/azure-dev/issues/4282) when running jobs.

## 5. Modify the workflow files as needed for deployment

### Updating environment names

> [!IMPORTANT]
>
> - The environment names in the below described `azure-dev.yml` **need to be edited to match the environment names you created**.
> - The `workflow_dispatch` in the `azure-dev.yml` file is set to trigger on push to a branch `none`. You may modify this to trigger on a specific branch or event.

- The following files in the `.github/workflows` folder are used to deploy the service to Azure:
  - `azure-dev.yml`
    - This is the main file that triggers the deployment workflow. The environment names are passed as inputs to the deploy job.
  - `deploy-template.yml`
    - This is a template file invoked by `azure-dev.yml` that is used to deploy the service to Azure. This file needs to be edited if you are using client secret authentication and/or using a self-hosted runner.

### Updating runner configuration (if using a self-hosted runner)

- If you are using a self-hosted runner, you will need to update the runner configuration. The runner configuration is set in the `runs-on` field in the `.github/workflows/deploy-template.yml` workflow file. The value to set in this field can be found in the runner configuration settings in the GitHub organization.

## Next steps

If you haven't deployed the [Orchestrator](https://github.com/Azure/gpt-rag-orchestrator) or [Ingestion](https://github.com/Azure/gpt-rag-ingestion) services, deploy these next.
