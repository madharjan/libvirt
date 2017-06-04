## SMTP & Proxy Setup

### SMTP configuration (postfix)

Install `postfix`
```
apt-get install postfix mailutils
```

Edit configuration

`vi /etc/postfix/main.cf`

```
...
mydestination = ubuntu, localhost.localdomain, localhost, local.host
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 192.168.1.0/24 172.17.0.0/16
inet_interfaces = localhost, 192.168.1.1, 172.17.42.1
inet_protocols = ipv4
...
```

Restart

```
service postfix restart
```

Test SMTP

```
echo "body of your email" | mail -s "This is a Subject" -a "From: [sender email]" [recipient email address]

```
