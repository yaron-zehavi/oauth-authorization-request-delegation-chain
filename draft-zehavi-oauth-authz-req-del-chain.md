---
title: "OAuth Authorization Request Delegation Chain"
abbrev: "OAuth Authz Request Delegation Chain"
category: info

docname: draft-zehavi-oauth-authz-req-del-chain-latest
submissiontype: IETF
number:
date:
consensus: false
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - oauth
 - RAR
 - authorization request
 - delegation
 - brokering
 - CIMD
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "yaron-zehavi/oauth-authorization-request-delegation-chain"
  latest: "https://yaron-zehavi.github.io/oauth-authorization-request-delegation-chain/draft-zehavi-oauth-authz-req-del-chain.html"

author:
 -
    fullname: Yaron Zehavi
    organization: Raiffeisen Bank International
    email: yaron.zehavi@rbinternational.com

normative:
  RFC6749:
  RFC7515:
  RFC7517:
  RFC7518:
  RFC7519:
  RFC7591:
  RFC7662:
  RFC8259:
  RFC8414:
  RFC9396:

informative:
  RFC8693:
  OpenID.Federation:
    title: "OpenID Federation 1.0"
    target: https://openid.net/specs/openid-federation-1_0.html
    author:
      - org: OpenID Foundation
  I-D.ietf-oauth-security-topics-update:
    title: "OAuth 2.0 Security Best Current Practice Update"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-security-topics-update/
  I-D.ietf-oauth-client-id-metadata-document:
    title: "OAuth Client ID Metadata Document"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/
  I-D.mcguinness-oauth-actor-profile:
    title: "OAuth Actor Profile for Delegation"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-actor-profile/
  I-D.mcguinness-oauth-actor-proofs:
    title: "OAuth Actor-Signed Hop Proofs"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-actor-proofs/

--- abstract

Brokered OAuth redirect authorization requests involve intermediary authorization servers between a downstream client and the upstream authorization server that obtains user consent and issues tokens.
Such deployments have security risks because the upstream authorization server sees only the immediate OAuth client and is unaware of the downstream client or intermediary brokers obtaining its response.

This document defines an OAuth 2.0 profile for carrying a verifiable, signed authorization request delegation chain as a RAR `authorization_details` object {{RFC9396}}. Each node in the chain is a JSON object signed by an attesting authorization server using detached JWS {{RFC7515}}, attesting its validated client, hash-linked to the previous node, allowing the upstream authorization server to validate the integrity of the visible delegation path and apply policy before issuing tokens.

--- middle

# Introduction

OAuth redirect authorization requests increasingly pass through intermediary authorization servers before reaching the authorization server that obtains user consent and issues tokens.

In a brokered redirect authorization flow, a downstream client's authorization request is forwarded through one or more brokers. Each broker is both an authorization server for its downstream party, and an OAuth client of the next authorization server in the path.

The terminal upstream authorization server that ultimately processes the authorization request may only have a direct relationship with the immediate broker. Without additional information, it cannot determine which downstream client initiated the request and which broker path carried the request.

Such brokered consent flows are discussed as a risk in OAuth Security Topics update {{I-D.ietf-oauth-security-topics-update}}.

This document addresses this risk for redirect authorization request delegation. It defines a RAR {{RFC9396}} `authorization_details` object that carries a signed delegation chain in the authorization request. The chain allows each authorization server in the redirect path to attest the client it directly recognizes and to preserve the prior delegation evidence.

The resulting chain allows any upstream authorization server to evaluate its integrity and perform authorization, consent, and policy decisions based on:

* The integrity of the delegation chain.
* The immediate broker client,
* The downstream client that initiated the request,
* The ordered request delegation path,
* The authorization servers or brokers that attested each hop,
* The protected resource requested, and

This document defines a RECOMMENDED `authorization_details` type for representing a verifiable signed delegation chain. Each chain node states:

* Who is attesting the node,
* Who the node is intended for,
* Which client is being attested for this hop,
* Access to which resource is requested,
* Where the node appears in the chain, and
* A cryptographic proof over the node.

Each node is signed by the attesting entity using detached JWS and hash-linked to the previous node. The result is a JSON-structured, tamper-resistant, verifiable signed delegation chain for redirect authorization request processing.

This profile is intentionally narrow. It does not define a new grant type, token format, endpoint, token response parameter, or error code. It defines only a proposed RAR {{RFC9396}} `authorization_details` type and processing rules for redirect authorization requests.

## Relation to OpenID Federation

OpenID Federation {{OpenID.Federation}} defines mechanisms for establishing trust between entities using signed entity statements, trust chains, metadata, metadata policy, and federation authorities.

The authorization request delegation chain defined by this document is similar to OpenID Federation in that both mechanisms can involve signed statements about entities and can support trust decisions across organizational or administrative boundaries.

However, the two mechanisms address different layers of the problem.

OpenID Federation primarily addresses entity trust and metadata establishment. It can answer questions such as:

* Which entity controls this identifier?
* Which metadata applies to this entity?
* Which trust anchor or federation authority vouches for this entity?
* Which keys should be used to verify statements from this entity?

This document addresses the authrization request's specific delegation path. It can answer questions such as:

* Which downstream client initiated this redirect authorization request?
* Which brokers carried it?
* Which entity attested each hop?
* Was the visible authorization request delegation chain reordered, shortened at the tail, altered in the middle, or modified?
* Is the visible first node acceptable under local policy?

Deployments MAY use OpenID Federation to establish trust in the entities that appear in a delegation chain. For example, `iss` values in this profile can correspond to federated entity identifiers, and federation metadata can be used to discover keys or validate metadata policy.

This document does not replace OpenID Federation. Instead, it can consume or complement federation trust metadata while providing a per-request signed delegation chain suitable for OAuth authorization request processing.

## Relation to OAuth Client ID Metadata Document

The OAuth Client ID Metadata Document draft (aka: CIMD) defines a mechanism by which an OAuth client can use a URL as its `client_id`, where the URL references a client metadata document that can be fetched by an authorization server {{I-D.ietf-oauth-client-id-metadata-document}}.

This document is complementary to that mechanism and can reference CIMD-style `client_id` values when used.

A delegation node can use a CIMD-style `client_id` by setting `client_ns` to `cimd` and `client_id` to the metadata document URL. For example:

~~~ json
{
  "client_ns": "cimd",
  "client_id": "https://client.example.com/oauth-client-metadata.json"
}
~~~

