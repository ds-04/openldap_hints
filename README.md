# openldap_hints

openldap_hints - collection of commands or procedures to help managing OpenLDAP.




<h1>Build new master from dump - Import a dump on a new server build and migrate HDB to MDB</h1>

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
``slapadd -vv -F /etc/openldap/slapd.d -n0 -l /root/config.ldif``<br>
- import the <i>data.ldif</i> (data dump)<br>
``slapadd -vv -F /etc/openldap/slapd.d -l /root/data.ldif``<br>
- ensure ldap:ldap owner on slapd.d<br>
``chown -R ldap:ldap /etc/openldap/slapd.d``<br>
- sort out certificates<br>
``(generate of migrate certs - not covered here)``
- start sladp<br>
``systemctl start slapd``<br>
