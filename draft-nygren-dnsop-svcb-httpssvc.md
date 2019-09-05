---
title: Service binding and parameter specification via the DNS (DNS SVCB and HTTPSSVC)
abbrev: SVCB and HTTPSSVC RRs for DNS
docname: draft-nygren-dnsop-svcb-httpssvc-latest
date: {DATE}
category: std

ipr: trust200902
area: General
workgroup: DNSOP Working Group
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
 -
    ins: E. Nygren
    name: Erik Nygren
    org: Akamai Technologies
    email: erik+ietf@nygren.org

normative:

informative:

--- abstract

This document specifies the "SVCB" and "HTTPSSVC" DNS resource record types to
facilitate the lookup of information needed to make connections for
origin resources, such as for HTTPS URLs.  SVCB records
allow an origin to be served from multiple network
locations, each with associated parameters (such as transport protocol
configuration and keying material for encrypting TLS SNI).  They also
aliasing of apex domains, which not possible with CNAME.
The HTTPSSVC DNS RR is a variation of SVCB for HTTPS and HTTP origins.
By providing more information to the client before it attempts to
establish a connection, SVCB offers potential benefits to
both performance and privacy.

TO BE REMOVED: This proposal is inspired by and based on recent DNS
usage proposals such as ALTSVC, ANAME, and ESNIKEYS (as well as long
standing desires to have SRV or a functional equivalent implemented
for HTTP).  These proposals each provide an important function but are
potentially incompatible with each other, such as when an origin is
load-balanced across multiple hosting providers (multi-CDN).
Furthermore, these each add potential cases for adding additional
record lookups in-addition to AAAA/A lookups. This design attempts to
provide a unified framework that encompasses the key functionality of
these proposals, as well as providing some extensibility for
addressing similar future challenges.

TO BE REMOVED: The specific name for this RRtype is an open
topic for discussion.  "SVCB" and "HTTPSSVC" are meant as placeholders
as they are easy to replace.  Other names might include "B",
"SRV2", "SVCHTTPS", "HTTPS", and "ALTSVC".


--- middle

# Introduction

The SVCB and HTTPSSVC RRs provide clients with complete instructions
for optimal access to an origin.  This information enables improved
performance and privacy by avoiding transient connections to a suboptimal
default server, negotiating a preferred protocol, and providing relevant
public keys.

For example, when clients need to make a connection to fetch resources
associated with an HTTPS URI, they currently resolve only A and/or AAAA
records for the origin hostname.  This is adequate
when clients default to TCP port 443, do not support Encrypted SNI
{{!ESNI=I-D.ietf-tls-esni}}, and where the origin service operator
does not have a desire to put an CNAME at a zone apex (such as for
"https://example.com").  Handling situations beyond this within the
DNS requires learning additional information, and it is highly
desirable to minimize the number of round-trip and lookups required to
learn this additional information.

The SVCB and HTTPSSVC RRs also help in scenarios where an the operator of an origin
resource in a domain wishes to delegate operational control to one or
more service operators who manage other domains, such as for
delegating the origin resource "https://example.com" to a service
operator endpoint at "svc.example.net".  While this case can sometimes
be handled by a CNAME, that does not cover all use-cases.  It also is
inadequate when the service operator needs to provide a bound
collection of consistent configuration parameters through the DNS
(such as network location, protocol, and keying information).

This document first describes the SVCB RR as a general-purpose resource
record that can be applicable to a wide range of services.
As HTTPS is a primary use-case and with special requirements,
the HTTPSSVC RR is also defined within this document as a special case
of SVCB.

SVCB is intended to support a wide range of
services and protocols, many of which may be able to use it directly.
Services wishing to avoid the need for an {{?Attrleaf}} label with
SVCB may follow the pattern of HTTPSSVC and assign their own
SVCB-compatible RRTypes.

All behaviors described as applying to the SVCB RR also apply
to the HTTPSSVC RR unless explicitly stated otherwise.
{{https}} describes additional behaviors
specific to the HTTPSSVC record.  Apart from {{https}}
and introductory examples, much of this document refers only to the SVCB RR,
but those references should be taken to apply to SVCB, HTTPSSVC,
and any future SVCB-compatible RRTypes.

The SVCB RR has two forms: 1) the "Alias Form" simply delegates operational
control for a resource; 2) the "Service Form" binds together
configuration information for a service endpoint.
The Service Form provides additional key=value parameters
within each RRData set.

TO BE REMOVED: If we use this for providing configuration for DNS
authorities, it is likely we'd specify a distinct "NS2" RRType that is
an instantiation of SVCB for authoritative nameserver delegation and
parameter specification, similar to HTTPSSVC.

TO BE REMOVED: Another open question is whether SVCB records
should be self-descriptive and include the service name
(eg, "https") in the RRData section to avoid ambiguity.
Perhaps this could be included as a svc="baz" parameter
for protocols that are not the default for the RRType?
Current inclination is to not do so.

## Introductory Example

As an introductory example for an HTTPS origin resource, a set of
example HTTPSSVC and associated A+AAAA records might be:

    www.example.com.  7200  IN CNAME    svc.example.net.
    ; AliasForm
    example.com.      7200  IN HTTPSSVC 0 svc.example.net.
    ; ServiceForm
    svc.example.net.  7200  IN HTTPSSVC 2 svc3.example.net. ( alpn=h3
                                        port=8003 esnikeys="..." )
    svc.example.net.  7200  IN HTTPSSVC 3 svc2.example.net. ( alpn=h2
                                        port=8002 esnikeys="..." )
    svc2.example.net. 300   IN A        192.0.2.2
    svc2.example.net. 300   IN AAAA     2001:db8::2
    svc3.example.net. 300   IN A        192.0.2.3
    svc3.example.net. 300   IN AAAA     2001:db8::3