In such deployments, CIMD can provide retrievable client metadata, while this profile provides a signed per-request authorization request delegation chain showing how that client was carried through brokers.

## Relation to RFC 8693 Token Exchange and the `act` Claim

OAuth 2.0 Token Exchange {{RFC8693}} defines a token exchange grant and includes the `act` claim for representing an actor in issued tokens. The `act` claim can indicate that one party is acting on behalf of another party. Nested `act` claims can represent prior actors.

This document is related to the `act` claim because both mechanisms represent delegation. However, they apply at different phases of an OAuth deployment.

The `act` claim is a token-time representation. It appears in issued tokens or token introspection responses and is consumed after token issuance, typically by resource servers or downstream authorization servers.

This document defines an authorization-request-time representation. The delegation chain is carried in a RAR `authorization_details` object during the redirect authorization request, before the upstream authorization server has issued tokens.

The distinction is important for brokered redirect authorization flows. The upstream authorization server needs to know the downstream client and broker path before it can make a correct consent or authorization decision. A token claim such as `act` can describe delegation after issuance, but it does not by itself provide a redirect authorization request mechanism for presenting signed per-hop delegation evidence to the authorization endpoint before consent and token issuance.

This profile also differs from nested `act` claims in that:

* Each delegation node is signed by the authorization server or broker that attests that hop,
* Each node is hash-linked to the previous node,
* The chain is carried as JSON in RAR `authorization_details`,
* The chain is intended for authorization endpoint processing, and
* The upstream authorization server can bind consent to the terminal client and broker path before issuing tokens.

An authorization server MAY translate a validated delegation chain into issued-token claims, including `act` claims, after authorization succeeds. Such token representation is outside the scope of this document.

## Relation to the OAuth Actor Profile for Delegation

The OAuth Actor Profile for Delegation draft defines a common profile for representing delegated actor relationships using the `act` claim across JWT assertion grants, JWT access tokens, Transaction Tokens, and Token Exchange inputs. It also defines actor classification through `sub_profile` and discovery metadata for advertising support {{I-D.mcguinness-oauth-actor-profile}}.

This document is complementary to the OAuth Actor Profile but has a different scope.

The OAuth Actor Profile addresses token and assertion interoperability. It helps systems consistently express actor relationships in issued artifacts such as:

* JWT assertion grants,
* JWT access tokens,
* Transaction Tokens, and
* Token Exchange inputs and outputs.

This document addresses redirect authorization request delegation. It helps an upstream authorization server evaluate a brokered authorization request before token issuance by carrying a signed delegation chain in RAR `authorization_details`.

The two mechanisms can be used together. An authorization server can validate an `oauth_request_delegation_chain` authorization detail during the redirect authorization request and, after successful authorization, issue a token using the `act` claim profile defined by the OAuth Actor Profile.

In that combined model:

* This document provides pre-token authorization request evidence, and
* The OAuth Actor Profile provides post-authorization token representation.

## Relation to OAuth Actor-Signed Hop Proofs

OAuth Actor-Signed Hop Proofs defines an optional companion profile for delegated OAuth tokens that conform to the OAuth Actor Profile for Delegation. It introduces an `actor_proofs` claim containing a signed per-hop proof chain, where each visible actor signs its own participation and target binding {{I-D.mcguinness-oauth-actor-proofs}}.

This document is similar in that it also uses signed per-hop evidence and hash linking. However, the placement and processing model are different.

OAuth Actor-Signed Hop Proofs is token-oriented. It defines claims and mechanisms for delegated tokens and associated token processing.

This document is authorization-request-oriented. It carries the signed chain in RAR `authorization_details` so that the upstream authorization server can evaluate the delegation path during redirect authorization request processing, before consent and token issuance.

A deployment could use both mechanisms:

* `oauth_request_delegation_chain` in the redirect authorization request to support upstream authorization server consent and policy decisions.
* `act` and `actor_proofs` in issued tokens to support downstream resource server enforcement and audit.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

**Authorization Request Delegation Chain**:
: An ordered JSON array of signed delegation nodes carried in an OAuth authorization request.

**Delegation Node**:
: A JSON object representing one attestation event in the authorization request delegation chain.

**Attester**:
: The entity identified by the node's `iss` value. The attester signs the node.

**Attested Client**:
: The client identified by the node's `client_ns` and `client_id` values.

**Terminal Client**:
: The original or downstream client ultimately represented through the chain.

**Broker**:
: An entity that acts as an authorization server in one relationship and as an OAuth client of another authorization server or broker in another relationship.

**Client Namespace**:
: The client identifier namespace or resolution mode. This document defines `as` for AS-local client identifiers and `cimd` for URL-shaped client identifiers resolved using the OAuth Client ID Metadata Document mechanism.

# Motivation

In a simple OAuth authorization request, the authorization server evaluates the authenticated client, requested resource, requested authorization details, and user consent.

In a brokered redirect authorization request, the authorization server may see only the final broker as the OAuth client. For example:

~~~ text
client-123 -> broker-a -> broker-b -> broker-c -> as-domain-1
~~~

When `as-domain-1` receives the authorization request, it may only directly recognize `broker-c`. It may not know that the original downstream client was `client-123`, nor that the request passed through `broker-a` and `broker-b`.

If the upstream authorization server binds consent only to the immediate broker, then consent granted for one downstream client can be reused for a different downstream client that reaches the upstream authorization server through the same broker. This is the shared consent problem described in {{I-D.ietf-oauth-security-topics-update}}.

This document allows each intermediate authorization server in the redirect path to add a signed delegation node:

~~~ text
Hop 1: broker-a -> broker-b
       broker-a attests client-123

Hop 2: broker-b -> broker-c
       broker-b attests broker-a-client

Hop 3: broker-c -> as-domain-1
       broker-c attests broker-b-client
~~~

The requested protected resource remains constant across the chain:

~~~ text
resource = https://api-domain-1.example.com
~~~

The `aud` value in the delegation chain identifies only the next authorization server.

By validating the signed and hash-linked chain, any upstream authorization server can bind consent and policy to the entire redirect delegation path up to itself, rather than only to the immediate client which might not be the terminal client.

# Protocol Overview

A downstream client initiates an OAuth authorization request which is forwarded to other authorization servers. Each authorization server or broker can create or append a signed node to the delegation chain.

