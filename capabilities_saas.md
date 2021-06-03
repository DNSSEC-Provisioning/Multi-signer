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

All capabilities specify DDNS/API/Web.

Capability | deSEC.io | NS1 | Neustar
---------- | -------- | --- | -------
Add DNSKEY records (without access to private key) | No/Yes/Yes | <sup>1</sup> | <sup>1</sup>
Remove (previously added) DNSKEY record(s) | | <sup>1</sup> | <sup>1</sup>
Add CDS/CDNSKEY record for keys not in the DNSKEY set | No/Yes/Yes | <sup>1</sup> | <sup>1</sup>
Remove CDS/CDNSKEY records | | <sup>1</sup> | <sup>1</sup>
Add CSYNC record | No/Yes/Yes | <sup>1</sup> | <sup>1</sup>
Remove CSYNC record | | <sup>1</sup> | <sup>1</sup>

<sup>1</sup> Work In Progress
