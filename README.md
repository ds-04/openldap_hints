# openldap_hints

openldap_hints - collection of commands or procedures to help managing OpenLDAP.

```diff

CURRENT TODOS:

- Writeup: slapadd doesn't respect memberOf ... slapd needs to be online for memberOf ... as its an operational-on-write-action

```

<h1>Openldap versions</h1>

As of Q4 2024:

2.5 - outgoing LTS, move away from this or prior 2x versions<br>
2.6 - LTS<br>
2.7 - features release<br>
<br>

https://ldap.com/2024/07/16/openldap-2-6-long-term-support-announcement/

<h1>openldap database types</h1>

Current:

``mdb`` - Uses OpenLDAP's Lightning Memory-Mapped Database (LMDB) library to store data. Uses no caching and requires no tuning to deliver maximum search performance.

Now deprecated:

``hdb`` - Hierarchical variant of bdb. More spatially and execution efficient than bdb.<br>
``bdb`` - BerkeleyDB.


<h1>openldap access types - listeners</h1>

Need to move/tidy this section up... but for quick reference.

ldap:/// 	LDAP 	TCP port 389<br>
ldaps:/// 	LDAP over SSL 	TCP port 636<br>
ldapi:/// 	LDAP 	IPC (Unix-domain socket)<br>

Couple of examples<br>
<br>
`` ldapadd -Y EXTERNAL -H ldapi:/// -f some_ldif.ldif``<br>
<br>
`` ldapadd -H  ldaps:///  -D cn=bind_user,dc=some,dc=example -f some_ldif.ldif``<br>



<h1>ldap tools</h1>

Generally speaking commands with ``slap`` in their name require ``slapd`` to be stopped before they are run - e.g. ``slapadd``.

Whilst commands with ``ldap`` in their name are the oppsite and need ``slapd`` running - e.g. ``ldapadd``.

A notable exception is that if you have a sufficintly new OpenLDAP (2.4+ with hdb/mdb) , it is safe to ``slapcat``. So you can safely dump the DB. Of course if in doubt check.

<h1>ldapsearch commands and hints</h1>

Stop ldapsearch doing ldif line wrapping:

``-o ldif-wrap=no``

Find users within an OU, an example search could look like this, where the search base is the OU of interest:

``ldapsearch -xLLL -b ou=Users,dc=domain,dc=something,dc=something``

...refining this to count users:

``ldapsearch -xLLL -b ou=Users,dc=domain,dc=something,dc=something "uid=*" uid | grep "uid:" | wc -l``

Finding if a module is loaded (example ppolicy), using root+SASL:

``ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=config"  -s sub "(&(objectclass=olcModuleList)(olcModuleLoad={0}ppolicy.la))" | grep ^olcModuleLoad:``


<h2>EXAMPLE ldapsearches</h2>

<h3>General searches</h3>

Search an OU (by setting base to that OU) for entries and sort by gidNumber

``ldapsearch -S gidNumber -xLLL -b ou=some_ou_name,dc=something,dc=something``

<h3>Configuration searches</h3>

View config

(note use of SASL which allows a client to request the server uses credentials established by a method external to the mechanism, to authenticate this client)

``ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config "(|(cn=config)(olcDatabase={1}mdb))"``

View config, olcAccess

``ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config "(|(cn=config)(olcDatabase={1}mdb))" olcAccess``


``ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config '(olcAccess=*)' olcAccess olcSuffix``

Check replication contextCSN - here assumes ``-b`` need not be supplied otherwise supply it (e.g. if not coming from ldap.conf)

``ldapsearch -xLLL -s base ContextCSN``


<h2>Advanced searches</h2>

Use at own risk... this might go into a file you run an ldapdelete on. It will produce the children (one level) of the ou=people

 ``ldapsearch -xLLL -s one -b "ou=people,dc=foo,dc=com" "(cn=*)" dn | awk -F": " '$1~/^\s*dn/{print $2}' ``

 redirect above to file and then dry run delete e.g.

``ldapdelete -n -vv -r -f listOfDNRemove.txt``

