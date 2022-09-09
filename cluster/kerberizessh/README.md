# Kerberize SSH


```shell

sed -i 's/UsePAM no/UsePAM yes/' /etc/ssh/sshd_config
echo 'GSSAPIAuthentication yes
GSSAPICleanupCredentials yes' >>/etc/ssh/sshd_config

if [ ! -f ~/.user ]; then
if [ ! -z $USER ]; then
if [ $USER != 'root' ]; then
touch ~/.user
mkdir ~/.ssh
ssh-keygen -q -N "" -t rsa -f ~/.ssh/id_rsa
cp ~/.ssh/id_rsa.pub ~/.ssh/authorized_keys
cat <<EOF >~/.ssh/config
Host *
UserKnownHostsFile /dev/null
StrictHostKeyChecking no
LogLevel quiet
EOF

      chmod 600 ~/.ssh/config
      echo "GSSAPIDelegateCredentials yes" >>~/.ssh/config
      echo "GSSAPIRenewalForcesRekey yes" >>~/.ssh/config

    ###Additional in future

    fi
fi
fi

service ssh restart
```