~~~ ascii-art
+------------+       +----------+       +----------+       +----------+       +-------------+
| client-123 |       | broker-a |       | broker-b |       | broker-c |       | as-domain-1 |
+------------+       +----------+       +----------+       +----------+       +-------------+
      |                    |                  |                  |                    |
      | Authorization      |                  |                  |                    |
      | Request Intent     |                  |                  |                    |
      |------------------->|                  |                  |                    |
      |                    | attests          |                  |                    |
      |                    | client-123       |                  |                    |
      |                    |----------------->|                  |                    |
      |                    |                  | attests          |                    |
      |                    |                  | broker-a-client  |                    |
      |                    |                  |----------------->|                    |
      |                    |                  |                  | attests            |
      |                    |                  |                  | broker-b-client    |
      |                    |                  |                  |------------------->|
      |                    |                  |                  |                    | Validate chain
      |                    |                  |                  |                    | Apply consent
      |                    |                  |                  |                    | and policy
~~~

Figure: Brokered authorization request using an OAuth Authorization Request Delegation Chain

The upstream authorization server validates the final node from the broker it directly knows, then walks the prior signed nodes to validate the full delegation path and identify the terminal client.

# OAuth Broker Client Metadata

This document defines client metadata that allows an authorization server, when acting as a client, to identify itself as an OAuth broker.

The following client metadata attribute is defined:

~~~ json
{
  "client_roles": ["oauth_broker"]
}
~~~

`client_roles`
: OPTIONAL. JSON array of strings identifying roles the OAuth client is expected to perform when interacting with the authorization server. The value `oauth_broker` indicates that the client may act as an intermediary between the authorization server and one or more downstream clients.

A client metadata value of `oauth_broker` is a statement a client makes about its expected role. It does not by itself establish trust in the client, any downstream client, or any brokered delegation path.

An authorization server MAY also classify a client as an OAuth broker using local policy, even when the `client_roles` metadata member is absent.

# Authorization Details Type

This profile defines the following proposed `authorization_details` type:

~~~ json
"oauth_request_delegation_chain"
~~~

A delegated authorization request MAY include an authorization detail object of this type:

~~~ json
{
  "type": "oauth_request_delegation_chain",
  "chain": [
    {
      "iss": "https://broker-c.example.com",
      "aud": "https://as-domain-1.example.com",
      "n": 2,
      "p_hash": "base64url-sha256-of-previous-node",
      "client_ns": "as",
      "client_id": "broker-b-client",
      "resource": ["https://api-domain-1.example.com"],
      "proof": {
        "jws": "eyJhbGciOiJFUzI1NiIsImtpZCI6ImJyb2tlci1jLWtleS0xIn0..base64url-signature"
      }
    }
  ]
}
~~~

The `chain` member is a JSON array. JSON arrays are ordered by definition {{RFC8259}}. The explicit `n` value is nevertheless included to make event order unambiguous across storage, transformation, validation, and partial processing.

# Discovering Upstream Support

An authorization server acting as broker that intends to forward an authorization request to an upstream authorization server SHOULD determine whether the upstream authorization server supports the `oauth_request_delegation_chain` authorization details type before sending the authorization request.

The broker SHOULD retrieve the upstream authorization server metadata according to {{RFC8414}}.

If the upstream authorization server metadata publishes in `authorization_details_types_supported` support for the `oauth_request_delegation_chain` RAR type, the broker SHOULD opt into this mechanism by including an `authorization_details` object of type `oauth_request_delegation_chain` in the authorization request.

For example, an upstream authorization server can advertise support as follows:

~~~ json
{
  "issuer": "https://as-domain-1.example.com",
  "authorization_endpoint": "https://as-domain-1.example.com/authorize",
  "token_endpoint": "https://as-domain-1.example.com/token",
  "jwks_uri": "https://as-domain-1.example.com/jwks.json",
  "authorization_details_types_supported": [
    "oauth_request_delegation_chain"
  ]
}
~~~

A broker that discovers upstream support SHOULD either:

* create a new `oauth_request_delegation_chain` authorization detail object, if no chain is already present; or
* validate and extend the existing chain, if a chain is already present.

A broker MAY still use this profile with an upstream authorization server when support is established by other means, such as bilateral configuration, federation metadata, contractual onboarding, or local policy.

If a broker is unable to determine whether the upstream authorization server supports this profile, the broker SHOULD apply local policy. Local policy can include forwarding the request without this authorization detail, aborting the transaction, or using an alternative delegation mechanism.

# Delegation Chain Object

The delegation chain authorization details object has the following members.

| Member | Required | Description |
|---|---:|---|
| `type` | Yes | Authorization details type. Value: `oauth_request_delegation_chain`. |
| `chain` | Yes | Ordered JSON array of delegation nodes. |

The `chain` array MUST contain one or more delegation nodes.

# Delegation Node

A delegation node is a JSON object with the following members.

| Member | Required | Description |
|---|---:|---|
| `iss` | Yes | Issuer identifier of the attesting entity. |
| `sub` | No | Subject being carried through the chain, such as a user. |
| `aud` | Yes | Intended authorization server or broker-AS audience of this node. This profile uses `aud` only for authorization servers and broker-AS entities, not protected resource APIs. |
| `n` | Yes | Zero-based chain position. |
| `p_hash` | Yes | Hash of the previous signed node. `null` for the first node. |
| `client_ns` | Yes | Client identifier namespace or resolution mode. This profile defines `as` and `cimd`. |
| `client_id` | Yes | Client attested by this node. |
| `client_name` | No | Human-readable display name. Not a security identifier. |
| `resource` | No | Resource indicators or protected resource identifiers relevant to the authorization request. |
| `omit_chain` | No | Whether a broker is allowed to omit the delegation chain when forwarding to an upstream authorization server that does not support this profile. Values are `forbidden` and `allowed`; default is `forbidden`. |
| `proof` | Yes | Cryptographic proof object. |

Additional members MAY be included only if their signing-payload representation is defined by this document, a future specification, or a mutually understood extension. A receiver MUST reject unsupported extension members unless local policy explicitly allows them to be ignored.

The stable security identifier for an attested client depends on the `client_ns` value.

For `client_ns` value `as`, the stable identifier is:

~~~ text
AS issuer context + client_id
~~~

For `client_ns` value `cimd`, the stable identifier is:

~~~ text
client_id
~~~

For `client_ns` value `as`, the applicable AS issuer context is the `iss` value of the node that attests the client.

The `client_name` value is display-only and MUST NOT be used as a security identifier.

