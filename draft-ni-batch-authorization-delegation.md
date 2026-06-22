---
title: "Batch Authorization Delegation"
category: info

docname: draft-ni-batch-authorization-delegation-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
# area: AREA
# workgroup: WG Working Group
keyword:
  - Rich Authorization Request
  - Token Exchange
  - Identity Chaining
  - Identity Assertion
  - Batch Authorization Delegation
venue:
#  group: WG
#  type: Working Group
#  mail: oauth@ietf.org
#  arch: https://example.com/WG
  github: "NiYuan224/draft-ni-batch-authorization-delegation"
  latest: "https://NiYuan224.github.io/draft-ni-batch-authorization-delegation/draft-ni-batch-authorization-delegation.html"

author:
 -
  name: Yuan Ni
  organization: Huawei
  email: niyuan1@huawei.com

 -
  name: Chunchi Peter Liu
  organization: Huawei
  email: liuchunchi@huawei.com

normative:
  RFC2119:
  RFC8174:
  RFC6749:
  RFC9396:
  RFC8693:
  RFC7523:
  RFC9068:

informative:
  I-D.ietf-oauth-identity-chaining:
  I-D.ietf-oauth-identity-assertion-authz-grant:
  I-D.ietf-wimse-wpt:
  I-D.ietf-wimse-http-signature:
  I-D.ietf-wimse-mutual-tls:

...

--- abstract

This document describes a mechanism for batch authorization delegation, which enables a batch of fine‑grained, actor‑bound permissions in a single request and securely delegates them to multiple collaborating actors.


--- middle

# Introduction

Due to the rise of collaboration service ecosystems and AI Agents, resource access patterns are shifting from a traditional, single client-server model to complex multi-entity orchestration. For example, a network operation and management agent may coordinate specialized sub-agents, such as information collection, situation analysis, assisted decision-making sub-agents; a microservice architecture may include an orchestrator that manages a group of back-end services. Under these scenarios, a single user intent often requires the orchestration and collaboration of multiple entities.

While a single, batch authorization with a comprehensive set of permissions can minimize user interaction and improve experience, it risks granting excessive privileges to entities that require only a subset of permissions for their specific subtasks. Therefore, a critical challenge arises in multi-entity orchestration. The core issue is how to request and authorize permissions in a single batch, while securely delegating fine-grained privileges to ensure each entity obtains only the minimum permissions required for its sub-task, thereby balancing efficiency and security.

While the existing OAuth 2.0 protocol and its extensions provide a foundation for authorization, they  lack native support for efficient, secure delegation for coordinated tasks:

* OAuth 2.0{{RFC6749}}:  The OAuth 2.0 authorization framework enables a client
to obtain limited access to protected resources on behalf of a resource owner. However, in a multi-entity collaboration scenario, this requires each entity to independently initiate its own authorization flow. This results in multiple user interactions, introducing significant end-to-end latency and severe user fatigue.

* OAuth 2.0 Token Exchange{{RFC8693}}: Token Exchange defines a delegation mechanism and introduces the may_act claim to authorize an actor to act on behalf of a subject. However, the may_act claim only determines who is authorized to act. It does not provide a way to express which specific permissions are being delegated. The optional scope claim is coarse-grained and insufficient for fine-grained delegation across multiple entities.

* OAuth 2.0 Rich Authorization Request (RAR) {{RFC9396}}: RAR introduces a new parameter authorization_details to allow clients to express their fine-grained authorization requirements using the JSON structures. Such a parameter allows several fine-grained permissions to be included in a single authorization request, as well as the issued access token. However, RAR does not currently specify how these structured permissions can be partitioned and delegated to different actors during a subsequent token exchange.

In summary, the existing OAuth 2.0 protocol and its extensions lack a mechanism to express, partition, and delegate fine-grained structured permissions across multiple collaborating entities.

To bridge this gap, this document adds the delegation information directly to the multiple fine-grained permissions in the RAR. By extending the authorization_details with a may_act claim, a client can request a comprehensive set of permissions, each of which specify which actor is authorized to delegate. The Authorization Server (AS) then responds this request with a batch token, which contains the above permissions and their corresponding delegation constraints. When an actor performs a token exchange using the batch token, the AS applies a downscoping strategy, filtering the permissions based on the consistency of the may_act claim and actor's identity. This ensures a convenient authorization experience for the user while strictly enforcing the principle of least privilege across multiple collaborating entities.


# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

