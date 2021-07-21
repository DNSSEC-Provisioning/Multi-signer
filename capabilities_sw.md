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

Capability | Bind | Knot | PowerDNS | NSD
---------- | ---- | ---- | -------- | ---
Add DNSKEY records (without access to private key) | Yes/No/No | Yes/No/No | Yes<sup>1</sup>/Yes/Yes | n/a<sup>2</sup>
Remove (previously added) DNSKEY record(s) | | | Yes/Yes/Yes | n/a<sup>2</sup>
Add CDS/CDNSKEY record for keys not in the DNSKEY set | No/No/No| No/No/No | Yes<sup>1</sup>/Yes/Yes | n/a<sup>2</sup>
Remove (previously added) CDS/CDNSKEY records | | | Yes/Yes/Yes | n/a<sup>2</sup>
Add CSYNC record | ?/?/No | ?/?/No | Yes<sup>1</sup>/Yes/Yes | n/a<sup>2</sup>
Remove CSYNC record | | | Yes/Yes/Yes | n/a<sup>2</sup>

<sup>1</sup> `pdnsutil add-record example.com. . DNSKEY "257 3 13 aCo..."` (or `CDNSKEY`; TTL not needed if RRset already present)

<sup>2</sup> not applicable -  NSD does not do DNSSEC signing.

Good news from ISC. It seems they consider implementation: https://gitlab.isc.org/isc-projects/bind9/-/issues/2682

Things are moving for Knot DNS too https://gitlab.nic.cz/knot/knot-dns/-/merge_requests/1289/commits
