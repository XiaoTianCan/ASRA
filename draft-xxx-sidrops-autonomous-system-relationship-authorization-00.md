---
title: Autonomous System Relationship Authorization (ASRA) for AS_PATH Verification
abbrev: ASRA for AS_PATH Verification
docname: draft-xxx-sidrops-autonomous-system-relationship-authorization-00
obsoletes:
updates:
date:
category: std
submissionType: IETF

ipr: trust200902
area: ops
workgroup: sidrops
keyword: Internet-Draft

author:
 -
  ins: X. XX
  name: X XX
  organization: XXX
  email: xxx@xxx.com
  city: XX
  country: XX

normative:
  I-D.ietf-sidrops-aspa-verification:
  I-D.ietf-sidrops-aspa-profile:

informative:
  I-D.ietf-sidrops-8210bis:
  RFC8205:
  RFC9234:
  as-rela-1:
    title: Stable and Practical AS Relationship Inference with ProbLink
    author: 
    org: NSDI
    date: 2019-02
    target: https://www.usenix.org/system/files/nsdi19-jin.pdf
  as-rela-2:
    title: Inferring Internet AS Relationships Based on BGP Routing Policies
    author: 
    org: University College London
    date: 2011-01
    target: 
  privacy-analysis:
    title: Jumpstarting BGP Security with Path-End Validation
    author: 
    org: SIGCOMM
    date: 2016-08
    target: https://dl.acm.org/doi/pdf/10.1145/2934872.2934883

...

--- abstract
Autonomous System Provider Authorization (ASPA) is very helpful in detecting and mitigating route leaks and some route hijacks. ASPA record is authorized in one direction and only focuses on including all provider ASes while ignoring other neighboring ASes. These features of ASPA record may result in the failure of detecting invalid AS_PATHs. 

This document analyzes the limitations of ASPA-based AS_PATH verification, and puts forward the idea of Autonomous System Relationship Authorization (ASRA) to overcome them. An AS can optionally register ASRA records that include not only provider ASes, but also other neighboring ASes with different relationships (e.g., customer ASes). ASRA can efficiently enhance AS_PATH verification compared to ASPA, and ASRA is compatible to ASPA. 


--- middle

# Introduction
Autonomous System Provider Authorization (ASPA) is a technique for protecting AS_PATHs in BGP updates {{I-D.ietf-sidrops-aspa-verification}}{{I-D.ietf-sidrops-aspa-profile}}. Each AS can register ASPA records (also ASPA objects) in the RPKI to states all its provider ASes. An AS can obtain other ASes' ASPA records and conduct AS_PATH verification based the records. ASPA-based AS_PATH verification can detect and mitigate route leaks violating the valley-free principle and route hijacks such as forged-origin or forged-path-segment attacks. 

ASPA can significantly simplify and enhance AS_PATH verification and is promising to be widely deployed. Despite of the advantages of ASPA, there are also some limitations. ASPA record is authorized in one direction, e.g., an AS can unilaterally claim that another AS is its provider without the consent of other ASes. Besides, an ASPA record only focuses on including all provider ASes while ignoring other neighboring ASes. These features of ASPA record may result in the failure of detecting invalid AS_PATHs in some cases. 

This document provides a gap analysis of ASPA-based AS_PATH verification. To narrow the gaps, the idea of Autonomous System Relationship Authorization (ASRA) is proposed. An AS can optionally register ASRA records that include not only provider ASes, but also other neighboring ASes with different relationships (e.g., customer ASes). ASRA allows two-direction authorization and provides more detailed relationships with neighboring ASes. Thus, ASRA can efficiently enhance AS_PATH verification compared to ASPA. 

ASRA solution is completely compatible to ASPA solution. Without ASRA records, an AS can conduct ASPA-based AS_PATH verification. If there are some ASRA available, an enhanced verification can be carried out. 

## Terminology

The usage of terms follows {{I-D.ietf-sidrops-aspa-verification}}{{I-D.ietf-sidrops-aspa-profile}}. 

## Requirements Language

{::boilerplate bcp14-tagged}

# Gap Analysis of ASPA-based AS_PATH Verification
This section describes some gaps of ASPA-based AS_PATH verification that can be narrowed to enhance verification. 

## Hard to Detect Bogus Records
An AS can unilaterally authorize a set of its provider ASes. Under the one-direction authorization, an AS may intentionally or unintentionally register bogus records that are hard to be discovered. Consider two ASes, i.e., ASx and ASy. The bogus ASPA records are registered by ASx. ASy is an AS that either has no link with ASx or has some relationship with ASy. The possible types of bogus ASPA records are listed as follows: 

