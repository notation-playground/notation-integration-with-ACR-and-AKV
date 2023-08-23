# Sign an artifact with Notation in GitHub Actions

This document walks you through how to create a GitHub Actions workflow to sign the latest build of the project with the Notation Azure Key Vault plugin, and push the signed image with its associated signature to Azure Container Registry.

### Prerequisites

- You have created a Key Vault in Azure Key Vault and created a self-signed signing key and certificate. You can follow this [doc](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tutorial-sign-build-push#create-a-self-signed-certificate-azure-cli) to create self-signed key and certificate for testing purposes
- You have created a registry in Azure Container Registry
- You have a GitHub repository to store the sample workflow and GitHub Secret

### Add provider credentials to Github Secrets

Add two credentials to authenticate with ACR and AKV as follows, two Github Secrets are required in the whole process:

- `ACR_PASSWORD`: the password to log in to the ACR where your artifact will be released
- `AZURE_CREDENTIALS`: the credential to AKV where your key pair is stored
    
### Create service principal and generate Azure credential

- Execute the following command to create the service principal and generate Azure credentials. 

```
# login using your own account
az login

# Create a service principal
spn=notationtest
az ad sp create-for-rbac -n $spn --sdk-auth
```

> [!IMPORTANT]
> 1. Add the JSON output of the above `az ad sp` command to [Github Secret](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Cwindows#add-the-service-principal-as-a-github-secret) with name `AZURE_CREDENTIALS`.
>
> 2. Save the `clientId` from the JSON output into an environment variable (without double quotes) as it will be needed in the next step:
>```
>  clientId=<clientId_from_JSON_output_of_last_step>
>```

### Create GitHub Secret to store credentials 

- Create an encrypted secret to store the Azure Credentials in your own GitHub repository. See [GitHub Docs](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository) for details.

- Add the JSON output of the following `az ad sp` command to the value of GitHub Secret. Naming the GitHub Secret as `AZURE_CREDENTIALS`. 

- Similarly, create another encrypted secret to store the registry credential or password in your own GitHub repository. For example, we use `ACR_PASSWORD` to store the password of the ACR registry in the GitHub Repository secret.

### Grant AKV permission to the service principal

Grant AKV permission to the service principal that we created in the previous step.

```
# set policy for your AKV
akv=<your_akv_name>
az keyvault set-policy --name $akv --spn $clientId --certificate-permissions get --key-permissions sign --secret-permissions get
```

See [az keyvault set-policy](https://learn.microsoft.com/en-us/cli/azure/keyvault?view=azure-cli-latest#az-keyvault-set-policy) for details.

### Edit the sample workflow

- Create a `workflows` folder under `.github` and create a `workflow.yml` to run and test the CI/CD pipeline. You can copy the [signing template workflow](https://github.com/notation-playground/notation-integration-with-ACR-and-AKV/blob/template/sign-template.yml) to your own `workflow.yml` file. 

- Update the environmental variables based on your environment by following the comments in the template. Save it and commit it to the repository.

## Trigger the workflow

The workflow trigger logic has been set to `on: push` [event](https://docs.github.com/en/actions/using-workflows/triggering-a-workflow#using-events-to-trigger-workflows) so the workflow will run when a commit push is made to any branch in the workflow's repository.

When a new git commit is pushed to the registry, Notation workflow will build and sign every build and push the signed image with its associated signature to the registry. On success, you will be able to see the image pushed to your ACR with a COSE format signature attached.

Another use case is to trigger the workflow when a new tag is pushed to the Github repo. See [this doc](https://docs.github.com/en/actions/using-workflows/triggering-a-workflow#example-excluding-branches-and-tags) for details. This use case is typical in the software release process.

## Check the GitHub Actions workflow status

See the workflow logs from the GitHub Actions in your own repository.
