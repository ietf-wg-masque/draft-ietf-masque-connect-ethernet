---
title: "Proxying Ethernet in HTTP"
abbrev: "CONNECT-ETHERNET"
category: std
docname: draft-ietf-masque-connect-ethernet-latest
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
  github: "ietf-wg-masque/draft-ietf-masque-connect-ethernet"
  latest: "https://ietf-wg-masque.github.io/draft-ietf-masque-connect-ethernet/draft-ietf-masque-connect-ethernet.html"

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

informative:


--- abstract

This document describes how to proxy Ethernet frames in HTTP. This protocol
is similar to IP proxying in HTTP, but for Layer 2 instead of Layer 3. More
specifically, this document defines a protocol that allows an HTTP client to
create Layer 2 Ethernet tunnel through an HTTP server to an attached
physical or virtual Ethernet segment.


--- middle

# Introduction

HTTP provides the CONNECT method (see {{Section 9.3.6 of !HTTP=RFC9110}}) for
creating a TCP {{!TCP=RFC9293}} tunnel to a destination, a similar mechanism for
UDP {{!CONNECT-UDP=RFC9298}}, and an additional mechanism for IP
{{!CONNECT-IP=RFC9484}}. However, these mechanisms can't carry layer 2 frames
without further encapsulation inside of IP, for instance with EtherIP
{{?ETHERIP=RFC3378}}, GUE {{?GUE=I-D.ietf-intarea-gue}} or L2TP
{{!L2TP=RFC2661}} {{!L2TPv3=RFC3931}}, which imposes an additional MTU cost.

This document describes a protocol for exchanging Ethernet frames with an HTTP
server. Either participant in the HTTP connection can then relay Ethernet
frames to and from a local or virtual interface, allowing the bridging of two
Ethernet broadcast domains to establish a Layer 2 VPN. This can simplify
connectivity to network-connected appliances that are configured to only
interact with peers on the same Ethernet broadcast domain.

This protocol supports all existing versions of HTTP by using HTTP Datagrams
{{!HTTP-DGRAM=RFC9297}}. When using HTTP/2 {{H2}} or HTTP/3 {{H3}}, it uses
HTTP Extended CONNECT as described in {{!EXT-CONNECT2=RFC8441}} and
{{!EXT-CONNECT3=RFC9220}}. When using HTTP/1.x {{H1}}, it uses HTTP Upgrade as
defined in {{Section 7.8 of HTTP}}.

This protocol necessarily involves additional framing overhead. When possible,
users should use higher-level proxying protocols.

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

Clients are configured to use Ethernet proxying over HTTP via a URI Template
{{!TEMPLATE=RFC6570}}. The URI Templates used by this protocol do not require
any variables; implementations or extensions MAY specify their own. URI
Templates specified for this protocol MAY use the well-known location
{{!WELL-KNOWN=RFC8615}} registered by this document.

Examples are shown below:

~~~
https://example.org/.well-known/masque/ethernet/
https://proxy.example.org:4443/masque/ethernet/
https://masque.example.org/?user=bob
~~~

An implementation that supports connecting to different Ethernet segments might
add a "vlan-identifier" variable to specify which segment to connect to. The
optionality of variables needs to be considered when defining the template so
that variables are either self-identifying or possible to exclude in the syntax.
How valid values for such variables are communicated to the client is not a part
of this protocol.

Hypothetical examples are shown below:

~~~
https://proxy.example.org:4443/masque/ethernet?vlan={vlan-identifier}
https://etherproxy.example.org/{vlan-identifier}
~~~

# Tunnelling Ethernet over HTTP

To allow negotiation of a tunnel for Ethernet over HTTP, this document defines
the "connect-ethernet" HTTP upgrade token. The resulting Ethernet tunnels use the
Capsule Protocol (see {{Section 3.2 of HTTP-DGRAM}}) with HTTP Datagrams in the
format defined in {{payload-format}}.

To initiate an Ethernet tunnel associated with a single HTTP stream, a client
issues a request containing the "connect-ethernet" upgrade token.

By virtue of the definition of the Capsule Protocol (see {{Section 3.2 of
HTTP-DGRAM}}), Ethernet proxying requests do not carry any message content.
Similarly, successful Ethernet proxying responses also do not carry any message
content.

Ethernet proxying over HTTP MUST be operated over TLS or QUIC encryption, or another
equivalent encryption protocol, to provide confidentiality, integrity, and
authentication.

