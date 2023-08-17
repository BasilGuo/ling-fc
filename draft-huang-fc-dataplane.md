---
###
This is for Hanlin Huang
###
title: "FC for dataplane verification"
abbrev: "TODO - Abbreviation"
category: info

docname: draft-huang-fc-dataplane-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: AREA
workgroup: WG Working Group
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: WG
  type: Working Group
  mail: WG@example.com
  arch: https://example.com/WG
  github: USER/REPO
  latest: https://example.com/LATEST

author:
 -
    fullname: Hanlin Huang
    org: Tsinghua University
    city: Beijing
    country: China
    email: hhl21@mails.tsinghua.edu.cn
 -
    fullname: Ke Xu
    org: Tsinghua University
    city: Beijing
    country: China
    email: xuke@tsinghua.edu.cn
 -
    fullname: Xiaoliang Wang
    org: Tsinghua University
    city: Beijing
    country: China
    email: wangxiaoliang0623@foxmail.com 

normative:
  RFC4271:
  RFC6483:
  RFC6811:
  RFC6480:
informative:


--- abstract

Based on the draft IDxx, Forwarding Commitment (FC) can provide the ability to verify BGP routing updates at the control plane. Meanwhile, the flexibility of FC further enables efficient forwarding validation at the data plane. Because the FCs are self-proving, an AS can conceptually construct a certified AS-path using a list of consecutive per-hop FCs, and then binds its network traffic to the path. Therefore, by advertising the binding message globally, both on-path and off-path ASes are aware of the desired forwarding paths so that they can collaboratively discard the unwanted traffic that takes an unauthorized path.

--- middle

# Introduction

Based on the FC list received at the control plane, the source AS notifies the on-path ASes of the established traffic and forwarding path mappings through binding message before forwarding traffic. Based on the binding messages, the on-path nodes establish corresponding filtering tables for traffic and forwarding paths, completing the verification of the forwarding path while filtering out illegal traffic. In addition, the source AS can also broadcast binding messages to off-path ASes, which filter out traffic specified in the binding messages due to source spoofing or path manipulation. Through the collaboration of on-path and off-path nodes, the verification of traffic forwarding paths is completed together.

The key advantage of this off-path collaborative filtering is that it is completely location-independent, i.e., the off-path filtering is effective no matter how far is it between the filtering AS and the actual on-path ASes, and no matter how many legacy regions are between them.

This document primarily explains how a BGP speaker that supports FC-BGP can utilize the FC list to finish path verification. It mainly elaborates on the process of generating binding messages, establishing filtering tables based on the FC list, and verifying the forwarding path.

Similar to forming FC in the control plane, we still assume the existence of an RPKI repository{{RFC6480}} that contains ownership relationships between AS numbers and valid prefixes. The public key of each AS can be fetched from the repository.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Data Plane Validation

## Overview
Although FC was initially designed to verify BGP announcement the control plane, the key insight into the applicability of FCs in data plane is that the correctness of a forwarding path can be verified based on the carried FC list. Before forwarding traffic, the source AS needs to send binding messages containing the FC list to on-path ASes. The nodes deployed FC-BGP will establish filtering tables based on the FC list and filter out illegal traffic. In addition, the source also broadcasts the source and destination prefixes of traffic verified by FC to off-path nodes to facilitate their collaborative verification. As shown in the figure 1, the source AS C needs to forward traffic to AS A. AS C sends binding messages containing the FC list to on-path nodes AS A and AS B, and A and B establish filtering tables. For the off-path node AS E, it receives a binding message that does not carry FC and filters out traffic with the source being AS C and the destination being AS A.

~~~
                    off-binding
                AS E----<-----
                              \
                               \
AS A-----------AS B-----------AS C
  ^             ^              |
  |  on-binding |  on-binding  v   
  ------<---------------<------+
