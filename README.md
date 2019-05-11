
# LibVirt Virtualization

Poor's Man Virtualization

- Support Cloud Image (Ubuntu, CentOS)
- Networking (Bridge, NAT, Isolated)
- Web Based Administration using WebVirtMgr [http://github.com/retspen/webvirtmgr](http://github.com/retspen/webvirtmgr)
- NoVNC Remote Connection

## Server Setup

| Hostname     | IP            |
|--------------|---------------|
| ubuntu       | 192.168.0.2   |

## Host File

Add Following to Host File (/etc/hosts)

```txt
192.168.0.2 ubuntu
192.168.0.2 virtmgr.local  # Virtualization console
```

## Virtualization Console

| URL                                              | Username | Password  |
|--------------------------------------------------|----------|-----------|
| [http://virtmgr.local](http://virtmgr.local)     | admin    |[password] |

Virtualization Setup [Details](virtualization-setup.md)

## SMTP Configuration

| Server        | Port   |
|---------------|--------|
| smtp-server   | 25     |

SMTP Setup [Details](smtp-setup.md)

Note: `smtp-server` will resolve to IP of Virtualization Host

## Guest VMs Network

Isolated Network : 192.168.1.0/24

## Guest VMs

Upload Cloud Image

| VM           | Username   | Password  |
|--------------|------------|-----------|
| ubuntu-14.04 | ubuntu     |[password] |
| centos-6.6   | root       |[password] |

To create new VM, clone VM with **new** MAC address

Note: For CentOS

- Update 'eth0' MAC on udev /etc/udev/ruled.s/70-persistent-net.rules
- Update 'eth0' MAC on /etc/sysconfig/network-scripts/ifcfg-eth0
- Reboot