# Proof Object

The `proof` object contains a detached JWS compact serialization.

| Member | Required | Description |
|---|---:|---|
| `jws` | Yes | Detached JWS compact serialization over the delegation node payload. |

Example:

~~~ json
{
  "jws": "eyJhbGciOiJFUzI1NiIsImtpZCI6ImJyb2tlci1jLWtleS0xIn0..MEUCIQD..."
}
~~~

The `proof.jws` value is a JWS Compact Serialization {{RFC7515}} with a detached payload. The payload segment is empty, resulting in the following form:

~~~ text
BASE64URL(UTF8(JWS Protected Header)) || "." || "" || "." || BASE64URL(JWS Signature)
~~~

The JWS Protected Header MUST contain an `alg` value and a `kid` value.

For example, the protected header could be:

~~~ json
{
  "alg": "ES256",
  "kid": "broker-c-key-1"
}
~~~

The detached JWS payload is the deterministic serialization of the delegation node excluding the `proof` member.

The key used to verify `proof.jws` is resolved from the attester identified by `iss`.

A verifier obtains the attester's verification key by resolving:

~~~ text
iss -> authorization server metadata -> jwks_uri -> JWK selected by kid
~~~

Because brokers in this profile are also authorization servers, a broker is expected to publish authorization server metadata {{RFC8414}} and a `jwks_uri`.

## Upstream Support Failure

If a broker receives an authorization request containing an `oauth_request_delegation_chain` authorization detail object and determines that the next upstream authorization server does not support this profile, the broker MUST NOT silently discard the delegation chain.

The broker MUST either reject the transaction, use another trusted mechanism to preserve the delegation context, or forward the request without the chain only when doing so is permitted by local policy and by the signed chain requirement.

A delegation node MAY contain the following member:

`omit_chain`
: OPTIONAL. String indicating whether a broker is allowed to forward the
authorization request without the delegation chain if the next upstream
authorization server does not support this profile. Defined values are
`forbidden` and `allowed`. If omitted, the default value is `forbidden`.

If any validated node contains:

~~~ json
"omit_chain": "forbidden"
~~~

or omits `omit_chain`, then a broker that cannot forward the delegation chain to the next upstream authorization server, and cannot preserve equivalent delegation context by another trusted mechanism, MUST reject the transaction.

If every validated node contains:

~~~ json
"omit_chain": "allowed"
~~~

then the broker MAY forward the authorization request without the delegation chain, subject to local policy.

A broker MUST NOT silently discard a delegation chain. Forwarding without the chain is an explicit degradation of delegation evidence and is permitted only when allowed by the validated chain and by local policy.

# Attesting a Client

Attesting a client means:

~~~ text
The entity identified by iss attests to the entity identified by aud
that the client identified by client_ns and client_id is the delegated
client for this authorization request hop.
~~~

For example:

~~~ json
{
  "iss": "https://broker-c.example.com",
  "aud": "https://as-domain-1.example.com",
  "client_ns": "as",
  "client_id": "broker-b-client"
}
~~~

means:

~~~ text
broker-c attests to as-domain-1 that broker-b-client
is the delegated client for this hop.
~~~

The upstream authorization server can then validate prior nodes to discover the terminal downstream client.

# Signature Input {#signature-input}

The wire format of a delegation node is JSON. The node is not a JWT and MUST NOT be processed as a JWT claims set.

Each node is signed using JSON Web Signature (JWS) {{RFC7515}} with a detached payload. The detached payload is the deterministic serialization of the delegation node excluding the `proof` member. The resulting compact detached JWS is carried in the node's `proof.jws` member.

The use of JWS provides standard JOSE header handling, including `alg` and `kid`, while preserving a JSON wire format that can be validated using typed schemas before cryptographic verification.

## Delegation Chain Signing Payload

The detached JWS payload is a UTF-8 string formed by joining name-value lines with line feed `\n`.

The first line is:

~~~ text
oauth-authorization-request-delegation-chain-v1
~~~

Then each supported member is serialized in the following fixed order when present:

~~~ text
iss
sub
aud
n
p_hash
client_ns
client_id
client_name
resource
omit_chain
~~~

The `proof` member is excluded.

Members not included in this signing-payload definition are not protected by the node signature. Security-relevant extensions therefore MUST define how they are included in the signing payload, or receivers MUST reject them.

Array values are serialized by joining each array element's JSON string serialization, in array order, with a comma character. No extra whitespace is inserted.

For example:

~~~ text
oauth-authorization-request-delegation-chain-v1
iss=https://broker-c.example.com
aud=https://as-domain-1.example.com
n=2
p_hash=Vh6U...
client_ns=as
client_id=broker-b-client
resource=https://api-domain-1.example.com
~~~

This UTF-8 string is used as the detached JWS payload.

The compact detached JWS is produced according to {{RFC7515}} by signing the payload with the algorithm identified by the JWS Protected Header `alg` value. The JWS Protected Header MUST contain `alg` and `kid`.

The compact detached JWS serialization stored in `proof.jws` MUST contain an empty payload segment:

~~~ text
protected-header || "." || "" || "." || signature
~~~

Future specifications MAY define alternative signing-payload schemes. Such specifications MUST identify the scheme unambiguously.

# Hash Chain

Each delegation node is hash-linked to the previous signed node.

For node `i`, define:

~~~ text
signing_payload_i = deterministic detached JWS payload for node i
detached_jws_i = proof.jws value for node i
event_hash_i = BASE64URL(SHA-256(UTF8(signing_payload_i) || "." || ASCII(detached_jws_i)))
~~~

The following rules apply:

~~~ text
chain[0].p_hash = null
chain[i].p_hash = event_hash(chain[i - 1]) for i > 0
chain[i].n = chain[i - 1].n + 1
~~~

This construction cryptographically binds each node to the signed event that
precedes it. Validation of the hash chain, sequence numbers, and audience
continuity detects modification, insertion, deletion, reordering, and signature
substitution within the visible chain.

# Creating or Adding to a Delegation Chain

This section defines processing rules for an authorization server or broker creating a new delegation chain or adding a node to an existing chain.

## Step 1 - Determine the Attester

The attester sets `iss` to its issuer identifier:

~~~ json
"iss": "https://attester.example.com"
~~~

The `iss` value MUST identify the entity signing the node.

The `iss` value SHOULD resolve to authorization server metadata containing a `jwks_uri` {{RFC8414}}.

## Step 2 - Determine the Audience

