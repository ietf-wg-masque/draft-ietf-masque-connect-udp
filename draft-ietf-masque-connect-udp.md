---
title: The CONNECT-UDP HTTP Method
abbrev: CONNECT-UDP
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


--- abstract

This document describes the CONNECT-UDP HTTP method. CONNECT-UDP is similar
to the HTTP CONNECT method, but it uses UDP instead of TCP.


--- middle

# Introduction {#introduction}

This document describes the CONNECT-UDP HTTP method. CONNECT-UDP is similar
to the HTTP CONNECT method (see section 4.3.6 of {{!RFC7231}}), but it
uses UDP {{!UDP=RFC0768}} instead of TCP {{!TCP=RFC0793}}.


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

The CONNECT-UDP method is defined for all versions of HTTP. UDP payloads are
sent using HTTP Datagrams {{!HTTP-DGRAM=I-D.ietf-masque-h3-datagram}}. Note
that, when the HTTP version in use does not support multiplexing streams (such
as HTTP/1.1), then any reference to "stream" in this document is meant to
represent the entire connection.


# The CONNECT-UDP Method {#connect-udp-method}

The CONNECT-UDP method requests that the recipient establish a tunnel over a
single HTTP stream to the destination origin server identified by the
request-target and, if successful, thereafter restrict its behavior to blind
forwarding of packets, in both directions, until the tunnel is closed. Tunnels
are commonly used to create an end-to-end virtual connection, which can then be
secured using QUIC {{!QUIC=RFC9000}} or another protocol running over UDP.

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
a CONNECT-UDP request with a non-empty payload is malformed.

Responses to the CONNECT-UDP method are not cacheable.


# Encoding of Proxied UDP Packets {#datagram-encoding}

UDP packets are encoded using HTTP Datagrams {{HTTP-DGRAM}}. The payload of a
UDP packet (referred to as "data octets" in {{UDP}}) is sent unmodified in the
"HTTP Datagram Payload" field of an HTTP Datagram. In order to use HTTP
Datagrams, the CONNECT-UDP client will first decide whether or not to use HTTP
Datagram Contexts and then register its context ID (or lack thereof) using the
corresponding registration capsule, see {{HTTP-DGRAM}}.

Since HTTP Datagrams require prior negotiation (for example, in HTTP/3 it is
necessary to both send and receive the H3_DATAGRAM SETTINGS Parameter), clients
MUST NOT send any HTTP Datagrams until they have established support on a given
connection.

The proxy that is creating the UDP socket to the destination responds to the
CONNECT-UDP request with a 2xx (Successful) response.

Clients MAY optimistically start sending proxied UDP packets before receiving
the response to its CONNECT-UDP request, noting however that those may not be
processed by the proxy if it responds to the CONNECT-UDP request with a
failure, or if the datagrams arrive before the CONNECT-UDP request.

Extensions to CONNECT-UDP MAY leverage the "Context Extensions" field of
registration capsules in order to negotiate different semantics or encoding for
UDP payloads.


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
CONNECT-UDP stream before closing the socket. If a proxy is notified by its
operating system that its socket is no longer usable, it MUST close the
CONNECT-UDP stream.


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


--- back

# Acknowledgments {#acknowledgments}
{:numbered="false"}

This document is a product of the MASQUE Working Group, and the author thanks
all MASQUE enthusiasts for their contibutions. This proposal was inspired
directly or indirectly by prior work from many people. In particular, the
author would like to thank Eric Rescorla for suggesting to use an HTTP method
to proxy UDP. Thanks to Lucas Pardue for their inputs on this document.
