#NiFi Identity Conversion
In a secure NiFi environment, the identity of a user can be determined in a number of ways depending on the authentication configuration. Machines also have an identity that needs to be determined upon authentication. Determining the identity of an entity is important to ensure proper authorization and access to resources.

##Machine Identity
The identity of the node in a NiFi cluster is determined by the SSL certificate that is used for secure communication with other nodes in the cluster. This certificate can be generated by the internal Certificate Authority privided with HDF, or by an external CA. Once SSL is enabled on the cluster using the certificates, they will be stored (by default) in the `/etc/nifi/con/keystore.jks` keystore.

To get the node's identity as specified in the certificate, first get the keystore password from the nifi.properties file, then run the keytool command:
```
cat /etc/nifi/conf/nifi.properties | grep keystorePasswd
nifi.security.keystorePasswd=lF6e7sJsD3KxwNsrVqeXbYhGNu3QqTlhLmC5ztwlX/c

keytool -list -v -keystore /etc/nifi/conf/keystore.jks
```

This command will print out all of the information about the node's certificate. The `Owner` field contains the node's identity.
```
Alias name: nifi-key
Creation date: Oct 7, 2016
Entry type: PrivateKeyEntry
Certificate chain length: 2
Certificate[1]:
Owner: CN=nifi-2.example.com, CN=hosts, CN=accounts, DC=example, DC=com
Issuer: CN=nifi-1.example.com, OU=NIFI
Serial number: 157a059d1cb00000000
Valid from: Fri Oct 07 18:13:43 UTC 2016 until: Mon Oct 07 18:13:43 UTC 2019
Certificate fingerprints:
	 MD5:  C2:BD:6A:CE:86:05:C9:C1:E8:DE:0C:C1:62:B5:27:5B
	 SHA1: 3A:BA:E4:35:DA:91:D2:DB:E3:A1:BA:C8:7F:19:C4:C2:BD:81:5A:8F
	 SHA256: 2A:4F:05:51:9E:4F:50:8B:0D:B0:4C:55:AD:21:65:CF:5D:C2:85:8B:BA:0F:CB:5A:95:AC:C4:3D:08:62:13:02
	 Signature algorithm name: SHA256withRSA
	 Version: 3

Extensions:
...
```

In the example above, the identity of the node (Owner of the certificate) is `CN=nifi-2.example.com, CN=hosts, CN=accounts, DC=example, DC=com`.

If the certificates are managed by the internal CA, the node identity is determined by two parameters in the NiFi configuration that convert the hostname into a distinguished name (DN) format:

![Image](images/nifi-dn-params.png?raw=true)

The node idenity from the certificate above was generated using the parameters shown in the Ambari NiFi configuration. The NiFi CA uses the `CA DN Prefix` + `hostname` + `CA DN Suffix` to generate the Owner field stored in the certificate. It is important to note the transformation that occurs between the configuration parameteters.

```
Hostname: nifi-2.example.com
CA DN Prefix: CN=
CA DN Suffix: cn=hosts,cn=accounts,dc=example,dc=com
```
Is translated into a node identity of:
```
CN=nifi-2.example.com, CN=hosts, CN=accounts, DC=example, DC=com
```

The lowercase attribute identifiers (cn, dc, etc.) are converted to uppercase (CN, DC, etc.) and a space is added between each component of the distinguishted name. These transformations will become important later when identity conversions are created.

##User Identity
The user's identity can be determined in multiple ways depending on how security is configured within the cluster:
- If certificate based user authentication is used, the user identity is determined from the certificate just as it is for node identity.
- If LDAP authentication is used, the user identity is determined by the distinguised name attribute passed back from the LDAP server.
- If Kerberos authentication is used, the user idnetity is determined based on the Kerberos principal

###Certificate Based User Authentication
The user identity can be determined via SSL certificate in the same way that the node identity is. The same conversion for DN Prefix and DN Suffix occurs when generating user certificates using the SSL Toolkit, and the same methods for pulling the identity out of the certificate can be used.

###LDAP Based User Authentication
If LDAP authentication is enabled, the LDAP server will pass back the distinguished name (DN) of the user entry in the directory. This value is used to determine the user identity. It may not be clear from the LDAP server configuration exactly how the DN will be formatted when it is passed back. For pattern matching and idnetity conversion, the case of the field names and spacing of the DN value will be important. To determine the format, a simple ldapsearch can be performed for a known username.

Windows Active Directory:
```
ldapsearch -W -h adserver.example.com -p 389 -D "cn=hadoopadmin,OU=ServiceUsers,dc=cloud,dc=hortonworks,dc=com" -b "cn=Users,dc=example,dc=com" sAMAccountName=hadoopadmin
```

