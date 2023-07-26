# notation-integration-with-ACR-and-AKV
Integrating Notation and notation-akv-plugin to a project releasing to ACR using Github Actions. <br>
The `test-notation-action.yml` workflow builds and releases the artifact to ACR, then `notation sign` it with a key pair from AKV.

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

## Github Actions used in the test-notation-action workflow
1. `notaryproject/notation-action/setup@main` to setup Notation. (https://github.com/notaryproject/notation-action/tree/main/setup)
2. `notaryproject/notation-action/sign@main` to setup plugin and perform Sign operation. (https://github.com/notaryproject/notation-action/tree/main/sign)
3. `notaryproject/notation-action/verify@main` to verify the signature generated in step 2. (https://github.com/notaryproject/notation-action/tree/main/verify)

## Trigger the test-notation-action workflow
Create a new tag using `git tag` at local then `git push` the tag to Github repo. `test-notation-action` will be triggered automatically. On success, you should see the artifact pushed to your ACR with a COSE signature attached. 

One successful build: https://github.com/notation-playground/notation-integration-with-ACR-and-AKV/actions/runs/5666584706/job/15353616604