# Rotate Azure Active Directory Service Principal secret

Simple GitHub Action that generates new secret and deletes expired secrets for a given service principal.

## Recommendation

Use [federated identity](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Clinux#use-the-azure-login-action-with-openid-connect) to connect to Azure (for the azure/login action).

## Prerequisites

<GITHUB_ACTION_AZURE_CLIENT_ID> - is the application(client) id of the service principal (the enterprise application) with a configured federated identity that you use in the azure/login action

<GITHUB_ACTION_AZURE_OBJECT_ID> - is the object id of the service principal (the enterprise application) that you use in the azure/login action. You can query the object id from AAD by using the following command:

`az ad sp list --filter "appId eq '<GITHUB_ACTION_AZURE_CLIENT_ID>'" --query [].id -o tsv`

<SERVICE_PRINCIPAL_FOR_ROTATION_CLIENT_ID> -  is the application(client) id of the service principal that is subject to secret rotation

1. <GITHUB_ACTION_AZURE_OID> needs to be added to the list of owners for the application / service principal subject to secret rotation (by an existing owner). Today, this is not possible through the portal, only via [PowerShell](https://docs.microsoft.com/en-us/powershell/module/azuread/add-azureadapplicationowner?view=azureadps-2.0) or [CLI](https://docs.microsoft.com/en-us/cli/azure/ad/app/owner?view=azure-cli-latest#az_ad_app_owner_add):

    `az ad app owner add --id <SERVICE_PRINCIPAL_FOR_ROTATION_CLIENT_ID> --owner-object-id <GITHUB_ACTION_AZURE_OBJECT_ID>`
2. Assign the Application.ReadWrite.OwnedBy Microsoft Graph API permissions to your <GITHUB_ACTION_AZURE_OBJECT_ID>. Follow [these instructions](https://aztoso.com/security/microsoft-graph-permissions-managed-identity/).


## Inputs

* client-id - The client(application) id of the service principal that is subject to secret rotation.
* secret-validity-in-days - Desired validity, in days, of the new secret. The default is 90 days.

## Outputs

* new-secret - The newly generated secret for the provided service principal

## Example

Using the action:

```yaml
  - name: 'Rotate the secret'
    uses: tosokr/rotate-secret@v1
    id: rotate-secret
    with:
        client-id: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
        secret-validity-in-days: 30
```

Full example, including the login action:

```yaml
name: Example secret action
on:
  workflow_dispatch:

permissions:
  id-token: write
  contents: read
      
jobs: 
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:        
    - uses: actions/checkout@v3
    - name: 'Az CLI login'
      uses: azure/login@v1
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}          
        allow-no-subscriptions: true        
    - name: 'Rotate the secret'
      uses: tosokr/rotate-secret@v1
      id: rotate-secret
      with:
        client-id: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
        secret-validity-in-days: 30      
    - name: Use the value    
      run: |
        echo "${{ steps.rotate-secret.outputs.new-secret }}"
```