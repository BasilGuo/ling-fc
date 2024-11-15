---

title: "A Profile for Synchronizing Forwarding Commitments (FCs) without AS_PATH Leakage"
abbrev: "Forwarding Commitments"
category: std

docname: draft-ling-fc-verification-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
# area: AREA
# workgroup: WG Working Group
# keyword:
# - next generation
# - unicorn
# - sparkling distributed ledger
venue:
#  group: WG
#  type: Working Group
#  mail: WG@example.com
#  arch: https://example.com/WG

author:
  -
      fullname: Ke Xu
      org: Tsinghua University
      city: Beijing
      country: China
      email: xuke@tsinghua.edu.cn
  -
      fullname: Sitong Ling
      org: Tsinghua University
      city: Beijing
      country: China
      email: lingst21@mails.tsinghua.edu.cn
  -
      fullname: Xiaoliang Wang
      org: Tsinghua University
      city: Beijing
      country: China
      email: wangxiaoliang0623@foxmail.com
  -
      fullname: Jianping Wu
      org: Tsinghua University
      city: Beijing
      country: China
      email: jianping@cernet.edu.cn

normative:
    RFC3779:
    RFC4271:
    RFC5652:
    RFC6480:
    RFC6485:
    RFC6488:
    RFC7908:
    RFC8205:

informative:

--- abstract

This document defines a standard profile for synchronizing Forwarding Commitment (FC) across on-path ASes without leaking more AS_PATH information than BGP. An FC is a digitally signed object that verifies that an IP address prefix is announced from `AS a` to `AS b`. On-path ASes can then verify the FCs and filter the potential malicious BGP routes, which are generated based on previously received BGP-UPDATE messages.


--- middle

# Introduction

The fundamental cause of the path manipulation attacks in inter-domain routing is that the de facto Border Gateway Protocol (BGP) {{RFC4271}} does not have any built-in mechanism to authenticate routing announcements. As a result, an adversary can announce virtually arbitrary paths to a prefix while the receiver network cannot effectively verify the authenticity of the route announcements.

The most representative solution given by Internet Engineering Task Force (IETF) is path validation through replacing BGP with BGPsec {{RFC8205}}. Yet BGPsec is not incrementally deployable. It tightly couples the path authentication with the BGP path construction itself, where an AS is required to iteratively verify the signatures of each prior hop before extending the authentication chain with its own approval. As a result, a single legacy AS can terminate the authentication chain, preventing the downstream ASes from reinstating the authentication process. In addition, the performance hit introduced by BGPsec is significant because it has to authenticate the entire path even if only part of the hops are changed. Meanwhile, although path authorization is native to path-aware Internet architecture, none of these protocols are fully compatible with BGP. This implies that unless the current Internet routing system experiences a fundamental paradigm shift towards path-aware routing, enforcing path authorization in inter-domain routing is challenging.

Forwarding Commitment (FC) is a signed object that binds the IP prefix with AS and its next hops, eventually, it could compose and help to validate the path of BGP-UPDATE propagation. However, inserting the FC into BGP-UPDATE messages will introduce performance hits such as BGPsec.

This document describes a way to synchronize FC across all on-path ASes without AS_PATH leakage, on-path ASes can then verify the FCs and filter the potential malicious BGP routes, which are generated based on previously received BGP-UPDATE messages. To ensure that the FC-synchronization mechanism can be incrementally deployed, this document defines:

1.  The specification of Forwarding Commitment (FC).
2.  The way to synchronize Forwarding Commitment (FC) across on-path ASes when next-hop AS has deployed the mechanism.
3.  The way to synchronize Forwarding Commitment (FC) across on-path ASes when next-hop AS fails to deploy the mechanism.
4.  The way to revoke invalid Forwarding Commitment (FC) and synchronize the revocation message across all ASes, avoiding these invalid FCs being used to announce malicious BGP-UPDATE messages.

## Requirements Language

{::boilerplate bcp14-tagged}

# Assumptions

We assume that ASes deploying the described mechanism, i.e., upgraded ASes, have access to an Internet-scale trust base, namely Resource Public Key Infrastructure (RPKI). RPKI's repository stores authoritative information about the mapping between AS numbers and their owned IP prefixes, as well as ASes' public keys.

We assume a group management mechanism exists, and all upgraded ASes are divided into multiple groups. For a group G with N ASes, there are less than N/2 malicious Ases, and other Ases are honest. Honest AS is who strictly follows the protocol to complete FC synchronization. Within any two groups, e.g., group G_1 and group G_2, any AS in G_1 maps at least one AS in G_2, and there is at least one honest AS in G_1 maps to another honest AS in G_2.

We assume a dynamic member management mechanism exists. An upgraded AS can join or depart a group dynamically. This document will not introduce such a mechanism, thus only describing the synchronization mechanism in a static environment, i.e., the group members and mapping relationship are fixed.

# The specification of FC

Suppose that AS C receives a BGP UPDATE and the path shows that P:S <- A <- C. If AS C prefers to further advertise this path to its neighbor AS D, AS C computes FC{C,D,P} to authenticate the preference on the C <- D hop as follows: FC{C,D,P} = {H(C,D,P,Ver)_{SigC}, C, D, Ver}, where H is a (public) secure one-way hash function, C and D are endpoints of this hop, and Ver is the version number required in synchronizing missed FC messages. SigC is the signature signed by the private key of AS C.

# Synchronize FC across on-path ASes

An FC-TAG attribute is added to the BGP-UPDATE message, the upgraded AS should additionally set the Attr.VERSION and Attr.VALUE when receiving a BGP-UPDATE message. Then, the AS forward the BGP-UPDATE message and generate FC with Attr.VERSION.

