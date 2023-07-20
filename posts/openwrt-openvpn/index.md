# Openwrt下使用TOTP动态验证码连接Openvpn


# 背景

在家办公需要使用 openvpn 连接公司内网，而且密码使用  固定密码 +TOTP 验证码 的形式，每次都需要打开电脑的openvpn客户端，打开验证码客户端复制验证码，连接，只能说繁琐。希望通过家里的软路由openwrt的openvpn客户端来实现上述操作的一键连接，实现家里每个终端设备都能连接公司内网。

<!--more-->

# 实操

确保你的openwrt服务列表里已经安装openwrt，如没有请自行安装，在openwrt下新建 /root/openvpn 目录，一切操作均在这个目录下进行。

## （1）openwrt安装oathtool 

下载连接： https://downloads.openwrt.org/releases/packages-21.02/x86_64/packages/oath-toolkit_2.6.5-1_x86_64.ipk

下载上传至openwrt，安装 `opkg install  /tmp/upload/oath-toolkit_2.6.5-1_x86_64.ipk`

验证： `oathtool -b --totp '你的密钥'`  和谷歌身份验证器里的验证码此时是否一致。

## （2）生成账户密码

在/root/openvpn下新建`generate-password.sh` 脚本，内容如下，

```bash
yzm=$(/usr/bin/oathtool -b --totp '你的密钥')

echo -e "公司openvpn账号\n公司openvpn固定密码${yzm}" > /root/openvpn/userpass.txt

```

## （3）openvpn配置文件

上传`xxx.crt`、`xxx.key`、`xxx.ovpn`等文件到openwrt，并移动到/root/openvpn 目录下，修改ovpn文件。

auth-user-pass此项修改位为 `auth-user-pass userpass.txt`，从文件中读取用户名密码。

为了使指定的ip段使用openvpn，而其他的网络走路由默认线路，在ovpn文件下追加`route-nopull`及其下面的内容。

其中 10.87.1.0、10.88.1.0 修改为需要走openvpn的ip段，这样 10.87.1.1~10.87.1.254 、10.88.1.1~10.88.1.254 就会走openvpn，根据自己需要修改。

```bash
client
dev tun
link-mtu 1500
proto tcp
remote xxx.com 8888
resolv-retry infinite
nobind
persist-key
persist-tun
ca /root/openvpn/xxx.crt
remote-cert-tls server
tls-auth /root/openvpn/xxx.key 1
cipher AES-256-CBC
auth-user-pass /root/openvpn/userpass.txt
verb 3
auth-nocache
reneg-sec 36000

route-nopull
pull-filter ignore "dhcp-option DNS"

route 10.87.1.0 255.255.255.0 vpn_gateway
route 10.88.1.0 255.255.255.0 vpn_gateway
```

## （4）openwrt配置修改

1. 修改 /etc/config/openvpn ，在其下追加自己的openvpn配置，如下所示，名字根据需要修改。

   ```
   config openvpn '公司vpn'
           option config '/root/openvpn/xxx.ovpn'
           option auth_nocache '1'
   ```

2. 为了在启动时先生成密码文件userpass.txt，修改 /etc/init.d/openvpn 脚本，在 `openvpn_add_instance()` 函数下第一行加上

    `sh /root/openvpn/generate-password.sh`，如下所示。

   ```bash
   openvpn_add_instance() {
           sh /root/openvpn/generate-password.sh
           local name="$1"
           local dir="$2"
           local conf="$3"
           local security="$4"
           local up="$5"
           local down="$6"
   		...
   }
   ```

3. 此时openwrt网页openvpn下就有了刚刚加的配置了，勾选启用，点击保存应用。

![image-20230720120756596](https://i2.wp.com/telegra.ph/file/0f4dea7f988b1a619fab3.png)

## （5）修改openwrt防火墙

上面的操作完成后，你会发现在openwrt里能ping通需要通过openvpn连接的ip段，但是连接路由的设备并不能连通，需要修改防火墙，在防火墙自定义规则里添加如下内容，其中`192.168.8.0/24`修改为你的路由网段，`10.87.1.0/24,10.88.1.0/24`修改为你需要通过openvpn连接的公司ip网段`10.87.1.*,10.88.1.*`,如果想要达到连接 `10.87.*.*`的效果，可以改成 `10.87.0.0/16` ，这方面的网络知识，请自行查阅相关资料。

注：我使用了openwrt的mosdns功能，公司的内网域名放到了mosdns自定义hosts里了。没使用这个功能可以通过网络-DHCP/DNS里额外的HOSTS文件来实现自定义域名。

```bash
iptables -I FORWARD -o tun0 -j ACCEPT
iptables -t nat -I POSTROUTING -s 192.168.8.0/24 -d 10.87.1.0/24,10.88.1.0/24 -o tun0 -j MASQUERADE
```



# 参考



[openwrt下的openvpn client实现 - PHPERZ中文资讯站](http://www.phperz.com/article/15/1226/177358.html)

[Soul Of Free Loop » 博客存档 » OpenWRT实现OpenVPN按ipset自动分流 (zohead.com)](https://m.zohead.com/archives/openwrt-openvpn-ipset/?wpmp_switcher=true)



---

> 作者: [hpkaiq](https://hpk.me)  
> URL: https://hpk.me/posts/openwrt-openvpn/  