OpenLDAP:
```
ldapsearch -W -h ldapserver.example.com -p 389 -D "uid=hadoopadmin,cn=users,cn=accounts,dc=example,dc=com" uid=hadoopadmin
```

In the output, find the `dn` field for the user:

Windows Active Directory:
```
dn: CN=hadoopadmin,OU=ServiceUsers,DC=example,DC=com
```

OpenLDAP:
```
dn: uid=hadoopadmin,cn=users,cn=accounts,dc=example,dc=com
```

Note the case and the spacing of the returned value for later configuration steps.

###Kerberos Based User Authentication
When Kerberos authentication is used, the identity of the user is determined from the Kerberos principal. The principal takes a form of `username@REALM`. For example:

```
hadoopadmin@EXAMPLE.COM
```

The realm is (by convention) the domain in uppercase.

##Identity Conversion
NiFi uses the identity that it determines from the various authentication mechanisms during authorization procedures. In an HDP cluster, authorization is provided by Apache Ranger. Ranger syncs usernames from Active Directory or LDAP, but it does not sync them in the distinguished name format that is returned during authentication against these mechanisms. Likewise, the Kerberos principal format is not typically used in Ranger. As such, the interesting portion of the DN or principal style identity must be parsed out for use with Ranger.

NiFi provides a mechanism for transforming the certificate, LDAP, or Kerberos based identity. This is done via pairings of configuration parameters of the form:

```
nifi.security.identity.mapping.pattern.<unique>
nifi.security.identity.mapping.value.<unique>
```

The `<unique>` portion is replaced with a unique string identifying the purpose of the transformation. There are two pairings created by default (`<unique>=dn, and <unique>=kerb`), but other pairings can be created as needed. For the `pattern` portion of the pairing, Regular Expression syntax is used to parse the original identity into components. The `value` portion of the pairing uses these parsed components in variable substition format to build the translated version of the idenity. A few important operators for the translation are:
- `^` - Denotes the beginning of the value
- `$` - Denotes the end of the value
- `()` - Assigns matched strings to a variable. Variable names start with `1` and increment for each time used in the Regular Expression
- `.` - Matches any character
- `*` - Matches 0 or more of the preceding character
- `?` - Matches exactly one of any character

Using these operators, it is possible to separate any of the identities discussed so far into their components. Using the `dn` pairing of configuration parameters, separating the DN returned by LDAP into just the username can be accomplished with the following.

Windows Active Directory:
```
nifi.security.identity.mapping.pattern.dn = ^CN=(.*?),OU=ServiceUsers.*$
nifi.security.identity.mapping.value.dn = $1
```

OpenLDAP:
```
nifi.security.identity.mapping.pattern.dn = ^uid=(.*?),cn=users.*$
nifi.security.identity.mapping.value.dn = $1
```

If there is a need to use additional components of the DN for the user identity, the DN can be split into additional variables
```
nifi.security.identity.mapping.pattern.dn = ^CN=(.*?),OU=(.*?),DC=(.*?),DC=(.*?)$
```

The full list of variables created by the `pattern` variable in this example is:
- `$1` = hadoopadmin
- `$2` = ServiceUsers
- `$3` = example
- `$4` = com

To convert the host identity from SSL certificates (and user identities from internal CA generated user certificates), use an identity mapping pairing such as:
```
nifi.security.identity.mapping.pattern.host = ^CN=(.*?), CN=hosts.*$
nifi.security.identity.mapping.value.host = $1
```
in this example, note the space in `, CN=` and the case of the `CN`. These are becaue of the conversion that the CA performs when generating the SSL certificate as described aboe.

If Kerberos is enabled on the NiFi cluster, the Kerberos principal can be converted to a username in the following way:
```
nifi.security.identity.mapping.pattern.kerb = ^(.*?)@(.*?)$
nifi.security.identity.mapping.value.kerb = $1
```

##Conclusion
Identity determination in a NiFi cluster can be a complex topic, but thankfully, NiFi provides a powerful mechanism for parsing identities into a common format understandable by the Ranger authorization mechanisms. Identity mapping pairings should exist for all methods of identity mapping that will be needed in the NiFi cluster. An identity to be mapped should only match a single set of mapping rules to ensure reliable mapping of identities. The default pair of mappings (`dn` and `kerb`) are defined in the `Advanced nifi-properties` section of the Ambari NiFi configuration. Additional pairings can be added to the `Custom nifi-properties` section of the Ambari NiFi configuration.
