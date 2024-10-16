<!-- omit in toc -->
# Use SSL Certificate from Public ACME CA

If you want to use a certificate from public ACME CA such as Let's Encrypt or ZeroSSL instead of Self-Signed certificate, some additional steps are required.

<!-- omit in toc -->
## Table of Contents

- [Concepts](#concepts)
- [Procedure](#procedure)
  - [Deploy cert-manager](#deploy-cert-manager)
  - [Prepare Issuer using Azure DNS with Service Principal](#prepare-issuer-using-azure-dns-with-service-principal)
  - [Prepare Issuer using Cloudflare](#prepare-issuer-using-cloudflare)
  - [Modify configuration files for AWX](#modify-configuration-files-for-awx)

## Concepts

In order to use a valid SSL certificate issued by public ACME CA for Ingress, you we to run some kind of ACME client software.

Traefik, the default Ingress controller for K3s, has a built-in ACME client, but it is a bit complicated to make it work, so in this guide, we will use [cert-manager](https://cert-manager.io/).

To issue a certificate from the ACME CA using [cert-manager](https://cert-manager.io/), we must first create [an Issuer or ClusterIssuer](https://cert-manager.io/docs/concepts/issuer/) resource that contains the account information to be registered with the ACME CA and the information to be used for the HTTP-01 and DNS-01 challenges.

Once the Issuer has been created, simply specify the Issuer in the annotation of the Ingress resource, and cert-manager will do all the necessary work automatically.

Fortunately, AWX Operator has a parameter to add arbitrary annotations to Ingress, so protecting our AWX instance with a certificate from a public ACME CA is a breeze.

In this example, we will use:

- [**DNS-01** challenge](https://cert-manager.io/docs/configuration/acme/dns01/)
- with [**Azure DNS**](https://cert-manager.io/docs/configuration/acme/dns01/azuredns/)
- with [**Service Principal**](https://cert-manager.io/docs/configuration/acme/dns01/azuredns/#service-principal)

This guide does not provide any information how to configure Azure, other DNS services, or how to use HTTP-01 challenge here. Please refer to the document of cert-manager for details.

- [Configure Issuer for DNS-01 challenge](https://cert-manager.io/docs/configuration/acme/dns01/)
- [Configure Issuer for HTTP-01 challenge](https://cert-manager.io/docs/configuration/acme/http01/)

## Procedure

### Deploy cert-manager

Deploy cert-manager first.

<!-- shell: instance: deploy cert manager -->
```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.15.1/cert-manager.yaml
```

Ensure the pods in `cert-manager` namespace are running.

```bash
$ kubectl -n cert-manager get pod
NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-55658cdf68-rfqmf             1/1     Running   0          21h
cert-manager-cainjector-967788869-xnq2n   1/1     Running   0          21h
cert-manager-webhook-6668fbb57d-r9dmj     1/1     Running   0          21h
```

### Prepare Issuer using Azure DNS with Service Principal

To use **DNS-01** challenge with **Azure DNS** with **Service Principal**, the following information is required.

- **Subscription ID**
  - `DNS zones` > Your Zone > `Subscription ID`
- **Name of Resource Group**
  - `DNS zones` > Your Zone > `Resource group`
- **Name of DNS Zone**
  - `DNS zones` > Your Zone
- **Tenant ID**
  - `Microsoft Entra ID` > `Properties` > `Tenant ID`
- **Client ID**
  - `Microsoft Entra ID` > `App registrations` > Your Application > `Application (client) ID`
- **Client Secret**
  - `Microsoft Entra ID` > `App registrations` > Your Application > `Certificates & secrets` > `Client secrets` > `Value`

Then modify required fields in `acme/issuer.yaml`.

```yaml
...
spec:
  acme:
    email: cert@example.com                                          👈👈👈

    server: https://acme-staging-v02.api.letsencrypt.org/directory   👈👈👈

    privateKeySecretRef:
      name: awx-issuer-account-key

    solvers:
      - dns01:
          azureDNS:
            environment: AzurePublicCloud
            subscriptionID: 00000000-0000-0000-0000-000000000000     👈👈👈
            resourceGroupName: example-rg                            👈👈👈
            hostedZoneName: example.com                              👈👈👈
            tenantID: 00000000-0000-0000-0000-000000000000           👈👈👈
            clientID: 00000000-0000-0000-0000-000000000000           👈👈👈
            clientSecretSecretRef:
              name: azuredns-config
              key: client-secret
```

To store Client Secret for the Service Principal to Secret resource in Kubernetes, modify `acme/kustomization.yaml`.

```yaml
...
  - name: azuredns-config
    type: Opaque
    literals:
      - client-secret=0000000000000000000000000000000000   👈👈👈
...
```

Once the file has been modified to suit your environment, deploy the Issuer.

<!-- shell: instance: deploy issuer -->
```bash
kubectl apply -k acme
```

Ensure your Issuer exists in `awx` namespace.

```bash
$ kubectl -n awx get issuer
NAME         READY   AGE
awx-issuer   True    21h
```

### Prepare Issuer using Cloudflare

The other option is to use Cloudflare. You first need to login to your cloudflare account and create an `API-Token` which is better as compared to an `API-Global-Key` from security point of view. 

Tokens can be created at `User Profile > API Tokens > API Tokens`. You can create a `custom` API Token with the following recommended settings:

- Permissions:
  - Zone - DNS - Edit
  - Zone - Zone - Read
- Zone Resources:
  - Include - All Zones

Then modify required fields in `acme/issuer.yaml`.

```yaml
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: awx-issuer
spec:
  acme:
    # The email address to be registered with ACME CA
    email: your-cloudflare-account-email@example.com          👈👈👈

    # The URL of the ACME API endpoint.
    # In Let's Encrypt, this is one of the following:
    #   Production: https://acme-v02.api.letsencrypt.org/directory
    #   Staging   : https://acme-staging-v02.api.letsencrypt.org/directory
    #server: https://acme-staging-v02.api.letsencrypt.org/directory
    server: https://acme-v02.api.letsencrypt.org/directory          👈👈👈

    privateKeySecretRef:
      name: awx-issuer-account-key

    solvers:
    - dns01:
        cloudflare:
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token
```

To store Client Secret for the Service Principal to Secret resource in Kubernetes, modify `acme/kustomization.yaml`.

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: awx

generatorOptions:
  disableNameSuffixHash: true

secretGenerator:
  - name: cloudflare-api-token-secret
    type: Opaque
    literals:
      - api-token=your-cloudflare-api-token          👈👈👈

resources:
  - issuer.yaml
```

Once the file has been modified to suit your environment, deploy the Issuer.

<!-- shell: instance: deploy issuer -->
```bash
kubectl apply -k acme
```

Ensure your Issuer exists in `awx` namespace and is in the READY=True state.

```bash
$ kubectl -n awx get issuer
NAME         READY   AGE
awx-issuer   True    21h
```

### Modify configuration files for AWX

Now that we have an Issuer, the last step is to add annotations to Ingress. A few files under the `base` directory need to be modified.

In `base/awx.yaml`, correct `hostname` to the FQDN which the certificate will be issued, and add `ingress_annotations` parameter to specify which Issuer will be used.

```yaml
spec:
  ...
  ingress_type: ingress
  ingress_hosts:
    - hostname: awx.example.com          👈👈👈
      tls_secret: awx-secret-tls

  ingress_annotations: |                 👈👈👈
    cert-manager.io/issuer: awx-issuer   👈👈👈
```

Finally, comment out or delete all of the `awx-secret-tls` part in `base/kustomization.yaml`, as the actual contents of `awx-secret-tls` are automatically managed by cert-manager and do not need to be specified manually.

```yaml
...
generatorOptions:
  disableNameSuffixHash: true

secretGenerator:
  # - name: awx-secret-tls      👈👈👈
  #   type: kubernetes.io/tls   👈👈👈
  #   files:                    👈👈👈
  #     - tls.crt               👈👈👈
  #     - tls.key               👈👈👈

  - name: awx-postgres-configuration
    type: Opaque
...
```

Now your configuration files to ready to use ACME CA. Go back [`README.md`](https://github.com/kurokobo/awx-on-k3s#prepare-required-files) and proceed the procedure.

Bear in mind that you don't have to create any certificates and keys etc.

Once the AWX instance is up and running, we can access it over HTTPS and we will see that our AWX protected by a valid SSL certificate.