The attester sets `aud` to the intended authorization server or broker-AS recipient of the node.

For an intermediate broker:

~~~ json
"aud": "https://next-broker.example.com"
~~~

For the final upstream authorization server:

~~~ json
"aud": "https://as-domain-1.example.com"
~~~

The `aud` value MUST NOT be used to identify the protected resource API. Protected resources are identified using the `resource` member.

## Step 3 - Determine the Attested Client

The attester sets:

~~~ json
"client_ns": "as",
"client_id": "client-or-broker-identifier"
~~~

or:

~~~ json
"client_ns": "cimd",
"client_id": "https://client.example.com/oauth-client-metadata.json"
~~~

For an AS-local client known by the attesting broker or authorization server:

~~~ json
"client_ns": "as",
"client_id": "client-123"
~~~

For a client using the OAuth Client ID Metadata Document mechanism:

~~~ json
"client_ns": "cimd",
"client_id": "https://client.example.com/oauth-client-metadata.json"
~~~

The authorization server can use the client metadata document to obtain client metadata according to {{I-D.ietf-oauth-client-id-metadata-document}}, while using the delegation chain to validate the transaction-specific authorization request delegation path.

The mechanism for resolving metadata from `client_ns` and `client_id` is determined by local policy, federation metadata, or client metadata mechanisms.

## Step 4 - Set the Position

If creating a new chain:

~~~ json
"n": 0,
"p_hash": null
~~~

If extending an existing chain, the attester sets `n` to the previous node's `n` plus one and sets `p_hash` to `event_hash(previous_node)`.

## Step 5 - Add Optional Display or Resource Information

The attester MAY include `client_name` for user interface purposes.
The attester MAY include `resource` to identify intended protected resources.
The `client_name` value MUST NOT be used as a security identifier.
Additional claims MAY be added as parties see fit, subject to local policy or future specifications.

## Step 6 - Sign the Node

The attester constructs the detached JWS payload as described in {{signature-input}}.
The attester creates a JWS Protected Header containing at least:

~~~ json
{
  "alg": "ES256",
  "kid": "attester-key-1"
}
~~~

The attester signs the detached JWS payload using the private key corresponding to the public key published in its `jwks_uri`.

The attester places the compact detached JWS in `proof.jws`:

~~~ json
"proof": {
  "jws": "eyJhbGciOiJFUzI1NiIsImtpZCI6ImF0dGVzdGVyLWtleS0xIn0..base64url-signature"
}
~~~

The JWS payload segment MUST be empty in the compact serialization, because the payload is detached and represented by the delegation node JSON object itself.

## Step 7 - Forward the Chain

The attester includes the updated chain in an `authorization_details` object with type `oauth_request_delegation_chain`.

# Validating a Delegation Chain

An authorization server validating a delegation chain performs the following
checks.

Validation is anchored at the receiving authorization server and proceeds from
the final node of the chain toward the first node.  This reflects the trust
model: the final node is the node addressed to the receiving authorization
server, and each valid signed node commits to the previous node through
`p_hash`.

A node's signature is over the node's deterministic detached JWS payload,
including its `p_hash` value and excluding the `proof` member.  Therefore, a
valid signature on node `i` commits to node `i - 1` when `p_hash` is valid.

## Step 1 - Schema Validation

The authorization server validates that the authorization detail object
contains:

~~~ json
"type": "oauth_request_delegation_chain"
~~~

and that `chain` is a non-empty JSON array.

Each node MUST contain:

~~~ text
iss
aud
n
p_hash
client_ns
client_id
proof.jws
~~~

A receiver MAY reject nodes containing unsupported values or unsupported
extension members.

## Step 2 - Client Namespace Validation

The authorization server verifies that each node contains a supported `client_ns` value.

This profile defines:

~~~ text
as
cimd
~~~

If the authorization server does not support the `client_ns` value, it MUST reject the authorization detail object.

## Step 3 - Terminal Node Checks {#delegation-chain-terminal-node-checks}

Let `last` be the index of the final node in the chain.

The authorization server performs the following checks on the final node before
performing signature validation. These checks are fail-fast checks over
unauthenticated input: failure is sufficient to reject the request, but success
does not authenticate the node or the chain.

The authorization server verifies that the final node is intended for it:

~~~ text
chain[last].aud == receiving_authorization_server_issuer
~~~

If the final node's `aud` value does not identify the receiving authorization
server, the authorization server MUST reject the chain.

The authorization server also verifies that the final node is plausibly bound
to the OAuth client submitting the authorization request.

The exact binding is deployment-specific, but the receiving authorization server
MUST be able to establish that:

~~~ text
chain[last].iss
~~~

identifies, or is authorized to speak for, the authenticated OAuth client submitting the request.

For example, if the request is submitted by an authenticated broker client, the
authorization server can verify that the registered metadata for that OAuth
client identifies:

~~~ text
chain[last].iss
~~~

as the broker authorization server or broker issuer for that client.

If the authorization server cannot bind the final node's `iss` to the OAuth
client submitting the request, it MUST reject the chain.

The checks in this step are fail-fast checks.  Until the final node's signature
has been verified, the authorization server MUST treat the final node's `iss`,
`aud`, and other members as unauthenticated input.  A successful preflight check
does not by itself authenticate the node or the chain.

The `aud` value identifies an authorization server or broker-AS, not a
protected resource API.  Protected resource identifiers are represented using
the `resource` member.

A protected API endpoint MUST NOT appear in `aud`.

## Step 4 - Signature Validation

For each node, the authorization server:

1. Reads `iss`. The `iss` value is untrusted until the node signature is verified. Before using
`iss` for metadata retrieval, the authorization server MUST apply its normal
issuer validation, discovery, allow-list, federation, or local trust policy.
2. Reads `proof.jws`.
3. Parses `proof.jws` as a compact detached JWS.
4. Verifies that the compact JWS contains an empty payload segment.
5. Decodes the JWS Protected Header.
6. Verifies that the JWS Protected Header contains `alg` and `kid`.
7. Resolves the issuer metadata for `iss`. The resolved authorization server metadata issuer value MUST match `iss`.
8. Obtains the issuer's `jwks_uri`.
9. Fetches the issuer's JWK Set.
10. Selects a key using the JWS Protected Header `kid`.
11. Constructs the deterministic detached JWS payload for the node by serializing the node excluding the `proof` member.
12. Verifies the detached JWS signature over that payload according to
    {{RFC7515}}.

