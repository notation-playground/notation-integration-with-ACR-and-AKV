# notation-integration-with-ACR-and-AKV
Integrating Notation and notation-akv-plugin to a project releasing to ACR using Github Actions.

The `test-notation-action.yml` workflow build and release the artifact to ACR, sign it with a key pair from AKV, then verify the signature.

## Github Secrets
In total, two Github Secrects are required as prerequisites:
1. `ACR_PASSWORD`: the passowrd to log into the ACR where your artifact will be released.
2. `AZURE_CREDENTIALS`: the credential to AKV where your key pairs are stored.
    
    ### How to generate `AZURE_CREDENTIALS`:
    ```
    # login using your own account
    az login

    # create a new service principal
    spn=<your_service_principal_name>
    az ad sp create-for-rbac -n $spn --sdk-auth
    ```
    > [!IMPORTANT]
    > 1. Add the JSON output of the above `az ad sp` command to [Github Secret](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Cwindows#add-the-service-principal-as-a-github-secret) with name `AZURE_CREDENTIALS`.
    >
    > 2. Save the `clientId` from the JSON output into an environment varialbe (without double quotes) as it will be needed in the next step:
    >```
    >    clientId=<clientId_from_JSON_output_of_last_step>
    >```

    ### Grant AKV permission to that principal
    ```
    # set policy for your AKV
    akv=<your_akv_name>
    az keyvault set-policy --name $akv --spn $clientId --certificate-permissions get --key-permissions sign --secret-permissions get
    ```


## Github Actions used in the test-notation-action workflow
1. `notaryproject/notation-action/setup@main` to setup Notation. (https://github.com/notaryproject/notation-action/tree/main/setup)
2. `notaryproject/notation-action/sign@main` to setup plugin and perform Sign operation. (https://github.com/notaryproject/notation-action/tree/main/sign)
3. `notaryproject/notation-action/verify@main` to verify the signature generated in step 2. (https://github.com/notaryproject/notation-action/tree/main/verify)

## Trigger the test-notation-action workflow
On push to the repo, `test-notation-action` will be triggered automatically. On success, you should see the artifact pushed to your ACR with a COSE signature attached. 