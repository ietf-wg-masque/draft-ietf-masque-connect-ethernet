---
title: "Proxying Ethernet in HTTP"
abbrev: "CONNECT-ETHERNET"
category: std

docname: draft-asedeno-masque-connect-ethernet-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Transport"
workgroup: "Multiplexed Application Substrate over QUIC Encryption"
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: "Multiplexed Application Substrate over QUIC Encryption"
  type: "Working Group"
  mail: "masque@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/masque/"
  github: "asedeno/draft-asedeno-masque-connect-ethernet"
  latest: "https://asedeno.github.io/draft-asedeno-masque-connect-ethernet/draft-asedeno-masque-connect-ethernet.html"

author:
 -
    ins: A. Sedeño
    name: Alejandro R Sedeño
    organization: Google LLC
    email: asedeno@google.com

normative:
  H1:
    =: RFC9112
    display: HTTP/1.1
  H2:
    =: RFC9113
    display: HTTP/2
  H3:
    =: RFC9114
    display: HTTP/3
informative:


--- abstract

This document describes how to proxy Ethernet frames in HTTP. This
protocol is similar to IP proxying in HTTP, but allows transmitting arbitrary
Ethernet frames.  More specifically, this document defines a protocol that
allows an HTTP client to create Layer 2 Ethernet tunnel through and HTTP server
that acts as an Ethernet switch.


--- middle

# Introduction

HTTP provides the CONNECT method (see {{Section 9.3.6 of !HTTP=RFC9110}}) for
creating a TCP {{!TCP=RFC9293}} tunnel to a destination, a similar mechanism for
UDP {{?CONNECT-UDP=RFC9298}}, and an additional mechanism for IP
[CONNECT-IP]. However, these mechanisms can't carry layer 2 frames without
further encapsulation inside of IP, for instance with GUE, which imposes an MTU
cost.

This document describes a protocol for tunnelling Ethernet frames through an
HTTP server acting as an Ethernet switch over HTTP. This can be used to extend
an Ethernet broadcast domain.

This protocol supports all existing versions of HTTP by using HTTP Datagrams
{{!HTTP-DGRAM=RFC9297}}. When using HTTP/2 {{H2}} or HTTP/3 {{H3}}, it uses
HTTP Extended CONNECT as described in {{!EXT-CONNECT2=RFC8441}} and
{{!EXT-CONNECT3=RFC9220}}. When using HTTP/1.x {{H1}}, it uses HTTP Upgrade as
defined in {{Section 7.8 of HTTP}}.



# Conventions and Definitions

{::boilerplate bcp14-tagged}

In this document, we use the term "Ethernet proxy" to refer to the HTTP server
that responds to the Ethernet proxying request. The term "client" is used in the
HTTP sense; the client constructs the Ethernet proxying request. If there are
HTTP intermediaries (as defined in {{Section 3.7 of HTTP}}) between the client
and the Ethernet proxy, those are referred to as "intermediaries" in this
document. The term "Ethernet proxying endpoints" refers to both the client and
the Ethernet proxy.

This document uses terminology from {{!QUIC=RFC9000}}. Where this document
defines protocol types, the definition format uses the notation from {{Section
1.3 of QUIC}}. This specification uses the variable-length integer encoding
from {{Section 16 of !QUIC=RFC9000}}. Variable-length integer values do not
need to be encoded in the minimum number of bytes necessary.

Note that, when the HTTP version in use does not support multiplexing streams
(such as HTTP/1.1), any reference to "stream" in this document represents the
entire connection.


# Configuration of Clients {#client-config}

TODO config


# Tunnelling Ethernet over HTTP


# Context Identifiers

The mechanism for proxying Ethernet in HTTP defined in this document allows
futer extensions to exchange HTTP Datagrams that carry different semantics from
Ethernet frames. Some of these extensions can augment Ethernet payloads with
additional data or compress Ethernet frame header fields, while others can
exchange data that is completely separate from Ethernet payloads. In order to
accomplish this, all HTTP Datagrams associated with Ethernet proxying requests
streams start with a Context ID field; see {{payload-format}}.

Context IDs are 62-bit integers (0 to 2<sup>62</sup>-1). Context IDs are encoded
as variable-length integers; see {{Section 16 of QUIC}}. The Context ID value of
0 is reserved for Ethernet payloads, while non-zero values are dynamically
allocated. Non-zero even-numbered Context-IDs are client allocated, and
odd-numbered Context IDs are proxy-allocated. The Context ID namespace is tied
to a given HTTP request; it is possible for a Context ID with the same numeric
value to be simultaneously allocated in distinct requests, potentially with
different semantics. Context IDs MUST NOT be re-allocated within a given HTTP
request but MAY be allocated in any order. The Context ID allocation
restrictions to the use of even-numbered and odd-numbered Context IDs exist in
order to avoid the need for synchronization between endpoints. However, once a
Context ID has been allocated, those restrictions do not apply to the use of the
Context ID; it can be used by either the client or the IP proxy, independent of
which endpoint initially allocated it.

Registration is the action by which an endpoint informs its peer of the
semantics and format of a given Context ID. This document does not define how
registration occurs. Future extensions MAY use HTTP header fields or capsules
to register Context IDs. Depending on the method being used, it is possible for
datagrams to be received with Context IDs that have not yet been registered.
For instance, this can be due to reordering of the packet containing the
datagram and the packet containing the registration message during transmission.


# HTTP Datagram Payload Format {#payload-format}

When associated with Ethernet proxying request streams, the HTTP Datagram
Payload field of HTTP Datagrams (see {{HTTP-DGRAM}}) has the format defined in
{{dgram-format}}. Note that when HTTP Datagrams are encoded using QUIC DATAGRAM
frames, the Context ID field defined below directly follows the Quarter Stream
ID field which is at the start of the QUIC DATAGRAM frame payload.

~~~
Ethernet Proxying HTTP Datagram Payload {
  Context ID (i),
  Payload (..),
}
~~~
{: #dgram-format title="Ethernet Proxying HTTP Datagram Format"}

The Ethernet Proxying HTTP Datagram Payload contains the following fields:

Context ID:

: A variable-length integer that contains the value of the Context ID. If an
HTTP/3 datagram which carries an unknown Context ID is received, the receiver
SHALL either drop that datagram silently or buffer it temporarily (on the order
of a round trip) while awaiting the registration of the corresponding Context
ID.

Payload:

: The payload of the datagram, whose semantics depend on value of the previous
field. Note that this field can be empty.
{: spacing="compact"}

Ethernet frames are encoded using HTTP Datagrams with the Context ID set to
zero.  When the Context ID is set to zero, the Payload field contains a full
Layer 2 Ethernet Frame (from the MAC destination field until the last byte of
the Frame check sequence field).

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
