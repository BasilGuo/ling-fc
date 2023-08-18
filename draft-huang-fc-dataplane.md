---
###
# This is for Hanlin Huang
###
title: "Forwarding Commitment for Data Plane Verification"
abbrev: "FCDP Verification"
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

Based on the draft IDxx, Forwarding Commitment (FC) can provide the ability to verify BGP routing updates at the control plane. Meanwhile, the flexibility of FC further enables efficient forwarding validation at the data plane. Because the FCs are self-proving, an AS can conceptually construct a certified AS-path using a list of consecutive per-hop FCs and then bind its network traffic to the path. Therefore, by advertising the binding message globally, both on-path and off-path ASes are aware of the desired forwarding paths. They can collaboratively discard the unwanted traffic that takes an unauthorized path.

--- middle

# Introduction

Based on the FC list received at the control plane, the source AS, which is the destination of the BGP-UPDATE message and the start point of the traffic, notifies the on-path ASes, which are on the propagation path of the BGP-UPDATE message, with the established traffic and forwarding path mappings through binding messages before forwarding traffic.

With binding messages, on-path nodes establish corresponding filtering tables for traffic and forwarding paths, completing the verification of the forwarding path while filtering out illegal traffic. In addition, the source AS MUST broadcast binding messages to off-path ASes. These off-path ASes SHOULD collaboratively filter out traffic that is not allowed according to the binding messages and this traffic may be performed with source spoofing or path manipulation. Through the collaboration of on-path and off-path nodes, the verification of traffic forwarding paths is completed together.

The key advantage of this off-path collaborative filtering is that it is completely location-independent, i.e., the off-path filtering is effective no matter how far is it between the filtering AS and the actual on-path ASes, and no matter how many legacy regions are between them.

This document primarily explains how a BGP speaker that supports FC-BGP can utilize the FC list to finish path verification. It mainly elaborates on the process of generating binding messages, establishing filtering tables based on the FC list, and verifying the forwarding path.

Similar to forming FC in the control plane, we still assume the existence of an RPKI repository {{RFC6480}} that contains ownership relationships between AS number and valid IP prefixes. The public key of each AS can be fetched from the repository.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Data Plane Validation

## Overview

Although FC was initially designed to verify BGP announcement in the control plane, the key insight into the applicability of FCs in the data plane is that the correctness of a forwarding path can be verified based on the carried FC list. Before forwarding traffic, the source AS needs to send binding messages containing the FC list to on-path ASes. The nodes deployed FC-BGP will establish filtering tables based on the FC list and filter out illegal traffic. In addition, the source also broadcasts the source and destination prefixes of traffic verified by FC to off-path nodes to facilitate their collaborative verification.


As shown in {{figure1}}, the source AS C needs to forward traffic to AS A. AS C sends binding messages containing the FC list to on-path nodes AS A and AS B, and A and B establish filtering tables. AS C sends a binding message that does not carry the FC list to the off-path node AS E. Thus AS E SHOULD drop received traffic from AS C to AS A. But if AS E is in another AS-path which corresponds traffic from AS C to AS A, e.g. to do load balance or as a backup link, AS E MUST keep forwarding this traffic.

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

As mentioned in the previous section, both on-path and off-path nodes receive binding messages as credentials for verification and filtering, with the difference being the presence of an FC list. Based on the FC information received at the control plane, we construct the binding message formats as shown in Formulas (1) and (2), respectively.

    Monpath = {Psrc,Pdst,FClist, Ver,Versub}_Sigsrc --- (1)

    Moffpath = {Psrc,Pdst,Ver,Versub}_Sigsrc --- (2)

TODO AS A PROTOCOL SPECIFICATION, IT SHOULD USE THE ASCII FORMAT TO DESCRIBE BINDING MESSAGES AND FORWARDING COMMITMENT INSTEAD OF FORMULA. And with these formats, one can implement FC and BM mechanisms independently. The following is a sample and many many details are ignored. Need to be reviewed and discussed.


