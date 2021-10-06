---
title: UDP Proxying Support for HTTP
abbrev: HTTP UDP CONNECT
docname: draft-ietf-masque-connect-udp-latest
category: std
wg: MASQUE

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: "D. Schinazi"
    name: "David Schinazi"
    organization: "Google LLC"
    street: "1600 Amphitheatre Parkway"
    city: "Mountain View, California 94043"
    country: "United States of America"
    email: dschinazi.ietf@gmail.com

normative:
  MESSAGING: I-D.ietf-httpbis-messaging
  SEMANTICS: I-D.ietf-httpbis-semantics

--- abstract

This document describes how to proxy UDP over HTTP. Similar to how the CONNECT
method allows proxying TCP over HTTP, this document defines a new mechanism to
proxy UDP. It is built using HTTP Extended CONNECT.


--- middle

# Introduction {#introduction}

This document describes how to proxy UDP over HTTP. Similar to how the CONNECT
method (see {{Section 9.3.6 of SEMANTICS}}) allows proxying TCP {{!TCP=RFC0793}}
over HTTP, this document defines a new mechanism to proxy UDP {{!UDP=RFC0768}}.

UDP Proxying supports all versions of HTTP and uses HTTP Datagrams
{{!HTTP-DGRAM=I-D.ietf-masque-h3-datagram}}. When using HTTP/2 or HTTP/3, UDP
proxying uses HTTP Extended CONNECT as described in {{!EXT-CONNECT2=RFC8441}}
and {{!EXT-CONNECT3=I-D.ietf-httpbis-h3-websockets}}. When using HTTP/1.x, UDP
proxying uses HTTP Upgrade as defined in {{Section 7.8 of SEMANTICS}}.


## Conventions and Definitions {#conventions}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

In this document, we use the term "proxy" to refer to the HTTP server that
opens the UDP socket and responds to the UDP proxying request. If there are
HTTP intermediaries (as defined in {{Section 3.7 of SEMANTICS}}) between the
client and the proxy, those are referred to as "intermediaries" in this
document.

Note that, when the HTTP version in use does not support multiplexing streams
(such as HTTP/1.1), any reference to "stream" in this document represents the
entire connection.


# Configuration of Clients {#client-config}

Clients are configured to use UDP Proxying over HTTP via an URI Template
{{!TEMPLATE=RFC6570}}. The URI template MUST contain exactly two variables:
"target_host" and "target_port". Examples are shown below:

~~~
https://masque.example.org/{target_host}/{target_port}/
https://proxy.example.org:4443/masque?h={target_host}&p={target_port}
https://proxy.example.org:4443/masque{?target_host,target_port}
~~~
{: #fig-template-examples title="URI Template Examples"}

Since the original HTTP CONNECT method allowed conveying the target host and
port but not the scheme, proxy authority, path, nor query, there exist proxy
configuration interfaces that only allow the user to configure the proxy host
and the proxy port. Client implementations of this specification that are
constrained by such limitations MUST use the default template which is defined
as: "https://{proxy_host}:{proxy_port}/{target_host}/{target_port}/" where
"proxy_host" and "proxy_port" are the configured host and port of the proxy
respectively. Proxy deployments SHOULD use the default template to facilitate
interoperability with such clients.


# HTTP Exchanges

This document defines the "masque-udp" HTTP Upgrade Token. "masque-udp" uses
the Capsule Protocol as defined in {{HTTP-DGRAM}}.

A "masque-udp" request requests that the recipient establish a tunnel over a
single HTTP stream to the destination target server identified by the
"target_host" and "target_port" variables of the URI template (see
{{client-config}}). If the request is successful, the proxy commits to
converting received HTTP Datagrams into UDP packets and vice versa until the
tunnel is closed. Tunnels are commonly used to create an end-to-end virtual
connection, which can then be secured using QUIC {{!QUIC=RFC9000}} or another
protocol running over UDP.

When sending its UDP proxying request, the client SHALL perform URI template
expansion to determine the path and query of its request. target_host supports
using DNS names, IPv6 literals and IPv4 literals. Note that this URI template
expansion requires using pct-encoding, so for example if the target_host is
"2001:db8::42", it will be encoded in the URI as "2001%3Adb8%3A%3A42".

A payload within a UDP proxying request message has no defined semantics; a UDP
proxying request with a non-empty payload is malformed.

Responses to UDP proxying requests are not cacheable.

## Proxy Handling

Upon receiving a UDP proxying request, the recipient proxy extracts the
"target_host" and "target_port" variables from the URI it has reconstructed
from the request headers, and establishes a tunnel by directly opening a UDP
socket to the requested target.

Unlike TCP, UDP is connection-less. The proxy that opens the UDP socket has no
way of knowing whether the destination is reachable. Therefore it needs to
respond to the request without waiting for a packet from the target. However,
if the target_host is a DNS name, the proxy MUST perform DNS resolution before
replying to the HTTP request. If DNS resolution fails, the proxy MUST fail the
request and SHOULD send details using the Proxy-Status header
{{?PROXY-STATUS=I-D.ietf-httpbis-proxy-status}}.

Proxies can use connected UDP sockets if their operating system supports them,
as that allows the proxy to rely on the kernel to only send it UDP packets that
match the correct 5-tuple. If the proxy uses a non-connected socket, it MUST
validate the IP source address and UDP source port on received packets to
ensure they match the client's request. Packets that do not match MUST be
discarded by the proxy.

The lifetime of the socket is tied to the request stream. The proxy MUST keep
the socket open while the request stream is open. Proxies MAY choose to close
sockets due to a period of inactivity, but they MUST close the request stream
before closing the socket. If a proxy is notified by its operating system that
its socket is no longer usable, it MUST close the request stream.

A successful response (as defined in {{resp1}} and {{resp23}}) indicates that
the proxy has opened a socket to the requested target and is willing to proxy
UDP payloads. Any response other than a successful response indicates that the
request has failed, and the client MUST therefore abort the request.


## HTTP Request over HTTP/1.1 {#req1}

When using HTTP/1.1, a UDP proxying request will meet the following requirements:

* the method SHALL be "CONNECT".

* the request-target SHALL use absolute-form (see {{Section 3.2.2 of
  MESSAGING}}).

* the request SHALL include a single Host header containing the origin of the
  proxy.

* the request SHALL include a single "Connection" header with value "Upgrade".

* the request SHALL include a single "Upgrade" header with value "masque-udp".

For example, if the client is configured with URI template
"https://proxy.example.org/{target_host}/{target_port}/" and wishes to open a
UDP proxying tunnel to target 192.0.2.42:443, it could send the following
request:

~~~
CONNECT https://proxy.example.org/192.0.2.42/443/ HTTP/1.1
Host: proxy.example.org
Connection: upgrade
Upgrade: masque-udp
~~~
{: #fig-req-h1 title="Example HTTP Request over HTTP/1.1"}


## HTTP Response over HTTP/1.1 {#resp1}

The proxy SHALL indicate a successful response by replying with the following
requirements:

* the HTTP status code on the response SHALL be 101 (Switching Protocols).

* the reponse SHALL include a single "Connection" header with value "Upgrade".

* the response SHALL include a single "Upgrade" header with value "masque-udp".

* the response SHALL NOT include any Transfer-Encoding or Content-Length header
  fields.

If any of these requirements are not met, the client MUST treat this proxying
attempt as failed and abort the connection.

For example, the proxy could respond with:

~~~
HTTP/1.1 101 Switching Protocols
Connection: upgrade
Upgrade: masque-udp
~~~
{: #fig-resp-h1 title="Example HTTP Response over HTTP/1.1"}


## HTTP Request over HTTP/2 and HTTP/3 {#req23}

When using HTTP/2 {{!H2=RFC7540}} or HTTP/3 {{!H3=I-D.ietf-quic-http}}, UDP
proxying requests use HTTP pseudo-headers with the following requirements:

* The ":method" pseudo-header field SHALL be "CONNECT".

* The ":protocol" pseudo-header field SHALL be "masque-udp".

* The ":authority" pseudo-header field SHALL contain the authority of the proxy.

* The ":path" and ":scheme" pseudo-header fields SHALL NOT be empty. Their
  values SHALL contain the scheme and path from the URI template after the URI
  template expansion process has been completed.

A UDP proxying request that does not conform to these restrictions is
malformed (see {{Section 8.1.2.6 of H2}}).

For example, if the client is configured with URI template
"https://proxy.example.org/{target_host}/{target_port}/" and wishes to open a
UDP proxying tunnel to target 192.0.2.42:443, it could send the following
request:

~~~
HEADERS
:method = CONNECT
:protocol = masque-udp
:scheme = https
:path = /192.0.2.42/443/
:authority = proxy.example.org
~~~
{: #fig-req-h2 title="Example HTTP Request over HTTP/2"}


## HTTP Response over HTTP/2 and HTTP/3 {#resp23}

The proxy SHALL indicate a successful response by replying with any 2xx
(Successful) HTTP status code, without any Transfer-Encoding or Content-Length
header fields.

If any of these requirements are not met, the client MUST treat this proxying
attempt as failed and abort the request.

For example, the proxy could respond with:

~~~
HEADERS
:status = 200
~~~
{: #fig-resp-h2 title="Example HTTP Response over HTTP/2"}


# Encoding of Proxied UDP Packets {#datagram-encoding}

UDP packets are encoded using HTTP Datagrams {{HTTP-DGRAM}} with the
UDP_PAYLOAD HTTP Datagram Format Type (see value in {{iana-format-type}}). When
using the UDP_PAYLOAD HTTP Datagram Format Type, the payload of a UDP packet
(referred to as "data octets" in {{UDP}}) is sent unmodified in the "HTTP
Datagram Payload" field of an HTTP Datagram.

In order to use HTTP Datagrams, the client will first decide whether or not to
use HTTP Datagram Contexts and then register its context ID (or lack thereof)
using the corresponding registration capsule, see {{HTTP-DGRAM}}.

When sending a REGISTER_DATAGRAM_CONTEXT or REGISTER_DATAGRAM_NO_CONTEXT
capsule using the "Datagram Format Type" set to UDP_PAYLOAD, the "Datagram
Format Additional Data" field SHALL be empty. Servers MUST NOT register
contexts using the UDP_PAYLOAD HTTP Datagram Format Type. Clients MUST NOT
register more than one context using the UDP_PAYLOAD HTTP Datagram Format Type.
Endpoints MUST NOT close contexts using the UDP_PAYLOAD HTTP Datagram Format
Type. If an endpoint detects a violation of any of these requirements, it MUST
abort the stream.

Clients MAY optimistically start sending proxied UDP packets before receiving
the response to its UDP proxying request, noting however that those may not be
processed by the proxy if it responds to the request with a failure, or if the
datagrams are received by the proxy before the request.

Extensions to this mechanism MAY define new HTTP Datagram Format Types in order
to use different semantics or encodings for UDP payloads.


# Performance Considerations {#performance}

Proxies SHOULD strive to avoid increasing burstiness of UDP traffic: they
SHOULD NOT queue packets in order to increase batching.

When the protocol running over UDP that is being proxied uses congestion
control (e.g., {{QUIC}}), the proxied traffic will incur at least two nested
congestion controllers. This can reduce performance but the underlying
HTTP connection MUST NOT disable congestion control unless it has an
out-of-band way of knowing with absolute certainty that the inner traffic is
congestion-controlled.

If a client or proxy with a connection containing a UDP proxying request stream
disables congestion control, it MUST NOT signal ECN support on that connection.
That is, it MUST mark all IP headers with the Not-ECT codepoint. It MAY
continue to report ECN feedback via ACK_ECN frames, as the peer may not have
disabled congestion control.

When the protocol running over UDP that is being proxied uses loss recovery
(e.g., {{QUIC}}), and the underlying HTTP connection runs over TCP, the proxied
traffic will incur at least two nested loss recovery mechanisms. This can
reduce performance as both can sometimes independently retransmit the same
data. To avoid this, HTTP/3 datagrams SHOULD be used.


## Tunneling of ECN Marks

UDP proxying does not create an IP-in-IP tunnel, so the guidance in {{?RFC6040}}
about transferring ECN marks between inner and outer IP headers does not apply.
There is no inner IP header in UDP proxying tunnels.

Note that UDP proxying clients do not have the ability in this specification to
control the ECN codepoints on UDP packets the proxy sends to the server, nor can
proxies communicate the markings of each UDP packet from server to proxy.

A UDP proxy MUST ignore ECN bits in the IP header of UDP packets received from
the server, and MUST set the ECN bits to Not-ECT on UDP packets it sends to the
server. These do not relate to the ECN markings of packets sent between client
and proxy in any way.

# Security Considerations {#security}

There are significant risks in allowing arbitrary clients to establish a tunnel
to arbitrary servers, as that could allow bad actors to send traffic and have
it attributed to the proxy. Proxies that support UDP proxying SHOULD restrict
its use to authenticated users.

Because the CONNECT method creates a TCP connection to the target, the target
has to indicate its willingness to accept TCP connections by responding with a
TCP SYN-ACK before the proxy can send it application data. UDP doesn't have
this property, so a UDP proxy could send more data to an unwilling target than
a CONNECT proxy. However, in practice denial of service attacks target open TCP
ports so the TCP SYN-ACK does not offer much protection in real scenarios.
Proxies MUST NOT introspect the contents of UDP payloads as that would lead to
ossification of UDP-based protocols by proxies.


# IANA Considerations {#iana}

## HTTP Upgrade Token {#iana-upgrade}

This document will request IANA to register "masque-udp" in the
HTTP Upgrade Token Registry maintained at
<[](https://www.iana.org/assignments/http-upgrade-tokens)>.

Value:

: masque-udp

Description:

: Proxying of UDP Payloads.

Expected Version Tokens:

: None.

Reference:

: This document.


## Datagram Format Type {#iana-format-type}

This document will request IANA to register UDP_PAYLOAD in the "HTTP Datagram
Format Types" registry established by {{HTTP-DGRAM}}.

|    Type     |   Value   | Specification |
|:------------|:----------|:--------------|
| UDP_PAYLOAD | 0xff6f00  | This Document |
{: #iana-format-type-table title="Registered Datagram Format Type"}


--- back

# Acknowledgments {#acknowledgments}
{:numbered="false"}

This document is a product of the MASQUE Working Group, and the author thanks
all MASQUE enthusiasts for their contibutions. This proposal was inspired
directly or indirectly by prior work from many people. In particular, the
author would like to thank Eric Rescorla for suggesting to use an HTTP method
to proxy UDP. Thanks to Lucas Pardue for their inputs on this document.