This document uses common OAuth and token processing terms such as "access token", "authorization server" (AS), "resource server" (RS), "authorization request", "access token response" defined by {{RFC6749}}, "Trust Domain", "JWT Authorization Grant" defined by {{I-D.ietf-oauth-identity-chaining}}, "identity assertion" defined by {{I-D.ietf-oauth-identity-assertion-authz-grant}}.

In addition, the following terms are defined for this document:

Leader-Client:  The entity responsible for coordinating tasks and requesting permissions in a batch on behalf of sub-clients. The leader-client initiates rich authorization requests to the AS, and sends the obtained batch tokens to sub-clients. The leader-client includes a master agent, a microservice orchestrator, or an API gateway.

Sub-Client:  The entity that is subordinate to the leader-client and interacts with Resource Servers (RS) to perform specific sub-tasks. The sub-client exchanges the received batch token for a Downscoped Token. Sub-clients include sub-agents and microservices.

Batch Authorization Delegation: A mechanism proposed in this document that a leader-client requests permissions in a batch and delegates them to its subordinate sub-clients.

Authorization Item: An authorization item includes a fine-grained description of a permission and its corresponding delegation constraint. Multiple authorization items are expressed with the structure of authorization_details, and included in the rich authorization request and the issued batch token.

Batch Token: A specific JWT access token issued by the AS to the leader-client. It encapsulates multiple authorization items, enabling subsequent delegation.

Delegation Constraint: The may_act claim within an authorization item that specifies the sub-client authorized to act on that permission.

Downscoped Token: The access token issued to a sub-client as a result of token exchange, which is to be used at the RS to access protected resources. It contains only the specific subset of authorization items whose delegation constraints match the requesting sub-client's identity.

# Batch Authorization Delegation

This document specifies a mechanism combining OAuth RAR {{RFC9396}} and Token Exchange {{RFC8693}} to achieve Batch Authorization Delegation.

The leader-client sends a rich authorization request with multiple authorization items to the AS. Each authorization item not only describes the required permission but also declares the sub-client to which the permission can be delegated via delegation constraints. Upon user confirmation, the AS issues a batch token to the leader-client, which encapsulates the authorization items confirmed by the user. The leader-client distributes the batch token to a sub-client.

The sub-client initiates a token exchange request to the AS. The AS matches the sub-client's identity with the delegation constraints in the batch token, and then filters out one or more authorization items corresponding to this specific sub-client. Then the AS generates a downscoped token containing only the permissions from the filtered authorization items.

Such a mechanism enables the leader-client to perform a one-time batch authorization as well as to ensure each sub-client obtains an access token containing only the permissions it requires, thereby realizing both efficiency and security.

## Overview

Figure 1 shows the flow of Batch Authorization Delegation. Note that a leader-client MAY coordinate several sub-clients. For simplicity, only one sub-client is shown in Figure 1.

~~~
+----++-------------+    +----+        +----------++----+
|User||Leader-Client|    | AS |        |Sub-Client|| RS |
+-+--++------+------+    +--+-+        +--------+-++--+-+
  |          |(1)RAR        |                   |     |
  |          |(with may_act)|                   |     |
  |          +-------------->                   |     |
  |          |(2)Ask for    |                   |     |
  |          | consent      |                   |     |
  <----------+--------------+                   |     |
  |(3)Consent|              |                   |     |
  +----------+-------------->                   |     |
  |          |(4)Issue      |                   |     |
  |          | Batch Token  |                   |     |
  |          <--------------+                   |     |
  |          |(5)Distribute |                   |     |
  |          | Batch Token  |                   |     |
  |          +--------------+------------------->     |
  |          |              |(6)Token exchange  |     |
  |          |              |   Request         |     |
  |          |              <-------------------+     |
  |          |              | [Batch Token]     |     |
  |          |              |                   |     |
  |          |              |(7)Match identity  |     |
  |          |              +-+ &Downscope      |     |
  |          |              | |                 |     |
  |          |              <-+                 |     |
  |          |              |(8)Downscoped Token|     |
  |          |              +------------------->     |
  |          |              |(9)access          |     |
  |          |              +-------------------+----->
  |          |              |                   |     |
~~~
*Figure 1: Batch Authorization Delegation Flow*

(1) The leader-client sends a rich authorization request containing multiple authorization items to the AS. Each authorization item includes a fine-grained permission (described by several claims defined in {{RFC9396}}, e.g., type, action, location, etc.) and a delegation constraint that uses the may_act claim to designate which sub-client is authorized for the permission.

(2) After receiving the rich authorization request, the AS presents all authorization items in the request to the user for consent.

