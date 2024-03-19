---
title: "DNS Extensions for Proxying IP in HTTP"
abbrev: "CONNECT-IP DNS"
category: std
docname: draft-schinazi-masque-connect-ip-dns-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
wg: MASQUE
venue:
  group: "Multiplexed Application Substrate over QUIC Encryption"
  type: "Working Group"
  mail: "masque@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/masque/"
  github: "DavidSchinazi/draft-schinazi-masque-connect-ip-dns"
  latest: "https://DavidSchinazi.github.io/draft-schinazi-masque-connect-ip-dns/draft-schinazi-masque-connect-ip-dns.html"
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
  - dns
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
informative:

--- abstract

Proxying IP in HTTP allows building a VPN through HTTP load balancers. However,
at the time of writing, that mechanism doesn't offer a mechanism for
communicating DNS information inline. In contrast, most existing VPN protocols
provide a mechanism to exchange DNS configuration information. This document
describes an extension that exchanges this information using HTTP capsules.

--- middle

# Introduction

Proxying IP in HTTP ({{!CONNECT-IP=RFC9484}}) allows building a VPN through
HTTP load balancers. However, at the time of writing, that mechanism doesn't
offer a mechanism for communicating DNS information inline. In contrast, most
existing VPN protocols provide a mechanism to exchange DNS configuration
information (e.g., {{?IKEv2=RFC7296}}). This document describes an extension
that exchanges this information using HTTP capsules ({{!HTTP-DGRAM=RFC9297}}).

## Conventions and Definitions

{::boilerplate bcp14-tagged}

# Mechanism

Similar to how Proxying IP in HTTP exchanges IP addresses ({{Section 4.7 of
CONNECT-IP}}), this mechanism leverages capsules to request configuration and
to assign it. Similarly, this mechanism is bidirectional: either endpoint can
request DNS configuration by sending a DNS_REQUEST capsule, and either endpoint
can send DNS configuration in a DNS_ASSIGN capsule. These capsules follow the
format defined below.

