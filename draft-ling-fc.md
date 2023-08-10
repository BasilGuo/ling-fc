---

title: "A Profile for Synchronizing Forwarding Commitments (FCs) without AS_Path Leakeage"
abbrev: "Forwarding Commitments"
category: std

docname: draft-guo-fc-so-latest
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
  github: "BasilGuo/fc-signed-object"
  latest: "https://BasilGuo.github.io/fc-signed-object/draft-guo-fc-so.html"

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

informative:
    X.680:
      title: "Information technology -- Abstract Syntax Notation One (ASN.1): Specification of basic notation"
      target: "https://itu.int/rec/T-REC-X.680-202102-I/en"
      date: Feb. 2021
      author:
        - ins: ITU-T
      seriesinfo: "Recommendation ITU-T X.680"
    X.690:
      title: "Information technology - ASN.1 encoding rules: Specification of Basic Encoding Rules (BER), Canonical Encoding Rules (CER) and Distinguished Encoding Rules (DER)"
      target: "[https://itu.int/rec/T-REC-X.680-202102-I/en](https://www.itu.int/rec/T-REC-X.690-202102-I/en)"
      date: Feb. 2021
      author:
        - ins: ITU-T
      seriesinfo: "Recommendation ITU-T X.690"


--- abstract

This document defines a standard profile for synchronizing Forwarding Commitment (FC) across on-path ASs without leaking more AS_path information than BGP. A FC is a digitally signed object that provides a means of verifying that an IP address prefix is announced from `AS a` to `AS b`. On-path ASs can then verify the FCs and filter the potential malicious BGP routes, which are generated based on previously received BGP-UPDATE messages.


--- middle

# Introduction

