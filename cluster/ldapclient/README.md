# Installing Ldap Client


```shell
DOMAIN_NAME=master.cloud.com
LDAP_DOMAIN=master.cloud.com
DC_1=master.cloud
DC_2=com
DC=dc=master,dc=cloud,dc=com
BASE_DN=dc=master,dc=cloud,dc=com
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
apt-get install -yq ntp ntpdate nmap libsasl2-modules-gssapi-mit
```

```shell

echo "$LDAP_PASSWORD" >/etc/ldap.secret
chmod 600 /etc/ldap.secret
```

```shell

# Cleanup Apt
apt-get autoremove
apt-get autoclean
apt-get clean
```

```shell
mkdir -p /etc/secret/ldap
echo "${LDAP_PASSWORD}" >/etc/secret/ldap/password
```

```shell
: ${ENABLE_KRB:=$ENABLE_KRB}
: ${REALM:=$(echo $DOMAIN_NAME | tr 'a-z' 'A-Z')}
: ${DOMAIN_REALM:=$DOMAIN_NAME}
: ${LDAP_HOST:=$LDAP_HOST}
: ${BASE_DN:=$DC}
: ${LDAP_PASSWORD:=$(</etc/secret/ldap/password)}
: ${DC_1:=$DC_1}
: ${DC_2:=$DC_2}
: ${ENABLE_KUBERNETES:=$ENABLE_KUBERNETES}
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

STATUS=$(grep "umask=0022 skel=/etc/skel" /etc/pam.d/common-session)

if [ -z "$STATUS" ]; then
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

fi

service nscd restart
```

```shell


STATUS=$(grep "%admins ALL=(ALL) ALL" /etc/sudoers)
if [ -z "$STATUS" ]; then
  $(sed -i '/%admin ALL=(ALL) ALL/a\%admins ALL=(ALL) ALL' /etc/sudoers)
fi
```