---------

Output an OU search then count the entries (DNs) whilst removing empty lines. Here using intermediate file, though you dont have to.

``ldapsearch -o ldif_wrap=no -xLLL -s sub -b "ou=people,dc=foo,dc=com" dn > some_file``
<br>
``grep "\S" some_file | wc -l``



<h1>ldap{add,modify} commands and hints</h1>


``ldapadd`` is similar to ``ldapmodify``, except as the name suggests it only adds. ``slapd`` needs to be online for the ``ldif`` to be processed by ``ldapadd`` or ``ldapmodify``.

It is not possible to ``ldapadd`` an ``ldif`` file generated by ``slapcat`` - which is typically used to backup an ldap instance's config and user-data. To do this would involve changes by hand. Excerpt from ``man page`` for ``slapcat``:  

``The  output  of  slapcat is intended to be used as input to slapadd(8).
The output of slapcat cannot generally be used as input  to  ldapadd(1)
or  other  LDAP clients without first editing the output.  This editing
would normally include reordering the records into superior first order
and removing no-user-modification operational attributes.``

<h2>Example LDIF change user password</h2>

``dn: cn=user,ou=example,dc=example,dc=com``<br>
``changetype: modify``<br>
``replace: userPassword``<br>
``userPassword: use_slappasswd_to_generate``<br>


<h1>slapadd commands and hints</h1>

Dry mode ``-u`` very useful to check before doing a final ``slapadd`` operation.

Slapadd performance:

``olcToolThreads`` - set this to 2. Specify the maximum number of threads to use in tool mode. This should not be greater than the number of CPUs in the system. The default is 1. This effects slapdadd performance. It gets added to ``cn: config``.


<h1>TLS settings and hints</h1>

Given the below ldif or similar, if you hit error ``ldap_modify: Other (e.g., implementation specific) error (80)`` you may need to check/disable ``selinux``.


``dn: cn=config``<br>
``changetype: modify``<br>
``replace: olcTLSCACertificateFile``<br>
``olcTLSCACertificateFile: /path/to/cert.crt``<br>
``-``<br>
``replace: olcTLSCertificateFile``<br>
``olcTLSCertificateFile: /path/to/cert.crt``<br>
``-``<br>
``replace: olcTLSCertificateKeyFile``<br>
``olcTLSCertificateKeyFile: /path/to/key.key``<br>


<h2>Require TLS</h2>

This ldif is requiring that access to the DIT is protected by a TLS connection - to disallow plain text communication.

``dn: olcDatabase={2}mdb,cn=config``<br>
``changetype: modify``<br>
``replace: olcSecurity``<br>
``olcSecurity: tls=1``




<h1>ACLs</h1>

Best to refer to the documentation so this...

https://www.openldap.org/doc/admin26/access-control.html

...but some guidance is here.

Using an LDIF like this will enable you to completely replace the FULL access list for mdb{2}.

This example is incomplete/partial and should be adapted as you see fit... it is merely an example ldif.

``dn: olcDatabase={2}mdb,cn=config``<br>
``changetype: modify``<br>
``replace: olcAccess``<br>
``olcAccess: {0} to attrs=userPassword by dn="cn=SomeAdmin,ou=Administrators,dc=example,dc=com" write by anonymous auth by self =xw by * none``<br>
``olcAccess: {1} to some_other_stuff by some_other_thing``<br>
``olcAccess: {2} to dn.base="" by * read``<br>

``=xw`` is giving explicit auth+write, whereas <i>write</i> gives more.

The last line is necessary otherwise binds are broken. It is providing access to the DSE, so a client can determine some information in order to connect.


<h2>Syncrepl and ACLs</h2>

If you make ACL changes on the master which you expect to impact a replica , it is advisable to clean and restart the replica.

See:

``If your ACLs change, then the LDAP server will be sending you notifications about entries which (until just now) did not exist to you, or the LDAP server will not send you notifications about entries that you now can no longer see. This manifests as weird consistency issues, which might cause exceptions to be thrown, or (even worse) might not.``

