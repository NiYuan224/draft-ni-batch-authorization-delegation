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
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
#  group: WG
#  type: Working Group
#  mail: WG@example.com
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

informative:
  I-D.ietf-oauth-identity-chaining:
  I-D.ietf-oauth-identity-assertion-authz-grant:
  RFC2119:
  RFC8174:
  RFC6749:
  RFC9396:
  RFC8693:
  RFC7523:
  I-D.ietf-wimse-wpt:
  I-D.ietf-wimse-http-signature:
  I-D.ietf-wimse-mutual-tls:

...

--- abstract

This document describes a mechanism for batch authorization delegation, which enables a batch of fine‑grained, actor‑bound permissions in a single request and securely allocates them to multiple collaborating actors.


--- middle

# Introduction

Due to the rise of collaboration service ecosystems and AI Agents, resource access patterns are shifting from a traditional, single client-server model to complex multi-entity orchestration. For example, a network operation and management agent may coordinate specialized sub-agents, such as information collection, situation analysis, assisted decision-making sub-agents; a miscroservice architecture may include an orchestrator that manages a group of back-end services. Under these scenarios, a single user intent often requires the orchestration and collaboration of multiple entities. 

While a single, batch authorization with a comprehensive set of permissions can minimize user interaction and improve experience, it risks granting excessive privileges to entities that only need a subset of permissions for their specific subtasks. Therefore, a critical challenge arises in multi-entity orchestration. The core issue is how to request and authorize permissions in a single batch, while securely delegating fine-grained privileges to ensure each entity obtains only the minimum permissions required for its sub-task, thereby balancing efficiency and security.

While the existing OAuth 2.0 protocal and its extensions provide a foundation for authorization, they still face difficulties in supporting efficient, secure delegation for coordinated tasks:

* OAuth 2.0{{RFC6749}}:  The OAuth 2.0 authorization framework enables a client 
to obtain limited access to protected resources on behalf of a resource owner. However, in a multi-entity collaboration scenario, this requires each entity to independently initiate its own authorization flow. This results in multiple user interactions, introducing significant end-to-end latency and severe user fatigue.

* OAuth 2.0 Token Exchange{{RFC8693}}: Token Exchange defines a delegation mechanism and introduces the may_act claim to authorize an actor to act on behalf of a subject. However, this parameter only determines who is authorized to act. It does not provide a way to express which specific permissions are being delegated. The optional scope claim is coarse-grained and insufficient for fine-grained delegation across multiple entities.

* OAuth 2.0 Rich Authorization Request (RAR) {{RFC9396}}: RAR introduces a new parameter authorization_details to allow clients to express their fine-grained authorization requirements using the JSON structures. Such a parameter allows several fine-grained permissions to be included in a single authorization request, as well as the issued access token. However, RAR does not currently specify how these structured permissions can be partitioned and delegated to different actors during a subsequent token exchange.

In summary, the existing OAuth 2.0 protocal and its extensions lack a mechanism to express, partition, and delegate fine-grained structured permissions across multiple collaborating entities. 

To bridge this gap, this document adds the delegation information directly to the multiple fine-grained permissions in the RAR. By extending the authorization_details with a may_act claim, a client can request a comprehensive set of permissions, each of which specify which actor is authorized to delegate. The Authorization Server (AS) then responds this request with a Batch Token, which contains the above permissions and their corresponding delegation constraints. When an actor performs a token exchange using the Batch Token, the AS applies a downscoping strategy, filtering the permissions based on the consistancy of the may_act claim and actor's identity. This ensures a convenient authorization experience for the user while strictly enforcing the principle of least privilege across multiple collaborating entities. 


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
