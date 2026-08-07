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

This document defines an informative OAuth 2.0 profile for carrying a verifiable, signed authorization request delegation chain as a RAR `authorization_details` object {{RFC9396}}. Each node in the chain is a JSON object signed by the attesting authorization server or broker using detached JWS {{RFC7515}}, attesting its validated client, hash-linked to the previous node, allowing the upstream authorization server to validate the exact delegation path before issuing tokens.

This document does not define new OAuth endpoints, grant types, error codes, token formats, token response parameters, or token request parameters.

--- middle

# Introduction

OAuth redirect authorization requests increasingly pass through intermediary authorization servers before reaching the authorization server that obtains user consent and issues tokens.

In a brokered redirect authorization flow, a downstream client may initiate an authorization request through one or more brokers. Each broker is both an authorization server for its downstream party, and an OAuth client of the next authorization server or broker in the path.

The upstream authorization server that ultimately processes the redirect authorization request may only have a direct relationship with the immediate broker. Without additional information, it may be unable to determine which downstream client initiated the request or which broker path carried the request.

The OAuth Security Topics update {{I-D.ietf-oauth-security-topics-update}} describes a shared consent problem in brokered OAuth deployments: an upstream authorization server can grant consent to a broker without being able to distinguish which downstream client is actually using that brokered access. This can result in consent granted for one downstream client being reused by another downstream client through the same broker.

This document addresses that problem for redirect authorization request delegation. It defines a RAR {{RFC9396}} `authorization_details` object that carries a signed delegation chain in the authorization request. The chain allows each authorization server or broker in the redirect path to attest the client it directly recognizes and to preserve the prior delegation evidence.

The resulting chain allows the upstream authorization server to make authorization, consent, and policy decisions based on:

* The immediate broker client,
* The downstream client that initiated the request,
* The ordered broker path,
* The authorization servers or brokers that attested each hop,
* The protected resource requested, and
* The integrity of the delegation chain.

This document defines a RECOMMENDED `authorization_details` type for representing a verifiable signed delegation chain. Each chain node states:

* Who is attesting the node,
* Who the node is intended for,
* Which client is being attested for this hop,
* Which resource is involved,
* Where the node appears in the chain, and
* A cryptographic proof over the node.

Each node is signed by the attesting entity using detached JWS and hash-linked to the previous node. The result is a JSON-structured, schema-validatable, tamper-resistant, verifiable signed delegation chain for use during redirect authorization request processing.

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

This document addresses transaction-specific authorization request delegation path preservation. It can answer questions such as:

* Which downstream client initiated this redirect authorization request?
* Which brokers carried the request?
* Which entity attested each hop?
* Was the authorization request delegation chain reordered, truncated, or modified?

Deployments MAY use OpenID Federation to establish trust in the entities that appear in a delegation chain. For example, `iss` values in this profile can correspond to federated entity identifiers, and federation metadata can be used to discover keys or validate metadata policy.

This document does not replace OpenID Federation. Instead, it can consume or complement federation trust metadata while providing a per-request signed delegation chain suitable for OAuth authorization request processing.

## Relation to OAuth Client ID Metadata Document

The OAuth Client ID Metadata Document draft (aka: CIMD) defines a mechanism by which an OAuth client can use a URL as its `client_id`, where the URL references a client metadata document that can be fetched by an authorization server {{I-D.ietf-oauth-client-id-metadata-document}}.

This document is complementary to that mechanism and points to CIMD client_id's when such were used.

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

This document allows each broker or authorization server in the redirect path to add a signed delegation node:

~~~ text
Hop 1: broker-a -> broker-b
       broker-a attests client-123

Hop 2: broker-b -> broker-c
       broker-b attests broker-a-client

Hop 3: broker-c -> as-domain-1
       broker-c attests broker-b-client
~~~

The protected resource remains constant across the chain:

~~~ text
resource = https://api-domain-1.example.com
~~~

The `aud` value in the delegation chain identifies only the next authorization server.

By validating the signed and hash-linked chain, the upstream authorization server can bind consent and policy to the full redirect delegation path rather than only to the immediate broker.

# Protocol Overview