If a signature cannot be verified, the authorization server MUST reject the chain.

After signature validation succeeds, the authorization server treats the signed
members of each node as authenticated statements by that node's `iss`.

In particular, the terminal preflight checks in {{delegation-chain-terminal-node-checks}}
are then authenticated because the final node's signature covers the same
`iss`, `aud`, `client_ns`, `client_id`, `p_hash`, and other signed members.

## Step 5 - Backward Chain Validation

The authorization server validates the chain from the final node toward the first node.
For every `i` from `last` down to `1`, the authorization server verifies:

~~~ text
chain[i].p_hash == event_hash(chain[i - 1])
chain[i - 1].aud == chain[i].iss
chain[i].n == chain[i - 1].n + 1
~~~

where `event_hash` is computed over the previous node's deterministic detached JWS payload and its `proof.jws` value.

If any hash comparison fails, the authorization server MUST reject the chain.

If any audience-continuity check fails, the authorization server MUST reject the chain.

If any sequence-number check fails, the authorization server MUST reject the chain.

The authorization server then verifies the first node:

~~~ text
chain[0].n == 0
chain[0].p_hash == null
~~~

If either check fails, the authorization server MUST reject the chain.

The audience-continuity check ensures that every hop intentionally delegated to the next hop in the chain:

~~~ text
chain[i - 1].aud == chain[i].iss
~~~

Thus, for a chain:

~~~ text
node[0] -> node[1] -> node[2]
~~~

the following MUST hold:

~~~ text
node[0].aud == node[1].iss
node[1].aud == node[2].iss
~~~

## Step 6 - Resource Consistency Validation

If multiple nodes contain `resource`, the authorization server SHOULD verify
that the resource value is consistent across the chain, unless local policy
explicitly permits resource transformation.

If local policy permits resource transformation, the authorization server
SHOULD verify that each transformation is allowed for the issuer performing the
transformation.

## Step 7 - Delegation Relationship Validation

The authorization server SHOULD verify that adjacent nodes are semantically
consistent.

The final node identifies the client that the immediate trusted broker is
attesting.  Prior nodes reveal what that client was itself carrying.

For example, if the final node is:

~~~ json
{
  "iss": "https://broker-c.example.com",
  "client_ns": "as",
  "client_id": "broker-b-client"
}
~~~

then the authorization server treats `https://broker-c.example.com` as
attesting `broker-b-client`.

The authorization server then validates the prior node signed by broker-b to
determine which client broker-b was carrying.

The authorization server MAY reject the chain if the attested client
relationship is inconsistent with registration metadata, federation metadata, or
local policy.

## Step 8 - Policy Validation

After cryptographic validation, the authorization server applies local policy.

Cryptographic validation proves the integrity of the visible chain.  It does not
prove that no upstream delegation context existed before `chain[0]`.

In particular, a broker can originate a new chain beginning with itself as
`chain[0]`.  Such re-origination can produce a cryptographically valid chain.
Whether that chain is acceptable is a local policy decision for the receiving
authorization server.

Policy decisions can consider:

* the OAuth client authenticated to the authorization server,
* the final node's `iss`, whether it is allowed for the authenticated OAuth client,
* the first node's `iss`, whether it is allowed to appear as a first-node issuer,
* the full set of brokers in the chain,
* whether each broker is allowed to appear in its position in the chain,
* the client identified by the first visible client attestation,
* the resources identified by `resource`,
* the user subject identified by `sub`, if present,
* the full delegation chain hash, and
* deployment-specific expectations about allowed direct and indirect paths.

The final node's `iss` is used to validate the relationship between the
authenticated OAuth client and the broker-AS or AS that produced the final
delegation-chain node.

The first node's `iss` is used to evaluate whether the visible chain is allowed
to begin with that issuer.  This is the policy check that addresses
head-truncation or re-origination.  Cryptographic validation cannot prove that
no upstream nodes existed before `chain[0]`.

The authorization server MAY reject the request if any broker, client,
namespace, resource, subject, first-node issuer, or path is not allowed.

# Consent Binding

In brokered OAuth, an authorization server SHOULD NOT bind consent only to:

~~~ text
user + immediate broker + requested access
~~~

Instead, when a valid delegation chain is present, the authorization server SHOULD bind consent to:

~~~ text
user
+ authorization server issuer
+ immediate broker
+ client identity accepted as the terminal client
+ broker path
+ resource
+ delegation chain hash
~~~

For example:

~~~ text
user-456
+ https://as-domain-1.example.com
+ https://broker-c.example.com
+ client-123
+ broker-a -> broker-b -> broker-c
+ https://api-domain-1.example.com
+ hash(chain)
~~~

This prevents consent granted to one downstream client from being silently reused by another downstream client through the same broker.

# Authorization Server Considerations

An authorization server that supports this profile SHOULD advertise support for the `oauth_request_delegation_chain` authorization details type using the `authorization_details_types_supported` authorization server metadata member defined by {{RFC9396}}.

An authorization server that receives an `oauth_request_delegation_chain` authorization detail object SHOULD evaluate whether the chain is required for the requested transaction.

An authorization server MAY require an `oauth_request_delegation_chain` authorization detail object when the immediate OAuth client is known or determined to be an OAuth broker.

For this purpose, an authorization server MAY determine that the immediate client is an OAuth broker based on:

* The client's registered `client_roles` metadata containing `oauth_broker`;
* Local client configuration;
* A trusted software statement;
* Federation metadata or metadata policy;
* Contractual onboarding;
* Transaction context; or
* Any other trusted local policy input.

If the authorization server determines that the immediate client is acting as an OAuth broker, and local policy requires this profile for the request, then the authorization request MUST contain an `authorization_details` object with:

~~~ json
{
  "type": "oauth_request_delegation_chain"
}
~~~

If the required `oauth_request_delegation_chain` authorization detail object is absent, the authorization server MUST reject the authorization request using normal OAuth 2.0 authorization endpoint error signaling {{RFC6749}}.

If the authorization server can safely redirect to a valid registered redirection URI for the client, the authorization server SHOULD return:

~~~ text
error=invalid_request
error_description=authorization_details must contain an object with type "oauth_request_delegation_chain"
~~~

For example:

~~~ http
HTTP/1.1 302 Found
Location: https://client.example/callback?
  error=invalid_request&
  error_description=authorization_details%20must%20contain%20an%20object%20with%20type%20%22oauth_request_delegation_chain%22&
  state=af0ifjsldkj