In the preceding example, both of the "example.com" and
"www.example.com" origin names are aliased to use alternative service
endpoints offered as "svc.example.net" (with "www.example.com"
continuing to use a CNAME alias).  HTTP/2 is available on a cluster of
machines located at svc2.example.net with TCP port 8002 and HTTP/3 is
available on a cluster of machines located at svc3.example.net with
UDP port 8003.  ESNI keys specified which allows the SNI values of
"example.com" and "www.example.com" to be encrypted in the handshake
with these alternative service endpoints.  One of these keys is
provided as a literal value while the other is provided by an external
DNS name ("esni-svc2.example.net").  When connecting, clients will
continue to treat the authoritative origins as "https://example.com"
and "https://www.example.com", respectively.

For services other than HTTPS (as well as for HTTPS origins
with non-default ports), the SVCB RR and an {{?Attrleaf}} label will be used.
For example, to reach an example resource of
"baz://api.example.com:8765", the following Alias Form
SVCB record would be used to delegate to "svc4-baz.example.net."
which in-turn could return AAAA/A records and/or SVCB
records in ServiceForm.

    _8765._baz.api.example.com. 2H IN SVCB 0 svc4-baz.example.net.


## Goals of the SVCB RR

The goal of the SVCB RR is to allow clients to resolve a single
additional DNS RR in a way that:

* Provides service endpoints authoritative for the service,
  along with parameters associated with each of these endpoints.
* Does not assume that all alternative service endpoints have the same parameters
  or capabilities, or are even
  operated by the same entity.  This is important as DNS does not
  provide any way to tie together multiple RRs for the same name.
  For example, if www.example.com is a CNAME alias that switches
  between one of three CDNs or hosting enviroments, successive queries
  for that name may return records that correspond to different environments.
* Enables the functional equivalent of a CNAME at a zone apex (such as
  "example.com"), and generally
  enables delegation of operational authority for an origin within the
  DNS to an alternate name.

Additional goals specific to HTTPSSVC and the HTTPS use-case include:

* Connect directly to {{!HTTP3=I-D.draft-ietf-quic-http-20}} (QUIC transport)
  alternative service endpoints
* Obtain the {{!ESNI}} keys associated with an alternative service endpoint
* Support alternate TCP and UDP ports for the alternative
  service endpoint, beyond the default for the service
* Address a set of long-standing issues due to HTTP(S) clients not
  implementing support for SRV records, as well as due to a limitation
  that a DNS name can not have both a CNAME record as well as NS RRs
  (as is the case for zone apex names)
* Provide an HSTS-like indication signalling
  for the duration of the DNS RR TTL that the HTTPS scheme should
  be used instead of HTTP (see {{hsts}}).

## Overview of the SVCB RR

This subsection briefly describes the SVCB RR in
a non-normative manner.  (As mentioned above, this all
applies equally to the HTTPSSVC RR which shares
the same encoding, format, and high-level semantics.)

The SVCB RR has two forms: AliasForm and ServiceForm.
SVCB RR entries with three primary non-empty fields are in ServiceForm.
When the third field is empty, this indicates that the SVCB RR
is in AliasForm.

1. SvcFieldPriority: The priority of this record (relative to others,
   with lower values preferred).  Applicable for the ServiceForm,
   and otherwise has value "0".  (Described in {{pri}}.)
2. SvcDomainName: The domain name of either the alias target (for
   AliasForm) or the alternative service endpoint (for ServiceForm).
3. SvcFieldValue: A list of key=value pairs
   describing the alternative service endpoint for the domain name specified in
   SvcDomainName (only for ServiceForm and otherwise empty).
   Described in {{svcfieldvalue}}.

Cooperating DNS recursive resolvers will perform subsequent record
resolution (for SVCB, A, and AAAA records) and return them in the
Additional Section of the response.  Clients must either use responses
included in the additional section returned by the recursive resolver
or perform necessary SVCB, A, and AAAA record resolutions.  DNS
authoritative servers may attach in-bailiwick SVCB, A, AAAA, and CNAME
records in the Additional Section to responses for an SVCB query.

When in the ServiceForm, the SvcFieldValue of the SVCB RR
provides an extensible data model for describing network
endpoints that are authoritative for the origin, along with
parameters associated with each of these endpoints.

For the HTTPS use-case with the HTTPSSVC RR, there is also direct mapping
from the SvcDomainName and SvcFieldValue into
HTTP Alternative Services (Alt-Svc) entries {{!AltSvc=RFC7838}}.
Encoding this information here enables many of the benefits of Alt-Svc,
without waiting for a full HTTP connection initiation (multiple roundtrips)
before learning of the preferred alternative,
and without publicly revealing the user's
intended destination to all entities along the network path.



## Parameter for ESNI

This document also defines a parameter for Encrypted SNI {{!ESNI}}
keys, both as a general SVCB parameter and also as a corresponding
Alt-Svc parameter. See {{esnikeys}}.

## Terminology

For consistency with {{!AltSvc}}, we adopt the following definitions:

* An "origin" is an information source as in {{!RFC6454}}.
  For services other than HTTPS, the exact definition will
  need to be provided by the document mapping that service
  onto the SVCB RR.
* The "origin server" is the server that the client would reach when
  accessing the origin in the absence of the SVCB record
  or an HTTPS Alt-Svc.
* An "alternative service" is a different server that can serve the
  origin over a specified protocol.

For example within HTTPS, the origin consists of a scheme (typically
"https"), a host name, and a port (typically "443").

Additional DNS terminology intends to be consistent
with {{?DNSTerm=RFC8499}}.

SVCB is a contraction of "service binding".  HTTPSSVC is a contraction of
"HTTPS service".  SVCB, HTTPSSVC, and future RRTypes that share SVCB's
format and registry are collectively known as SVCB-compatible RRTypes.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED",
"MAY", and "OPTIONAL" in this document are to be interpreted as
described in BCP 14 {{!RFC2119}} {{!RFC8174}} when, and only when, they
appear in all capitals, as shown here.



# The SVCB record type

The SVCB DNS resource record (RR) type (RRTYPE ???)
is used to locate endpoints that can service an origin.
There is special handling for the case of "https" origins.
The presentation format of the record is:

    RRName TTL Class SVCB SvcFieldPriority SvcDomainName SvcFieldValue

