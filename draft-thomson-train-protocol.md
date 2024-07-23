---
title: "Transparent Rate Adaptation Indicators for Network Elements (TRAIN)"
abbrev: "TRAIN Protocol"
category: info

docname: draft-thomson-train-protocol-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
# area: AREA
# workgroup: WG Working Group
keyword:
 - chugga chugga chugga chugga
 - mitm
venue:
#  group: WG
#  type: Working Group
#  mail: WG@example.com
#  arch: https://example.com/WG
  github: "martinthomson/train-protocol"
  latest: "https://martinthomson.github.io/train-protocol/draft-thomson-train-protocol.html"

author:
 -
    fullname: Martin Thomson
    organization: Mozilla
    email: mt@lowentropy.net

normative:

informative:


--- abstract

On-path network elements can sometimes be configured to apply rate limits to
flows that pass them.  This document describes a method for signaling to
endpoints that rate limiting policies are in force and approximately what that
rate limit is.


--- middle

# Introduction

Many access networks limit the maximum data rate that attached devices are able
to attain.  This is often done without any indication to the applications
running on devices.  The result can be that application performance is degraded,
as the manner in which rate limits are enforced can be incompatible with the
rate estimation or congestion control algorithms used at endpoints.

Having the network indicate what its rate limiting policy is, in a way that is
accessible to endpoints, might allow applications to use this inforation when
adapting their send rate.


# Overview

Network elements that have rate limiting policies can use this protocol on any
QUIC flow containing long header packets.  The network element indicates an
approximate rate limit by replacing the version field of the QUIC packet.

A number of QUIC versions are reserved for each canonical QUIC version. Each
version indicates a specific rate limit.

~~~ aasvg
+--------+    +---------+    +----------+
|  QUIC  |    | Network |    |   QUIC   |
| Sender |    | Element |    | Receiver |
+---+----+    +----+----+    +----------+
    |              |              |
    +---- v1 ----->|              |
    |              +----- vx ---->|
    |              |              |  vx ==> rate limit = y
    |              |              |     ==> v1
    |              |              |
~~~

QUIC endpoints that support this protocol and receive a long header packet with
one of these QUIC versions first recover the intended rate limit by mapping the
version number to a specific rate.  The packet is then processed as normal by
replacing the rate limit indication with the original or canonical version.

Indicated rate limits apply only in a single direction.  Separate indications
can be sent for the client-to-server direction and server-to-client direction.
The indicated rates do not need to be the same.

Indicated rate limits only apply to the path on which they are received.  A
connection that migrates or uses multipath {{?QUIC-MP=I-D.ietf-quic-multipath}}
cannot assume that rate limit indications from one path apply to new paths.


# Applicability

This protocol only works for flows that use specific QUIC versions and only
where those flows include packets using the long header {{Section 5.1 of
!RFC8999}}.

The information provided to endpoints (or applications at those endpoints) is
advisory.  The protocol requires that packets are modified as they transit a
network element, which provides endpoints proof that the network element has the
power to drop packets.  However, it does not prove that the rate limit that is
indicated would either be successful; nor does it prove that a higher rate would
not be successful.  Endpoints that receive this signal therefore need to treat
the information as advisory.

As an advisory signal, network elements cannot assume that endpoints will
respect the signal.  Though this might reduce the need for more active rate
limiting, whether rate limit enforcement is necessary is a matter for network
policy.

# Conventions and Definitions

{::boilerplate bcp14-tagged-bcp}


# Version Mappings

This document defines version number mappings for QUIC version 1 {{!RFC9000}}
and version 2 {{!RFC9369}}.  Rate limit mappings are defined for a limited
number of send rates, as shown in {{table-rates}}.

{:aside}
> Note: The exact set of rates that are included is subject to negotiation.

| Rate Limit | Version 1 | Version 2 |
|--:|:--|:--|
| 1Mbps      | 0xTBD | 0xTBD |
| 2Mbps      | 0xTBD | 0xTBD |
| 3Mbps      | 0xTBD | 0xTBD |
| 4Mbps      | 0xTBD | 0xTBD |
| 5Mbps      | 0xTBD | 0xTBD |
| 7Mbps      | 0xTBD | 0xTBD |
| 10Mbps      | 0xTBD | 0xTBD |
| 15Mbps      | 0xTBD | 0xTBD |
| 20Mbps      | 0xTBD | 0xTBD |
| 50Mbps      | 0xTBD | 0xTBD |
{: #table-rates title="Version and Rate Limit Mappings"}




# Security Considerations



# IANA Considerations

This document registers many new QUIC versions.

{:aside}
> TODO: actually pick some numbers and register them.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
