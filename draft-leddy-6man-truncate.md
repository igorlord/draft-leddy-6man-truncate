---
title: IPv6 Packet Truncation
docname: draft-leddy-6man-truncate-latest
date: {DATE}
category: std

ipr: trust200902
area: IPv6 Packet Truncation
workgroup: 6man
keyword: Internet-Draft
keyword: PMTUD
keyword: IPv6

stand_alone: yes
pi:
  rfcedstyle: yes
  toc: yes
  tocindent: yes
  sortrefs: yes
  symrefs: yes
  strict: yes
  comments: yes
  compact: yes
  inline: yes
  text-list-symbols: -o*+


author:
 -
    ins: J. Leddy
    name: John Leddy
    org: Unaffiliated
    email: john@leddy.net
 -
    ins: R. Bonica
    name: Ron Bonica
    org: Juniper Networks
    street: 2251 Corporate Park Drive
    city: Herndon
    code: 20171
    region: Virginia
    country: USA
    email: rbonica@juniper.net
 -
    ins: I. Lubashev
    name: Igor Lubashev
    org: Akamai Technologies
    street: 150 Broadway
    city: Cambridge
    code: 02142
    region: MA
    country: USA
    email: ilubashe@akamai.com

normative:

informative:
  IPv6Param:
    target: https://www.iana.org/assignments/ipv6-parameters/ipv6-parameters.xhtml#ipv6-parameters-2
    title: Internet Protocol Version 6 (IPv6) Parameters - Destination Options and Hop-by-Hop Options


--- abstract

This document defines IPv6 packet truncation procedures. These procedures make
Path MTU Discovery (PMTUD) more reliable. Upper-layer protocols can leverage
these procedures in order to take advantage of larger MTUs.

--- middle


