# Installing Ldap Server

```shell
DOMAIN_NAME=cloud.com
LDAP_DOMAIN=cloud.com
DC_1=cloud
DC_2=com
DC=dc=cloud,dc=com
BASE_DN=dc=cloud,dc=com
LDAP_HOSTNAME=master.cloud.com
KDC_ADDRESS=master.cloud.com
LDAP_HOST=ldap://master.cloud.com
ENABLE_SSL=false
ENABLE_KUBERNETES=false
LDAP_PASSWORD=sumit
DEBIAN_FRONTEND=noninteractive
LDAP_ORG=CloudInc
```

```shell

apt-get update
```

```shell

echo "slapd slapd/internal/adminpw password ${LDAP_PASSWORD}" | debconf-set-selections
echo "slapd slapd/internal/generated_adminpw password ${LDAP_PASSWORD}" | debconf-set-selections
echo "slapd slapd/password2 password ${LDAP_PASSWORD}" | debconf-set-selections
echo "slapd slapd/password1 password ${LDAP_PASSWORD}" | debconf-set-selections
echo "slapd slapd/domain string ${LDAP_DOMAIN}" | debconf-set-selections
echo "slapd shared/organization string ${LDAP_ORG}" | debconf-set-selections
echo "slapd slapd/backend string HDB" | debconf-set-selections
echo "slapd slapd/purge_database boolean true" | debconf-set-selections
echo "slapd slapd/move_old_database boolean true" | debconf-set-selections
echo "slapd slapd/allow_ldap_v2 boolean false" | debconf-set-selections
echo "slapd slapd/no_configuration boolean false" | debconf-set-selections
echo "slapd slapd/dump_database_destdir string /var/backups/slapd-VERSION" | debconf-set-selections
echo "slapd slapd/dump_database select when needed" | debconf-set-selections
```

```shell

LC_ALL=C DEBIAN_FRONTEND=noninteractive apt-get install -y slapd ldap-utils
```

```shell

echo "ldap-auth-config ldap-auth-config/rootbindpw password ${LDAP_PASSWORD}" | debconf-set-selections
echo "ldap-auth-config ldap-auth-config/bindpw password ${LDAP_PASSWORD}" | debconf-set-selections
echo "ldap-auth-config ldap-auth-config/dblogin boolean false" | debconf-set-selections
echo "ldap-auth-config ldap-auth-config/override boolean true" | debconf-set-selections
echo "ldap-auth-config ldap-auth-config/ldapns/ldap-server string ldap://$LDAP_HOSTNAME" | debconf-set-selections
echo "ldap-auth-config ldap-auth-config/pam_password string md5" | debconf-set-selections
echo "ldap-auth-config ldap-auth-config/dbrootlogin boolean true" | debconf-set-selections
echo "ldap-auth-config ldap-auth-config/binddn string cn=proxyuser,dc=example,dc=net" | debconf-set-selections
echo "ldap-auth-config ldap-auth-config/ldapns/ldap_version string 3" | debconf-set-selections
echo "ldap-auth-config ldap-auth-config/move-to-debconf boolean true" | debconf-set-selections
echo "ldap-auth-config ldap-auth-config/ldapns/base-dn string $BASE_DN" | debconf-set-selections
echo "ldap-auth-config ldap-auth-config/rootbinddn string cn=admin,$BASE_DN" | debconf-set-selections
```

```shell

apt-get install -yq ldap-auth-client nscd libpam-ccreds
apt-get install -yq ntp ntpdate nmap
```

```shell

mkdir -p /etc/secret/ldap
echo "${LDAP_PASSWORD}" >/etc/secret/ldap/password
```

```shell

STATUS=$(grep "ldap" /etc/nsswitch.conf)
if [ -z "$STATUS" ]; then
sed -i 's/files systemd/files ldap systemd/g' /etc/nsswitch.conf
sed -i 's/\(shadow:\)\(.*\)files/\1\2files ldap/g' /etc/nsswitch.conf
sed -i 's/netgroup:\(.*\)nis/netgroup:\1ldap/g' /etc/nsswitch.conf
fi
```

