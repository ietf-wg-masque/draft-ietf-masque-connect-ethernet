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
 - quic
 - http
 - datagram
 - VPN
 - proxy
 - tunnels
 - masque
 - http-ng
 - layer-2
 - ethernet
venue:
  group: "MASQUE"
  type: "Working Group"
  mail: "masque@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/masque/"
  github: "asedeno/draft-asedeno-masque-connect-ethernet"
  latest: "https://asedeno.github.io/draft-asedeno-masque-connect-ethernet/draft-asedeno-masque-connect-ethernet.html"

author:
 -
    ins: A. R. Sedeño
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
  TEMPLATE:
    =: RFC6570
    display: TEMPLATE

informative:


--- abstract

This document describes how to proxy Ethernet frames in HTTP. This
protocol is similar to IP proxying in HTTP, but allows transmitting arbitrary
Ethernet frames. More specifically, this document defines a protocol that
allows an HTTP client to create Layer 2 Ethernet tunnel through and HTTP server
that acts as an Ethernet switch.


--- middle

# Introduction

HTTP provides the CONNECT method (see {{Section 9.3.6 of !HTTP=RFC9110}}) for
creating a TCP {{!TCP=RFC9293}} tunnel to a destination, a similar mechanism for
UDP {{!CONNECT-UDP=RFC9298}}, and an additional mechanism for IP
{{?CONNECT-IP=I-D.ietf-masque-connect-ip}}. However, these mechanisms can't
carry layer 2 frames without further encapsulation inside of IP, for instance
with GUE {{?GUE=I-D.ietf-intarea-gue}}or L2TP {{!L2TP=RFC2661}}
{{!L2TPv3=RFC3931}}, which imposes an MTU cost.

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

_We don't have the same level of configuration as {{CONNECT-IP}}, such that a
URI Template {{TEMPLATE}} may not make sense. However, if we write this as a
more generic Layer 2 tunnel with an initial Ethernet implementation and
extensibility for other Layer 2 protocols, then a template carrying the desired
Layer 2 protocol may be worthwhile._

~~~
https://${PROXY_HOST}:${PROXY_PORT}/.well-known/masque/l2/
https://${PROXY_HOST}:${PROXY_PORT}/.well-known/masque/l2/${L2_PROTOCOL}/
~~~

# Tunnelling Ethernet over HTTP

To allow negotiation of a tunnel for Ethernet over HTTP, this documetn defines
the "connect-l2" HTTP upgrade token. The resulting Ethernet tunnels use the
Capsule Protocol (see {{Section 3.2 of HTTP-DGRAM}}) with HTTP Datagrams in the
format defined in {{payload-format}}.

To initiate an Ethernet tunnel associated with a single HTTP stream, a client
issues a request containing the "connect-l2" upgrade token.

By virtue of the definition of the Capsule Protocol (see {{Section 3.2 of
HTTP-DGRAM}}), Ethernet proxying requests do not carry any message content.
Similarly, successful Ethernet proxying responses also do not carry any message
content.

Ethernet proxying over HTTP MUST be operated over TLS or QUIC encryption, or another
equivalent encryption protocol, to provide confidentiality, integrity, and
authentication.

## Ethernet Proxy Handling

Upon receiving an Ethernet proxying request:

 * if the recipient is configured to use another HTTP proxy, it will act as an
   intermediary by forwarding the request to another HTTP server. Note that
   such intermediaries may need to re-encode the request if they forward it
   using a version of HTTP that is different from the one used to receive it,
   as the request encoding differs by version (see below).

 * otherwise, the recipient will act as an Ethernet proxy. The Ethernet proxy
   can choose to reject the Ethernet proxying request or establish an Ethernet
   tunnel.

The lifetime of the Ethernet tunnel is tied to the Ethernet proxying request
stream.

