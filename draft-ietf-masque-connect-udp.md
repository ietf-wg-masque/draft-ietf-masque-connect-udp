---
title: The CONNECT-UDP HTTP Method
abbrev: CONNECT-UDP
docname: draft-ietf-masque-connect-udp-latest
category: std

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


--- abstract

This document describes the CONNECT-UDP HTTP method. CONNECT-UDP is similar
to the HTTP CONNECT method, but it uses UDP instead of TCP.

Discussion of this work is encouraged to happen on the MASQUE IETF mailing
list [](masque@ietf.org) or on the GitHub repository which contains the draft:
[](https://github.com/ietf-wg-masque/draft-ietf-masque-connect-udp).


--- middle

# Introduction {#introduction}

This document describes the CONNECT-UDP HTTP method. CONNECT-UDP is similar
to the HTTP CONNECT method (see section 4.3.6 of {{!RFC7231}}), but it
uses UDP {{!UDP=RFC0768}} instead of TCP {{!TCP=RFC0793}}.


Discussion of this work is encouraged to happen on the MASQUE IETF mailing
list [](masque@ietf.org) or on the GitHub repository which contains the draft:
[](https://github.com/ietf-wg-masque/draft-ietf-masque-connect-udp).


## Conventions and Definitions {#conventions}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

In this document, we use the term "proxy" to refer to the HTTP server that
opens the UDP socket and responds to the CONNECT-UDP request. If there are
HTTP intermediaries (as defined in Section 2.3 of {{RFC7230}}) between the
client and the proxy, those are referred to as "intermediaries" in this
document.


# Supported HTTP Versions {#http-versions}

The CONNECT-UDP method is defined for all versions of HTTP starting with HTTP/2. When the HTTP
version used runs over QUIC {{!QUIC=I-D.ietf-quic-transport}}, UDP payloads
can be sent over QUIC DATAGRAM frames {{!DGRAM=I-D.ietf-quic-datagram}}.
Otherwise they are sent on the stream where the CONNECT-UDP request was made.


# The CONNECT-UDP Method {#connect-udp-method}

The CONNECT-UDP method requests that the recipient establish a tunnel over a
single HTTP stream to the destination origin server identified by the
request-target and, if successful, thereafter restrict its behavior to blind
forwarding of packets, in both directions, until the tunnel is closed.
Tunnels are commonly used to create an end-to-end virtual connection, which can
then be secured using QUIC or another protocol running over UDP.

The request-target of a CONNECT-UDP request is a URI {{!RFC3986}} which uses
the "masque" scheme and an immutable path of "/". For example:

~~~
     CONNECT-UDP masque://target.example.com:443/ HTTP/1.1
     Host: target.example.com:443
~~~

When using HTTP/2 {{!H2=RFC7540}} or later, CONNECT-UDP requests use HTTP
pseudo-headers with the following requirements:

* The ":method" pseudo-header field is set to "CONNECT-UDP".

* The ":scheme" pseudo-header field is set to "masque".

* The ":path" pseudo-header field is set to "/".

* The ":authority" pseudo-header field contains the host and port to connect
to (similar to the authority-form of the request-target of CONNECT requests;
see {{!RFC7230}}, Section 5.3).

A CONNECT-UDP request that does not conform to these restrictions is
malformed (see {{H2}}, Section 8.1.2.6).

The recipient proxy establishes a tunnel by directly opening a UDP socket to
the request-target. Any 2xx (Successful) response indicates that the proxy has
opened a socket to the request-target and is willing to proxy UDP payloads.
Any response other than a successful response indicates that the tunnel has not
yet been formed.

A proxy MUST NOT send any Transfer-Encoding or Content-Length header fields
in a 2xx (Successful) response to CONNECT-UDP. A client MUST treat a response
to CONNECT-UDP containing any Content-Length or Transfer-Encoding header
fields as malformed.

A payload within a CONNECT-UDP request message has no defined semantics;
a CONNECT-UDP request with a non-empty payload is malformed. Note that the
CONNECT-UDP stream is used to convey UDP packets, but they are not semantically
part of the request or response themselves.

Responses to the CONNECT-UDP method are not cacheable.


# Datagram Encoding of Proxied UDP Packets {#datagram-encoding}

When the HTTP connection supports HTTP/3 datagrams
{{!H3DGRAM=I-D.schinazi-masque-h3-datagram}}, UDP packets can be encoded using
QUIC DATAGRAM frames. This support is ascertained by checking the received
value of the H3_DATAGRAM SETTINGS Parameter.

If the client has both sent and received the H3_DATAGRAM SETTINGS Parameter
with value 1 on this connection, it SHOULD attempt to use HTTP/3 datagrams.
This is accomplished by requesting a datagram flow identifier from the flow
identifier allocation service {{H3DGRAM}}. That service generates an even flow
identifier, and the client sends it to the proxy by sending an HTTP/3
REGISTER_DATAGRAM_FLOW_ID frame with on the stream that carried the CONNECT-UDP
request.

The proxy that is creating the UDP socket to the destination responds to the
CONNECT-UDP request with a 2xx (Successful) response, and sends an HTTP/3
REGISTER_DATAGRAM_FLOW_ID frame with the same flow ID from the
REGISTER_DATAGRAM_FLOW_ID frame it received. Once the client has
received the REGISTER_DATAGRAM_FLOW_ID frame and the successful response, it knows
that it can use the HTTP/3 datagram encoding to send proxied UDP packets for
this particular request. It then encodes the payload of UDP datagrams into the
payload of HTTP/3 datagrams.

When the proxy processes a new REGISTER_DATAGRAM_FLOW_ID frame, it MUST ensure that the
flow identifier is not equal to flow identifiers from other
requests: if it is, the proxy MUST reject the request with a 4xx (Client
Error) status code. Extensions MAY weaken or remove this requirement.

Clients MAY optimistically start sending proxied UDP packets before receiving
the REGISTER_DATAGRAM_FLOW_ID frame and response to its CONNECT-UDP request, noting however that those may not be
processed by the proxy if it responds to the CONNECT-UDP request with a
failure, or if the datagrams
arrive before the CONNECT-UDP request.


# Stream Encoding of Proxied UDP Packets {#stream-encoding}

The RELIABLE_DATAGRAM HTTP/3 frame can be used to convey UDP payloads.
This is necessary when the version of HTTP in use does not
support QUIC DATAGRAM frames, but MAY also be used when datagrams are
supported. Note that empty UDP payloads are allowed.


# Proxy Handling {#proxy-handling}

Unlike TCP, UDP is connection-less. The proxy that opens the UDP socket has no
way of knowing whether the destination is reachable. Therefore it needs to
respond to the CONNECT-UDP request without waiting for a TCP SYN-ACK.

Proxies can use connected UDP sockets if their operating system supports them,
as that allows the proxy to rely on the kernel to only send it UDP
packets that match the correct 5-tuple. If the proxy uses a non-connected
socket, it MUST validate the IP source address and UDP source port on received
packets to ensure they match the client's CONNECT-UDP request. Packets that do
not match MUST be discarded by the proxy.

The lifetime of the socket is tied to the CONNECT-UDP stream. The proxy MUST
keep the socket open while the CONNECT-UDP stream is open. Proxies MAY choose
to close sockets due to a period of inactivity, but they MUST close the
CONNECT-UDP stream before closing the socket.


# HTTP Intermediaries {#intermediaries}

HTTP/3 DATAGRAM flow identifiers are specific to a given HTTP/3 connection.
However, in some cases, an HTTP request may travel across multiple HTTP
connections if there are HTTP intermediaries involved; see Section 2.3 of
{{RFC7230}}.

Intermediaries that support both CONNECT-UDP and HTTP/3 datagrams MUST
negotiate flow identifiers separately on the client-facing and server-facing
connections. This is accomplished by having the intermediary parse the
REGISTER_DATAGRAM_FLOW_ID HTTP/3 frame.
The intermediary uses its own flow identifier
allocation service to allocate a flow identifier for the server-facing
connection, and sends its own REGISTER_DATAGRAM_FLOW_ID HTTP/3 frames.

Intermediaries MUST NOT echo nor forward named elements or parameters on the
"Datagram-Flow-Id" header if they do not understand their semantics.
Extensions to CONNECT-UDP that leverage named elements or parameters in the
"Datagram-Flow-Id" header MUST specify how they are handled by intermediaries.


# Performance Considerations {#performance}

Proxies SHOULD strive to avoid increasing burstiness of UDP traffic: they
SHOULD NOT queue packets in order to increase batching.

When the protocol running over UDP that is being proxied uses congestion
control (e.g., {{QUIC}}), the proxied traffic will incur at least two nested
congestion controllers. This can reduce performance but the underlying
HTTP connection MUST NOT disable congestion control unless it has an
out-of-band way of knowing with absolute certainty that the inner traffic is
congestion-controlled.

If a client or proxy with a connection containing a CONNECT-UDP stream disables
congestion control, it MUST NOT signal ECN support on that connection. That is,
it MUST mark all IP headers with the Not-ECT codepoint. It MAY continue to
report ECN feedback via ACK_ECN frames, as the peer may not have disabled
congestion control.

When the protocol running over UDP that is being proxied uses loss recovery
(e.g., {{QUIC}}), and the underlying HTTP connection runs over TCP, the proxied
traffic will incur at least two nested loss recovery mechanisms. This can
reduce performance as both can sometimes independently retransmit the same
data. To avoid this, HTTP/3 datagrams SHOULD be used.


## Tunneling of ECN Marks

CONNECT-UDP does not create an IP-in-IP tunnel, so the guidance in {{?RFC6040}}
about transferring ECN marks between inner and outer IP headers does not apply.
There is no inner IP header in CONNECT-UDP tunnels.

Note that CONNECT-UDP clients do not have the ability in this specification to
control the ECN codepoints on UDP packets the proxy sends to the server, nor can
proxies communicate the markings of each UDP packet from server to proxy.

A CONNECT-UDP proxy MUST ignore ECN bits in the IP header of UDP packets
received from the server, and MUST set the ECN bits to Not-ECT on UDP packets
it sends to the server. These do not relate to the ECN markings of packets sent
between client and proxy in any way.

# Security Considerations {#security}

There are significant risks in allowing arbitrary clients to establish a
tunnel to arbitrary servers, as that could allow bad actors to send traffic
and have it attributed to the proxy. Proxies that support CONNECT-UDP SHOULD
restrict its use to authenticated users.

Because the CONNECT method creates a TCP connection to the target, the target
has to indicate its willingness to accept TCP connections by responding with
a TCP SYN-ACK before the proxy can send it application data. UDP doesn't have
this property, so a CONNECT-UDP proxy could send more data to an unwilling
target than a CONNECT proxy. However, in practice denial of service attacks
target open TCP ports so the TCP SYN-ACK does not offer much protection in
real scenarios. Proxies MUST NOT introspect the contents of UDP payloads as
that would lead to ossification of UDP-based protocols by proxies.


# IANA Considerations {#iana}

## HTTP Method {#iana-method}

This document will request IANA to register "CONNECT-UDP" in the
HTTP Method Registry (IETF review) maintained at
<[](https://www.iana.org/assignments/http-methods)>.

~~~
  +-------------+------+------------+---------------+
  | Method Name | Safe | Idempotent |   Reference   |
  +-------------+------+------------+---------------+
  | CONNECT-UDP |  no  |     no     | This document |
  +-------------+------+------------+---------------+
~~~


## URI Scheme Registration {#iana-scheme}

This document will request IANA to register the URI scheme "masque".

The syntax definition below uses Augmented Backus-Naur Form (ABNF)
{{!RFC5234}}.  The definitions of "host" and "port" are adopted from
{{RFC3986}}.  The syntax of a MASQUE URI is:

~~~
masque-URI = "masque:" "//" host ":" port "/"
~~~

The "host" and "port" component MUST NOT be empty, and the "port" component
MUST NOT be 0.


## Stream Chunk Type Registration {#iana-chunk-type}

This document will request IANA to create a "CONNECT-UDP Stream Chunk Type"
registry. This registry governs a 62-bit space, and follows the registration
policy for QUIC registries as defined in {{QUIC}}. In addition to the fields
required by the QUIC policy, registrations in this registry MUST include the
following fields:

Type:

: A short mnemonic for the type.

Description:

: A brief description of the type semantics, which MAY be a summary if a
specification reference is provided.

The initial contents of this registry are:

~~~
  +-------+------------+-----------------------+---------------+
  | Value |    Type    |      Description      |   Reference   |
  +-------+------------+-----------------------+---------------+
  | 0x00  | UDP_PACKET | Payload of UDP packet | This document |
  +-------+------------+-----------------------+---------------+
~~~

Each value of the format `37 * N + 23` for integer values of N (that is, 23,
60, 97, ...) are reserved; these values MUST NOT be assigned by IANA and MUST
NOT appear in the listing of assigned values.


--- back

# Acknowledgments {#acknowledgments}
{:numbered="false"}

This document is a product of the MASQUE Working Group, and the author thanks
all MASQUE enthusiasts for their contibutions. This proposal was inspired
directly or indirectly by prior work from many people. In particular, the
author would like to thank Eric Rescorla for suggesting to use an HTTP method
to proxy UDP. Thanks to Lucas Pardue for their inputs on this document.
