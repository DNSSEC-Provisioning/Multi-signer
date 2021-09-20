# Testing Multi-signer set up using Knot DNS.

* Ubuntu 20.04
* Knot DNS, version 3.1.2

#### Notes
* Third level domain (multisigner.examples.nu) used for easy management of DS records
* Provisioned om micro EC2 in AWS
* This example is for adding a server to the group set up in the multi-signer group created in [this guide](multi-signer_test_set_up_on_BIND.md)
* IMPORTANT! Setup is experimental. Modifications to suggested order of operations required. 


## Knot install and set up on Ubuntu

#### Install Knot
```bash
sudo add-apt-repository ppa:cz.nic-labs/knot-dns-latest
```
```bash
sudo apt-get update
```
```bash
sudo apt-get install knot
```
```bash
sudo apt-get install knot-dnsutils
```

#### Create zone file
```bash
sudo vi /var/lib/knot/multisigner.examples.nu
```
```
$ORIGIN multisigner.examples.nu.
$TTL 120
@       SOA     ns3.multisigner.examples.nu. dns.examples.nu. 1618586094 14400 3600 1814400 120

@       NS      ns3
ns3     A       13.51.108.122
```



#### Add configuration
```bash
sudo vi /etc/knot/knot.conf
```

Update server statement as needed.

Note: In AWS, the local address of the machine will be bound to the public IP of the EIP gateway. (This public IP address is not visible on the EC2 host). 

```
server:
    rundir: "/run/knot"
    user: knot:knot
    listen: [ 127.0.0.1@53, ::1@53, 172.31.29.72@53 ]
```


Add ACL statement for limiting nsupdate/axfr. For simplicity, give access to local machine only, based on IP.
```
acl:
   - id: aclupdate
     address: [ 127.0.0.1, ::1, 172.31.29.72 ]
     action: [ transfer, update ]
```

