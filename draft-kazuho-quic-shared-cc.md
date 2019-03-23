---
title: Shared Congestion Controller for QUIC
docname: draft-kazuho-quic-shared-cc-latest
category: std

ipr: trust200902
area: Transport
workgroup: QUIC
keyword: Internet-Draft

stand_alone: yes
pi: [toc, docindent, sortrefs, symrefs, strict, compact, comments, inline]

author:
  -
    ins: K. Oku
    name: Kazuho Oku
    org: Fastly
    email: kazuhooku@gmail.com

normative:
  RFC2119:
  QUIC-TRANSPORT:
    title: "QUIC: A UDP-Based Multiplexed and Secure Transport"
    seriesinfo:
      Internet-Draft: draft-ietf-quic-transport-16
    date: 2018-10-23
    author:
      -
        ins: J. Iyengar
        name: Jana Iyengar
        org: Fastly
        role: editor
      -
        ins: M. Thomson
        name: Martin Thomson
        org: Mozilla
        role: editor

informative:
  RFC8446:

--- abstract

This document explains a method of sharing address validation and congestion
control logic when multiple connections are established between two endpoints.

--- middle

# Introduction

One of the goals of QUIC [QUIC-TRANSPORT] is to reduce the connection
establishment latency.  It has reduced the latency to one round-trip time for
the most common case, or to zero when 0-RTT resumption is being used. However,
the endpoints are still restricted to slow start when the connection is
established.  Also, the server is not allowed to send more than three times of
data until it validates the address of the peer.  Note that TLS [RFC8446]
requires connection for each server name to be established independently even if
the servers resolve to the same address.

This document defines a QUIC transport parameter that changes the scope of the
token sent using a NEW_TOKEN frame from the server identity to the server's
address, so that the server can use the information carried in the token to
consolidate the congestion control state among the connections going to the
same client.

## Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC2119].

# The unbound_token Transport Parameter

The unbound_token transport parameter (0xTBD) indicates the client that the
tokens provided by NEW_TOKEN frames can be used for future connections that go
to the same server address tuple rather than that to the same server name.

The unbound_token transport parameter does not carry a value; the length of the
value MUST be empty.  Only the server sends the unbound_token transport
parameter.  When an endpoint observes a peer not obeying to these rules, it
MUST terminate the connection with a PROTOCOL_VIOLATION error..

A server can include the identifier of the congestion controller in the token so
that it can consolidate newly established connections using that token to the
existing congestion controller.

Clients SHOULD retain provided tokens with the original server address tuple of
the connections being the keys rather than those advertised as servers'
preferred addresses, so that the associated unbound token can be found for
subsequent connections even if the preceding connections migrate to a different
address.


# Sharing the Congestion Controller

When a consolidated congestion controller is being used, an endpoint can
distribute the congestion window between the connections in any way.  However,
acknowledgements MUST be sent out at the same moments as when the congestion
controller is not consolidated.  The loss recovery logic SHOULD operate
independently for each connection, but forward receipts of acknowledgements and
loss signals to the consolidated congestion controller.

Unbound token also provides increased chance of skipping address validation
during connection establishment, when multitudes of server names are hosted by a
single server.

# Cross-connection Prioritization

The PRIORITY frame (0xTBD) indicates the priority of the connection within
the connections associated to the same consolidated congestion controller.

The PRIORITY frames is as follows:

~~~
 0
 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+
|  Priority (8) |
+-+-+-+-+-+-+-+-+
~~~

The Priority field carries the priority of the connection, subtracted by one.

Each connection is assigned a priority value between 1 and 256.  The initial
priority is 16.

The PRIORITY frame is sent by an endpoint to encourage the receiver to assign
bandwidth proportional to the suggested priority value for each connection.

# Security Considerations

## Reflection Attack

An attacker can create a connection to obtain an unbound token, then initiate a
new connection by using the token with a spoofed client address, thereby
skipping address validation.  The impact of the attack is equivalent to spoofed
NAT rebinding.  A server SHOULD NOT skip path validation if the source IP
address of an initiating connection is different from the address for which the
unbound token was issued.

A server associates an Initial packet to an existing connection using the
Destination Connection ID, QUIC version, and the five tuple.  If all of the
values match to that of an existing connection, the packet is processed
accordingly.  Otherwise, a server MUST handle the packet as potentially
creating a new connection.

## Plaintext Tokens

A server MUST NOT issue an unbound token that includes the name of the original
server or the identifier of the congestion controller in cleartext, because if
visible on the wire, observers can use that information to correlate the ongoing
connection establishment and the properties of the connection that previously
existed.

# IANA Considerations

TBD

--- back

# Acknowledgements

A similar proposal can be found at <https://svs.informatik.uni-hamburg.de/publications/2019/2019-03-22-Sy-preprint-Surfing-the-Web-quicker-than-QUIC-via-a-shared-Address-Validation.pdf>
that advocates for a transport parameter for changes the scope of token to a
list of server names, and uses token to skip address validation.  This proposal
is different from that in three aspects:

* The scope of the token is the server's IP address.
* The token is used also for consolidating the congestion controller.
* Cross-connection prioritization is defined.
