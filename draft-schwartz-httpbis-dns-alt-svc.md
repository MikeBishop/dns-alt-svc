---
title: Finding HTTP Alternative Services via the Domain Name Service
abbrev: Alt-Svc via DNS
docname: draft-schwartz-httpbis-dns-alt-svc-latest
date: {DATE}
category: std

ipr: trust200902
area: General
workgroup: HTTP Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: B. Schwartz
    name: Ben Schwartz
    organization: Google
    email: bemasc@google.com
 -
    ins: M. Bishop
    name: Mike Bishop
    org: Akamai Technologies
    email: mbishop@evequefou.be

normative:

informative:

--- abstract
The HTTP Alternative Services (Alt-Svc) mechanism allows an
HTTP origin to be served from multiple network endpoints, and over
multiple protocols.  However, the client must first contact the
origin server, in order to learn of the alternative services.  This
draft proposes a straightforward mapping of Alt-Svc into
DNS, allowing clients to learn of these services before their first
contact with the origin.  This arrangement offers potential benefits to
both performance and privacy.

--- middle

# Introduction

The HTTP Alternative Services standard {{!AltSvc=RFC7838}} defines

* an extensible data model for describing alternative network endpoints
  that are authoritative for an origin
* the "Alt-Svc Field Value", a text format for representing this
  information
* standards for sending information in this format from a server to a
  client over HTTP/1.1 and HTTP/2.

Together, these components provide a toolkit that has proven useful and
effective for informing a client of alternative services for an origin.
However, making use of an alternative service requires contacting the
origin server first.  This creates an obvious performance cost: users
wait for a full HTTP connection initiation (multiple roundtrips) before
learning of an alternative service that is preferred by the origin.  The
first connection also publicly reveals the user's intended destination
to all entities along the network path.

This draft proposes a straightforward mechanism to distribute the
Alt-Svc Field Value, in its standard text format, through
the DNS.  If a client receives this information during DNS resolution,
it can skip the initial connection and proceed directly to an
alternative service.

## Terminology
For consistency with {{!AltSvc}}, we adopt the following definitions
* An "origin" is an information source as in {{!RFC6454}}.
* The "origin server" is the server that the client would reach when
  accessing the origin in the absence of Alt-Svc.
* An "alternative service" is a different server that can serve the
  origin.

# The ALTSVC record type

The ALTSVC DNS resource record (RR) type (RRTYPE ???) is used to
associate an Alternative Service Field Value with an origin.
Abstractly, the origin consists of a scheme (typically "https"), a host
name, and a port (typically "443").

In the case of the ALTSVC RR, the origin is represented by prefixing the
scheme and port with "_", then concatenating them with the host,
resulting in a domain name like "_https._443.www.example.com.".

The RDATA portion of an ALTSVC resource record contains an Alt-Svc
Field Value, exactly as defined in Section 4 of {{!AltSvc}}.

For example, if the operator of https://www.example.com
intends to include an HTTP response header like

 Alt-Svc: h2=":8000"; ma=60

They would also publish an ALTSVC DNS record like

 _https._443.www.example.com. 60S IN ALTSVC "h2=\\":8000\\""

This data type can be represented as an Unknown RR as described in
{{!RFC3597}}:

 _https._443.www.example.com. 60S IN TYPE??? \\# 10 68323D223A3830303022

This construction is intended to be extensible in two ways.  First,
any extensions that are made to the Alt-Svc format for transmission over
HTTPS are also applicable here, unless expressly mentioned otherwise.
Second, including the scheme in the DNS name allows for ALTSVC to serve
schemes other than HTTPS, such as HTTP with Opportunistic Security
{{!RFC8164}}) and any future schemes for which Alt-Svc may be defined.

## Comparison with alternatives

The ALTSVC record type closely resembles some existing record types.

### Differences from the SRV RRTYPE

An SRV record can perform a similar function to the ALTSVC record,
informing a client to look in a different location for a service.
However, there are several differences:

* SRV records are typically mandatory, whereas clients will always
  continue to function correctly without making use of Alt-Svc.
* SRV records cannot instruct the client to switch or upgrade
  protocols, whereas Alt-Svc can signal such an upgrade (e.g. to
  HTTP/2).
* SRV records are not extensible, whereas Alt-Svc can be extended with
  new parameters.  For example, this is what allows the privacy
  improvements related to SNI selection in {{!AltSvcSNI}}.
* Using SRV records would not allow a client to skip processing of the
  Alt-Svc information in a subsequent connection, so it does not confer
  a performance advantage.

### Differences from the TXT RRTYPE

The ALTSVC record uses an identical format to a TXT record, and could
be implemented as such.  However, we define a new record type for
clarity, and to respect the use of TXT for human-readable notes as
recommended in {{!RFC5507}}.

# Differences from Alt-Svc as transmitted over HTTP

Publishing an ALTSVC record in DNS is intended to be equivalent to
transmitting this field value over HTTP, and receiving an ALTSVC record
is intended to be equivalent to receiving this field value over HTTP.
However, there are some small differences in the intended client and
server behavior.

## Omitting Max Age

When publishing an ALTSVC record in DNS, server operators MUST omit the
"ma" parameter, which encodes the "max age" (i.e. expiration time) of
an Alt-Svc Field Value.  Instead, server operators SHOULD encode the
expiration time in the DNS TTL, and MUST NOT set a TTL longer than the
intended "max age".