- ASy is ASx's provider, but ASx DOES NOT put ASy into its provider-set in the ASPA record. This type of bogus records may lead to improper block of legitimate BGP routes. 

- ASy is ASx's customer/peer (or other non-provider roles) instead of ASx's provider, but ASx puts ASy into its provider-set in the ASPA record. The bogus records may be leveraged by attackers to launch route hijack attacks. 

- ASx and ASy are not adjacent, but ASx puts ASy into its provider-set in the ASPA record. The bogus records may be leveraged by attackers to launch route hijack attacks. 

Bogus records bring two risks:

- An AS unintentionally registers a bogus record that affects the verification accuracy while is hard to be discovered. 

- An AS maliciously registers bogus records that open a door to potential attacks. {{fig-bogus}} shows an example of AS_PATH hijack based on bogus ASPA records. AS4 lies in that the nonadjacent AS3 is its provider in the ASPA record. AS5 cannot detect the attack even when AS5 enables ASPA-based AS_PATH verification. As a result, AS5 will wrongly think its traffic to AS1 traverses AS3, while the real forwarding path to AS1 is through AS2 instead of AS3. 

~~~
              +-------+
              |  AS1  |
              +-------+
     path[1] /        \
            /(C2P)     \(C2P)
    +-------+           +-------+
    |  AS2  |           |  AS3  |
    +-------+           +-------+
           \             *
  path[2,1] \(C2P)      * (faked C2P in AS4's
             \         *   bogus ASPA)
              +-------+
              |  AS4  |ASPA{AS4, [AS2, AS3]}
              +-------+
      path[4,3,1] |(C2P)
              +-------+
              |  AS5  |
              +-------+
  * All ASes register ASPA records
  * AS4's record states a faked C2P link with AS3
  * AS5 enables ASPA-based path verification