~~~
Nameserver Address {
  IP Version (8),
  IP Address (32..128),
}
~~~
{: #ns-addr-format title="Nameserver Address Format"}

Each Nameserver Address contains the following fields:

IP Version:

: IP Version of this nameserver address, encoded as an unsigned 8-bit integer.
It MUST be either 4 or 6.

IP Address:

: DNS nameserver IP address. If the IP Version field has value 4, the IP
Address field SHALL have a length of 32 bits. If the IP Version field has value
6, the IP Address field SHALL have a length of 128 bits.


~~~
Domain {
  Domain Length (i),
  Domain Name (..),
}
~~~
{: #domain-format title="Internal Domain Format"}

Each Domain contains the following fields:

Domain Length:

: Length of the following Domain field, encoded as a variable-length integer.

Domain Name:

: Fully Qualified Domain Name in DNS presentation format and using an
Internationalized Domain Names for Applications (IDNA) A-label
({{!IDNA=RFC5890}}).

~~~
DNS Configuration {
  Request ID (i),
  Nameserver Address Count (i),
  Nameserver Address (..) ...,
  Internal Domain Count (i),
  Internal Domain (..) ...,
  Search Domain Count (i),
  Search Domain (..) ...,
}
~~~
{: #assigned-addr-format title="Assigned Address Format"}

Each DNS Configuration contains the following fields:

Request ID:

: Request identifier, encoded as a variable-length integer. If this DNS
Configuration is part of a request, then this contains a unique request
identifier. If this DNS configuration is part of an assignment that is in
response to a DNS configuration request then this field SHALL contain the value
of the corresponding field in the request. If this DNS configuration is part of
an unsolicited assignment, this field SHALL be zero.

Nameserver Address Count:

: The number of Nameserver Address structures following this field. Encoded as
a variable-length integer.

Nameserver Address:

: A series of Nameserver Address structures representing DNS name servers.

Internal Domain Count:

: The number of Domain structures following this field. Encoded as a
variable-length integer.

Internal Domain:

: A series of Domain structures representing internal DNS names.

Search Domain Count:

: The number of Domain structures following this field. Encoded as a
variable-length integer.

Search Domain:

: A series of Domain structures representing search domains.
{:newline="true" spacing="normal"}

## DNS_REQUEST Capsule {#dns-req}

The DNS_REQUEST capsule (see {{iana}} for the value of the capsule type) allows
an endpoint to request DNS configuration from its peer. The capsule allows the
endpoint to optionally indicate a preference for which DNS configuration it
would get assigned. The sender can indicate that it has no preference by not
sending any addresses or names in its request DNS Configuration.

~~~
DNS_REQUEST Capsule {
  Type (i) = DNS_REQUEST,
  Length (i),
  DNS Configuration (..),
}
~~~
{: #dns-req-format title="DNS_REQUEST Capsule Format"}

When sending a DNS_REQUEST capsule, the sender MUST generate and send a new
non-zero request ID that was not previously used on this IP Proxying stream.

An endpoint that receives a DNS_REQUEST capsule SHALL reply by sending a
DNS_ASSIGN capsule with the corresponding request ID. That DNS_ASSIGN capsule
MAY be empty, that indicates that its sender has no DNS configuration to share
with its peer.

## DNS_ASSIGN Capsule {#dns-assign}

The DNS_ASSIGN capsule (see {{iana}} for the value of the capsule type) allows
an endpoint to send DNS configuration to its peer.

~~~
DNS_ASSIGN Capsule {
  Type (i) = DNS_ASSIGN,
  Length (i),
  DNS Configuration (..),
}
~~~
{: #dns-assign-format title="DNS_ASSIGN Capsule Format"}

When sending a DNS_ASSIGN capsule in response to a received DNS_REQUEST
capsule, the Request ID field in the DNS_ASSIGN capsule SHALL be set to the
value in the received DNS_REQUEST capsule. Otherwise the request ID MUST be set
to zero.

# Handling

Note that internal domains include subdomains. In other words, if the DNS
configuration contains a domain, that indicates that the corresponding domain
and all of its subdomains can be resolved by the nameservers exchanged in the
same DNS configuration.

As with other IP Proxying capsules, the receiver can decide to whether to use
or ignore the configuration information. For example, in the consumer VPN
scenario, clients will trust the server and apply received DNS configuration,
whereas servers will ignore any DNS configuration sent by the client.

# Security Considerations

Acting on received DNS_ASSIGN capsules can have significant impact on endpoint
security. Endpoints MUST ignore DNS_ASSIGN capsules unless it has reason to
trust its peer and is expecting DNS configuration from it.

The requirement for an endpoint to always send DNS_ASSIGN capsules in response
to DNS_REQUEST capsules could lead it to buffer unbounded amounts of memory if
the underlying stream is blocked by flow or congestion control. Implementations
MUST place an upper bound on that buffering and abort the stream if that limit
is reached.

# IANA Considerations {#iana}

This document, if approved, will request IANA add the following values to the
"HTTP Capsule Types" registry maintained at
<[](https://www.iana.org/assignments/masque)>.

|   Value    | Capsule Type |
|:-----------|:-------------|
| 0x2B40144C |  DNS_ASSIGN  |
| 0x2B40144D |  DNS_REQUEST |
{: #iana-capsules-table title="New Capsules"}

Note that, if this document is approved, the values defined above will be
replaced by smaller ones before publication.

All of these new entries use the following values for these fields:

Status:

: provisional (permanent if this document is approved)

Reference:

: This document

Change Controller:

: IETF

Contact:

: masque@ietf.org

Notes:

: None
{: spacing="compact" newline="false"}

--- back

# Acknowledgments
{:numbered="false"}

The mechanism is this document was inspired by {{IKEv2}} and
{{?IKEv2-DNS=RFC8598}}.