(3) The user confirms all or some of the authorization items and sends the result back to the AS.

(4) The AS generates a batch token, which encapsulates the authorization items confirmed by the user and issues it to the leader-client.

(5) The leader-client distributes the batch token to the sub-client.

(6) The sub-client initiates a token exchange request to the AS to exchange the batch token to a downscoped token.

(7) The AS matches the identity of sub-client to the delegation constraints in the batch token to filter the authorization items corresponding to the sub-client.

(8) The AS generates a downscoped token that only contains the permissions in the filtered authorization items and send it to the sub-client.

(9) The sub-client accesses the RS with the downscoped token.

## Batch Token

### RAR Extension with the may_act claim

In {{RFC8693}}, the may_act is a top-level claim, which makes a statement of one party is authorized to become the actor and act on behalf of another party. It is an entity-level delegation, because it only defines who is authorized to act.

In contrast, this document leverages authorization_details defined in {{RFC9396}} to describe multiple authorization items, each of which contains a permission and a nested may_act claim serving as a delegation constraint. This helps to define who is authorized to perform which permission, thus changing the entity-level delegation into an item-level delegation.

**may_act**<br>
&nbsp;&nbsp;&nbsp;&nbsp;**REQUIRED.** A JSON object that identifies the specific sub-client authorized to exercise the permission.

The may_act JSON object contains fields that specify the intended actor for a particular authorization item. This document defines the following two fields:

**sub**<br>
&nbsp;&nbsp;&nbsp;&nbsp;**REQUIRED.** A string that uniquely identifies the sub-client authorized to exercise the permission.


**aud**<br>
&nbsp;&nbsp;&nbsp;&nbsp;**OPTIONAL.** A string that identifies the AS of the trust domain where the sub-client resides. If the sub-client and the leader-client are located in different Trust Domains, this claim MUST be present to specify the target AS. If this claim is omitted, it MUST be assumed that the sub-client and the leader-client reside in the same trust domain.

Figure 2 shows an example of authorization_details containing the may_act claim. It represents a typical travel management scenario in which the leader-client requests two distinct authorization items: a flight booking permission delegated to the Flight-Agent, and a hotel reservation permission delegated to the Hotel-Agent. Both sub-clients are located in the same trust domain as the leader-client so that the may_act.aud field is omitted. This structure demonstrates how multiple permissions can be bound to different actors within an authorization request.

```text
[
  {
    "type": "flight_booking",
    "actions": ["search", "book"],
    "locations": ["https://example.com/flights"],
    "may_act": {
        "sub": "flight_agent@example.com"
    }
  },
  {
    "type": "hotel_reservation",
    "actions": ["search", "book"],
    "locations": ["https://example.com/hotels"],
    "may_act": {
        "sub": "hotel_agent@example.com"
    }
  }
]
```
*Figure 2: authorization_details with the may_act claim*

### Processing Rules of the Authorization Server
When processing an authorization request that contains authorization_details with may_act claims, the AS MUST ask the user for consent to the requested authorization items. The general processing rules of the AS defined in {{RFC9396}} apply, with the following extensions:

* The AS MUST validate that the sub-client identified in the may_act.sub field is known. If the may_act.aud field is present, the AS MUST validate that the target AS is reachable and trusted according to local policies.

* The AS MUST present all the authorization items, including the requested permissions and their corresponding intended sub-clients to the user. The user MAY:

  * Grant all requested authorization items.

  * Grant only a subset of the authorization items.

  * For a specific item, accept the proposed sub-client or select an alternative eligible sub-client if permitted by the AS.

* The AS MUST record the consented authorization items, including the may_act claims, as part of a grant (e.g., the authorization code).

### Batch Token Issuance
After user consent, the AS issues an authorization code as a grant to the leader-client. The leader-client then exchanges the authorization code for a batch token, following the standard authorization response and token request flow as defined in {{RFC6749}}.

Subsequently,  the AS generates a batch token and returns it to the leader-client in the Token response. The batch token is a JWT access token conforming to {{RFC9068}}, with the following constraints on its claim values:

**iss**<br>
&nbsp;&nbsp;&nbsp;&nbsp;**REQUIRED.** The identifier of the AS.

**sub**<br>
&nbsp;&nbsp;&nbsp;&nbsp;**REQUIRED.** The identifier of the user who grants the consent.

**aud**<br>
&nbsp;&nbsp;&nbsp;&nbsp;**REQUIRED.** MUST be set to the AS itself, as the batch token is intended only for token exchange at the AS and cannot be directly used at the RS.

