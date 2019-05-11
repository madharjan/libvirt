
# Virtualization Setup

## Install Dependencies

```bash
curl http://retspen.github.io/libvirt-bootstrap.sh | sudo sh
apt-get install git python-pip python-libvirt python-libxml2 novnc supervisor nginx
```

## Install webvirtmgr

```bash
mkdir -p /var/www/
cd /var/www/

git clone http://github.com/retspen/webvirtmgr.git

cd webvirtmgr
pip install -r requirements.txt
./manage.py syncdb
./manage.py collectstatic

chown -R www-data:www-data /var/www/webvirtmgr

```

`sudo vi /etc/supervisor/conf.d/webvirtmgr.conf`

```txt
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

`sudo vi /var/www/webvirtmgr/conf/gunicorn.conf.py`

```txt
...
...
workers = 8 #get_workers()
...

```bash
service supervisor stop
service supervisor start
```

## Configure Nginx

`sudo vi /etc/nginx/conf.d/virtmgr.conf`

```txt
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

```bash
service nginx stop
service nginx start
```

## Setup TCP authorization & Verify

Set password for user admin

```bash
saslpasswd2 -a libvirt admin
```

## Libvirtd Configuration

`vi /etc/libvirt/libvirtd.conf`

```txt
listen_tls = 0
listen_tcp = 1
listen_addr = "0.0.0.0"
```

## Systemd Configuration

`systemctl edit libvirt-bin`

```txt
[Service]
Environment=LIBVIRTD_ARGS=-l KRB5_KTNAME=/etc/libvirt/libvirt.keytab
```

Verty authorization for admin

```bash
virsh -c qemu+tcp://localhost/system nodeinfo
```

## Edit virtual bridge network

`virsh edit-network default`

```txt
...
  <ip address='192.168.1.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.1.200' end='192.168.1.254'/>
    </dhcp>
  </ip>
...
```

## Restart libvirt-bin

```bash
service libvirt-bin restart
```

## WebVirt Manager Config

### General

- Browser to `http://virtmgr.local/servers/`
- Add Connection

```txt
TCP Connections
  Label : ubuntu
  FQDN / IP : localhost
  Username : admin
  Password : [password]

```

### Storage

- Browser to `http://virtmgr.local/storages/1/`

```txt
ISO

  Name : iso
  Path :/var/www/webvirtmgr/images

DIR
  Type : dir
  Name : default
  Path : /var/lib/libvirt/images

```

## Base Ubuntu VM

### Upload ISO

- Browser to `http://virtmgr.local/storage/1/iso/`
- Download `http://mirror.nus.edu.sg/ubuntu-ISO/16.04.5/ubuntu-16.04.5-server-amd64.iso`
- Upload `ubuntu-16.04.5-server-amd64.iso`

### Create VM

- Browser to `http://virtmgr.local/storage/1/default/`

```txt
Add New Image
  Name : ubuntu-16.04
  Format : qcow2
  Size : 15 GB
  Metadata: checked

```

- Browser to `http://virtmgr.local/create/1/`

```txt
Create Custom Instance
   Name : ubuntu-16.04
   VCPU : 1
   Host-Model: checked

   RAM : 512 MB
   HDD
   hdd0 : ubuntu-16.04.img
   HDD cache mode : Default
   Network
   eth0 -> default
   VirtIO : checked

```

Browser to `http://virtmgr.local/instance/1/ubuntu-16.04/#instancemedia`

```txt
Media
  CDROM 1 : ubuntu-16.04.5-server-amd64.iso (connect)
```

- Browser to  Power/Start
- Power ->  Start

- Browser to  Access/Console
- Console -> console

### Ubuntu Installation

```txt
  Language :  English
  Install Ubuntu Server
  Select a Language: English
  Select your location: Singapore
  Configure the keyboard:
    Auto:   No
    Language: English (US)
    Layout: English (US)
  Configure the network:
    Hostname: ubuntu
    Fullname: ubuntu
    Username: ubuntu
    Password: ubuntu
    Re-enter Password: ubuntu
    Use weak password: Yes
    Encrypt your home directory: No
  Partition disks
    Method: Guided - use entire disk and setup LVM
    Disk: Virtual disk 1 (vda) - 16.1 GB Virtio Block Device
    Write the changes to disks and configure LVM: Yes
    Amount of volume: 15.3 GB
    Write the changes to disk: Yes
  Configure the packge manager
    HTTP proxy: <blank>
  Configuring tasksel
    Manage upgrade: No automatic updates
  Software selection
    install:
      * standard system utilities
      * OpenSSH Server
  Install the GRUB: Yes
  Finish Install: Continue
```

### Ubuntu Customization

- SSH to VM

```bash
cat <<EOT | sudo tee -a /etc/systemd/timesyncd.conf
NTP=192.168.1.1
EOT
sudo systemctl restart systemd-timesyncd
sudo systemctl status systemd-timesyncd

sudo apt-get remove -y --purge at snapd lxcfs accountsservice mdadm policykit-1 open-iscsi
sudo apt update
sudo apt upgrade -y

sudo apt-get autoremove -y
sudo apt-get clean
sudo rm -rf /tmp/* /var/tmp/*
sudo rm -rf /var/lib/apt/lists/*
sudo rm -f /etc/dpkg/dpkg.cfg.d/02apt-speedup
sudo rm -rf /usr/share/doc/*
```

### Ubuntu VM by duplication

- Browse to `http://virtmgr.local/instance/1/ubuntu-16.04/`

```txt
Settings -> Clone ->  

  Clone Name  - <new-name>
  Click on  Random MAC Address
  Storage devices
  vda (default)->  <new-name>

  Click on Clone

Browse to VCPUS'S and Memory

  Description and Autostart
       Autostart  - Enable
       Description - None

    Logical host CPUs:
       Current allocation  : 2
       Maximum allocation : 2

  Total host memory:
       Current allocation (MB) : 1024
       Maximum allocation (MB) : 1024

     Click on Change

Browse to Power
       Start -> Click on Start

Browse to Access
    Console -> Click on Console

```