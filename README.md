# dns64とnat64の構築
dns64をunboundで立てます
nat64をjoolで立てます

環境は[ここ](https://www.webessentials.biz/parallelsdesktop/esxi/)を参考にして
Macbookの中にesxiを入れて検証した



client用 ubuntu16.04 gui
nat64/DNS64用 ubuntu18.04 cui only
の２種のVMを用意


## 参考にした記事たち
https://jool.mx/en/run-nat64.html
https://blog.techlab-xe.net/archives/5269
https://www.webessentials.biz/parallelsdesktop/esxi/


## VMの準備
nat64/DNS64用 ubuntu18.04 cui onlyの方で
```bash
auto lo
iface lo inet loopback

auto ens160
iface ens160 inet6 static
address fc01::1:20c:29ff:feb9:197b/64
netmask 64
gateway fe80::20c:29ff:fe6d:e7d6
dns-nameservers fc01:29ff:fe6d:e7d6
```
`/etc/netplan/50-cloud-init.yaml`
を編集して
`netplan apply` を実行
```yaml:/etc/netplan/50-cloud-init.yaml
network:
    ethernets:
        ens160:
            dhcp4: true
            dhcp6: no
        ens192:
            dhcp4: no
            dhcp6: no
            addresses: 
                - fc01:0:0:1::1/64
            gateway6: fc01:0:0:1::1
            nameservers: 
                addresses: 
                    - fc01:0:0:1::1
    version: 2
```

## install 手順
この手順に習って
[これ](https://jool.mx/en/install.html)


### 面倒なのでsuで操作します。
```bash
sudo su
```

### aptから
```bash
apt install gcc make
apt install linux-headers-$(uname -r)
apt install libnl-genl-3-dev

apt install libxtables-dev

apt install dkms
apt install git autoconf
apt install tar
```
### Joolの本体をcloneしてinstall
```bash
git clone https://github.com/NICMx/Jool.git
dkms install Jool/
cd Jool/
./autogen.sh
./configure
cd src/usr/
make
make install
```

## 立ち上げ

[ここ](https://jool.mx/en/run-nat64.html)を参考にした

```bash
YOUR_IPV4_ADDR=10.0.0.1  #適宜変更
modprobe jool pool6=64:ff9b/96 pool4=$YOUR_IPV4_ADDR
jool instance add "example" --iptables  --pool6 64:ff9b::/96
```

## ダウン
```bash
jool instance remove "example"
/sbin/modprobe -r jool
```
## DNS64 unbound install
[この記事](https://blog.techlab-xe.net/archives/5269)を参考にしました。
### install
これだけでできるらしい
```bash
sudo apt install unbound
```
でもping
が通らないのでこっちにしてください
### /etc/unbound/unbound.conf.d/dns64.confを編集
`interface: fc01:0:0:1::1` は適宜変更
```/etc/unbound/unbound.conf.d/dns64.conf
server:
  verbosity: 2
  pidfile: "/var/run/unbound.pid"
  use-syslog: yes
  module-config: "dns64 iterator"
  dns64-prefix: 64:ff9b::/96
  dns64-synthall: yes
  interface: fc01:0:0:1::1
  access-control: ::0/0 allow
  #interface-automatic: yes
 
forward-zone:
 name: "."
 forward-addr: 8.8.8.8
```
### 再起動

```bash
sudo systemctl restart unbound.service
```

### 確認
DNS/NAT64マシンでこれを実行
```bash
dig ipv4.google.com AAAA @8.8.8.8
```
```bash
dig ipv4.google.com AAAA @localhost
```