A downstream client initiates an OAuth authorization request through one or more brokers. Each authorization server or broker can create or append a signed node to the delegation chain.

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

The upstream authorization server validates the final node from the broker it directly knows, then walks the prior signed nodes to identify the terminal client and the full broker path.

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
| `proof` | Yes | Cryptographic proof object. |

Additional members MAY be included by deployments or future specifications. Receivers MUST ignore unknown members unless local policy requires rejecting them.

The stable security identifier for an attested client depends on the `client_ns` value.

For `client_ns` value `as`, the stable identifier is:

~~~ text
AS issuer context + client_id
~~~

For `client_ns` value `cimd`, the stable identifier is:

~~~ text
client_id
~~~

The applicable AS issuer context is determined from the attesting node or from the prior node that originally introduced the AS-local client.

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

# Operation: `attest_client`

This profile defines one operation value:

~~~ json
"attest_client"
~~~

The operation means:

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
broker-c attests to as-domain-1 that broker-b-client is the delegated
client for this hop.
~~~

The upstream authorization server can then validate prior nodes to discover the terminal downstream client.

# Signature Input {#signature-input}

The wire format of a delegation node is JSON. The node is not a JWT and MUST NOT be processed as a JWT claims set.

Each node is signed using JSON Web Signature (JWS) {{RFC7515}} with a detached payload. The detached payload is the deterministic serialization of the delegation node excluding the `proof` member. The resulting compact detached JWS is carried in the node's `proof.jws` member.

The use of JWS provides standard JOSE header handling, including `alg` and `kid`, while preserving a JSON wire format that can be validated using typed schemas before cryptographic verification.

## Delegation Chain Signing Payload Version 1

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
~~~

The `proof` member is excluded.

Array values are serialized as comma-separated JSON string values in array order.

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

This construction binds both the delegation node payload and the JWS protected header and signature. It protects against:

* reordering nodes,
* inserting nodes,
* deleting nodes,
* modifying a prior node,
* replacing a prior signed node with a different signed node, and
* substituting a different JWS protected header or signature for a prior node.

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

An authorization server validating a delegation chain performs the following checks.

## Step 1 - Schema Validation

The authorization server validates that the authorization detail object contains:

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

A receiver MAY reject nodes containing unsupported values or unsupported extension members.

## Step 2 - Client Namespace Validation

The authorization server verifies that each node contains a supported `client_ns` value.

This profile defines:

~~~ text
as
cimd
~~~

If the authorization server does not support the `client_ns` value, it MUST reject the authorization detail object.

## Step 3 - Ordering Validation

The authorization server verifies:

~~~ text
chain[0].n == 0
chain[0].p_hash == null
chain[i].n == chain[i - 1].n + 1
~~~

for every `i > 0`.

## Step 4 - Hash-Chain Validation

For every node after the first, the authorization server verifies:

~~~ text
chain[i].p_hash == event_hash(chain[i - 1])
~~~

where `event_hash` is computed over the previous node's deterministic detached JWS payload and its `proof.jws` value.

If any hash comparison fails, the authorization server MUST reject the chain.

## Step 5 - Signature Validation

For each node, the authorization server:

1. Reads `iss`.
2. Reads `proof.jws`.
3. Parses `proof.jws` as a compact detached JWS.
4. Verifies that the compact JWS contains an empty payload segment.
5. Decodes the JWS Protected Header.
6. Verifies that the JWS Protected Header contains `alg` and `kid`.
7. Resolves the issuer metadata for `iss`.
8. Obtains the issuer's `jwks_uri`.
9. Fetches the issuer's JWK Set.
10. Selects a key using the JWS Protected Header `kid`.
11. Constructs the deterministic detached JWS payload for the node by serializing the node excluding the `proof` member.
12. Verifies the detached JWS signature over that payload according to {{RFC7515}}.

If a signature cannot be verified, the authorization server MUST reject the chain.

## Step 6 - Audience Validation

The authorization server verifies that the final node is intended for it:

~~~ text
chain[last].aud == receiving_authorization_server_issuer
~~~

For intermediate nodes, the authorization server SHOULD verify:

~~~ text
chain[i].aud == chain[i + 1].iss
~~~

