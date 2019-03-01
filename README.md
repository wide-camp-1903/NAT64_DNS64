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
テスト環境をv6線に繋いで
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
nat64/DNS64用 ubuntu18.04 cui onlyの方を
v6線とインターネットに繋がる線にに繋いで
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
練習環境
YOUR_IPV4_ADDR=10.211.55.12  #適宜変更

##本番環境
YOUR_IPV4_ADDR=10.0.11.1  #適宜変更
/sbin/modprobe jool pool6=2001:200:0:ff43::/96
jool instance add "example" --netfilter  --pool6 2001:200:0:ff43::/96

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
  dns64-prefix: 2001:200:0:ff43::/96
  dns64-synthall: no
  #private-address: "0.0.0.0/0”
  #do-ip4: no
  interface: ::
  access-control: ::0/0 allow
  interface-automatic: yes
 
forward-zone:
 name: "."
 forward-addr: 203.178.156.130
 forward-addr: 203.178.156.131
```
### 再起動

```bash
sudo systemctl restart unbound.service
```

### 確認
DNS/NAT64マシンでこれを実行
```bash
$ dig ipv4.google.com AAAA @8.8.8.8

; <<>> DiG 9.11.3-1ubuntu1.5-Ubuntu <<>> ipv4.google.com AAAA @8.8.8.8
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 45251
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;ipv4.google.com.		IN	AAAA

;; ANSWER SECTION:
ipv4.google.com.	21599	IN	CNAME	ipv4.l.google.com.

;; AUTHORITY SECTION:
l.google.com.		59	IN	SOA	ns1.google.com. dns-admin.google.com. 235857667 900 900 1800 60

;; Query time: 83 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Wed Feb 27 08:44:19 UTC 2019
;; MSG SIZE  rcvd: 115


```

自身のIPを指定してdigを走らせる
```bash
$ dig ipv4.google.com AAAA @fc01:0:0:1::1

; <<>> DiG 9.11.3-1ubuntu1.5-Ubuntu <<>> ipv4.google.com AAAA @fc01:0:0:1::1
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 57039
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;ipv4.google.com.		IN	AAAA

;; ANSWER SECTION:
ipv4.google.com.	21573	IN	CNAME	ipv4.l.google.com.
ipv4.l.google.com.	273	IN	AAAA	64:ff9b::acd9:19ce

;; Query time: 0 msec
;; SERVER: fc01:0:0:1::1#53(fc01:0:0:1::1)
;; WHEN: Wed Feb 27 08:50:20 UTC 2019
;; MSG SIZE  rcvd: 93
```

## 全体を通してテスト
```bash
curl ipv4.google.com
# htmlっぽいのが帰って来ます
```

## 自動起動
```bash
sudo systemctl restart unbound.service

```