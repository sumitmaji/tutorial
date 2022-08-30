
### DNS
```shell
apt-get update
apt-get install net-tools
```

```shell
hostnamectl set-hostname master.cloud.com
```

```shell
rm /etc/netplan/00-installer-config.yaml
touch /etc/netplan/00-installer-config.yaml
cat > /etc/netplan/00-installer-config.yaml << EOF
network:
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [11.0.0.1/24]
      nameservers:
        addresses: [8.8.8.8,8.8.4.4]
    enp0s8:
      dhcp4: true
  version: 2
EOF
```

```shell
netplan apply
```

```shell
apt-get install -y isc-dhcp-server wget nfs-common bind9 ntp gcc make ifupdown
```

```shell
cat >>  /etc/bind/named.conf.default-zones << EOF
##zone append begin
zone "cloud.com" {
	type master;
	file "/etc/bind/cloud.com.fwd";
	allow-update { key rndc-key; };
};

zone "0.0.11.in-addr.arpa" {
	type master;
	file "/etc/bind/cloud.com.rev";
	allow-update { key rndc-key; };
};
##zone append end
EOF
```

```shell
sed -i '$r /etc/bind/rndc.key' /etc/bind/named.conf.options
```

```shell
sed -i 's_/etc/bind/\*\* r,_/etc/bind/\*\* rw,_' /etc/apparmor.d/usr.sbin.named
```

```shell
service apparmor restart
```

```shell
cat >> /etc/bind/cloud.com.fwd << EOF
\$TTL	86400
@	IN 		SOA 	master.cloud.com.		root.cloud.com. (
	1		;Serial
	604800	;Refresh
	86400	;Retry
	2419200	;Expire
	86400	;minimum
)
@	IN		NS		master.cloud.com.
master		IN		A	11.0.0.1
EOF
```

```shell
cat >> /etc/bind/cloud.com.rev << EOF
\$TTL    86400
@       IN              SOA     master.cloud.com.         root.cloud.com. (
        1               ;Serial
        3600            ;Refresh
        1800            ;Retry
        604800          ;Expire
        86400           ;minimum ttl
)
@       IN      NS      master.cloud.com
master.cloud.com  IN      A       11.0.0.1
1       IN      PTR     master.cloud.com
EOF
```

```shell
service bind9 restart
```

```shell

cat << EOF > /etc/dhcp/dhcpd.conf
`cat /etc/bind/rndc.key`
zone cloud.com. {
        primary 127.0.0.1;
        key rndc-key;
}

zone 0.0.11.in-addr.arpa. {
        primary 127.0.0.1;
        key rndc-key;
}

`cat /etc/dhcp/dhcpd.conf`
EOF

sed -i 's/ddns-update-style none/ddns-update-style interim/' /etc/dhcp/dhcpd.conf
sed -i '/ddns-update-style interim/ a\autorotive;' /etc/dhcp/dhcpd.conf
sed -i '/autorotive/ a\ddns-domainname "cloud.com";' /etc/dhcp/dhcpd.conf
sed -i '/ddns-domainname "cloud.com"/ a\ddns-rev-domainname "in-addr.arpa";' /etc/dhcp/dhcpd.conf
sed -i '/ddns-rev-domainname "in-addr.arpa"/ a\ddns-updates on;' /etc/dhcp/dhcpd.conf
sed -i 's/option domain-name "example.org";/option domain-name "cloud.com";/' /etc/dhcp/dhcpd.conf
sed -i "s/option domain-name-servers ns1.example.org, ns2.example.org;/option domain-name-servers 11.0.0.1, 192.168.0.1;/" /etc/dhcp/dhcpd.conf
sed -i '/^max-lease-time 7200;/ a\subnet 11.0.0.0 netmask 255.255.255.0 {' /etc/dhcp/dhcpd.conf
sed -i '/subnet 11.0.0.0 netmask 255.255.255.0 {/ a\option routers 11.0.0.1;'  /etc/dhcp/dhcpd.conf
sed -i '/option routers 11.0.0.1;/ a\option subnet-mask 255.255.255.0;' /etc/dhcp/dhcpd.conf
sed -i '/option subnet-mask 255.255.255.0;/ a\option time-offset -18000;' /etc/dhcp/dhcpd.conf
sed -i '/option time-offset -18000;/ a\range 11.0.0.1 11.0.0.254;' /etc/dhcp/dhcpd.conf
sed -i '/range 11.0.0.1 11.0.0.254;/ a\}' /etc/dhcp/dhcpd.conf
```

```shell
service isc-dhcp-server restart
```

```shell

chattr -i /etc/resolv.conf
sed -i '/nameserver/ i nameserver 11.0.0.1' /etc/resolv.conf
sed -i '/nameserver 11.0.0.1/ a\nameserver 192.168.0.1' /etc/resolv.conf
sed -i 's/search.*/search cloud.com ./' /etc/resolv.conf
chattr +i /etc/resolv.conf
```

```shell

chmod 775 -R /etc/bind
chown -R bind /etc/bind
service bind9 restart
service isc-dhcp-server restart
```

### NAT
```shell
apt-get install -y iptables-persistent
echo "1" > /proc/sys/net/ipv4/ip_forward
sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
iptables -t nat -A POSTROUTING -o enp0s8 -j MASQUERADE
iptables-save > /etc/iptables/rules.v4
```

### NFS
```shell
apt-get install -y nfs-kernel-server
```

```shell
echo "/export 11.0.0.0/24(rw,async,no_root_squash)" >> /etc/exports
mkdir -p /export
chmod 777 /export
exportfs -va
systemctl start nfs-kernel-server
systemctl enable nfs-kernel-server
mount 11.0.0.1:/export /export
```

### NTP
```shell
sed -i '/^restrict ::1$/a\ restrict 11.0.0.0 mask 255.255.255.0 nomodify notrap' /etc/ntp.conf
service ntp start
```

```shell
echo 'export MOUNT_PATH=/export' >> /etc/bash.bashrc
echo 'iptables -P FORWARD ACCEPT' >> /root/.bashrc
```
