# Verify image with Notation in GitHub Actions

This document walks you through how to create a GitHub Actions workflow to verify a signed image with Notation.

## Prerequisites

- You have signed an image with Notation by following the document [Sign an image with Notation in GitHub Actions](sign-action.md).
- You have a GitHub repository to store public certificate and trust policy for verification. For demonstration convenience, we will use the same GitHub repository that we demonstrated in the signing process.

## Prepare the trust policy and public certificate 

To verify the image signature, we need to create a Notation [trust policy](https://github.com/notaryproject/specifications/blob/main/specs/trust-store-trust-policy.md) file and a Notation trust store to place the public certificate in the specified folder of the GitHub runner machine. Follow the steps below:

- Enter `.github` directory in your GitHub repository, create a Notation trust policy file `trustpolicy/trustpolicy.json`. You can copy the [trust policy template](https://github.com/notation-playground/notation-integration-with-ACR-and-AKV/blob/template/.github/trustpolicy/trustpolicy.json) from the collapsed section below into your `trustpolicy.json` file. 

<details>

<summary>See the trust policy template (Click here).</summary>

```JSON
{
    "version": "1.0",
    "trustPolicies": [
        {
            "name": "remote",
            "registryScopes": [ "your-registry.azurecr.io/integration" ],
            "signatureVerification": {
                "level" : "strict" 
            },
            "trustStores": [ "ca:integration"],
            "trustedIdentities": [
                "*"
            ]
        }
    ]
}
```

</details>

- Update the value of `registryScopes` with your registry `${registry-name}/${namespace}/${repository-name}` in `trustpolicy.json` to specify which registry repository that your signed image stored.

- Enter `.github` directory, create a Notation trust store directory `truststore/x509/ca/integration/` in your GitHub repository. 

- Sign in Azure with Azure CLI. Download your public certificate from AKV using the following commands. You can refer to [the Azure doc](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tutorial-sign-build-push#create-a-self-signed-certificate-azure-cli) to learn more about this step.

```
CERT_ID=$(az keyvault certificate show -n $KEY_NAME --vault-name $AKV_NAME --query 'id' -o tsv)
az keyvault certificate download --file $CERT_PATH --id $CERT_ID --encoding PEM
```

- Push the downloaded public certificate to your Notation trust store directory in the GitHub repository.

### Create the GitHub Actions workflow

- In the `.github/workflows` directory, create a file named `<your_verify_workflow>.yml`. 

- You can copy the [verification template workflow](https://github.com/notation-playground/notation-integration-with-ACR-and-AKV/blob/template/verify-template.yml) from the collapsed section below into your `<your_verify_workflow>.yml` file.

- Update the environmental variables based on your environment by following the comments in the template. Save and commit it to the repository.

<details>

<summary>See the verification workflow template (Click here).</summary>

```yaml
# Set up notation and verify an image stored in ACR
name: notation-github-actions-verify-template

on:
  push:

env:
  ACR_REGISTRY_NAME: <registry_name_of_your_ACR>                       # example: myRegistry.azurecr.io
  ACR_REPO_NAME: <repository_name_of_your_ACR>                         # example: myRepo
  target_artifact_reference: <ACR_REGISTRY_NAME/ACR_REPO_NAME@digest>  # example: myRegistry.azurecr.io/myRepo@sha256:abcdef
  NOTATION_EXPERIMENTAL: 1                                             # [Optional] when set, can use Referrers API in the workflow

jobs:
  notation-verify:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      # Log into your ACR
      - name: docker login
        uses: azure/docker-login@v1
        with:
          login-server: ${{ env.ACR_OF_RELEASE }}
          username: ${{ env.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}

      # Install Notation CLI, the default version is "1.0.0"
      - name: setup notation
        uses: notaryproject/notation-action/setup@main
      
      # Verify the image
      - name: verify image
        uses: notaryproject/notation-action/verify@main
        with:
          target_artifact_reference: ${{ env.target_artifact_reference }}
          trust_policy: .github/trustpolicy/trustpolicy.json
          trust_store: .github/truststore
          allow_referrers_api: 'true'
```

</details>

## Trigger the GitHub Actions workflow

The workflow trigger logic has been set to `on: push` [event](https://docs.github.com/en/actions/using-workflows/triggering-a-workflow#using-events-to-trigger-workflows) in the sample workflow. Committing the workflow file to a branch in your repository triggers the push event and runs your workflow. You can also configure to run a workflow manually by following the [GitHub doc](https://docs.github.com/en/actions/using-workflows/manually-running-a-workflow) based on your need. 

## View your GitHub Actions workflow results

Under your GitHub repository name, click **Actions** tab of your GitHub repository to see the workflow logs.

On success, you will be able to see the signed image is successfully verified using the trust policy. The sha256 digest of the verified image is returned in a successful output message.