Edit template to reflect naming standard for zone files (file: %s since we're using the zone name as file name)
```
template:
  - id: default
    storage: "/var/lib/knot"
    file: "%s"
```

Add a DNSSEC policy statement
```
policy:
  - id: ecdsa
    algorithm: ecdsap256sha256
    ksk-size: 256
    zsk-size: 256
```

Note on policy statement:
* 256 is standard size for ecdsap256sha256, but is included here for clarity
* Knot will automatically generate keys to match the policy statement
* Keys are kept in the Knot database
* By default, Knot will also generate and publish CDS and CDNSKEY records


Add zone statement
```
zone:
  - domain: multisigner.examples.nu
    zonefile-sync: 0
    zonefile-load: whole
    journal-content: none
    acl: aclupdate
    dnssec-signing: on
    dnssec-policy: ecdsa
```

Notes on zone statement:
* Parameters set with testing in mind
* No journal file (journal-content: none)
* Full zone file always loadded from disc (zonefile-load: whole)
* All changes made to zone through knot cli written to zone without delay 

#### Check Knot configuration and zone file

```bash
sudo knotc conf-check
```
Expect:
```
Configuration is valid
```

```bash
sudo knotc zone-check multisigner.examples.nu
```
Expect no/empty output

#### Restart Knot

```bash
sudo service knot restart
```

Note: Restart of service required, since Knot need to rebind the listening IP/port. A reload will not suffice.

#### Verify that the Keys (and CDS/CDNSKEY) is in the zone
```bash
 dig @localhost multisigner.examples.nu DNSKEY +multi
```
``` bash
;; ANSWER SECTION:
multisigner.examples.nu. 120 IN	DNSKEY 256 3 13 (
				q2fN0krBXnxYBOp54830DqcmHK2+yLCfz2Qsd/EbMJh2
				wSnP4OKpzhqcHIrcQxpqMkBLXdMh9B4+5XiuhsZjAg==
				) ; ZSK; alg = ECDSAP256SHA256 ; key id = 53501
multisigner.examples.nu. 120 IN	DNSKEY 257 3 13 (
				ZH9ze60H9nPvK5ftDKj16+4NcBhfGdvocZiiiamMt0Qu
				pZX4x6tazEXXuortj63Da0RQotWa31LCn8P3cGZglw==
				) ; KSK; alg = ECDSAP256SHA256 ; key id = 25219
```
```bash
 dig @localhost multisigner.examples.nu CDS
```
```bash
;; ANSWER SECTION:
multisigner.examples.nu. 0	IN	CDS	25219 13 2 ED909461B183C478332EDB0D22FFA535FCF6A0995A9EE88D4C117CAC 90A96BC7
```
```bash
 dig @localhost multisigner.examples.nu CDNSKEY
```
```bash
;; ANSWER SECTION:
multisigner.examples.nu. 0	IN	CDNSKEY	257 3 13 ZH9ze60H9nPvK5ftDKj16+4NcBhfGdvocZiiiamMt0QupZX4x6tazEXX uortj63Da0RQotWa31LCn8P3cGZglw==
```


### Set up a second master server (Optional)

Alternatives:
* Repeat process above to set up another Knot server.
* Set up a BIND server [see guide](multi-signer_test_set_up_on_BIND.md)
* Set up a PowerDNS server [see guide](dnssec_on_pdns.md)

Note: Adjust names for NS records accordingly
 


## Adding the additional master server to the group

### Cross import keys

#### Notes
* Both master servers need to have identical CDS and CDNSKEY records
* While only the foreign ZSK needs to be published in the DNSKEY record set, importing the foreign KSK as well is recommended.
* Chain of trust will be intact, even if you don't att the foreign KSK to the DNSKEY set, but zone checking tools may give a warning or error.
* Same DNSKEY algorithms expected unless `unsafe-operation: no-check-keyset` is configured in the policy.

#### Get DNSKEYs from other master

Obtain the relevant public keys from the other master(s) in the multi signer group. For Knot to be able to read them, they must be saved in BIND-like public file format, with the .key extension.

```bash
dig @ns1.multisigner.examples.nu multisigner.examples.nu DNSKEY +multi
```
```
multisigner.examples.nu. 120 IN	DNSKEY 256 3 13 (
				ca4SqKRUR0GWRy/C8lChaFWtYB+zO3nX+byozlS1fxDs
				qVHRSYImDzR6ZkgOihJMWRQ3pSrYeubtIuOstSw4hw==
				) ; ZSK; alg = ECDSAP256SHA256 ; key id = 34191
multisigner.examples.nu. 120 IN	DNSKEY 257 3 13 (
				NvvCwBO9w8aCW2N884uA1VhJlSkSMvXf4jsfDiIgV2gu
				25LqIL2KyitKwyH/rEAEiR5Po3MpGVvvW744fnhIhw==
				) ; KSK; alg = ECDSAP256SHA256 ; key id = 5412
```

If there are multiple sets of keys in the key set (if joining a multi signer group with 2+ members, or due to a key rollover), use a part of the key as a unique identifier in order to separate the keys.


```bash
cd /var/lib/knot/
dig @ns1.multisigner.examples.nu multisigner.examples.nu DNSKEY | egrep 'rEAEiR5Po3MpGVvvW744fnhIhw==' > /tmp/ksk_5412.key
dig @ns1.multisigner.examples.nu multisigner.examples.nu DNSKEY | egrep 'ihJMWRQ3pSrYeubtIuOstSw4hw==' > /tmp/zsk_34191.key
```

#### Import ZSK and (prepare to) publish in DNSKEY set
```bash
sudo keymgr multisigner.examples.nu import-pub /tmp/zsk_34191.key
```

#### Import KSK and (prepare to) publish in DNSKEY set
```bash
sudo keymgr multisigner.examples.nu import-pub /tmp/ksk_5412.key
```
```
314a370c7643ad4423ec5bf66b18a143ae605f83
OK
```

#### Adjust foreign KSK to not published but with corresponding CDS and CDNSKEY records published
```bash
sudo keymgr multisigner.examples.nu 314a370c7643ad4423ec5bf66b18a143ae605f83 publish=0 ready=+0
```

#### Check the key set
```bash
sudo keymgr multisigner.examples.nu -b list
```
```
937ef367443f3014dd001eba35fd48646069422a 25219 KSK ECDSAP256SHA256 publish=1632150545 ready=1632150545
94a6a0eb68a7043a58957b34bfe3bf220cd39799 53501 ZSK ECDSAP256SHA256 publish=1632150545 active=1632150545
2b53986bd472eb17a6e3f1f4f9fe3125b52ae463 34191 ZSK ECDSAP256SHA256 public-only publish=1632150984
314a370c7643ad4423ec5bf66b18a143ae605f83  5412 KSK ECDSAP256SHA256 public-only ready=1632151241
```

#### Reload the key set and update RRSIGs
```bash
sudo knotc zone-keys-load multisigner.examples.nu.
```

#### Check server for DNSKEY, CDS and CDNSKEY records


```bash
dig @localhost multisigner.examples.nu axfr | egrep 'IN\s+(CDS|[C]?DNSKEY)'
```

```
multisigner.examples.nu. 120	IN	DNSKEY	256 3 13 ca4SqKRUR0GWRy/C8lChaFWtYB+zO3nX+byozlS1fxDsqVHRSYImDzR6 ZkgOihJMWRQ3pSrYeubtIuOstSw4hw==
multisigner.examples.nu. 120	IN	DNSKEY	256 3 13 q2fN0krBXnxYBOp54830DqcmHK2+yLCfz2Qsd/EbMJh2wSnP4OKpzhqc HIrcQxpqMkBLXdMh9B4+5XiuhsZjAg==
multisigner.examples.nu. 120	IN	DNSKEY	257 3 13 ZH9ze60H9nPvK5ftDKj16+4NcBhfGdvocZiiiamMt0QupZX4x6tazEXX uortj63Da0RQotWa31LCn8P3cGZglw==
multisigner.examples.nu. 0	IN	CDS	5412 13 2 9DA5358E29E521BC72AA75CC54C7C863C5DA8BE2F1018566717EAF86 09FBE346
multisigner.examples.nu. 0	IN	CDS	25219 13 2 ED909461B183C478332EDB0D22FFA535FCF6A0995A9EE88D4C117CAC 90A96BC7
multisigner.examples.nu. 0	IN	CDNSKEY	257 3 13 NvvCwBO9w8aCW2N884uA1VhJlSkSMvXf4jsfDiIgV2gu25LqIL2KyitK wyH/rEAEiR5Po3MpGVvvW744fnhIhw==
multisigner.examples.nu. 0	IN	CDNSKEY	257 3 13 ZH9ze60H9nPvK5ftDKj16+4NcBhfGdvocZiiiamMt0QupZX4x6tazEXX uortj63Da0RQotWa31LCn8P3cGZglw==
```

#### Update NS record set and add CSYNC record (if supported by parent)

```
$ORIGIN multisigner.examples.nu.
$TTL 120
@       SOA     ns1.multisigner.examples.nu. dns.examples.nu. 1618586097 14400 3600 1814400 120

@		IN		CSYNC	0 1 A NS AAAA

@       NS      ns1
@       NS      ns3
ns1     A       13.51.70.181
ns3     A       13.51.108.122
```

```bash
sudo knotc zone-reload multisigner.examples.nu
```


#### Check that the parent nameservers have picked up and published the correct DS records

```bash
dig @ns1.examples.nu multisigner.examples.nu DS
```
```
multisigner.examples.nu. 120	IN	DS	5412 13 2 9DA5358E29E521BC72AA75CC54C7C863C5DA8BE2F1018566717EAF86 09FBE346
multisigner.examples.nu. 120	IN	DS	41461 13 2 338571BB87C733A237CE40609104F0423F9F15CDE88B57BC897D80E1 1CF58FD3
```

```bash
dig @ns2.examples.nu multisigner.examples.nu DS
```
```
multisigner.examples.nu. 120	IN	DS	5412 13 2 9DA5358E29E521BC72AA75CC54C7C863C5DA8BE2F1018566717EAF86 09FBE346
multisigner.examples.nu. 120	IN	DS	41461 13 2 338571BB87C733A237CE40609104F0423F9F15CDE88B57BC897D80E1 1CF58FD3
```

#### Check that the parent nameservers have picked up and published the correct DS records

```bash
dig @ns1.examples.nu multisigner.examples.nu NS
```
```
multisigner.examples.nu. 120	IN	NS	ns1.multisigner.examples.nu.
multisigner.examples.nu. 120	IN	NS	ns3.multisigner.examples.nu.
```

```bash
dig @ns2.examples.nu multisigner.examples.nu NS
```
```
multisigner.examples.nu. 120	IN	NS	ns3.multisigner.examples.nu.
multisigner.examples.nu. 120	IN	NS	ns1.multisigner.examples.nu.
```

### Cleanup

Note: Wait 2 x maximum TTL of DS at parent and DNSKEY at all children before proceding with this step.

#### Remove the CSYNC record

```
$ORIGIN multisigner.examples.nu.
$TTL 120
@       SOA     ns1.multisigner.examples.nu. dns.examples.nu. 1618586094 14400 3600 1814400 120

@       NS      ns1
@       NS      ns2
ns1     A       13.51.70.181
ns2     A       13.53.109.125
```

```bash
sudo knotc zone-reload multisigner.examples.nu
```

 