# Name Server Capabilities

This document tries to list all capabilities needed for running a DNSSEC Multi-Signer setup.
Capabilities are devided in groups by mode of access.

- command line access
- dynamic DNS updates (secured with TSIG)
- Rest API

Please let us know if you miss any particulare brand of name server, capabilities or access method.

**Authentication**: determined by provider (usually ssh)

## Capabilities

All capabilities specify CLI (💻) / DDNS (🗃) / API (⚙) with possible states of **completed** (✔), **in progress** (▶), **planned** (⏸), **not planned** (➖), **unsupported** (❌), or unknown (❔).

<table>
  <tr>
    <th rowspan="2">Capability</th>
    <th colspan="3">Bind</th>
    <th colspan="3">Knot</th>
    <th colspan="3">PowerDNS<sup>1</sup></th>
    <th colspan="3">NSD<sup>2</sup></th>
  </tr>
  <tr>
    <td>💻</td>
    <td>🗃</td>
    <td>⚙</td>
    <td>💻</td>
    <td>🗃</td>
    <td>⚙</td>
    <td>💻</td>
    <td>🗃</td>
    <td>⚙</td>
    <td>💻</td>
    <td>🗃</td>
    <td>⚙</td>
  </tr>
  <tr>
    <td>Add DNSKEY records (without access to private key)</td>
    <td>✔</td>
    <td>✔</td>
    <td>➖</td>
    <td>✔</td>
    <td>▶</td>
    <td>➖</td>
    <td>✔</td>
    <td>✔</td>
    <td>✔</td>
    <td>❌</td>
    <td>❌</td>
    <td>❌</td>
  </tr>
  <tr>
    <td>Remove (previously added) DNSKEY record(s)</td>
    <td>✔</td>
    <td>✔</td>
    <td>➖</td>
    <td>✔</td>
    <td>▶</td>
    <td>➖</td>
    <td>✔</td>
    <td>✔</td>
    <td>✔</td>
    <td>❌</td>
    <td>❌</td>
    <td>❌</td>
  </tr>
  <tr>
    <td>Add CDS/CDNSKEY record for keys not in the DNSKEY set</td>
    <td>✔</td>
    <td>✔</td>
    <td>➖</td>
    <td>❔</td>
    <td>▶</td>
    <td>➖</td>
    <td>✔</td>
    <td>✔</td>
    <td>✔</td>
    <td>❌</td>
    <td>❌</td>
    <td>❌</td>
  </tr>
  <tr>
    <td>Remove (previously added) CDS/CDNSKEY records</td>
    <td>✔</td>
    <td>✔</td>
    <td>➖</td>
    <td>✔</td>
    <td>⏸</td>
    <td>➖</td>
    <td>✔</td>
    <td>✔</td>
    <td>✔</td>
    <td>❌</td>
    <td>❌</td>
    <td>❌</td>
  </tr>
  <tr>
    <td>Add CSYNC record</td>
    <td>✔</td>
    <td>✔</td>
    <td>➖</td>
    <td>✔</td>
    <td>▶</td>
    <td>➖</td>
    <td>✔</td>
    <td>✔</td>
    <td>✔</td>
    <td>❌</td>
    <td>❌</td>
    <td>❌</td>
  </tr>
  <tr>
    <td>Remove CSYNC record</td>
    <td>✔</td>
    <td>✔</td>
    <td>➖</td>
    <td>✔</td>
    <td>▶</td>
    <td>➖</td>
    <td>✔</td>
    <td>✔</td>
    <td>✔</td>
    <td>❌</td>
    <td>❌</td>
    <td>❌</td>
  </tr>
</table>

<!--
Capability | Bind | Knot | PowerDNS | NSD
---------- | ---- | ---- | -------- | ---
Add DNSKEY records (without access to private key) | Yes/No/No | Yes/No/No | Yes<sup>1</sup>/Yes/Yes | n/a<sup>2</sup>
Remove (previously added) DNSKEY record(s) | | | Yes/Yes/Yes | n/a<sup>2</sup>
Add CDS/CDNSKEY record for keys not in the DNSKEY set | No/No/No| No/No/No | Yes<sup>1</sup>/Yes/Yes | n/a<sup>2</sup>
Remove (previously added) CDS/CDNSKEY records | | | Yes/Yes/Yes | n/a<sup>2</sup>
Add CSYNC record | ?/?/No | ?/?/No | Yes<sup>1</sup>/Yes/Yes | n/a<sup>2</sup>
Remove CSYNC record | | | Yes/Yes/Yes | n/a<sup>2</sup>
-->
<sup>1</sup> `pdnsutil add-record example.com. . DNSKEY "257 3 13 aCo..."` (or `CDNSKEY`; TTL not needed if RRset already present)

<sup>2</sup> not applicable -  NSD does not do DNSSEC signing.


Good news from ISC. It seems they consider implementation: https://gitlab.isc.org/isc-projects/bind9/-/issues/2682

Things are moving for Knot DNS too https://gitlab.nic.cz/knot/knot-dns/-/merge_requests/1289/commits
