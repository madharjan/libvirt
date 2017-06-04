
## Virtualization Setup

### Install Dependencies
```
curl http://retspen.github.io/libvirt-bootstrap.sh | sudo sh
apt-get install git python-pip python-libvirt python-libxml2 novnc supervisor nginx
```

### Install webvirtmgr

```
mkdir -p /var/www/
cd /var/www/

git clone http://github.com/retspen/webvirtmgr.git

cd webvirtmgr
pip install -r requirements.txt
./manage.py syncdb
./manage.py collectstatic

chown -R www-data:www-data /var/www/webvirtmgr

```

`vi /etc/supervisor/conf.d/webvirtmgr.conf`

```
[program:webvirtmgr]
command=/usr/bin/python /var/www/webvirtmgr/manage.py run_gunicorn -c /var/www/webvirtmgr/conf/gunicorn.conf.py
directory=/var/www/webvirtmgr
autostart=true
autorestart=true
stdout_logfile=/var/log/supervisor/webvirtmgr.log
redirect_stderr=true
user=www-data

[program:webvirtmgr-console]
command=/usr/bin/python /var/www/webvirtmgr/console/webvirtmgr-console
directory=/var/www/webvirtmgr
autostart=true
autorestart=true
stdout_logfile=/var/log/supervisor/webvirtmgr-console.log
redirect_stderr=true
user=www-data
```

```
service supervisor stop
service supervisor start
```

### Configure Nginx

`vi /etc/nginx/conf.d/virtmgr.conf`

```
upstream virtmgr {
    server 127.0.0.1:8000 fail_timeout=0;
}

server {
    listen       80;
    server_name  virtmgr.local;

    access_log /var/log/nginx/virtmgr_access.log;

    location /static/ {
        root /var/www/webvirtmgr/webvirtmgr; # or /srv instead of /var
        expires max;
    }

    location / {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-for $proxy_add_x_forwarded_for;
        proxy_set_header Host $host:$server_port;
        proxy_set_header X-Forwarded-Proto $remote_addr;
        proxy_connect_timeout 600;
        proxy_read_timeout 600;
        proxy_send_timeout 600;
        client_max_body_size 1024M; # Set higher depending on your needs

        proxy_pass http://virtmgr;
    }
}
```

```
service nginx stop
service nginx start
```


### Setup TCP authorization & Verify

Set password for user admin

```
saslpasswd2 -a libvirt admin
```

### Libvirtd Configuration

`vi /etc/libvirt/libvirtd.conf`
```
listen_tls = 0
listen_tcp = 1
listen_addr = "0.0.0.0"
```

### Systemd Configuration

`systemctl edit libvirt-bin`
```
[Service]
Environment=LIBVIRTD_ARGS=-l KRB5_KTNAME=/etc/libvirt/libvirt.keytab
```

Verty authorization for admin

```
virsh -c qemu+tcp://localhost/system nodeinfo
```

### Edit virtual bridge network

`virsh edit-network default`
```
...
  <ip address='192.168.1.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.1.200' end='192.168.1.254'/>
    </dhcp>
  </ip>
...
```

### Restart libvirt-bin

```
service libvirt-bin restart
```