A successful response (as defined in Sections {{<resp1}} and {{<resp23}})
indicates that the Ethernet proxy has established an Ethernet tunnel and is
willing to proxy Ethernet frames. Any response other than a successful response
indicates that the request has failed; thus, the client MUST abort the request.

## HTTP/1.1 Request {#req1}

When using HTTP/1.1 {{H1}}, an Ethernet proxying request will meet the following
requirements:

* the method SHALL be "GET".

* the path SHALL be "/.well-known/masque/l2/"

* the request SHALL include a single Host header field containing the host
  and optional port of the IP proxy.

* the request SHALL include a Connection header field with value "Upgrade"
  (note that this requirement is case-insensitive as per {{Section 7.6.1 of
  HTTP}}).

* the request SHALL include an Upgrade header field with value "connect-l2".

An Ethernet proxying request that does not conform to these restrictions is
malformed. The recipient of such a malformed request MUST respond with an error
and SHOULD use the 400 (Bad Request) status code.

_[TODO: Non-Ethernet L2 protocols as an additional path element.]_

~~~ http-message
GET https://example.org/.well-known/masque/l2/ HTTP/1.1
Host: example.org
Connection: Upgrade
Upgrade: connect-l2
Capsule-Protocol: ?1
~~~
{: #fig-req-h1 title="Example HTTP/1.1 Request"}

## HTTP/1.1 Response {#resp1}

The server indicates a successful response by replying with the following
requirements:

* the HTTP status code on the response SHALL be 101 (Switching Protocols).

* the response SHALL include a Connection header field with value "Upgrade"
  (note that this requirement is case-insensitive as per {{Section 7.6.1 of
  HTTP}}).

* the response SHALL include a single Upgrade header field with value
  "connect-l2".

* the response SHALL meet the requirements of HTTP responses that start the
  Capsule Protocol; see {{Section 3.2 of HTTP-DGRAM}}.

If any of these requirements are not met, the client MUST treat this proxying
attempt as failed and close the connection.

For example, the server could respond with:

~~~ http-message
HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: connect-l2
Capsule-Protocol: ?1
~~~
{: #fig-resp-h1 title="Example HTTP/1.1 Response"}

## HTTP/2 and HTTP/3 Requests {#req23}

When using HTTP/2 {{H2}} or HTTP/3 {{H3}}, Ethernet proxying requests use HTTP
Extended CONNECT. This requires that servers send an HTTP Setting as specified
in {{EXT-CONNECT2}} and {{EXT-CONNECT3}} and that requests use HTTP
pseudo-header fields with the following requirements:

* The :method pseudo-header field SHALL be "CONNECT".

* The :protocol pseudo-header field SHALL be "connect-l2".

* The :authority pseudo-header field SHALL contain the authority of the IP
  proxy.

* The :path and :scheme pseudo-header fields SHALL NOT be empty. Their values
  SHALL contain the scheme and path …

_[TODO: uri template, optional layer 2 protocol field, hook for future
extensions.]_

An Ethernet proxying request that does not conform to these restrictions is
malformed; see {{Section 8.1.1 of H2}} and {{Section 4.1.2 of H3}}.

~~~ http-message
HEADERS
:method = CONNECT
:protocol = connect-l2
:scheme = https
:path = /.well-known/masque/l2/
:authority = example.org
capsule-protocol = ?1
~~~
{: #fig-req-h2 title="Example HTTP/2 or HTTP/3 Request"}

## HTTP/2 and HTTP/3 Responses {#resp23}

The server indicates a successful response by replying with the following
requirements:

* the HTTP status code on the response SHALL be in the 2xx (Successful) range.

* the response SHALL meet the requirements of HTTP responses that start the
  Capsule Protocol; see {{Section 3.2 of HTTP-DGRAM}}.

If any of these requirements are not met, the client MUST treat this proxying
attempt as failed and abort the request. As an example, any status code in the
3xx range will be treated as a failure and cause the client to abort the
request.

For example, the server could respond with:

~~~ http-message
HEADERS
:status = 200
capsule-protocol = ?1
~~~
{: #fig-resp-h2 title="Example HTTP/2 or HTTP/3 Response"}

# Context Identifiers

The mechanism for proxying Ethernet in HTTP defined in this document allows
futer extensions to exchange HTTP Datagrams that carry different semantics from
Ethernet frames. Some of these extensions can augment Ethernet payloads with
additional data or compress Ethernet frame header fields, while others can
exchange data that is completely separate from Ethernet payloads. In order to
accomplish this, all HTTP Datagrams associated with Ethernet proxying requests
streams start with a Context ID field; see {{payload-format}}.

Context IDs are 62-bit integers (0-2<sup>62</sup>-1). Context IDs are encoded as
variable-length integers; see {{Section 16 of QUIC}}. The Context ID value of 0
is reserved for Ethernet payloads, while non-zero values are dynamically
allocated. Non-zero even-numbered Context-IDs are client allocated, and
odd-numbered Context IDs are proxy-allocated. The Context ID namespace is tied
to a given HTTP request; it is possible for a Context ID with the same numeric
value to be simultaneously allocated in distinct requests, potentially with
different semantics. Context IDs MUST NOT be re-allocated within a given HTTP
request but MAY be allocated in any order. The Context ID allocation
restrictions to the use of even-numbered and odd-numbered Context IDs exist in
order to avoid the need for synchronization between endpoints. However, once a
Context ID has been allocated, those restrictions do not apply to the use of the
Context ID; it can be used by either the client or the Ethernet proxy,
independent of which endpoint initially allocated it.

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
zero. When the Context ID is set to zero, the Payload field contains a full
Layer 2 Ethernet Frame (from the MAC destination field until the last byte of
the Frame check sequence field).

# Ethernet Frame Handling

_TODO: Expand this._

 * Be like a switch, not a hub.
 * Handle broadcast and multicast well.
 * Maybe offload the details to a kernel-level network bridge.

## Error Signalling

_TODO: Expand this._

 * Maybe borrow some bits from L2TPv3 [L2TPv3] for fault signalling via capsule.

# Examples

Some examples; if you can, a Layer 3 option is probably better, but hey, Layer 2.

## Layer 2 VPN

The following example shows how a point to point VPN setup where a client
appears to be connected to a remote Layer 2 network.

~~~ aasvg

+--------+                    +--------+            +---> HOST 1
|        +--------------------+   L2   |  Layer 2   |
| Client |<--Layer 2 Tunnel---|  Proxy +------------+---> HOST 2
|        +--------------------+        |  Broadcast |
+--------+                    +--------+   Domain   +---> HOST 3

~~~
{: #diagram-tunnel title="L2 VPN Tunnel Setup"}

In this case, the client connects to the Ethernet proxy and immediately can
start sending ethernet frames to the attached broadcast domain.

~~~
[[ From Client ]]             [[ From IP Proxy ]]

SETTINGS
  H3_DATAGRAM = 1

                              SETTINGS
                                ENABLE_CONNECT_PROTOCOL = 1
                                H3_DATAGRAM = 1

STREAM(44): HEADERS
:method = CONNECT
:protocol = connect-l2
:scheme = https
:path = /.well-known/masque/l2/
:authority = proxy.example.com
capsule-protocol = ?1

                              STREAM(44): HEADERS
                              :status = 200
                              capsule-protocol = ?1

DATAGRAM
Quarter Stream ID = 11
Context ID = 0
Payload = Encapsulated Ethernet Frame

                              DATAGRAM
                              Quarter Stream ID = 11
                              Context ID = 0
                              Payload = Encapsulated Ethernet Frame
~~~
{: #fig-full-tunnel title="VPN Full-Tunnel Example"}

# Security Considerations

TODO Security

* MAC address collisions?

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.

Much of the initial version of this draft borrows heavily from {{CONNECT-IP}}.
