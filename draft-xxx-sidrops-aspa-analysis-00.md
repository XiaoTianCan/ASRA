---
title: An Analysis of ASPA-based AS_PATH Verification
abbrev: ASPA analysis
docname: draft-xxx-sidrops-aspa-analysis-00
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
  RFC7908:

informative:
  I-D.ietf-sidrops-8210bis:
  RFC8205:
  RFC8893:
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
  valley-path:
    title: Valley-free violation in Internet routing - Analysis based on BGP Community data
    author: 
    org: ICC
    date: 2012-11
    target: https://ieeexplore.ieee.org/document/6363987

...

--- abstract
Autonomous System Provider Authorization (ASPA) is very helpful in detecting and mitigating route leaks (valley-free violations) and a majority of forged-origin hijacks. This document does an analysis on ASPA-based AS_PATH verification to help people understand its strengths and deficiencies. 


--- middle

# Introduction
Autonomous System Provider Authorization (ASPA) is a technique for verifying AS_PATHs in BGP updates {{I-D.ietf-sidrops-aspa-verification}}{{I-D.ietf-sidrops-aspa-profile}}. Each AS can register ASPA records (also ASPA objects) in the RPKI to state all its provider ASes. An AS can obtain other ASes' ASPA records through RTRv2 protocol {{I-D.ietf-sidrops-8210bis}} and conduct AS_PATH verification based the records. ASPA-based AS_PATH verification can detect and mitigate route leaks violating the valley-free principle and route hijacks such as forged-origin or forged-path-segment attacks. 

ASPA can significantly enhance AS_PATH verification and is promising to be widely deployed. Despite of the strengths of ASPA, there are also some deficiencies. This document provides a detailed analysis on the strengths and deficiencies of ASPA. 


## Terminology

The usage of terms follows {{I-D.ietf-sidrops-aspa-verification}}{{I-D.ietf-sidrops-aspa-profile}}. 

## Requirements Language

{::boilerplate bcp14-tagged}

# ASPA Strengths and Disclaimers
ASPA records can be registered by an AS to include all the provider ASes of the AS. For the two ASes with mutual transit relationship, each of the two ASes will put the other AS into its own ASPA record (i.e., consider the other AS as its provider). 

## Protecting Against Route Leak
In full ASPA deployment (within a given region of interest), all "route leaks" (valley-free violations [RFC7908]) are detectable. Route leaks involve one of the following four valley-free violations: 

- A route is propagated through a P2C (Provider-to-Customer) link and then a C2P (Customer-to-Provider) link. 

- A route is propagated through a P2P (Peer-to-Peer) link and then a C2P link. 

- A route is propagated through a P2P link and then a P2P link. 

- A route is propagated through a P2C link and then a P2P link. 

It is expected that in partial ASPA deployment, not all route leaks are detectable. 

## Protecting Against Path Hijack
Path hijack means path modification based on insertion or removal of unique ASN in this document. Forged-origin hijack and fake link-based hijack are all path hijacks. 

In full ASPA deployment (within a given region of interest), ASPA protects against a majority of forged-origin hijacks. Each AS can attest its upstream ASes, so provider or lateral peer cannot be deceived. Customer could be deceived because ASPA does not provide attestations to downstream ASes or peering ASes. 

Even in full ASPA deployment, not all path hijack attacks can be detected. ASPA does not guarantee path correctness like that provided by BGPsec [RFC8205]. 


# ASPA Deficiencies
This section describes the deficiencies of ASPA-based AS_PATH verification. 

## Hard to Detect Bogus Records
An AS can unilaterally authorize a set of its provider ASes. Under the one-direction authorization, an AS may intentionally or unintentionally register bogus records that are hard to be discovered. Consider two ASes, i.e., ASx and ASy. The bogus ASPA records are registered by ASx. ASy is an AS that either has no link with ASx or has some relationship with ASy. Some examples of bogus ASPA records are listed as follows: 

- ASx and ASy are not adjacent, but ASx puts ASy into its provider-set in the ASPA record. The bogus records may be leveraged by attackers to launch route hijack attacks. 

- ASy is ASx's customer/peer (or other non-provider roles) instead of ASx's provider, but ASx puts ASy into its provider-set in the ASPA record. Some route leaks may be undetectable due to the existence of bogus records. 

- ASy is ASx's provider, but ASx DOES NOT put ASy into its provider-set in the ASPA record. This type of bogus records may lead to improper block of legitimate BGP routes. 

Bogus records bring two risks:

- An AS unintentionally registers a bogus record that affects the verification accuracy while is hard to be discovered. 

- An AS maliciously registers bogus records that open a door to potential attacks. 

