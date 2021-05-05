# Testing Multi-signer set up using BIND.

* Ubuntu 20.04
* BIND 9.16.1-Ubuntu (Stable Release) <id:d497c32>

#### Notes
* Third level domain (multisigner.examples.nu) used for easy management of DS records
* Provisioned om micro EC2 in AWS
* This version uses two nameservers, both running BIND


## BIND install and set up on Ubuntu

#### Install BIND9
```bash
sudo apt install bind9
```

#### Create zone file
```bash
sudo vi /var/cache/bind/multisigner.examples.nu
```
```
	$ORIGIN multisigner.examples.nu.
	$TTL 120
	@       SOA     ns1.multisigner.examples.nu. dns.examples.nu. 1618586094 14400 3600 1814400 120

	@       NS      ns1
	ns1     A       13.51.70.181
```

#### Add configuration
```bash
sudo vi /etc/bind/named.zones.local
```
```
	zone "multisigner.examples.nu" {
	    type master;
        key-directory "/var/cache//bind/keys";
        auto-dnssec maintain;
        inline-signing yes;
	};
```

#### Generate keys
Keep the keys in a separate directory under /var/cache/bind/ to avoid problems with BIND or Apparmor
```bash
sudo mkdir /var/cache/bind/keys
cd /var/cache/bind/keys
```
```bash
sudo dnssec-keygen -a ECDSAP256SHA256 multisigner.examples.nu
sudo dnssec-keygen -a ECDSAP256SHA256 -f KSK multisigner.examples.nu
```
BIND needs read access on the keys and directory...
```bash
sudo chown -R bind /var/cache/bind/keys
```
```bash
ls -l /var/cache/bind/keys/K*
```
```
-rw-r--r-- 1 bind root 477 Apr 28 13:48 Kmultisigner.examples.nu.+013+05412.key
-rw------- 1 bind root 242 Apr 28 13:48 Kmultisigner.examples.nu.+013+05412.private
-rw-r--r-- 1 bind root 366 Apr 19 10:35 Kmultisigner.examples.nu.+013+34191.key
-rw------- 1 bind root 187 Apr 19 10:35 Kmultisigner.examples.nu.+013+34191.private
```

To publish CDS and CDNSKEY records:
```bash
sudo dnssec-settime -P sync -1h Kmultisigner.examples.nu.+013+05412.private
```

#### Restart BIND
```bash
sudo service bind9 restart
```

#### Verify that the Keys (and CDS/CDNSKEY) is in the zone
```bash
 dig @localhost multisigner.examples.nu DNSKEY +multi
```
``` bash
;; ANSWER SECTION:
multisigner.examples.nu. 120 IN	DNSKEY 257 3 13 (
				NvvCwBO9w8aCW2N884uA1VhJlSkSMvXf4jsfDiIgV2gu
				25LqIL2KyitKwyH/rEAEiR5Po3MpGVvvW744fnhIhw==
				) ; KSK; alg = ECDSAP256SHA256 ; key id = 5412
multisigner.examples.nu. 120 IN	DNSKEY 256 3 13 (
				ca4SqKRUR0GWRy/C8lChaFWtYB+zO3nX+byozlS1fxDs
				qVHRSYImDzR6ZkgOihJMWRQ3pSrYeubtIuOstSw4hw==
				) ; ZSK; alg = ECDSAP256SHA256 ; key id = 34191
```
```bash
 dig @localhost multisigner.examples.nu CDS
```
```bash
;; ANSWER SECTION:
multisigner.examples.nu. 120	IN	CDS	5412 13 2 9DA5358E29E521BC72AA75CC54C7C863C5DA8BE2F1018566717EAF86 09FBE346
```
```bash
 dig @localhost multisigner.examples.nu CDNSKEY
```
```bash
;; ANSWER SECTION:
multisigner.examples.nu. 120	IN	CDNSKEY	257 3 13 NvvCwBO9w8aCW2N884uA1VhJlSkSMvXf4jsfDiIgV2gu25LqIL2KyitK wyH/rEAEiR5Po3MpGVvvW744fnhIhw==
```


### Repeat process for second master server