The fundamental cause of the path manipulation attacks in Internet inter-domain routing is that the de facto Border Gateway Protocol (BGP) {{RFC4271}} does not have built-in mechanisms to authenticate routing announcements. As a result, an adversary can announce virtually arbitrary paths to a prefix while the network cannot effectively verify the authenticity of the route  announcements. The most representative solutions given by Internet Engineering Task Force (IETF) are path validation through replacing BGP with BGPsec[<sup>1</sup>](#BGPsec). Yet BGPsec is not incrementally deployable. It tightly couples the path authentication with the BGP path construction itself, where an AS is required to iteratively verify the signatures of each prior hop before extending the authentication chain with its own approval. As a result, a single legacy AS can terminate the authentication chain, preventing the downstream ASes from reinstating the authentication process. In addition, the performance hit introduced by BGPsec is significant because it has to authenticate the entire path even if only part of the hops is changed. Meanwhile, although path authorization~\cite{kim2014lightweight,legner2020epic} is native to path-aware Internet architecture, none of these protocols are fully compatible with BGP. This implies that unless the current Internet routing system would experience a fundamental paradigm shift towards path-aware routing, it is challenging to enforce path authorization in the inter-domain routing. 

Forwarding Commitment (FC) is a signed object that binds IP prefix with AS and its next hops, eventually it could compose and help to validate the path of BGP-UPDATE propagation. However, inserting the FC into BGP-UPDATE messages will introduce performance hit such as BGPsec. This document describe a way to synchronize FC across all on-path ASs without AS_Path leakeage, on-path ASs can then verify the FCs and filter the potential malicious BGP routes, which are generated based on previously received BGP-UPDATE message. To ensure that the FC-synchronizition mechanism can be incrementally deployed, this document defines:

1.  The specification of Forwarding Commitment (FC).
2.  The way to synchronize Forwarding Commitment (FC) across on-path ASs when next-hop AS has deployed the mechanism.
3.  The way to synchronize Forwarding Commitment (FC) across on-path ASs when next-hop AS fail to deploy the mechanism.
4.  The way to revoke invalid Forwarding Commitment (FC) and synchronize the revocation message across all ASs, avoiding these invalid FCs be used to announce malicious BGP-UPDATE message.

## Requirements Language

{::boilerplate bcp14-tagged}

# Assumptions

We assume that ASes deploying the described mechanism, i.e., upgraded ASes, have access to an Internet-scale trust base, namely Resource Public Key Infrastructure (RPKI), that stores authoritative information about the mapping between AS numbers and their owned IP prefixes, as well as ASes' public keys. 

We assume a group management mechanism exists, and all upgraded ASes are divided into multiple groups. For a group $G$ with $N$ ASes, there are less than $N/2$ malicious Ases exist, and other Ases are honest, i.e., the AS who strictly follow the protocol to complete FC synchronization. For any two groups (e.g., group $G_1$ and group $G_2$), any AS in $G_1$ maps at least one AS in $G_2$, and there is at least one honest AS in $G_1$ maps to another honest AS in $G_2$. 

We assume a dynamic member management mechanism exists, an upgraded AS can join or depart a group dynamically. This document will not introduce such a mechanism, thus only describe the synchronization mechanism in a static environment, i.e., the group members and mapping relationship are fixed.

# The specification of FC

Suppose that AS C receives a BGP UPDATE P:S <-- A <-- C, if AS C prefers to further advertise this path to its neighbor AS D, AS C computes F{C,D,P} to authenticate the preference on the C <-- D hop as follows:

$$ FC\{C,D,P\} = \{H(C,D,P,Ver)_{SigC}, C, D, Ver \} $$

where $H$ is a (public) secure one-way hash function, $C$ and $D$ are endpoints of this hop, $\textsf{Ver}$ is the version number required in synchronizing missed FC message. $SigC$ is the signature using privatekey of AS $C$. 

# Synchronize FC across on-path ASs

A FC-Tag attribute is added to the BGP_UPDATE message, the upgraded AS should additional set the Attr.VERSION and Attr.VALUE when receiving a BGP_UPDATE message. Then, the AS forward the BGP_UPDATE message and generate FC with Attr.VERSION.

~~~~~~

+---------------------------------------------+
|     +---------------+     +------------+    |
|     |  Attr.VERSION |     | Attr.VALUE |    |
|     +---------------+     +------------+    |
|     |    Version    |     |   Random   |    |       
|     +---------------+     +------------+    |
+---------------------------------------------+

Figure 1: The structure of the FC-Tag attribute.
~~~~~~

## Next-hop AS has deployed the mechanism

When next-hop AS is upgraded, the AS only need to set the Attr.VERSION of FC-Tag attribute in BGP-UPDATE message, and set Attr.VAlUE to 0. Then AS uses the version number to construct the FC and forward it to next-hop AS.


## Next-hop AS fail to deploy the mechanism

When next-hop AS fail to deploy the mechanism, the AS should also generate a random number and set it to the Attr.VAlUE of FC-Tag attribute in BGP-UPDATE message. The AS who get the random number from BGP-UPDATE message can require FC from the AS.

<!-- ~~~~~~
+------+     +------+     +------+     +------+
| AS A | <-- | AS B | <-- | AS C | <-- | AS D |
+------+     +------+     +------+     +------+
Figure 2: The structure of the FC-Tag attribute.
~~~~~~ -->


# Revoke the invalid FC 
To avoid an invalid FC be used to announce a BGP_UPDATE message, AS should revoke the FC when revoke the related BGP path, and the revocation message should be synchronized to all ASes

## Notations

- Revocation Version View. Suppose there are K ASes in the current ecosystem. Each of them maintains a K-dimensional vector named Revocation Version View, i.e. RVV, where the i-th element records the version number of the latest revocation message received from AS i. Specifically, the RVV of AS s is as follow:
$$RVV_s=\{\{ASN_i,Ver_i \}  | i∈(1,K)\}$$ where $ASN_i$ is the AS Number of AS i, and $Ver_i$ is latest version number from AS i


- Intra-group Revocation Version View (IG-BVV) is a subset of BVV, and only include the element records related to the ASes in the same group. Specifically, for an AS s in group k, i.e., the IG-RVV of AS s is as follow:
$$IG-RVV_s=\{\{ASN_i,Ver_i \}  | ASN_i  ∈ G_k  \}$$ where $G_k$ is a set which include all ASN of AS in group k.


- Incremental Revocation Version View (IRVV) is a incremental format of RVV, is tagged with a version number v. 
$$IRVV_j^v=\{ v,\{ASN_i,Ver_i^v \}  | Ver_i^v>Ver_{i}^{v-1} \} $$ where v is the version number of IRVV.


## Broadcast the revocation message
When AS want to revocate a invalid FC, 

$$M_{revocation}=\{Version,Revote-version\}_{Sig}$$ where $Version$ is the version number of the revocation message, $Revote-version$ is the version number of revoted FC, and $Sig$ is the signature of AS who construct the revocation message.

## Synchronize the missed messages
Considering that some ASes may miss the revocation message during the message broadcast phase, a lightweight version-based Consistency Check Protocol (CCP) is used to synchronize the missed revocation messages. The CCP includes two parts: intra-group CPP and inter-group CPP.

In each period v, ASes will broadcasting their local Incremental BVV, i.e., $IRVV^v$ to all other members in the same group. Upon receiving the  $IRVV_s^v$ from AS s, an AS j compares with its local BVV. If the $Ver_i^v$ of AS i in $IRVV_s^v$ is greater than $Ver_i^v$ in local $RVV$, then AS j misses at least one new binding message from AS i. In this case, besides simply adopting a newer version, AS j can proactively request missed binding messages from AS s. The rationale is that the missing of prior updates is an indicator of weak or unstable network connectivities for AS j, and therefore the proactive requests are helpful.

At the start of period v+1, ASes will broadcast their IG-BVV to mapped ASes in other groups. When receiving an IG-BVV, ASes will compare it with local BVV, and require missed binding messages from the sender of IG-BVV. 

# Security Considerations

## Protect Policy Preference

According the synchronization process of FC, only ASes who receive the BGP_UPDATE message will get the FC. Therefore, the synchronization process will not leak more AS_path information than BGP.

## Mitigation of Route Hijacking Attack

To mitigate route hijacking attacks, an AS peer can choose a more reliable AS for routing and forwarding. There are four types of paths:

   - Trusted paths: Every hop in the AS-path has deploy the mechanism, and all related FCs are received and verified.
   - Partially trusted paths: At least one hop fail to deploy the mechanism.
   - Legacy paths: All ASes in the AS_Path fail to deploy the mechanism.
   - Suspicious paths: Misses a corresponding FC from at least one upgraded hop on the ASpath.

   For legacy paths, we cannot directly determine whether they are suspicious because the ASes that have not deployed FC-BGP. Therefore, the security of a route announcement is directly proportional to the number of ASes that have deployed FC-BGP.

   For non-legacy paths, they will be set to suspicious directly because ASes accept BGP_UPDATE messages first and then require and verify the FC list. Therefore, the route related to a suspicious path can only exist for a fixed time, e.g., 10 seconds. Only when the AS receive and verify the FC list from the last upgraded AS will the related path be set to trusted or partially trusted. 
   
   In the route hijacking attack, the attacker cannot generate a correct corresponding FC because the FC lacks a signature and can be successfully detected by subsequent receivers. In this case, the path will be set to suspicious and only exist for a fixed time, i.e., the effect of route hijacking attack can only exist for a fixed time. 

   Overall, to favor secure routing, the ASes joining the FC-BGP ecosystem shall design routing policies in their local routing policy engines, preferring trusted paths over partially trusted paths over legacy paths.


# IANA Considerations

This document requires the following IANA actions:

This document requests the assignment of a new attribute code described in Section 1 in the "FC-BGP Path Attributes" registry. The attribute code for this new path attribute "FC-TAG" to provide consistency between the control and data planes. This document is the reference for the new attribute.


#  Contributors

xxx


# Acknowledgments
{:numbered="false"}

TODO acknowledge.
