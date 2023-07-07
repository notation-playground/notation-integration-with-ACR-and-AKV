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

## Github Actions used in the release-acr workflow
1. `notation-playground/notation-actions/setup@main` to setup Notation. (https://github.com/notation-playground/notation-actions/tree/main/setup)
2. `notation-playground/notation-azure-kv-sign-actions@main` to setup notation-azure-kv plugin and perform Sign operation. (https://github.com/notation-playground/notation-azure-kv-sign-actions)
3. `notation-playground/notation-actions/verify@main` to verify the signature generated in step 2. (https://github.com/notation-playground/notation-actions/tree/main/verify)

## Trigger the release-acr workflow
Create a new tag using `git tag` at local then `git push` the tag to Github repo. `release-acr` will be triggered automatically. On success, you should see the artifact pushed to your ACR with a COSE signature attached. 

A successful build: https://github.com/notation-playground/notation-integration-with-ACR-and-AKV/actions/runs/5483437647/jobs/9989778619