#### Notes
Zone
```
	$ORIGIN multisigner.examples.nu.
	$TTL 120
	@       SOA     ns2.multisigner.examples.nu. dns.examples.nu. 1618586094 14400 3600 1814400 120

	@       NS      ns2
	ns2     A       13.53.109.125
```
Keys
```bash
ls -l /var/cache/bind/keys/K*
```
```
-rw-r--r-- 1 bind root 366 Apr 19 10:11 Kmultisigner.examples.nu.+013+19251.key
-rw------- 1 bind root 187 Apr 19 10:11 Kmultisigner.examples.nu.+013+19251.private
-rw-r--r-- 1 bind root 478 Apr 28 12:46 Kmultisigner.examples.nu.+013+40598.key
-rw------- 1 bind root 242 Apr 28 12:46 Kmultisigner.examples.nu.+013+40598.private
```


## Adding the second master server to the group


#### Cross import keys

#### Notes
* Both master servers need to have identical CDS and CDNSKEY records
* While only the foreign ZSK needs to be published in the DNSKEY record set, BIND needs the foreign KSK as well, in order to add it to the published CDS and CDNSKEY record set.
* Chain of trust will be intact, even if you don't att the foreign KSK to the DNSKEY set, but zone checking tools may give a warning or error.

#### Fetch DNSKEYs from second master

```bash
cd /var/cache/bind/keys
dig @ns2.multisigner.examples.nu multisigner.examples.nu DNSKEY | egrep 'DNSKEY\s+257' > ext_ksk.txt
dig @ns2.multisigner.examples.nu multisigner.examples.nu DNSKEY | egrep 'DNSKEY\s+256' > ext_zsk.txt
```

#### Import ZSK and (prepare to) publish in DNSKEY set
```bash
sudo dnssec-importkey -f /var/cache/bind/keys/ext_zsk.txt -P -1h multisigner.examples.nu
```

#### Import KSK and (prepare to) publish in CDS and CDNSKEY set
```bash
sudo dnssec-importkey -f /var/cache/bind/keys/ext_ksk.txt -P sync -1h multisigner.examples.nu
```

#### (Optional) Add the foreign KSK to the DNSKEY set
```bash
sudo dnssec-settime -P -1h /var/cache/bind/keys/Kmultisigner.examples.nu.+013+40598.private
```

```bash
ls -l /var/cache/bind/keys/K*
```
```
-rw-r--r-- 1 bind root 370 Apr 28 12:45 Kmultisigner.examples.nu.+013+05412.key
-rw------- 1 bind root 146 Apr 28 12:45 Kmultisigner.examples.nu.+013+05412.private
-rw-r--r-- 1 bind root 366 Apr 19 10:11 Kmultisigner.examples.nu.+013+19251.key
-rw------- 1 bind root 187 Apr 19 10:11 Kmultisigner.examples.nu.+013+19251.private
-rw-r--r-- 1 bind root 259 Apr 19 12:18 Kmultisigner.examples.nu.+013+34191.key
-rw------- 1 bind root  91 Apr 19 12:18 Kmultisigner.examples.nu.+013+34191.private
-rw-r--r-- 1 bind root 478 Apr 28 12:46 Kmultisigner.examples.nu.+013+40598.key
-rw------- 1 bind root 242 Apr 28 12:46 Kmultisigner.examples.nu.+013+40598.private
```


#### Load keys into BIND
Make sure BIND has read access on the keys and directory...
```bash
sudo chown -R bind /var/cache/bind/keys
```
```bash
sudo rndc loadkeys multisigner.examples.nu
```


### Repeat Fetch/Import/Load steps for second master



#### Reload BIND on both servers
Note: Introducing the foreign DNSKEY records on one server at a time works fine, but fetching the right DNSKEYs to import requires more effort.

#### Check both servers for DNSKEY, CDS and CDNSKEY records

Note: The imported foreign KSK will appear only if you have opted to publish it (like below).

