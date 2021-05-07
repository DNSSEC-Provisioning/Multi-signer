# Name Server Capabilities

This document tries to list all capabilities needed for running a DNSSEC Multi-Signer setup.
Capabilities are devided in groups by mode of access.

- command line access
- dynamic DNS updates (secured with TSIG)
- Rest API

Please let us know if you miss any particulare brand of name server, capabilities or access method.

**Authentication**: determined by provider (usually ssh)

## Capabilities

All capabilities specify CLI/DDNS/API.

Capability | Bind | Knot | PowerDNS
---------- | ---- | ---- | --------
Add DNSKEY records (without access to private key) | Yes/No/No | Yes/No/No | Yes<sup>1</sup>/Yes/Yes
Remove (previously added) DNSKEY record(s) | | |
Add CDS/CDNSKEY record for keys not in the DNSKEY set | No/No/No| No/No/No | Yes<sup>1</sup>/Yes/Yes
Remove CDS/CDNSKEY records | | |
Add CSYNC record | ?/?/No | ?/?/No | Yes<sup>2</sup>/Yes<sup>2</sup>/Yes<sup>2</sup>
Remove CSYNC record | | |

<sup>1</sup> `pdnsutil add-record example.com. . DNSKEY "257 3 13 aCo..."` (or `CDNSKEY`; TTL not needed if RRset already present)

<sup>2</sup> Untested, but should work with latest build. Remove this note when tested.

Good news from ISC. It seems they consider implementation: https://gitlab.isc.org/isc-projects/bind9/-/issues/2682
