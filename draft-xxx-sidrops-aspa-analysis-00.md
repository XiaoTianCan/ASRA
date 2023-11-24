---
title: An analysis of ASPA-based AS_PATH Verification
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
Autonomous System Provider Authorization (ASPA) is very helpful in detecting and mitigating route leaks (valley-free violations) and a majority of forged-origin hijacks. This document does an analysis on ASPA-based AS_PATH verification to help people understand the strengths and deficiencies of ASPA. 


--- middle

# Introduction
Autonomous System Provider Authorization (ASPA) is a technique for verifying AS_PATHs in BGP updates {{I-D.ietf-sidrops-aspa-verification}}{{I-D.ietf-sidrops-aspa-profile}}. Each AS can register ASPA records (also ASPA objects) in the RPKI to state all its provider ASes. An AS can obtain other ASes' ASPA records and conduct AS_PATH verification based the records. ASPA-based AS_PATH verification can detect and mitigate route leaks violating the valley-free principle and route hijacks such as forged-origin or forged-path-segment attacks. 

ASPA can significantly simplify and enhance AS_PATH verification and is promising to be widely deployed. Despite of the strengths of ASPA, there are also some deficiencies. This document provides a detailed analysis on the strengths and deficiencies of ASPA. 


## Terminology

The usage of terms follows {{I-D.ietf-sidrops-aspa-verification}}{{I-D.ietf-sidrops-aspa-profile}}. 

## Requirements Language

{::boilerplate bcp14-tagged}

# ASPA Strengths and Disclaimers
ASPA records can be registered by an AS to include all the provider ASes of the AS. For the two ASes with mutual transit relationship, each of the two ASes will put the other AS into its own ASPA record (i.e., consider the other AS as its provider). 

In full ASPA deployment (within a given region of interest), all "route leaks" (valley-free violations [RFC7908]) are detectable. Route leaks involve one of the following four valley-free violations: 

- A route is propagated through a P2C (Provider-to-Customer) link and then a C2P (Customer-to-Provider) link. 

- A route is propagated through a P2P (Peer-to-Peer) link and then a C2P link. 

- A route is propagated through a P2P link and then a P2P link. 

- A route is propagated through a P2C link and then a P2P link. 

It is expected that in partial ASPA deployment, not all route leaks are detectable. 

In full ASPA deployment (within a given region of interest), ASPA protects against a majority of forged-origin hijacks. Each AS can attest its upstream ASes, so provider or lateral peer cannot be deceived. Customer could be deceived because ASPA does not provide attestations to downstream ASes or peering ASes. 

Even in full ASPA deployment, not all path hijack (i.e., path modification based on insertion or removal of unique ASN) attacks can be detected. ASPA does not guarantee path correctness like that provided by BGPsec [RFC8205]. 

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

{{fig-bogus}} shows an example of AS_PATH hijack based on bogus ASPA records. AS4 lies in that the nonadjacent AS3 is its provider in the ASPA record. The attack cannot be detected even when AS1, AS2, AS3, and AS5 register ASPA records correctly and enable ASPA-based AS_PATH verification locally. As a result, AS5 will wrongly consider its traffic to AS1 traverses AS3, while the real forwarding path to AS1 is through AS2 instead of AS3. 

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
ASPA-based AS_PATH verification cannot effectively detect the AS_PATH maliciously shortened by a provider. {{fig-path-shortened}} shows an example. AS1 originates the BGP route and propagates the route to other ASes. The AS_PATH received by AS5 is path\[4,3,2,1\]. However, AS5 maliciously shortens the path by falsely claim a fake link with AS2 before AS5 propagates the route to AS6. AS6's traffic to AS1 may be hijacked by AS5 if the path\[5,2,1\] is shorter than any other AS_PATHs. In the example, AS5 may not intend to drop data traffic from AS6. That is, AS5 (provider) wants AS6 (customer) to prefer AS5's transit path for earning more money. 

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
      (C2P) |                    \           /
        +-------+ (Fake link) +-------+     /
        |  AS2  |*************|  AS5  |    /
        +-------+             +-------+   /path[7,4,3,2,1]
    path[1] |          path[5,2,1]|      /
      (C2P) |     (path shoterned)|(C2P)/(C2P)
        +-------+             +-------+/
        |AS1(P1)|             |  AS6  |
        +-------+             +-------+
      Originate route
~~~
{: #fig-path-shortened  title="AS_PATH maliciously shortened by a provider"}


## Cannot Distinguish Leak and Hijack for an Invalid AS_PATH
Existing ASPA verification algorithm can identify invalid AS_PATHs, but it cannot distinguish leak and hijack for an invalid AS_PATH. The main reason is that ASPA records only focus on registering all provider ASes while not indicating the adjacency/topology information. When the Hop-check(x, y) function returns "Not provider+", the algorithm does not know whether the real cause of the result is 1) ASy is ASx's customer/peer instead of provider or 2) link(x,y) is a fake link. Therefore, when the algorithm returns invalid, it is not able to indicate whether the path is caused by a fake link-based hijack or not. 

