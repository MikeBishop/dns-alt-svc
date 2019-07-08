The following are known prototype implementations 
of [draft-nygren-httpbis-httpssvc](https://datatracker.ietf.org/doc/draft-nygren-httpbis-httpssvc/).
At least one of these is using TYPE65479 so that may make sense to use for private testing
until a assignment is made.

Note that wire format and behavior changes are still being made so HTTPSSVC
should not be used for production purposes.

# Prototype private type implementations #

## BIND9 ##

[Private Type implementation for BIND9](https://gitlab.isc.org/isc-projects/bind9/merge_requests/2135)

* Author: Mark Andrews \<marka@isc.org\> 
* Version: Implements -01 but also anticipated -02 changes under an ifdef
* TYPENN: 65479

