---
status: proposed
implementation: proposed
last_reviewed: 2025-07-07
notes:
---

# JWT Service-to-Service Authentication

## Summary

This RFC proposes a service-to-service authentication scheme using JSON Web Tokens (JWTs). It enables secure,
sender-constrained API calls in decentralised systems, allowing services to authenticate requests using cryptographic
proofs linked to their DID documents or JWK thumbprints.

## Problem

Traditional service authentication models often rely on centralised PKI (e.g. X.509 certificates) or static API keys.
These approaches:  

- Do not align with decentralised identity ecosystems.  
- Are cumbersome for key rotation and delegation.  
- Lack sender-constrained token protection, leaving APIs vulnerable to token replay.  

There is a need for a decentralised, cryptographically verifiable mechanism that allows services to authenticate
API calls while using DID allows for key rotation and delegation.

## Proposal

This RFC defines a service-to-service authentication mechanism using:  

1. **DIDs as Service Identities**
   - Each service SHOULD have a DID (e.g. `did:web:forms.service.gov.uk`) and expose a DID document with one or more
verification methods.  

2. **JWTs for Proof-of-Possession**
   - API requests MUST include a JWT, either as:
     - Bearer token - [RFC 6750: The OAuth 2.0 Authorization Framework: Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750#section-2.1)
     - `DPoP` header - [RFC 9449: OAuth 2.0 Demonstrating Proof-of-Possession (DPoP)](https://datatracker.ietf.org/doc/html/rfc9449)
   - The JWT header MUST include:
     - `typ`: A field with the value `dpop+jwt` or `JWT`.
     - `alg`: An identifier for a JWS asymmetric digital signature algorithm.
   - The JWT header MAY include:
     - `jwk`: Represents the public key chosen by the client in JSON Web Key.
     - `kid`: A hint indicating which key the client used to generate the token signature.
   - The JWT payload MUST include at least:
     - `jti`: Unique identifier for the JWT.
     - `htu`: The HTTP target URI of the API call (e.g. `https://api.notifications.service.gov.uk/v2/notifications/email`).
     - `htm`: The HTTP method (e.g. `POST`).
     - `iat`: Issued-at timestamp (UNIX time).
   - The JWT payload SHOULD include:
     - `iss`: The caller as a `https://` plus authority (e.g. `https://forms.service.gov.uk`) or DID urn (e.g. `did:web:forms.service.gov.uk`).
     - `exp`: Expiry time (UNIX time).
     - `cnf`: A JSON object and the members of that object identify the proof-of-possession key.

3. **API Endpoint Validation**
   - On receiving a request, the API server:
     - Checks the caller’s DID or the calculated JWK thumbprint is known.
     - If required, fetching the caller’s DID document.
     - Uses the `jwk` header or locates the public key matching the `kid` in the JWT header.
     - Verifies the JWT signature using this key.
     - Validates `htu` matches the expected URI prefix for the API call.
     - Validates `htm` matches the method for the API call.
     - Rejects requests with invalid or expired JWTs.

4. **API Response**
   - On successful validation, APIs MAY return additional information derived from the caller’s DID document.

5. **Security Considerations**
   - Servers MUST validate `htu` and `htm` to prevent replay.
   - Servers MUST enforce a short lifetime for JWTs (recommended less than or equal to 300 seconds) if `exp` not set.
   - Servers SHOULD utilise the DID or JWK thumbprint for authorisation.
   - Servers SHOULD support both bearer and DPoP methods.
   - Servers SHOULD support `cache-control` headers to understand when to refetch updated `did.json` files.
   - Callers SHOULD set a short lifetime `exp` header (recommended less than or equal to 300 seconds).
   - Servers MAY authorise DIDs by patterns (e.g. `/^(https:\/\/|did:web:)[a-z0-9\-\.]+\.gov\.uk(\/|:|$)/i`).

## Examples

### Using bearer token with known DID

HTTP request header (with added newlines)

```jwt
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJFUzI1NiIsImt
pZCI6ImV4YW1wbGUtMSJ9.eyJpc3MiOiJkaWQ6d2ViOmZvcm1zLnNlcnZpY2U
uZ292LnVrIiwianRpIjoiNTA0ZDJhZjItODI2My00ZTUyLWJlNjUtOGNlNzVh
ZmYxNWQyIiwiaHR1IjoiaHR0cHM6Ly9hcGkubm90aWZpY2F0aW9ucy5zZXJ2a
WNlLmdvdi51ay92Mi9ub3RpZmljYXRpb25zL2VtYWlsIiwiaHRtIjoiUE9TVC
IsImlhdCI6MTc1MTg5MjY2NH0.DmTZPDHbg1Bmw-XHNrn1J86avNTrYTrdpJW
aj4yuc85eLokYSHN4a2YDipo6XVd-ZoSLPHTpBEx_k_zDQOjfg
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
- [RFC 7638: JSON Web Key (JWK) Thumbprint](https://datatracker.ietf.org/doc/html/rfc7638)
- [RFC 7800: Proof-of-Possession Key Semantics for JSON Web Tokens (JWTs)](https://datatracker.ietf.org/doc/html/rfc7800)
- [RFC 9449: OAuth 2.0 Demonstrating Proof-of-Possession (DPoP)](https://datatracker.ietf.org/doc/html/rfc9449)
- [W3C DID Core Specification](https://www.w3.org/TR/did-core/)
