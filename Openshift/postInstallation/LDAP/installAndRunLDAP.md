## Key points to remember
* LDAP runs on port 389 and 636 for unencrypted and TLS based connections repsectively.
* Generate the hashed password as root

## Installation steps
* Get a RHEL based VM

```sh
hostnamectl
   Static hostname: node-0.rajranjabastion.lab.pnq2.cee.redhat.com
         Icon name: computer-vm
           Chassis: vm
        Machine ID: --------------
           Boot ID: --------------
    Virtualization: kvm
  Operating System: --------------
       CPE OS Name: cpe:/o:redhat:enterprise_linux:7.6:GA:server
            Kernel: Linux 3.10.0-957.1.3.el7.x86_64
      Architecture: x86-64
```
* DNS is configured for the choosen host. Make sure LDAP server is reachable from the client servers.

```sh
nslookup node-0.rajranjabastion.lab.pnq2.cee.redhat.com
Server:		10.75.5.25
Address:	10.75.5.25#53

Non-authoritative answer:
Name:	node-0.rajranjabastion.lab.pnq2.cee.redhat.com
Address: 10.74.177.84
```
* Install package

```sh
yum -y install openldap compat-openldap openldap-clients openldap-servers openldap-servers-sql openldap-devel
```
* Start the LDAP server

```sh
systemctl start slapd
systemctl enable slapd
netstat -antup | grep -i 389
```

* Generate the hashed password for ldap admin user
```sh
slappasswd -h {SSHA} -s ldapadmin

{SSHA}****HASHEDPASSWORD****
```
* Create a db.ldif file to edit the domain server, root user and password. See the content of the [db.ldif](db.ldif) for more detail.

* Update the LDAP server with configs in db.ldif

```sh
ldapmodify -Y EXTERNAL  -H ldapi:/// -f db.ldif
```
* Restrict the monitor access only to ldap root (ldapadm) user not to others. Create a [monitor.ldif](monitor.ldif) and update the changes to the LDAP server.

```sh
ldapmodify -Y EXTERNAL  -H ldapi:/// -f monitor.ldif
```
* To config ldap database; use a sample config file and load the LDAP schemas.

```sh
cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
chown ldap:ldap /var/lib/ldap/*
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif 
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```

* Create the base org structure for users to be stored. See [base.ldif](base.ldif).
```sh
ldapadd -x -W -D "cn=ldapadmin,dc=rajranjabastion,dc=lab,dc=pnq2,dc=cee,dc=redhat,dc=com" -f base.ldif
```

* create your first user. Review the file [rajiv.ldif](rajiv.ldif)
```sh
ldapadd -x -W -D "cn=ldapadmin,dc=rajranjabastion,dc=lab,dc=pnq2,dc=cee,dc=redhat,dc=com" -f rajiv.ldif
```


## Setup logging in LDAP
* check whether logging is active. In a new installation it will not be.

```sh
ldapsearch -Q -LLL -Y EXTERNAL -H ldapi:/// -b cn=config olcLogLevel
```
Output
```sh
dn: cn=config

dn: cn=schema,cn=config

dn: cn={0}core,cn=schema,cn=config

dn: cn={1}cosine,cn=schema,cn=config

dn: cn={2}nis,cn=schema,cn=config

dn: cn={3}inetorgperson,cn=schema,cn=config

dn: olcDatabase={-1}frontend,cn=config

dn: olcDatabase={0}config,cn=config

dn: olcDatabase={1}monitor,cn=config

dn: olcDatabase={2}hdb,cn=config
```
* change the olcLogLevel to level - ***any*** or ***stats*** based on requirement

```sh
ldapmodify -Q  -Y EXTERNAL -H ldapi:/// -f olcLogLevel-any.ldif
```
Output
```sh
modifying entry "cn=config"
```
```sh
ldapmodify -Q  -Y EXTERNAL -H ldapi:/// -f olcLogLevel-stats.ldif
modifying entry "cn=config"
```

 * Verify by looking at journalctl for slapd
```sh
ldapsearch -Q -LLL -Y EXTERNAL -H ldapi:/// -b cn=config olcLogLevel
```
Output
```sh
dn: cn=config
olcLogLevel: any

dn: cn=schema,cn=config

dn: cn={0}core,cn=schema,cn=config

dn: cn={1}cosine,cn=schema,cn=config

dn: cn={2}nis,cn=schema,cn=config

dn: cn={3}inetorgperson,cn=schema,cn=config

dn: olcDatabase={-1}frontend,cn=config

dn: olcDatabase={0}config,cn=config

dn: olcDatabase={1}monitor,cn=config

dn: olcDatabase={2}hdb,cn=config
```
```sh
journalctl -u slapd -n 0 -f
```

Useful link for further read.

|Topic|Link|
|----|----|
|OPENLDAP|[Click Here](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system-level_authentication_guide/openldap#s2-ldap-configuration)|
|How to Configure OpenLDAP server in Red Hat Enterprise Linux 7 using cn=config method?|[Click Here](https://access.redhat.com/solutions/2484371)|
|||