~~~

If the `oauth_request_delegation_chain` authorization detail object is present but malformed, semantically invalid, contains an unsupported value, or fails validation, the authorization server SHOULD reject the request according to RAR error processing rules {{RFC9396}}.

If the authorization server supports this profile and delegates the authorization request to another upstream authorization server, it SHOULD include the `oauth_request_delegation_chain` authorization detail object and extend the delegation chain with details about the current hop.

An authorization server MAY include the approved `authorization_details` object in an access token or token introspection response when appropriate. However, this document does not define a token format or require the chain to be propagated to resource servers.

Where token size or privacy considerations apply, an authorization server SHOULD consider storing the validated chain server-side and exposing only necessary authorization results to resource servers via token introspection {{RFC7662}}.

# Security Considerations

## Chain Integrity

The hash chain, per-node detached JWS signatures, sequence numbers, and audience
continuity checks are intended to detect modification, insertion, deletion,
reordering, and signature substitution within the visible delegation chain.

A receiver MUST reject a chain if any required validation check described in
{{validating-a-delegation-chain}} fails.

A valid signature proves only that the identified issuer signed the node. It
does not imply that the issuer is trusted for the requested delegation.

## Truncation and Re-origination

Truncation has three relevant cases.

Tail truncation is detected by the terminal audience check: a shortened chain
will not end in a node whose `aud` identifies the receiving authorization
server.

Middle removal is detected by the hash-chain and audience-continuity checks:
removing an intermediate node breaks the successor's `p_hash` and the adjacent
issuer/audience relationship.

Head truncation, or re-origination, is different. A broker can create a fresh
chain beginning with itself as `chain[0]`. Cryptographic validation proves the
integrity of the visible chain, but cannot prove that no upstream context
existed before `chain[0]`.

Authorization servers MUST handle re-origination through local policy,
including whether `chain[0].iss` is allowed to appear as the first visible
issuer for the requested client, resource, and deployment context.

## Replay

This profile does not define expiration, nonce, or replay-cache claims in the
base structure.

Deployments that require replay protection MAY add such claims as
deployment-specific extensions and validate them according to local policy.

## Display Names

`client_name` is intended only for display.

Authorization servers MUST NOT use `client_name` as a security identifier.

The stable security identifier depends on `client_ns`, `client_id`, and the applicable issuer or namespace context.

## Trust in Attesters

A valid chain establishes integrity and provenance of the visible attestations.
It does not establish that the attesters, clients, resources, subjects, or path
are acceptable.

Authorization servers MUST apply local trust policy before accepting a
delegation chain.

## Trust in Broker Client Metadata

The `client_roles` client metadata member can indicate that a client is expected
to act as an OAuth broker.

An authorization server MUST NOT treat self-asserted `client_roles` metadata as
proof that the client is trustworthy, authorized to broker authorization
requests, or authorized to represent downstream clients.

An authorization server MUST rely on `client_roles` for security decisions only
when the metadata was established through a trusted mechanism, such as
administrative registration, trusted dynamic client registration, a trusted
software statement, federation metadata, or local trust policy.

## Metadata Resolution

This profile assumes that attesters publish verification keys through
authorization server metadata and `jwks_uri`.

If metadata cannot be resolved, is not trusted, or does not contain the key
identified by `kid`, the receiver MUST reject the affected node.

## Immediate Client Authentication

The delegation chain does not replace OAuth client authentication.

An authorization server MUST still authenticate the immediate OAuth client
according to its normal OAuth processing rules.

The authorization server MUST verify that the authenticated immediate client is
consistent with the final delegation node.

## Privacy

A delegation chain can reveal intermediaries, downstream clients, resources, and
possibly subjects.

Deployments SHOULD minimize included data and avoid including unnecessary
personally identifiable information.

# IANA Considerations

## OAuth Dynamic Client Registration Metadata Registration

This document requests registration of the following value in the IANA "OAuth Dynamic Client Registration Metadata" registry established by {{RFC7591}}.

Client Metadata Name:
: `client_roles`

Client Metadata Description:
: JSON array of strings identifying roles the OAuth client is expected to perform when interacting with the authorization server. The value `oauth_broker` indicates that the client may act as an intermediary between the authorization server and one or more downstream clients, applications, agents, relying parties, resource servers, or trust domains.

Change Controller:
: IETF

Specification Document:
: This document.

## OAuth Authorization Details Type

This document defines the `oauth_request_delegation_chain` authorization details type.

A future standards-track version of this document may request registration of the `oauth_request_delegation_chain` authorization details type in the applicable IANA registry.

--- back

# Brokered OAuth Example Without CIMD

This example uses AS-local client identifiers.

~~~ text
client-123 -> broker-a -> broker-b -> broker-c -> as-domain-1
~~~

Each AS or broker attests the client it directly recognizes:

* broker-a attests `client-123` to broker-b.
* broker-b attests `broker-a-client` to broker-c.
* broker-c attests `broker-b-client` to as-domain-1.

The `aud` value always identifies the next authorization server or broker-AS. The protected API is represented only by the `resource` member.

## Full Authorization Details Object

~~~ json
{
  "type": "oauth_request_delegation_chain",
  "chain": [
    {
      "iss": "https://broker-a.example.com",
      "aud": "https://broker-b.example.com",
      "n": 0,
      "p_hash": null,
      "client_ns": "as",
      "client_id": "client-123",
      "client_name": "Client 123",
      "resource": ["https://api-domain-1.example.com"],
      "proof": {
        "jws": "eyJhbGciOiJFUzI1NiIsImtpZCI6ImJyb2tlci1hLWtleS0xIn0..sig0"
      }
    },
    {
      "iss": "https://broker-b.example.com",
      "aud": "https://broker-c.example.com",
      "n": 1,
      "p_hash": "hash-of-node-0-event",
      "client_ns": "as",
      "client_id": "broker-a-client",
      "client_name": "Broker A",
      "resource": ["https://api-domain-1.example.com"],
      "proof": {
        "jws": "eyJhbGciOiJFUzI1NiIsImtpZCI6ImJyb2tlci1iLWtleS0xIn0..sig1"
      }
    },
    {
      "iss": "https://broker-c.example.com",
      "aud": "https://as-domain-1.example.com",
      "n": 2,
      "p_hash": "hash-of-node-1-event",
      "client_ns": "as",
      "client_id": "broker-b-client",
      "client_name": "Broker B",
      "resource": ["https://api-domain-1.example.com"],
      "proof": {
        "jws": "eyJhbGciOiJFUzI1NiIsImtpZCI6ImJyb2tlci1jLWtleS0xIn0..sig2"
      }
    }
  ]
}
~~~

