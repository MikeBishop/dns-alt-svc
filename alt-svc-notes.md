# Alt-Svc ideas related to HTTPSSVC and ESNI

This file contains notes related to Alt-Svc that were removed
from draft-ietf-dnsop-svcb-httpssvc.  This includes some
requirements or specifications that have been considered
during development of that draft but are no longer part of it.

## Populating Alt-Used

When using an HTTPSSVC RR in ServiceForm, all clients SHOULD
include the "Alt-Used" HTTP header (Section 5 of {{!RFC7838}}).
The header's value (in ABNF) SHOULD be

    uri-host ":" port

where uri-host is the final value of HOST ({client-behavior}) minus
the trailing ".", and port is the port number in use.

----

Per {{?AltSvc}}, please add the following entry to the HTTP Alt-Svc
Parameter Registry:

| Alt-Svc Parameter | Meaning                     | Reference       |
| ----------------- | --------------------------- | --------------- |
| esniconfig        | Encrypted SNI configuration | (This document) |


----

{{map2altsvc}} defines a limited mapping between Alt-Svc ({{!AltSvc}}) values
and the SVCB ServiceForm.  Protocols using SVCB may use this Alt-Svc
mapping if they also use Alt-Svc.  

----

If the client has a cached Alt-Svc entry that is expiring, the
client MAY perform an HTTPSSVC query to refresh the entry.

----

# Mapping between HTTPSSVC and Alt-Svc {#map2altsvc}

Conversion between HTTPSSVC's ServiceForm and Alt-Svc is possible.
Note that conversion in either direction can be lossy, because
some parameters are only defined for HTTPSSVC or Alt-Svc.

To construct an Alt-Svc Field Value (as defined in Section 4 of
{{!AltSvc}}) from an HTTPSSVC record:

* The SvcDomainName is mapped into the uri-host portion of alt-authority
  with the trailing "." removed.
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

Conversion between HTTPSSVC and Alt-Svc Field Value MUST ignore any
SvcParamKeys and Alt-Svc parameters that are unrecognized or do not have
a defined mapping.

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

## Multiple records and preference ordering {#pri}

Server operators MAY publish multiple ServiceForm HTTPSSVC
records as an RRSet.  When converting a collection of alt-values
into an HTTPSSVC RRSet, the server operator MUST set the
overall TTL to a value no larger than the minimum
of the "max age" values (following Section 5.2 of {{!RFC2181}}).

Each RR corresponds to exactly one alt-value, as described
in Section 3 of {{!AltSvc}}.

As discussed in {{svcfieldpri}}, HTTPSSVC RRs with
a smaller SvcFieldPriority value SHOULD be sorted ahead of and given
preference over RRs with a larger SvcFieldPriority value.

When constructing equivalent Alt-Svc headers from an RRSet:

1. The RRs SHOULD be ordered by increasing SvcFieldPriority, with shuffling
   for equal SvcFieldPriority values.  Clients MAY choose to further
   prioritize alt-values where address records are immediately
   available for the alt-value's SvcDomainName.
2. The client SHOULD concatenate the thus-transformed-and-ordered
   SvcFieldValues in the RRSet, separated by commas.  (This is
   semantically equivalent to receiving multiple Alt-Svc HTTP response
   headers, according to Section 3.2.2 of {{?HTTP=RFC7230}}).


## Additional examples

The following:

    www.example.com.  7200  IN CNAME    svc.example.net.
    example.com.      7200  IN HTTPSSVC 0 svc.example.net.
    svc.example.net.  7200  IN HTTPSSVC 2 svc3.example.net. (
        alpn=h3 port=8003 esniconfig="ABC..." )
    svc.example.net.  7200  IN HTTPSSVC 3 . (
        alpn=h2 port=8002 esniconfig="123..." )

is equivalent to the Alt-Svc record:

    Alt-Svc: h3="svc3.example.net:8003"; esniconfig="ABC..."; ma=7200, \
             h2="svc.example.net:8002"; esniconfig="123..."; ma=7200

for the origins of both "https://www.example.com" and "https://example.com".

## SNI Alt-Svc parameter

Defining an Alt-Svc sni= parameter
(such as from {{!AltSvcSNI=I-D.bishop-httpbis-sni-altsvc}}) would
have provided some benefits to clients and servers not implementing ESNI,
such as for specifying that "_wildcard.example.com" could be sent as an SNI
value rather than the full name.  There is nothing precluding SVCB from
being used with an sni= parameter if one were to be defined, but it
is not included here to reduce scope, complexity, and additional potential
security and tracking risks.

