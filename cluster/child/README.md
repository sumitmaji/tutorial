### DNS

```shell
apt-get update
apt-get install -y net-tools ifupdown openssh-client openssh-server
```

```shell
cat > /etc/netplan/00-installer-config.yaml << EOF
network:
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: true
      optional: true
  version: 2
EOF
netplan generate
netplan apply
```

```shell
echo "send fqdn.fqdn \"$NODE_NAME\";" > /etc/dhcp/dhclient-enp0s3.conf
echo "send fqdn.encoded on;" >> /etc/dhcp/dhclient-enp0s3.conf
echo "send fqdn.server-update off;" >> /etc/dhcp/dhclient-enp0s3.conf
echo "also request fqdn.fqdn;" >> /etc/dhcp/dhclient-enp0s3.conf
```

```shell
chattr -i /etc/resolv.conf
sed -i '/nameserver/ i nameserver 11.0.0.1' /etc/resolv.conf
sed -i '/nameserver 11.0.0.1/ a\nameserver 192.168.0.1' /etc/resolv.conf
sed -i 's/serach.*/serach cloud.com ./' /etc/resolv.conf
chattr +i /etc/resolv.conf
```

### NTP

```shell
apt-get update
apt-get install -y sntp libopts25 ntp
```

```shell
sed -i 's/server 0.ubuntu.pool.ntp.org/server master.cloud.com/' /etc/ntp.conf
sed -i 's/server 1.ubuntu.pool.ntp.org//' /etc/ntp.conf
sed -i 's/server 2.ubuntu.pool.ntp.org//' /etc/ntp.conf
sed -i 's/server 3.ubuntu.pool.ntp.org//' /etc/ntp.conf
service ntp start
```

### NFS

```shell
apt-get install -y nfs-common
```

```shell
mkdir -p /export
sed -i '$a\master:/export    /export   nfs     defaults,_netdev,x-systemd.automount,timeo=14,retry=0        0 0' /etc/fstab
mount -a
```

```shell
echo 'export MOUNT_PATH=/export' >> /etc/bash.bashrc
echo 'iptables -P FORWARD ACCEPT' >> /root/.bashrc
```