Server operators MAY publish multiple ALTSVC records as an RRSET, with
semantics equivalent to other mechanisms of providing multiple Alt-Svc
values to the client.  When publishing an RRSET with multiple ALTSVC
records, the server operator MUST set the overall TTL to the minimum
of the "max age" values (following Section 5.2 of {{!RFC2181}}).

When receiving an ALTSVC record, clients MAY synthesize a new "ma"
parameter from the DNS
TTL, in order to interoperate with Alt-Svc processing subsystems.

## Interaction with other standards

The purpose of this standard is to reduce connection latency and
improve user privacy.  Server operators implementing this standard
SHOULD also implement TLS 1.3 {{!I-D.ietf-tls-tls13}} and OCSP Stapling
{{!RFC6066}}, both of which confer substantial performance and privacy
benefits when used in combination with ALTSVC records.

To realize the greatest privacy benefits, this proposal is intended for
use with a privacy-preserving DNS transport (like DNS over TLS
{{!RFC7858}} or DNS over HTTPS {{!DOH=I-D.ietf-doh-dns-over-https}}),
and with the "SNI" Alt-Svc Parameter
{{!AltSvcSNI=I-D.bishop-httpbis-sni-altsvc}}.  However, performance
improvements, and some modest privacy improvements, are possible without
the use of those standards.

## Granularity and lifetime control

Sending Alt-Svc over HTTP allows the server to tailor the Alt-Svc
Field Value specifically to the client.  When using an ALTSVC DNS
record, groups of clients will necessarily receive the same Alt-Svc
Field Value.  Therefore, this standard is not suitable for servers that
require single-client granularity in Alt-Svc.  Server operators that
want to serve different Alt-Svc Field Values to different geographic
or network regions SHOULD configure their authoritative DNS server to
respect the EDNS0 Client Subnet extension {{!RFC7871}}.

Some DNS caching systems incorrectly extend the lifetime of DNS
records beyond the stated TTL.  Server operators MUST NOT rely on
ALTSVC records expiring on time, and MAY shorten the TTL to compensate.

# Client behaviors

## Cache interaction

If the client has an Alt-Svc cache, and a usable Alt-Svc value is
present in that cache, then the client SHOULD NOT issue an ALTSVC DNS
query.  Instead, the client SHOULD proceed with alternative service
connection as usual.

If the client has a cached Alt-Svc entry that is expiring, the
client MAY perform an ALTSVC query to refresh the entry.

## Optimizing for performance

Clients that are optimizing for performance (i.e. minimum connection
setup time) SHOULD implement the following connection sequence:

1. Issue address (AAAA and/or A) queries, immediately followed by the
   ALTSVC query.
1. If an ALTSVC response is received first, proceed with alternative
   service connection and ignore the address responses if they are no
   longer relevant.
1. Otherwise, initiate connection to the origin server.
1. As soon as an Alt-Svc field value is received, through the DNS or
   over HTTP, proceed with alternative service connection.  Do not
   abort this connection if an Alt-Svc field value is received from the
   other source later.

If the ALTSVC and address queries return approximately simultaneously,
this process typically saves three roundtrips on a fresh connection
that uses Alt-Svc: one each for TCP, TLS 1.3, and HTTP.  (On subsequent
connections, the Alt-Svc information is expected to be cached, so this
procedure does not apply.)

If a client can cache Alt-Svc entries that were received over both HTTP
and DNS, the client MAY prefer entries that were received over HTTP.
These records may be more narrowly targeted for the specific client.

As an additional optimization, when choosing among multiple Alt-Svc
values, clients MAY prefer those that will not require an address
query, either because the corresponding address record is
already in cache or because the host is an IP address.

Note that this procedure does not rely on recursive resolvers handling
the ALTSVC record type correctly.  If ALTSVC queries receive spurious
NXDOMAIN responses, or even no response at all, connections will proceed
as usual without any delay.

## Optimizing for privacy

Clients that are optimizing for privacy SHOULD implement {{!AltSvcSNI}}
and DNS over a secure transport (e.g. {{!RFC7858}} or {{!DOH}}).
Use of a secure transport is important not only for privacy protection,
but also to ensure that queries for the new ALTSVC RRTYPE are handled
correctly.  Additionally, these clients SHOULD implement the following
connection sequence:

1. Issue the ALTSVC DNS query first, immediately followed by the
   address queries.
1. Wait for the ALTSVC record response.
1. If the response is nonempty, proceed with alternative service
   connection and ignore the address query responses.
1. Otherwise, wait for the address queries and connect as usual.

Note that this process is also expected to be faster than Alt-Svc over
HTTP in the case of HTTP Opportunistic Upgrade Probing (Section 2 of
{{!RFC8164}}).

# Security Considerations

Alt-Svc Field Values are intended for distribution over untrusted
channels, and clients are REQUIRED to verify that the alternative
service is authoritative for the origin (Section 2.1 of {{!AltSvc}}).
Therefore, DNSSEC signing and validation are NOT REQUIRED for publishing
and using ALTSVC records.

# IANA Considerations

This draft requires assignment of a new DNS RRTYPE value.
--- back

