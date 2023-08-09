---
author: "Fajrian"
title: "[VPN] Bypassing Corporate Firewall with SSTP VPN (Educational Purpose Only)"
date: "2023-05-12"
tags: [
    "vpn",
    "sstp vpn",
    "softether",
]
toc: true
---

### Why choose SSTP VPN
Currently, I work in the finance industry with a high level of security system, the network engineer implement DPI(Deep Packet Inspection) in the network architecture, Which allows them to block or restrict access to certain websites, applications, and services(including SSH). 

Working within an environment where Wi-Fi access comes with a maze of permissions and corporate firewalls only permit communication through ports 80 and 443. and entertainment platform such as youtube, spotify, and reddit. 

#### Prerequisites

- VPS with IP public
- Port TCP/443 is open

#### Upgrade & Install Depedencies
```sh
$ sudo apt-get update && sudo apt-get -y upgrade && sudo apt install -y wget build-essential nano tar
```

#### Download softether
```sh
$ wget "http://www.softether-download.com/files/softether/v4.34-9745-rtm-2020.04.05-tree/Linux/SoftEther_VPN_Server/64bit_-_Intel_x64_or_AMD64/softether-vpnserver-v4.34-9745-rtm-2020.04.05-linux-x64-64bit.tar.gz" -O softether-vpnserver-linux.tar.gz
```

#### Extract softether
```sh
$ tar -xvf softether-vpnserver-linux.tar.gz
```

#### Move directory
```sh
$ sudo mv vpnserver /usr/local
$ cd /usr/local/vpnserver
$ make
```

6. Change permission
```sh
$ chmod 600 * && chmod 700 vpnserver && chmod 700 vpncmd
```

buat user untuk limitasi permission
```sh
sudo adduser vpn --shell /usr/sbin/nologin
sudo setfacl -m user:vpn:rwx -R /usr/local/vpnserver
```

7. buat service systemd agar autostart ketika reboot
```sh
$ sudo vi /etc/systemd/system/softether-vpn.service
```

```yaml
[Unit]
Description=SoftEther VPN server
After=network-online.target
After=dbus.service

[Service]
Type=forking
User=vpn
Group=vpn
ExecStart=/usr/local/vpnserver/vpnserver start
ExecReload=/bin/kill -HUP $MAINPID
# Maximum number of restart is 5, and attempts for every 60 seconds
StartLimitBurst=5
StartLimitInterval=60

[Install]
WantedBy=multi-user.target
```

memberi izin ke binary untuk membuka port dibawah 1024
```sh
$ sudo setcap 'CAP_NET_BIND_SERVICE=+eip CAP_NET_RAW=+eip' /usr/local/vpnserver/vpnserver

$ sudo getcap /usr/local/vpnserver/vpnserver
```

8. reload dan start service
```sh
$ sudo systemctl daemon-reload && sudo systemctl enable --now softether-vpn.service
```

#### Create vpn user and vpn config
9. login ke vpncmd
```sh
$ bash /usr/local/vpnserver/vpncmd
```

```sh
1. Management of VPN Server or VPN Bridge
2. Management of VPN Client
3. Use of VPN Tools (certificate creation and Network Traffic Speed Test Tool)

pilih angka: 3
ketik: check ### system requirement checking 
ketik: exit
```

10. login kembali ke vpncmd, untuk membuat virtual hub
```sh
$ bash /usr/local/vpnserver/vpncmd

ketik angka: 1
ketik huruf: HubCreate VPN
ketik huruf: Hub VPN
ketik huruf: SecureNatEnable
```

11. login kembali ke vpncmd, untuk membuat user
```sh
$ bash /usr/local/vpnserver/vpncmd

ketik angka: 1
ketik huruf: Hub VPN
ketik huruf: UserCreate ajinha
ketik huruf: UserPasswordSet ajinha
```

12. relogin to vpncmd, then create a certificate
```sh
$ bash /usr/local/vpnserver/vpncmd

ketik angka: 1
ketik huruf: Hub VPN
ketik huruf: IPsecEnable

enable l2tp: yes
enable raw l2tp: yes
enable ipsec: yes
preshared key ipsec: <your_ipsec_pass>
default virtualhub: VPN

ketik huruf: ServerCertRegenerate 172.104.48.41 or sstp.ajinfajrian.id
ketik huruf: ServerCertGet ~/cert.cer
ketik huruf: SstpEnable yes
ketik huruf: ServerKeyGet ~/privatekey.key
ketik hurif: SetEnumDeny # for hardening

exit
```

#### Hardening
Forbidden access to the web dashboard
```sh
sudo systemctl stop softether-vpn.service
sudo sed -i 's/DisableJsonRpcWebApi false/DisableJsonRpcWebApi true/g' vpn_server.config
sudo systemctl start softether-vpn.service
```


##### Optional for mikrotik router sstp client
- create a new `cert.key` for mikrotik
```sh
$ bash /usr/local/vpnserver/vpncmd

type 1 (management vpn server)
Hostname of IP Address of Destination: <blank or 443 depends on your port config>
Specify Virtual Hub Name: VPN
Password: <your_password>

VPN Server> ServerKeyGet ~/privkey.key
```


#### Troubleshoot selinux for rhel family
```yaml
softether.service: Failed to locate executable /usr/local/vpnserver/vpnserver: Permission denied
SELinux is preventing /usr/lib/systemd/systemd from execute access on the file vpnserver.
```

```sh
chcon -R -t bin_t /usr/local/vpnserver/
semanage fcontext -a -t bin_t "/usr/local/vpnserver(/.*)?"
restorecon -r -v /usr/local/vpnserver/
```


## download softether client windows

### Source
- https://www.kangarif.net/2020/11/cara-install-softether.html
- https://protonvpn.com/blog/deep-packet-inspection