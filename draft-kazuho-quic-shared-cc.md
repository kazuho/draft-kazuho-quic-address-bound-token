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
  RFC7301:
  RFC8446:
  QUIC-HTTP:
    title: "Hypertext Transfer Protocol Version 3 (HTTP/3)"
    date: 2018-10-23
    seriesinfo:
      Internet-Draft: draft-ietf-quic-http-16
    author:
      -
        ins: M. Bishop
        name: Mike Bishop
        org: Akamai
        role: editor

--- abstract

This document describes a QUIC extension for sharing address validation and
congestion control logic among multiple connections established between the same
two endpoints.

--- middle

# Introduction

QUIC [QUIC-TRANSPORT] requires clients to establish different connections when
either the name of the server or the application protocol specified using ALPN
[RFC7301] is different, even when the connections are established against the
same server.  This restriction introduces several drawbacks:

* Address validation is required for each connection establishment, thereby
  restricting the amount of data that the server can send to the client.
* Each connection would go through the slow-start phase, limiting the amount of
  data that can be exchanged during the early stages of each connection.
* It is impossible to control the distribution of bandwidth among the set of the
  connections, even when some connections require preferential treatment than
  others (e.g., video streaming vs. file download).

Tokens sent by NEW_TOKEN frames mitigate the first two concerns to some extent,
though the effectiveness depends on the probability of clients reestablishing
the connections using the same server name and application protocol.

To resolve these isssues, this document defines a QUIC frame that carries a
token that is valid for the server's address tuple, rather than the server's
name and the selected application protocol.

A server includes the identifier of the congestion controller in the token it
offers, and when the client establishes another connection to the same server
address tuple using the provided token, binds the newly established connection
to the already existing congestion controller identified by the token.  By doing
so, the server skips address validation and slow-start phase for the newly
established connection.

Furthermore, the PRIORITY frame can be used to communicate the precedence
between  the connections that share the same congestion controller.  As an
example, a client might specify highest priority for a connection that carries
live video streams, while specifying normal priority for a HTTP/3 [QUIC-HTTP]
connection.

Even when there is no existing connection, sharing the tokens between different
server names and application protocols raises the chance of the server receiving
a fresh token, thereby improving the odds of skipping address validation and
reusing the information of the path carried by the token.

## Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC2119].

# The shared_cc Transport Parameter

An endpoint send the `shared_cc` transport parameter (0xTBD) to indicate the
peer that the sender is capable of processing the frames introduced by this
specification.

The transport parameter does not carry a value; the length of the value MUST be
set to zero.  A receiver MUST terminate the connection with a PROTOCOL_VIOLATION
error when it receives the `shared_cc` transport parameter with a non-empty
value.

# The NEW_UNBOUND_TOKEN frame

A server sends a NEW_UNBOUND_TOKEN frame (type=0xTBD) to provide the client with
a token to be used for a future connection.

The format of the NEW_UNBOUND_TOKEN is equivalent to that of the NEW_TOKEN
frame.

The only difference between a token carried by the NEW_TOKEN frame and the one
carried by the NEW_UNBOUND_TOKEN is the scope of the token.  The scope of the
token carried by the NEW_UNBOUND_TOKEN frame is the original server address
tuple of the connection even when the server offers a server's preferred
address.

# The PRIORITY frame

The PRIORITY frame (type=0xTBD) indicates the precedence of the connection
within the connections associated to the same consolidated congestion
controller.

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

The priority value carried by the PRIORITY frame is unidirectional.  A client
advertises it's preference on how the data sent by the server should be
prioritized; a server advertises it's preference on how the data sent by the
client should be prioritized.

# Sharing the Congestion Controller

When multiple QUIC connections are associated to a single congestion controller,
how the send window is distributed between the connections is up to the sender's
discretion, even though the peer can indicate it's preference by using the
PRIORITY frame.

However, acknowledgements MUST be sent out at the same moments as when the
congestion controller is not consolidated.  The loss recovery logic SHOULD
operate independently for each connection, while forwarding receipts of
acknowledgements and loss signals to the consolidated congestion controller.

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

A proposal exists that advocates for having a transport parameter to change the
scope of a token to a list of server names: <https://svs.informatik.uni-hamburg.de/publications/2019/2019-03-22-Sy-preprint-Surfing-the-Web-quicker-than-QUIC-via-a-shared-Address-Validation.pdf>.
The approach described in this document is different from that is the following
aspects:

* The scope of the token is the server's address tuple.
* The token is carried by a new frame, so that a server can send a token
  identified by the name and one identified by hte server's address tuple within
  each connection.
* The token is used also for consolidating the congestion controller.
* Cross-connection prioritization is defined.