Introduction     {#Intro}
============

An Internet path connects a source node to a destination node. A path can
contain links and routers.

Each link is constrained by the number of bytes that it can convey in a single
IP packet. This constraint is called the link Maximum Transmission Unit
(MTU). {{!IPv6=RFC8200}} requires every link to have an MTU of 1280 bytes or
greater. This value is called IPv6 minimum link MTU.

Likewise, each Internet path is constrained by the number of bytes that it can
convey in a single IP packet. This constraint is called the Path MTU (PMTU). For
any given path, the PMTU is equal to the smallest of its link MTUs.

IPv6 allows fragmentation at the source node only. If an IPv6 source node sends
a packet whose length exceeds the PMTU, an intermediate node will discard the
packet. In order to prevent this, IPv6 nodes can either:

* Refrain from sending packets whose length exceeds the IPv6 minimum link MTU.

* Maintain a running estimate of the PMTU and refrain from sending packets whose
  length exceeds that estimate.

In order to maintain a running estimate of the PMTU, IPv6 nodes can execute Path
MTU Discovery (PMTUD) {{!RFC8201}} procedures. In these procedures, the source
node produces an initial PMTU estimate. This initial estimate equals the MTU of
the first link along the path to the destination. It can be greater than the
actual PMTU.

Having produced an initial PMTU estimate, the source node sends packets to the
destination node. If one of these packets is larger than the actual PMTU, an
intermediate node will not be able to forward the packet through the next link
along the path. Therefore, the intermediate node discards the packet and sends
an Internet Control Message Protocol (ICMP) {{!RFC4443}} Packet Too Big
(PTB) message to the source node. The ICMP PTB message indicates the MTU of the
link through which the packet could not be forwarded. The source node uses this
information to refine its PMTU estimate.

PMTUD relies on the network to deliver ICMP PTB messages from the intermediate
node to the source node. If the network cannot deliver these messages, a
persistent black hole can develop. In this scenario, the source node sends a
packet whose length exceeds the PMTU. An intermediate node discards the packet
and sends an ICMP PTB message to the source. However, the network cannot deliver
the ICMP PTB message to the source. Therefore, the source node does not update
its PMTU estimate and it continues to send packets whose length exceeds the
PMTU. The intermediate node discards these packets and sends more ICMP PTB
messages to the source. These ICMP PTB messages are lost, exactly as previous
ICMP PTB messages were lost.

In some operational scenarios {{OpCon}}, networks cannot deliver ICMP PTB
messages from an intermediate node to the source node. Therefore, enhanced
procedures are required.

This document defines IPv6 packet truncation procedures. When an IPv6 source
node originates a packet, it executes the following procedure:

* Marks the packet as being eligible for truncation.

* Forward the packet towards its destination.

If an intermediate node cannot forward the packet because of an MTU issue, it
executes the following procedure:

* Detect that the packet is eligible for truncation.

* Send an ICMP PTB message to the source node, with the MTU field indicating the
  MTU of the link through which the packet could not be forwarded.

* Truncate the packet.

* Mark the packet as being truncated.

* Update the packet's upper-layer checksum (if possible).

* Forward the packet towards its destination.

When the destination node receives the packet, it executes the following
procedure:

* Detect that the packet has been truncated.

* Send an ICMP PTB message to the source node, with the MTU field
  indicating the length of the truncated packet.

* Discard the packet.

Both ICMP PTB messages, mentioned above, contain MTU information that the source
node can use to refine its PMTU estimate.

The procedures described herein prevent incomplete (i.e., truncated) data from
being delivered to upper-layer protocols. While IPv6 packet truncation may
facilitate new upper-layer procedures, upper-layer procedures are beyond the
scope of this document.

The procedures described herein make PMTUD more reliable by increasing the
probability that the source node will receive ICMP PTB feedback from a
downstream device. Even when the network cannot deliver ICMP PTB messages from
an intermediate router to a source node, it may be able to deliver an ICMP PTB
messages from the destination node to the source node.

However, the procedures described herein do not make PMTUD one hundred per cent
reliable. In some operational scenarios, the network cannot deliver any ICMP
messages to the source node, regardless of their origin.


Requirements Language    {#ReqLang}
=====================

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.


Operational Considerations   {#OpCon}
==========================

The packet truncation procedures described herein make PMTUD more resilient
when:

* The network can deliver ICMP messages from the destination node to the source
  node.

* The network cannot deliver ICMP messages from an intermediate node to the
  source node.

The following are common operational scenarios in which packet truncation
procedures can make PMTUD more resilient:

* The destination node has a viable route to the source node, but the
  intermediate node does not.

* The source node is protected by a firewall that administratively blocks all
  packets except for those from specified subnetworks. The destination node
  resides in one of the specified subnetworks, but the intermediate node does
  not.

* The source address of the original packet (i.e., the packet that elicited the
  ICMP message) was an anycast address. Therefore, the destination address of
  the ICMP message is the same anycast address.  In this case, an ICMP message
  from the destination node is likely to be delivered to the correct anycast
  instance. By contrast, an ICMP message from an intermediate node is less
  likely to be delivered to the correct anycast instance.

Packet truncation procedures do not make PMTUD more resilient when the network
cannot reliably deliver any ICMP messages to the source node. The following are
operational scenarios where the network cannot reliably deliver any ICMP PTB
messages to the source node:

* The source node is protected by a firewall that administratively blocks all
  ICMP messages.

* The source node is an anycast instance served by a load-balancer as defined in
  {{?RFC7690}}. The load-balancer does not implement the mitigations defined in
  {{?RFC7690}}.


IPv6 Destination Options
========================

This document defines the following IPv6 Destination options:

The Truncation Eligible Option   {#TuncEilig}
------------------------------

The Truncation Eligible option indicates that the packet is eligible for
truncation. It also indicates that the packet has not been truncated.

The Truncation Eligible option contains the following fields:

* Option Type - Truncation Eligible option. Value TBD by IANA. See Notes below.

* Opt Data Len - Length of Option Data, measured in bytes. MUST be equal to 0.

IPv6 packets that include the Fragment header MUST NOT include the Truncation
Eligible option.

IPv6 packets whose length is less than the IPv6 minimum link MTU SHOULD NOT
include the Truncation Eligible option.

The IPv6 Hop-by-hop Options header SHOULD NOT include the Truncation Eligible
option.

The IPv6 Destination Options header:

* MAY include a single instance of the Truncation Eligible option.

* SHOULD NOT include multiple instances of the Truncation Eligible option.

* MUST NOT include both the Truncation Eligible option and the Truncated Packet
  option {{TruncatedOpt}}.

NOTE 1: According to {{!IPv6}}, the highest-order two bits of the Option Type
        (i.e., the "act" bits) specify the action taken by a processing node
        that does not recognize Option Type. The required action is skip over
        this option and continue processing the header. Therefore, IANA is
        requested to assign this Option Type with "act" bits "00".

NOTE 2: According to {{!IPv6}}, the third-highest-order bit (i.e., the "chg"
        bit) of the Option Type specifies whether Option Data can change on
        route to the packet's destination. Because this option contains no
        Option Data, IANA can assign this Option Type without regard to the
        "chg" bit.


The Truncated Packet Option     {#TruncatedOpt}
---------------------------

The Truncated Packet option indicates that the packet has been truncated and is
eligible for further truncation.

The Truncated Packet option contains the following fields:

* Option Type - Truncated Packet option. Value TBD by IANA. See Notes below.

* Opt Data Len - Length of Option Data, measured in bytes. MUST be equal to 0.

IPv6 packets that include the Fragment header MUST NOT include the Truncated
Packet option.

IPv6 packets whose length is less than the IPv6 minimum link MTU MUST NOT
include the Truncated Packet option.

The IPv6 Hop-by-hop Options header SHOULD NOT include the Truncated Packet
option.

The IPv6 Destination Options:

* MAY include a single instance of the Truncated Packet option.

* SHOULD NOT include multiple instances of the Truncated Packet option.

* MUST NOT include both the Truncated Packet option and the Truncation Eligible
  option.

NOTE 1: According to {{!IPv6}}, the highest-order two bits of the Option Type
(i.e., the "act" bits) specify the action taken by a processing node that does
not recognize Option Type. The required action is to discard the packet and send
an ICMP Parameter Problem, Code 2, message to the packet's Source Address,
pointing to the unrecognized Option Type. Therefore, IANA is requested to assign
this Option Type with "act" bits "10".

NOTE 2: According to {{!IPv6}}, the third-highest-order bit (i.e., the "chg"
bit) of the Option Type specifies whether Option Data of that option can change
on route to the packet's destination. Because this option contains no Option
Data, IANA can assign this Option Type without regard to the "chg" bit.


Reference Topology     {#ReferenceTopology}
==================

~~~~~
 -----------           -----------           -----------           -----------
|   Upper   |         |           |         |           |         |   Upper   |
|   Layer   |         |           |         |           |         |   Layer   |
|           |         |           |         |           |         |           |
|    IP     |<------->|    IP     |<------->|    IP     |<------->|    IP     |
|   Layer   |   MTU   |   Layer   |   MTU   |   Layer   |   MTU   |   Layer   |
 -----------   9000    -----------   4000    -----------   1500    -----------
   Source                Router 1              Router 2            Destination
    Node                                                              Node
~~~~~
{: #RefTopoSect title="Reference Topology"}

The reference network topology contains a Source Node, intermediate nodes (i.e.,
Router 1, Router 2), and a Destination Node. The link that connects the Source
Node to Router 1 has an MTU of 9000 bytes. The link that connects Router 1 to
Router 2 has an MTU of 4000 bytes, and the link that connects Router 2 to the
Destination Node has an MTU of 1500 bytes. The PMTU between the Source Node and
the Destination Node is 1500 bytes.

This topology is used in examples throughout the document.


Truncation Procedures     {#signaling}
=====================

In the {{ReferenceTopology}}, the Source Node produces an initial estimate of
the PMTU between itself and the Destination Node. This initial estimate equals
the MTU of the first link on the path to the Destination Node (e.g., 9000
bytes).

The Source Node refrains from sending packets whose length exceeds the
above-mentioned estimate. However, the above-mentioned estimate is significantly
larger than the actual PMTU (1500 bytes). Therefore, the Source Node may send
packets whose length exceeds the actual PMTU.

At some time in the future, an upper-layer protocol on the Source Node causes
the IP layer to emit a packet. The packet contains a Destination Options header
and the Destination Options header contains a Truncation Eligible option. The
total packet length, including all headers and the payload, is 1350
bytes. Because the total packet length is less than the actual PMTU, this packet
can be delivered to the Destination Node without encountering any MTU
issues.

The IP layer on the Source Node forwards the packet to the Router 1, Router 1
forwards the packet to Router 2, and the Router 2 forwards the packet to the
Destination Node. The IP layer on the Destination Node examines the Destination
Options header and finds the Truncation Eligible option. The Truncation Eligible
option requires no action by the Destination Node. Therefore, the Destination
Node processes the next header and delivers the packet to an upper-layer
protocol.

Subsequently, the same upper-layer protocol on the Source Node causes the IP
layer to emit another packet. This packet is identical to the first, except that
the total packet length is 2000 bytes. Because the packet length is greater than
the actual PMTU, this packet cannot be delivered without encountering an MTU
issue.

The IP layer on the Source Node forwards the packet to Router 1.  Router 1
forwards the packet to Router 2, but the Router 2 cannot forward the packet
because its length exceeds the MTU of the next link in the path (i.e., 1500
bytes). Because an MTU issue has been encountered, Router 2 examines the
Destination Options header, searching for either a Truncation Eligible option or
a Truncated Packet option.  (Normally, the Router 2 would ignore the Destination
Options header).

Because Router 2 finds one of the above-mentioned options, it:

* Sends an ICMP PTB message to the Source Node. The ICMP PTB message contains an
  MTU field indicating the MTU of the next link in the path (i.e. 1500 bytes).

* Truncates the packet, so that its total length equals the MTU of the next link
  in the path.

* Updates the IPv6 Payload Length.

* Overwrites all instances of the Truncation Eligible option with a Truncated
  Packet option.

* Updates the upper-layer checksum (if possible)

* Forwards the packet to the Destination Node.

The IP layer on the Destination Node receives the packet and examines the
Destination Options header. Because it finds the Truncated Packet option, it
discards the packet and sends an ICMP PTB message to the Source Node. The MTU
field in the ICMP PTB message represents the length of the received packet.

When the Source Node receives the ICMP PTB message, it updates its PMTU
estimate, as per {{!RFC8201}}.

NOTE: A packet can be truncated multiple times. In the reference topology,
assume that the Source Node sends a 5000 byte packet containing a Truncation
Eligible option to the Destination Node. Using the above-mentioned procedures,
Router 1 truncates the packet to 4000 bytes and Router 2 truncates it again,
this time to 1500 bytes. The Destination Node receives the packet, discards it,
and sends an ICMP PTB message to the Source Node. The MTU field in the ICMP PTB
message represents the length of the received packet (i.e., 1500 bytes).


Backwards Compatibility
=======================

The {{signaling}} of this document assumes that all nodes recognize the
Truncation Eligible and Truncated Packet options. This section explores
backwards compatibility issues, where one or more nodes do not recognize the
above-mentioned options.

An intermediate node that does not recognize the above-mentioned options behaves
exactly as described in {{!IPv6}}. When it receives a packet that does not cause
an MTU issue, it processes the packet. When it receives a packet that causes an
MTU issue, it discards the packet and sends an ICMP PTB message to the source
node. In neither case does the intermediate node examine the Destination Options
header or truncate the packet.

A destination node that does not recognize the Truncation Eligible option also
behaves exactly as described in {{!IPv6}}.  When it receives a packet that
contains the Truncation Eligible option, its behavior is determined by the
highest-order two bits of the Option Type (i.e., the "act" bits). Because the
"act" bits are equal to "00", the destination node skips over the option and
continues to process the packet. This is exactly what the destination node would
have done if it had recognized the Truncation Eligible option.

A destination node that does not recognize the Truncated Packet option also
behaves exactly as described in {{!IPv6}}.  When it receives a packet that
contains the Truncated Packet option, its behavior is determined by the
highest-order two bits of the Option Type (i.e., the "act" bits). Because the
"act" bits are equal to "10", the destination node discards the packet and sends
an ICMP Parameter Problem, Code 2, message to the packet's Source Address,
pointing to the Truncated Packet option. The destination node does not emit an
ICMP PTB message.

The source node takes appropriate action when it receives the ICMP Parameter
Problem message.


Checksum Considerations
=======================

When an intermediate node truncates a packet, it SHOULD update the upper-layer
checksum, if possible. This is desirable because it increases the probability
that the truncated packet will be delivered to the destination node.

Stateful middleboxes residing downstream of the intermediate node may attempt to
validate the upper-layer checksum. If the validation fails, they may discard the
packet without sending an ICMP message.


Invalid Packet Types
====================

The following packet types are invalid:

* Packets that contain the Fragment header and the Truncation Eligible option.

* Packets that contain the Fragment header and the Packet Truncated option.

* Packets that contain the Truncation Eligible option and the Packet Truncated
  option.

* Packets that specify an Option Data Length greater than 0 in the Truncation
  Eligible option.

* Packets that specify an Option Data Length greater than 0 in the Truncated
  Packet option.

* Packets that have a total length less than the IPv6 minimum link MTU and
  contain the Packet Truncated option.

If an intermediate node cannot forward one of the above-mentioned packets
because of an MTU issue, its behavior is as described in {{!IPv6}}. The
intermediate node discards the packet and sends an ICMP PTB message to the
source node. It does not truncate or forward the packet.

When the destination node receives one of the above-mentioned packets, it MUST:

* Discard the packet

* Send an ICMP Parameter Problem, Code 2, message to the packet's Source
  Address, pointing to the first invalid option.

The destination node MUST NOT send an ICMP PTB message.


Network Considerations
======================

The procedures described herein rely upon the networks ability:

* To convey packets that contain destination options from the source node to the
  destination node.

* To convey ICMP Parameter Problem messages in the reverse direction.

Operational experience {{?RFC7872}} reveals that a significant number of
networks drop packets that contain IPv6 destination options. Likewise, many
networks drop ICMP Parameter Problem messages.

{{?I-D.bonica-6man-unrecognized-opt}} describes procedures that upper-layer
protocols can execute to verify that the above-mentioned requirements are
satisfied. Upper-layer protocols can execute these procedures before emitting
packets that contain the Truncation Eligible option.

Encapsulating Security Payload Considerations
=============================================

An IPv6 packet can contain both:

* An Encapsulating Security Payload (ESP) header {{!RFC4303}}.

* Truncation options (i.e., the Truncation Eligible or Truncated Packet
  options).

In this case, the packet MUST contain a Destination Options header that precedes
the ESP. That Destination Options header contains the truncation options and is
not protected by the ESP. The packet MAY also contain another Destination
Options header the follows the ESP.  That Destination Options header is
protected by the ESP and MUST NOT contain the truncation options.

As per {{!RFC4303}}, a packet can contain two Destination Options headers one
preceding the ESP and one following the ESP.


Extension Header Considerations
===============================

According to {{IPv6}}, the following IPv6 extension headers can contain options:

* The Hop-by-hop Options header.

* The Destination Options header.

The Hop-by-hop option can be examined by each node along the path to a packet's
destination. Destination options are examined by the destination node
only. However, {{?RFC2473}} provides a precedent for intermediate nodes
examining the Destination options on an exception basis. (See the Tunnel
Encapsulation Limit.)

The truncation options described herein are examined by:

* Intermediate nodes, on an exception basis (i.e, when the packet cannot be
 forwarded due to MTU issues).

* The Destination node.

Therefore, the above-mentioned options can be processed most efficiently when
they are contained by the Destination Option header.  When contained by the
Destination Options header, the above-mentioned options are examined by
intermediate nodes on an exception basis, only when they are relevant. If
contained by the Hop-by-hop Options header, they are always examined by
intermediate nodes, even when they are irrelevant.


Security Considerations    {#Security}
=======================

PMTUD is vulnerable to ICMP PTB forgery attacks. The procedures described herein
do nothing to mitigate that vulnerability.

The procedures described herein are susceptible to a new variation on that
attack, in which an attacker forges a truncated packet. In this case, the
attackers cause the Destination Node to produce an ICMP PTB message on their
behalf. To some degree, this vulnerability is mitigated, because the Destination
Node will not emit an ICMP PTB message in response to a truncated packet whose
length is less than the IPv6 minimum link MTU.

In order to mitigate denial of service attacks, intermediate nodes MUST rate
limit the number of packets that they truncate per second.


IANA Considerations
===================

IANA is requested to allocate the following codepoints from the Destination
Options and Hop-by-hop Options registry {{IPv6Param}}.

* Truncation Eligible ("act-bits" are "00. "chg-bit" can be either 0 or 1.)

* Truncated Packet ("act-bits" are "10". "chg-but can be either 0 or 1.)


Acknowledgements   {#Acknowledgements}
================

Special thanks to Mike Heard, Geoff Huston, Joel Jaeggli, Tom Jones, Andy Smith,
and Jinmei Tatuya who reviewed and commented on this document.

--- back