~~~~~~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     AFI       |  Prefix Size  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Length      |                 Prefix                        ~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~
{: #fig-prefix-fmt title="Prefix Format"}

AFI:
:  8-bit field. Address Family Identifier. The value is 1 for IPv4 and 2 for IPv6.

Prefix Size:
:  8-bit field. It indicates the number of prefixes with the following Length and Prefix format.

Length:
:  8-bit field. The length of the IP prefix with Prefix.

Prefix:
:  variable length field. It defines the IP prefix owned by one AS.


~~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Version    |      Flags    |  FC Version   | FC Subversion |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          FC List Size         |            Reserved           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Source AS Number                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Destination AS Number                    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             FC List                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~                          Source Prefix                        ~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~                        Destination Prefix                     ~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~
{: #fig-bm-fmt title="Binding Message Format"}


### Version

The Version is an 8-bit field. It defines the version of the Binding Message Format. Here the version is 0.

### Flags

The Flags is an 8-bit field. The first bit of Flags indicates whether this is an on-path Binding Message or an off-path Binding Message. If it is 0, it means this is an on-path Binding Message and it MUST carry the FC List field. Otherwise, it means this is an off-path Binding Message and it MUST NOT carry the FC List field and the FC List size MUST be 0.

Other bits of Flags are not defined and MUST be 0.

### FC Version

TODO The FC Version is an 8-bit field. It defines the major version of this Binding Message.

### FC Subversion

TODO The FC Subversion is an 8-bit field. It defines the minor version of this Binding Message.

### FC List Size

The number of FC in the FC List. It occupies 2-octet.

### Reserved

It occupies 2-octet and MUST be 0.

### Source AS Number and Destination AS Number

This indicates the direction of traffic. They are all 4-octet fields.

### FC List

This is a list of ASes that are all on-path nodes that deployed FCBGP and chosen to forward the traffic from Source AS to Destination AS. So it is a variable field and the length is dependent on FC List Size.

### Source Prefix and Destination Prefix

They are both variable fields. These are the prefixes belonging to source AS and destination AS separately. They are using the format defined in {{fig-prefix-fmt}}. They both include source and destination prefixes to distinguish different flows. In the on-path binding message, the FC list received at the control plane is included, representing the valid path for the current flow. In addition, both binding messages also include version number information to keep binding messages consistent, referring to draft IDxx. Each binding message is signed using the private key of the source AS and decrypted by the receiving node.

## Filtering Table

For an AS that has deployed FC-BGP, when it receives a binding message and completes the verification, it will establish a filtering table. For example, if AS A receives the following binding message:

            Monpath = {Psrc,Pdst,F{A,B,P}--F{B,C,P}, Ver,Versub}_Sigsrc --- (3)

AS A will bind the filtering message at the inbound of the link and construct the filtering table in the following form, specifying traffic with the source belonging to Psrc and the destination belonging to Pdst to enter the A-B link.

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

TODO YOU SHOULD EXPLAIN THE FORMAT OF <src:Pd,dst:P> and {F{C,D,P};F{B,C,P};F{A,B,P}}.

For the purpose of illustration, we provide a comprehensive example to explain the entire process described above. Suppose that the source AS D needs to forward traffic <src:Pd,dst:P> to AS A, it implies to all on-path and upgraded ASes (i.e., A and C) that the network traffic <src:Pd,dst:P> (Pd represents the prefix owned by D) shall take the forwarding path D->C->B->A. To make this implicated forwarding path explicit, AS D can send the FC list (i.e., {F{C,D,P};F{B,C,P};F{A,B,P}}) to AS A and C. Afterwards, AS C confirms that the traffic <src:Pd,dst:P> must inbound using the hop D->C, and AS A confirms the traffic must inbound via the hop B->A.

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

TODO If you need a TCP port to transfer/receive these binding messages, you would need IANA to assign one.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