```bash
dig @ns1.multisigner.examples.nu multisigner.examples.nu axfr | egrep 'IN\s+(CDS|[C]?DNSKEY)'
```
```
multisigner.examples.nu. 120	IN	CDS	5412 13 2 9DA5358E29E521BC72AA75CC54C7C863C5DA8BE2F1018566717EAF86 09FBE346
multisigner.examples.nu. 120	IN	CDS	40598 13 2 B337F82CB34D453F8D9309F57367B07F0E8FA58EF947B2924A50E5AB B21285CF
multisigner.examples.nu. 120	IN	CDNSKEY	257 3 13 NvvCwBO9w8aCW2N884uA1VhJlSkSMvXf4jsfDiIgV2gu25LqIL2KyitK wyH/rEAEiR5Po3MpGVvvW744fnhIhw==
multisigner.examples.nu. 120	IN	CDNSKEY	257 3 13 jNDEQ5zVp6tYqqtC6hujGPzyVbnQ082zRur71xY7oHz5o7HMCZ9tWg5n bjo8WN0YRTAqRlsBr0ZS1pxjn3XIOA==
multisigner.examples.nu. 120	IN	DNSKEY	256 3 13 PncpJ/Xoyo8D7CNJl/K+l2HLROiWwdItFbdMu+D+wPoTMlFz5kh4h8IF TLcJp6MyixKvByX884IZ8eJlFI2ptg==
multisigner.examples.nu. 120	IN	DNSKEY	256 3 13 ca4SqKRUR0GWRy/C8lChaFWtYB+zO3nX+byozlS1fxDsqVHRSYImDzR6 ZkgOihJMWRQ3pSrYeubtIuOstSw4hw==
multisigner.examples.nu. 120	IN	DNSKEY	257 3 13 NvvCwBO9w8aCW2N884uA1VhJlSkSMvXf4jsfDiIgV2gu25LqIL2KyitK wyH/rEAEiR5Po3MpGVvvW744fnhIhw==
multisigner.examples.nu. 120	IN	DNSKEY	257 3 13 jNDEQ5zVp6tYqqtC6hujGPzyVbnQ082zRur71xY7oHz5o7HMCZ9tWg5n bjo8WN0YRTAqRlsBr0ZS1pxjn3XIOA==
```

```bash
dig @ns2.multisigner.examples.nu multisigner.examples.nu axfr | egrep 'IN\s+(CDS|[C]?DNSKEY)'
```
```
multisigner.examples.nu. 120	IN	CDS	5412 13 2 9DA5358E29E521BC72AA75CC54C7C863C5DA8BE2F1018566717EAF86 09FBE346
multisigner.examples.nu. 120	IN	CDS	40598 13 2 B337F82CB34D453F8D9309F57367B07F0E8FA58EF947B2924A50E5AB B21285CF
multisigner.examples.nu. 120	IN	CDNSKEY	257 3 13 NvvCwBO9w8aCW2N884uA1VhJlSkSMvXf4jsfDiIgV2gu25LqIL2KyitK wyH/rEAEiR5Po3MpGVvvW744fnhIhw==
multisigner.examples.nu. 120	IN	CDNSKEY	257 3 13 jNDEQ5zVp6tYqqtC6hujGPzyVbnQ082zRur71xY7oHz5o7HMCZ9tWg5n bjo8WN0YRTAqRlsBr0ZS1pxjn3XIOA==
multisigner.examples.nu. 120	IN	DNSKEY	256 3 13 PncpJ/Xoyo8D7CNJl/K+l2HLROiWwdItFbdMu+D+wPoTMlFz5kh4h8IF TLcJp6MyixKvByX884IZ8eJlFI2ptg==
multisigner.examples.nu. 120	IN	DNSKEY	256 3 13 ca4SqKRUR0GWRy/C8lChaFWtYB+zO3nX+byozlS1fxDsqVHRSYImDzR6 ZkgOihJMWRQ3pSrYeubtIuOstSw4hw==
multisigner.examples.nu. 120	IN	DNSKEY	257 3 13 jNDEQ5zVp6tYqqtC6hujGPzyVbnQ082zRur71xY7oHz5o7HMCZ9tWg5n bjo8WN0YRTAqRlsBr0ZS1pxjn3XIOA==
multisigner.examples.nu. 120	IN	DNSKEY	257 3 13 NvvCwBO9w8aCW2N884uA1VhJlSkSMvXf4jsfDiIgV2gu25LqIL2KyitK wyH/rEAEiR5Po3MpGVvvW744fnhIhw==
```

#### Update the NS record set for both master servers
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
sudo rndc reload
```

#### Update NS records at parent through CSYNC (if supported) on both master servers

```
	$ORIGIN multisigner.examples.nu.
	$TTL 120
	@       SOA     ns1.multisigner.examples.nu. dns.examples.nu. 1618586094 14400 3600 1814400 120

	@		IN		CSYNC	0 1 A NS AAAA

	@       NS      ns1
	@       NS      ns2
	ns1     A       13.51.70.181
	ns2     A       13.53.109.125