**client_id**<br>
&nbsp;&nbsp;&nbsp;&nbsp;**REQUIRED.** The identifier of the leader-client that initiated the authorization request.

**authorization_details**<br>
&nbsp;&nbsp;&nbsp;&nbsp;**REQUIRED.** An JSON array containing all the consented authorization items, including the permission descriptions and their corresponding may_act delegation constraints.

Figure 3 shows an example of the batch token JWT payload, which encapsulates all consented authorization items from Figure 2. The sub claim identifies the user who consents, the client_id claim identifies the travel assistant as the leader-client that initiates the request, each may_act.sub field identifies the sub-client authorized to exercise the corresponding permission. The aud claim is set to the AS itself, as the batch token will be presented back to the AS for further token exchange.

```text
{
  "iss": "https://as.example.com",
  "sub": "user@example.com",
  "aud": "https://as.example.com",
  "client_id": "travel_assistant@example.com",
  "exp": 1777881600,
  "iat": 1777795200,
  "jti": "batch-token-8b4729cc-32e4-4370-8cf0-5796154d1296",
  "authorization_details": [
    {
      "type": "flight_booking",
      "actions": ["search", "book"],
      "locations": ["https://example.com/flights"],
      "may_act": {
        "sub": "flight_agent@example.com"
      }
    },
    {
      "type": "hotel_reservation",
      "actions": ["search", "book"],
      "locations": ["https://example.com/hotels"],
      "may_act": {
        "sub": "hotel_agent@example.com"
      }
    }
  ]
}
```
*Figure 3: Batch Token JWT Payload*

## Downscoped Token


### Token Exchange Request

When a sub-client receives a batch token from the leader-client, it MUST perform a token exchange request to the AS to obtain a downscoped token. The parameters described in section 2.1 of {{RFC8693}} apply here with the following restrictions:

**subject_token**<br>
&nbsp;&nbsp;&nbsp;&nbsp;**REQUIRED.** The batch token received from the leader-client.

**subject_token_type**<br>
&nbsp;&nbsp;&nbsp;&nbsp;**REQUIRED.** urn:ietf:params:oauth:token-type:jwt.


Since the AS has to identify the requesting sub-client to filter the authorization items, it MUST authenticate the sub-client. The AS MAY use the following authentication methods:

*  Password-based authentication as defined in Section 2.3.1 of {{RFC6749}}.

*  Bearer JWT-based authentication as defined in {{RFC7523}}.

*  Workload Proof Token, HTTP Message Signatures, Mutual TLS as defined in {{I-D.ietf-wimse-wpt}}, {{I-D.ietf-wimse-http-signature}}, and {{I-D.ietf-wimse-mutual-tls}}.

*  Actor token as defined in {{RFC8693}}.

Figure 4 shows a token exchange request initiated by the Flight-Agent. In this request, the Flight-Agent provides the received batch token from Figure 3 as the subject_token. To satisfy the requirement for client authentication, the Flight-Agent provides its own identity token as an actor_token (detailed in Figure 5).

```text
 POST /as/token.oauth2 HTTP/1.1
 Host: as.example.com
 Content-Type: application/x-www-form-urlencoded

 grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Atoken-exchange
 &subject_token=[Encoded batch token]
 &subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Ajwt
 &actor_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJodHRwc
 zovL2FzLmV4YW1wbGUuY29tIiwiaXNzIjoiaHR0cHM6Ly9vcmlnaW5hbC1pc3N1ZXIu
 ZXhhbXBsZS5uZXQiLCJleHAiOjE3Nzg0MDAwMDAsInN1YiI6ImZsaWdodF9hZ2VudEB
 leGFtcGxlLm5ldCJ9.lcg7QKoGrsD_TICwHJnb0_Fsd5FlocXXXQhjYi2-hC4
 &actor_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Ajwt
 ```
*Figure 4: Token Exchange Request*


```text
{
  "aud":"https://as.example.com",
  "iss":"https://original-issuer.example.net",
  "exp": 1778400000,
  "sub":"flight_agent@example.net"
}
```
*Figure 5: Actor Token Claims*


### Processing Rules of the Authorization Server

In addition to authenticating the requesting sub-client, the AS MUST

* Validate the batch token provided in the subject_token.

* Retrieve the authorization_details from the batch token’s payload and filter only those authorization items whose may_act.sub fields exactly matches the authenticated sub-client’s identity.

### Downscoped Token Issuance

The AS generates a downscoped token that only contains the permissions in the filtered authorization items. The format of this token is at the discretion of the AS. It MAY be a JWT access token conforming to {{RFC9068}} or any other format. If a JWT downscoped token is used, the following constraints on its claim values apply:

