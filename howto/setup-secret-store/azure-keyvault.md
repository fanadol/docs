# Secret Store for Azure Key Vault

This document shows how to enable Azure Key Vault secret store using [Dapr Secrets Component](../../concepts/components/secrets.md) for Standalone and Kubernetes mode. The Dapr secret store component uses Service Principal using certificate authorization to authenticate Key Vault. 

> **Note:** Managed Identity for Azure Key Vault is not currently supported.

## Contents

  - [Prerequisites](#prerequisites)
  - [Create Azure Key Vault and Service principal](#create-azure-key-vault-and-service-principal)
  - [Use Azure Key Vault secret store in Standalone mode](#use-azure-key-vault-secret-store-in-standalone-mode)
  - [Use Azure Key Vault secret store in Kubernetes mode](#use-azure-key-vault-secret-store-in-kubernetes-mode)
  - [References](#references)

## Prerequisites

* [Azure Subscription](https://azure.microsoft.com/en-us/free/)
* [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)

## Create an Azure Key Vault and a service principal

This creates new service principal and grants it the permission to keyvault.

1. Login to Azure and set the default subscription

```bash
# Log in Azure
az login

# Set your subscription to the default subscription
az account set -s [your subscription id]
```

2. Create an Azure Key Vault in a region

```bash
az keyvault create --location [region] --name [your_keyvault] --resource-group [your resource group]
```

3. Create a service principal

Create a service principal with a new certificate and store the 1-year certificate inside [your keyvault]'s certificate vault.

> **Note** you can skip this step if you want to use an existing service principal for keyvault instead of creating new one

```bash
az ad sp create-for-rbac --name [your_service_principal_name] --create-cert --cert [certificate_name] --keyvault [your_keyvault] --skip-assignment --years 1

{
  "appId": "a4f90000-0000-0000-0000-00000011d000",
  "displayName": "[your_service_principal_name]",
  "name": "http://[your_service_principal_name]",
  "password": null,
  "tenant": "34f90000-0000-0000-0000-00000011d000"
}
```

**Save the both the appId and tenant from the output which will be used in the next step**

3. Get the Object Id for [your_service_principal_name]

```bash
az ad sp show --id [service_principal_app_id]

{
    ...
    "objectId": "[your_service_principal_object_id]",
    "objectType": "ServicePrincipal",
    ...
}
```

4. Grant the service principal the GET permission to your Azure Key Vault

```bash
az keyvault set-policy --name [your_keyvault] --object-id [your_service_principal_object_id] --secret-permissions get
```

Now, your service principal has access to your keyvault,  you are ready to configure the secret store component to use secrets stored in your keyvault to access other compoents securely. 

5. Download PFX cert from your Azure Keyvault

* **Using Azure Portal**
  Go to your keyvault on Portal and download [certificate_name] pfx cert from certificate vault
* **Using Azure CLI**
   For Linux/MacOS
   ```bash
   # Download base64 encoded cert
   az keyvault secret download --vault-name [your_keyvault] --name [certificate_name] --file [certificate_name].txt

   # Decode base64 encoded cert to pfx cert for linux/macos
   base64 --decode [certificate_name].txt > [certificate_name].pfx
   ```

   For Windows, on powershell
   ```powershell
   # Decode base64 encoded cert to pfx cert for linux/macos
   $EncodedText = Get-Content -Path [certificate_name].txt -Raw
   [System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String($EncodedText)) | Set-Content -Path [certificate_name].pfx -Encoding Byte
   ```

## Use Azure Key Vault secret store in Standalone mode

This section walks you through how to enable an Azure Key Vault secret store to store a password to securely access a Redis state store in Standalone mode.

1. Create a components directory in your application root

All Dapr components are stored in a directory called 'components' below at application root. Create this directory.

```bash
mkdir components
```

2. Copy downloaded PFX cert from your Azure Keyvault Certificate Vault into `./components` or a secure location in your local disk

3. Create a file called azurekeyvault.yaml in the components directory

Now create an Dapr azurekeyvault component. Create a file called azurekeyvault.yaml in the components directory with the content below

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: azurekeyvault
spec:
  type: secretstores.azure.keyvault
  metadata:
  - name: vaultName
    value: [your_keyvault_name]
  - name: spnTenantId
    value: "[your_service_principal_tenant_id]"
  - name: spnClientId
    value: "[your_service_principal_app_id]"
  - name: spnCertificateFile
    value : "[pfx_certificate_file_local_path]"
```

4. Store redisPassword secret to keyvault

```bash
az keyvault secret set --name redisPassword --vault-name [your_keyvault_name] --value "your redis passphrase"
```

5. Create redis.yaml in the components directory with the content below

Create a statestore component file. This Redis component yaml shows how to use the `redisPassword` secret stored in an Azure Key Vault called azurekeyvault as a Redis connection password.

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  metadata:
  - name: redisHost
    value: "[redis]:6379"
  - name: redisPassword
    secretKeyRef:
      name: redisPassword
auth:
    secretStore: azurekeyvault
```

6. Run your app

You can check that `secretstores.azure.keyvault` component is loaded and redis server connects successfully by looking at the log output when using the dapr `run` command

Here is the log when you run [HelloWorld sample](https://github.com/dapr/samples/tree/master/1.hello-world) with Azure Key Vault secret store.

```bash
$ dapr run --app-id mynode --app-port 3000 --port 3500 node app.js

ℹ️  Starting Dapr with id mynode on port 3500
✅  You're up and running! Both Dapr and your app logs will appear here.

...
== DAPR == time="2019-09-25T17:57:37-07:00" level=info msg="loaded component azurekeyvault (secretstores.azure.keyvault)"
== APP == Node App listening on port 3000!
== DAPR == time="2019-09-25T17:57:38-07:00" level=info msg="loaded component statestore (state.redis)"
== DAPR == time="2019-09-25T17:57:38-07:00" level=info msg="loaded component messagebus (pubsub.redis)"
...
== DAPR == 2019/09/25 17:57:38 redis: connecting to [redis]:6379
== DAPR == 2019/09/25 17:57:38 redis: connected to [redis]:6379 (localAddr: x.x.x.x:62137, remAddr: x.x.x.x:6379)
...
```

## Use Azure Key Vault secret store in Kubernetes mode

In Kubernetes mode, you store the certificate for the service principal into the Kubernetes Secret Store and then enable Azure Key Vault secret store with this certificate in Kubernetes secretstore.

1. Create a kubernetes secret using the following command

* **[pfx_certificate_file_local_path]** is the path of PFX cert file you downloaded from [Create Azure Key Vault and Service principal](#create-azure-key-vault-and-service-principal)

* **[your_k8s_spn_secret_name]** is secret name in Kubernetes secret store

```bash
kubectl create secret generic [your_k8s_spn_secret_name] --from-file=[pfx_certificate_file_local_path]
```

2. Create azurekeyvault.yaml component file

The component yaml refers to the Kubernetes secretstore using `auth` property and  `secretKeyRef` refers to the certificate stored in Kubernetes secret store.

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: azurekeyvault
spec:
  type: secretstores.azure.keyvault
  metadata:
  - name: vaultName
    value: [your_keyvault_name]
  - name: spnTenantId
    value: "[your_service_principal_tenant_id]"
  - name: spnClientId
    value: "[your_service_principal_app_id]"
  - name: spnCertificate
    secretKeyRef:
      name: [your_k8s_spn_secret_name]
      key: [pfx_certficiate_file_local_name]
auth:
    secretStore: kubernetes
```

3. Apply azurekeyvault.yaml component

```bash
kubectl apply -f azurekeyvault.yaml
```

4. Store the redisPassword as a secret into your keyvault
Now store the redisPassword as a secret into your keyvault

```bash
az keyvault secret set --name redisPassword --vault-name [your_keyvault_name] --value "your redis passphrase"
```

5. Create redis.yaml state store component

This redis state store component refers to `azurekeyvault` component as a secretstore and uses the secret for `redisPassword` stored in Azure Key Vault.

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  metadata:
  - name: redisHost
    value: "[redis_url]:6379"
  - name: redisPassword
    secretKeyRef:
      name: redisPassword
auth:
    secretStore: azurekeyvault
```

6. Apply redis statestore component

  ```bash
  kubectl apply -f redis.yaml
  ```

7. Deploy your app to Kubernetes

Make sure that `secretstores.azure.keyvault` is loaded successfully in `daprd` sidecar log

Here is the nodeapp log of [HelloWorld Kubernetes sample](https://github.com/dapr/samples/tree/master/2.hello-kubernetes). Note: use the nodeapp name for your deployed container instance. 

```bash
$ kubectl logs nodeapp-f7b7576f4-4pjrj daprd

time="2019-09-26T20:34:23Z" level=info msg="starting Dapr Runtime -- version 0.4.0-alpha.4 -- commit 876474b-dirty"
time="2019-09-26T20:34:23Z" level=info msg="log level set to: info"
time="2019-09-26T20:34:23Z" level=info msg="kubernetes mode configured"
time="2019-09-26T20:34:23Z" level=info msg="dapr id: nodeapp"
time="2019-09-26T20:34:24Z" level=info msg="loaded component azurekeyvault (secretstores.azure.keyvault)"
time="2019-09-26T20:34:25Z" level=info msg="loaded component statestore (state.redis)"
...
2019/09/26 20:34:25 redis: connecting to redis-master:6379
2019/09/26 20:34:25 redis: connected to redis-master:6379 (localAddr: 10.244.3.67:42686, remAddr: 10.0.1.26:6379)
...
```

## References

* [Azure CLI Keyvault CLI](https://docs.microsoft.com/en-us/cli/azure/keyvault?view=azure-cli-latest#az-keyvault-create)
* [Create an Azure service principal with Azure CLI](https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?view=azure-cli-latest)
* [Secrets Component](../../concepts/components/secrets.md)
