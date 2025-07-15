---
status: proposed
implementation: proposed
last_reviewed: 2025-07-09
notes:
---

# JWT Service-to-Service Authentication

## Summary

This RFC proposes a service-to-service authentication scheme using JSON Web Tokens (JWTs) to enable secure,
sender-constrained API calls in decentralised systems. It introduces a mechanism where services authenticate
requests using cryptographic proofs linked to their Decentralised Identifiers (DIDs) or JSON Web Key (JWK)
thumbprints. This approach aims to simplify key rotation, support delegation, and strengthen API security.

## Problem

Current service authentication models (e.g., X.509 certificates, API keys) have limitations:

- Centralised trust: They rely on centralised Public Key Infrastructure (PKI) hierarchies that don't align with
decentralised identity models.
- Operational friction: Static API keys and certificates make key rotation and delegation cumbersome.
- Replay risk: Traditional Bearer tokens are not sender-constrained, leaving APIs vulnerable to misuse if a token is intercepted.
- Fragmentation across government and public sector services: Different departments adopt inconsistent authentication
patterns, requiring clients to implement bespoke logic and manage separate credentials for each API.
- Client credential management: Managing multiple API-specific keys or key pairs adds overhead and duplication and raises
the likelihood of scaling and management issues for clients.
- Integration barriers: A lack of standardisation drives up integration effort, blocks reuse of shared client libraries,
and increases time-to-value for consuming APIs.

## Background

