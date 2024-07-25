---
title: Autonomous System Relationship Authorization (ASRA) for AS_PATH Verification
abbrev: ASRA for AS_PATH Verification
docname: draft-xxx-sidrops-asra-verification-00
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
  ins: K. Sriram
  name: Kotikalapudi Sriram
  organization: NIST
  email: ksriram@nist.gov
  city: Gaithersburg, MD 20899
  country: United States of America

 -
  ins: N. Geng
  name: Nan Geng
  organization: Huawei
  email: gengnan@huawei.com
  city: Beijing
  country: China

normative:
  I-D.ietf-sidrops-aspa-verification:
  I-D.ietf-sidrops-aspa-profile:

informative:
  I-D.ietf-sidrops-8210bis:
  RFC8416:
...

--- abstract
Autonomous System Provider Authorization (ASPA) record is authorized in one direction and focuses on including all provider ASes without including other neighboring ASes. While ASPA-based AS_PATH verification can efficiently detect and mitigate route leaks and some forged-origin or forged-path-segment hijacks, it may fail to detect malicious path modifications for routes that are received from transit providers.

This document proposes Autonomous System Relationship Authorization (ASRA). ASRA allows an AS to register the mixed set of customer ASes and lateral peer ASes. AS_PATH verification based on both ASPA and ASRA records can enhance AS_PATH verification, and malicious path modifications by transit providers can be efficiently detected and mitigated. 

--- middle

# Introduction
Autonomous System Provider Authorization (ASPA) is a technique for protecting AS_PATHs in BGP updates {{I-D.ietf-sidrops-aspa-verification}}{{I-D.ietf-sidrops-aspa-profile}}. Each AS can register ASPA records (also ASPA objects) in the RPKI to states all its provider ASes. An AS can obtain other ASes' ASPA records and conduct AS_PATH verification based the records. ASPA-based AS_PATH verification can detect and mitigate route leaks violating the valley-free principle and route hijacks such as forged-origin or forged-path-segment attacks. 

As described in Section 12 of {{I-D.ietf-sidrops-aspa-verification}}-v17, ASPA-based AS_PATH verification might fail to detect some malicious path modifications, especially for routes that are received from transit providers. The reason of the limitation is that ASPA allows an AS to register only providers and not all fake links in the AS_PATH can be detected without the full information of neighboring ASes. 

This document proposes Autonomous System Relationship Authorization (ASRA). An AS can register ASRA records that include the mixed set of customer ASes and lateral peer ASes. ASRA can be used to enhance AS_PATH verification and improve verification accuracy. 

ASRA does not change the procedures of ASPA-based AS_PATH verification. Without ASRA records, an AS can conduct ASPA-based AS_PATH verification. If ASRA records are available, the enhanced verification can be carried out. 

## Terminology

The usage of terms follows {{I-D.ietf-sidrops-aspa-verification}}{{I-D.ietf-sidrops-aspa-profile}}. 

## Requirements Language

{::boilerplate bcp14-tagged}

# Use Case: AS_PATH Maliciously Shortened by a Provider {#use-case}

ASPA-based AS_PATH verification cannot effectively detect the AS_PATH maliciously shortened by a provider. {{fig-path-shortened}} shows an example. Suppose all ASes register ASPA records and doing ASPA-based AS_PATH verification. AS1 originates the BGP route and propagates the route to other ASes. The AS_PATH received by AS6 is path\[5,4,3,2,1\]. However, AS6 maliciously shortens the path by falsely claim a fake link with AS2 before AS6 propagates the route to AS7. AS7 fails to detect the path modifications even it enables ASPA. As a result, the traffic from AS7 to AS1 will be hijacked by AS6. 

~~~
            +-------+      +-------+  path[5,4,3,2,1]
            |  AS4  |------|  AS5  |--------------
            +-------+      +-------+              \
    path[3,2,1]/                \                  \ (C2P)
              /(C2P)             \                  \
      +-------+                   \              +-------+
      |  AS3  |    path[5,4,3,2,1] \             |  AS8  |
      +-------+               (C2P) \            +-------+
path[2,1] |                          \   path[8,5,4,/|
    (C2P) |                           \      3,2,1]/ |
      +-------+    (Fake link)     +-------+ (C2P)/  |path[8,5,4,
      |  AS2  |********************|  AS6  |------   |    3,2,1]
      +-------+                    +-------+         | (C2P)
  path[1] |                            |path[6,2,1]  |
    (C2P) |                       (C2P)|(shortened)  |
      +-------+                    +-------+         |
      |  AS1  |                    |  AS7  |----------
      +-------+                    +-------+
          | (Origin)            (Validating AS)
          P     
