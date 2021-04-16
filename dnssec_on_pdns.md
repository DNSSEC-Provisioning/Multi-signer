# One way to set up a DNESEC enabled example zone on PowerDNS.

* Ubuntu 20.04
* [PowerDNS Master Branch](https://repo.powerdns.com/)
* [PowerDNS Docs](https://doc.powerdns.com/authoritative/index.html)





## Power DNS install and set up on Ubuntu
* Add Repo

```bash
echo $'deb [arch=amd64] http://repo.powerdns.com/ubuntu focal-auth-master main' | sudo tee /etc/apt/sources.list.d/pdns.list
echo $'Package: pdns-* \nPin: origin repo.powerdns.com \nPin-Priority: 600' | sudo tee /etc/apt/preferences.d/pdns
```

#### Add apt-key for Repo ( your responsibility to check the key )
```bash
curl https://repo.powerdns.com/CBC8B383-pub.asc | sudo apt-key add -
sudo apt-get update
```

#### Install PowerDNS with mariaDB backend
```bash
sudo apt install pdns-server pdns-tools
sudo apt install mariadb-server pdns-backend-mysql
```

#### Turn on and configure mariaDB
```bash
sudo systemctl enable mariadb
sudo systemctl start mariadb
```
#### Secure your mariaDB installation ( Details of this is out of scope for this document, but fairly self explanatory)
* Set root password
* Remove anonymous users
* Disallow root login remotely
* Remove test database and access to it
```bash
sudo mysql_secure_installation
```

#### copy the SQL into a setup_pdns.sql file and run it on the database
``` bash
mysql -u root -p < setup_pdns.sql
```



```sql
CREATE USER 'pdns'@'localhost' IDENTIFIED BY 'REALLYGREATPASSWORD' ;
GRANT DELETE, INSERT, SELECT, UPDATE ON pdns.* TO 'pdns'@'localhost';
FLUSH PRIVILEGES;
CREATE DATABASE pdns;
USE pdns;


CREATE TABLE domains (
id INT AUTO_INCREMENT,
name VARCHAR(255) NOT NULL,
master VARCHAR(128) DEFAULT NULL,
last_check INT DEFAULT NULL,
type VARCHAR(6) NOT NULL,
notified_serial INT UNSIGNED DEFAULT NULL,
account VARCHAR(40) CHARACTER SET 'utf8' DEFAULT NULL,
PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';


CREATE UNIQUE INDEX name_index ON domains(name);


CREATE TABLE records (
id BIGINT AUTO_INCREMENT,
domain_id INT DEFAULT NULL,
name VARCHAR(255) DEFAULT NULL,
type VARCHAR(10) DEFAULT NULL,
content VARCHAR(64000) DEFAULT NULL,
ttl INT DEFAULT NULL,
prio INT DEFAULT NULL,
disabled TINYINT(1) DEFAULT 0,
ordername VARCHAR(255) BINARY DEFAULT NULL,
auth TINYINT(1) DEFAULT 1,
PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';

CREATE INDEX nametype_index ON records(name,type);

CREATE INDEX domain_id ON records(domain_id);

CREATE INDEX ordername ON records (ordername);


CREATE TABLE supermasters (
ip VARCHAR(64) NOT NULL,
nameserver VARCHAR(255) NOT NULL,
account VARCHAR(40) CHARACTER SET 'utf8' NOT NULL,
PRIMARY KEY (ip, nameserver)
) Engine=InnoDB CHARACTER SET 'latin1';


CREATE TABLE comments (
id INT AUTO_INCREMENT,
domain_id INT NOT NULL,
name VARCHAR(255) NOT NULL,
type VARCHAR(10) NOT NULL,
modified_at INT NOT NULL,
account VARCHAR(40) CHARACTER SET 'utf8' DEFAULT NULL,
comment TEXT CHARACTER SET 'utf8' NOT NULL,
PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';

CREATE INDEX comments_name_type_idx ON comments (name, type);

CREATE INDEX comments_order_idx ON comments (domain_id, modified_at);


CREATE TABLE domainmetadata (
id INT AUTO_INCREMENT,
domain_id INT NOT NULL,
kind VARCHAR(32),
content TEXT,
PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';


CREATE INDEX domainmetadata_idx ON domainmetadata (domain_id, kind);


CREATE TABLE cryptokeys (
id INT AUTO_INCREMENT,
domain_id INT NOT NULL,
flags INT NOT NULL,
active BOOL,
published BOOL DEFAULT 1,
content TEXT,
PRIMARY KEY(id)
) Engine=InnoDB CHARACTER SET 'latin1';


CREATE INDEX domainidindex ON cryptokeys(domain_id);


CREATE TABLE tsigkeys (
id INT AUTO_INCREMENT,
name VARCHAR(255),
algorithm VARCHAR(50),
secret VARCHAR(255),
PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';


CREATE UNIQUE INDEX namealgoindex ON tsigkeys(name, algorithm);
```



#### Configure PowerDNS by changing the following defaults, you may have to adjust these for your particular needs, this turns on DNSSEC, Webserver and Web-API, and allows DNSSEC foreign keys

```
# api=no
api=yes

# api-key=
api-key=SET-A-REALLY-GOOD-API-KEY

# direct-dnskey=no
direct-dnskey=yes

# include-dir=
include-dir=/etc/powerdns/pdns.d

# launch=
launch=gmysql
gmysql-host=127.0.0.1
gmysql-port=3306
gmysql-user=pdns
gmysql-password=REALLYGREATPASSWORD
gmysql-dbname=pdns
gmysql-dnssec=yes

# local-address=0.0.0.0, ::
local-address=127.0.0.1

# server-id=
server-id=ns1.example.se

# tcp-control-address=
tcp-control-address=127.0.0.1

# webserver=no
webserver=yes

# webserver-address=127.0.0.1
webserver-address=172.31.47.63

# webserver-allow-from=127.0.0.1,::1
webserver-allow-from=0.0.0.0/0,127.0.0.1,::1
webserver-password=AGOODPASSWORDFORHAPPYCISO
```

#### Turn on PowerDNS and Start it
```bash
sudo systemctl enable pdns.service
sudo systemctl start pdns.service
```


### Your PowerDNS name server is now working!
#### You can test it with: 
```bash
$ dig @localhost +nsid
```
You will get a "REFUSED" answer but your NSID will show "ns1.example.se"


### Set up a test zone example.se
#### Create the zone ( it will be empty )
```bash
sudo pdnsutil create-zone example.se
```
#### Add a base config and records to the zone using the information below
```bash
sudo pdnsutil edit-zone example.se
```


```bash
$ORIGIN .
example.se. 300 IN SOA ns1.example.se. hostmaster.example.se 1 10800 3600 604800 3600
example.se. 300 IN NS ns1.example.se.
ns1.example.se. 300 IN A 127.0.0.1
```

#### Now you can enable DNSSEC and sign your zone

```bash
sudo pdnsutil add-zone-key example.se KSK inactive ecdsa256
sudo pdnsutil add-zone-key example.se ZSK inactive ecdsa256
sudo pdnsutil list-keys example.se
sudo pdnsutil set-publish-cdnskey example.se
sudo pdnsutil set-publish-cds example.se
sudo pdnsutil activate-zone-key example.se 1
sudo pdnsutil activate-zone-key example.se 2
```

#### Check the zone and list DNSKEY and DS for the zone
```bash
sudo pdnsutil check-zone example.se
sudo pdnsutil show-zone example.se
sudo pdnsutil export-zone-dnskey example.se 1
sudo pdnsutil export-zone-ds example.se
```



#### Test the zone locally
```bash
dig @localhost example.se dnskey +multi +dnssec

; <<>> DiG 9.16.1-Ubuntu <<>> @localhost example.se dnskey +multi +dnssec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 39495
;; flags: qr aa rd; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags: do; udp: 1232
;; QUESTION SECTION:
;example.se. IN DNSKEY

;; ANSWER SECTION:
example.se. 3600 IN DNSKEY 257 3 13 (
f9M6q4pm145aqYTVz9AobQZDWvJzh5Ebim/r+JQrmOqz
LGQrbUeucFwCyBYsf/dIpcRNBYazItAFOn0BZ1ewPQ==
) ; KSK; alg = ECDSAP256SHA256 ; key id = 45733
example.se. 3600 IN DNSKEY 256 3 13 (
YlG7g5H3nY4n7Mgcqkem0kdHVkBL9dFb/6kF3YzOhAXx
AUb4sbcSJCpkCQW6PKkL5gB5r94RkVdfNK02b3EllA==
) ; ZSK; alg = ECDSAP256SHA256 ; key id = 26321
example.se. 3600 IN RRSIG DNSKEY 13 2 3600 (
20210318000000 20210225000000 45733 example.se.
Yv1f0Y+OwWu7y6mWkyN1ZQ1yw7FMyREhdKiJpI8xR8Ov
XKQEpwqlsFQZRY1rByS82YTYi4hgiIqCS1Jxw1Hxhw== )

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Wed Mar 10 14:59:07 UTC 2021
;; MSG SIZE rcvd: 305
```


#### To add external DNSKEY Public Part via pdnsutil
```bash
pdnsutil add-record example.se . DNSKEY 3600 "256 3 13 Uwz80gx2CoUFmdjq3YpEpN4FbJCQvHknKqfGYGcETNju++fbqVw6rfV6 H1MF1dAFDZDg+rFbJjtYFlViYn9VFg=="
```
