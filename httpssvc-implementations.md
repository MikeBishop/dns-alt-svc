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

## Unbound ##

* Prototype of draft-nygren-httpbis-httpssvc-02 during IETF 105 hackathon

## Others ##

These were found by some web searches so you milage may vary:

* https://github.com/miekg/dns/pull/1067


# Clients using and/or announced support for 

(TBD) 

# Services using and/or announced support for SVCB/HTTPS records #

(TBD)