{{fig-bogus}} shows an example of path hijack based on bogus ASPA records. AS4 lies in that the nonadjacent AS3 is its provider in the ASPA record. The attack cannot be detected even when AS1, AS2, AS3, and AS5 register ASPA records correctly and enable ASPA-based AS_PATH verification locally. As a result, AS5 will wrongly consider its traffic to AS1 traverses AS3, while the real forwarding path to AS1 is through AS2 instead of AS3. 

~~~
              +-------+
              |AS1(P1)|Originate route
              +-------+
     path[1] /        \
            /(C2P)     \(C2P)
    +-------+           +-------+
    |  AS2  |           |  AS3  |
    +-------+           +-------+
           \             *
  path[2,1] \(C2P)      * (faked C2P in AS4's
             \         *   bogus ASPA)
              +-------+ ASPA{AS4, [AS2, AS3]}
              |  AS4  | Offender
              +-------+
      path[4,3,1] |(C2P)
              +-------+
              |  AS5  | Victim
              +-------+
  * All ASes register ASPA records
  * AS4's record states a faked C2P link with AS3
  * AS5 enables ASPA-based path verification
~~~
{: #fig-bogus  title="Path hijack based on bogus ASPA records"}


## Fail to Detect AS_PATH Maliciously Shortened by a Provider
ASPA-based AS_PATH verification cannot effectively detect the AS_PATH maliciously shortened by a provider. {{fig-path-shortened}} shows an example. AS1 originates the BGP route and propagates the route to other ASes. The AS_PATH received by AS5 is path\[4,3,2,1\]. However, AS5 maliciously shortens the path by falsely claim a fake link with AS2 before AS5 propagates the route to AS6. AS6's traffic to AS1 may be hijacked by AS5 if the path\[5,2,1\] is shorter than any other AS_PATHs. In the example, AS5 may not intend to drop data traffic from AS6. That is, AS5 (provider) wants AS6 (customer) to prefer AS5's transit path for increasing revenue. 

In this case, the attack cannot be detected even when all the ASes register the correct ASPA records and all the ASes other than AS5 enable ASPA-based AS_PATH verification. 

~~~
                  +-------+
                  |  AS4  |    path[4,3,2,1]
                  +-------+----------------
      path[3,2,1]/         \               \
                /(C2P)      \               \(C2P)
        +-------+            \path[4,3,2,1] +-------+
        |  AS3  |             \             |  AS7  |
        +-------+              \            +-------+
  path[2,1] |             (C2P) \             /
      (C2P) |                    \ Offender  /
        +-------+ (Fake link) +-------+     /
        |  AS2  |*************|  AS5  |    /
        +-------+             +-------+   /path[7,4,3,2,1]
    path[1] |          path[5,2,1]|      /
      (C2P) |     (path shoterned)|(C2P)/(C2P)
        +-------+             +-------+/
        |AS1(P1)|             |  AS6  | Victim
        +-------+             +-------+
      Originate route
~~~
{: #fig-path-shortened  title="AS_PATH maliciously shortened by a provider"}


## Cannot Distinguish Leak and Hijack for an Invalid AS_PATH
Existing ASPA verification algorithm can identify invalid AS_PATHs, but it cannot distinguish leak and hijack for an invalid AS_PATH. The main reason is that ASPA records only focus on registering all provider ASes while not indicating the adjacency/topology information. When the Hop-check(x, y) function returns "Not provider+", the algorithm does not know whether the real cause of the result is i) ASy is ASx's customer/peer instead of provider or ii) link(x,y) is a fake link. Therefore, when the algorithm returns invalid, it is not able to indicate whether the path is caused by a fake link-based hijack or not. 

For operators, it's instructive to know the type of an invalid AS_PATH. If there exists no hijack, the invalid AS_PATH is likely to be an unintentional route leak. Otherwise, the network may be attacked by route hijacks. 


## Not Directly Applicable to IBGP Ingress and EBGP Egress Verification
IBGP ingress verification and eBGP egress verification are meaningful in many scenarios. IBGP ingress verification is to check the AS_PATH received through iBGP connections. IBGP ingress verification helps an AS do verification on any BGP routers like non-ASBRs. EBGP egress verification means verifying the AS_PATH before sending it to the neighbor AS. It can prevent an AS from sending routes with Invalid AS_PATH to its neighbor ASes (just like eBGP egress RPKI-ROV [RFC8893]). 

However, current ASPA document {{I-D.ietf-sidrops-aspa-verification}} does not specify how to do iBGP ingress verification. For iBGP ingress verification, the router (e.g., an RR) conducting the verification may not have BGP sessions with the neighbor AS that propagates the route and thus does not know the local BGP role with respect to the neighbor AS. Even so, iBGP ingress verification is doable because the router can obtain the local BGP role from the ASPA records of local AS. In {{fig-egress-veri}}, when RR wants to do iBGP ingress verification, it can look up AS2's own ASPA records. If AS1 is listed as a provider, then apply the Downstream verification algorithm. If AS1 is not listed as a provider, then apply the Upstream verfication algorithm. Such an iBGP ingress verification also works correctly in RS and RS-client scenarios. 

~~~
          +-------------------------------+
          | AS2                           |
   path[1]| +-----+    +-----+    +-----+ |path[2,1]
AS1-------|->ASBR1|----> RR  |---->ASBR2|-|----->AS3
         /| +-----+   /+-----+    +-----+ |\
        / |          /                    | \
       /  +---------/---------------------+  \
      /            /                          \
  eBGP ingress   iBGP ingress              eBGP egress
~~~
{: #fig-egress-veri  title="IBGP and eBGP verification"}

{{I-D.ietf-sidrops-aspa-verification}} does not specify how to do eBGP egress verification either. To try to do this, the router should add its own AS (i.e., AS2) to the AS_PATH and then performs ASPA-based AS_PATH verification from the perspective of next-hop AS (see Section 7.2 in version-15 of {{I-D.ietf-sidrops-aspa-verification}}). According to the verification result, the router decides to propagate the route or not. 
In {{fig-egress-veri}}, ASBR2 would add its own AS (i.e., AS2) to the AS_PATH. If the BGP role of ASBR2 with respect to AS3 is customer/lateral peer/RS/RS-client, the Upstream verification algorithm will be conducted. If the BGP role of ASBR2 with respect to AS3 is provider, the Downstream verification algorithm will be performed. The verification process also works well in mutual transit scenarios. 

The problem of eBGP egress verification described above is that the Invalid AS_PATH cannot be prevented from being propagated in all cases. Suppose the next-hop AS (i.e., neighbor AS3) is a lateral peer or provider. When the preceding AS (i.e., neighbor AS1) is also a lateral peer or provider but does not have an ASPA registered, the verification result can be Unknown (rather than Invalid). The route leak cannot be prevented in the case. 

The relationship between AS1 and AS2 can sometimes be obtained by ASBR2 from AS2's ASPA records. If AS1 is listed as a provider in the AS2's ASPA records, then the above route leak can be detected and prevented. If AS1 is not listed in the AS2's ASPA records, ASBR2 cannot decide AS1 is a lateral peer or a customer. Therefore, the above route leak cannot be detected directly. 

Overall, iBGP ingress verification is doable with help of local AS's own ASPA records, while it is not possible to do eBGP egress verification correctly without more complexity. 


## Not Applicable to Complex Relationship Scenarios
AS relationships in practical networks may be more complex than the traditional P2C/P2P model {{as-rela-1}}{{as-rela-2}}. In ASPA, only the complex relationship of mutual transit relationship has been considered. The followings are some other complex scenarios that are not covered by ASPA: 

- Hybrid relationship {{as-rela-1}}{{as-rela-2}}. Two ASes may agree to have different traditional relationships for different Points-of-Presence (PoPs). A hybrid relationship may be dependent on IP versions or/and PoP locations (see {{fig-hybrid-rela}} for examples), even prefixes. 

~~~
+-------+  Europe P2C  +-------+
|       |-------------->       |
|  AS1  |              |  AS2  |
|       |-------------->       |
+-------+   Asia P2P   +-------+

      (a) Location-dependent


+-------+   IPv6 P2C   +-------+
|       |-------------->       |
|  AS1  |              |  AS2  |
|       |-------------->       |
+-------+   IPv4 P2P   +-------+

      (b) IP version-dependent
~~~
{: #fig-hybrid-rela  title="Hybrid relationship"}

- Partial transit relationship {{as-rela-1}}{{as-rela-2}}. For a customer, the provider offers transit only toward the provider's peers and customers (or specific regions), but not the provider's providers, or restricts transit to a specific geographic region. {{fig-hybrid-rela}} shows an example. AS2 is the partial transit provider of AS1. AS2 should only propagate AS1's route to AS4 (i.e., AS2's peer) and AS5 (i.e., AS2's customer), but should not propagate the route to AS3 (i.e., AS2's provider). 

~~~
        +-------+
        |  AS3  |
        +-------+
  path[2,1] |
(route leak)|
            |
       (C2P)|
        +-------+  path[2,1]  +-------+
Offender|  AS2  |-------------|  AS4  |
        +-------+    (P2P)    +-------+
     Partial|    \                |
     transit|     \ path[2,1]     | path[4,2,1]
            |      ----------     |
    path[1] |           (C2P)\    |(C2P)
        +-------+             +-------+
        |  AS1  |             |  AS5  |
        +-------+             +-------+
      Originate route
~~~
{: #fig-partial-transit  title="Partial transit relationship"}

- Persistent valley-path (legitimate valley-path) ({{valley-path}}). In some cases, there may be legitimate valley-paths, i.e., violating the valley-free principle but the AS_PATH is legitimate. 

~~~
  Originate route
    +-------+             +-------+
    |AS1(P1)|             |  AS3  |
    +-------+             +-------+
            \             /
      path[1]\           / path[2,1] (not route leak)
              \(C2P)    /(C2P)
              +---------+
              |   AS2   |
              +---------+
~~~
{: #fig-valley-path  title="Persistent valley-path"}

ASPA records do not support the registration of complex relationships except the mutual transit relationship. As a result, in the complex scenarios, AS_PATH cannot be effectively protected by ASPA-based AS_PATH verification.

## Reduced Protection Capacity in Partial Deployment
To verfify an AS_PATH, ASPA verification algorithms need to check each hop of the AS_PATH. When ASPA records of the ASes along the path are partially registered, not all hops in the path can be checked. In such partial deployment scenarios, ASPA may have a reduced protection capacity. Compared to ROA, ASPA provides less benefits in the early deployment. 

{{fig-partial-deploy}} shows two examples of partial deployment. In {{fig-partial-deploy}} (a), AS3 cannot detect the route leak of P1 induced by AS2 if AS1 has no ASPA record registered. This is because the Hop-check(AS1, AS2) function returns "No Attestation" and the final verification result is Unknown. In {{fig-partial-deploy}} (b), AS3 is deceived by AS2 who falsely claims AS1 is AS2's neighbor. The attack in the example is undetectable because AS1 registers no ASPA record and AS3 cannot judge the validity of the link between AS1 and AS2. 

~~~
                            +-------+
                            |  AS3  |
                            +-------+
                            /
                           / path[2,1] (route leak)
   No ASPA                /(C2P)
  +-------+  (P2P)  +-------+
  |AS1(P1)|---------|  AS2  | Offender
  +-------+ path[1] +-------+
Originate route

      (a) route leak in partial deployment

                            +-------+
                            |  AS3  |
                            +-------+
                            /
                           / path[2,1] (path hijack)
   No ASPA (no adjacency) /(C2P)
  +-------+         +-------+
  |AS1(P1)|*********|  AS2  | Offender
  +-------+ path[1] +-------+
Originate route

      (b) Forged-origin attack in partial deployment
~~~
{: #fig-partial-deploy  title="Partial deployment"}

## Not all Malicious Route Leaks are Prevented
ASPA is not designed for path hijack (path modification). So, some malicious route leaks with path hijack involved cannot be prevented. 

{{fig-malicious-leak}} shows an example. AS2 is AS1's provider and arguably it may not leak its customer's prefix (P2) intentionally. But to increase revenue, AS2 may maliciously leak P2 with a modified AS_PATH (i.e., AS3 is removed) to AS4 for attracting more traffic to traverse AS2. Sometimes this attack may happen and goes undetected. 

~~~
        +-------+
        |  AS4  |-------------
        +-------+             \
P1 path[2,1]|                  \
P2 path[2,1]|                   \ P2 path[3,1]
       (C2P)|                    \(C2P)
        +-------+ P2 path[3,1]+-------+
Offender|  AS2  |-------------|  AS3  |
        +-------+    (P2P)    +-------+
               \             /
      P1 path[1]\           /P2 path[1]
                 \(C2P)    /(C2P)
                  +---------+
                  |   AS1   |Originate route
                  | (P1&P2) |
                  +---------+
~~~
{: #fig-malicious-leak  title="Malicious route leak"}


<!-- ........................................................................ -->

# Reasons of ASPA Deficiencies
This section summarizes three main reasons that result in deficiencies of ASPA:

- ASPA record is authorized in one direction, e.g., an AS can unilaterally claim that another AS is its provider without the consent of other ASes. Related deficiencies:
  - Hard to detect bogus records

- An ASPA record only focuses on including all provider ASes while ignoring other topology or relationship information. Related deficiencies:
  - Hard to detect bogus records
  - Fail to detect AS_PATH maliciously shortened by a provider
  - Cannot distinguish leak and hijack for an Invalid AS_PATH
  - Not directly applicable to iBGP Ingress and eBGP egress verification
  - Not applicable to complex relationship scenarios
  - Reduced protection capacity in partial deployment

- The kind of method that verifies AS_PATH based on relationships does not guarantee path correctness like that provided by BGPsec. 
  - Not all malicious route leaks are prevented


# Security Considerations {#sec-security}

This document does not involve security problems. 

# IANA Considerations {#sec-iana}

No IANA requirements. 

# Acknowledgements

TBD

--- back

