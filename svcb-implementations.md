The following are known prototype implementations 
of [draft-ietf-dnsop-svcb-https](https://datatracker.ietf.org/doc/draft-ietf-dnsop-svcb-https/) 

Note some prototypes started off using TYPE65479 and other private types but are now switching over to the production types now that the wire format is stable.

Please feel free to submit PRs to update this page.

# Production / shipped implementations #

(TBD)

# Work-in-progress and prototype implementations #

## BIND9 ##

[Work-in-progress implementation for BIND9](https://gitlab.isc.org/isc-projects/bind9/merge_requests/2135)

* Author: Mark Andrews \<marka@isc.org\> 
* Tracker: [BIND9 GL 1132](https://gitlab.isc.org/isc-projects/bind9/-/issues/1132)
* Version: Implement draft-ietf-dnsop-svcb-https-01 (work-in-progress)
** Previous versions implemented draft-nygren-httpbis-httpssvc-02 (and -01) and draft-nygren-dnsop-svcb-httpssvc-00
** Previous versions used TYPENN of HTTPS/65482 and SVBC/65481

## PowerDNS ##

[Pull Request](https://github.com/PowerDNS/pdns/pull/9369).

## Unbound ##

* Prototype of draft-nygren-httpbis-httpssvc-02 during IETF 105 hackathon

## dnspython ##

Support for draft-ietf-dnsop-svcb-https-01 is available on the master
branch and will be included in dnspython 2.1, which is currently targeted
for release around the end of September 2020.

## Perl Net::DNS ##

Per Dick Franks <rwfranks@gmail.com>, HTTPS and SVCB will be in Net::DNS 1.25_01 coming soon to CPAN.

## dnsjava ##

[Work-in-progress implementation for dnsjava](https://github.com/dnsjava/dnsjava/pull/116) by [adam-stoler](https://github.com/adam-stoler)

## Others ##

These were found by some web searches so you milage may vary:

* https://github.com/miekg/dns/pull/1067


# Clients using and/or announced support for 

## iOS & macOS ##

iOS 14 supports HTTPS/SVCB records as defined in draft-ietf-dnsop-svcb-https-01. Type 65 (HTTPS) is requested
for all URLSession or Network.framework connections that use an http or https scheme, or use ports 80 or 443.

Betas of macOS 11 support the same, since beta 4.

# Services using and/or announced support for SVCB/HTTPS records #

## Cloudflare ##

Cloudflare's authoritative DNS servers reply to HTTPS queries for domains for
which Cloudflare provides HTTPS termination.

The following domains can be used for testing (along any other domain served by
Cloudflare): blog.cloudflare.com, www.cloudflare.com, cdnjs.cloudflare.com,
cloudflare-http3.com, cloudflare-http2.com, cloudflare-http1.com.

## Firefox ##

Firefox supports HTTPS RR since Firefox 81. It is currently disabled and can be enabled by changing prefs: network.dns.upgrade_with_https_rr and network.dns.use_https_rr_as_altsvc (type about:config in the address bar and search for the prefs; set them to true). At the moment this is only supported if DoH is enabled.
