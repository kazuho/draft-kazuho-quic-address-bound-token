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
  RFC7838:
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
congestion controller state among multiple connections established between the
same two endpoints.

--- middle

# Introduction

Some if not all of the application protocols that are built on top of QUIC
[QUIC-TRANSPORT], including HTTP/3 [QUIC-HTTP], require or would require
clients to establish different connections for each server name, even when those
server names are hosted by the same server.  This restriction introduces several
drawbacks:

* Address validation is required for each connection establishment specifying a
  different server name, thereby restricting the amount of data that a server
  can initially send.
* Each connection would go through the slow-start phase, limiting the amount of
  data that can be sent by a server during the early stages of each connection.
* It is hard if not impossible to control the distribution of the bandwidth
  among the connections.

To resolve these issues, this document defines a QUIC transport parameter that
expands the scope of the token from the server name to a union of the server
name and the server's address tuple.

When sending a token, a server would embed an identifier of the congestion
controller associated to the connection.  Then, when accepting a new connection
using the advertised token, the server associates the new connection to the
existing congestion controller by using the identifier found in the provided
token.  Once the server succeeds in associating the new connection to the
existing congestion controller, it can skip address validation and slow-start
phase for the new connection, as well as using the congestion controller for
distributing bandwidth between the old and the new connection.

Even when there is no existing connection, sharing the tokens between different
server names raises the chance of the server receiving a token that has not yet
expired, thereby improving the odds of skipping address validation and reusing
the information of the path, such as the estimated round-trip time or the
bandwidth.

## Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC2119].

# The address_bound_token Transport Parameter

A server sends the `address_bound_token` transport parameter (0xTBD) to
indicate the client that the token it would send using the NEW_TOKEN frame can
be used for future connections established against the same server name, or for
those sharing the same server IP address and port number.

Only the server sends the `address_bound_token` transport parameter.  The
transport parameter does not carry a value; the length of the value MUST be set
to zero.  An endpoint that receives the transport parameter not conforming to
these requirements MUST terminate the connection with a PROTOCOL_VIOLATION
error.

# Sharing the Congestion Controller

When multiple QUIC connections are associated to a single congestion controller,
how the send window is distributed between the connections is up to the sender's
discretion.

However, acknowledgements MUST be sent out no later than as when the congestion
controller is not consolidated.  The loss recovery logic SHOULD operate
independently for each connection, while forwarding receipts of acknowledgements
and loss signals to the consolidated congestion controller.

# Security Considerations

## Reflection Attack

An attacker can create a connection to obtain an address-bound token, warm up
the connection, then initiate a new connection by using the token with a
spoofed client address or port number.  If the server skips address validation
and retains the congestion window as-is, the spoofed address might receive a
fair amount of packet.

The impact of the attack is equivalent to the spoofed NAT rebinding attack.  A
server SHOULD NOT skip path validation if the source IP address of an initiating
connection is different from the address for which the address-bound token was
issued.

## Plaintext Tokens

A server MUST NOT issue an unbound token that includes the name of the original
server or the identifier of the congestion controller in cleartext, because if
visible on the wire, observers can use that information to correlate the ongoing
connection establishment and the properties of the connection that previously
existed.

# IANA Considerations

TBD

--- back

# Design Variations

## Using Alt-Svc Name as the Key

An alternative approach to using the server's address tuple as the scope of the
token is to use the `host` value of the Alt-Svc [RFC7838] header field as the
scope.

In such an approach, a server would send one host value for all the origins it
hosts.  Then, a client using the value of the host as the scope of the tokens
 would be able to send a token received on any of the connections that went to
the server on any of the future connections that goes to the server.

The downside of the approach is that the design works only for HTTP/3
connections being upgraded by the Alt-Svc header field.

## Cross-connection Prioritization

A natural extension to the proposed scheme would be to define a way of
prioritizing the connections, so that some connections can be given higher
precedence than others.  As an example, it would be sensible to prioritize a
connection carrying real-time video stream above a connection that is
transferring an update image of an operating system.

A simple way of prioritizing between the connections would be to associate a
priority value to every connection that would be respected by the sender when
it distributes the bandwidth among the connections.

The PRIORITY frame (type=0xTBD) indicates the priority.

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
advertises its preference on how the data sent by the server should be
prioritized; a server advertises its preference on how the data sent by the
client should be prioritized.

# Acknowledgements

A proposal exists that advocates for having a transport parameter to change the
scope of a token to a list of server names: <https://svs.informatik.uni-hamburg.de/publications/2019/2019-03-22-Sy-preprint-Surfing-the-Web-quicker-than-QUIC-via-a-shared-Address-Validation.pdf>.
The approach described in this document is different from that in the following
aspects:

* The scope of the token is the union of the server name and the server's
  address tuple.
* The token is used also for consolidating the congestion controller.