For operators, it's instructive to know the type of an invalid AS_PATH. If there exists no hijack, the invalid AS_PATH is likely to be an unintentional route leak. Otherwise, the network may be attacked by route hijacks.  


## Not Directly Applicable to IBGP Ingress and EBGP Egress Verification
IBGP ingress verification and eBGP egress verification are meaningful in many scenarios. IBGP ingress verification is to check the AS_PATH received through iBGP connections. IBGP ingress verification helps an AS do verification on any BGP routers like non-ASBRs. EBGP egress verification means verify the AS_PATH before sending it to the neighbor AS. It can help ASes avoid making some mistakes (e.g., route leaks). However, current ASPA documents does not specify how to do iBGP ingress verification and eBGP egress verification. 

For iBGP ingress verification, 不知道相对发来路由的AS的对端AS的角色。事实上，可以从本地的ASPA上获得。一般来说，本地ASPA和role配置应该是一致的。如图所示，R2 looks up AS2’s own ASPA
If AS1 is listed as a Provider, then apply the Downstream algorithm
If AS1 is not listed as a Provider, then apply the Upstream algorithm
Works correctly, including RS and RS-client scenarios


For eBGP egress verification, 一方面不知道对端发来的路由的AS的对端角色是什么，

The main challenge to do iBGP verification is how to know the relationship between the 

Actually, iBGP ingress: Doable with help from AS2’s own ASPA


An AS enabling existing ASPA cannot directly apply egress verification or iBGP verification. {{fig-egress-veri}} shows an example. Suppose AS2 enables ASPA-based AS_PATH verification and only registers ASPA records. Suppose ASBR2 propagates path\[2,1\] to AS3, which results in route leak. The egress verification on ASBR2 cannot detect the route leak if AS1 does not register records. So, the protection provided by egress verification is limited. {{I-D.ietf-sidrops-aspa-verification}} suggests to use [RFC9234] for preventing route leak. On the other hand, when RR receives path\[1\] from ASBR1, RR cannot directly do verification because RR does not know the BGP role corresponding to AS1. 

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


## Not Applicable to Complex Relationship Scenarios
AS relationships in practical networks may be more complex than the traditional P2C/P2P model {{as-rela-1}}{{as-rela-2}}. In ASPA, the complex relationship of mutual transit relationship has been considered. The followings are some other complex scenarios: 

- Hybrid relationship. A typical complex relationship is hybrid relationship, i.e., two ASes agree to have different traditional relationships for different Points-of-Presence (PoPs). A hybrid relationship may be dependent on IP versions or/and PoP locations. 

In some cases, even per prefix. 

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

- Partial transit relationship. This is another type of complex relationship. For a customer, the provider offers transit only toward the provider's peers and customers, but not the provider's providers, or restricts transit to a specific geographic region. 

~~~
        +-------+
        |  AS3  |
        +-------+
  path[2,1] |
(route leak)|
            |
       (C2P)|
        +-------+  path[2,1]  +-------+
        |  AS2  |-------------|  AS4  |
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

- Persistent valley-path (legitimate valley-path). In some cases, there may be legitimate valley-paths (i.e., violating the valley-free principle but the AS_PATH is legitimate). 

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


~~~
                            +-------+
                            |  AS3  |
                            +-------+
                            /
                           / path[2,1] (route leak)
                          /(C2P)
  +-------+  (P2P)  +-------+
  |AS1(P1)|---------|  AS2  |
  +-------+ path[1] +-------+
Originate route

      (a) route leak in partial deployment

                            +-------+
                            |  AS3  |
                            +-------+
                            /
                           / path[2,1] (path hijack)
         (no adjacency)   /(C2P)
  +-------+         +-------+
  |AS1(P1)|*********|  AS2  |
  +-------+ path[1] +-------+
Originate route

      (b) Forged-origin attack in partial deployment
~~~
{: #fig-partial-deploy  title="Partial deployment"}

## Not all Malicious Route Leaks are Prevented

~~~
        +-------+
        |  AS4  |
        +-------+
P1 path[2,1]|
P2 path[2,1]|
       (C2P)|
        +-------+ P2 path[3,1]+-------+
        |  AS2  |-------------|  AS3  |
        +-------+             +-------+
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
There are three main reasons:

- ASPA record is authorized in one direction, e.g., an AS can unilaterally claim that another AS is its provider without the consent of other ASes. 

- An ASPA record only focuses on including all provider ASes while ignoring other topology or relationship information. 

- The kind of method that verifies AS_PATH based on relationships does not guarantee path correctness like that provided by BGPsec. 


# Security Considerations {#sec-security}

This document does not involve security problems. 

# IANA Considerations {#sec-iana}

No IANA requirements. 

# Acknowledgements

TBD

--- back