https://syncrepl-client.readthedocs.io/en/latest/gotchas.html


<h1>Logging</h1>

Example LDIF to turn ACL logging on.

``dn: cn=config``<br>
``changetype: modify``<br>
``replace: olcLogLevel``<br>
``olcLogLevel: 128``<br>

<h1>PROCEDURE: Build new master from dump - Import a dump on a new server build and migrate HDB to MDB</h1>

Before proceeding with final stags (slapadd) verify /etc/openldap/slapd.d is correct location.


- install OS on new server<br>
- install ldap server via RPM/DEB/compile etc.<br>
- ensure slapd is stopped<br>
``systemctl stop slapd``<br>
- copy over <i>config.ldif</i> to /root/ on new master (<i>config.ldif</i> being the slapcat dump/backup from previous master)<br>
- replace hdb with mdb in config.ldif<br>
``sed -i 's/hdb/mdb/g' config.ldif``<br>
``sed -i 's/Hdb/Mdb/g' config.ldif``<br>

- ensure olcDbMaxSize is set by editing the config.ldif<br>

``cn: olcDatabase={2}mdb,cn=config``<br>
``...``<br>
``...``<br>
``...``<br>
``olcDbMaxSize: 1000000000``<br>
<br>
- move/backup the slapd.d dir that was provided by the install earlier<br>
``cd /etc/openldap/``<br>
``cp -a slapd.d slapd.d.orig``<br>
- empty the slapd.d<br>
``rm -rf slapd.d/*``<br>
- import the <i>config.ldif</i> (config db dump)<br>
``slapadd -v -F /etc/openldap/slapd.d -n0 -l /root/config.ldif``<br>
- import the <i>data.ldif</i> (data dump)<br>
``slapadd -v -F /etc/openldap/slapd.d -l /root/data.ldif``<br>
- ensure ldap:ldap owner on slapd.d<br>
``chown -R ldap:ldap /etc/openldap/slapd.d``<br>
- sort out certificates<br>
``(generation or migration of certs is not covered here)``
- start sladp<br>
``systemctl start slapd``<br>

<h1>Issue: slapd not starting via systemd</h1>

Obviously this can be caused by many things, but do check the systemd pidfile location AND also check olcPidFile in cn=config. It may be olcPidFile needs set (to match systemd). Check logs for pidfile error.

<h1>Other/Misc</h1>

Search a slapcat dump entry within an ldif dump file.

``sed -n '/dn: cn=Something/,/modifiersName:/p' data.ldif  | grep something``

<h1>Useful man pages</h1>

slapo-ppolicy


<h1>SSSD considerations</h1>

https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/configuring_authentication_and_authorization_in_rhel/understanding-sssd-and-its-benefits_configuring-authentication-and-authorization-in-rhel#identity-and-authentication-providers-for-SSSD_understanding-SSSD-and-its-benefits

Taken from above URL 13.09.24

Table 3.1. Available Combinations of Identity and Authentication Providers

| Identity Provider	| Authentication Provider |
| :--- | :--- |
| Identity Management [a] | Identity Management |
| Active Directory | Active Directory |
| LDAP | LDAP |
| LDAP | Kerberos |
| Proxy | Proxy |
| Proxy | LDAP |
| Proxy | Kerberos |

[a] An extension of the LDAP provider type.



<h1>Python ldap3</h1>

https://ldap3.readthedocs.io/en/latest/index.html

```
#!/srv/example/example_venv/bin/python3 or python3 somewhere
from ldap3 import Server, Connection, ALL
server = Server('ldaps://example', get_info=ALL, use_ssl=True, port=636)
conn = Connection(server, 'cn=binduser,dc=example,dc=ex,dc=uk', 'SOME_PW', auto_bind=True)
filter = '(objectclass=posixAccount)'
var_attributes=['sn','cn']
conn.search(search_base='dc=example,dc=ex,dc=uk', search_filter=filter, attributes=var_attributes, search_scope='SUBTREE')
debug1=conn.entries
print(debug1)
```