~~~~~~
+---------------------------------------------+
|     +---------------+     +------------+    |
|     |  Attr.VERSION |     | Attr.VALUE |    |
|     +---------------+     +------------+    |
|     |    Version    |     |   Random   |    |
|     +---------------+     +------------+    |
+---------------------------------------------+
~~~~~~
{: #fig1: title="The structure of the FC-TAG attribute"}

## Next-hop AS has deployed the mechanism

When next-hop AS is upgraded, the AS only needs to set the Attr.VERSION of FC-TAG attribute in BGP-UPDATE message, and sets Attr.VALUE to 0. Then AS uses the version number to construct the FC and forward it to the next-hop AS.


## Next-hop AS fails to deploy the mechanism

When next-hop AS fails to deploy the mechanism, the AS should also generate a random number and set it to the Attr.VALUE of FC-TAG attribute in BGP-UPDATE message. The AS who gets the random number from the BGP-UPDATE message can require FC from the AS.


# Revoke the invalid FC

To avoid an invalid FC being used to announce a BGP-UPDATE message, AS should revoke the FC when revoking the related BGP path, and the revocation message should be synchronized to all ASes.

## Notations

- Revocation Version View (RVV). Suppose there are K ASes in the current ecosystem. Each of them maintains a K-dimensional vector named Revocation Version View, where the i-th element records the version number of the latest revocation message received from AS i. Specifically, the RVV of AS s is as follows: RVV_s={ASN_i,Ver_i},  where i is included in (1,K), ASN_i is the AS Number of AS i, and Ver_i is the latest version number from AS i.


- Intra-group Revocation Version View (IG-RVV) is a subset of RVV, and only includes the element records related to the ASes in the same group. Specifically, for an AS s in group k, i.e., the IG-RVV of AS s is as follows: IG-RVV_s={ASN_i,Ver_i}, where ASN_i is included in G_k, G_k is a set that includes all ASN of AS in group k.


- Incremental Revocation Version View (IRVV) is an incremental format of RVV, tagged with a version number v. IRVV_j^v={v,{ASN_i,Ver_i^v}}, where v is the version number of IRVV and Ver_i^v is greater than Ver_{i}^{v-1}.

## Broadcast the revocation message

When AS wants to revocate an invalid FC, it could use this message: M_{revocation}={Version,Revoke-version}_{Sig}, where Version is the version number of the revocation message, Revoke-version is the version number of revoked FC, and Sig is the signature of AS who construct the revocation message.

## Synchronize the missed messages

Considering that some ASes may miss the revocation message during the message broadcast phase, a lightweight version-based Consistency Check Protocol (CCP) is used to synchronize the missed revocation messages. The CCP includes two parts: intra-group CPP and inter-group CPP.

In each period v, ASes will broadcast their local Incremental RVV, i.e., IRVV^v to all other members in the same group. Upon receiving the IRVV_s^v from AS s, an AS j compares with its local RVV. If the Ver_i^v of AS i in IRVV_s^v is greater than the Ver_i^v in local RVV, then AS j misses at least one new binding message from AS i. In this case, besides simply adopting a newer version, AS j can proactively request missed binding messages from AS s. The rationale is that the missing of prior updates is an indicator of weak or unstable network connectivities for AS j, and therefore the proactive requests are helpful.

At the start of period v+1, ASes will broadcast their IG-RVV to mapped ASes in other groups. When receiving an IG-RVV, ASes will compare it with a local RVV, and require missed binding messages from the sender of IG-RVV.

# Security Considerations

## Protect Policy Preference

According to the synchronization process of FC, only ASes who receive the BGP-UPDATE message will get the FC. Therefore, the synchronization process will not leak more AS_PATH information than BGP.

## Mitigation of Route Hijacking Attack

To mitigate route hijacking attacks, an AS peer can choose a more reliable AS for routing and forwarding. There are four types of paths:

   - TRUSTED PATH: Every hop in the AS_PATH has deployed the mechanism, and all related FCs are received and verified.
   - PARTIALLY TRUSTED PATH: At least one hop fails to deploy the mechanism.
   - LEGACY PATH: All ASes in the AS_PATH fail to deploy the mechanism.
   - SUSPICIOUS PATH: Misses a corresponding FC from at least one upgraded hop on the AS_PATH.

For LEGACY PATHs, we cannot directly determine whether they are SUSPICIOUS PATHs because the ASes have not deployed FC-BGP. Therefore, the security of a route announcement is directly proportional to the number of ASes that have deployed FC-BGP.

For non-LEGACY PATHs, they will be set to SUSPICIOUS directly because ASes accept BGP-UPDATE messages first and then require and verify a series of related FCs (FC-LIST). Therefore, the route related to a SUSPICIOUS PATH can only exist for a fixed time, e.g., 10 seconds. Only when the AS receives and verifies the FC-LIST from the last upgraded AS will the related path be set to TRUSTED or PARTIALLY TRUSTED.

In the route hijacking attack, the attacker cannot generate a correct corresponding FC because the FC lacks a signature and can be successfully detected by subsequent receivers. In this case, the path will be set to SUSPICIOUS and only exist for a fixed time, i.e., the effect of a route hijacking attack can only exist for a fixed time.

Overall, to favor secure routing, the ASes joining the FC-BGP ecosystem SHOULD design routing policies in their local routing policy engines, preferring TRUSTED PATHs over PARTIALLY TRUSTED PATHs over LEGACY PATHs.


# IANA Considerations

This document requires the following IANA actions:

This document requests the assignment of a new attribute code described in Section 1 in the "FC-BGP Path Attributes" registry. The attribute code for this new path attribute "FC-TAG" to provide consistency between the control and data planes. This document is the reference for the new attribute.


#  Contributors

xxx


# Acknowledgments

TODO acknowledge.