**sub**<br>
&nbsp;&nbsp;&nbsp;&nbsp;**REQUIRED.** The owner who grants the consent (copied from the sub claim of the batch token in the subject_token field).

**client_id**<br>
&nbsp;&nbsp;&nbsp;&nbsp;**REQUIRED.** The identifier of the authenticated sub‑client.

**aud**<br>
&nbsp;&nbsp;&nbsp;&nbsp;**OPTIONAL.** The target RS where the sub-client intends to use the downscoped token.

**authorization_details**<br>
&nbsp;&nbsp;&nbsp;&nbsp;**OPTIONAL.** A JSON array containing the permissions from the filtered authorization items. Since the may_act claim has already been enforced as a delegation constraint by the AS during token exchange, it MAY be omitted in the Downscoped Token.

Figure 6 shows a downscoped token issued to the Flight-Agent after token exchange. Its sub claim identifies the end‑user who grants the consent, its aud claim points directly to target service (https://example.com/flights), and the client_id identifies the Flight-Agent. The authorization_details array contains only the flight‑booking permission, with the may_act claim removed.

```text
{
  "iss": "https://as.example.com",
  "sub": "user@example.com",
  "aud": "https://example.com/flights",
  "client_id": "flight_agent@example.com",
  "exp": 1777881600,
  "iat": 1777795210,
  "jti": "downscoped-token-7c8d9a2e-4b12-4a3f-8e6c-2f9a8b3c7d5e",
  "authorization_details": [
    {
      "type": "flight_booking",
      "actions": ["search", "book"],
      "locations": ["https://example.com/flights"],
    }
  ]
}
```
*Figure 6: Downscoped Token Payload*

# Scenarios
Section 3 defines the basic workflow and the message structures of Batch Authorization Delegation. On this basis, this section gives some detailed scenarios, including intra-domain and cross-domain cases.

## Intra-Domain Batch Authorization Delegation
In this case, the user, the leader-client, and the sub-clients all reside within the same trust domain managed by a single AS. The workflow and the examples are already given in Section 3 Figure.1-6.

## Batch Authorization Delegation with Cross-Domain sub-clients
In this case, the user and the leader-client belong to the same trust domain, but sub-clients reside in one or more other trust domains. Each trust domain has its own AS.

### Workflow
The workflow is a combination of batch authorization delegation and identity chaining {{I-D.ietf-oauth-identity-chaining}}.


Since the sub-client and leader-client reside in different trust domains, the leader-client in Domain A requests a JWT Authorization Grant (JAG) from the AS in Domain A via token exchange. The AS matches Domain B with the may_act.aud fields in the batch token to filter out one or more authorization items, then generates a JAG including only the Domain B-related authorization items.

The leader-client then presents the received JAG as an assertion to the AS in Domain B to obtain an access token, which is subsequently exchanged by the specific sub-client in Domain B for a final downscoped token. This token exchange ensures that the final downscoped token used by the sub-client only contains the permissions it required.


~~~
+--------++-------------+   +----------++----------+     +----------++----------+
|  User  ||Leader-Client|   |    AS    ||    AS    |     |Sub-Client||    RS    |
|Domain A||   Domain A  |   | Domain A || Domain B |     | Domain B || Domain B |
+---+----++------+------+   +-------+--++-+--------+     +----+-----++---+------+
    |            |(1)RAR            |     |                   |          |
    |            |(with may_act)    |     |                   |          |
    |            +------------------>     |                   |          |
    |            |(2)Ask for consent|     |                   |          |
    <------------+------------------+     |                   |          |
    |(3)Consent  |                  |     |                   |          |
    +------------+------------------>     |                   |          |
    |            |(4)Issue          |     |                   |          |
    |            | Batch Token      |     |                   |          |
    |            <------------------+     |                   |          |
    |            |(a)Token exchange |     |                   |          |
    |            |   Request        |     |                   |          |
    |            +------------------>     |                   |          |
    |            |(b)Match domain   |     |                   |          |
    |            |   &Downscope  +--+     |                   |          |
    |            |               |  |     |                   |          |
    |            |               +-->     |                   |          |
    |            |(c)JAG            |     |                   |          |
    |            <------------------+     |                   |          |
    |            |(d)JAG            |     |                   |          |
    |            +------------------+----->                   |          |
    |            |(e)Access Token   |     |                   |          |
    |            <------------------+-----+                   |          |
    |            |(f)Access Token   |     |                   |          |
    |            +------------------+-----+------------------->          |
    |            |                  |     |(6)Token exchange  |          |
    |            |                  |     |   Request         |          |
    |            |                  |     <-------------------+          |
    |            |                  |     |(7)Match identity  |          |
    |            |                  |     +-+ &Downscope      |          |
    |            |                  |     | |                 |          |
    |            |                  |     <-+                 |          |
    |            |                  |     |(8)Downscoped Token|          |
    |            |                  |     +------------------->          |
    |            |                  |     |                   |(9)access |
    |            |                  |     |                   +---------->
    |            |                  |     |                   |          |
~~~
*Figure 7: Batch Authorization Delegation with Cross-Domain Sub-Clients*

Steps (1)-(4) are the same as in Figure 1.

(a) The leader-client in Domain A exchanges the batch token with the AS in Domain A for a JAG that can be used in the AS in Domain B.

(b) The AS of Domain A inspects the may_act.aud fields within the batch token, filters the authorization items down to those matching Domain B.

(c) The AS of Domain A issues the JAG including the filtered authorization items. Since the may_act.aud fields have already been used for the domain-level filtering, and the top‑level aud claim of the JAG already identifies the target AS of domain B, the may_act.aud claim MAY be omitted from the issued JAG.  This step requires trust relationship between the ASs in Domain A and Domain B.  See Section 2.1 of {{I-D.ietf-oauth-identity-chaining}} for the trust relationship establishment.

(d) The leader-client in Domain A presents the JAG as an assertion to the AS of Domain B.

(e) The AS of Domain B validates the JAG, extracts and encapsulates the filtered authorization items in an access token and sends it to the leader-client. This access token itself constitutes a Domain B-specific batch token, whose authorization items are a subset of those in the original batch token issued in step (4).

(f) The leader-client sends the access token to the sub-client.

Steps (6)-(9) are the same as in Figure 1.

### Key Messages

The following shows the key tokens in the workflow of Figure 7.

#### Batch Token

Figure 8 illustrates a batch token payload issued by the Financial Alliance AS in step (4).

The batch token contains two authorization items: one for searching and calculating financial benefits at Bank_A, and another for the same actions at Bank_B. The may_act.aud field anchors the permission to the target bank's AS (e.g., https://as.bank_a.example.com), while the may_act.sub field strictly binds the execution of the search and calculation to that target bank's designated agent (e.g., benefit_agent@bank_a.example.com).

```text
{
  "iss": "https://as.alliance.example.com",
  "sub": "user@alliance.example.com",
  "aud": "https://as.alliance.example.com",
  "client_id": "finance_assistant@alliance.example.com",
  "exp": 1777881600,
  "iat": 1777795200,
  "jti": "4a7b203c-e591-4916-b8df-d2b389f4621c",
  "authorization_details": [
    {
      "type": "benefit_bank_a",
      "actions": ["search", "calculate"],
      "locations": ["https://bank_a.example.com/benefits"],
      "may_act": {
        "sub": "benefit_agent@bank_a.example.com",
        "aud": "https://as.bank_a.example.com"
      }
    },
    {
      "type": "benefit_bank_b",
      "actions": ["search", "calculate"],
      "locations": ["https://bank_b.example.com/benefits"],
      "may_act": {
        "sub": "benefit_agent@bank_b.example.com",
        "aud": "https://as.bank_b.example.com"
      }
    }
  ]
}
```
*Figure 8: Batch Token Payload*


#### JWT Authorization Grant
Figure 9 shows the JAG payload issued by the Finance Alliance AS in step (c), specifically filtered for Bank_A's domain.

In accordance with step (b), the Finance Alliance AS inspects the may_act.aud fields of the batch token (Figure 8), extracts and encapsulates the benefit_bank_a item into the JAG. Moreover, the top-level aud claim targets Bank_A's AS.

```text
{
  "iss": "https://as.alliance.example.com",
  "sub": "user@alliance.example.com",
  "aud": "https://as.bank_a.example.com",
  "client_id": "finance_assistant@alliance.example.com",
  "iat": 1777795210,
  "exp": 1777795300,
  "authorization_details": [
    {
      "type": "benefit_bank_a",
      "actions": ["search", "calculate"],
      "locations": ["https://bank_a.example.com/benefits"],
      "may_act": {
        "sub": "benefit_agent@bank_a.example.com"
      }
    }
  ]
}
```
*Figure 9: JAG Payload*

#### Access Token
Figure 10 illustrates the cross-domain Access Token payload issued by Bank_A's AS in step (e). As described in step (e), this token constitutes a batch token localized for Bank_A's domain, so that the aud claim is set to Bank_A's AS for further token exchange.

```text
{
  "iss": "https://as.bank_a.example.com",
  "sub": "user@alliance.example.com",
  "aud": "https://as.bank_a.example.com",
  "client_id": "finance_assistant@alliance.example.com",
  "iat": 1777795220,
  "exp": 1777881600,
  "jti": "2d8f93e1-7c5b-4a3d-81ee-629b3f4c7d5a",
  "authorization_details": [
    {
      "type": "benefit_bank_a",
      "actions": ["search", "calculate"],
      "locations": ["https://bank_a.example.com/benefits"],
      "may_act": {
        "sub": "benefit_agent@bank_a.example.com"
      }
    }
  ]
}
```
*Figure 10: Access Token Payload*

#### Downscoped Token
Figure 11 illustrates the final downscoped token payload issued by Bank_A's AS in step (8).

When the benefit_agent exchanges the access token, Bank_A's AS matches its authenticated identity with the may_act.sub claim, filters the corresponding permissions and removes the may_act claim entirely. Moreover, Bank_A's AS sets aud as https://bank_a.example.com/benefits, the specific target RS, and sets client_id as benefit_agent.

```text
{
  "iss": "https://as.bank_a.example.com",
  "sub": "user@alliance.example.com",
  "client_id": "benefit_agent@bank_a.example.com",
  "aud": "https://bank_a.example.com/benefits",
  "iat": 1777795230,
  "exp": 1777881600,
  "jti": "e9a4f2b1-3d7c-482e-96bb-1823c4d5f6a7",
  "authorization_details": [
    {
      "type": "benefit_bank_a",
      "actions": ["search", "calculate"],
      "locations": ["https://bank_a.example.com/benefits"]
    }
  ]
}
```
*Figure 11: Downscoped Token Payload*

## Batch Authorization Delegation with the Cross-Domain User
In this case, the leader-client and sub-clients belong to the same trust domain, while the user resides in another trust domain. Each trust domain has its own AS.

### Workflow
The workflow is a combination of batch authorization delegation and identity assertion {{I-D.ietf-oauth-identity-assertion-authz-grant}}.

The user's client obtains an Identity Assertion from the IdP of Domain U. This Identity Assertion is exchanged for an Identity Assertion JWT Authorization Grant (ID‑JAG) with authorization_details that contains several permissions from Domain U's IdP. The user’s client then directly exchanges this ID‑JAG for an access token from Domain A's AS. This access token is securely delivered to the leader‑client in Domain A.

The leader‑client uses this access token to make a token exchange request to the AS of Domain A, including an authorization_details parameter that carries may_act delegation constraints. The AS verifies that the leader‑client has only added delegation constraints without altering the original permissions, and then issues a batch token.

The remaining steps are the same as the intra‑domain batch authorization delegation flow: the leader‑client distributes the batch token to sub‑clients; each sub‑client performs a token exchange to obtain a downscoped token containing only its own permissions and uses the downscoped token to access the RS.
~~~
+--------+         +--------++--------++-------------+ +----------+ +--------+
|User’s  |         |  IdP   ||   AS   ||Leader Client| |Sub-Client| |   RS   |
|client  |         |Domain U||Domain A||   Domain A  | | Domain A | |Domain A|
|Domain U|         |        ||        ||             | |          | |        |
+-+------+         +-----+--+++-------++-------+-----+ +-----+----+ +--+-----+
  |(a)User SSO           |    |                |             |         |
  +---------------------->    |                |             |         |
  |(b)Identity Assertion |    |                |             |         |
  <----------------------+    |                |             |         |
  |(c)Token Exchange     |    |                |             |         |
  | Request              |    |                |             |         |
  +---------------------->    |                |             |         |
  |[Identity Assertion]  |    |                |             |         |
  |(d)ID-JAG (with       |    |                |             |         |
  | authorization_details|    |                |             |         |
  <----------------------+    |                |             |         |
  |(e)Present ID-JAG     |    |                |             |         |
  +----------------------+---->                |             |         |
  |(f)Access Token       |    |                |             |         |
  <----------------------+----+                |             |         |
  |(g)Access Token       |    |                |             |         |
  +----------------------+----+---------------->             |         |
  |                      |    |(h)Token Exchange             |         |
  |                      |    | Request        |             |         |
  |                      |    <----------------+             |         |
  |                      |    |(4)Issue        |             |         |
  |                      |    | Batch Token    |             |         |
  |                      |    +---------------->(5)Distribute|         |
  |                      |    |                | Batch Token |         |
  |                      |    |                +------------->         |
  |                      |    |(6)Token exchange Request     |         |
  |                      |    <----------------+-------------+         |
  |                      |    +-+              |             |         |
  |                      |    | |(7)Match identity&Downscope |         |
  |                      |    <-+              |             |         |
  |                      |    |(8)Downscoped Token           |         |
  |                      |    +----------------+------------->         |
  |                      |    |                |             |(9)Access|
  |                      |    |                |             +--------->
  |                      |    |                |             |         |

  ~~~
  *Figure 12: Batch Authorization Delegation with the Cross-Domain User*

(a) The user authenticates with the IdP Server.

(b) The user's client obtains an Identity Assertion from the IdP.

(c) The user's client requests an ID-JAG from Domain U's IdP, with authorization_details intended for use at Domain A's AS. This step requires a trust relationship between the IdP in Domain U and the AS in Domain A.

(d) Domain U's IdP issues an ID-JAG to the user's client.

(e) The user's client presents the ID-JAG as an assertion to Domain A's AS.

(f) Domain A's AS issues an access token to the user's client.

(g) The user's client sends a request to the leader-client, including the access token.

(h) The leader-client orchestrates the task and assigns different permissions to different sub-clients. It then initiates a token exchange request to Domain A's AS. This request includes the authorization_details parameter, which contains the original content of authorization_details from step (c) along with the may_act claims as delegation constraints.

(5) Domain A's AS compares the authorization_details in the request with the original ones, verifies that the leader-client has only added delegation constraints without altering the original permissions, and then issues a batch token.

Steps (6)-(9) are the same as in Figure 1.

### Key Messages
The following shows the examples of key tokens and messages in the workflow of Figure 11. The steps and messages of token exchange request and response for ID-JAG follow Section 4.3.4.2 in {{I-D.ietf-oauth-identity-assertion-authz-grant}} and are not repeated in this section.

#### Access Token
This access token is issued by Domain A's AS to the user's client in step (f) and then delivered to the leader‑client in step (g). It carries the original permissions without any delegation constraints.

```text
{
  "iss": "https://as.domain_a.example",
  "sub": "user@domain_u.example",
  "aud": "https://as.domain_a.example",
  "client_id": "user_client@domain_u.example",
  "exp": 1777881600,
  "iat": 1777795100,
  "jti": "init-9a4f2b1c-3d7e-4a2b-8c6d-1e5f9a7b3c2d",
  "authorization_details": [
    {
      "type": "flight_booking",
      "actions": ["search", "book"],
      "locations": ["https://domain_a.example.com/flights"]
    },
    {
      "type": "hotel_reservation",
      "actions": ["search", "book"],
      "locations": ["https://domain_a.example.com/hotels"]
    }
  ]
}
```
*Figure 13: Access Token Payload*


#### Token Exchange Request for the Batch Token
The leader‑client sends a token exchange request to Domain A's AS, appending the nested may_act claims to the original authorization_details.

```text
POST /token HTTP/1.1
Host: as.domain_a.example
Content-Type: application/x-www-form-urlencoded
Authorization: Basic bGVhZGVyLWNsaWVudDpzZWNyZXQ=

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&subject_token=[Encoded access token]
&subject_token_type=urn:ietf:params:oauth:token-type:access_token
&authorization_details=[{"type":"flight_booking","actions":["search","book"],"locations":["https://domain_a.example.com/flights"],"may_act":{"sub":"flight_agent@domain_a.example.com"}},{"type":"hotel_reservation","actions":["search","book"],"locations":["https://domain_a.example.com/hotels"],"may_act":{"sub":"hotel_agent@domain_a.example.com"}}]
```
*Figure 14: Token Exchange Request for Batch Token*


The AS verifies that the leader‑client has solely appended delegation constraints without modifying the original permissions, and then issues a batch token as shown in Figure 3. The subsequent steps, including distribution of the batch token to sub‑clients, token exchange initiated by each sub‑client, and issuance of downscoped tokens as shown in Figure 6, are identical to the intra‑domain flow.

The handling of verification failures (e.g., rejection or re‑consent) is implementation‑specific and out of scope for this specification.

## Combined Cross‑Domain Scenario

In some deployments, the user, the leader‑client, and the sub‑clients may each reside in different trust domains. A combination of the workflows defined in Sections 4.2 and 4.3 directly supports this scenario. Therefore, the detailed flow is omitted here for brevity.


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
