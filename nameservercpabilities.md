# Name Server Capabilities

This document tries to list all capabilities needed for running a DNSSEC Multi-Signer setup.
Capabilities are devided in groups by mode of access.

- command line access
- dynamic DNS updates (secured with TSIG)
- Rest API

Please let us know if you miss any particulare brand of name server, capabilities or access method.

## Command line
Capability | Bind | Knot | PowerDNS
---------- | ---- | ---- | --------
Add DNSKEY records (without access to private key)| | | Yes<sup>1</sup>
Add CDS/CDNSKEY record for keys not in the DNSKEY set| | | Yes<sup>1</sup>
Add CSYNC record| | | Yes<sup>2</sup>

<sup>1</sup> `pdnsutil add-record example.com. . DNSKEY "257 3 13 aCo..."` (or `CDNSKEY`; TTL not needed if RRset already present)

<sup>2</sup> Untested, but should work with latest build. Remove this note when tested.

## Dynamic DNS update
Capability | Bind | Knot | PowerDNS
---------- | ---- | ---- | --------
Add DNSKEY records (without access to private key)|No|No|No
Add CDS/CDNSKEY record for keys not in the DNSKEY set|No|No|No
Add CSYNC record|No|No|No

## Rest API
Capability | Bind | Knot | PowerDNS
---------- | ---- | ---- | --------
Add DNSKEY records (without access to private key)|No|No|Yes
Add CDS/CDNSKEY record for keys not in the DNSKEY set|No|No|Yes
Add CSYNC record|No|No|Yes (with latest build)