This ensures that the chain path matches the intended authorization server or broker-AS path.

The `aud` value identifies an authorization server or broker-AS, not a protected resource API. Protected resource identifiers are represented using the `resource` member.

## Step 7 - Resource Consistency Validation

If multiple nodes contain `resource`, the authorization server SHOULD verify that the resource value is consistent across the chain, unless local policy explicitly permits resource transformation.

A protected API endpoint MUST NOT appear in `aud`.

## Step 8 - Relationship Validation

The authorization server SHOULD verify that adjacent nodes are consistent.

For a chain:

~~~ text
node[0] -> node[1] -> node[2]
~~~

the following should hold:

~~~ text
node[0].aud == node[1].iss
node[1].aud == node[2].iss
~~~

The final node identifies the client that the immediate trusted broker is attesting. Prior nodes reveal what that client was itself carrying.

For example, if the final node is:

~~~ json
{
  "iss": "https://broker-c.example.com",
  "client_ns": "as",
  "client_id": "broker-b-client"
}
~~~

then the authorization server treats broker-c as attesting broker-b-client.

The authorization server then validates the prior node signed by broker-b to determine which client broker-b was carrying.

## Step 9 - Policy Validation

After cryptographic validation, the authorization server applies local policy.

Policy decisions can consider:

* the immediate client authenticated to the authorization server,
* the final node's `iss`,
* the full set of brokers in the chain,
* the terminal client identified by the earliest client attestation,
* the resources identified by `resource`,
* the user subject identified by `sub`, if present, and
* the full delegation chain hash.

The authorization server MAY reject the request if any broker, client, namespace, resource, or path is not allowed.

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
+ terminal client identity
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

An authorization server that receives an `oauth_request_delegation_chain` authorization detail object SHOULD evaluate whether the chain is required for the requested transaction.

If the authorization server supports this profile but the chain is absent, incomplete, or invalid, the authorization server MAY reject the request according to normal RAR processing rules {{RFC9396}}.

If the authorization server supports this profile and delegates the authorization request to another upstream authorization server, it SHOULD include the `oauth_request_delegation_chain` authorization detail object and extend the delegation chain with details about the current hop.

An authorization server MAY include the approved `authorization_details` object in an access token or token introspection response when appropriate. However, this document does not define a token format or require the chain to be propagated to resource servers.

Where token size or privacy considerations apply, an authorization server SHOULD consider storing the validated chain server-side and exposing only necessary authorization results to resource servers via token introspection {{RFC7662}}.

# Security Considerations

## Tampering

The hash chain and per-node detached JWS signatures are intended to detect node modification, insertion, deletion, reordering, and signature substitution.

A receiver MUST reject a chain if any hash-link or signature check fails.

## Replay

This profile does not define expiration, nonce, or replay-cache claims in the base structure.

Deployments that require replay protection MAY add additional claims, such as timestamps, nonces, transaction identifiers, or request references, as deployment-specific extensions.

## Display Names

`client_name` is intended only for display.

Authorization servers MUST NOT use `client_name` as a security identifier.

The stable security identifier depends on `client_ns` and applicable issuer context as described in this document.

## Trust in Attesters

A valid signature proves only that the attester signed the node. It does not imply that the attester is trusted for the requested delegation.

Authorization servers MUST apply local trust policy before accepting any attester, broker, namespace, client, or delegation path.

## Metadata Resolution

This profile assumes that attesters publish verification keys through authorization server metadata and `jwks_uri`.

If metadata cannot be resolved, or if the key identified by `kid` cannot be found, the receiver MUST reject the affected node.

## Immediate Client Authentication

The delegation chain does not replace OAuth client authentication.

An authorization server MUST still authenticate the immediate OAuth client according to its normal OAuth processing rules.

The authorization server SHOULD verify that the authenticated immediate client is consistent with the final delegation node.

## Privacy

A delegation chain can reveal intermediaries, downstream clients, resources, and possibly subjects.

Deployments SHOULD minimize included data and avoid including unnecessary personally identifiable information.

# IANA Considerations

This document makes no IANA requests.

A future standards-track version of this document may request registration of the `oauth_request_delegation_chain` authorization details type.

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
