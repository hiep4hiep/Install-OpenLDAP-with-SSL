# Install-OpenLDAP-with-SSL
Install OpenLDAP on Ubuntu with SSL enabled and group memberOf overlay

### Install OpenLDAP app
```
sudo apt update
sudo apt -y install slapd ldap-utils
```

### Reconfigure slapd to add domain
```
dpkg-reconfigure slapd
```
Under that, change the domain to your system like example.com. This will be the based dn for every OpenLDAP components later on.


### Add a basedn ldif
```
vi basedn.ldif
```
```
dn: ou=people,dc=example,dc=com
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=example,dc=com
objectClass: organizationalUnit
ou: groups
```
Apply the basedn
```
ldapadd -x -D cn=admin,dc=example,dc=com -W -f basedn.ldif
```


### Enable the memberOf overlay
By default, OpenLDAP does not support the group membership of `PosixAccount user` with `groupOfNames`. You need to enable the memberOf overlay module feature so when creating a group LDIF, the group membership will be added to the user account automatically.

- Enable the module
```
ldapmodify -Q -Y EXTERNAL -H ldapi:///
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: memberof.la
```
You should expect `modifying entry "cn=module{0},cn=config"` message

- Verify the module, you can see the memberof.la
```
slapcat -n 0 | grep olcModuleLoad
olcModuleLoad: {0}back_mdb
olcModuleLoad: {1}memberof.la
```

- Apply the overlay to backend database
```
ldapadd -Y EXTERNAL -H ldapi:///
dn: olcOverlay=memberof,olcDatabase={1}mdb,cn=config
objectClass: olcOverlayConfig
objectClass: olcMemberOf
olcOverlay: memberof
olcMemberOfRefint: TRUE
```

### Test the configuration
- Create a new user ldif file 
```vi ldapusers.ldif```
```
dn: uid=user01,ou=people,dc=example,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: user01
sn: user01
userPassword: {SSHA}QI5YFIBdOMzv+UfWy5fE8JsWE94ZmpyO
loginShell: /bin/bash
gidNumber: 2000
uidNumber: 2000
homeDirectory: /home/user01
```
- Add the user01
```
ldapadd -x -D cn=admin,dc=example,dc=com -W -f ldapusers.ldif
```
- Create a new group ldif file
```vi ldapgroups.ldif```
```
dn: cn=standard,ou=groups,dc=asgardian,dc=network
objectClass: groupOfNames
cn: standard
member: uid=user01,ou=people,dc=example,dc=com
```
- Add the group
```
ldapadd -x -D cn=admin,dc=example,dc=com -W -f ldapgroups.ldif
```
- Verify that the memberOf is there for the user01
```
slapcat

dn: uid=user01,ou=people,dc=example,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: user01
sn: user01
userPassword:: e1NTSEF9UUk1WUZJQmRPTXp2K1VmV3k1ZkU4SnNXRTk0Wm1weU8=
loginShell: /bin/bash
gidNumber: 2000
uidNumber: 2000
homeDirectory: /home/user01
structuralObjectClass: inetOrgPerson
uid: user01
entryUUID: b6cc0832-7436-103d-9964-3dbc870410a6
creatorsName: cn=admin,dc=asgardian,dc=network
createTimestamp: 20230421021939Z
entryCSN: 20230421021939.823066Z#000000#000#000000
modifyTimestamp: 20230421021939Z
memberOf: cn=standard,ou=groups,dc=example,dc=com
modifiersName: cn=admin,dc=example,dc=com
```
