# The SPIFFE Transaction Token Profile

> **Stability: Experimental** - see [STABILITY.md](STABILITY.md).

## Status of this Memo

This document specifies a credential profile standard for the internet community, and requests discussion and suggestions for improvements. Distribution of this document is unlimited.

## Abstract

[Transaction Tokens][1] (Txn-Tokens) carry the identity of the principal and the context of a transaction through the chain of workloads that handle an external request, inside one trust domain. They are short-lived JWTs issued by a Transaction Token Service (TTS) and passed from workload to workload in the `Txn-Token` HTTP header.

A SPIFFE deployment already attests workloads, names them with SPIFFE IDs, and serves them a local endpoint. This profile lets a SPIFFE implementation act as the Transaction Token Service: it defines how a Txn-Token is issued for a SPIFFE workload, and a gRPC service on the [SPIFFE Workload Endpoint](SPIFFE_Workload_Endpoint.md) from which a workload fetches Txn-Tokens and through which it validates the Txn-Tokens it receives.

## Table of Contents

1\. [Introduction](#1-introduction)  
2\. [Terminology](#2-terminology)  
3\. [Txn-Token Trust Domains](#3-txn-token-trust-domains)  
4\. [The Txn-Token Issued Under This Profile](#4-the-txn-token-issued-under-this-profile)  
4.1. [Audience - `aud`](#41-audience---aud)  
4.2. [Requesting Workload - `req_wl`](#42-requesting-workload---req_wl)  
4.3. [Subject - `sub`](#43-subject---sub)  
4.4. [Scope - `scope`](#44-scope---scope)  
4.5. [Transaction and Request Context - `tctx` and `rctx`](#45-transaction-and-request-context---tctx-and-rctx)  
4.6. [Issuer - `iss`](#46-issuer---iss)  
4.7. [Lifetime and Transaction Identifier](#47-lifetime-and-transaction-identifier)  
5\. [Signing Keys](#5-signing-keys)  
6\. [Service Definition](#6-service-definition)  
6.1. [Profile Definition](#61-profile-definition)  
6.2. [FetchTransactionToken](#62-fetchtransactiontoken)  
6.3. [ValidateTransactionToken](#63-validatetransactiontoken)  
6.4. [Error Codes](#64-error-codes)  
7\. [Client Behavior](#7-client-behavior)  
8\. [Security Considerations](#8-security-considerations)  
Appendix A. [Mapping to the Txn-Token Request](#appendix-a-mapping-to-the-txn-token-request)  
Appendix B. [Example](#appendix-b-example)  

## 1. Introduction

The Transaction Tokens specification requires that a workload and the TTS authenticate each other, and that the TTS decide, per requesting workload, whether to issue a Txn-Token. It leaves open how this is done, and names SPIFFE as one way to provision the workload's credentials. In a SPIFFE deployment there is a simpler answer: the workload already calls a local endpoint whose implementation attests the caller. This profile adds two RPCs to that endpoint, one that returns a Txn-Token and one that validates a Txn-Token.

A Txn-Token is not an SVID. It does not identify the workload that presents it, it MUST NOT be used to authenticate a workload, and it is validated against the keys of a TTS rather than against a SPIFFE bundle. For that reason this profile is not part of the [SPIFFE Workload API](SPIFFE_Workload_API.md). It is an independent gRPC service, versioned on its own, served on the SPIFFE Workload Endpoint as described in [Extensibility and Services Rendered](SPIFFE_Workload_Endpoint.md#7-extensibility-and-services-rendered). Workloads continue to authenticate to each other with SVIDs, and carry the Txn-Token alongside.

The service follows the shape of the Txn-Token request, which is itself an [OAuth 2.0 Token Exchange][2] request: it reuses the request parameter names, but it is carried over gRPC, the requested token type is fixed by the RPC, and the requesting workload is always the attested caller. The request only selects among its identities.

## 2. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119][3].

Terms from the Transaction Tokens specification are used as follows in a SPIFFE deployment:

| Txn-Token term | In this profile |
|----------------|-----------------|
| Trust Domain | A Txn-Token trust domain, as described in [Section 3](#3-txn-token-trust-domains). To avoid confusion with a SPIFFE trust domain, this document always says "Txn-Token trust domain". |
| Transaction Token Service | The issuing authority of a Txn-Token trust domain, acting through the Workload Endpoint implementation. |
| Workload | A workload, named by its SPIFFE ID. |
| Requesting Workload | The attested caller of `FetchTransactionToken`. |

## 3. Txn-Token Trust Domains

A Txn-Token trust domain is the set of workloads within which a Txn-Token is valid. It is named by a string chosen by the operator, which appears in the `aud` claim of every Txn-Token valid in it.

A Txn-Token trust domain MUST lie within a single SPIFFE trust domain. A SPIFFE trust domain MAY contain more than one Txn-Token trust domain, for example one per application. Where a SPIFFE trust domain contains exactly one Txn-Token trust domain, its name SHOULD be the SPIFFE ID of the trust domain, for example `spiffe://example.org`.

The Transaction Tokens specification requires exactly one logical TTS per Txn-Token trust domain. All Workload Endpoint implementations that serve this profile for a Txn-Token trust domain together form that TTS: they MUST sign with keys of the same TTS, and MUST apply the same issuance policy.

Operators decide which workloads are members of each Txn-Token trust domain. A server MUST NOT issue a Txn-Token for a Txn-Token trust domain to a workload that is not a member of it, and MUST NOT validate a Txn-Token on behalf of a workload that is not a member of the Txn-Token trust domain named in its `aud`.

## 4. The Txn-Token Issued Under This Profile

A Txn-Token issued under this profile MUST conform to [Section 9][1] of the Transaction Tokens specification, including its `typ` header value `txntoken+jwt`. This section adds SPIFFE-specific requirements. It does not relax any requirement of the Transaction Tokens specification.

### 4.1. Audience - `aud`

The `aud` claim MUST be the `audience` from the request. It MUST be the name of a Txn-Token trust domain of which the caller is a member.

### 4.2. Requesting Workload - `req_wl`

The `req_wl` claim MUST be set to a SPIFFE ID of an identity the attested caller is entitled to (see [Identifying the Caller](SPIFFE_Workload_API.md#41-identifying-the-caller)). If the request carries a `spiffe_id`, `req_wl` MUST be that SPIFFE ID, and the server MUST reject the request if the caller is not entitled to it. If not, `req_wl` MUST be the SPIFFE ID of the default identity (see [Default Identity](SPIFFE_Workload_API.md#8-default-identity)). It MUST NOT be taken from any other field of the request.

When a workload requests a replacement Txn-Token, the server sets `req_wl` of the replacement in the same way, to the workload that requested the replacement.

### 4.3. Subject - `sub`

The `sub` claim identifies the principal on whose behalf the transaction is handled, as determined by the server from the `subject_token`. It is typically a user or an external client, not a workload. As required by the Transaction Tokens specification, its value MUST be unique within the Txn-Token trust domain named in `aud`.

A server MUST NOT set `sub` to the caller's SPIFFE ID merely because the caller requested the Txn-Token. Where the transaction is started by a workload acting on its own behalf, the workload MAY present a `subject_token` of type `urn:ietf:params:oauth:token-type:unsigned_json` or `urn:ietf:params:oauth:token-type:self_signed` naming its own SPIFFE ID as subject, and the server MAY accept it according to operator policy.

### 4.4. Scope - `scope`

The `scope` claim is set by the server according to operator policy, based on the requested `scope` and the scope of the `subject_token`, as described in [Section 9.2.1][1] and [Section 13.6][1] of the Transaction Tokens specification. If the scope of the `subject_token` is unknown to the server, the server MUST reject the request.

### 4.5. Transaction and Request Context - `tctx` and `rctx`

The server SHOULD determine the `tctx` claim from `request_details` and the `rctx` claim from `request_context`, as described in the Transaction Tokens specification. The server is authoritative for both claims. It MAY drop, reshape or add values according to operator policy, and the claims are its assertion, not a copy of the request.

The server MAY add values it knows from attestation to `rctx`, for example an indication that the request was received through the SPIFFE Workload Endpoint.

### 4.6. Issuer - `iss`

The `iss` claim is OPTIONAL in a Txn-Token. A server MAY include it to identify the TTS whose key signed the Txn-Token, for example while keys rotate. If present, it MUST be the same for every Txn-Token of the Txn-Token trust domain, and MUST NOT be a SPIFFE ID of a workload.

### 4.7. Lifetime and Transaction Identifier

Lifetime and the `iat`, `exp` and `txn` claims are as defined in the Transaction Tokens specification. A server MUST mint a new Txn-Token for each call to `FetchTransactionToken`.

When the `subject_token` is a Txn-Token, the server issues a replacement Txn-Token as described in [Section 13.15][1] of the Transaction Tokens specification. In particular it MUST NOT change `txn`, `sub` or `aud`, MUST NOT expand the permitted scope, and MUST NOT issue a replacement for an expired Txn-Token.

## 5. Signing Keys

Txn-Tokens are signed with keys held by the TTS of the Txn-Token trust domain. Keys that sign Txn-Tokens MUST NOT be used to sign any other kind of document, including JWT-SVIDs and WIT-SVIDs, and keys that sign SVIDs MUST NOT be used to sign Txn-Tokens. Keys of different Txn-Token trust domains SHOULD be distinct.

Where an implementation records Txn-Token signing keys in the trust domain's [SPIFFE bundle](SPIFFE_Trust_Domain_and_Bundle.md), each such key MUST carry the `use` value `txn-token`. SVID validators ignore keys with this `use` value, as they do any `use` value unknown to them. Txn-Token signing keys MUST NOT be used to validate SVIDs.

This profile defines no RPC for trust material. Workloads validate Txn-Tokens through `ValidateTransactionToken`. A workload that cannot call it obtains the keys out of band.

## 6. Service Definition

The service is defined by a Protocol Buffer (version 3) service definition in the `spiffe.txntoken.v1` package. The complete definition is found in [txntoken.proto](txntoken.proto).

The service is served on the SPIFFE Workload Endpoint and inherits its transport, security header, authentication, and error code rules (see the [SPIFFE Workload Endpoint](SPIFFE_Workload_Endpoint.md) specification). Because the Workload Endpoint attests the caller, a workload does not present a client credential to call this service, and the mutual authentication that the Transaction Tokens specification requires between workload and TTS is provided by the endpoint. Implementation of this profile is OPTIONAL. Clients can discover support through gRPC Server Reflection as described in that specification. A server that does not implement this profile returns "Unimplemented" for its RPCs, and a client receiving "Unimplemented" SHOULD NOT retry.

### 6.1. Profile Definition

```protobuf
package spiffe.txntoken.v1;

service TransactionToken {
    // Fetch a Txn-Token for a transaction the calling workload is handling.
    rpc FetchTransactionToken(FetchTransactionTokenRequest) returns (FetchTransactionTokenResponse);

    // Validate a Txn-Token on behalf of the calling workload.
    rpc ValidateTransactionToken(ValidateTransactionTokenRequest) returns (ValidateTransactionTokenResponse);
}

message FetchTransactionTokenRequest {
    // Required. Name of the Txn-Token trust domain.
    string audience = 1;

    // Required. Space-separated purpose of the transaction.
    string scope = 2;

    // Required. Token representing the principal.
    bytes subject_token = 3;

    // Required. URI identifying the type of subject_token.
    string subject_token_type = 4;

    // Optional. JSON object describing the request context.
    google.protobuf.Struct request_context = 5;

    // Optional. JSON object with details of the request.
    google.protobuf.Struct request_details = 6;

    // Optional. The SPIFFE ID to request the Txn-Token as.
    string spiffe_id = 7;
}

message FetchTransactionTokenResponse {
    // Required. The Txn-Token, a JWT using JWS Compact Serialization.
    string txn_token = 1;

    // Required. The Txn-Token's `exp` claim, in seconds since the Unix epoch.
    int64 expires_at = 2;
}

message ValidateTransactionTokenRequest {
    // Required. The Txn-Token to validate.
    string txn_token = 1;

    // Optional. The Txn-Token trust domain it must be valid in.
    string audience = 2;
}

message ValidateTransactionTokenResponse {
    // Required. The `txn` claim.
    string txn = 1;

    // Required. The `sub` claim.
    string sub = 2;

    // Required. The `req_wl` claim.
    string req_wl = 3;

    // Required. Claims contained within the payload of the Txn-Token.
    google.protobuf.Struct claims = 4;
}
```

### 6.2. FetchTransactionToken

The `FetchTransactionToken` RPC is unary. It returns one Txn-Token.

The server MUST process the request as described in [Section 11.3][1] of the Transaction Tokens specification, with the attested caller as the requesting workload. In particular, the server MUST validate the `subject_token`, including its signature where it is signed, and MUST decide according to operator policy whether the caller may obtain a Txn-Token for the requested `audience`, `scope` and subject.

The `audience` field is mandatory and MUST contain the name of a Txn-Token trust domain of which the caller is a member.

The `scope` field is mandatory and is handled as described in [Scope](#44-scope---scope).

The `subject_token` and `subject_token_type` fields are mandatory. `subject_token_type` MUST be a URI. It MAY be a token type defined in [Section 3][2] of OAuth 2.0 Token Exchange other than `urn:ietf:params:oauth:token-type:refresh_token`, one of the types defined in [Section 11.2][1] of the Transaction Tokens specification, `urn:ietf:params:oauth:token-type:txn_token` to request a replacement Txn-Token, or another type agreed with the operator. A server MUST reject a `subject_token_type` it does not support, and MUST reject `urn:ietf:params:oauth:token-type:refresh_token`.

A workload SHOULD NOT present a JWT-SVID or WIT-SVID as `subject_token`. The caller is already authenticated by the endpoint, and an SVID names a workload, not the principal of a transaction.

The `request_context` and `request_details` fields are optional. Each carries the JSON object of the parameter of the same name in the Transaction Tokens specification as a `google.protobuf.Struct`, and is processed as described in [Transaction and Request Context](#45-transaction-and-request-context---tctx-and-rctx). Because a `Struct` represents every number as a double-precision floating point value, clients SHOULD send numbers that need more precision, such as large identifiers, as strings.

The `spiffe_id` field is optional and selects the identity the Txn-Token is requested as, as described in [Requesting Workload](#42-requesting-workload---req_wl).

The `FetchTransactionTokenResponse` message consists of two mandatory fields. `txn_token` contains the Txn-Token. `expires_at` MUST equal the `exp` claim of the Txn-Token.

### 6.3. ValidateTransactionToken

The `ValidateTransactionToken` RPC is unary. It validates a Txn-Token on behalf of a workload that received it, as described in [Section 12.2][1] of the Transaction Tokens specification.

The `txn_token` field is mandatory. The `audience` field is optional. If set, it MUST name a Txn-Token trust domain of which the caller is a member. If unset, the server uses the Txn-Token trust domain of which the caller is a member, and MUST reject the request if the caller is a member of more than one.

The server MUST verify that the `typ` header is `txntoken+jwt`, that the JWS signature is valid under a signing key of the TTS of that Txn-Token trust domain, that the `aud` claim names that Txn-Token trust domain, and that the Txn-Token has not expired. A server MUST NOT use SVID signing keys for this verification.

If validation succeeds, the server returns the `txn`, `sub` and `req_wl` claims in the fields of the same name, and SHOULD return all claims of the payload in the `claims` field. Implementations MAY filter claims other than those required by the Transaction Tokens specification before returning them to the client.

Successful validation means only that the Txn-Token was issued by the TTS for this Txn-Token trust domain and is current. Whether the caller acts on the transaction is an authorization decision of the caller.

### 6.4. Error Codes

In addition to the codes defined by the [SPIFFE Workload Endpoint](SPIFFE_Workload_Endpoint.md#6-error-codes), the server uses:

| RPC | Code | Condition | Client Behavior |
|-----|------|-----------|-----------------|
| FetchTransactionToken | InvalidArgument | A mandatory field is missing, `subject_token_type` is unsupported, or `subject_token` fails validation. | Report an error, don't retry. |
| FetchTransactionToken | PermissionDenied | The caller is not entitled to the requested `spiffe_id`, is not a member of the requested Txn-Token trust domain, or operator policy refuses the requested scope or subject. | Report an error, don't retry. |
| ValidateTransactionToken | InvalidArgument | `txn_token` is missing, or the Txn-Token fails validation. | Treat the Txn-Token as invalid. |
| ValidateTransactionToken | PermissionDenied | The caller is not a member of the requested Txn-Token trust domain, or `audience` is unset and the caller is a member of more than one. | Report an error, don't retry. |
| Both | Unimplemented | The server does not implement this profile. | Report an error, don't retry. |

## 7. Client Behavior

A workload that receives an external request calls `FetchTransactionToken` with the token that came with the request as `subject_token`, and sends the returned Txn-Token in the `Txn-Token` HTTP header of each call it makes to other workloads while handling that request. It authenticates those calls with its SVIDs as usual. A Txn-Token MUST NOT be sent in the `Authorization` header and MUST NOT be used in place of an SVID.

A workload that receives a Txn-Token SHOULD validate it with `ValidateTransactionToken`, and MUST forward it unmodified on outbound calls made for the same transaction. A workload that needs to narrow the scope of the transaction, or add context to it, calls `FetchTransactionToken` with the received Txn-Token as `subject_token` and `urn:ietf:params:oauth:token-type:txn_token` as `subject_token_type`, and forwards the replacement instead.

A client MUST NOT present a Txn-Token after its `expires_at`, and MUST NOT present it outside the Txn-Token trust domain named in its `aud`. Complete Txn-Tokens MUST NOT be logged.

## 8. Security Considerations

A Txn-Token is a bearer token. Anyone who obtains it can present it inside its Txn-Token trust domain until it expires. A short lifetime, a Txn-Token trust domain limited to the workloads that need it, and mutually authenticated connections between workloads limit what a stolen Txn-Token is worth. Because every call between workloads in a SPIFFE deployment is also authenticated with an SVID, a recipient can check both who sent the request and on whose behalf.

The server issues a Txn-Token for whatever subject the `subject_token` establishes. The strength of `sub` is therefore the strength of the `subject_token` validation. Operators SHOULD restrict which workloads may request Txn-Tokens at all, typically to the workloads that receive external requests, and SHOULD restrict which `subject_token_type` values each may use. Accepting `unsigned_json` lets the caller assert any subject. It SHOULD only be allowed for workloads trusted to do so.

Separating Txn-Token signing keys from SVID signing keys, together with the distinct `typ` header, prevents a Txn-Token from being accepted as a JWT-SVID or WIT-SVID, and the reverse.

A compromised Workload Endpoint implementation can mint Txn-Tokens for any subject and any Txn-Token trust domain it serves, as it can SVIDs for any SPIFFE ID it serves.

The `rctx` and `tctx` claims may contain personal information, such as the IP address of an end user. Servers SHOULD include only what recipients need, and the privacy considerations of the Transaction Tokens specification apply.

## Appendix A. Mapping to the Txn-Token Request

`FetchTransactionToken` takes its parameter names from the Txn-Token request in [Section 11][1] of the Transaction Tokens specification. The table lists each parameter and how it is carried in this profile.

| Txn-Token request parameter | In this profile |
|-----------------------------|-----------------|
| `grant_type` | Implied by the RPC. |
| `requested_token_type` | Not sent. Implied by the RPC: always `urn:ietf:params:oauth:token-type:txn_token`. |
| `audience` | `audience`. |
| `scope` | `scope`. |
| `subject_token` | `subject_token`. Carried as bytes. |
| `subject_token_type` | `subject_token_type`. |
| `request_context` | `request_context`. A `google.protobuf.Struct`. |
| `request_details` | `request_details`. A `google.protobuf.Struct`. |
| Client authentication | Not sent. The caller is attested by the Workload Endpoint, and its SPIFFE ID becomes `req_wl`. `spiffe_id` selects among its identities. |
| `access_token` (response) | `txn_token`. Renamed because the issued token is not an access token. |
| `issued_token_type` (response) | Not returned. Implied by the RPC. |
| `token_type` (response) | Not used. It is always `N_A`. |
| `expires_in` (response) | `expires_at`. An absolute time stays correct when the response is held before use. |
| `refresh_token` (response) | Not used. |

## Appendix B. Example

A workload with SPIFFE ID `spiffe://example.org/ns/shop/sa/gateway` receives an external request with an OAuth access token, and calls `FetchTransactionToken` with:

```
audience:           "spiffe://example.org"
scope:              "trade.stocks"
subject_token:      "eyJhbGciOiJFUzI1NiIs..."
subject_token_type: "urn:ietf:params:oauth:token-type:access_token"
request_context:    { req_ip: "69.151.72.123" }
request_details:    { action: "BUY", ticker: "MSFT", quantity: "100" }
```

The server returns a Txn-Token that decodes to the following header and claims:

```json
{
  "typ": "txntoken+jwt",
  "alg": "ES256",
  "kid": "txn-2026-11"
}
.
{
  "aud": "spiffe://example.org",
  "iat": 1795142400,
  "exp": 1795142460,
  "txn": "97053963-771d-49cc-a4e3-20aad399c312",
  "sub": "d084sdrt234fsaw34tr23t",
  "scope": "trade.stocks",
  "req_wl": "spiffe://example.org/ns/shop/sa/gateway",
  "rctx": {
    "req_ip": "69.151.72.123"
  },
  "tctx": {
    "action": "BUY",
    "ticker": "MSFT",
    "quantity": "100"
  }
}
```

The workload calls the next workload over mutual TLS with its X.509-SVID and adds the header:

```
Txn-Token: eyJ0eXAiOiJ0eG50b2tlbitqd3Qi...
```

The receiving workload calls `ValidateTransactionToken` with the header value as `txn_token`, and receives the `txn`, `sub` and `req_wl` claims and the full claim set.

[1]: https://datatracker.ietf.org/doc/html/draft-ietf-oauth-transaction-tokens-11
[2]: https://www.rfc-editor.org/rfc/rfc8693
[3]: https://www.rfc-editor.org/rfc/rfc2119