## Interpretation

Node 0 says:

~~~ text
broker-a attests to broker-b that client-123 is the delegated client.
~~~

Node 1 says:

~~~ text
broker-b attests to broker-c that broker-a-client is the delegated client
for this hop.
~~~

Node 2 says:

~~~ text
broker-c attests to as-domain-1 that broker-b-client is the delegated
client for this hop.
~~~

The upstream AS validates the final trusted hop first:

~~~ text
broker-c -> broker-b-client
~~~

Then walks the prior signed nodes:

~~~ text
broker-b -> broker-a-client
broker-a -> client-123
~~~

The terminal client is therefore:

~~~ json
{
  "client_ns": "as",
  "client_id": "client-123",
  "client_name": "Client 123",
  "attested_by": "https://broker-a.example.com"
}
~~~

The resource is constant across the chain:

~~~ json
["https://api-domain-1.example.com"]
~~~

# Brokered OAuth Example With CIMD

This example uses CIMD-style URL-shaped client identifiers.

~~~ text
client-123 -> broker-a -> broker-b -> broker-c -> as-domain-1
~~~

Each AS or broker attests the client it directly recognizes:

* broker-a attests the CIMD-identified terminal client to broker-b.
* broker-b attests the CIMD-identified broker-a client to broker-c.
* broker-c attests the CIMD-identified broker-b client to as-domain-1.

The `aud` value always identifies the next authorization server or broker-AS. The protected API is represented only by the `resource` member.

## Full Authorization Details Object

~~~ json
{
  "type": "oauth_request_delegation_chain",
  "chain": [
    {
      "iss": "https://broker-a.example.com",
      "aud": "https://broker-b.example.com",
      "n": 0,
      "p_hash": null,
      "client_ns": "cimd",
      "client_id": "https://client-123.example.com/oauth-client-metadata.json",
      "resource": ["https://api-domain-1.example.com"],
      "proof": {
        "jws": "eyJhbGciOiJFUzI1NiIsImtpZCI6ImJyb2tlci1hLWtleS0xIn0..sig0"
      }
    },
    {
      "iss": "https://broker-b.example.com",
      "aud": "https://broker-c.example.com",
      "n": 1,
      "p_hash": "hash-of-node-0-event",
      "client_ns": "cimd",
      "client_id": "https://broker-a.example.com/client",
      "resource": ["https://api-domain-1.example.com"],
      "proof": {
        "jws": "eyJhbGciOiJFUzI1NiIsImtpZCI6ImJyb2tlci1iLWtleS0xIn0..sig1"
      }
    },
    {
      "iss": "https://broker-c.example.com",
      "aud": "https://as-domain-1.example.com",
      "n": 2,
      "p_hash": "hash-of-node-1-event",
      "client_ns": "cimd",
      "client_id": "https://broker-b.example.com/client",
      "resource": ["https://api-domain-1.example.com"],
      "proof": {
        "jws": "eyJhbGciOiJFUzI1NiIsImtpZCI6ImJyb2tlci1jLWtleS0xIn0..sig2"
      }
    }
  ]
}
~~~

## Interpretation

Node 0 says:

~~~ text
broker-a attests to broker-b that the CIMD-identified client is the
delegated client.
~~~

Terminal client:

~~~ json
{
  "client_ns": "cimd",
  "client_id": "https://client-123.example.com/oauth-client-metadata.json",
  "attested_by": "https://broker-a.example.com"
}
~~~

Node 1 says:

~~~ text
broker-b attests to broker-c that broker-a is the delegated client for
this hop, identified by https://broker-a.example.com/client.
~~~

Node 2 says:

~~~ text
broker-c attests to as-domain-1 that broker-b is the delegated client for
this hop, identified by https://broker-b.example.com/client.
~~~

The upstream AS sees the immediate trusted path as:

~~~ text
broker-c -> broker-b -> broker-a -> client-123
~~~

And the resource remains:

~~~ json
["https://api-domain-1.example.com"]
~~~

# Example Signing Payload

For this node:

~~~ json
{
  "iss": "https://broker-c.example.com",
  "aud": "https://as-domain-1.example.com",
  "n": 2,
  "p_hash": "hash-of-node-1-event",
  "client_ns": "as",
  "client_id": "broker-b-client",
  "resource": ["https://api-domain-1.example.com"],
  "proof": {
    "jws": "eyJhbGciOiJFUzI1NiIsImtpZCI6ImJyb2tlci1jLWtleS0xIn0..sig2"
  }
}
~~~

the detached JWS payload is:

~~~ text
oauth-authorization-request-delegation-chain-v1
iss=https://broker-c.example.com
aud=https://as-domain-1.example.com
n=2
p_hash=hash-of-node-1-event
client_ns=as
client_id=broker-b-client
resource=https://api-domain-1.example.com
~~~

The JWS Protected Header is:

~~~ json
{
  "alg": "ES256",
  "kid": "broker-c-key-1"
}
~~~

The `proof.jws` value is the compact detached JWS over the UTF-8 bytes of the detached JWS payload.

# Document History

-01

* Added `client_roles` OAuth Dynamic Client Registration metadata with `oauth_broker` as a client role value.
* Added IANA request for the `client_roles` client metadata name.
* Added broker processing rules for discovering upstream support for the `oauth_request_delegation_chain` authorization details type.
* Added authorization server processing rules for rejecting brokered authorization requests that omit a required `oauth_request_delegation_chain` authorization detail.
* Added mitigation of chain truncation and tampering.

-00

* Initial version.
* Defined `oauth_request_delegation_chain` authorization details type.
* Defined signed authorization request delegation nodes using `iss`, `aud`, `client_ns`, and `client_id`.
* Defined `client_ns` values `as` and `cimd`.
* Defined detached JWS proof processing using `proof.jws`.
* Defined `p_hash` hash-chain processing.
* Added processing rules for creating, extending, and validating delegation chains.
* Added brokered OAuth examples with and without CIMD.

# Acknowledgments
{:numbered="false"}

The author would like to thank the participants in the OAuth Working Group discussions on brokered OAuth, Rich Authorization Requests, client metadata, actor delegation, and authorization request security.
