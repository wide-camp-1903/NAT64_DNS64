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
いい感じにIPv6を固定で割り当てておく、
DHCPやRAはout of scope

## install 手順
この手順に倣う
[これ](https://jool.mx/en/install.html)


### 面倒なのでsuで操作します。
```bash
sudo su
```

### aptから
```bash
apt install -y gcc make linux-headers-$(uname -r) libnl-genl-3-dev libxtables-dev dkms git autoconf tar
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
/sbin/modprobe jool pool6=64:ff9b::/96
jool instance add "example" --netfilter  --pool6 64:ff9b::/96
```

### ダウン
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
  dns64-synthall: no
  interface: ::0
  port: 53
  access-control: ::0/0 allow

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
sudo systemctl enable unbound.service
```

## joolの自動起動
`/etc/systemd/system/jool.service`
```/etc/systemd/system/jool.service
[Unit]
Description=Jool - NAT64
After=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
EnvironmentFile=/etc/jool.conf
ExecStart=/bin/sh -ec '\
    /sbin/modprobe jool; \
    /usr/local/bin/jool instance add "${JOOL_INSTANCE_NAME}" --iptables  --pool6 ${JOOL_IPV6_POOL}; \
    /sbin/ip6tables -t mangle -A PREROUTING -d ${JOOL_IPV6_POOL} -j JOOL --instance "${JOOL_INSTANCE_NAME}"; \
    /sbin/iptables  -t mangle -A PREROUTING -d ${JOOL_IPV4_ADDRESS} -p tcp --dport ${JOOL_NAPT_START}:${JOOL_NAPT_END} -j JOOL --instance "${JOOL_INSTANCE_NAME}"; \
    /sbin/iptables  -t mangle -A PREROUTING -d ${JOOL_IPV4_ADDRESS} -p udp --dport ${JOOL_NAPT_START}:${JOOL_NAPT_END} -j JOOL --instance "${JOOL_INSTANCE_NAME}"; \
    /sbin/iptables  -t mangle -A PREROUTING -d ${JOOL_IPV4_ADDRESS} -p icmp -j JOOL --instance "${JOOL_INSTANCE_NAME}"; \
    /sbin/ip6tables -A FORWARD -i ${INTERFACE_V6ONLY} -o ${INTERFACE_WAN} -s ${INTERNAL_IPv6} -d ::/0 -j ACCEPT; \
    /sbin/ip6tables -A FORWARD -i ${INTERFACE_WAN} -o ${INTERFACE_V6ONLY} -s ::/0 -d ${INTERNAL_IPv6} -j ACCEPT -m state --state ESTABLISHED,RELATED; \
    /sbin/ip6tables -t nat -A POSTROUTING -o ${INTERFACE_WAN} -s ${INTERNAL_IPv6} -d ::/0 -j MASQUERADE; \
'


ExecStop=/sbin/ip6tables -t mangle -F
ExecStop=/sbin/ip6tables -t nat -F
ExecStop=/sbin/ip6tables -t nat -F
ExecStop=/sbin/iptables  -t mangle -F
ExecStop=/usr/local/bin/jool instance remove "${JOOL_INSTANCE_NAME}"
ExecStop=/sbin/modprobe -r jool

[Install]
WantedBy=multi-user.target
```

``/etc/jool.conf``を作成し、以下の様に記述
```
JOOL_IPV4_ADDRESS="131.113.136.24"
JOOL_IPV6_POOL="64:ff9b::/96"
JOOL_INSTANCE_NAME="example"
JOOL_NAPT_START="61001"
JOOL_NAPT_END="65535"
INTERNAL_IPv6=2001:df0:eb:41ea:0:2600:0:1/96
INTERFACE_WAN=ens160
INTERFACE_V6ONLY=ens192
```
