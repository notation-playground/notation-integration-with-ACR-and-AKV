# Sign an image with Notation in GitHub Actions

This document walks you through how to create a GitHub Actions workflow to achieve the following goals:

1. Build an image and push it to Azure Container Registry (ACR).
2. Sign the image with Notation and Notation AKV plugin with a signing key stored in Azure Key Vault (AKV). The generated signature is automatically pushed to ACR.

## Prerequisites

- You have created a Key Vault in AKV and created a self-signed signing key and certificate. You can follow this [doc](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tutorial-sign-build-push#create-a-self-signed-certificate-azure-cli) to create self-signed key and certificate for testing purposes. 
- You have created a registry in Azure Container Registry.
- You have a GitHub repository to store the sample workflow file and GitHub Secrets.

## Authenticate from Azure to GitHub
There are two ways to connect GitHub Actions to your ACR and AKV, pick one of them based on your needs:
1. [Use the Azure login action with a service principal secret](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Clinux#use-the-azure-login-action-with-a-service-principal-secret)
2. [Use the Azure login action with OpenID Connect](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Clinux#use-the-azure-login-action-with-openid-connect)

### Use the Azure login action with a service principal secret

Enter your GitHub repository, create an encrypted secret `AZURE_CREDENTIALS` in the repository to authenticate with ACR and AKV. See [creating encrypted secrets for a repository](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository) for details.

Follow the steps below to get the value of `AZURE_CREDENTIALS`. 

- Execute the following commands to create a new service principal on Azure. 

```
# Log in to Azure CLI
az login

# Create a new service principal with AcrPush role
spn={service_principal_name}
acr_scope=/subscriptions/{subscription_id}/resourceGroups/{resource_group}
az ad sp create-for-rbac -n $spn --scopes $acr_scope --role acrpush --sdk-auth
```

> [!IMPORTANT]
> 1. Copy the entire JSON output from the `az ad sp` command execution result into the **Secret** filed of `AZURE_CREDENTIALS`. See [this doc](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Cwindows#add-the-service-principal-as-a-github-secret) for reference.
>
> 2. Save the `clientId` from the JSON output into an environment variable (without double quotes) as it will be needed in the next step:
>```
>  clientId={clientId_from_JSON_output_of_last_step}
>```

- Grant the AKV access permissions to the service principal that we created in the previous step.

```
# set policy for your AKV
akv={your_akv_name}
az keyvault set-policy --name $akv --spn $clientId --certificate-permissions get --key-permissions sign --secret-permissions get
```

See [az keyvault set-policy](https://learn.microsoft.com/en-us/cli/azure/keyvault?view=azure-cli-latest#az-keyvault-set-policy) for reference.

### Use the Azure login action with OpenID Connect

Create an Access Policy under your AKV following [this doc](https://review.learn.microsoft.com/en-us/azure/key-vault/general/assign-access-policy?branch=pr-en-us-248675&tabs=azure-portal).

> [!IMPORTANT]
> You need to enable the following AKV permissions:
> 1. Key permissions: Sign
> 2. Secret permissions: Get
> 3. Certificate permissions: Get 

On success, the application will be displayed under "App registrations" in Azure portal. From there, follow [this doc](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Clinux#add-federated-credentials) to add a federated credential to your application.

Then follow [this doc](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Clinux#create-github-secrets) to add GitHub Secrets. They are AZURE_CLIENT_ID, AZURE_TENANT_ID, and AZURE_SUBSCRIPTION_ID.

Next, run the following commands to authenticate from ACR
```
# Assign AcrPush role to your application
AZURE_CLIENT_ID={Application (client) ID from last step}
acr_scope=/subscriptions/{subscription_id}/resourceGroups/{resource_group}
az role assignment create --assignee $AZURE_CLIENT_ID --scopes $acr_scope --role acrpush
```

## Create the GitHub Actions workflow

- Create a `.github/workflows` directory in your repository on GitHub if this directory does not already exist.

- In the `.github/workflows` directory, create a file named `{your_workflow}.yml`. 

- You can copy the [signing template workflow](https://github.com/notation-playground/notation-integration-with-ACR-and-AKV/blob/template/sign-template.yml) from the collapsed section below into your own `{your_workflow}.yml` file.

- Update the environmental variables based on your environment by following the comments in the template. Save and commit it to the repository.

<details>

<summary>See the signing workflow template (Click here).</summary>

```yaml
# Build and push an image to ACR, setup notation and sign the image
name: notation-github-actions-sign-template

on:
  push:

env:
  ACR_REGISTRY_NAME: <registry_name_of_your_ACR>          # example: myRegistry.azurecr.io
  ACR_REPO_NAME: <repository_name_of_your_ACR>            # example: myRepo
  AKV_NAME: <your_Azure_Key_Vault_Name>                   # example: myAzureKeyVault
  KEY_ID: <key_id_of_your_private_key_to_sign_from_AKV>   # example: https://mynotationakv.vault.azure.net/keys/notationLeafCert/c585b8ad8fc542b28e41e555d9b3a1fd
  NOTATION_EXPERIMENTAL: 1                                # [Optional] when set, use Referrers API in the workflow (Recommended)

jobs:
  notation-sign:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: prepare
        id: prepare
        # Use `v1` as an example tag, user can pick their own
        run: |
          echo "target_artifact_reference=${{ env.ACR_REGISTRY_NAME }}/${{ env.ACR_REPO_NAME }}:v1" >> "$GITHUB_ENV"
      # Log in to Azure with your service principal
      - name: Azure login
        uses: Azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      # Log in to your ACR registry
      - name: ACR login
        run: |
            az acr login --name ${{ env.ACR_REGISTRY_NAME }}
      # Build and push an image to the registry
      # Use `Dockerfile` as an example to build an image
      - name: Build and push
        id: push
        uses: docker/build-push-action@v4
        with:
          push: true
          tags: ${{ env.target_artifact_reference }}
      # Get the manifest digest of the OCI artifact
      - name: Retrieve digest
        run: |
          echo "target_artifact_reference=${{ env.ACR_REGISTRY_NAME }}/${{ env.ACR_REPO_NAME }}@${{ steps.push.outputs.digest }}" >> "$GITHUB_ENV"
      
      # Install Notation CLI with the default version "1.0.0"
      - name: setup notation
        uses: notaryproject/notation-action/setup@main
      
      # Sign your OCI artifact using private key stored in AKV
      - name: sign OCI artifact using key pair from AKV
        uses: notaryproject/notation-action/sign@main
        with:
          plugin_name: azure-kv
          plugin_url: https://github.com/Azure/notation-azure-kv/releases/download/v1.0.0/notation-azure-kv_1.0.0_linux_amd64.tar.gz
          plugin_checksum: 82d4fee34dfe5e9303e4340d8d7f651da0a89fa8ae03195558f83bb6fa8dd263
          key_id: ${{ env.KEY_ID }}
          target_artifact_reference: ${{ env.target_artifact_reference }}
          signature_format: cose
          plugin_config: |-
            ca_certs=.github/cert-bundle/cert-bundle.crt
            self_signed=false
          # if using self-signed certificate from AKV, then the plugin_config should be:
          # plugin_config: |-
          #   self_signed=true
          allow_referrers_api: 'true'
```

</details>

## Trigger the GitHub Actions workflow

The workflow trigger logic has been set to `on: push` [event](https://docs.github.com/en/actions/using-workflows/triggering-a-workflow#using-events-to-trigger-workflows) in the sample workflow. Committing the workflow file to a branch in your repository triggers the push event and runs your workflow.

On success, you will be able to see the image is built and pushed to your ACR with a COSE format signature attached.

Another use case is to trigger the workflow when a new tag is pushed to the Github repository. See [this doc](https://docs.github.com/en/actions/using-workflows/triggering-a-workflow#example-excluding-branches-and-tags) for details. It is typical to secure a software release process with the trigger event in the GitHub Actions.

## View your GitHub Actions workflow results

Under your GitHub repository name, click **Actions** tab of your GitHub repository to see the workflow logs.
