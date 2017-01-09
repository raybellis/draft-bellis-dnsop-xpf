---
title: EDNS X-Proxied-For
docname: draft-bellis-dnsop-xpf-01

ipr: trust200902
area: Internet
wg: DNSOP Working Group
kw: Internet-Draft
cat: std

coding: utf-8
pi:
  - toc
  - symrefs
  - sortrefs

author:
  -
    ins: R. Bellis
    name: Ray Bellis
    org: Internet Systems Consortium, Inc.
    abbrev: ISC
    street: 950 Charter Street
    city: Redwood City
    code: CA 94063
    country: USA
    phone: +1 650 423 1200
    email: ray@isc.org

normative:
  IANA-IP:
    date: 2016-11-17
    author:
      org: IANA
    title: IANA IP Version Registry
    target: http://www.iana.org/assignments/version-numbers/

--- abstract

It is becoming more commonplace to install front end proxy devices in
front of DNS servers to provide (for example) load balancing or to
perform transport layer conversions.

This document defines an option within the EDNS(0) Extension Mechanism
for DNS that allows a DNS server to receive the original client source
IP address when supplied by trusted proxies.

--- middle

# Introduction

It is becoming more commonplace to install front end proxy devices in
front of DNS servers {{!RFC1035}} to provide load balancing or to
perform transport layer conversions (e.g. to add DNS over TLS
{{?RFC7858}} to a DNS server that lacks native support).

This has the unfortunate side effect of hiding the clients' source IP
addresses from the server, making it harder to employ server-side
technologies that rely on knowing those address (e.g. ACLs, DNS Response
Rate Limiting, etc).

This document defines an option within the EDNS(0) Extension Mechanism
for DNS {{!RFC6891}} that allows a DNS server to receive the original
client source IP address when supplied by trusted proxies.

Whilst in some circumstances it would be possible to re-use the Client
Subnet EDNS Option {{?RFC7871}} to carry this information, a new option
is defined to allow both this option and the Client Subnet option to
co-exist in the same packet.

# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
"Key words for use in RFCs to Indicate Requirement Levels" {{!RFC2119}}.

The word "proxy" in this document means a network component that sits on
the inbound query path in front of a recursive or authoritative DNS
server, receiving DNS queries from clients and dispatching them to local
servers.  This is to distinguish these from a "forwarder" since that
term is usually understood to describe a network component that sits on
the oubound query path of a client.

# Description

## EDNS Option Format

The overall format of an EDNS option is shown for reference below,
per {{!RFC6891}}, followed by the option specific data:

       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    0: |                          OPTION-CODE                          |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    2: |                         OPTION-LENGTH                         |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    4: |                                                               |
       /                          OPTION-DATA                          /
       /                                                               /
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

OPTION-CODE: TBD, with mnemonic "XPF".

OPTION-LENGTH: Size (in octets) of OPTION-DATA.

OPTION-DATA: Option specific, as below:

                    +0 (MSB)                            +1 (LSB)
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    0: |     Unused    |   IP Version  |        Address Octet 0        |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    2: |        Address Octet 1        |              ...              |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |              ...             ///                              |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

Unused: Currently reserved.  These MUST be zero unless redefined in a
subsequent specification.

IP Version: The IP protocol version number used by the client, as
defined in the IANA IP Version Number Registry {{IANA-IP}}.
Implementations MUST support IPv4 (4) and IPv6 (6).

Address: The source IP address of the client.

##  Proxy Processing

Proxies MUST append this option to each request packet received before
sending it to the intended DNS server.

If this option is already present in an incoming request it MUST be
stripped from the request unless the request was received from an
upstream proxy that is itself white-listed by the receiving proxy (i.e.
if the proxies are configured in a multi-tier architecture), in which
case the original value of the option MUST be preserved.

If the proxy has to create a new OPT RR (because none was present in the
original request) it MUST strip any OPT RR subsequently seen in the
response for conformance with Section 7 of {{!RFC6891}}.

##  Server Processing

When this option is received from a white-listed client the DNS server
MUST (SHOULD?) use the address from the option contained therein in
preference to the client's source IP address for any data processing
logic that would otherwise depend on the latter.

If this option is received from a non-white-listed client the server
MUST return a REFUSED response.

If the IP version is not understood by the server it MUST return a
REFUSED response.

If the length of the client IP address contained in the OPTION-DATA is
not consistent with that expected for the given IP version then the
server MUST return a FORMERR response.

Servers MUST NOT send this option in DNS responses.

##  Secret Key Transaction Authentication for DNS (TSIG) {#tsig}

The considerations for TSIG {{!RFC2845}} from Section 4.5 of "DNS Proxy
Implementation Guidelines" {{!RFC5625}} apply here.

A TSIG-signed request MUST either:

1.  be forwarded according to RFC 5625 without addition of this option,
or

2.  be verified using a secret shared between client and proxy, updated
with this option, and then re-signed with a (potentially different)
shared secret before sending to the server.

In the case of option 1, the server might still be able to uniquely
identify and authenticate the client through its shared key, but not by
its IP address.

If option 2 is used, there is an operational trade-off to be considered
as to whether the two secrets (between client and proxy, and between
proxy and server) are actually the same secret.  A potential advantage
of three-way sharing of the secret is that if the server response
requires no modifications it may be returned directly to the client
without any further TSIG operations.

Author's note: A third alternative exists, which is to append an
additional TSIG signature to the packet based on a secret shared only
between the proxy and server.  If end-to-end TSIG validation is required
alongside TSIG validation between proxy and server, the server would
have to 1) validate that second signature, 2) strip it, and then 3)
perform further validation on the original signature.  Feedback is
sought on whether this is worth pursuing.

# Security Considerations {#security}

If the white-list of trusted proxies is implemented as a list of IP
addresses, the server administrator MUST have the ability to selectively
disable this feature for any transport where there is a possibility of
the proxy's source address being spoofed.

This does not mean to imply that use over UDP is impossible - if for
example the network architecture keeps all proxy-to-server traffic on a
dedicated network and clients have no direct access to the servers then
the proxies' source addresses can be considered unspoofable.

# Privacy Considerations

Used incorrectly, this option could expose internal network information,
however it is not intended for use on proxy / forwarder devices that sit
on the client-side of a DNS request.

This specification is only intended for use on server-side proxy devices
that are under the same administrative control as the DNS servers
themselves.  As such there is no change in the scope within which any
private information might be shared.

# IANA Considerations

IANA are directed to assign the value TBD for the XPF option
in the DNS EDNS0 Option Codes Registry.

# Acknowledgements

--- back
