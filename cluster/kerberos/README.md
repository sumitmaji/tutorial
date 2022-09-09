# Installing Kerberos

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

apt-get update
```

```shell

echo "kerberos krb5-config/default_realm string ${DOMAIN_NAME}" | debconf-set-selections
```

```shell

LC_ALL=C DEBIAN_FRONTEND=noninteractive apt-get install -yq krb5-kdc krb5-admin-server krb5-kdc-ldap
```
```shell

# kerberos client and pam configuration for kerberos
apt-get install -yq krb5-user libpam-krb5
```

```shell

mkdir -p /etc/secret/krb
echo "${LDAP_PASSWORD}" >/etc/secret/ldap/password
echo "${KDC_PASSWORD}" >/etc/secret/krb/password
echo "${KDC_PASSWORD}" >/etc/secret/krb/kdcpassword
echo "${KDC_PASSWORD}" >/etc/secret/krb/admpassword
```


```shell

: ${REALM:=$(echo $DOMAIN_NAME | tr 'a-z' 'A-Z')}
: ${DOMAIN_REALM:=$DOMAIN_NAME}
: ${KERB_MASTER_KEY:=masterkey}
: ${KERB_ADMIN_USER:=root}
: ${KERB_ADMIN_PASS:=$(</etc/secret/krb/password)}
: ${LDAP_HOST:=$LDAP_HOST}
: ${BASE_DN:=$DC}
: ${LDAP_PASSWORD:=$(</etc/secret/ldap/password)}
: ${KDC_PASSWORD:=$(</etc/secret/krb/kdcpassword)}
: ${ADM_PASSWORD:=$(</etc/secret/krb/admpassword)}
```

```shell

# Create kerberos schema
# https://ubuntu.com/server/docs/service-kerberos-with-openldap-backend
apt-get install -yq schema2ldif
cp /usr/share/doc/krb5-kdc-ldap/kerberos.schema.gz config/
gzip -d config/kerberos.schema.gz
ldap-schema-manager -i config/kerberos.schema

# Create kdc and kadmin user
# https://ubuntu.com/server/docs/service-kerberos-with-openldap-backend
echo "dn: ou=krb5,$BASE_DN
ou: krb5
objectClass: organizationalUnit

dn: cn=kdc-srv,ou=krb5,$BASE_DN
cn: kdc-srv
objectClass: simpleSecurityObject
objectClass: organizationalRole
description: Default bind DN for the Kerberos KDC server
userPassword: $KDC_PASSWORD

dn: cn=adm-srv,ou=krb5,$BASE_DN
cn: adm-srv
objectClass: simpleSecurityObject
objectClass: organizationalRole
description: Default bind DN for the Kerberos Administration server
userPassword: $ADM_PASSWORD" >/tmp/krb5.ldif
ldapadd -x -D "cn=admin,$BASE_DN" -w $LDAP_PASSWORD -H ldapi:/// -f /tmp/krb5.ldif

# Grant access to kdc and kadmin
# https://ubuntu.com/server/docs/service-kerberos-with-openldap-backend
ldapmodify -c -Y EXTERNAL -H ldapi:/// <<EOF
# 2.1.1
dn: olcDatabase={1}mdb,cn=config
changetype: modify
delete: olcAccess
-
# 2.2.1.
add: olcAccess
olcAccess: to attrs=userPassword,shadowLastChange
  by anonymous auth
  by * none
-
# 2.2.2.
add: olcAccess
olcAccess: to dn.subtree="cn=$REALM,cn=krbContainer,$BASE_DN"
  by dn="cn=adm-srv,ou=krb5,$BASE_DN" write
  by dn="cn=kdc-srv,ou=krb5,$BASE_DN" read
  by * none
-
# 2.2.3.
add: olcAccess
olcAccess: to attrs=loginShell
  by self write
  by users read
  by * none
-
# 2.2.4.
add: olcAccess
olcAccess: to dn.base=""
  by * read
-
# 2.2.5.
add: olcAccess
olcAccess: to *
  by users read
  by * none
EOF



```


```shell

: ${KDC_ADDRESS:=$(hostname -f)}

sudo mkdir /var/log/kerberos
sudo touch /var/log/kerberos/{krb5kdc,kadmin,krb5lib}.log
sudo chmod -R 750 /var/log/kerberos

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
cat >/etc/krb5kdc/kdc.conf <<EOF
[kdcdefaults]
kdc_ports = 750,88

[realms]
$REALM = {
database_name = /var/lib/krb5kdc/principal
admin_keytab = FILE:/etc/krb5kdc/kadm5.keytab
acl_file = /etc/krb5kdc/kadm5.acl
key_stash_file = /etc/krb5kdc/stash
kdc_ports = 750,88
max_life = 10h 0m 0s
max_renewable_life = 7d 0h 0m 0s
master_key_type = des3-hmac-sha1
supported_enctypes = aes256-cts:normal arcfour-hmac:normal des3-hmac-sha1:normal des-cbc-crc:normal des:normal des:v4 des:norealm des:onlyrealm des:afs3
default_principal_flags = +preauth
}
EOF
cat >/etc/krb5kdc/kadm5.acl <<EOF
*/admin *
EOF
```

```shell

# Create kerberos db in ldap
kdb5_ldap_util -D cn=admin,$BASE_DN -w $LDAP_PASSWORD \
-H $LDAP_HOST create -subtrees cn=krbContainer,$BASE_DN -r $REALM -s -P $KERB_ADMIN_PASS 2>error
output=$(grep "Can't contact LDAP server while initializing database" error)
while [ ! -z "$output" ]; do
sleep 10
kdb5_ldap_util -D cn=admin,$BASE_DN -w $LDAP_PASSWORD \
-H $LDAP_HOST create -subtrees cn=krbContainer,$BASE_DN -r $REALM -s -P $KERB_ADMIN_PASS 2>error
output=$(grep "Can't contact LDAP server while initializing database" error)
done

# Setting up password to kdc and kadmin to access kerberos db in ldap
kdb5_ldap_util -D cn=admin,$BASE_DN -w $LDAP_PASSWORD stashsrvpw \
-f /etc/krb5kdc/service.keyfile cn=kdc-srv,ou=krb5,$BASE_DN <<EOF
$KDC_PASSWORD
$KDC_PASSWORD
EOF
kdb5_ldap_util -D cn=admin,$BASE_DN -w $LDAP_PASSWORD stashsrvpw \
-f /etc/krb5kdc/service.keyfile cn=adm-srv,ou=krb5,$BASE_DN <<EOF
$ADM_PASSWORD
$ADM_PASSWORD
EOF
```

```shell

kadmin.local -q "addprinc -pw $KERB_ADMIN_PASS $KERB_ADMIN_USER/admin"
echo "*/admin@$REALM *" >/etc/krb5kdc/kadm5.acl
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
```

```shell

invoke-rc.d krb5-admin-server start
invoke-rc.d krb5-kdc start
```

```shell

kadmin -p root/admin -w $KERB_ADMIN_PASS -q "addprinc -randkey host/$(hostname -f)@$REALM"
kadmin -p root/admin -w $KERB_ADMIN_PASS -q "xst -k /etc/krb5.keytab host/$(hostname -f)@$REALM"
```