~~~
{: #figure1 title="Overview"}

## Binding Message

As mentioned in the previous section, both on-path and off-path nodes receive binding messages as credentials for verification and filtering, with the difference being the presence of an FC list. Based on the FC information received by the control plane, we construct the binding message formats shown in Formulas 1 and 2, respectively. 

            Monpath = {Psrc,Pdst,FClist, Ver,Versub}_Sigsrc --- (1)

            Moffpath = {Psrc,Pdst,Ver,Versub}_Sigsrc --- (2)

They both include source and destination prefixes to distinguish different flows. In the on-path binding message, the FC list received by the control plane is included, representing the valid path for the current flow. In addition, both binding messages also include version number information to keep bingding messages consist, refering to draft IDxx. Each binding message is signed using the private key of the source AS and decrypted by the receiving node.

## Filtering Table

For an AS that has deployed FC-BGP, when it receives a binding message and completes the verification, it will establish a filtering table. For example, if AS A receives the following binding message:

            Monpath = {Psrc,Pdst,F{A,B,P}--F{B,C,P}, Ver,Versub}_Sigsrc --- (3)

AS A will bind the filtering message at the inbound of the link and construct the filtering table in the following form, specifying traffic with the source being Psrc and the destination being Pdst to enter the A-B link.

~~~
    -------------------------            -------------------------
    | traffic   |   port     |           |  traffic  |    port    |
    -------------------------            -------------------------
    | src->dst  | port1(A-B) |           | src->dst  | port1(B-C) |
    -------------------------            -------------------------
     filtering table for AS A             filtering table for AS B
~~~
{: #figure2 title="On-path AS filtering table"}

Please note that in a filter table, the port indicated is the one that received the binding message, assuming that the FC propagation path and the traffic forwarding path are symmetric.

For an off-path AS, it will directly discard the traffic, regardless of which entrance it comes from.

~~~
-----------------------
| traffic   | action  |
-----------------------
| src->dst  |   drop  |
-----------------------
~~~
{: #figure3 title="Off-path AS filtering table"}

## On-path And Off-path Validation

For the purpose of illustration, we provide a comprehensive example to explain the entire process described above. Suppose that the source AS D needs to forward traffic <src:Pd,dst:P> to AS A, it implies to all on-path and upgraded ASes (i.e., A and C) that the network traffic <src:Pd,dst:P> (Pd represents the prefix owned by D) shall take the forwarding path D->C->B->A. To make this implicated forwarding path explicit, AS D can send the FC list (i.e., {F{C,D,P};F{B,C,P};F{A,B,P}}) to AS A and C. Afterwards, AS C confirms that the traffic <src:Pd,dst:P> must inbound using the hop D->C, and AS A confirms the traffic must inbound via the the hop B->A.

The forward binding can be extended to off-path ASes to enable large-scale traffic filtering. In particular, in Figure 4, AS D can broadcast the binding relationship for traffic <src:Pd,dst:P> to off-path ASes. As a result, all off-path ASes only learn that they should not expect to serve traffic <src:Pd,dst:P>, instead of learning the actual forwarding path for the traffic. off-path ASes may receive traffic <src:Pd,dst:P> due to either source spoofing or path manipulation. In both cases, they can discard these flows based on the binding message received from AS D.

~~~
                    ----------------------------
                    |  Bing Message Broadcast  |
                    |         Moffpath         |
                    ----------------------------
                    | Psrc:20.0.0.0/24         |
                    | Pdst:10.0.0.0/24         |
                    | Ver:1.0                  |
                    | Versub:0                 |
                    ----------------------------
                    off-path ASes  <------<-----+
               collaborative defense             \
                          ^                       |
                          |                       |
                          v                       ^
                 +-------------------+            |
10.0.0./24       |        Undeployed |        20.0.0.0/24     
AS A  -----------|--AS B     Zone  --|--AS C-----AS D
|                |                   |   |        |
^                +-------------------+   ^        |
|                                        |        |
+----------<---------------<---------------<------+
        ----------------------------
        |  Bing Message Broadcast  |
        |         Monpath          |
        ----------------------------
        | Psrc:20.0.0.0/24         |
        | Pdst:10.0.0.0/24         |
        | FClist:F{A,B,P}-F{C,D,P} |
        | Ver:1.0                  |
        | Versub:0                 |
        ----------------------------
~~~
{: #figure4 title="A comprehensive example for path validation"}

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
