# notation-integration-with-ACR-and-AKV
Integrating Notation and notation-akv-plugin to a project releasing to ACR using Github Actions. <br>
The `release-acr.yml` workflow builds and releases the artifact to ACR, then `notation sign` it with a key pair from AKV.

## Github Secrets
In total, two Github Secrects are required in the whole process:
1. `ACR_PASSWORD`: the passowrd to log into the ACR where your artifact will be released.
2. `AZURE_CREDENTIALS`: the credential to log into AKV where your key pair are stored.
    How to generate `AZURE_CREDENTIALS`:
    ```
    az login
    az ad sp create-for-rbac --name {myApp} --role contributor --scopes /subscriptions/{akv-subscription-id}/resourceGroups/{akv-resource-group} --sdk-auth
    ```
    Add the JSON output of above `az ad sp` command to Github Secret (https://learn.microsoft.com/en-us/azure/developer/github/github-key-vault#create-a-github-secret) with name `AZURE_CREDENTIALS`

## Triggering release-acr workflow
After the Github secrets are set up, create a new tag using `git tag` at local then `git push` to Github repo. `release-acr` will be triggered automatically. On success, you should see the artifact pushed to your ACR with a COSE signature attached. 

The workflow uses `shizhMSFT/setup-notation@main` to setup Notation.

The workflow uses `/notation-azure-kv-sign-actions@main` to setup notation-azure-kv plugin and perform Sign operation.