~~~
{: #fig-path-shortened  title="AS_PATH maliciously shortened by a provider"}


<!-- ........................................................................ -->

# Autonomous System Relationship Authorization Record
ASRA record lists a subject AS (signing AS) and a specified set of object ASes with which the subject AS has a BGP relationship. ASRA record contains an octet designated as the relationship type field. When this field has a value 0, it means that the listed set of object ASes are the combinations of customers and/or lateral peers. In this design choice, there is no distinction made between a customer and a lateral peer in the list of ASes. 
A value 1 for the field indicates that the listed set of object ASes are customers. A value 2 for the field indicates that the listed set of object ASes are lateral peers. 

When there is a complex relationship between two ASes AS x and AS y, such as, they are Customer-Provider (AS x is Customer and AS y is Provider) on one BGP session and lateral peers on another BGP session, then AS x MUST include AS y in its ASPA as well as ASRA. In another instance of a complex relationship where AS y is Customer on one BGP session and lateral peer on another BGP session, AS x MUST include AS y in its ASRA record only. In the case of a mutual transit relationship, each AS MUST include the other AS in its ASPA as well as ASRA. This is because each is a Provider as well as a Customer of the other.   

For the purpose of this document, the design choice with value 0 is made exclusively (i.e., values 1 and 2 are not allowed). An AS SHOULD have a single ASRA record (with type field = 0) listing all its customers and lateral peers. If multiple ASRA records exist for a given AS, the union of the sets of object ASes constitutes the whole set of object ASes {customers and lateral peers}. 

The design choice above, where the relationship type field is set to 0, is likely to be attractive to network operators since customer ASes need not be disclosed explicitly. 

Noted that, if the ASRA record of an AS exists, the ASPA record of the AS MUST also exist. If ASRA exist but ASRA not, then the ASRA MUST NOT be considered for AS_PATH verification. 

To support the registration and transmission of ASRA records, a new RPKI object, SLURM file [RFC8416] extensions, RTR protocol {{I-D.ietf-sidrops-8210bis}} extensions will be needed. 

<!-- ........................................................................ -->

# Authorization Functions
Let ASx and ASy represent two unique ASes. The authorization function, ASPA_authorized(ASx, ASy), checks if the ordered pair of ASNs, (ASx, ASy), has the property that ASy is an attested provider of ASx per union set of provider ASes of ASx. ASPA_authorized(ASx, ASy) is same as the Provider Authorization Function, i.e., authorized(ASx, ASy), defined in {{I-D.ietf-sidrops-aspa-verification}}. 

The authorization function, ASRA_authorized(ASx, ASy), checks if the ordered pair of ASNs, (ASx, ASy), has the property that ASy is an attested AS of ASx per union set of customer and lateral peer ASes of ASx. By term "True" the function should signal if ASy plays role of customer, lateral peer, or RS-client. This function is specified as follows:

~~~
                            /
                            | "True" if ASy is included in the
                            | union set of customer and lateral
                            | peer ASes of ASx
ASRA_authorized(ASx, ASy) = /
                            \
                            |
                            | Else, "False"
                            \
~~~
{: #fig-authorization  title="ASRA authorization function"}


# AS_PATH Verification Algorithm {#sec-algo}

ASPA has precedence over ASRA. If ASPA asserts there is a relationship, that prevails over any ASRA assertion to the contrary. This means that ASRA cannot change anything about the up-ramp and down-ramp as determined by ASPA. ASRA can only influence fake link determination in-between the two apexes of the ramps.

The AS_PATH path verification procedure based on the combined use of ASRA and ASPA is formally specified as follows:

1. Run the ASPA-based AS_PATH verification algorithm (upstream or downstream as appropriate).

2. If the AS_PATH verification outcome is "Invalid", then the procedure stops.

3. Else, considering the same AS sequence {AS(N), AS(N-1), ..., AS(2), AS(1)} as in the ASPA-based algorithm, for any i from 1 to N-1, 

    - If both ASPA and ASRA of AS(i) exist, AND
  
    - if ASPA_authorized(AS(i), AS(i+1)) = "Not Provider+", AND

    - if ASPA_authorized(AS(i+1), AS(i)) = "No Attestation" or "Not Provider+", AND

    - if ASRA_authorized(AS(i), AS(i+1)) = False,

    - then, the AS_PATH verification outcome is changed from "Valid"/"Unknown" to "Invalid" and the procedure stops.

    - otherwise, continue.

4. Else, the "Valid" or "Unknown" AS_PATH verification outcome (as determined at Step 1) holds. 

ASRA record MUST be utilized in the direction of the Update propagation to check whether there exists a physical link between two adjacent unique ASes in an AS_PATH. ASRA will not be utilized in the direction opposite to the Update propagation, so as to avoid the misleading of bogus ASRA records. 

In the example of {{fig-path-shortened}}, suppose AS2 registers both ASPA and ASRA records that does not include AS6. If AS7 conducts ASRA-based AS_PATH verification, the fake link between AS2 and AS6 can be successfully detected according to AS2' ASRA record. Then, the path modification attacks can be detected and mitigated. 

# Security Considerations {#sec-security}

Since ASPA has precedence over ASRA, in some cases, the fake link cannot be detected. For example, in {{fig-path-shortened}}, if AS6 has ASPA records which maliciously include AS2 as a provider, the verification algorithm in {{sec-algo}} will fail to detect the fake link between AS6 and AS2. 

Such **possible* * fake links may exist when ASPA_authorized(AS(i+1),AS(i))="Provider+" but ASPA_authorized(AS(i),AS(i+1))="Not Provider+" and ASRA_authorized(AS(i),AS(i+1))=False. Then ISPs SHOULD report this anomaly disagreement between AS(i) and AS(i+1). 

Every ISP MUST periodically check ASPAs and make sure their AS is not included incorrectly in any ASPA. 

# IANA Considerations {#sec-iana}

TBD

# Acknowledgements

Much thanks to the contributions of Mingqing (Michael) Huang, Jeff Haas, etc. 

--- back

