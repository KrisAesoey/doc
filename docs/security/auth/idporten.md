---
description: Enabling public-facing authentication using ID-porten.
---

# ID-porten

!!! warning "Status: Opt-In Open Beta"
    This feature is currently only available in the [Google Cloud Platform \(GCP\)](../../clusters/gcp.md) clusters.

## Abstract

!!! abstract
    ID-porten is a common log-in system used for logging into Norwegian public e-services for citizens.

    The NAIS platform provides support for simple, declarative provisioning of an [ID-porten client](https://difi.github.io/felleslosninger/oidc_index.html) with sensible defaults that your application may use to integrate with ID-porten.

    An ID-porten client allows your application to leverage ID-porten for authentication of citizen end-users, providing sign-in capabilities with single sign-on \(SSO\). To achieve this, your application must implement [OpenID Connect with the Authorization Code](https://difi.github.io/felleslosninger/oidc_guide_idporten.html) flow.

    This is also a critical first step in request chains involving an end-user whose identity and permissions should be propagated through each service/web API when accessing services in NAV using the [OAuth 2.0 Token Exchange](https://www.rfc-editor.org/rfc/rfc8693.html) protocol. See the [TokenX documentation](tokenx.md) for details.

!!! info
    **See the** [**NAV Security Guide**](https://security.labs.nais.io/) **for NAV-specific usage of this client.**

    Refer to the [ID-porten Integration guide](https://difi.github.io/felleslosninger/oidc_guide_idporten.html) for further details.

## Configuration

### Getting Started
=== "nais.yaml"
    ```yaml
    spec:
      idporten:
        enabled: true

        # optional, default shown
        clientURI: "https://nav.no"

        # optional, generated default shown
        redirectURI: "https://my.application.dev.nav.no/callback"

        # optional, generated default shown
        frontchannelLogoutURI: "https://my.application.dev.nav.no/logout"

        # optional, defaults shown
        postLogoutRedirectURIs: 
          - "https://nav.no"

        # optional, in seconds - defaults shown (1 hour)
        accessTokenLifetime: 3600

        # optional, in seconds - defaults shown (2 hours)
        sessionLifetime: 7200
    ```

### Spec

See the [NAIS manifest](../../nais-application/nais.yaml/reference.md#specidporten).

### Access Policies

The following [outbound external hosts](../../nais-application/access-policy.md#external-services) are automatically added when enabling this feature:

* `oidc-ver2.difi.no` in development
* `oidc.difi.no` in production

You do not need to specify these explicitly.

### Ingresses

!!! danger
    For security reasons you may only specify **one** ingress when this feature is enabled.

The ingress specified will by default generate a redirect URL using the formula:

```text
spec.ingresses[0] + "/oauth2/callback"
```

In other words, this:

```yaml
spec:
  idporten:
    enabled: true
  ingresses:
    - "https://my.application.dev.nav.no"
```

will generate a spec equivalent to this:

```yaml
spec:
  idporten:
    enabled: true
    redirectURI: "https://my.application.dev.nav.no/oauth2/callback"
  ingresses:
    - "https://my.application.dev.nav.no"
```

### Redirect URI

If you wish to use a different path than the auto-generated one, you may do so by manually specifying `spec.idporten.redirectURI`.

!!! warning
    Specifying `spec.idporten.redirectURI` will replace the auto-generated redirect URI.

The redirect URI must adhere to the following rules:

* It **must** start with `https://`.
* It **must** be a path within your ingress.

For example, if you have specified an ingress at `https://my.application.dev.nav.no`, then:

* ✅ `https://my.application.dev.nav.no/some/path` is valid
* ❌ `http://localhost/some/path` is invalid 

## Usage

!!! info
    **See the** [**NAV Security Guide**](https://security.labs.nais.io/) **for NAV-specific usage.**

### Runtime Variables & Credentials

The following environment variables and files \(under the directory `/var/run/secrets/nais.io/idporten`\) are available at runtime:

??? example "`IDPORTEN_CLIENT_ID`"

    ID-porten client ID. Unique ID for the application in ID-porten.

    Example value: `e89006c5-7193-4ca3-8e26-d0990d9d981f`

??? example "`IDPORTEN_CLIENT_JWK`"

    Private JWK containing the private RSA key for creating signed JWTs when [authenticating to ID-porten with a JWT grant](https://difi.github.io/felleslosninger/oidc_guide_idporten.html#klientautentisering-med-jwt-token).

    ```javascript
    {
      "use": "sig",
      "kty": "RSA",
      "kid": "jXDxKRE6a4jogcc4HgkDq3uVgQ0",
      "alg": "RS256",
      "n": "xQ3chFsz...",
      "e": "AQAB",
      "d": "C0BVXQFQ...",
      "p": "9TGEF_Vk...",
      "q": "zb0yTkgqO...",
      "dp": "7YcKcCtJ...",
      "dq": "sXxLHp9A...",
      "qi": "QCW5VQjO..."
    }
    ```

??? example "`IDPORTEN_REDIRECT_URI`"

    The redirect URI registered for the client at ID-porten. This must be a valid URI for the application where the user is redirected back to after successful authentication and authorization.
    
    Example value: `https://my.application.dev.nav.no/callback` 

??? example "`IDPORTEN_WELL_KNOWN_URL`"

    The well-known URL for the OIDC metadata discovery document for ID-porten.

    Example value: `https://oidc-ver2.difi.no/idporten-oidc-provider/.well-known/openid-configuration`

### Test Users for Logins

ID-porten maintains a public list of test users found [here](https://difi.github.io/felleslosninger/idporten_testbrukere.html).

## Internals

This section is intended for readers interested in the inner workings of this feature.

Provisioning is handled by [Digdirator](https://github.com/nais/digdirator) - a Kubernetes operator that watches a [custom resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) \(called `IDPortenClient`\) that we've defined in our clusters.

Digdirator generates a Kubernetes Secret containing the values needed for your application to integrate with ID-porten, e.g. credentials and URLs. This secret will be mounted to the pods of your application during deploy.

Every deploy will trigger rotation of credentials, invalidating any credentials that are not in use. _In use_ in this context refers to all credentials that are currently mounted to an existing pod - regardless of their status \(`Running`, `CrashLoopBackOff`, etc.\). In other words, credential rotation should happen with zero downtime.

More details in the [Digdirator](https://github.com/nais/digdirator) repository.

