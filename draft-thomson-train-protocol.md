---
title: "Transparent Rate Adaptation Indications for Networks (TRAIN) Protocol"
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
 -
    fullname: Christian Huitema
    org: Private Octopus Inc.
    email: huitema@huitema.net
 -
    fullname:
      :: 奥 一穂
      ascii: Kazuho Oku
    org: Fastly
    email: kazuhooku@gmail.com


normative:
  QUIC: RFC9000
  INVARIANTS: RFC8999

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

The Transparent Rate Adaptation Indications for Networks (TRAIN) protocol is
negotiated by QUIC endpoints.  This protocol provides a means for network
elements to signal the maximum available sustained throughput, or rate limits,
for flows of UDP datagrams that transit that network element to a QUIC endpoint.


# Overview

QUIC endpoints can negotiate the use of TRAIN by including a transport parameter
({{tp}}) in the QUIC handshake.  Endpoints then occasionally coalesce a TRAIN
packet with ordinary QUIC packets that they send.

Network elements that have rate limiting policies can detect flows that include
TRAIN packets.  The network element can indicate a maximum sustained throughput
by modifying the TRAIN packet as it transits the network element.

~~~ aasvg
+--------+    +---------+     +----------+
|  QUIC  |    | Network |     |   QUIC   |
| Sender |    | Element |     | Receiver |
+---+----+    +----+----+     +----+-----+
    |              |               |
    +--- TRAIN --->|   TRAIN+rate  |
    |    +QUIC     +---- +QUIC --->|
    |              |               |  Validate QUIC packet
    |              |               |  and record rate
    |              |               |
~~~

QUIC endpoints that receive modified TRAIN packets observe the indicated
version, process the QUIC packet, and then record the indicated rate.

Indicated rate limits apply only in a single direction.  Separate indications
can be sent for the client-to-server direction and server-to-client direction.
The indicated rates do not need to be the same.

Indicated rate limits only apply to the path on which they are received.  A
connection that migrates or uses multipath {{?QUIC-MP=I-D.ietf-quic-multipath}}
cannot assume that rate limit indications from one path apply to new paths.


# Applicability

This protocol only works for flows that use the TRAIN packet ({{packet}}).

The protocol requires that packets are modified as they transit a
network element, which provides endpoints strong evidence that the network
element has the power to drop packets; though see {{security}} for potential
limitations on this.

The rate limit signal that this protocol carries is independent of congestion
signals, limited to a single path and UDP packet flow, unidirectional, and
strictly advisory.

## Independent of Congestion Signals

Rate limit signals are not a substitute for congestion feedback.  Congestion
signals, such as acknowledgments, provide information on loss, delay, or ECN
markings {{?ECN=RFC3168}} that indicate the real-time condition of a network
path.  Congestion signals might indicate a throughput that is different from the
signaled rate limit.

Endpoints cannot assume that a signaled rate limit is achievable if congestion
signals indicate otherwise.  Congestion could be experienced at a different
point on the network path than the network element that indicates a rate limit.
Therefore, endpoints need to respect the send rate constraints that are set by a
congestion controller.

## On Path Signal

Modifying a packet does not prove that the rate limit that is indicated would be
achievable.  A signal that is sent for a specific flow might be enforced at a
different scope.  For instance, limits might apply at a network subscription
level, such that multiple flows receive the same signal.  Nor does a signal
prove that a higher rate would not be successful.  Endpoints that receive this
signal therefore need to treat the information as advisory.

## Per-Flow Signal

The same UDP address tuple might be used for multiple QUIC connections.  A
single signal might be lost or only reach a single application endpoint.
Network elements that signal about a flow might choose to send additional
signals, using connection IDs to indicate when new connections could be
involved.

## Undirectional Signal

The endpoint that receives a rate limit signal is not the endpoint that might
adapt its sending behavior as a result of receiving the signal.  An endpoint
might need to communicate the value it receives to its peer in order to ensure
that the limit is respected.  This document does not define how that signaling
occurs as this is specific to the application in use.

## Advisory Signal

As an advisory signal, network elements cannot assume that endpoints will
respect the signal.  Though this might reduce the need for more active rate
limiting, how rate limit enforcement is applied is a matter for network policy.