```shell

echo '# Disable if using Kerberos:
account [success=2 new_authtok_reqd=done default=ignore]        pam_unix.so
account [success=1 default=ignore]      pam_ldap.so

# Enable if using Kerberos:
#account [success=1 new_authtok_reqd=done default=ignore]        pam_unix.so

account requisite                       pam_deny.so

account required                        pam_permit.so

# Enable if using Kerberos:
#account required                        pam_krb5.so minimum_uid=1000' >/etc/pam.d/common-account

echo '# Disable if using Kerberos:
auth    [success=2 default=ignore]      pam_unix.so nullok_secure
auth    [success=1 default=ignore]      pam_ldap.so use_first_pass

# Enable if using Kerberos:
#auth    [success=2 default=ignore]      pam_krb5.so minimum_uid=1000
#auth    [success=1 default=ignore]      pam_unix.so nullok_secure try_first_pass

auth    requisite                       pam_deny.so

auth    required                        pam_permit.so' >/etc/pam.d/common-auth

echo '# Disable if using Kerberos:
password        [success=2 default=ignore]      pam_unix.so obscure sha512
password        [success=1 user_unknown=ignore default=die]     pam_ldap.so use_authtok try_first_pass

# Enable if using Kerberos:
#password        [success=2 default=ignore]      pam_krb5.so minimum_uid=1000
#password        [success=1 default=ignore]      pam_unix.so obscure use_authtok try_first_pass sha512

password        requisite                       pam_deny.so

password        required                        pam_permit.so' >/etc/pam.d/common-password

echo 'session [default=1]                     pam_permit.so

session requisite                       pam_deny.so

session required                        pam_permit.so

# Enable if using Kerberos:
#session  optional  pam_krb5.so minimum_uid=1000

session required        pam_unix.so

# Disable if using Kerberos:
session optional                        pam_ldap.so
session required        pam_mkhomedir.so        skel=/etc/skel umaks=0022' >/etc/pam.d/common-session
```

```shell

service nscd restart
```

```shell

STATUS=$(grep "%admins ALL=(ALL) ALL" /etc/sudoers)
if [ -z "$STATUS" ]; then
sed -i '/%admin ALL=(ALL) ALL/a\%admins ALL=(ALL) ALL' /etc/sudoers
fi
```

```shell

echo "$LDAP_PASSWORD" >/etc/ldap.secret
chmod 600 /etc/ldap.secret
adduser openldap sudo
echo openldap:openldap | chpasswd
chown -R openldap:openldap /var/lib/ldap
chown -R openldap:openldap /etc/ldap

chgrp openldap /etc/init.d/slapd
chmod g+x /etc/init.d/slapd
echo "local4.*			/var/log/sldapd.log" >/etc/rsyslog.d/slapd.conf
```

```shell

# Cleanup Apt
apt-get autoremove
apt-get autoclean
apt-get clean
```

```shell

touch /var/userid
echo '1000' >/var/userid
chown root:root /var/userid
touch /var/groupid
chown root:root /var/groupid
echo '502' >/var/groupid
```

```shell

: ${REALM:=$(echo $DOMAIN_NAME | tr 'a-z' 'A-Z')}
: ${DOMAIN_REALM:=$DOMAIN_NAME}
: ${LDAP_HOST:=$LDAP_HOST}
: ${BASE_DN:=$DC}
: ${LDAP_PASSWORD:=$(</etc/secret/ldap/password)}
: ${DC_1:=$DC_1}
: ${DC_2:=$DC_2}
```

```shell
ldapmodify -c -Y EXTERNAL -H ldapi:/// <<EOF
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcAccess
olcAccess: to * by dn="cn=admin,$BASE_DN" write
EOF

ldapmodify -c -Y EXTERNAL -H ldapi:/// <<EOF
dn: cn=config
changetype: modify
replace: olcLogLevel
olcLogLevel: 256
EOF
  
ldapadd -x -D "cn=admin,$BASE_DN" -w $LDAP_PASSWORD -H ldapi:/// <<EOF
dn: ou=users,$BASE_DN
ou: users
objectClass: organizationalUnit
objectclass: top

dn: ou=groups,$BASE_DN
ou: groups
objectClass: organizationalUnit
objectclass: top
EOF

ldapadd -x -D "cn=admin,$BASE_DN" -w $LDAP_PASSWORD -H ldapi:/// <<EOF
dn: cn=admins,ou=groups,$BASE_DN
cn: admins
gidnumber: 500
objectclass: posixGroup
objectclass: top

dn: cn=users,ou=groups,$BASE_DN
cn: users
gidnumber: 501
objectclass: posixGroup
objectclass: top
EOF
  

ldapadd -x -D "cn=admin,$BASE_DN" -w $LDAP_PASSWORD -H ldapi:/// <<EOF  
dn: cn=smaji,ou=users,$BASE_DN
cn: smaji
gidnumber: 500
givenname: Sumit
homedirectory: /home/users/smaji
loginshell: /bin/bash
objectclass: inetOrgPerson
objectclass: posixAccount
objectclass: top
sn: smaji
uid: smaji
uidnumber: 1001
userpassword: sumit
EOF
  

```
