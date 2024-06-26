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
...

--- abstract
Autonomous System Provider Authorization (ASPA) record is authorized in one direction and focuses on including all provider ASes while ignoring other neighboring ASes. While ASPA-based AS_PATH verification can efficiently detect and mitigate route leaks and some forged-origin or forged-path-segment hijacks, it may fail to detect malicious path modifications for routes that are received from transit providers.

This document proposes Autonomous System Relationship Authorization (ASRA). ASRA allows an AS to register not only provider ASes, but also other neighboring ASes with different relationships (e.g., customer ASes). ASRA can efficiently detect malicious path modifications by transit providers and enhance AS_PATH verification compared to ASPA. 


--- middle

# Introduction
Autonomous System Provider Authorization (ASPA) is a technique for protecting AS_PATHs in BGP updates {{I-D.ietf-sidrops-aspa-verification}}{{I-D.ietf-sidrops-aspa-profile}}. Each AS can register ASPA records (also ASPA objects) in the RPKI to states all its provider ASes. An AS can obtain other ASes' ASPA records and conduct AS_PATH verification based the records. ASPA-based AS_PATH verification can detect and mitigate route leaks violating the valley-free principle and route hijacks such as forged-origin or forged-path-segment attacks. 

As described in Section 12 of {{I-D.ietf-sidrops-aspa-verification}}, ASPA-based AS_PATH verification might fail to detect some malicious path modifications, especially for routes that are received from transit providers. The reason of the limitation is that ASPA allows an AS to register only providers and not all fake links in the AS_PATH can be detected without the full information of neighboring ASes. 

To solve the problem, this document proposes Autonomous System Relationship Authorization (ASRA). An AS can register ASRA records that include not only provider ASes, but also other neighboring ASes with different relationships (e.g., customer ASes). ASRA allows two-direction authorization and provides more detailed relationships with neighboring ASes. Thus, ASRA can efficiently enhance AS_PATH verification compared to ASPA. 

ASRA solution is completely compatible to ASPA solution. Without ASRA records, an AS can conduct ASPA-based AS_PATH verification. If there are some ASRA available, an enhanced verification can be carried out. 

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
This section proposes Autonomous System Relationship Authorization (ASRA) record. The record includes not only providers (i.e., what an ASPA record cares) but also other neighboring ASes with different relationships. The followings are the contents that an ASx can put in an ASRA record: 

- Mandatory contents

  - Provider-set (mandatory). Provider-set is the same as ASPA record and includes all the provider ASes of ASx. 

- Optional contents (highly recommended)

  - Neighbor-set (optional). Neighbor-set (or adjacency-set) includes all the neighboring ASes of ASx. This set provides the topology information of ASx without exposing the detailed relationships with the neighboring ASes. 

- Optional contents (recommended)

  - Customer-set (optional). Customer-set includes all the customer ASes of ASx. 

  - Peer-set (optional). Peer-set includes all the lateral peer ASes of ASx. 

To support the registration of the above content types, existing ASPA objects MAY need to be extended or new RPKI objects SHOULD be defined, which is TBD. Existing protocols {{I-D.ietf-sidrops-8210bis}} for synchronizing RPKI data from Relying Party (RP) to routers MAY also need to be extended for supporting the synchronization of ASRA data. 

The above content types can be registered together but there MUST be no conflicts between the contents. For example, an AS appearing in both the provider-set and peer-set means a conflict. The registration software of ASRA MUST have a check on the contents. If conflicts exist, the registration will fail. 

Any type of the content MUST include the complete set of the corresponding ASes. For example, customer-set MUST include every customer AS. 


<!-- ........................................................................ -->

# ASRA-based AS_PATH Verification
The upstream and downstream validation procedures of ASRA follow ASPA {{I-D.ietf-sidrops-aspa-verification}}. The main difference from ASPA's validation procedures is Hop-check() function. The Hop-check(ASx,ASy) in the ASRA-based AS_PATH verification is: 

- If ASx has no ASRA record (i.e., no record or only ASPA record), Hop-check(ASx, ASy) is run as that of ASPA's validation procedures. 

- If ASx has ASRA record that includes some types of the optional contents, Hop-check(ASx, ASy) will do the normal checking as ASPA while considering whether there are fake links in the AS_PATH. 

The enhanced Hop-check() function does not need to be called for each hop in the AS_PATH. It is needed only when there is a doubt at the apex of the path. Besides, ASRA record will be utilized in the direction of the Update propagation to confirm there exists a relationship between the two ASes at the apex. ASRA will not be utilized in the direction opposite to the Update propagation.

In the example of {{fig-path-shortened}}, suppose AS2 registers ASRA record that includes all its neighbor ASes. If AS7 conducts ASRA-based AS_PATH verification, the fake link between AS2 and AS6 can be successfully detected according to AS2' ASRA record. Then, the path modification attacks can be detected and mitigated. 

# Security Considerations {#sec-security}

TBD

# IANA Considerations {#sec-iana}

TBD

# Acknowledgements

TBD

--- back