The time and scope over which a rate limit applies is not specified.  The
effective rate limit might change without being signaled.  The signaled limit
can be assumed to apply to the flow of packets on the same UDP address tuple for
the duration of that flow.  Rate limiting policies often apply on the level of a
device or subscription, but endpoints cannot assume that this is the case.  A
separate signal can be sent for each flow.


# Conventions and Definitions

{::boilerplate bcp14-tagged-bcp}


# TRAIN Packet {#packet}

A TRAIN packet is a QUIC long header packet that follows the QUIC invariants;
see {{Section 5.1 of INVARIANTS}}.

{{fig-train-packet}} shows the format of the train packet using the conventions
from {{Section 4 of INVARIANTS}}.

~~~ artwork
TRAIN Packet {
  Header Form (1) = 1,
  Reserved (1),
  Rate Signal (6),
  Version (32) = 0xTBD,
  Destination Connection ID Length (8),
  Destination Connection ID (0..2040),
  Source Connection ID Length (8),
  Source Connection ID (0..2040),
}
~~~
{: #fig-train-packet title="TRAIN Packet Format"}

The most significant bit (0x80) of the packet indicates that this is a QUIC long
header packet.  The next bit (0x40) is reserved and can be set according to
{{!QUIC-BIT=RFC9287}}.

The entire payload of the TRAIN packet is carried in the Rate Signal field that
forms the low 6 bits (0x3f) of the first byte.  Values for this field are
described in {{rate-signal}}.

This packet includes a Destination Connection ID field that is set to the same
value as other packets in the same datagram; see {{Section 12.2 of QUIC}}.

The Source Connection ID field is set to match the Source Connection ID field of
any packet that follows.  If the next packet in the datagram does not have a
Source Connection ID field, which is the case for packets with a short header
({{Section 5.2 of INVARIANTS}}), the Source Connection ID field is empty.

TRAIN packets SHOULD be included as the first packet in a datagram.  This is
necessary in many cases for QUIC versions 1 and 2 because packets with a short
header cannot precede any other packets.


## Rate Signals {#rate-signal}

{:aside}
> Note: The exact set of rates that are included is subject to negotiation.
> We should aim to find typical rate limits that are used in real networks
> and by real applications.

The Rate Signal field of a TRAIN packet is set to 0xTBD when sent by a QUIC
endpoint.  Receiving value indicates that there is no rate limit in place or
that the TRAIN protocol is not supported by network elements on the path.

The values from {{table-rates}} are specified to carry a the corresponding rate
limit signal.

| Rate Signal | Rate Limit |
|:------------|:-----------|
| 0xTBD       | X Mbps     |
| 0xTBD       | X Mbps     |
| 0xTBD       | X Mbps     |
| 0xTBD       | X Mbps     |
| 0xTBD       | X Mbps     |
| 0xTBD       | X Mbps     |
| 0xTBD       | X Mbps     |
| 0xTBD       | X Mbps     |
{: #table-rates title="Table of Rate Signals and Rate Limits"

All other values are reserved.

{:aside}
> TODO: Work out how reserved values can be used, if at all.


## Processing TRAIN Packets

Processing a TRAIN packet involves reading the value from the Rate Signal field.
However, this value MUST NOT be used unless another packet from the same
datagram is successfully processed.  Therefore, a TRAIN packet always needs to
be coalesced with other QUIC packets.

A TRAIN packet is defined by the use of the longer header bit (0x80 in the first
byte) and the TRAIN protocol version (0xTBD in the next four bytes).  A TRAIN
packet MAY be discarded, along with any packets that come after it in the same
datagram, if the Source Connection ID is not consistent with those coalesced
packets, as specified in {{packet}}.

A TRAIN packet MUST be discarded if the Destination Connection ID does not match
one recognized by the receiving endpoint.


# Negotiating TRAIN {#tp}

A QUIC endpoint indicates that it is willing to receive TRAIN packets by
including the train_supported transport parameter (0xTBD).

This transport parameter is valid for QUIC versions 1 {{QUIC}} and 2
{{!QUICv2=RFC9369}} and any other version that recognizes the versions,
transport parameters, and frame types registries established in {{Sections 22.2,
22.3, and 22.4 of QUIC}}.


# Deployment

QUIC endpoints can enable the use of the TRAIN protocol by sending TRAIN packets
{{packet}}.  Network elements then apply or replace the Rate Signal field
({{apply}}) according to their policies.


## Applying Rate Limit Signals {#apply}

A network element detects a TRAIN packet by observing that a packet has a QUIC
long header and the TRAIN protocol version of 0xTBD.

A network element then conditionally replaces the six bits of the Rate Signal
field with a value of its choosing.

A network element might receive a packet that already includes a rate limit
signal.  If the network element wishes to signal a lower rate limit, they can
replace the Rate Signal field with a different value that indicates the lower
limit.  If the network element wishes to signal a higher rate limit, they leave
the Rate Signal field alone, preserving the signal from the network element that
has a lower rate limit policy.

The following pseudocode indicates how a TRAIN packet might be detected and
replaced.  This assumes a target rate that is preconfigured and a means of
comparing the rate signal in the packet to the target rate signal.

~~~ pseudocode
target_rate_signal = rate_signal_for(target_rate)

is_long = packet[0] & 0x80 == 0x80
is_train = compare(packet[1..5], TRAIN_VERSION)
if is_long and is_train:
  packet_rate_signal = packet[0] & 0x3f
  if target_rate_signal.is_lower_than(packet_rate_signal):
    packet[0] = packet[0] & 0xc0 | target_rate_signal
~~~

{:aside}
> TODO: When defining the signal values, make this comparison easy.


## Providing Opportunities to Apply Rate Limit Signals {#extra-packets}

Endpoints that wish to offer network elements the option to add rate limit
markings can send TRAIN packets at any time.  This is a decision that a sender
makes when constructing datagrams, so TRAIN packets can be sent as frequently as
the application requires.

Endpoints MUST send any TRAIN packet they send as the first packet of a
datagram, coalesced with additional packets.  An endpoint that receives and
discards a TRAIN without also successfully processing another packets from the
same datagram SHOULD ignore any rate limit signal.  Such a datagram might be
entirely spoofed.


# Security Considerations {#security}

The modification of packets provides endpoints proof that a network element is
in a position to drop datagrams and thereby enforce the indicated rate limit.
{{extra-packets}} states that endpoints only accept signals if the datagram
contains a packet that it accepts to prevent an off-path attacker from inserting
spurious rate limit signals.

Some off-path attackers may be able to both
observe traffic and inject packets. Attackers with such capabilities could
observe packets sent by an endpoint, create datagrams coalescing an
arbitrary TRAIN packet and the observed packet, and send these datagrams
such that they arrive at the peer endpoint before the original
packet. Spoofed packets that seek to advertise a higher limit
than might otherwise be permitted also need to bypass any
rate limiters. The attacker will thus get arbitrary TRAIN packets accepted by
the peer, with the result being that the endpoint receives a false
or misleading rate limit.

The recipient of a rate limit signal therefore cannot guarantee that
the signal was generated by an on-path network element. However,
the capabilities required of an off-path attacker are substantially
similar to those of on path elements.

The actual value of the rate limit signal is not authenticated.  Any signal
might be incorrectly set in order to encourage endpoints to behave in ways that
are not in their interests.  Endpoints are free to ignore limits that they think
are incorrect.  The congestion controller employed by a sender provides
real-time information about the rate at which the network path is delivering
data.

Similarly, if there is a strong need to ensure that a rate limit is respected,
network elements cannot assume that the signaled limit will be respected by
endpoints.


# Privacy Analysis {#privacy}

Any network element that can observe the content of that packet can read the
rate limit that was applied.  Any signal is visible on the path, from the point
at which it is applied to the point at which it is consumed at an endpoint.

The signals that this protocol carries are only capable of expressing a small
number of discrete rate limits. However, revealing any information about rate
limits could have privacy implications.  Different rate limits are often tied to
different pricing schedules for access networks. Rate limit signals could
therefore act as a low quality predictor for sensitive traits like socioeconomic
status.

The analysis that follows makes some simplifying assumptions:

* Rate adaptation is assumed to be predominantly in the server-to-client
  direction, for things like on-demand video.  However, there are significant
  applications where data flows in the other direction, such as real-time
  communications.

* Access networks are assumed to be primary point at which rate limit signals
  are applied.  The details of client access is assumed to be more sensitive
  than that of a server.

* This also assumes that information carried in rate limit signals are also
  observable by endpoints through observation of network throughput, especially
  when limits apply to the network through which an endpoint connects to the
  Internet.

This last point largely removes endpoints and endpoint applications from
consideration with respect to privacy.  That limits are observable to endpoints
somewhat mitigates concerns about indirectly revealing socioeconomic status.

Under these assmptions, different considerations apply to clients and servers.

## Client Privacy

Signals about the downlink capacity that are applied by an access provider are
only visible to clients and network elements in the access network.

Where the scope of the rate limit is a single person, as can be the case for
mobile subscriptions, downlink signals from an access network carries very
little privacy risk.  The signal is only visible to access network nodes and the
destination endpoint.  Though eavesdropping might reveal that information, an
eavesdropper would also able to infer rate limits by other means.

Downlink signals reveal information about the service, not specific clients, so
a client could conclude that accepting signals is a reasonable privacy risk.

Downlink rate limits for a shared service would be available any client on that
network unless a firewall took specific steps to remove signals.  Such a
firewall could not prevent the application of rate limit signals, which would be
visible to network elements prior to the removal of the signal. Removing signals
would also prevent endpoints from consuming the TRAIN signal.

Uplink capacity from an access network is visible to the entire network path,
plus any remote endpoint. For the predominant uses of network capacity, where
downlink usage is most likely to be subject to rate adaptation, uplink rate
limit signals might have reduced value. The need to send TRAIN packets on an
uplink therefore needs to be balanced against any value that might be obtained.

For applications where downlink capacity is most likely to be important, clients
might choose to advertise support for this protocol, but avoid sending TRAIN
packets. This limits the possibility that any rate limits associated with their
access service are revealed to arbitrary servers.

Even when an application requires rate adaptation in two directions -- such as
real-time video conferencing -- the privacy risk associated with sending TRAIN
packets needs to be weighed against the value of the information gained.

## Server Privacy

Conversely, a server might have fewer concerns about the privacy of its own
service rate limits, in either direction. For a server uplink in particular (a
client downlink), information that might inform rate adaptation is most likely
to be useful.

Servers that support applications with bidirectional exchange patterns that
might benefit from understanding rate limits might consume rate limit signals
when clients enable the feature. However, servers have to operate correctly even
when clients act to protect the privacy of their service configuration.

## Conditional Enablement

The presence or absence of TRAIN packets could carry information, even when
networks do not apply rate limit signals.  Making a decision to accept or send
TRAIN packets conditional on client or server state leads to leaking information
about that state.

Clients and servers can avoid leaking information through the availability of
the feature by enabling the signaling unconditionally.  While certain actions
might only be taken by applications that understand and can use rate limit
signals, the presence of signals would not carry any significant information.

QUIC implementations are therefore encouraged to make the feature available
unconditionally.  Endpoints might send TRAIN packets whenever a peer can accept
them.

False signals could create some uncertainty about the true rate
limits. Endpoints could therefore send TRAIN packets with false rate limit
signals, which can be filtered out of any feedback.

Avoiding that being conditional ensures that those cases where the signals are
actively used are harder to distinguish from other activity.  Endpoints that do
act on a rate limit signal might still be detected. For an observer, any change
in send rate that results might be attributable to changes in application state
or a reaction to congestion signals.


# IANA Considerations {#iana}

This document registers a new QUIC version ({{iana-version}} and a QUIC
transport parameter {{iana-tp}}.


## TRAIN Version {#iana-version}

This document registers the following entry to the "QUIC Versions" registry
maintained at <https://www.iana.org/assignments/quic>.

Value:
: 0xTBD

Status:
: permanent

Specification:
: This document

Change Controller:
: IETF (iesg@ietf.org)

Contact:
: QUIC Working Group (quic@ietf.org)

Notes:
: TRAIN Protocol
{: spacing="compact"}


## train_supported Transport Parameter {#iana-tp}

This document registers the grease_quic_bit transport parameter in the "QUIC
Transport Parameters" registry established in {{Section 22.3 of QUIC}}. The
following fields are registered:

Value:
: 0xTBD

Parameter Name:
: train_supported

Status:
: Permanent

Specification:
: This document

Date:
: This date

Change Controller:
: IETF (iesg@ietf.org)

Contact:
: QUIC Working Group (quic@ietf.org)

Notes:
: (none)
{: spacing="compact"}

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
