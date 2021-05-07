# Capabilities of DNS Service providers

This document tries to list all capabilities needed for running a DNSSEC Multi-Signer setup.
Capabilities are devided in groups by mode of access.

- dynamic DNS updates (secured with TSIG)
- Rest API
- Web based user interface
- 
Please let us know if you miss any particulare brand of name server, capabilities or access method.

## Authentication
  - DDNS usually TSIG
  - Rest API usually basic auth or jwt, but decided by provider
  - user interface, decided by provider

## Capabilities

All capabilities specify CLI/DDNS/API.

Capability | deSEC.io
---------- | ---- 
Add DNSKEY records (without access to private key) | No/Yes/Yes 
Remove (previously added) DNSKEY record(s) | | |
Add CDS/CDNSKEY record for keys not in the DNSKEY set | No/Yes/Yes
Remove CDS/CDNSKEY records | | |
Add CSYNC record | No/Yes/Yes
Remove CSYNC record | | |