- JSON Web Token (JWT): A compact, URL-safe token format used for transmitting claims between parties, often signed
with asymmetric keys, see [RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519).
- Decentralised Identifier (DID): A W3C standard for self-sovereign identifiers, resolved via `/.well-known/did.json`
or other DID methods, see [W3C DID Core Specification](https://www.w3.org/TR/did-core/).
- JSON Web Key (JWK): A JSON representation of cryptographic keys, see [RFC 7517](https://datatracker.ietf.org/doc/html/rfc7517)
- DPoP (Demonstrating Proof of Possession): An OAuth 2.0 extension for binding tokens to sender keys, preventing replay,
see [RFC 7800](https://datatracker.ietf.org/doc/html/rfc7800).

## Proposal

This RFC defines a service-to-service authentication mechanism using:  

1. **DIDs as Service Identities**

    - Each service SHOULD have a DID (e.g. `did:web:forms.service.gov.uk`) and expose a DID document with one or more
    verification methods.  

1. **JWTs for Proof-of-Possession**

    - API requests MUST include a JWT, either as:
        - Bearer token - [RFC 6750](https://www.rfc-editor.org/rfc/rfc6750#section-2.1)
        - `DPoP` header - [RFC 9449](https://datatracker.ietf.org/doc/html/rfc9449)
    - The JWT header MUST include:
        - `typ`: A field with the value `dpop+jwt` or `JWT`.
        - `alg`: An identifier for a JWS asymmetric digital signature algorithm.
    - The JWT header MAY include:
        - `jwk`: Represents the public key chosen by the client in JSON Web Key.
        - `kid`: A hint indicating which key the client used to generate the token signature.
    - The JWT payload MUST include at least:
        - `jti`: Unique identifier for the JWT.
        - `iat`: Issued-at timestamp (UNIX time).
    - The JWT payload MUST include either:
        - `htu` and `htm`
            - `htu`: The HTTP target URI of the API call (e.g. `https://api.notifications.service.gov.uk/v2/notifications/email`).
            - `htm`: The HTTP method (e.g. `POST`).  

        or
        - `aud`: The audience as scheme, authority and path (heirarchical part) (e.g. `https://api.notifications.service.gov.uk/v2/notifications/email`)
                or DID urn (e.g. `did:web:api.notifications.service.gov.uk:v2:notifications`).
    - The JWT payload SHOULD include:
        - `iss`: The client as scheme, authority and path (heirarchical part) (e.g. `https://forms.service.gov.uk/api/`)
                or DID urn (e.g. `did:web:forms.service.gov.uk%3A443:api`).
        - `exp`: Expiry time (UNIX time).
        - `cnf`: A JSON object and the members of that object identify the proof-of-possession key.

1. **API Endpoint Validation**

    - On receiving a request, the API server:
        - Checks the client's DID or the calculated JWK Thumbprint
            ([RFC 7638](https://www.rfc-editor.org/rfc/rfc7638.html)) is known and valid.
        - If required, fetching the client's DID document.
        - Uses the `jwk` header or locates the public key matching the `kid` in the JWT header.
        - Verifies the JWT signature using the public key.
        - Either;
            - Validates `htu` matches the expected URI prefix for the API call.
            - Validates `htm` matches the method for the API call.
        - Or;
            - Validates `aud` matches the requested actions.
        - Rejects requests with invalid or expired JWTs.

1. **API Authorisation**

    - Where typically a server would give something (bearer token or client ID and secret) to a client, this
    mechanism means clients should give something (either a DID or JWK Thumbprint) for the server to trust.
    - Servers SHOULD support both bearer and DPoP methods.
    - Servers SHOULD utilise the DID or JWK Thumbprint for authorisation.
    - Servers MAY authorise DIDs by patterns (e.g. `/^(https:\/\/|did:web:)[a-z0-9\-\.]+\.gov\.uk(\/|:|$)/i`).

1. **API Response**

    - On successful validation, APIs MAY return additional information derived from the client's DID document.

1. **Security Considerations**

    - Servers MUST validate `htu` and `htm`, or `aud`, to prevent misuse.
    - Servers MUST compute the JWT signature and compare with given JWT signature.
    - Servers MAY implement a revocation or deny list of `jti`, `kid`, JWK Thumbprints, or DIDs.

1. **Caching**

    - Servers SHOULD support `cache-control` headers to understand when to refetch updated `did.json` files.

1. **Long Lifetime**

    - Long lifetime is anything over 300 seconds (five minutes).
    - If `exp` not set, Servers MUST enforce a short lifetime for JWTs (recommended less than or equal to 300 seconds).
    - Clients SHOULD set a short lifetime `exp` header (recommended less than or equal to 300 seconds).
    - `exp` SHOULD NOT be greater than one year (epoch + 31536000 seconds).

1. **Server-Provisioned**

    - If clients cannot manage their own asymmetric cryptography, servers MAY want to create long lifetime tokens:
        - Servers SHOULD create a key pair per client.

## Examples

### Using bearer token with known DID

HTTP request header (with added newlines)

```jwt
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJFUzI1NiIsImt
pZCI6ImV4YW1wbGUtMSJ9.eyJpc3MiOiJkaWQ6d2ViOmZvcm1zLnNlcnZpY2U
uZ292LnVrIiwianRpIjoiNTA0ZDJhZjItODI2My00ZTUyLWJlNjUtOGNlNzVh
ZmYxNWQyIiwiaHR1IjoiaHR0cHM6Ly9hcGkubm90aWZpY2F0aW9ucy5zZXJ2a
WNlLmdvdi51ay92Mi9ub3RpZmljYXRpb25zL2VtYWlsIiwiaHRtIjoiUE9TVC
IsImlhdCI6MTc1MTg5MjY2NH0.r2y14rJ5SG-SxcAWjMdOJBaRElv5pMSam8Q
hVJ6ItPKjTaR3ykv4HRPY-DnR_PnZGuMLIgPtgRd1CULiSj80lQ
```

Decoded JWT

```json
{
    "typ": "JWT",
    "alg": "ES256",
    "kid": "example-1"
}
.
{
    "iss": "did:web:forms.service.gov.uk",
    "jti": "504d2af2-8263-4e52-be65-8ce75aff15d2",
    "htu": "https://api.notifications.service.gov.uk/v2/notifications/email",
    "htm": "POST",
    "iat": 1751892664
}
```

DID document at `https://forms.service.gov.uk/.well-known/did.json`:

```json
{
    "@context": "https://www.w3.org/ns/did/v1.1",
    "id": "did:web:forms.service.gov.uk",
    "authentication": [{
        "id": "did:web:forms.service.gov.uk#example-1",
        "type": "JsonWebKey",
        "controller": "did:web:forms.service.gov.uk",
        "publicKeyJwk": {
            "kty": "EC",
            "alg": "ES256",
            "kid": "example-1",
            "crv": "P-256",
            "x": "EVs_o5-uQbTjL3chynL4wXgUg2R9q9UU8I5mEovUf84",
            "y": "kGe5DgSIycKp8w9aJmoHhB1sB3QTugfnRWm5nU_TzsY"
        }
    }]
}
```

Sequence

```mermaid
sequenceDiagram
    participant GOV.UK Forms as forms.service.gov.uk
    participant GOV.UK Notifications as api.notifications.service.gov.uk

    GOV.UK Forms->>GOV.UK Forms: Create JWT (with type = JWT and iss = did:web:forms.service.gov.uk)
    GOV.UK Forms->>GOV.UK Forms: Sign JWT using DID private key
    GOV.UK Forms->>GOV.UK Notifications: HTTP request (Authorization: Header <JWT>)
    GOV.UK Notifications->>GOV.UK Notifications: Extracts iss from JWT payload
    GOV.UK Notifications->>GOV.UK Notifications: Checks DID in iss is known and authorised
    GOV.UK Notifications->>GOV.UK Forms: GET /.well-known/did.json
    GOV.UK Forms-->>GOV.UK Notifications: Return DID document (public keys)
    GOV.UK Notifications->>GOV.UK Notifications: Locate key (by kid in JWT header)
    GOV.UK Notifications->>GOV.UK Notifications: Verify JWT signature
    GOV.UK Notifications->>GOV.UK Notifications: Validate htu, htm, iat
    alt Valid JWT
        GOV.UK Notifications-->>GOV.UK Forms: 200 OK (success)
    else Invalid JWT
        GOV.UK Notifications-->>GOV.UK Forms: 401 Unauthorized (failure)
    end
```

### Using DPoP with known JWK Thumbprint

HTTP request header (with added newlines)

```jwt
DPoP: eyJ0eXAiOiJkcG9wK2p3ayIsImFsZyI6IkVTMjU2IiwiandrIjp7Imt
0eSI6IkVDIiwiY3J2IjoiUC0yNTYiLCJ4IjoiRVZzX281LXVRYlRqTDNjaHlu
TDR3WGdVZzJSOXE5VVU4STVtRW92VWY4NCIsInkiOiJrR2U1RGdTSXljS3A4d
zlhSm1vSGhCMXNCM1FUdWdmblJXbTVuVV9UenNZIn19.eyJqdGkiOiJlNWU3O
TliOC1hN2UzLTQzZDYtODM0OS00N2E4ZmMxODlhNDIiLCJodHUiOiJodHRwcz
ovL2FwaS5ub3RpZmljYXRpb25zLnNlcnZpY2UuZ292LnVrL3YyL25vdGlmaWN
hdGlvbnMvZW1haWwiLCJodG0iOiJQT1NUIiwiaWF0IjoxNzUxODk0NzYwfQ.7
6C2u7mImQNnOX-6Ml-7GmreGVE6NrS8YFP28OHps-P5LBrqHLM0ljhudXIqFH
95O8ps-XWCzvP2a7g1BFj1kQ
```

Decoded JWT

```json
{
    "typ": "dpop+jwk",
    "alg": "ES256",
    "jwk": {
        "kty": "EC",
        "crv": "P-256",
        "x": "EVs_o5-uQbTjL3chynL4wXgUg2R9q9UU8I5mEovUf84",
        "y": "kGe5DgSIycKp8w9aJmoHhB1sB3QTugfnRWm5nU_TzsY"
    }
}
.
{
    "jti": "e5e799b8-a7e3-43d6-8349-47a8fc189a42",
    "htu": "https://api.notifications.service.gov.uk/v2/notifications/email",
    "htm": "POST",
    "iat": 1751894760
}
```

Thumbprint

`19J8y7Zprt2-QKLjF2I5pVk0OELX6cY2AfaAv1LC_w8`

Sequence

```mermaid
sequenceDiagram
    participant GOV.UK Forms as forms.service.gov.uk
    participant GOV.UK Notifications as api.notifications.service.gov.uk

    GOV.UK Forms->>GOV.UK Forms: Create JWT (with type = dpop+jwt and jwk in header)
    GOV.UK Forms->>GOV.UK Forms: Sign JWT using private key
    GOV.UK Forms->>GOV.UK Notifications: HTTP request (DPoP: <JWT>)
    GOV.UK Notifications->>GOV.UK Notifications: Extracts jwk public key from JWT payload
    GOV.UK Notifications->>GOV.UK Notifications: Computes the JWK thumprint
    GOV.UK Notifications->>GOV.UK Notifications: Checks JWK thumbprint is known and authorised
    GOV.UK Notifications->>GOV.UK Notifications: Verify JWT signature using jwk public key
    GOV.UK Notifications->>GOV.UK Notifications: Validate htu, htm, iat
    alt Valid JWT
        GOV.UK Notifications-->>GOV.UK Forms: 200 OK (success)
    else Invalid JWT
        GOV.UK Notifications-->>GOV.UK Forms: 401 Unauthorized (failure)
    end
```

## References

- [RFC 6750: The OAuth 2.0 Authorization Framework: Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750)
- [RFC 7517: JSON Web Key (JWK)](https://datatracker.ietf.org/doc/html/rfc7517)
- [RFC 7519: JSON Web Token (JWT)](https://datatracker.ietf.org/doc/html/rfc7519)
- [RFC 7638: JSON Web Key (JWK) Thumbprint](https://datatracker.ietf.org/doc/html/rfc7638)
- [RFC 7800: Proof-of-Possession Key Semantics for JSON Web Tokens (JWTs)](https://datatracker.ietf.org/doc/html/rfc7800)
- [RFC 9449: OAuth 2.0 Demonstrating Proof-of-Possession (DPoP)](https://datatracker.ietf.org/doc/html/rfc9449)
- [W3C DID Core Specification](https://www.w3.org/TR/did-core/)