SvcFieldPriority is a number in the range 0-65535,
SvcDomainName is a domain name,
and SvcFieldValue is a set of key=value pairs present for the ServiceForm.
The SvcFieldValue is but empty and not included for the AliasForm.

The algorithm for resolving SVCB records and associated
address records is specified in {{client-behavior}}.

## Parameter specification via ServiceFieldValue {#svcfieldvalue}

In ServiceForm, the SvcFieldValue contains key=value pairs.
Keys are IANA-registered SvcParamKeys ({{svcparamregistry}})
with both a case-insensitive string representation and
a numeric representation in the range 0-65535.
They should only contain characters from the ranges
"a"-"z", "0"-"9", and "-".  In ABNF {{!RFC5234}},

    key = ALPHA / DIGIT / "-"

Values are in a format specific to the SvcParamKey.
Their definition should specify both their presentation format
and wire encoding (e.g., domain names, binary data, or numeric values).

The SVCB format preserves the order of values and can encode multiple
values for the same parameter.  However, a parameter should not appear
more than once unless its specification explicitly allows this.

### Presentation format

The presentation format for SvcFieldValue is a whitespace-separated
list of the key=value pairs.  Each pair is presented in the following form:

    basic-visible = %x21 / %x23-5B / %x5D-7E ; VCHAR minus DQUOTE and "\"
    escaped       = "\" (VCHAR / WSP)
    contiguous    = *(basic-visible / escaped-character)
    quoted-string = DQUOTE *(contiguous / WSP) DQUOTE
    value         = quoted-string / contiguous
    pair          = key "=" value

The value format is intended to match the definition of &lt;character-string&gt;
in {{!RFC1035}} Section 5.1.  (Unlike &lt;character-string&gt;, the length
of a value is not limited to 255 characters.)

Unrecognized keys are represented in presentation
format as "keyNNNNN" where NNNNN is the numeric
value of the key type.  In presentation format, values of unrecognized keys
should be represented in wire format, using decimal escape codes
(e.g. \255) when necessary.


## SVCB RDATA Wire Format

The RDATA for the SVCB RR consists of:

* a 2 octet field for SvcFieldPriority as an integer in network
  byte order.  For AliasForm, SvcFieldPriority MUST be 0.
* the uncompressed SvcDomainName, represented as
  a sequence of length-prefixed labels as in Section 3.1 of {{!RFC1035}}.
* the SvcFieldValue byte string, consuming the remainder of the record
  (so smaller than 65535 octets and constrained by the RRDATA
  and DNS message sizes).

AliasForm is defined by SvcFieldValue being empty and not present.

When SvcFieldValue is non-empty (ServiceForm), it contains a list of
SvcParamKey=SvcParamValue pairs with length-prefixes for the SvcParamValues,
each of which contains:

* a 2 octet field containing the SvcParamKey as an
  integer in network byte order.
* a 2 octet field containing the length of the SvcParamValue
  as an integer between 0 and 65535 in network byte order
  (but constrained by the RRDATA and DNS message sizes).
* an octet string of the length defined by the previous field.

If the parser reaches the end of the RDATA while parsing a SvcFieldValue,
the RR is invalid and MUST be discarded.

TODO: decide if we want special handling for any SvcParamKey ranges?
For example: range for greasing; experimental range;
range-of-manditory-to-use-the-RR vs range of
ignore-just-param-if-unknown.


## RRNames {#rrnames}

When querying the SVCB RR, an origin is typically translated into a QName
by prefixing the port and scheme with "_", then concatenating them with the
host name, resulting in a domain name like "_8004._examplescheme.api.example.com.".

Protocol mappings for SVCB MAY remove the port or replace it with other
protocol-specific information, but MUST retain the scheme in the QName.
RRTypes other than SVCB can define additional behavior for translating
origins to RRNames.  See {{httpsrrnames}} for the HTTPSSVC behavior.

When a prior CNAME or SVCB record has aliased to
an SVCB record, each RR shall be returned under its own RRNAME.

Note that none of these forms alter the origin or authority for validation
purposes.
For example, clients MUST continue to validate TLS certificate
hostnames based on the origin host.

As an example:

    _8443._foo.api.example.com. 2H IN SVCB 0 svc4.example.net.
    svc4.example.net.  7200  IN SVCB 3 ( svc4.example.net. alpn="bar"
                                       port="8004" esnikeys="..." )

would indicate that "foo://api.example.com:8443" is aliased
to use ALPN protocol "bar" service endpoints offered as "svc4.example.net"
on port 8004.



## SvcRecordType

The SvcRecordType is implicit based on the presence of SvcFieldValue,
and defines the form of the SVCB RR.
When SvcFieldValue is empty, the SVCB SvcRecordType is defined to be
in AliasForm.  Otherwise, the SVCB SvcRecordType is defined to be
in ServiceForm.

Within an SVCB RRSet,
all RRs should have the same SvcRecordType.
If an RRSet contains a record in AliasForm, the client MUST ignore
any records in the set with ServiceForm.



## SVCB records: AliasForm

When SvcRecordType is AliasForm, the SVCB record is to be treated
similar to a CNAME alias pointing to the domain name specified in
SvcDomainName.  SVCB RRSets SHOULD only have a single resource
record in this form.  If multiple are present, clients or recursive
resolvers SHOULD pick one at random.

The common use-case for this form of the SVCB record is as an
alternative to CNAMEs at the zone apex where they are not allowed.
For example, if an operator of https://example.com wanted to
point HTTPS requests to a service operating at svc.example.net,
they would publish a record such as:

    example.com. 3600 IN SVCB 0 svc.example.net.

The SvcDomainName MUST point to a domain name that contains
another SVCB record or address (AAAA and/or A) records.

Note that the RRName and the SvcDomainName MAY themselves be CNAMEs.
Clients and recursive resolvers MUST follow CNAMEs as normal.

