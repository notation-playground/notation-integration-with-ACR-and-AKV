# notation-integration-with-ACR-and-AKV
Integrating Notation and notation-akv-plugin to a project releasing to ACR using Github Actions. <br>
The `release-acr.yml` workflow builds and releases the artifact to ACR, then `notation sign` it with a key pair from AKV.

## Github Secrets
1. Create the ACR registry where your artifact will be released. Save its password to Github secret with name `ACR_PASSWORD`.
2. Log into AKV:
    ```
    az ad sp create-for-rbac --name {myApp} --role contributor --scopes /subscriptions/{subscription-id}/resourceGroups/{MyResourceGroup} --sdk-auth
    ```
    Add the JSON output of above `az` command to Github Secret (https://learn.microsoft.com/en-us/azure/developer/github/github-key-vault#create-a-github-secret) with name `AZURE_CREDENTIALS`

## Triggering release-acr workflow
After the Github secrets are set up, create a new tag using `git tag` at local then `git push` to Github repo. `release-acr` will be triggered automatically. On success, you should see the artifact pushed to your ACR with a COSE signature attached.