~~~
{: #fig-bogus  title="AS_PATH hijack based on bogus ASPA records"}


## Fail to Detect AS_PATH Maliciously Shortened by a Provider
ASPA-based AS_PATH verification cannot effectively detect the AS_PATH maliciously shortened by a provider. {{fig-path-shortened}} shows an example. AS1 originates the BGP route and propagates the route to other ASes. The AS_PATH received by AS5 is path\[4,3,2,1\]. However, AS5 maliciously shortens the path by falsely claim a fake link with AS2 before AS5 propagates the route to AS6. AS6's traffic to AS1 may be hijacked by AS5 if the path\[5,2,1\] is shorter than any other AS_PATHs. 

In this case, the hijack cannot be detected even when all the ASes register the correct ASPA records and all the ASes other than AS5 enable ASPA-based AS_PATH verification. 

~~~
                  +-------+
                  |  AS4  |
                  +-------+
      path[3,2,1]/         \
                /(C2P)      \ path[4,3,2,1]
        +-------+            \
        |  AS3  |             \
        +-------+              \
  path[2,1] |             (C2P) \
      (C2P) |                    \
        +-------+ (Fake link) +-------+
        |  AS2  |*************|  AS5  |
        +-------+             +-------+
    path[1] |                     |path[5,2,1]
      (C2P) |                (C2P)|(path shoterned)
        +-------+             +-------+
        |  AS1  |             |  AS6  |
        +-------+             +-------+
~~~
{: #fig-path-shortened  title="AS_PATH maliciously shortened by a provider"}


## Cannot Distinguish Leak and Hijack for an Invalid AS_PATH
Existing ASPA verification algorithm can identify invalid AS_PATHs, but it cannot distinguish leak and hijack for an invalid AS_PATH. The main reason is that ASPA records only focus on registering all provider ASes while not indicating the adjacency/topology information. When the Hop-check(x, y) function returns "Not provider+", the algorithm does not know whether the real cause of the result is 1) ASy is ASx's customer/peer instead of provider or 2) link(x,y) is a fake link. Therefore, when the algorithm returns invalid, it is not able to indicate whether the path is caused by a fake link-based hijack or not. 

For operators, it's instructive to know the type of an invalid AS_PATH. If there exists no hijack, the invalid AS_PATH is likely to be an unintentional route leak. Otherwise, the network may be attacked by route hijacks.  


## Not Directly Applicable to Egress/IBGP Verification
Egress verification and iBGP verification are meaningful in some scenarios. Egress verification means verify the AS_PATH before sending it to the neighbor AS. It can help ASes avoid making some mistakes (e.g., route leaks). IBGP verification is to check the AS_PATH received through iBGP connections. IBGP verification helps an AS do verification on any BGP routers like non-ASBRs. 

An AS enabling existing ASPA cannot directly apply egress verification or iBGP verification. {{fig-egress-veri}} shows an example. Suppose AS2 enables ASPA-based AS_PATH verification and only registers ASPA records. Suppose ASBR2 propagates path\[2,1\] to AS3, which results in route leak. The egress verification on ASBR2 cannot detect the route leak if AS1 does not register records. So, the protection provided by egress verification is limited. {{I-D.ietf-sidrops-aspa-verification}} suggests to use [RFC9234] for preventing route leak. On the other hand, when RR receives path\[1\] from ASBR1, RR cannot directly do verification because RR does not know the BGP role corresponding to AS1. 

~~~
          +-------------------------------+
          | AS2                           |
   path[1]| +-----+    +-----+    +-----+ |path[2,1]
AS1-------|->ASBR1|----> RR  |---->ASBR2|-|----->AS3
          | +-----+    +-----+    +-----+ |
          |                               |
          +-------------------------------+
~~~
{: #fig-egress-veri  title="Egress/IBGP Verification"}


## Not Applicable to Complex Relationship Scenarios
AS relationships in practical networks may be more complex than the traditional P2C/P2P model {{as-rela-1}}{{as-rela-2}}. In ASPA, the complex relationship of mutual transit relationship has been considered. The followings are some other complex scenarios: 

- Hybrid relationship. A typical complex relationship is hybrid relationship, i.e., two ASes agree to have different traditional relationships for different Points-of-Presence (PoPs). A hybrid relationship may be dependent on IP versions or/and PoP locations. 

- Partial transit relationship. This is another type of complex relationship. For a customer, the provider offers transit only toward the provider's peers and customers, but not the provider's providers, or restricts transit to a specific geographic region. 

- Legitimate valley-paths. In some cases, there may be legitimate valley-paths (i.e., violating the valley-free principle but the AS_PATH is legitimate). 

ASPA records do not support the registration of complex relationships except the mutual transit relationship. As a result, in the complex scenarios, AS_PATH cannot be effectively protected by ASPA-based AS_PATH verification.


<!-- ........................................................................ -->

# Autonomous System Relationship Authorization Record
This section proposes Autonomous System Relationship Authorization (ASRA) record. The record includes not only providers (i.e., what an ASPA record cares) but also other neighboring ASes with different relationships. The followings are the contents that an ASx can put in an ASRA record: 

- Mandatory contents

  - Provider-set (mandatory). Provider-set is the same as ASPA record and includes all the provider ASes of ASx. 

- Optional contents (highly recommended)

  - Neighbor-set (optional). Neighbor-set (or adjacency-set) includes all the neighboring ASes of ASx. This set provides the topology information of ASx without exposing the detailed relationships with the neighboring ASes. 

- Optional contents (recommended)

  - Customer-set (optional). Customer-set includes all the customer ASes of ASx. 

  - Peer-set (optional). Peer-set includes all the lateral peer ASes of ASx. 

- Optional contents (on-demand)

  - Hybrid-set (optional). Hybrid-set includes all the ASes having a hybrid relationship with ASx and the concrete hybrid relationship for different IP versions or/and PoP locations. 

  - Partial-set (optional). Partial-set includes all the ASes having a partial transit relationship with ASx. 

  - Valley-path-set (optional). Valley-path-set includes all the ASes that may construct valley-paths with ASx. For example, ASy and ASz are providers/peers of ASx but are put into the valley-path-set. Then, the path segment of \[ASy, ASx, ASz\] or \[ASz, ASx, ASy\] is valid. 

To support the registration of the above content types, existing ASPA objects MAY need to be extended or new RPKI objects SHOULD be defined. Existing protocols {{I-D.ietf-sidrops-8210bis}} for synchronizing RPKI data from Relay Party (RP) to routers MAY also need to be extended for supporting the synchronization of ASRA data. 

The above content types can be registered together but there MUST be no conflicts between the contents. For example, an AS appearing in both the provider-set and peer-set means a conflict. The registration software of ASRA MUST have a check on the contents. If conflicts exist, the registration will fail. 

Any type of the content MUST include the complete set of the corresponding ASes. For example, customer-set MUST include every customer AS. 


<!-- ........................................................................ -->

# ASRA-based AS_PATH Verification
After that RP obtains validated ASPA/ASRA records, it will cross-validate the correctness of records' content ({{sec-cross-vali}}). The content conflicts will be reported to RP managers. After cross-validation, whether content conflicts are detected or not, the data will be synchronized to the router as it is (See explanations in {{sec-cross-vali}}). 

The upstream and downstream validation procedures of ASRA follow ASPA {{I-D.ietf-sidrops-aspa-verification}}. The concrete validation procedure of ASRA-based AS_PATH verification is TBD. The main difference from ASPA's validation procedures is Hop-check() function. The Hop-check() function in the ASRA-based AS_PATH verification is: 

- If neither ASx nor ASy have ASRA records (i.e., no record or only ASPA record), Hop-check(ASx, ASy) is run as that of ASPA's validation procedures. 

- If ASx has ASRA record that includes some types of the optional contents, Hop-check(ASx, ASy) will do the normal checking as ASPA while considering whether there are fake links or complex relationships in the AS_PATH (See {{sec-fake-link}}, {{sec-distinguish}}, and {{sec-complex-rela}}). 

- If ASy has ASRA record including some types of the optional contents that ASx's ASPA/ASRA record not having (or no record at all), Hop-check(ASx, ASy) will do the normal checking as ASPA. Some additional checking based on ASy's record can also be conducted (See {{sec-reverse-check}}), which is up to the configuration of users. 

The ASRA-based AS_PATH verification can also be applied to egress/iBGP BGP updates if the local AS registers ASRA records {{sec-egress-veri}}. 


## Cross-validate Records of Different ASes {#sec-cross-vali}
ASRA allows two-direction authorization. An AS can register not only all its providers but also all its customers. Two-direction authorization makes it possible to cross-validate the correctness of records of different ASes. Through cross-validation, record conflicts can be detected. Here are some examples of record conflicts:

- Example 1: ASx puts ASy in its customer-set, but ASy does not include ASx in its provider-set. 

- Example 2: ASx does not include ASy in its neighbor-set, but ASy takes ASx as its provider (like the example in {{fig-bogus}}). 

The cross-validation can be conducted on RPs. When a record conflict is detected, at least one of the records should be a bogus record, and there must be at least one AS having made mistakes on the record. After detecting record conflicts, the RP should report the conflict to RP managers who should then contact the corresponding two ASes or RIRs to locate and fix the bogus record. 

An important question is whether the RP should synchronize the two conflicted records to routers for AS_PATH verification. There are two available options: 

- Option 1: Give up the synchronization of the two conflicted records and do the synchronization when the conflict is fixed. 

- Option 2: Synchronize the two records to routers as they are for AS_PATH verification. 

This section recommends option 1. {{fig-cross-vali}} shows why not giving up the synchronization of the two conflicted records. In the figure, AS1 correctly registers its ASRA record that includes the customer-set (i.e., AS2 is its customer). AS2 intentionally or unintentionally registers a bogus ASPA/ASRA record that does not include AS1 in its provider-set. Cross-validation finds the conflict of the records of AS1 and AS2 but does not know which is the bogus one. Supposing RP does not synchronize the two records to the router of AS3, the route leak in the figure cannot be detected. A malicious AS may intentionally register records that have conflicts with other ASes's records to disable them on routers. To avoid the vulnerability, the RP should synchronize the two records to routers as they are no matter whether conflicts exist or not. In the case shown in the figure, the route leak can be detected successfully when AS3 does Hop-check(AS1, AS2) based on AS1's record. 

The drawback of option 1 is the verification may make mistakes when AS1's record is wrong. That is, the negative impact of bogus records faced by ASPA cannot be solved by ASRA. The reason for choosing 2 instead of 1 is mainly to avoid malicious behavior and to ensure that ASRA does not perform worse than ASPA. 

~~~
    +-------+           +-------+
    |  AS1  |           |  AS3  |
    +-------+           +-------+
           \             /
    path[1] \(C2P) (C2P)/ path[2,1]
             \         /
              +-------+
              |  AS2  |
              +-------+
  * AS1 puts AS2 into the customer-set of its ASRA record.
  * AS2 intentionally or unintentionally not includes AS1
    in its provider-set of its ASPA/ASRA record.
~~~
{: #fig-cross-vali  title="AS_PATH Verification based on conflicted records"}


## Detect Fake Links {#sec-fake-link}
The neighbor-set in ASRA records can help routers detect fake links in AS_PATHs. In {{fig-path-shortened}}, AS5 lies a fake link with AS2 and maliciously shortens the AS_PATH before propagating it to AS6. Suppose AS2 registers neighbor-set that only includes AS1 and AS3. AS6 enabling ASRA verification can successfully detect the invalid AS_PATH based on AS2's record, which is shown in {{fig-detect-fake}}. 

~~~
                   +-------+
                   |  AS4  |
                   +-------+
       path[3,2,1]/         \
                 /(C2P)      \ path[4,3,2,1]
         +-------+            \
         |  AS3  |             \
         +-------+              \
   path[2,1] |             (C2P) \
       (C2P) |                    \
neighbor +-------+ (Fake link) +-------+
set={AS1 |  AS2  |*************|  AS5  |
,AS3}    +-------+             +-------+
     path[1] |                     |path[5,2,1]
       (C2P) |                (C2P)|(path shoterned)
         +-------+             +-------+ AS6 detects the fake
         |  AS1  |             |  AS6  | link based on the 
         +-------+             +-------+ neighbor-set of AS2
~~~
{: #fig-detect-fake  title="AS_PATH maliciously shortened by a provider"}

## Distinguish Leak and Hijack for an Invalid AS_PATH {#sec-distinguish}
ASRA can help detect fake links as described in {#sec-fake-link}. If fake links are detected in an AS_PATH, it means route hijack happens, and the verification algorithm can return the result of "invalid-hijack" instead of just "invalid". 

## Verification based on the Next-hop's Record {#sec-reverse-check}
ASRA supports two-direction authorization, so Hop-check(ASx, ASy) can be conducted based on ASy's record if ASx's record is not available. 

In {{fig-reverse-check}}, AS1 is AS2's provider but registers no record. AS2 registers an ASRA record that puts AS1 into the provider-set and does not put AS1 into the customer-set. According to AS2's record, the relationship between AS1 and AS2 is P2C instead of C2P or mutual transit. 

If AS3 enables the verification based on the next-hop's record, Hop-check(ASx, ASy) will not return "No attestation" but return "Not provider+". As a result, the route leak in the figure can be detected. 

The verification based on the next-hop's record is helpful when the record is not widely registered. However, this usage may be leveraged by attackers. The attacks based on maliciously registered bogus records may be launched. That is, AS2 in the figure maliciously registers AS1 as its customer and makes AS3 fail to detect leak. Such kind of attacks should seldom happen because the attacker cannot repudiate the attack if the attacker registers the bogus ASRA data in advance. Whether to enable this kind of verification depends on the network managers. 

~~~
    +-------+           +-------+
    |  AS1  |           |  AS3  |
    +-------+           +-------+
           \             /
    path[1] \(C2P) (C2P)/ path[2,1]
             \         /
              +-------+
              |  AS2  |
              +-------+
  * AS1 registers no record.
  * AS2 includes AS1 in its provider-set and 
    does not put AS1 in its customer-set.
~~~
{: #fig-reverse-check  title="An example of verification based on the next-hop's record"}

## Egress/IBGP Verification {#sec-egress-veri}
ASRA can solve the problems of ASPA and meet the requirements of egress verification and iBGP verification. An AS can register ASRA records first, and all the routers can get the records through RP. The route leak on ASBR2 shown in {{fig-egress-veri}} can be prevented by egress verification even if AS1 does not register its records. This is because the ASRA records of AS2 can be used to do verification as described in {{sec-reverse-check}}. 

In {{fig-egress-veri}}, when RR receives AS_PATH from ASBR1, RR can do iBGP verification because RR can know the BGP role corresponding to AS1 based on the locally registered ASRA record. 

## Cope with Complex Scenarios {#sec-complex-rela}
TBD
<!-- If ASx registers hybrid or partial transit relationships in its ASRA record, the Hop-check(ASx, ASy) function should check whether ASx and ASy have the complex relationships.  -->


<!-- ........................................................................ -->

# Deployment Considerations
Although there are some analysis results on why privacy problem is not much important in many cases {{privacy-analysis}}, many ASes may still be concerned about privacy problems. ASRA supports selective registration. ASes can choose to register neighbor-set, detailed relationships, etc. Neighbor-set induces relatively less privacy concerns. 

Selective registration does not solve the privacy problem, and globally public record registration is likely to take a long time. Regional deployment is a feasible choice to quickly deploy ASRA. ASRA records can be registered and only used within a region. Deploying ASRA within a region can also provide effective protection to the Internet. Route leaks and hijacks (especially type-1 hijack and type-2 hijack) can be detected and mitigated before leaving the region. 

ASRA-based AS_PATH verification can efficiently detect route leaks and fake link-based hijacks. However, it cannot prevent AS_PATH tampering that obeys the valley-free principle and does not induce fake links. BGPsec [RFC8205] or similar mechanisms are needed as a complement to ASRA. 


# Security Considerations {#sec-security}

TBD

# IANA Considerations {#sec-iana}

TBD

# Acknowledgements

TBD

--- back

