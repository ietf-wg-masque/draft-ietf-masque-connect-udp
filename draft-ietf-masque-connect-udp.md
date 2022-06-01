---
title: Proxying UDP in HTTP
docname: draft-ietf-masque-connect-udp-latest
v: 3
submissiontype: IETF
category: std
area: Transport
wg: MASQUE
number:
date:
consensus: true
venue:
  group: "MASQUE"
  type: "Working Group"
  mail: "masque@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/masque/"
  github: "ietf-wg-masque/draft-ietf-masque-connect-udp"
  latest: "https://ietf-wg-masque.github.io/draft-ietf-masque-connect-udp/draft-ietf-masque-connect-udp.html"
keyword:
  - quic
  - http
  - datagram
  - udp
  - proxy
  - tunnels
  - quic in quic
  - turtles all the way down
  - masque
  - http-ng
author:
  -
    ins: D. Schinazi
    name: David Schinazi
    org: Google LLC
    street: 1600 Amphitheatre Parkway
    city: Mountain View
    region: CA
    code: 94043
    country: United States of America
    email: dschinazi.ietf@gmail.com
normative:
  H1:
    =: I-D.draft-ietf-httpbis-messaging
    display: HTTP/1.1
  H2:
    =: I-D.draft-ietf-httpbis-http2bis
    display: HTTP/2
  H3:
    =: I-D.draft-ietf-quic-http
    display: HTTP/3


--- abstract

This document describes how to proxy UDP in HTTP, similar to how the HTTP
CONNECT method allows proxying TCP in HTTP. More specifically, this document
defines a protocol that allows HTTP clients to create a tunnel for UDP
communications through an HTTP server that acts as a proxy.


--- middle

# Introduction {#introduction}

While HTTP provides the CONNECT method (see {{Section 9.3.6 of
!HTTP=I-D.ietf-httpbis-semantics}}) for creating a TCP {{!TCP=RFC0793}} tunnel
to a proxy, it lacks a method for doing so for UDP {{!UDP=RFC0768}} traffic.

This document describes a protocol for tunnelling UDP to a server acting as a
UDP-specific proxy over HTTP. UDP tunnels are commonly used to create an
end-to-end virtual connection, which can then be secured using QUIC
{{!QUIC=RFC9000}} or another protocol running over UDP. Unlike CONNECT, the UDP
proxy itself is identified with an absolute URL containing the traffic's
destination. Clients generate those URLs using a URI Template
{{!TEMPLATE=RFC6570}}, as described in {{client-config}}.

This protocol supports all versions of HTTP by using HTTP Datagrams
{{!HTTP-DGRAM=I-D.ietf-masque-h3-datagram}}. When using HTTP/2 {{H2}} or HTTP/3
{{H3}}, it uses HTTP Extended CONNECT as described in {{!EXT-CONNECT2=RFC8441}}
and {{!EXT-CONNECT3=I-D.ietf-httpbis-h3-websockets}}. When using HTTP/1.x
{{H1}}, it uses HTTP Upgrade as defined in {{Section 7.8 of HTTP}}.


## Conventions and Definitions {#conventions}

{::boilerplate bcp14-tagged}

In this document, we use the term "UDP proxy" to refer to the HTTP server that
acts upon the client's UDP tunnelling request to open a UDP socket to a target
server, and generates the response to this request. If there are HTTP
intermediaries (as defined in {{Section 3.7 of HTTP}}) between the client and
the UDP proxy, those are referred to as "intermediaries" in this document.

Note that, when the HTTP version in use does not support multiplexing streams
(such as HTTP/1.1), any reference to "stream" in this document represents the
entire connection.


# Client Configuration {#client-config}

HTTP clients are configured to use a UDP proxy with a URI Template
{{!TEMPLATE=RFC6570}} that has the variables "target_host" and "target_port".
Examples are shown below:

