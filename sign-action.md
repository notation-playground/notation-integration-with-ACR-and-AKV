# Sign an OCI artifact with Notation in GitHub Actions

This document walks you through how to create a GitHub Actions workflow to achieve following goals:

1. Generate an OCI artifact and push it to Azure Container Registry
2. Sign the OCI artifact with Notation and Notation AKV plugin. The generated signature is automatically pushed to Azure Container Registry

## Prerequisites

- You have created a Key Vault in Azure Key Vault and created a self-signed signing key and certificate. You can follow this [doc](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tutorial-sign-build-push#create-a-self-signed-certificate-azure-cli) to create self-signed key and certificate for testing purposes
- You have created a registry in Azure Container Registry
- You have a GitHub repository to store the sample workflow and GitHub Secret

## Create GitHub Secret to store Azure credentials

Enter your GitHub repository, create two encrypted secret two GitHub Secret `ACR_PASSWORD` and `AZURE_CREDENTIALS` to store credentials for authenticating with ACR and AKV:

- `ACR_PASSWORD`: the password to log in to the ACR where your artifact will be released
- `AZURE_CREDENTIALS`: the credential to AKV where your key pair is stored.

See [GitHub Docs](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository) for details.

Follow the steps below to get the value of `AZURE_CREDENTIALS` that we need to fill out in the GitHub Secret.

- Execute the following command to create the service principal on Azure and generate Azure credentials. 

```
# login using your own account
az login

# Create a service principal
spn=notationtest
az ad sp create-for-rbac -n $spn --sdk-auth
```

> [!IMPORTANT]
> 1. Add the JSON output from the `az ad sp` command execution result to [Github Secret](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Cwindows#add-the-service-principal-as-a-github-secret) `AZURE_CREDENTIALS` that we created in the previous step.
>
> 2. Save the `clientId` from the JSON output into an environment variable (without double quotes) as it will be needed in the next step:
>```
>  clientId=<clientId_from_JSON_output_of_last_step>
>```

- Grant AKV access permissions to the service principal that we created in the previous step.

```
# set policy for your AKV
akv=<your_akv_name>
az keyvault set-policy --name $akv --spn $clientId --certificate-permissions get --key-permissions sign --secret-permissions get
```

See [az keyvault set-policy](https://learn.microsoft.com/en-us/cli/azure/keyvault?view=azure-cli-latest#az-keyvault-set-policy) for details.

### Edit the sample workflow

- Create `<your_gitrepo>/.github/workflows/<your_workflow>.yml` to run and test the CI/CD pipeline. You can copy the [signing template workflow](https://github.com/notation-playground/notation-integration-with-ACR-and-AKV/blob/template/sign-template.yml) to your own `<your_workflow>.yml` file. 

- Update the environmental variables based on your environment by following the comments in the template. Save it and commit it to the repository.

<details>

<summary>See the signing workflow template (Click here)</summary>

```yaml
# build and push an OCI artifact to ACR, setup notation and sign the artifact
name: notation-github-actions-sign-template

on:
  push:

env:
  ACR_REGISTRY_NAME: <registry_name_of_your_ACR>        # example: myRegistry.azurecr.io
  ACR_REPO_NAME: <repository_name_of_your_ACR>          # example: myRepo
  ACR_USERNAME: <user_name_of_your_ACR>                 # example: myRegistry
  AKV_NAME: <your_Azure_Key_Vault_Name>                 # example: myAzureKeyVault
  KEY_ID: <key_id_of_your_private_key_to_sign_in_AKV>   # example: https://mynotationakv.vault.azure.net/keys/notationLeafCert/c585b8ad8fc542b28e41e555d9b3a1fd
  NOTATION_EXPERIMENTAL: 1                              # [Optional] when set, can use Referrers API in the workflow

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
        # using `v1` as an example tag, user can pick their own
        run: |
          echo "target_artifact_reference=${{ env.ACR_REGISTRY_NAME }}/${{ env.ACR_REPO_NAME }}:v1" >> "$GITHUB_ENV"
      # Log into your ACR
      - name: docker login
        uses: azure/docker-login@v1
        with:
          login-server: ${{ env.ACR_REGISTRY_NAME }}
          username: ${{ env.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}
      # Build and Push an OCI artifact to ACR
      # Using `Dockerfile` as an example to build an OCI artifact
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
      # Log into Azure in order to access AKV
      - name: Azure login
        uses: Azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
          allow-no-subscriptions: true
      
      # Install Notation CLI, the default version is "1.0.0"
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

## Trigger the workflow

The workflow trigger logic has been set to `on: push` [event](https://docs.github.com/en/actions/using-workflows/triggering-a-workflow#using-events-to-trigger-workflows) so the workflow will run when a commit push is made to any branch in the workflow's repository.

When a new git commit is pushed to the registry, Notation workflow will build and sign every build and push the signed image with its associated signature to the registry. On success, you will be able to see the image pushed to your ACR with a COSE format signature attached.

Another use case is to trigger the workflow when a new tag is pushed to the Github repo. See [this doc](https://docs.github.com/en/actions/using-workflows/triggering-a-workflow#example-excluding-branches-and-tags) for details. This use case is typical in the software release process.

## Check the GitHub Actions workflow status

See the workflow logs from the GitHub Actions in your own repository.
