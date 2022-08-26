The following are known implementations 
of [draft-ietf-dnsop-svcb-https](https://datatracker.ietf.org/doc/draft-ietf-dnsop-svcb-https/) 

Please feel free to submit PRs to update this page.

# Production / shipped implementations #

## iOS & macOS ##

iOS 14 (September 2020) and macOS 11 (November 2020) support HTTPS/SVCB records. Type 65 (HTTPS) is requested
for all URLSession or Network.framework connections that use an http or https scheme, or use ports 80 or 443.

## PowerDNS ##

Implemented in [PowerDNS 4.4.0](https://doc.powerdns.com/authoritative/changelog/4.4.html#change-4.4.0), December 2020.

## Knot DNS ##

Supported since [Knot DNS 3.0.3](https://gitlab.nic.cz/knot/knot-dns/-/releases/v3.0.3), December 2020.

## Perl Net::DNS ##

Implemented in [Net::DNS 1.26](https://www.net-dns.org/blog/2020/08/05/netdns-1-26-released/), August 2020.

## dnspython ##

Implemented [in dnspython 2.1.0](https://dnspython.readthedocs.io/en/stable/whatsnew.html#id1), January 2021.

## dnsjava ##

Supported [in dnsjava 3.3.0](https://github.com/dnsjava/dnsjava/blob/master/Changelog), September 2020.

## miekg/dns (Go) ##

[Supported](https://github.com/miekg/dns/pull/1067) since v1.1.35, October 2020.

## trust-dns (Rust) ##

Implemented in [trust-dns 0.21.0](https://github.com/bluejekyll/trust-dns/blob/main/CHANGELOG.md#0201), March 2021.

# Services supporting SVCB/HTTPS records #

## Cloudflare ##

Cloudflare's authoritative DNS servers reply to HTTPS queries for domains for
which Cloudflare provides HTTPS termination.

The following domains can be used for testing (along any other domain served by
Cloudflare): blog.cloudflare.com, www.cloudflare.com, cdnjs.cloudflare.com,
cloudflare-http3.com, cloudflare-http2.com, cloudflare-http1.com.

## Akamai services ##

Akamai Global Traffic Management (load balancing) and Edge DNS (authoritative DNS) [support the use of HTTPS and SVCB records](https://community.akamai.com/customers/s/article/NetworkOperatorCommunityNewSVCBHTTPSResourceRecordsinthewild20201128135350) (November 2020).

Akamai CacheServe (recursive resolver) supports HTTPS records.  [As of February 2021](https://indico.dns-oarc.net/event/37/contributions/810/attachments/784/1413/dns-https-rr-final.pdf), HTTPS QTYPEs accounted for several percent of queries.

## Google Public DNS JSON API ##

The Google Public DNS JSON API supports [typed serialization](https://dns.google/query?name=blog.cloudflare.com&rr_type=HTTPS&ecs=) of SVCB and HTTPS records (March 2021).

## NS1 ##

NS1 authoritative DNS supports [SVCB and HTTPS](https://ns1.com/blog/ns1-announces-support-for-svcb-and-https-records) records (July 2022).

# Pre-release and public testing implementations #

## Firefox ##

Firefox supports HTTPS RR since Firefox 81. It is currently disabled and can be enabled by changing prefs: network.dns.upgrade_with_https_rr and network.dns.use_https_rr_as_altsvc (type about:config in the address bar and search for the prefs; set them to true). At the moment this is only supported if DoH is enabled.

## Chrome ##

Chrome 88 [performs HTTPS queries](https://groups.google.com/a/chromium.org/g/blink-dev/c/brZTXr6-2PU) in some configurations (e.g. when Secure DNS is enabled on the Beta channel, December 2020).

## NSD ##

Support [is implemented](https://github.com/NLnetLabs/nsd/blob/master/doc/ChangeLog) in the development branch (April 2021).

# Under development #

## BIND9 ##

[Work-in-progress implementation for BIND9](https://gitlab.isc.org/isc-projects/bind9/merge_requests/2135)

* Author: Mark Andrews \<marka@isc.org\> 
* Tracker: [BIND9 GL 1132](https://gitlab.isc.org/isc-projects/bind9/-/issues/1132)
* Version: Implement draft-ietf-dnsop-svcb-https-01 (work-in-progress)
** Previous versions implemented draft-nygren-httpbis-httpssvc-02 (and -01) and draft-nygren-dnsop-svcb-httpssvc-00
** Previous versions used TYPENN of HTTPS/65482 and SVBC/65481

## Unbound ##

* Prototype of draft-nygren-httpbis-httpssvc-02 during IETF 105 hackathon

## Others ##

