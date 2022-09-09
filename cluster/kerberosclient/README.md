# Install Kerberos in client node

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
KDC_PASSWORD=admin
```

```shell

echo "kerberos krb5-config/default_realm string ${DOMAIN_NAME}" | debconf-set-selections
```

```shell

# kerberos client and pam configuration for kerberos
LC_ALL=C DEBIAN_FRONTEND=noninteractive apt-get install -yq krb5-user libpam-krb5
```

```shell

# Kerberize services
apt-get install -yq libsasl2-modules-gssapi-mit
```

```shell

mkdir -p /etc/secret/krb
mkdir -p /etc/secret/ldap
echo "${LDAP_PASSWORD}" >/etc/secret/ldap/password
echo "${KDC_PASSWORD}" >/etc/secret/krb/password
echo "${KDC_PASSWORD}" >/etc/secret/krb/kdcpassword
echo "${KDC_PASSWORD}" >/etc/secret/krb/admpassword
```

```shell

: ${ENABLE_KRB:=$ENABLE_KRB}
: ${REALM:=$(echo $DOMAIN_NAME | tr 'a-z' 'A-Z')}
: ${DOMAIN_REALM:=$DOMAIN_NAME}
: ${KERB_MASTER_KEY:=masterkey}
: ${KERB_ADMIN_USER:=root}
: ${KERB_ADMIN_PASS:=$(</etc/secret/krb/password)}
: ${KDC_ADDRESS:=$KDC_ADDRESS}
: ${LDAP_HOST:=$LDAP_HOST}
: ${BASE_DN:=$DC}
: ${LDAP_PASSWORD:=$(</etc/secret/ldap/password)}
: ${DC_1:=$DC_1}
: ${DC_2:=$DC_2}
: ${ENABLE_KUBERNETES:=$ENABLE_KUBERNETES}
```


```shell

echo 'account [success=1 new_authtok_reqd=done default=ignore]        pam_unix.so
account requisite                       pam_deny.so
account required                        pam_permit.so
account required                        pam_krb5.so minimum_uid=1000' >/etc/pam.d/common-account

echo 'auth    [success=2 default=ignore]      pam_krb5.so minimum_uid=1000
auth    [success=1 default=ignore]      pam_unix.so nullok_secure try_first_pass
auth    requisite                       pam_deny.so
auth    required                        pam_permit.so' >/etc/pam.d/common-auth

echo 'password        [success=2 default=ignore]      pam_krb5.so minimum_uid=1000
password        [success=1 default=ignore]      pam_unix.so obscure use_authtok try_first_pass sha512
password        requisite                       pam_deny.so
password        required                        pam_permit.so' >/etc/pam.d/common-password

echo 'session [default=1]                     pam_permit.so
session requisite                       pam_deny.so
session required                        pam_permit.so
session optional                        pam_krb5.so minimum_uid=1000
session required        pam_unix.so
session required        pam_mkhomedir.so        skel=/etc/skel umaks=0022' >/etc/pam.d/common-session

service nscd restart
```

```shell

mkdir -p /var/log/kerberos

touch /var/log/kerberos/krb5libs.log
touch /var/log/kerberos/krb5kdc.log
touch /var/log/kerberos/kadmind.log
cat >/etc/krb5.conf <<EOF
[logging]
default = FILE:/var/log/kerberos/krb5libs.log
kdc = FILE:/var/log/kerberos/krb5kdc.log
admin_server = FILE:/var/log/kerberos/kadmind.log

[libdefaults]
default_realm = $REALM
dns_lookup_realm = false
dns_lookup_kdc = false
ticket_lifetime = 24h
renew_lifetime = 7d
forwardable = true
proxiable = true

[realms]
$REALM = {
kdc = $KDC_ADDRESS
admin_server = $KDC_ADDRESS
database_module = openldap_ldapconf
}

[domain_realm]
.$DOMAIN_REALM = $REALM
$DOMAIN_REALM = $REALM

[dbdefaults]
ldap_kerberos_container_dn = cn=krbContainer,$BASE_DN

[dbmodules]
openldap_ldapconf = {
db_library = kldap
ldap_kdc_dn = cn=kdc-srv,ou=krb5,$BASE_DN
ldap_kadmind_dn = cn=adm-srv,ou=krb5,$BASE_DN
ldap_service_password_file = /etc/krb5kdc/service.keyfile
ldap_conns_per_server = 5
ldap_servers = $LDAP_HOST
}
EOF
```

```shell

kadmin -p root/admin -w $KERB_ADMIN_PASS -q "addprinc -randkey host/$(hostname -f)@$REALM"
kadmin -p root/admin -w $KERB_ADMIN_PASS -q "xst -k /etc/krb5.keytab host/$(hostname -f)@$REALM"
```