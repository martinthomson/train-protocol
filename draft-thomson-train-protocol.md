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

The time and scope over which a rate limit applies is not specified.  Rate
limits might change without being signaled.  The signaled limit can be assumed
to apply to the flow of packets on the same UDP address tuple for the duration
of that flow.  Rate limiting policies often apply on the level of a device or
subscription, but endpoints cannot assume that this is the case.  A separate
signal can be sent for each flow.

The same UDP address tuple might be used for multiple QUIC connections.  A
single signal might be lost or only reach a single application endpoint.
Network elements that signal about a flow might choose to send additional
signals, using connection IDs to indicate when new connections could be
involved.

The endpoint that receives a rate limit signal is not the endpoint that might
adapt its sending behavior as a result of receiving the signal.  An endpoint
might need to communicate the value it receives to its peer in order to ensure
that the limit is respected.  This document does not define how that signaling
occurs as this is specific to the application in use.


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

Packets for each of these versions is processed exactly as they would be for
QUIC version 1 or QUIC version 2.  In particular, when a packet with one of
these versions is received, the value of the version field is replaced by the
version number for either QUIC version 1 (0x00000001) or QUIC version 2
(0x6b3343cf).  Thus, the packet protection mechanism used by the canonical QUIC
version is identical and AEAD decryption would fail if the limit-specific
version number is left in place.

For example, the example Retry packet from {{Appendix A.4 of RFC9369}} might be
modified to indicate a rate limit of TBD to produce a packet of:

~~~
cfXXXXXXXX0008f067a5502a4262b574 6f6b656ec8646ce8bfe33952d9555436
65dcc7b6
~~~

But the 4 bytes of the version number will be restored to 0x6b3343cf before
attempting to process the packet further.


# Deployment

Rate limiting signals that are applied in this way do not guarantee that the
signal is successfully received.

The primary cost associated with the application of rate limit signals is the
potential for the packet to be dropped as a consequence of the changed version.
An endpoint that does not recognize these new versions will treat the
modification as damage and discard the packet.

{:aside}
> TODO: do we need to define a new QUIC version where support for that version
> also indicates support for this feature?  That might allow endpoints to know
> that their long header packets, if modified, won't get dropped.

{:aside}
> TODO: determine whether we want a frame that solicits the sending of a long
> header packet.


## Changing Rate Limit Signals

A network element might receive a packet that already includes a rate limit
signal.  If the network element wishes to signal a lower rate limit, they can
replace the Version field with a different value that indicates the lower limit.
If the network element wishes to signal a higher rate limit, they leave the
signal alone, preserving the signal from the network element that has a lower
rate limit policy.


## Providing Opportunities to Attach Rate Limit Signals {#bogus-long}

A UDP that contains QUIC packets does not always use the long header.  For
example, a connection can migrate to a new path without using anything other
than a short header on that new path.  Endpoints that wish to offer network
elements the option to add rate limit markings can send long header packets

Long header packets might indicate the use of packet protection keys that are
long discarded by an endpoint.  Endpoints that receive long header packets can
process rate limit signals before attempting to remove packet protection or
before discarding these packets.

An endpoint that send long header packets for this purpose SHOULD send those
packets as the first packet of a datagram, including additional valid packets.
An endpoint that receives and discards a packet when there are no valid packets
in the datagram SHOULD ignore any rate limit signal.  Such a datagram might be
entirely spoofed.



# Security Considerations

The modification of packets provides endpoints proof that a network element is
in a position to drop datagrams and thereby enforce the indicated rate limit.
{{bogus-long}} suggests that endpoints only accept signals if the datagram
contains a packet that it accepts to prevent an off-path attacker from inserting
spurious rate limit signals.

The actual value of the rate limit signal is not authenticated.  Any signal
might be incorrectly set in order to encourage endpoints to behave in ways that
are not in their interests.  Endpoints are free to ignore limits that they think
are incorrect.

Similarly, if there is a strong need to ensure that a rate limit is respected,
network elements cannot assume that the signaled limit will be respected by
endpoints.



# IANA Considerations

This document registers many new QUIC versions.

{:aside}
> TODO: actually pick some numbers and register them.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