Due to the risk of loops, clients and recursive resolvers MUST
implement loop detection.  Chains of consecutive SVCB and CNAME
records SHOULD be limited to (8?) prior to reaching terminal address
records.

The SvcFieldValue in this form MUST be empty (and not present
in presentation format).

As legacy clients will not know to use this record, service
operators will likely need to retain fallback AAAA and A records
alongside this SVCB record, although in a common case
the target of the SVCB record might offer better performance, and
therefore would be preferable for clients implementing this specification
to use.

Note that SVCB AliasForm RRs do not alias to RRTypes other than
address records (AAAA and A), CNAMEs, and ServiceForm SVCB records.
For example, an AliasForm SVCB record does not alias to an HTTPSSVC record, or
vice-versa.

## SVCB records: ServiceForm

When SvcRecordType is the ServiceForm, the combination of
SvcDomainName and SvcFieldValue parameters within each resource record
associates an alternative service and associated parameters with an origin.

{{map2altsvc}} defines a direct mapping between Alt-Svc ({{!AltSvc}}) values
and the SVCB ServiceForm.  Protocols using SVCB may use this Alt-Svc
mapping or specify their own semantics.  Each protocol mapping must specify
client behavior when a SvcFieldValue contains unrecognized parameters.

### Special handling of "." for SvcDomainName in ServiceForm {#svcdomainnamedot}

For ServiceForm SVCB RRs, if SvcDomainName has the value ".", then the
RRNAME of this record MUST be used as the effective
SvcDomainName.  (The SvcDomainName of an SVCB RR in AliasForm MUST NOT have
this value.)

### SvcFieldPriority  {#svcfieldpri}

As RRs within an RRSET are explicitly unordered collections, the
SvcFieldPriority value serves to indicate priority.
SVCB RRs with a smaller SvcFieldPriority value SHOULD be given
preference over RRs with a larger SvcFieldPriority value.

When receiving an RRSET containing multiple SVCB records with the
same SvcFieldPriority value, clients SHOULD apply a random shuffle within a
priority level to the records before using them, to ensure uniform
load-balancing.


# Client behavior {#client-behavior}

An SVCB-aware client resolves an origin by attempting to determine
the preferred SvcFieldValue and IP addresses and SvcFieldValue for its service, using the
following procedure:

1. Issue parallel AAAA/A and SVCB queries for the name HOST.
   The answers for these may or may not include CNAME pointers
   before reaching one or more of these records.

2. If an SVCB record of AliasForm SvcRecordType is returned for HOST,
   clients MUST loop back to step 1 replacing HOST with SvcDomainName,
   subject to loop detection heuristics.

3. If one or more SVCB records of ServiceForm SvcRecordType are returned
   for HOST, clients should select the highest-priority option with
   acceptable parameters, and resolve AAAA and/or A records for its
   SvcDomainName if they are not already available.  These are the
   preferred SvcFieldValue and IP addresses.  If connection fails, the
   client MAY try to connect using values from a lower-priority record.

4. If an SVCB record for HOST does not exist, the received AAAA and/or A
   records are the preferred IP addresses, and there is no preferred
   SvcFieldValue.

This procedure does not rely on any recursive or authoritative server to
comply with this specification or have any awareness of SVCB.

When selecting between AAAA and A records to use, clients may
use an approach such as {{!HappyEyeballsV2=RFC8305}}

Some important optimizations are discussed in {{optimizations}}
to avoid additional latency in comparison to ordinary AAAA/A lookups.

# DNS Server Behavior {#server-behavior}

Authoritative DNS servers SHOULD return A, AAAA, and SVCB records (as
well as any relevant CNAME records) in the Additional Section for any
in-bailiwick SvcDomainNames.

Recursive resolvers that are aware of SVCB SHOULD ensure that the client can
execute the procedure in {{client-behavior}} without issuing a second
round of queries, by following this procedure while constructing a response
to a stub resolver:

1. When processing an SVCB response from an authoritative server, add it to
   the Additional section (unless it is the Answer).

2. Inspect whether each record is in AliasForm or ServiceForm.
   If at least one record is in AliasForm, ignore all other SVCB records in the
   RRSET.

3. If the record is in AliasForm, resolve A, AAAA, and SVCB records for
   the SvcDomainName.  If the SVCB record does not exist, add the A and AAAA
   records to the Additional section.  Otherwise, go to step 1.

3. If the records are in ServiceForm, resolve A and AAAA records for each
   SvcDomainName (or for the RRNAME if the SvcDomainName is "."), and include all
   the results in the Additional section.

All DNS servers SHOULD treat the SvcParam portion of the SVCB RR
as opaque and SHOULD NOT try to alter their behavior based
on its contents.


# Performance optimizations {#optimizations}

For optimal performance (i.e. minimum connection setup time), clients
SHOULD issue address (AAAA and/or A) and SVCB queries
simultaneously, and SHOULD implement a client-side DNS cache.
Responses in the Additional section of an SVCB response SHOULD be placed
in cache before performing any followup queries.
With these optimizations in place, and conforming DNS servers,
using SVCB does not add network latency to connection setup.

A nonconforming recursive resolver might not return all the information
required to use all the records in an SVCB response.  If
some of the SVCB RRs in the response can be used without requiring
additional DNS queries, the client MAY prefer those RRs, regardless of
their priorities.

To avoid a delay for clients using a nonconforming recursive resolver,
domain owners SHOULD use a single SVCB record whose SvcDomainName is in the
origin hostname's CNAME chain if possible.  This will ensure that the required
address records are already present in the client's DNS cache as part of the
responses to the address queries that were issued in parallel.


# Initial SvcParamKeys

A few initial SvcParamKeys are defined here.  These keys are useful for
HTTPS, and most are applicable to other protocols as well.

## "alpn"