~~~
https://masque.example.org/.well-known/masque/udp/{target_host}/{target_port}/
https://proxy.example.org:4443/masque?h={target_host}&p={target_port}
https://proxy.example.org:4443/masque{?target_host,target_port}
~~~
{: #fig-template-examples title="URI Template Examples"}

The following requirements apply to the URI Template:

* The URI Template MUST be a level 3 template or lower.

* The URI Template MUST be in absolute form, and MUST include non-empty scheme,
  authority and path components.

* The path component of the URI Template MUST start with a slash "/".

* All template variables MUST be within the path or query components of the URI.

* The URI template MUST contain the two variables "target_host" and
  "target_port" and MAY contain other variables.

* The URI Template MUST NOT contain any non-ASCII unicode characters and MUST
  only contain ASCII characters in the range 0x21-0x7E inclusive (note that
  percent-encoding is allowed).

* The URI Template MUST NOT use Reserved Expansion ("+" operator), Fragment
  Expansion ("#" operator), Label Expansion with Dot-Prefix, Path Segment
  Expansion with Slash-Prefix, nor Path-Style Parameter Expansion with
  Semicolon-Prefix.

If the client detects that any of the requirements above are not met by a URI
Template, the client MUST reject its configuration and fail the request without
sending it to the UDP proxy. While clients SHOULD validate the requirements
above, some clients MAY use a general-purpose URI Template implementation that
lacks this specific validation.

Since the original HTTP CONNECT method allowed conveying the target host and
port but not the scheme, proxy authority, path, nor query, there exist clients
with proxy configuration interfaces that only allow the user to configure the
proxy host and the proxy port. Client implementations of this specification that
are constrained by such limitations MAY attempt to access UDP proxying
capabilities using the default template, which is defined as:
"https://$PROXY_HOST:$PROXY_PORT/.well-known/masque/udp/{target_host}/{target_port}/"
where $PROXY_HOST and $PROXY_PORT are the configured host and port of the UDP
proxy respectively. UDP proxy deployments SHOULD offer service at this location
if they need to interoperate with such clients.


# Tunnelling UDP over HTTP

To allow negotiation of a tunnel for UDP over HTTP, this document defines the
"connect-udp" HTTP Upgrade Token. The resulting UDP tunnels use the Capsule
Protocol (see {{Section 3.2 of HTTP-DGRAM}}) with HTTP Datagram in the format
defined in {{format}}.

To initiate a UDP tunnel associated with a single HTTP stream, clients issue a
request containing the "connect-udp" upgrade token. The target of the tunnel is
indicated by the client to the UDP proxy via the "target_host" and "target_port"
variables of the URI Template, see {{client-config}}. If the request is
successful, the UDP proxy commits to converting received HTTP Datagrams into UDP
packets and vice versa until the tunnel is closed.

When sending its UDP proxying request, the client SHALL perform URI Template
expansion to determine the path and query of its request. target_host supports
using DNS names, IPv6 literals and IPv4 literals. Note that this URI Template
expansion requires using pct-encoding, so for example if the target_host is
"2001:db8::42", it will be encoded in the URI as "2001%3Adb8%3A%3A42".

By virtue of the definition of the Capsule Protocol (see {{HTTP-DGRAM}}), UDP
proxying requests do not carry any message content. Similarly, successful
UDP proxying responses also do not carry any message content.


## UDP Proxy Handling {#handling}

Upon receiving a UDP proxying request:

 * if the recipient is configured to use another HTTP proxy, it will act as an
   intermediary: it forwards the request to another HTTP server. Note that such
   intermediaries may need to reencode the request if they forward it using a
   version of HTTP that is different from the one used to receive it, as the
   request encoding differs by version (see below).

* otherwise, the recipient will act as a UDP proxy: it extracts the
  "target_host" and "target_port" variables from the URI it has reconstructed
  from the request headers, and establishes a tunnel by directly opening a UDP
  socket to the requested target.

Unlike TCP, UDP is connection-less. The UDP proxy that opens the UDP socket has
no way of knowing whether the destination is reachable. Therefore it needs to
respond to the request without waiting for a packet from the target. However, if
the target_host is a DNS name, the UDP proxy MUST perform DNS resolution before
replying to the HTTP request. If errors occur during this process, the UDP proxy
MUST fail the request and SHOULD send details using an appropriate
"Proxy-Status" header field {{!PROXY-STATUS=I-D.ietf-httpbis-proxy-status}} (for
example, if DNS resolution returns an error, the proxy can use the dns_error
Proxy Error Type from {{Section 2.3.2 of PROXY-STATUS}}).

UDP proxies can use connected UDP sockets if their operating system supports
them, as that allows the UDP proxy to rely on the kernel to only send it UDP
packets that match the correct 5-tuple. If the UDP proxy uses a non-connected
socket, it MUST validate the IP source address and UDP source port on received
packets to ensure they match the client's request. Packets that do not match
MUST be discarded by the UDP proxy.

The lifetime of the socket is tied to the request stream. The UDP proxy MUST
keep the socket open while the request stream is open. If a UDP proxy is
notified by its operating system that its socket is no longer usable (for
example, this can happen when an ICMP "Destination Unreachable" message is
received, see {{Section 3.1 of ?ICMP6=RFC4443}}), it MUST close the request
stream. UDP proxies MAY choose to close sockets due to a period of inactivity,
but they MUST close the request stream when closing the socket. UDP proxies that
close sockets after a period of inactivity SHOULD NOT use a period lower than
two minutes, see {{Section 4.3 of ?BEHAVE=RFC4787}}.

A successful response (as defined in {{resp1}} and {{resp23}}) indicates that
the UDP proxy has opened a socket to the requested target and is willing to
proxy UDP payloads. Any response other than a successful response indicates that
the request has failed, and the client MUST therefore abort the request.

UDP proxies MUST NOT introduce fragmentation at the IP layer when forwarding
HTTP Datagrams onto a UDP socket. In IPv4, the Don't Fragment (DF) bit MUST be
set if possible, to prevent fragmentation on the path. Future extensions MAY
remove these requirements.


## HTTP/1.1 Request {#req1}

When using HTTP/1.1 {{H1}}, a UDP proxying request will meet the following
requirements:

* the method SHALL be "GET".

* the request SHALL include a single "Host" header field containing the origin
  of the UDP proxy.

* the request SHALL include a "Connection" header field with value "Upgrade"
  (note that this requirement is case-insensitive as per {{Section 7.6.1 of
  HTTP}}).

* the request SHALL include an "Upgrade" header field with value "connect-udp".

For example, if the client is configured with URI Template
"https://proxy.example.org/.well-known/masque/udp/{target_host}/{target_port}/"
and wishes to open a
UDP proxying tunnel to target 192.0.2.42:443, it could send the following
request:

~~~
GET https://proxy.example.org/.well-known/masque/udp/192.0.2.42/443/ HTTP/1.1
Host: proxy.example.org
Connection: Upgrade
Upgrade: connect-udp
~~~
{: #fig-req-h1 title="Example HTTP/1.1 Request"}

In HTTP/1.1, this protocol uses the GET method to mimic the design of the
WebSocket Protocol {{?WEBSOCKET=RFC6455}}.


## HTTP/1.1 Response {#resp1}

The UDP proxy SHALL indicate a successful response by replying with the
following requirements:

* the HTTP status code on the response SHALL be 101 (Switching Protocols).

* the reponse SHALL include a single "Connection" header field with value
  "Upgrade" (note that this requirement is case-insensitive as per {{Section
  7.6.1 of HTTP}}).

* the response SHALL include a single "Upgrade" header field with value
  "connect-udp".

* the response SHALL NOT include any "Transfer-Encoding" or "Content-Length"
  header fields.

If any of these requirements are not met, the client MUST treat this proxying
attempt as failed and abort the connection.

For example, the UDP proxy could respond with:

~~~
HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: connect-udp
~~~
{: #fig-resp-h1 title="Example HTTP/1.1 Response"}


## HTTP/2 and HTTP/3 Requests {#req23}

When using HTTP/2 {{H2}} or HTTP/3 {{H3}}, UDP proxying requests use Extended
CONNECT. This requires that servers send an HTTP Setting as specified in
{{EXT-CONNECT2}} and {{EXT-CONNECT3}}, and that requests use HTTP pseudo-header
fields with the following requirements:

* The ":method" pseudo-header field SHALL be "CONNECT".

* The ":protocol" pseudo-header field SHALL be "connect-udp".

* The ":authority" pseudo-header field SHALL contain the authority of the UDP
  proxy.

* The ":path" and ":scheme" pseudo-header fields SHALL NOT be empty. Their
  values SHALL contain the scheme and path from the URI Template after the URI
  template expansion process has been completed.

A UDP proxying request that does not conform to these restrictions is
malformed (see {{Section 8.1.1 of H2}}).

For example, if the client is configured with URI Template
"https://proxy.example.org/{target_host}/{target_port}/" and wishes to open a
UDP proxying tunnel to target 192.0.2.42:443, it could send the following
request:

~~~
HEADERS
:method = CONNECT
:protocol = connect-udp
:scheme = https
:path = /.well-known/masque/udp/192.0.2.42/443/
:authority = proxy.example.org
~~~
{: #fig-req-h2 title="Example HTTP/2 Request"}


## HTTP/2 and HTTP/3 Responses {#resp23}

The UDP proxy SHALL indicate a successful response by replying with any 2xx
(Successful) HTTP status code, without any "Transfer-Encoding" or
"Content-Length" header fields.

If any of these requirements are not met, the client MUST treat this proxying
attempt as failed and abort the request.

For example, the UDP proxy could respond with:

~~~
HEADERS
:status = 200
~~~
{: #fig-resp-h2 title="Example HTTP/2 Response"}


## Note About Draft Versions

\[\[RFC editor: please remove this section before publication.]]

In order to allow implementations to support multiple draft versions of this
specification during its development, we introduce the "connect-udp-version"
header field. When sent by the client, it contains a list of draft numbers
supported by the client (e.g., "connect-udp-version: 0, 2"). When sent by the
UDP proxy, it contains a single draft number selected by the UDP proxy from the
list provided by the client (e.g., "connect-udp-version: 2"). Sending this
header field is RECOMMENDED but not required. The "connect-udp-version" header
field is a List Structured Field, see {{Section 3.1 of !STRUCT-FIELD=RFC8941}}.
Each list member MUST be an Integer.


# Context Identifiers {#context-id}

This protocol allows future extensions to exchange HTTP Datagrams which carry
different semantics from UDP payloads. Some of these extensions can augment UDP
payloads with additional data, while others can exchange data that is completely
separate from UDP payloads. In order to accomplish this, all HTTP Datagrams
associated with UDP Proxying request streams start with a context ID, see
{{format}}.

Context IDs are 62-bit integers (0 to 2<sup>62</sup>-1). Context IDs are encoded
as variable-length integers, see {{Section 16 of QUIC}}. The context ID value of
0 is reserved for UDP payloads, while non-zero values are dynamically allocated:
non-zero even-numbered context IDs are client-allocated, and odd-numbered
context IDs are proxy-allocated. The context ID namespace is tied to a given
HTTP request: it is possible for a context ID with the same numeric value to be
simultaneously allocated in distinct requests, potentially with different
semantics. Context IDs MUST NOT be re-allocated within a given HTTP namespace
but MAY be allocated in any order. The context ID allocation restrictions to the
use of even-numbered and odd-numbered context IDs exist in order to avoid the
need for synchronisation between endpoints. However, once a context ID has been
allocated, those restrictions do not apply to the use of the context ID: it can
be used by any client or UDP proxy, independent of which endpoint initially
allocated it.

Registration is the action by which an endpoint informs its peer of the
semantics and format of a given context ID. This document does not define how
registration occurs. Future extensions MAY use HTTP header fields or capsules to
register contexts. Depending on the method being used, it is possible for
datagrams to be received with Context IDs which have not yet been registered,
for instance due to reordering of the packet containing the datagram and the
packet containing the registration message during transmission.


# HTTP Datagram Payload Format {#format}

When HTTP Datagrams (see {{HTTP-DGRAM}}) are associated with UDP proxying
request streams, the HTTP Datagram Payload field has the format defined in
{{dgram-format}}. Note that when HTTP Datagrams are encoded using QUIC DATAGRAM
frames, the Context ID field defined below directly follows the Quarter Stream
ID field which is at the start of the QUIC DATAGRAM frame payload:

~~~
UDP Proxying HTTP Datagram Payload {
  Context ID (i),
  Payload (..),
}
~~~
{: #dgram-format title="UDP Proxying HTTP Datagram Format"}

Context ID:

: A variable-length integer (see {{Section 16 of QUIC}}) that contains the value
of the Context ID. If an HTTP/3 datagram which carries an unknown Context ID is
received, the receiver SHALL either drop that datagram silently or buffer it
temporarily (on the order of a round trip) while awaiting the registration of
the corresponding Context ID.

Payload:

: The payload of the datagram, whose semantics depend on value of the previous
field. Note that this field can be empty.
{: spacing="compact"}

UDP packets are encoded using HTTP Datagrams with the Context ID set to zero.
When the Context ID is set to zero, the Payload field contains the
unmodified payload of a UDP packet (referred to as "data octets" in {{UDP}}).

By virtue of the definition of the UDP header {{UDP}}, it is not possible to
encode UDP payloads longer than 65527 bytes. Therefore, endpoints MUST NOT send
HTTP Datagrams with a Payload field longer than 65527 using Context ID zero. An
endpoint that receives a DATAGRAM capsule using Context ID zero whose Payload
field is longer than 65527 MUST abort the stream. If a UDP proxy knows it can
only send out UDP packets of a certain length due to its underlying link MTU, it
SHOULD discard incoming DATAGRAM capsules using Context ID zero whose Payload
field is longer than that limit without buffering the capsule contents.

If a UDP proxy receives an HTTP Datagram before it has received the
corresponding request, it SHALL either drop that HTTP Datagram silently or
buffer it temporarily (on the order of a round trip) while awaiting the
corresponding request.

Note that buffering datagrams (either because the request was not yet received,
or because the Context ID is not yet known) consumes resources. Receivers that
buffer datagrams SHOULD apply buffering limits in order to reduce the risk of
resource exhaustion occuring. For example, receivers can limit the total number
of buffered datagrams, or the cumulative size of buffered datagrams, on a
per-stream, per-context, or per-connection basis.

A client MAY optimistically start sending UDP packets in HTTP Datagrams before
receiving the response to its UDP proxying request. However, implementors should
note that such proxied packets may not be processed by the UDP proxy if it
responds to the request with a failure, or if the proxied packets are received
by the UDP proxy before the request and the UDP proxy chooses to not buffer them.


# Performance Considerations {#performance}

UDP proxies SHOULD strive to avoid increasing burstiness of UDP traffic: they
SHOULD NOT queue packets in order to increase batching.

When the protocol running over UDP that is being proxied uses congestion
control (e.g., {{QUIC}}), the proxied traffic will incur at least two nested
congestion controllers. This can reduce performance but the underlying
HTTP connection MUST NOT disable congestion control unless it has an
out-of-band way of knowing with absolute certainty that the inner traffic is
congestion-controlled.

If a client or UDP proxy with a connection containing a UDP proxying request
stream disables congestion control, it MUST NOT signal Explicit Congestion
Notification (ECN) {{!ECN=RFC3168}} support on that connection. That is, it MUST
mark all IP headers with the Not-ECT codepoint. It MAY continue to report ECN
feedback via QUIC ACK_ECN frames or the TCP "ECE" bit, as the peer may not have
disabled congestion control.

When the protocol running over UDP that is being proxied uses loss recovery
(e.g., {{QUIC}}), and the underlying HTTP connection runs over TCP, the proxied
traffic will incur at least two nested loss recovery mechanisms. This can reduce
performance as both can sometimes independently retransmit the same data. To
avoid this, UDP proxying SHOULD be performed over HTTP/3 to allow leveraging the
QUIC DATAGRAM frame.


## MTU Considerations

When using HTTP/3 with the QUIC Datagram extension {{!DGRAM=RFC9221}}, UDP
payloads are transmitted in QUIC DATAGRAM frames. Since those cannot be
fragmented, they can only carry payloads up to a given length determined by the
QUIC connection configuration and the path MTU. If a UDP proxy is using QUIC
DATAGRAM frames and it receives a UDP payload from the target that will not fit
inside a QUIC DATAGRAM frame, the UDP proxy SHOULD NOT send the UDP payload in a
DATAGRAM capsule, as that defeats the end-to-end unreliability characteristic
that methods such as Datagram Packetization Layer Path MTU Discovery (DPLPMTUD)
depend on {{?DPLPMTUD=RFC8899}}. In this scenario, the UDP proxy SHOULD drop the
UDP payload and send an ICMP "Packet Too Big" message to the target, see
{{Section 3.2 of ICMP6}}.


## Tunneling of ECN Marks

UDP proxying does not create an IP-in-IP tunnel, so the guidance in
{{?ECN-TUNNEL=RFC6040}} about transferring ECN marks between inner and outer IP
headers does not apply. There is no inner IP header in UDP proxying tunnels.

Note that UDP proxying clients do not have the ability in this specification to
control the ECN codepoints on UDP packets the UDP proxy sends to the target, nor
can UDP proxies communicate the markings of each UDP packet from target to UDP
proxy.

A UDP proxy MUST ignore ECN bits in the IP header of UDP packets received from
the target, and MUST set the ECN bits to Not-ECT on UDP packets it sends to the
target. These do not relate to the ECN markings of packets sent between client
and UDP proxy in any way.


# Security Considerations {#security}

There are significant risks in allowing arbitrary clients to establish a tunnel
to arbitrary targets, as that could allow bad actors to send traffic and have it
attributed to the UDP proxy. HTTP servers that support UDP proxying ought to
restrict its use to authenticated users.

Because the CONNECT method creates a TCP connection to the target, the target
has to indicate its willingness to accept TCP connections by responding with a
TCP SYN-ACK before the CONNECT proxy can send it application data. UDP doesn't
have this property, so a UDP proxy could send more data to an unwilling target
than a CONNECT proxy. However, in practice denial of service attacks target open
TCP ports so the TCP SYN-ACK does not offer much protection in real scenarios.
While a UDP proxy could potentially limit the number of UDP packets it is
willing to forward until it has observed a response from the target, that is
unlikely to provide any protection against denial of service attacks because
such attacks target open UDP ports where the protocol running over UDP would
respond, and that would be interpreted as willingness to accept UDP by the UDP
proxy.

UDP sockets for UDP proxying have a different lifetime than TCP sockets for
CONNECT, therefore implementors would be well served to follow the advice in
{{handling}} if they base their UDP proxying implementation on a preexisting
implementation of CONNECT.

The security considerations described in {{HTTP-DGRAM}} also apply here.


# IANA Considerations {#iana}

## HTTP Upgrade Token {#iana-upgrade}

This document will request IANA to register "connect-udp" in the "HTTP Upgrade
Tokens" registry maintained at
<[](https://www.iana.org/assignments/http-upgrade-tokens)>.

Value:

: connect-udp

Description:

: Proxying of UDP Payloads

Expected Version Tokens:

: None

Reference:

: This document
{: spacing="compact"}


## Well-Known URI {#iana-uri}

This document will request IANA to register "masque/udp" in the "Well-Known
URIs" registry maintained at
<[](https://www.iana.org/assignments/well-known-uris)>.

URI Suffix:

: masque/udp

Change Controller:

: IETF

Reference:

: This document

Status:

: permanent (if this document is approved)

Related Information:

: Includes all resources identified with the path prefix
"/.well-known/masque/udp/"
{: spacing="compact"}


--- back

# Acknowledgments {#acknowledgments}
{:numbered="false"}

This document is a product of the MASQUE Working Group, and the author thanks
all MASQUE enthusiasts for their contibutions. This proposal was inspired
directly or indirectly by prior work from many people. In particular, the author
would like to thank Eric Rescorla for suggesting to use an HTTP method to proxy
UDP. The author is indebted to Mark Nottingham and Lucas Pardue for the many
improvements they contributed to this document. The extensibility design in this
document came out of the HTTP Datagrams Design Team, whose members were Alan
Frindell, Alex Chernyakhovsky, Ben Schwartz, Eric Rescorla, Lucas Pardue, Marcus
Ihlar, Martin Thomson, Mike Bishop, Tommy Pauly, Victor Vasiliev, and the author
of this document.
