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