The "alpn" SvcParamKey defines the Application Layer Protocol
(ALPN, as defined in {{!RFC7301}) supported by a TLS-based alternative
service.  Its value SHOULD be an entryin the IANA registry "TLS
Application-Layer Protocol Negotiation (ALPN) Protocol IDs".

The presentation format and wire format of SvcParamValue
is its registered "Identification Sequence".

Clients MUST ignore SVCB RRs where the "alpn" SvcParamValue
is unknown or unsupported.

## "port"

The "port" SvcParamKey defines the TCP or UDP port
that should be used to contact this alternative service.

The presentation format of the SvcParamValue is a numeric value
between 0 and 65535 inclusive.  The wire format of the SvcParamValue
is the corresponding 2 octet numeric value in network byte order.

## "esnikeys" {#svcparamkeys-esnikeys}

The SvcParamKey for ESNI is "esnikeys".  Its value is defined i
 {{esnikeys}}.  It is applicable to most TLS-based protocols.

## "a" and "aaaa"

The "a" and "aaaa" keys represent IP address hints for the service.
If A and AAAA records for SvcDomainName are locally available, the client SHOULD
ignore these hints.  When A and AAAA records arrive, clients SHOULD
switch to an IP address in those records as soon as possible.

The wire format for each parameter is a sequence of IP addresses in network
byte order.  Like an A or AAAA RRSet, the list of addresses represents an
unordered collection.

The presentation format for each parameter is a comma-separated list of
IP addresses in standard textual format {{!RFC5952}}.

These parameters are intended to minimize additional connection latency
when a recursive resolver is not compliant with the requirements in
{{server-behavior}}, and SHOULD NOT be included if most clients are using
compliant recursive resolvers.



# Using SVCB with HTTPS and HTTP {#https}

Use of any protocol with SVCB requires a protocol-specific mapping
specification.  This section specifies the mapping for HTTPS and HTTP.

To enable special handling for the HTTPS and HTTP use-cases,
the HTTPSSVC RRType is defined as an instantiation
of SVCB, specific to the https and http schemes.
This handling includes a mapping from HTTPSSVC records
directly into Alt-Svc entries.  HTTP and HTTPS clients MUST NOT
perform SVCB queries or accept SVCB responses.

The HTTPSSVC wire format and presentation format are
identical to SVCB, and both share the SvcParamKey registry.  SVCB
semantics apply equally to HTTPSSVC unless specified otherwise.

The presence of an HTTPSSVC record for an HTTP or HTTPS service also
provides an indication that all resources are available over HTTPS, as
discussed in {{hsts}}.  This also allows a HTTPSSVC RR to apply to
pre-existing HTTP scheme URLs but while ensuring that a secure and
authenticated HTTPS connection is still used.

For the HTTPS use-case, the HTTPSSVC RR extends the concept
introduced in the HTTP Alternative Services proposed standard
{{!AltSvc}}.  Alt-Svc defines:

* an extensible data model for describing alternative network endpoints
  that are authoritative for an origin
* the "Alt-Svc Field Value", a text format for representing this
  information
* standards for sending information in this format from a server to a
  client over HTTP/1.1 and HTTP/2.

Each ServiceForm HTTPSSVC RR provides a set of information that can be
mapped into an Alt-Svc Field Value.  A client receiving this
information during DNS resolution can skip the initial connection and
proceed directly to an alternative service.


## RRNames for HTTPSSVC record {#httpsrrnames}

The HTTPSSVC RR extends the behavior for determining
an RRName specified above in {{rrnames}}.

In particular, if the scheme is "https" with port 443, or the scheme
is "http" and the port is 80, then the RRName for the HTTPSSVC RR is
equal to the origin host name.  In this case, when a prior CNAME or
HTTPSSVC record has aliased to an HTTPSSVC record, RRName shall continue to be
the name of the alias target.

For schemes and ports other than https with port 443 and http with port 80,
the port and scheme continue to be prefixed to the hostname
as described in {{rrnames}}.

Note that none of these forms alter the HTTPS origin or authority.
For example, clients MUST continue to validate TLS certificate
hostnames based on the origin host.



## Mapping between ServiceForm and Alt-Svc {#map2altsvc}

To construct an Alt-Svc Field Value (as defined in Section 4 of
{{!AltSvc}}) from an HTTPSSVC record:

* The SvcDomainName is mapped into the uri-host portion of alt-authority.
  (If SvcDomainName is ".", the special handling described in
  {{svcdomainnamedot}} MUST be applied first.)

* The SvcParamValue of the "port" service parameter, or 443 if no such
  parameter is present, is written to the port portion of the alt-authority.

* The SvcParamValue of the "alpn" service parameter is mapped to the
  protocol-id.  This MUST follow the normalization and encoding
  requirements for protocol-id specified in {{!AltSvc}} Section 3.
  This parameter is MANDATORY.

* The DNS TTL is mapped to the "ma" (max age) Alt-Svc parameter.

* For SVCB parameters with defined mappings to HTTPS Alt-Svc, each should be
  included as an Alt-Svc parameter, typically as the SvcParamKey name
  "=" a defined encoding of the SvcParamValue.

Converting an Alt-Svc Field Value into an HTTPSSVC record follows the reverse
of this procedure.

Conversion from HTTPSSVC to Alt-Svc Field Value SHOULD ignore any
unrecognized SvcParamKeys, and conversion from Alt-Svc Field Value to
HTTPSSVC SHOULD ignore any Alt-Svc parameters that do not have a
corresponding SvcParamKey.

For example, if the operator of https://www.example.com
intends to include an HTTP response header like

    Alt-Svc: h3="svc.example.net:8003"; ma=3600; foo=123, \
             h2="svc.example.net:8002"; ma=3600; foo=123

they could also publish an HTTPSSVC DNS RRSet like

    www.example.com. 3600 IN HTTPSSVC 2 svc.example.net. (
                                        alpn=h3 port=8003 foo=123 )
                             HTTPSSVC 3 svc.example.net. (
                                        alpn=h2 port=8002 foo=123 )

Where "foo" is a hypothetical future HTTPSSVC and Alt-Svc parameter.

This data type can also be represented as an Unknown RR as described in
{{!RFC3597}}:

    www.example.com. 3600 IN TYPE??? \\# TBD:WRITEME

On connections to an HTTPSSVC alternative service, clients SHOULD
include the same Alt-Used header that they would include if the
corresponding Alt-Svc Field Value were received over HTTPS.


## Differences from Alt-Svc as transmitted over HTTP

Publishing an alternative services form HTTPSSVC record in DNS is intended
to be equivalent to transmitting the corresponding Alt-Svc value over
HTTPS, and receiving an HTTPSSVC record is intended to be equivalent to
receiving this field value over HTTPS.  However, there are some small
differences in the intended client and server behavior.

### Max Age and Persist

There is no SvcParamKey corresponding to the Alt-Svc "ma" (max age) parameter.
Instead, server operators SHOULD encode the
expiration time in the DNS TTL, and MUST NOT set a TTL longer than the
intended "max age".

For security reasons, there is no SvcParamKey corresponding to the Alt-Svc
"persist" parameter.

## Multiple records and preference ordering {#pri}

Server operators MAY publish multiple ServiceForm HTTPSSVC
records as an RRSET.  When converting a collection of alt-values
into an HTTPSSVC RRSET, the server operator MUST set the
overall TTL to a value no larger than the minimum
of the "max age" values (following Section 5.2 of {{!RFC2181}}).

Each RR corresponds to exactly one alt-value, as described
in Section 3 of {{!AltSvc}}.

As discussed in {{svcfieldpri}}, HTTPSSVC RRs with
a smaller SvcFieldPriority value SHOULD be sorted ahead of and given
preference over RRs with a larger SvcFieldPriority value.

Clients SHOULD prefer Alt-values received via HTTPS over any Alt-value
received via DNS.



### Constructing Alt-Svc equivalent headers

1. The RRs SHOULD be ordered by increasing SvcFieldPriority, with shuffling
   for equal SvcFieldPriority values.  Clients MAY choose to further
   prioritize alt-values where address records are immediately
   available for the alt-value's SvcDomainName.
2. The client SHOULD concatenate the thus-transformed-and-ordered
   SvcFieldValues in the RRSET, separated by commas.  (This is
   semantically equivalent to receiving multiple Alt-Svc HTTP response
   headers, according to Section 3.2.2 of {{?HTTP=RFC7230}}).

## Granularity and lifetime control

Sending Alt-Svc over HTTP allows the server to tailor the Alt-Svc
Field Value specifically to the client.  When using an HTTPSSVC DNS
record, groups of clients will necessarily receive the same Alt-Svc
Field Value.  Therefore, this standard is not suitable for uses that
require single-client granularity in Alt-Svc.

Some DNS caching systems incorrectly extend the lifetime of DNS
records beyond the stated TTL.  Server operators MUST NOT rely on
HTTPSSVC records expiring on time, and MAY shorten the TTL to compensate.


## HTTP Strict Transport Security {#hsts}

By publishing an HTTPSSVC record, the server
operator indicates that all useful HTTP resources on that origin are
reachable over HTTPS, similar to HTTP Strict Transport Security
{{!HSTS=RFC6797}}.  When an HTTPSSVC record is present for an origin,
all "http" scheme requests for that origin SHOULD logically be redirected
to "https".

Prior to making an "http" scheme request, the client SHOULD perform a lookup
to determine if an HTTPSSVC record is available for that origin.  To do so,
the client SHOULD construct a corresponding "https" URL as follows:

1. Replace the "http" scheme with "https".

2. If the "http" URL explicitly specifies port 80, specify port 443.

3. Do not alter any other aspect of the URL.

This construction is equivalent to Section 8.3 of {{HSTS}}, point 5.

If an HTTPSSVC record is present for this "https" URL, the client
should treat this as the equivalent of receiving an HTTP "307
Temporary Redirect" redirect to the "https" URL.
Because HTTPSSVC is received over an often insecure channel (DNS),
clients MUST NOT place any more trust in this signal than if they
had received a 307 redirect over cleartext HTTP.

If the HTTPSSVC query results in a SERVFAIL error, and the connection
between the client and the recursive resolver is cryptographically protected
(e.g. using TLS {{!RFC7858}} or HTTPS {{!RFC8484}}), the client SHOULD
abandon the connection attempt and display an error message.  A SERVFAIL
error can occur if the domain is DNSSEC-signed, the recursive resolver is
DNSSEC-validating, and an active attacker between the recursive resolver
and the authoritative DNS server is attempting to prevent the upgrade to
HTTPS.

Similarly, if the client enforces DNSSEC validation on A/AAAA
RRs, it SHOULD abandon the connection attempt if the HTTPSSVC RR fails
to validate.

## Cache interaction

If the client has an Alt-Svc cache, and a usable Alt-Svc value is
present in that cache, then the client SHOULD NOT issue an HTTPSSVC DNS
query.  Instead, the client SHOULD proceed with alternative service
connection as usual.

If the client has a cached Alt-Svc entry that is expiring, the
client MAY perform an HTTPSSVC query to refresh the entry.


# Extensions to enhance privacy


## Alt-Svc and SVCB/HTTPSSVC parameter for ESNI keys {#esnikeys}

Both SVCB/HTTPSSVC and Alt-Svc "esnikeys" parameters are defined for specifying
ESNI keys corresponding to an alternative service.
The value of the parameter is an ESNIKeys structure {{!ESNI}}
or the empty string.  ESNI-aware clients SHOULD prefer alt-values
and SVCB/HTTPSSVC RRs with non-empty esnikeys.

Both the SVCB SvcParamValue presentation format as well
as the Alt-Svc parameter value is the ESNIKeys structure {{!ESNI}}
encoded in {{!base64=RFC4648}} or the empty string.
The SVCB SvcParamValue wire format is the octet string
containing the binary ESNIKeys structure.

This parameter MAY also be sent in Alt-Svc HTTP response
headers and HTTP/2 ALTSVC frames.  This parameter MUST NOT appear more than
once in a single alt-value.


### Handling a mixture of alternatives not supporting esnikeys

The Alt-Svc specification states that "the client MAY fall back to using
the origin" in case of connection failure (Section 2.4 of {{!AltSvc}}).
This behavior is
not suitable for ESNI, because fallback would negate the privacy benefits of
ESNI.

Accordingly, any connection attempt that uses ESNI MUST fall back only to
another alt-value that also has the esnikeys parameter.  If the parameter's
value is the empty string, the client SHOULD connect as it would in the
absence of any ESNIKeys information.

For example, suppose a server operator has two alternatives.  Alternative A
is reliably accessible but does not support ESNI.  Alternative B supports
ESNI but is not reliably accessible.  The server operator could include a
full esnikeys value in Alternative B, and mark Alternative A with esnikeys=""
to indicate that fallback from B to A is allowed.

Other clients and services implementing SVCB or HTTPSSVC with esnikeys
are encouraged to take a similar approach.


# Interaction with other standards

The purpose of this standard is to reduce connection latency and
improve user privacy.  Server operators implementing this standard
SHOULD also implement TLS 1.3 {{!RFC8446}} and OCSP Stapling
{{!RFC6066}}, both of which confer substantial performance and privacy
benefits when used in combination with SVCB records.

To realize the greatest privacy benefits, this proposal is intended for
use over a privacy-preserving DNS transport (like DNS over TLS
{{!RFC7858}} or DNS over HTTPS {{!RFC8484}}).
However, performance improvements, and some modest privacy improvements,
are possible without the use of those standards.

Any specification for use of SVCB with a protocol MUST have an entry
for its scheme under the SVCB RRType in
the IANA DNS Underscore Global Scoped Entry Registry
{{!Attrleaf=I-D.ietf-dnsop-attrleaf}}.  The scheme SHOULD have an
entry in the IANA URI Schemes Registry {{!RFC7595}}.  The scheme
SHOULD have a defined specification for use with SVCB, unless it
already has a specification for use with Alt-Svc.



# Security Considerations

SVCB/HTTPSSVC RRs and Alt-Svc Field Values are intended for distribution over untrusted
channels, and clients are REQUIRED to verify that the alternative
service is authoritative for the origin (Section 2.1 of {{!AltSvc}}).
Therefore, DNSSEC signing and validation are OPTIONAL for publishing
and using SVCB and HTTPSSVC records.

Clients MUST ensure that their DNS cache is partitioned for each local
network, or flushed on network changes, to prevent a local adversary in one
network from implanting a forged DNS record that allows them to
track users or hinder their connections after they leave that network.


# IANA Considerations

## New registry for Service Parameters {#svcparamregistry}

The "Service Binding (SVCB) Parameter Registry" defines the name space
for parameters, including string representations and numeric
SvcParamKey values.  This registry is shared with other RRTypes
extending SVCB, such as HTTPSSVC.

ACTION: create and include a reference to this registry.

### Procedure

A registration MUST include the following fields:

* Name: Service parameter key name
* SvcParamKey: Service parameter key numeric identifier (range 0-65535)
* Meaning: a short description
* Pointer to specification text

Values to be added to this name space require Expert Review (see
{{!RFC5226}}, Section 4.1).  Apart from the initial contents, the name
MUST NOT start with "key".

### Initial contents

The "Service Binding (SVCB) Parameter Registry" shall initially
be populated with the registrations below:

| SvcParamKey | NAME        | Meaning                       | Reference       |
| ----------- | ------      | ----------------------        | --------------- |
| 0           | key0        | Reserved                      | (This document) |
| 1           | alpn        | ALPN for alternative service  | (This document) |
| 2           | port        | Port for alternative service  | (This document) |
| 3           | esnikeys    | ESNI keys literal             | (This document) |
| 65280-65534 | keyNNNNN    | Private Use                   | (This document) |
| 65535       | key65535    | Reserved                      | (This document) |

TODO: do we also want to reserve a range for greasing?


## Registry updates

Per {{?RFC6895}}, please add the following entry to the data type
range of the Resource Record (RR) TYPEs registry:

| TYPE     | Meaning                                      | Reference       |
| ------   | ----------------------                       | --------------- |
| SVCB     | Service Location and Parameter Binding       | (This document) |
| HTTPSSVC | HTTPS Service Location and Parameter Binding | (This document) |


Per {{?Attrleaf}}, please add the following entries to the DNS Underscore
Global Scoped Entry Registry:

| RR TYPE   | _NODE NAME | Meaning           | Reference       |
| --------- | ---------- | ----------------- | --------------- |
| HTTPSSVC  | _https     | Alt-Svc for HTTPS | (This document) |
| HTTPSSVC  | _http      | Alt-Svc for HTTPS | (This document) |


Per {{?AltSvc}}, please add the following entries to the HTTP Alt-Svc
Parameter Registry:

| Alt-Svc Parameter | Meaning              | Reference       |
| ----------------- | -------------------- | --------------- |
| esnikeys          | Encrypted SNI keys   | (This document) |



# Acknowledgements and Related Proposals

There have been a wide range of proposed solutions over the years to
the "CNAME at the Zone Apex" challenge proposed.  These include
{{?I-D.draft-bellis-dnsop-http-record-00}},
{{?I-D.draft-ietf-dnsop-aname-03}}, and others.

Thank you to Ian Swett, Ralf Weber, Jon Reed,
Martin Thompson, Lucas Pardue, Ilari Liusvaara,
Tim Wicinski, Tommy Pauly, Chris Wood,
and others for their feedback and suggestions on this draft.


--- back

# Additional examples

## Equivalence to Alt-Svc records

The following:

    www.example.com.  7200  IN CNAME    svc.example.net.
    example.com.      7200  IN HTTPSSVC 0 svc.example.net.
    svc.example.net.  7200  IN HTTPSSVC 2 svc3.example.net. (
                                        alpn=h3 port=8003 esnikeys="ABC..." )
    svc.example.net.  7200  IN HTTPSSVC 3 . alpn=h2 port=8002 esnikeys="123..."

is equivalent to the Alt-Svc record:

    Alt-Svc: h3="svc3.example.net:8003"; esnikeys="ABC..."; ma=7200, \
             h2="svc.example.net:8002"; esnikeys="123..."; ma=7200

for the origins of both "https://www.example.com" and "https://example.com".

# Comparison with alternatives

The SVCB and HTTPSSVC records type closely resembles,
and is inspired by, some existing
record types and proposals.  A complaint with all of the alternatives
is that web clients have seemed unenthusiastic about implementing
them.  The hope here is that by providing an extensible solution that
solves multiple problems we will overcome the inertia and have a path
to achieve client implementation.

## Differences from the SRV RRTYPE

An SRV record {{?RFC2782}} can perform a similar function to the SVCB record,
informing a client to look in a different location for a service.
However, there are several differences:

* SRV records are typically mandatory, whereas clients will always
  continue to function correctly without making use of Alt-Svc or SVCB.
* SRV records cannot instruct the client to switch or upgrade
  protocols, whereas Alt-Svc can signal such an upgrade (e.g. to
  HTTP/2).
* SRV records are not extensible, whereas SVCB and HTTPSSVC and
  Alt-Svc can be extended with
  new parameters.  For example, this is what allows the incorporation of
  ESNI keys in SVCB.
* Using SRV records would not allow a client to skip processing of the
  Alt-Svc information in a subsequent connection, so it does not confer
  a performance advantage.

## Differences from the proposed HTTP record

Unlike {{?I-D.draft-bellis-dnsop-http-record-00}}, this approach is
extensible to cover Alt-Svc and ESNIKeys use-cases.  Like that
proposal, this addresses the zone apex CNAME challenge.

Like that proposal it remains necessary to continue to include
address records at the zone apex for legacy clients.


## Differences from the proposed ANAME record

Unlike {{?I-D.draft-ietf-dnsop-aname-03}}, this approach is extensible to
cover Alt-Svc and ESNIKeys use-cases.  This approach also does not
require any changes or special handling on either authoritative or
master servers, beyond optionally returning in-bailiwick additional records.

Like that proposal, this addresses the zone apex CNAME challenge
for clients that implement this.

However with this SVCB proposal it remains necessary to continue
to include address records at the zone apex for legacy clients.
If deployment of this standard is successful, the number of legacy clients
will fall over time.  As the number of legacy clients declines, the operational
effort required to serve these users without the benefit of SVCB indirection
should fall.  Server operators can easily observe how much traffic reaches this
legacy endpoint, and may remove the apex's address records if the observed legacy
traffic has fallen to negligible levels.


## Differences from the proposed ESNI record

Unlike {{!ESNI}}, this approach is extensible and covers
the Alt-Svc case as well as addresses the zone apex CNAME challenge.

By using the Alt-Svc model we also provide a way to solve
the ESNI multi-CDN challenges in a general case.

Unlike ESNI, this is focused on the specific case of HTTPS,
although this approach could be extended for other protocols.
It also allows specifying ESNI keys for a specific port, not an entire host.

## SNI Alt-Svc parameter

Defining an Alt-Svc sni= parameter
(such as from {{!AltSvcSNI=I-D.bishop-httpbis-sni-altsvc}}) would
have provided some benefits to clients and servers not implementing ESNI,
such as for specifying that "_wildcard.example.com" could be sent as an SNI
value rather than the full name.  There is nothing precluding SVCB from
being used with an sni= parameter if one were to be defined, but it
is not included here to reduce scope, complexity, and additional potential
security and tracking risks.

# Design Considerations and Open Issues

This draft is intended to be a work-in-progress for discussion.
Many details are expected to change with subsequent refinement.
Some known issues or topics for discussion are listed below.

## Record Name

Naming is hard.  The "SVCB" and "HTTPSSVC" are proposed as placeholders
that are easy to search for and replace when a final
name is chosen.
Other names for this record might include B, ALTSVC,
HTTPS, HTTPSSRV, HTTPSSVC, SVCHTTPS, or something else.

## Applicability to other schemes

The focus of this record is on optimizing the common case of the
"https" scheme.  It is worth discussing whether this is a valid
assumption or if a more general solution is applicable.  Past efforts
to over-generalize have not met with broad success.

## Wire Format

Advice from experts in DNS wire format best practices would be greatly
appreciated to refine the proposed details, overall.

## Where to include Priority

The SvcFieldPriority could alternately be included as a pri= Alt-Svc attribute.
It wouldn't be applicable for Alt-Svc returned via HTTP, but it is also
not necessarily needed by DNS servers.  It is also not used for AliasForm RRs.
Regardless, having a series of sequential numeric
values in the textual representation has risk of user error, especially
as MX, SRV, and others all have their own variations here.

## Whether to include Weight

Some other similar mechanisms such as SRV have a weight in-addition
to priority.  That is excluded here for simplicity.  It could always be
added as an optional SVCB parameter.


# Change history

* draft-nygren-dnsop-svcb-httpssvc-00
    * Generalize to a SVCB record, with special-case
      handling for Alt-Svc and HTTPS separated out
      to dedicated sections.
    * Split out a separate HTTPSSVC record for
      the HTTPS use-case.
    * Remove the explicit SvcRecordType=0/1 and instead
      make the AliasForm vs ServiceForm be implicit.
      This was based on feedback recommending against
      subtyping RRTYPE.
    * Remove one optimization.

* draft-nygren-httpbis-httpssvc-03
    * Change redirect type for HSTS-style behavior
      from 302 to 307 to reduce ambiguities.

* draft-nygren-httpbis-httpssvc-02
    * Remove the redundant length fields from the wire format.
    * Define a SvcDomainName of "." for SvcRecordType=1
      as being the HTTPSSVC RRNAME.
    * Replace "hq" with "h3".

* draft-nygren-httpbis-httpssvc-01
    * Fixes of record name.  Replace references to "HTTPSVC" with "HTTPSSVC".

* draft-nygren-httpbis-httpssvc-00
    * Initial version

