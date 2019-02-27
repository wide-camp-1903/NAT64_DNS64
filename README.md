# dns64とnat64の構築
dns64をunboundで立てます
nat64をjoolで立てます

環境は[ここ](https://www.webessentials.biz/parallelsdesktop/esxi/)を参考にして
Macbookの中にesxiを入れて検証した

client用 ubuntu16.04 gui

nat64用 ubuntu18.04 cui only  
/etc/netplan/50-cloud-init.yaml
```yaml:/etc/netplan/50-cloud-init.yaml
network:
    ethernets:
        ens160:
            dhcp4: true
            #dhcp4: no
            #addresses: 
            #    - 10.0.0.1/8
            #gateway4: 10.0.0.1
            #nameservers:
            #    addresses: [8.8.8.8, 8.8.4.4]
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


## 参考にした記事たち
https://jool.mx/en/run-nat64.html
https://blog.techlab-xe.net/archives/5269
https://www.webessentials.biz/parallelsdesktop/esxi/

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