## Ethernet Proxy Handling

Upon receiving an Ethernet proxying request:

 * If the recipient is configured to use another HTTP proxy, it will act as an
   intermediary by forwarding the request to another HTTP server. Note that
   such intermediaries may need to re-encode the request if they forward it
   using a version of HTTP that is different from the one used to receive it,
   as the request encoding differs by version (see below).

 * Otherwise, the recipient will act as an Ethernet proxy. The Ethernet proxy
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

* The method SHALL be "GET".

* The request SHALL include a single Host header field containing the host
  and optional port of the Ethernet proxy.

* The request SHALL include a Connection header field with value "Upgrade"
  (note that this requirement is case-insensitive as per {{Section 7.6.1 of
  HTTP}}).

* The request SHALL include an Upgrade header field with value "connect-ethernet".

An Ethernet proxying request that does not conform to these restrictions is
malformed. The recipient of such a malformed request MUST respond with an error
and SHOULD use the 400 (Bad Request) status code.

For example, if the client is configured with the URI Template
"https://example.org/.well-known/masque/ethernet/" and wishes to open an
Ethernet tunnel, it could send the following request.

~~~ http-message
GET https://example.org/.well-known/masque/ethernet/ HTTP/1.1
Host: example.org
Connection: Upgrade
Upgrade: connect-ethernet
Capsule-Protocol: ?1
~~~
{: #fig-req-h1 title="Example HTTP/1.1 Request"}

## HTTP/1.1 Response {#resp1}

The server indicates success by replying with a response that conforms to the
following requirements:

* The HTTP status code on the response SHALL be 101 (Switching Protocols).

* The response SHALL include a Connection header field with value "Upgrade"
  (note that this requirement is case-insensitive as per {{Section 7.6.1 of
  HTTP}}).

* The response SHALL include a single Upgrade header field with value
  "connect-ethernet".

* The response SHALL meet the requirements of HTTP responses that start the
  Capsule Protocol; see {{Section 3.2 of HTTP-DGRAM}}.

If any of these requirements are not met, the client MUST treat this proxying
attempt as failed and close the connection.

For example, the server could respond with:

~~~ http-message
HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: connect-ethernet
Capsule-Protocol: ?1
~~~
{: #fig-resp-h1 title="Example HTTP/1.1 Response"}

## HTTP/2 and HTTP/3 Requests {#req23}

When using HTTP/2 {{H2}} or HTTP/3 {{H3}}, Ethernet proxying requests use HTTP
Extended CONNECT. This requires that servers send an HTTP Setting as specified
in {{EXT-CONNECT2}} and {{EXT-CONNECT3}} and that requests use HTTP
pseudo-header fields with the following requirements:

* The :method pseudo-header field SHALL be "CONNECT".

* The :protocol pseudo-header field SHALL be "connect-ethernet".

* The :authority pseudo-header field SHALL contain the authority of the IP
  proxy.

* The :path and :scheme pseudo-header fields SHALL NOT be empty. Their values
  SHALL contain the scheme and path from the configured URL; see
  {{client-config}}.

An Ethernet proxying request that does not conform to these restrictions is
malformed; see {{Section 8.1.1 of H2}} and {{Section 4.1.2 of H3}}.

For example, if the client is configured with the URI Template
"https://example.org/.well-known/masque/ethernet/" and wishes to open an
Ethernet tunnel, it could send the following request.

~~~ http-message
HEADERS
:method = CONNECT
:protocol = connect-ethernet
:scheme = https
:path = /.well-known/masque/ethernet/
:authority = example.org
capsule-protocol = ?1
~~~
{: #fig-req-h2 title="Example HTTP/2 or HTTP/3 Request"}

## HTTP/2 and HTTP/3 Responses {#resp23}

The server indicates success by replying with a response that conforms to the
following requirements:

* The HTTP status code on the response SHALL be in the 2xx (Successful) range.

* The response SHALL meet the requirements of HTTP responses that start the
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
future extensions to exchange HTTP Datagrams that carry different semantics from
Ethernet frames. Some of these extensions could augment Ethernet payloads with
additional data or compress Ethernet frame header fields, while others could
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
the Frame check sequence field), as defined by IEEE 802.3, with support for
optional IEEE 802.1Q tagging (see {{vlan-recommendations}}).

# Ethernet Frame Handling

This document defines a tunnelling mechanism that is conceptually an Ethernet
link. Implementations might need to handle some of the responsibilities of an
Ethernet switch or bridge if they do not delegate them to another implementation
such as a kernel. Those responsibilities are beyond the scope of this document,
and include, but are not limited to, the handling of broadcast packets and
multicast groups, or the local termination of PAUSE frames.

If an Ethernet proxying endpoint fails to deliver a frame to an underlying
Ethernet segment, the endpoint MUST drop the frame.

# Examples

Ethernet proxying in HTTP enables the bridging of Ethernet broadcast domains.
These examples are provided to help illustrate some of the ways in which Ethernet
proxying can be used.

## Remote Access L2VPN {#example-remote}

The following example shows a point to point VPN setup where a client
appears to be connected to a remote Layer 2 network.

~~~ aasvg

+--------+                    +--------+            +---> HOST 1
|        +--------------------+   L2   |  Layer 2   |
| Client |<--Layer 2 Tunnel---|  Proxy +------------+---> HOST 2
|        +--------------------+        |  Broadcast |
+--------+                    +--------+  Domain    +---> HOST 3

~~~
{: #diagram-tunnel title="L2VPN Tunnel Setup"}

In this case, the client connects to the Ethernet proxy and immediately can
start sending ethernet frames to the attached broadcast domain.

~~~
[[ From Client ]]             [[ From Ethernet Proxy ]]

SETTINGS
  H3_DATAGRAM = 1

                              SETTINGS
                                ENABLE_CONNECT_PROTOCOL = 1
                                H3_DATAGRAM = 1

STREAM(44): HEADERS
:method = CONNECT
:protocol = connect-ethernet
:scheme = https
:path = /.well-known/masque/ethernet/
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


## Site-to-Site L2VPN

The following example shows a site-to-site VPN setup where a client
joins a locally attached broadcast domain to a remote broadcast domain
through the Proxy.

~~~ aasvg

         +--------+               +--------+
         |        +---------------+   L2   |
         | Client |---L2 Tunnel---|  Proxy |
         |        +---------------+        |
         +-+------+               +------+-+
           |                             |
HOST A <---+ Layer 2             Layer 2 +---> HOST 1
           | Broadcast         Broadcast |
HOST B <---+ Domain               Domain +---> HOST 2
           |                             |
HOST C <---+                             +---> HOST 3



~~~
{: #diagram-s2s title="Site-to-site L2VPN Example"}

In this case, the client connects to the Ethernet proxy and immediately can
start relaying Ethernet frames from its attached broadcast domain to the
proxy. The difference between this example and {{example-remote}} is limited to
what the Client is doing with the the tunnel; the exchange between the Client
and the Proxy is the same as in {{fig-full-tunnel}} above.

# Performance Considerations

When the protocol running inside the tunnel uses congestion control (e.g.,
{{TCP}} or {{QUIC}}), the proxied traffic will incur at least two nested
congestion controllers. Implementers will benefit from reading the guidance in
{{Section 3.1.11 of ?UDP-USAGE=RFC8085}}. By default the tunneling of Ethernet
frames MUST NOT assume that the carried Ethernet frames contain congestion
controlled traffic. Optimizations for traffic flows carried within the Ethernet
Frames MAY be done in cases where the content of the Ethernet Frames have been
identified to be congestion controlled traffic.

Some implementations might find it benefitial to maintain a small buffer of
frames to be sent through the tunnel to smooth out short term variations and
bursts in tunnel capacity. Any such buffer MUST be limited, and when exceeded
any Ethernet frames that would otherwise be buffered MUST be dropped.

When the protocol running inside the tunnel uses loss recovery (e.g., {{TCP}} or
{{QUIC}}) and the outer HTTP connection runs over TCP, the proxied traffic will
incur at least two nested loss recovery mechanisms. This can reduce performance,
as both can sometimes independently retransmit the same data. To avoid this,
Ethernet proxying SHOULD be performed over HTTP/3 to allow leveraging the QUIC
DATAGRAM frame.

## MTU and Frame Ordering Considerations

When using HTTP/3 with the QUIC Datagram extension {{!QUIC-DGRAM=RFC9221}},
Ethernet frames can be transmitted in QUIC DATAGRAM frames. Since DATAGRAM
frames cannot be fragmented, they can only carry Ethernet frames up to a given
length determined by the QUIC connection configuration and the Path MTU
(PMTU). Implementations MAY rely on {{QUIC}}'s use of {{!DPLPMTUD=RFC8899}} to
probe and discover the PMTU over the connection's lifetime, and adjust any
associated interface MTU as needed. Furthermore, the UDP packets carrying these
frames could be reordered by the network.

When using HTTP/1.1 or HTTP/2, and when using HTTP/3 without the QUIC Datagram
extension {{QUIC-DGRAM}}, Ethernet frames are transmitted in DATAGRAM capsules as
defined in {{HTTP-DGRAM}}. DATAGRAM capsules are transmitted reliably over an
underlying stream, maintaining frame order, though they could be split across
multiple QUIC or TCP packets.

The trade-off between supporting a larger MTU and avoiding fragmentation should
be considered when deciding what mode(s) to operate in. Implementations SHOULD
NOT intentionally reorder Ethernet frames, but are not required to provide
guaranteed in-order delivery. If in-order delivery of Ethernet frames is
required, DATAGRAM capsules can be used.

## IEEE 802.1Q VLAN tagging {#vlan-recommendations}

While the protocol as described can proxy Ethernet frames with IEEE 802.1Q VLAN
tags, it is RECOMMENDED that individual VLANs be proxied in separate
connections, and VLAN tags be stripped and applied by the Ethernet proxying
endpoints as needed.

# Security Considerations

There are risks in allowing arbitrary clients to establish a tunnel to a Layer 2
network. Bad actors could abuse this capability to attack hosts on that network
that they would otherwise be unable to reach. HTTP servers that support Ethernet
proxying SHOULD restrict its use to authenticated users. Depending on the
deployment, possible authentication mechanisms include mutual TLS between IP
proxying endpoints, HTTP-based authentication via the HTTP Authorization header
{{HTTP}}, or even bearer tokens. Proxies can enforce policies for authenticated
users to further constrain client behavior or deal with possible abuse. For
example, proxies can rate limit individual clients that send an excessively
large amount of traffic through the proxy.

Users of this protocol may send arbitrary Ethernet frames through the tunnel,
including frames with arbitrary source MAC addresses. This could allow
impersonation of other hosts, poisoning of ARP {{!RFC826}}, NDP {{!RFC4861}} and
CAM (Content Addressable Memory) tables, and cause a denial of service to other
hosts on the network. These are the same attacks available to an arbitrary
client with physical access to the network. An implementation that is intended
for point-to-site connections might limit clients to a single source MAC
address, or Ethernet proxying endpoints might be configured to limit forwarding
to pre-configured MAC addresses, though such filtering is outside the scope of
this protocol. Dynamic signalling or negotiation of MAC address filtering is
left to future extensions.

This protocol is agnostic to where on the Ethernet segment a gateway for
higher-level routing might be located. A client may connect via an Ethernet
proxy and discover an existing gateway on the Ethernet segment, supply a new
gateway to the Ethernet segment, both, or neither.

# IANA Considerations

## HTTP Upgrade Token

This document will request IANA to register "connect-ethernet" in the HTTP
Upgrade Token Registry maintained at
<[](https://www.iana.org/assignments/http-upgrade-tokens)>.

Value:

: connect-ethernet

Description:

: Proxying of Ethernet Payloads

Expected Version Tokens:

: None

References:

: This document
{: spacing="compact"}

## Updates to the MASQUE URI Suffixes Registry {#iana-suffix}

This document will request IANA to register "ethernet" in the MASQUE URI
Suffixes Registry maintained at <[](https://www.iana.org/assignments/masque)>,
created by {{Section 12.2 of CONNECT-IP}}.

| Path Segment |    Description    |   Reference   |
|:-------------|:------------------|:--------------|
|    ethernet  | Ethernet Proxying | This Document |
{: #iana-suffixes-table title="New MASQUE URI Suffixes"}

--- back

# Acknowledgments
{:numbered="false"}

Much of the initial version of this draft borrows heavily from {{CONNECT-IP}}.

The author would like to thank Alexander Chernyakhovsky and David Schinazi
for their advice while preparing this document, and Etienne Dechamps for
useful discussion on the subject material. Additionally, Mirja Kühlewind,
Magnus Westerlund, and Martin Thompson provided valuable feedback on the
document.
