# The SPIFFE Workload Authorization Grant Profile

> **Stability: Incubating** - see [STABILITY.md](STABILITY.md).

## Status of this Memo

This document specifies a credential profile standard for the internet community, and requests discussion and suggestions for improvements. Distribution of this document is unlimited.

## Abstract

The [Workload Authorization Grant][1] (WAG) lets a workload obtain OAuth access tokens from a third-party authorization server that has been configured, once, to trust the workload's platform, without a per-workload registration at that authorization server. The grant is a JWT that names one workload and is signed by the platform's Workload Identity Provider.

A SPIFFE deployment already attests workloads and names them with SPIFFE IDs. This profile lets a SPIFFE implementation act as the Workload Identity Provider: it defines how a WAG is issued for a SPIFFE workload, and a gRPC service on the [SPIFFE Workload Endpoint](SPIFFE_Workload_Endpoint.md) from which a workload fetches its WAGs.

## Table of Contents

1\. [Introduction](#1-introduction)  
2\. [Terminology](#2-terminology)  
3\. [The WAG Issued Under This Profile](#3-the-wag-issued-under-this-profile)  
3.1. [Issuer - `iss`](#31-issuer---iss)  
3.2. [Subject - `sub`](#32-subject---sub)  
3.3. [Audience - `aud`](#33-audience---aud)  
3.4. [Scope - `scope`](#34-scope---scope)  
3.5. [Freshness](#35-freshness)  
3.6. [Additional Claims](#36-additional-claims)  
4\. [Signing Keys](#4-signing-keys)  
5\. [Service Definition](#5-service-definition)  
5.1. [Profile Definition](#51-profile-definition)  
5.2. [FetchWAG](#52-fetchwag)  
5.3. [Error Codes](#53-error-codes)  
6\. [Client Behavior](#6-client-behavior)  
7\. [Security Considerations](#7-security-considerations)  
Appendix A. [Mapping to OAuth 2.0 Token Exchange](#appendix-a-mapping-to-oauth-20-token-exchange)  
Appendix B. [Example](#appendix-b-example)  

## 1. Introduction

The WAG specification leaves out of scope how a workload obtains its grant. In a SPIFFE deployment the answer is natural: the workload already calls a local, unauthenticated endpoint whose implementation attests the caller and issues it credentials for its SPIFFE ID. This profile adds one RPC to that endpoint which returns a WAG instead of an SVID.

A WAG is not an SVID. It is not presented to other SPIFFE workloads, it is not validated against a SPIFFE bundle by its recipient, and it is consumed only by an OAuth authorization server. For that reason this profile is not part of the [SPIFFE Workload API](SPIFFE_Workload_API.md). It is an independent gRPC service, versioned on its own, served on the SPIFFE Workload Endpoint as described in [Extensibility and Services Rendered](SPIFFE_Workload_Endpoint.md#7-extensibility-and-services-rendered).

The service follows the shape of [OAuth 2.0 Token Exchange][2]: it reuses the token exchange parameter names, but it is carried over gRPC, the token type is fixed by the RPC, and the subject is never sent in the request. The subject is always the attested caller.

This profile defines no RPC for trust material. Authorization servers obtain the keys that verify a WAG as described in the WAG specification, not from the Workload Endpoint.

## 2. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119][3].

Terms from the WAG specification are used as follows in a SPIFFE deployment:

| WAG term | In this profile |
|----------|-----------------|
| Platform | The SPIFFE deployment of a trust domain. |
| Workload Identity Provider | The trust domain's issuing authority, acting through its Workload Endpoint implementation. |
| Agent | The workload named by one SPIFFE ID. |
| Agent Identifier | The SPIFFE ID. |
| Authorization Server | The OAuth authorization server named in the `audience` of a request. |

## 3. The WAG Issued Under This Profile

A WAG issued under this profile MUST conform to [Section 5][1] of the WAG specification, including its `typ` header value `oauth-wag+jwt`. This section adds SPIFFE-specific requirements. It does not relax any requirement of the WAG specification.

### 3.1. Issuer - `iss`

The `iss` claim is the Workload Identity Provider's issuer identifier: an `https` URL with no query or fragment component, configured by the operator of the trust domain. It is not derived from the trust domain name.

Each trust domain MUST use its own issuer identifier, and an issuer identifier MUST NOT be used by more than one trust domain. This is the dedicated-issuer arrangement the WAG specification recommends, and it lets an authorization server tell trust domains apart by `iss` alone.

### 3.2. Subject - `sub`

The `sub` claim MUST be set to the SPIFFE ID of the workload to which the WAG is issued. It MUST be the SPIFFE ID of an identity the attested caller is entitled to (see [Identifying the Caller](SPIFFE_Workload_API.md#41-identifying-the-caller)), and MUST NOT be taken from any field of the request other than as a selection among those identities.

The WAG specification requires that an Agent Identifier never be reassigned to a different Agent. A SPIFFE ID names a workload rather than an instance of it, and many instances may share it, so a WAG identifies the logical workload, not the instance that fetched it. Operators MUST NOT reassign a SPIFFE ID that has been used as the subject of a WAG to an unrelated workload, because authorization servers may hold records and permissions keyed on it.

### 3.3. Audience - `aud`

The `aud` claim MUST contain exactly one value, the `audience` from the request. It is the issuer identifier of the authorization server at which the workload will present the grant.

### 3.4. Scope - `scope`

If the request carries a `scope`, the server MAY include it in the `scope` claim, or a subset of it, according to operator policy. The server MUST NOT add scopes the workload did not request. If no scope is requested, the `scope` claim MUST be omitted.

The authorization server decides which scopes to grant. A scope in the WAG is an assertion by the Workload Identity Provider, not a grant of access.

### 3.5. Freshness

Lifetime and the `exp`, `iat` and `jti` claims are as defined in [Section 5][1] of the WAG specification. A server MUST mint a new WAG for each call to `FetchWAG` and MUST NOT return the same WAG in response to more than one call, so that authorization servers that enforce single use of a `jti` can do so.

### 3.6. Additional Claims

A server MAY include the `nickname` claim and other claims about the workload, as permitted by the WAG specification. A server MUST NOT include a claim whose value was supplied by the workload in the request, except for `aud` and `scope` as described above.

## 4. Signing Keys

WAGs are signed with keys held by the trust domain's issuing authority. The public keys are published for authorization servers as described in [Section 6.1][1] of the WAG specification, for example as a JWK Set at the `jwks_uri` of metadata published under the issuer identifier.

Keys that sign WAGs MUST NOT be used to sign any other kind of document, including JWT-SVIDs and WIT-SVIDs, and keys that sign SVIDs MUST NOT be used to sign WAGs. A JWK Set published for WAG verification SHOULD contain only WAG signing keys.

Where an implementation records WAG signing keys in the trust domain's [SPIFFE bundle](SPIFFE_Trust_Domain_and_Bundle.md), each such key MUST carry the `use` value `wag`. SVID validators ignore keys with this `use` value, as they do any `use` value unknown to them.

## 5. Service Definition

The service is defined by a Protocol Buffer (version 3) service definition in the `spiffe.wag.v1` package. The complete definition is found in [wag.proto](wag.proto).

The service is served on the SPIFFE Workload Endpoint and inherits its transport, security header, authentication, and error code rules (see the [SPIFFE Workload Endpoint](SPIFFE_Workload_Endpoint.md) specification). Implementation of this profile is OPTIONAL. Clients can discover support through gRPC Server Reflection as described in that specification. A server that does not implement this profile returns "Unimplemented" for its RPCs, and a client receiving "Unimplemented" SHOULD NOT retry.

### 5.1. Profile Definition

```protobuf
package spiffe.wag.v1;

service WorkloadAuthorizationGrant {
    // Fetch Workload Authorization Grants for the calling workload, addressed
    // to the requested audience (an OAuth authorization server). If an
    // optional SPIFFE ID is requested, only the grant for that SPIFFE ID is
    // returned.
    rpc FetchWAG(FetchWAGRequest) returns (FetchWAGResponse);
}

message FetchWAGRequest {
    // Required. Issuer identifier of the authorization server.
    string audience = 1;

    // Optional. Space-separated list of requested scopes.
    string scope = 2;

    // Optional. The SPIFFE ID to request a grant for.
    string spiffe_id = 3;
}

message FetchWAGResponse {
    // Required. One grant per returned identity.
    repeated WAG grants = 1;
}

message WAG {
    // Required. The SPIFFE ID of the workload (the `sub` claim).
    string spiffe_id = 1;

    // Required. The grant, a JWT using JWS Compact Serialization.
    bytes issued_token = 2;

    // Required. The grant's `exp` claim, in seconds since the Unix epoch.
    int64 expires_at = 3;

    // Optional. The grant's `scope` claim, if any.
    string scope = 4;

    // Optional. Operator guidance on how to use this identity.
    string hint = 5;
}
```

### 5.2. FetchWAG

The `FetchWAG` RPC is unary. It returns WAGs for one audience.

The `audience` field is mandatory and MUST contain an absolute URI. The server MAY refuse audiences that operator policy does not permit for the caller.

The `scope` field is optional and is handled as described in [Scope](#34-scope---scope).

The `spiffe_id` field is optional. If set, the server MUST return only the WAG for that SPIFFE ID, or reject the request if the caller is not entitled to it. If unset, the server MUST return a WAG for every identity the caller is entitled to, with the WAG for the default identity first (see [Default Identity](SPIFFE_Workload_API.md#8-default-identity)).

The `FetchWAGResponse` message consists of a mandatory `grants` field, which MUST contain one or more `WAG` messages. In each `WAG` message, all fields are mandatory except `scope` and `hint`. The `spiffe_id`, `expires_at` and `scope` fields MUST equal the `sub`, `exp` and `scope` claims of the grant in `issued_token`. When `hint` is set, its value MUST be unique among the grants in the response, and is subject to the same rules as in the [Default Identity](SPIFFE_Workload_API.md#8-default-identity) section of the SPIFFE Workload API.

### 5.3. Error Codes

In addition to the codes defined by the [SPIFFE Workload Endpoint](SPIFFE_Workload_Endpoint.md#6-error-codes), the server uses:

| Code | Condition | Client Behavior |
|------|-----------|-----------------|
| InvalidArgument | `audience` is missing or is not an absolute URI. | Report an error, don't retry. |
| PermissionDenied | The caller is not entitled to any identity, to the requested `spiffe_id`, or to a WAG for the requested `audience`. | Retry with a backoff. |
| Unimplemented | The server does not implement this profile. | Report an error, don't retry. |

## 6. Client Behavior

A client presents the WAG at the authorization server's token endpoint as described in [Section 5][1] of the WAG specification: `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer`, the value of `issued_token` as the `assertion`, and the target resource in the `resource` parameter.

A client SHOULD fetch a new WAG for each token request rather than reuse one, since an authorization server may reject a `jti` it has already accepted. A client MUST NOT present a WAG after its `expires_at`.

A client MUST NOT present a WAG to any party other than the authorization server named by its audience.

## 7. Security Considerations

The WAG is a bearer credential. Anyone who obtains it can redeem it at the authorization server named in its `aud` until it expires, or, where the authorization server enforces single use, until it is first redeemed. A short lifetime, a single audience, and a new WAG per call limit what a stolen WAG is worth. Binding a WAG to a key held by the workload is an open issue in the WAG specification. A future version of this profile may add a key or CSR to the request once that is settled.

Because the server signs whatever audience the caller asks for, any workload entitled to a WAG can obtain one addressed to any authorization server. This does not grant access by itself: an authorization server accepts a WAG only from a Workload Identity Provider it has been configured to trust. Operators who want to limit which authorization servers a workload can reach SHOULD restrict permitted audiences per workload.

Separating WAG signing keys from SVID signing keys, together with the distinct `typ` header, prevents a WAG from being accepted as a JWT-SVID or WIT-SVID, and the reverse.

An authorization server keys its records on the pair of issuer identifier and SPIFFE ID. Whoever controls the issuer identifier, or the DNS name under it, controls which SPIFFE IDs every trusting authorization server accepts. A compromised Workload Endpoint implementation can mint WAGs for any SPIFFE ID it can serve, as it can SVIDs.

## Appendix A. Mapping to OAuth 2.0 Token Exchange

`FetchWAG` takes its parameter names from [OAuth 2.0 Token Exchange][2]. The table lists each token exchange parameter and how it is carried in this profile.

| RFC 8693 parameter | In this profile |
|--------------------|-----------------|
| `grant_type` | Implied by the RPC. |
| `resource` | Not used. The resource is sent by the workload to the authorization server, not encoded in the WAG. |
| `audience` | `audience`. Exactly one value. |
| `scope` | `scope`. |
| `requested_token_type` | Not sent. Implied by the RPC: always `urn:ietf:params:oauth:token-type:wag`. |
| `subject_token`, `subject_token_type` | Not sent. The subject is the attested caller. `spiffe_id` selects among its identities. |
| `actor_token`, `actor_token_type` | Not used. Delegation is out of scope of the WAG specification. |
| `access_token` (response) | `issued_token`. Renamed because the issued token is a grant, not an access token. Carried as bytes. |
| `issued_token_type` (response) | Not returned. Implied by the RPC. |
| `token_type` (response) | Not used. It would always be `N_A`. |
| `expires_in` (response) | `expires_at`. An absolute time stays correct when the response is held before use. |
| `scope` (response) | `scope`. |
| `refresh_token` (response) | Not used. The workload calls `FetchWAG` again. |

## Appendix B. Example

A workload with SPIFFE ID `spiffe://example.org/agents/invoice-triage` calls `FetchWAG` with:

```
audience: "https://as.saas.example"
scope:    "issues:read"
```

The server returns one grant whose `issued_token` decodes to the following header and claims:

```json
{
  "typ": "oauth-wag+jwt",
  "alg": "ES256",
  "kid": "wag-2026-11"
}
.
{
  "iss": "https://wag.example.org",
  "sub": "spiffe://example.org/agents/invoice-triage",
  "aud": "https://as.saas.example",
  "exp": 1795142700,
  "iat": 1795142400,
  "jti": "0b6e1c0e-6b3a-4a7e-9f1e-6a3f2d1c9b77",
  "scope": "issues:read",
  "nickname": "Invoice triage"
}
```

The workload then sends this token request to the authorization server:

```
POST /token HTTP/1.1
Host: as.saas.example
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer
&assertion=eyJ0eXAiOiJvYXV0aC13YWcrand0Ii...
&resource=https%3A%2F%2Fapi.saas.example
```

[1]: https://datatracker.ietf.org/doc/draft-carleton-workload-authz-grant/
[2]: https://www.rfc-editor.org/rfc/rfc8693
[3]: https://www.rfc-editor.org/rfc/rfc2119
