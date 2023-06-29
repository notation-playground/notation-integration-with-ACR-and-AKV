# notation-integration
PoC of integrating Notation and notation-akv-plugin to a project releasing to ACR using Github Actions. <br>
The `release-acr.yml` workflow builds and releases an artifact to ACR, then notation sign it with a key pair from AKV.

## Workflow
1. Create or import a key pair in AKV.
2. Add the ACR registry (where your artifact will be released) password to Github secret with name `ACR_PASSWORD`.
3. Setup AKV:
    ```
    az ad sp create-for-rbac --name {myApp} --role contributor --scopes /subscriptions/{subscription-id}/resourceGroups/{MyResourceGroup} --sdk-auth
    ```
4. Add the JSON output of step 3 to Github Secret: https://learn.microsoft.com/en-us/azure/developer/github/github-key-vault#create-a-github-secret with name `AZURE_CREDENTIALS` <br>
5. Create a new tag using `git tag` at local and push to your repo. The build `release-acr` will be triggered automatically. On success, you should see the artifact in your ACR with a COSE signature attached.