---
description: Enabling authentication and authorization in internal web applications.
---

# Azure AD

!!! warning
    This feature is only available in [team namespaces](../../clusters/team-namespaces.md)

## Abstract

!!! abstract
    The NAIS platform provides support for simple, declarative provisioning of an Azure AD client configured with sensible defaults.

    An Azure AD client allows your application to leverage Azure AD for authentication and authorization.

    The most common cases include:

    * User \(employees only\) sign-in with SSO, using [OpenID Connect with Authorization Code flow](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-auth-code-flow)
    * Request chains involving an end-user whose identity and permissions should be propagated through each service/web API, using the [OAuth 2.0 On-Behalf-Of flow](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-on-behalf-of-flow)
    * Daemon / server-side applications for server-to-server interactions without a user, using the [OAuth 2.0 client credentials flow](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow)

!!! info
    **See the** [**NAV Security Guide**](https://security.labs.nais.io/) **for NAV-specific usage of this client.**

## Concepts

The following describes a few core concepts in Azure AD referred to throughout this documentation.

### Tenants

A tenant represents an organization in Azure AD. Each tenant will have their own set of applications, users and groups. In order to log in to a tenant, you must use an account specific to that tenant.

NAV has two tenants in Azure AD:

- `nav.no` - available in all clusters, default tenant for production clusters
- `trygdeetaten.no` - only available in `dev-*`-clusters, default tenant for development clusters

!!! warning
    If your use case requires you to use `nav.no` in the `dev-*`-clusters, then you must [explicitly configure this](azure-ad.md#tenants_1).
    Note that you _cannot_ interact with clients or applications across different tenants.

The same application in different clusters will result in unique Azure AD clients, with each having their own client IDs and access policies. For instance, the following applications in the same `nav.no` tenant will result in separate, unique clients in Azure AD:

* `app-a` in `dev-gcp` 
* `app-a` in `prod-gcp`

### Naming format

An Azure AD client has an associated name within a tenant. NAIS uses this name for lookups and identification.

All clients provisioned through NAIS will be registered in Azure AD using the following naming scheme:

```text
<cluster>:<namespace>:<app-name>
```

???+ example
    ```text
    dev-gcp:aura:nais-testapp
    ```

Equivalently, the identifier used to refer to the application almost follows the same format.

The only notable difference is that `:` replaced by `.`

```text
api://<cluster>.<namespace>.<app-name>
```

!!! tip "Scopes in token requests"
    The above means that instead of using the API provider's client ID:

    ```text
    api://e89006c5-7193-4ca3-8e26-d0990d9d981f/.default
    ```

    you can do this:

    ```text
    api://dev-gcp.aura.nais-testapp/.default
    ```

### Client ID

An Azure AD client has its own ID that uniquely identifies the client within a tenant, and is used in authentication requests to Azure AD.

Your application's Azure AD client ID is available at multiple locations:

1. The environment variable `AZURE_APP_CLIENT_ID`, available inside your application at runtime
2. In the Kubernetes resource - `kubectl get azureapp <app-name>`
3. The [Azure Portal](https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/RegisteredApps). You may have to click on `All applications` if it does not show up in `Owned applications`. Search using the naming scheme mentioned earlier: `<cluster>:<namespace>:<app>`.

## Configuration

### Getting started

=== "nais.yaml"
    ```yaml
    spec:
      azure:
        application:
          enabled: true
            
          # optional, enum of {trygdeetaten.no, nav.no}
          # defaults to trygdeetaten.no in dev-* clusters, nav.no in production
          tenant: nav.no

          # optional, generated defaults shown
          replyURLs: 
            - "https://my-app.dev.nav.no/oauth2/callback"

          # optional
          claims:
            extra:
              - "NAVident"
            groups:
              - id: "<object ID of Azure AD group>"

      # optional, only relevant if your application should receive requests from consumers
      accessPolicy:
        inbound:
          rules:
            - application: app-a
              namespace: othernamespace
              cluster: dev-fss
            - application: app-b

      # required for on-premises only
      webproxy: true 
    ```

### Spec

See the [NAIS manifest](../../nais-application/nais.yaml/reference.md#specazureapplication).

### Accessing external hosts

Azure AD is a third-party service outside of our clusters, which is not reachable by default like most third-party services.

#### Google Cloud Platform \(GCP\)

The following [outbound external hosts](../../nais-application/access-policy.md#external-services) are automatically added when enabling this feature:

* `login.microsoftonline.com`
* `graph.microsoft.com`

You do not need to specify these explicitly.

#### On-premises

You must enable and use [`webproxy`](../../nais-application/nais.yaml/reference.md#specwebproxy) for external communication.

### Reply URLs

> A redirect URI, or reply URL, is the location that the authorization server will send the user to once the app has been successfully authorized, and granted an authorization code or access token. The code or token is contained in the redirect URI or reply token so it's important that you register the correct location as part of the app registration process.
>
> -- Microsoft's documentation on [reply URLs](https://docs.microsoft.com/en-us/azure/active-directory/develop/reply-url)

!!! tip
    Note that `spec.azure.application.replyURLs[]` can be omitted if `spec.ingresses` are specified. See [ingresses](azure-ad.md#ingresses).

You may set reply URLs manually by specifying `spec.azure.application.replyURLs[]`. Doing so will **replace** all of the [auto-generated reply URLs](azure-ad.md#ingresses).

!!! danger
    If you do override the reply URLs, make sure that you specify **all** the URLs that should be registered for the Azure AD client.

    **Ensure that these URLs conform to the restrictions and limitations of** [**reply URLs**](https://docs.microsoft.com/en-us/azure/active-directory/develop/reply-url) **as specified by Microsoft.**

#### Ingresses

If you have _not_ specified any reply URLs, we will automatically generate a reply URL for each ingress specified using this formula:

```text
spec.ingresses[n] + "/oauth2/callback"
```

???+ example
    In other words, this:

    ```yaml
    spec:
      azure:
        application:
          enabled: true
      ingresses:
        - "https://my.application.dev.nav.no"
        - "https://my.application.dev.nav.no/subpath"
    ```

    will generate a spec equivalent to this:

    ```yaml
    spec:
      ingresses:
        - "https://my.application.dev.nav.no"
        - "https://my.application.dev.nav.no/subpath"
      azure:
        application:
          enabled: true
          replyURLs:
            - "https://my.application.dev.nav.no/oauth2/callback"
            - "https://my.application.dev.nav.no/subpath/oauth2/callback"
    ```

### Tenants

To explicitly target a specific [tenant](azure-ad.md#tenants), add a `spec.azure.application.tenant` to your `nais.yaml`:

```yaml
spec:
  azure:
    application:
      enabled: true
      
      # enum of {trygdeetaten.no, nav.no}
      tenant: trygdeetaten.no 
```

### Pre-authorization

For proper scoping of tokens when performing calls between clients, one must either:

1. Request a token for a specific client ID using the client using the [OAuth 2.0 client credentials flow](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow) \(service-to-service calls\).
2. Exchange a token containing an end-user context using the [OAuth 2.0 On-Behalf-Of flow](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-on-behalf-of-flow) \(service-to-service calls on behalf of an end-user\).

Azure AD will enforce authorization for both flows. In other words, you **must** pre-authorize any consumer clients for your application.

Clients that should receive and validate access tokens from other clients should [pre-authorize](azure-ad.md#pre-authorization) said clients.

These are declared by specifying [`spec.accessPolicy.inbound.rules[]`](../../nais-application/nais.yaml/reference.md#specaccesspolicy):

!!! danger
    Any client referred to **must** already exist in Azure AD in order to be assigned the access policy permissions.

    **Clients that do** _**not**_ **exist in Azure AD will be skipped. Assignment will not be retried until next deploy.**

```yaml
spec:
  accessPolicy:
    inbound:
      rules:
        - application: app-a

        - application: app-b
          namespace: other-namespace

        - application: app-c
          namespace: other-namespace
          cluster: other-cluster
```

The above configuration will pre-authorize the Azure AD clients belonging to:

* application `app-a` running in the **same namespace** and **same cluster** as your application 
* application `app-b` running in the namespace `other-namespace` in the **same cluster**
* application `app-c` running in the namespace `other-namespace` in the cluster `other-cluster`

### Groups

By default, all users within the tenant is allowed to log in to your application. 

For some use cases, it is desirable to restrict access to smaller groups of users.

This can be done by explicitly declaring which groups are allowed to access the application:

```yaml
spec:
  azure:
    application:
      enabled: true
      claims:
        groups:
          - id: "<object ID of group in Azure AD>"
```

Azure AD will now only allow sign-ins and token exchanges with the on-behalf-of flow if a given user is a _direct_ member of at least _one of_ the groups declared. 

This also controls the `groups` claim for a user token, which will only contain groups that are both explicitly assigned to the application _and_ which the user is a direct member of.

## Usage

!!! info
    **See the** [**NAV Security Guide**](https://security.labs.nais.io/) **for NAV-specific usage.**


### Runtime Variables & Credentials

The following environment variables and files \(under the directory `/var/run/secrets/nais.io/azure`\) are available at runtime:

???+ example "`AZURE_APP_CLIENT_ID`"

    Azure AD client ID. Unique ID for the application in Azure AD
    
    Example value: `e89006c5-7193-4ca3-8e26-d0990d9d981f`

???+ example "`AZURE_APP_CLIENT_SECRET`"

    Azure AD client secret, i.e. password for [authenticating the application to Azure AD](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow#first-case-access-token-request-with-a-shared-secret) 

    Example value: `b5S0Bgg1OF17Ptpy4_uvUg-m.I~KU_.5RR`

???+ example "`AZURE_APP_JWKS`"

    A JWK Set as defined in [RFC7517 section 5](https://tools.ietf.org/html/rfc7517#section-5). This will always contain a single key, i.e. `AZURE_APP_JWK` - the newest key registered.

    Example value: 

    ```javascript
    {
      "keys": [
        {
            "use": "sig",
            "kty": "RSA",
            "kid": "jXDxKRE6a4jogcc4HgkDq3uVgQ0",
            "n": "xQ3chFsz...",
            "e": "AQAB",
            "d": "C0BVXQFQ...",
            "p": "9TGEF_Vk...",
            "q": "zb0yTkgqO...",
            "dp": "7YcKcCtJ...",
            "dq": "sXxLHp9A...",
            "qi": "QCW5VQjO...",
            "x5c": [
              "MIID8jCC..."
            ],
            "x5t": "jXDxKRE6a4jogcc4HgkDq3uVgQ0",
            "x5t#S256": "AH2gbUvjZYmSQXZ6-YIRxM2YYrLiZYW8NywowyGcxp0"
        }
      ]
    }
    ``` 

???+ example "`AZURE_APP_JWK`"

    Private JWK as defined in [RFC7517](https://tools.ietf.org/html/rfc7517), i.e. a JWK with the private RSA key for creating signed JWTs when [authenticating to Azure AD with a certificate](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow#second-case-access-token-request-with-a-certificate).

    Example value: 

    ```javascript
    {
      "use": "sig",
      "kty": "RSA",
      "kid": "jXDxKRE6a4jogcc4HgkDq3uVgQ0",
      "n": "xQ3chFsz...",
      "e": "AQAB",
      "d": "C0BVXQFQ...",
      "p": "9TGEF_Vk...",
      "q": "zb0yTkgqO...",
      "dp": "7YcKcCtJ...",
      "dq": "sXxLHp9A...",
      "qi": "QCW5VQjO...",
      "x5c": [
        "MIID8jCC..."
      ],
      "x5t": "jXDxKRE6a4jogcc4HgkDq3uVgQ0",
      "x5t#S256": "AH2gbUvjZYmSQXZ6-YIRxM2YYrLiZYW8NywowyGcxp0"
    }
    ```
   
???+ example "`AZURE_APP_PRE_AUTHORIZED_APPS`"
    
    A JSON string. List of names and client IDs for the valid \(i.e. those that exist in Azure AD\) applications defined in [`spec.accessPolicy.inbound.rules[]`](../../nais-application/nais.yaml/reference.md#specaccesspolicy)
    
    Example value: 

    ```javascript
    [
      {
        "name": "dev-gcp:othernamespace:app-a",
        "clientId": "381ce452-1d49-49df-9e7e-990ef0328d6c"
      },
      {
        "name": "dev-gcp:aura:app-b",
        "clientId": "048eb0e8-e18a-473a-a87d-dfede7c65d84"
      }
    ]
    ```

???+ example "`AZURE_APP_TENANT_ID`"

    The Azure AD tenant ID for which the Azure AD client resides in. 

    Example value: `77678b69-1daf-47b6-9072-771d270ac800` 

???+ example "`AZURE_APP_WELL_KNOWN_URL`"

    The well-known URL to the [metadata discovery document](https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderConfig) for the specific tenant in which the Azure AD client resides in.

    Example value: `https://login.microsoftonline.com/77678b69-1daf-47b6-9072-771d270ac800/v2.0/.well-known/openid-configuration`

???+ example "`AZURE_OPENID_CONFIG_ISSUER`"

    `issuer` from the metadata discovery document.

    Example value: `https://login.microsoftonline.com/77678b69-1daf-47b6-9072-771d270ac800/v2.0`

???+ example "`AZURE_OPENID_CONFIG_JWKS_URI`"

    `jwks_uri` from the metadata discovery document.

    Example value: `https://login.microsoftonline.com/77678b69-1daf-47b6-9072-771d270ac800/discovery/v2.0/keys`

???+ example "`AZURE_OPENID_CONFIG_TOKEN_ENDPOINT`"

    `token_endpoint` from the metadata discovery document.

    Example value: `https://login.microsoftonline.com/77678b69-1daf-47b6-9072-771d270ac800/oauth2/v2.0/token`

## Administration

### Owner Access

Generally, configuration of the Azure AD client should be done through [`nais.yaml`](azure-ad.md#configuration). In most cases you will not need to configure the application through the [Azure Portal](https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/RegisteredApps) at all.

Access is limited in accordance with the principle of least privilege.

Rules:

- If your Azure AD client exists in the `nav.no` tenant, your [team](https://github.com/navikt/teams)'s owners will automatically be given owner access. 
- Otherwise, the application will not be assigned any owners.
- Special cases such as extra permissions are manually handled on a case-by-case basis.

If you are not registered as an owner in your team, you should either have an existing owner promote you or have them perform whatever you need.

## Internals

This section is intended for readers interested in the inner workings of this feature.

Provisioning is handled by [Azurerator](https://github.com/nais/azurerator) - a Kubernetes operator that watches a [custom resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) \(called `AzureAdApplication`\) that we've defined in our clusters.

Azurerator generates a Kubernetes Secret containing the values needed for your application to integrate with Azure AD, e.g. credentials and URLs. This secret will be mounted to the pods of your application during deploy.

Every deploy will trigger rotation of credentials, invalidating any passwords and keys that are not in use. _In use_ in this context refers to all credentials that are currently mounted to an existing pod - regardless of their status \(`Running`, `CrashLoopBackOff`, etc.\). In other words, credential rotation should happen with zero downtime.

More details in the [Azurerator](https://github.com/nais/azurerator) repository.

## Legacy

This section only applies if you have an existing Azure AD client registered in the [IaC repository](https://github.com/navikt/aad-iac).

### Why migrate?

* Declarative provisioning, straight from your application's [`nais.yaml`](../../nais-application/nais.yaml/reference.md#specazureapplication)
* No longer dependent on manual user approvals in multiple IaC repositories
* No longer dependent on Vault
* Credentials are rotated on _every_ deploy, completely transparent to the application. This ensures that credentials are fresh and lessens the impact in the case of exposure.
* The exact same feature is present in the [GCP](../../clusters/gcp.md) clusters, which simplifies [migration](../../clusters/migrating-to-gcp.md).

### Tenants

### Pre-authorization

???+ example "aad-iac -> NAIS"

    #### Referring to a NAIS client from legacy

    Prerequisites:

    * You have a legacy Azure client registered in the [IaC](https://github.com/navikt/aad-iac) repository.
    * You would like to [pre-authorize](azure-ad.md#pre-authorization) an Azure client provisioned through NAIS.

    Steps:

    * Refer to the NAIS client using its _fully qualified name_ \(see [naming format](azure-ad.md#naming-format)\).
    * See [this example](https://github.com/navikt/aad-iac/blob/073664fc5e455c17f1a33ec394c4f07464ae0a2f/prod/hookd.yaml#L4-L5).

???+ example "NAIS -> aad-iac"

    #### Referring to a legacy client from NAIS

    Prerequisites:

    * You have an Azure client provisioned through NAIS.
    * You would like to [pre-authorize](azure-ad.md#pre-authorization) a legacy Azure client registered in the [IaC](https://github.com/navikt/aad-iac) repository.

    Steps:

    * The legacy client **must** follow the expected [naming format](azure-ad.md#naming-format). Follow step 1 and step 2 in the [migration guide](azure-ad.md#migration-guide-step-by-step).
    * Refer to the legacy client [analogously to a NAIS application](azure-ad.md#pre-authorization_1)

### Migration guide - step by step

The following describes the steps needed to migrate an existing legacy client where you wish to keep the existing client ID and configuration.

If keeping the existing client ID and configuration is not important, it should be much easier to just provision new clients instead.

!!! warning
    Be aware of the [differences in tenants](azure-ad.md#tenants) between the [IaC repository](https://github.com/navikt/aad-iac) and NAIS:

    * `nonprod` -> `trygdeetaten.no`
    * `prod` -> `nav.no`

???+ check "Step 1 - Rename your application in the Azure Portal" 
    The `Display name` of the application registered in the [Azure Portal](https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/RegisteredApps) **must** match the [expected format](azure-ad.md#naming-format).

    * Go to the **`Branding`** tab for your client in the [Azure Portal](https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/RegisteredApps).
    * Update the `Name`.

???+ check "Step 2 - Update your application \(and any dependants\) in the IaC repository"

    * Ensure the **`name`** of the client registered in the [IaC repository](https://github.com/navikt/aad-iac) is updated to match the name set in [step 1](azure-ad.md#step-1-rename-your-application-in-the-azure-portal).
    * Ensure that any clients that has a reference to the previous name in their **`preauthorizedapplications`** is also updated. 

???+ check "Step 3 - Deploy your NAIS application with Azure AD provisioning enabled"

    * See [getting started](azure-ad.md#getting-started).

???+ check "Step 4 - Delete your application from the IaC repository"

    * Verify that everything works after the migration
    * Delete the application from the [IaC repository](https://github.com/navikt/aad-iac) in order to maintain a single source of truth

## Permanently deleting a client

!!! warning
Permanent deletes are irreversible. Only do this if you are certain that you wish to completely remove the client from DigDir.

When an `AzureAdApplication` resource is deleted from a Kubernetes cluster, the client is not deleted from DigDir.

!!! info
The `Application` resource owns the `AzureAdApplication` resource, deletion of the former will thus trigger a deletion of the latter.

If the `AzureAdApplication` resource is recreated, the client will thus retain the same client ID.

If you want to provision a new client, you must add the following annotation to the `AzureAdApplication` resource:

```bash
kubectl annotate azureapp <app> azure.nais.io/delete=true
```

When this annotation is in place, deleting the `AzureAdApplication` resource from Kubernetes will trigger removal of the client from DigDir.