```


#### Check that the parent nameservers have picked up and published the correct DS records

```bash
dig @ns1.examples.nu multisigner.examples.nu DS
```
```
multisigner.examples.nu. 120	IN	DS	5412 13 2 9DA5358E29E521BC72AA75CC54C7C863C5DA8BE2F1018566717EAF86 09FBE346
multisigner.examples.nu. 120	IN	DS	40598 13 2 B337F82CB34D453F8D9309F57367B07F0E8FA58EF947B2924A50E5AB B21285CF
```

```bash
dig @ns2.examples.nu multisigner.examples.nu DS
```
```
multisigner.examples.nu. 120	IN	DS	5412 13 2 9DA5358E29E521BC72AA75CC54C7C863C5DA8BE2F1018566717EAF86 09FBE346
multisigner.examples.nu. 120	IN	DS	40598 13 2 B337F82CB34D453F8D9309F57367B07F0E8FA58EF947B2924A50E5AB B21285CF
```

#### Check that the parent nameservers have picked up and published the correct DS records

```bash
dig @ns1.examples.nu multisigner.examples.nu NS
```
```
multisigner.examples.nu. 120	IN	NS	ns1.multisigner.examples.nu.
multisigner.examples.nu. 120	IN	NS	ns2.multisigner.examples.nu.
```

```bash
dig @ns2.examples.nu multisigner.examples.nu NS
```
```
multisigner.examples.nu. 120	IN	NS	ns2.multisigner.examples.nu.
multisigner.examples.nu. 120	IN	NS	ns1.multisigner.examples.nu.
```

### Cleanup

#### Remove the CDS/CDNSKEY records

Note: Wait 2 x maximum TTL of DS at parent and DNSKEY at all children before proceding with this step.

```bash
dnssec-settime -D sync -1h Kmultisigner.examples.nu.+013+05412.private
```
```bash
dnssec-settime -D sync -1h Kmultisigner.examples.nu.+013+40598.private
```
```bash
sudo rndc reload
```

#### Remove the CSYNC record (if present)


### Repeat process for second master server


#### Check both servers for DNSKEY, CDS and CDNSKEY records to verify that the CDS and CDNSKEY records have been removed.

```bash
dig @ns1.multisigner.examples.nu multisigner.examples.nu axfr | egrep 'IN\s+(CDS|[C]?DNSKEY)'
```
```
multisigner.examples.nu. 120	IN	DNSKEY	256 3 13 PncpJ/Xoyo8D7CNJl/K+l2HLROiWwdItFbdMu+D+wPoTMlFz5kh4h8IF TLcJp6MyixKvByX884IZ8eJlFI2ptg==
multisigner.examples.nu. 120	IN	DNSKEY	256 3 13 ca4SqKRUR0GWRy/C8lChaFWtYB+zO3nX+byozlS1fxDsqVHRSYImDzR6 ZkgOihJMWRQ3pSrYeubtIuOstSw4hw==
multisigner.examples.nu. 120	IN	DNSKEY	257 3 13 NvvCwBO9w8aCW2N884uA1VhJlSkSMvXf4jsfDiIgV2gu25LqIL2KyitK wyH/rEAEiR5Po3MpGVvvW744fnhIhw==
multisigner.examples.nu. 120	IN	DNSKEY	257 3 13 jNDEQ5zVp6tYqqtC6hujGPzyVbnQ082zRur71xY7oHz5o7HMCZ9tWg5n bjo8WN0YRTAqRlsBr0ZS1pxjn3XIOA==
```

```bash
dig @ns2.multisigner.examples.nu multisigner.examples.nu axfr | egrep 'IN\s+(CDS|[C]?DNSKEY)'
```
```
multisigner.examples.nu. 120	IN	DNSKEY	256 3 13 PncpJ/Xoyo8D7CNJl/K+l2HLROiWwdItFbdMu+D+wPoTMlFz5kh4h8IF TLcJp6MyixKvByX884IZ8eJlFI2ptg==
multisigner.examples.nu. 120	IN	DNSKEY	256 3 13 ca4SqKRUR0GWRy/C8lChaFWtYB+zO3nX+byozlS1fxDsqVHRSYImDzR6 ZkgOihJMWRQ3pSrYeubtIuOstSw4hw==
multisigner.examples.nu. 120	IN	DNSKEY	257 3 13 NvvCwBO9w8aCW2N884uA1VhJlSkSMvXf4jsfDiIgV2gu25LqIL2KyitK wyH/rEAEiR5Po3MpGVvvW744fnhIhw==
multisigner.examples.nu. 120	IN	DNSKEY	257 3 13 jNDEQ5zVp6tYqqtC6hujGPzyVbnQ082zRur71xY7oHz5o7HMCZ9tWg5n bjo8WN0YRTAqRlsBr0ZS1pxjn3XIOA==
```






