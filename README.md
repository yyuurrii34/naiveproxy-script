# NaiveProxy  一键配置脚本

```shell
wget -N --no-check-certificate https://raw.githubusercontent.com/blog-misaka/naiveproxy-script/main/naiveproxy.sh && bash naiveproxy.sh
```

# 手动搭建tuic v5


~~本来看着这些还是实验性功能，因为目前UI界面上没有version设置，通过配置文件文本模式编辑后的节点，如果用UI打开，会丢失部分配置信息，一般这种情况下我是不想折腾的，但是有几个人问起来了，也就结合以前写的TUIC部署步骤，简单写一下吧~~

目前已经可以UI界面编辑

折腾前需要准备的：~~最新版TF版的surge，并且订阅没有过期；~~ 有一个vps，有一个属于你的域名

**建立服务端**

进入ssh ，输入指令获取管理员权限

```
sudo -i
```

然后依次输入：

升级服务器

```
apt -y update
```

CentOS系统用这个命令

```
yum update -y
```

~~获取申请证书的certbot~~
不用了，直接用搭建naiveproxy时内建caddy申请的证书。
```
apt -y install wget certbot
```

建立服务器端的文件夹并进入该文件夹

```
mkdir /opt/tuic && cd /opt/tuic
```

获取服务器端程序：
~~因为最近作者一直在更新，版本更新比较快，所以去 [作者的库](https://github.com/EAimTY/tuic/releases)查看下最新版~~

直接通过命令和github发行档信息REST API获取最新版


X86

```
wget https://github.com/tuic-protocol/tuic/releases/latest/download/$(curl -s https://api.github.com/repos/tuic-protocol/tuic/releases/latest | grep -o '"tag_name": "[^"]*"' | sed 's/"tag_name": "//; s/"//')-x86_64-unknown-linux-gnu -O /opt/tuic/tuic-server
```

ARM

```
wget https://github.com/tuic-protocol/tuic/releases/latest/download/$(curl -s https://api.github.com/repos/tuic-protocol/tuic/releases/latest | grep -o '"tag_name": "[^"]*"' | sed 's/"tag_name": "//; s/"//')-aarch64-unknown-linux-gnu -O /opt/tuic/tuic-server
```

赋予服务器端程序权限：

X86

```
chmod +x /opt/tuic/tuic-server
```

ARM

```
chmod +x /opt/tuic/tuic-server
```

这里每一行是一条指令，输入后按回车等执行完再进行下一条命令

**建立服务器端配置：**



由于caddy申请的证书有权限，其它user访问不了。
搞权限很麻烦，可能影响caddy运行，干脆把这个证书复制出来：
```
uuid=$(uuidgen)
pass=$(openssl rand -base64 8)
firstdomain=$(ls ~/.local/share/caddy/certificates/acme-v02.api.letsencrypt.org-directory | head -n 1)
/bin/cp -f ~/.local/share/caddy/certificates/acme-v02.api.letsencrypt.org-directory/$firstdomain/$firstdomain.crt $firstdomain.crt && chmod 644 $firstdomain.crt
/bin/cp -f ~/.local/share/caddy/certificates/acme-v02.api.letsencrypt.org-directory/$firstdomain/$firstdomain.key $firstdomain.key && chmod 600 $firstdomain.key
```

创建tuic的配置文件config.json:

```
cat << EOF >config.json
{
    "server": "[::]:443",
    "users": {
        "$uuid": "$pass"
    },
    "certificate": "./$firstdomain.crt",
    "private_key": "./$firstdomain.key",
    "congestion_control": "bbr",
    "alpn": ["h3", "spdy/3.1"],
    "udp_relay_ipv6": true,
    "zero_rtt_handshake": false,
    "auth_timeout": "3s",
    "max_idle_time": "10s",
    "max_external_packet_size": 1500,
    "gc_interval": "3s",
    "gc_lifetime": "15s",
    "log_level": "warn"
}
EOF
```
**（查看配置文件记下uuid和pass）**

```
cat config.json
```

**新建systemd配置文件**

```
nano /lib/systemd/system/tuic.service
```

写入如下配置：

X86:

```
[Unit]
Description=Delicately-TUICed high-performance proxy built on top of the QUIC protocol
Documentation=https://github.com/EAimTY/tuic
After=network.target

[Service]
User=root
WorkingDirectory=/opt/tuic
ExecStart=/opt/tuic/tuic-server -c config.json
Restart=on-failure
RestartPreventExitStatus=1
RestartSec=5

[Install]
WantedBy=multi-user.target

```

ARM

```
[Unit]
Description=Delicately-TUICed high-performance proxy built on top of the QUIC protocol
Documentation=https://github.com/EAimTY/tuic
After=network.target

[Service]
User=root
WorkingDirectory=/opt/tuic
ExecStart=/opt/tuic/tuic-server -c config.json
Restart=on-failure
RestartPreventExitStatus=1
RestartSec=5

[Install]
WantedBy=multi-user.target

```

至此其实服务器端已经建立好了。如果你已经之前玩过trojan有证书的话就直接把证书放入到/opt/tuic 文件夹里按照上面的配置公钥命名为：fullchain.pem，私钥命名为：private.pem那么就已经完成了。如果没有的话就接着往下看，通过certbot申请证书吧

~~**申请证书：**~~
**用了caddy的证书，这一步省了**
caddy是个好建站工具，比nginx好用，自动申请证书。把很多事情都省了。
```
certbot certonly \
--standalone \
--agree-tos \
--no-eff-email \
--email example@Gmail.com \
-d your.com
```

这里注意就是要把整个指令先复制到其它文本编辑器里面，把里面的：example@gmail.com 换成你的邮箱，your.com 换成你的域名，换好后再复制到ssh app里面按下回车执行

将获得的证书放到服务器配置文件内的位置：（把里面的your.com换成你自己的域名）

```
cat /etc/letsencrypt/live/your.com/fullchain.pem > /opt/tuic/fullchain.pem
```

```
cat /etc/letsencrypt/live/your.com/privkey.pem > /opt/tuic/privkey.pem
```

注意上面是两条指令，分别执行。


**禁用caddy运行http/3(默认)，解绑443/udp端口**

```
sed -i '1i\{\
    servers {\
        protocols h1 h2\
    }\
}' /etc/caddy/Caddy*
systemctl restart caddy

```

**加载并启动tuic服务并设置开机自启：**

```
systemctl daemon-reload
systemctl enable --now tuic.service
```

至此服务器端的配置已经全部完成了。~~你在surge配置里面就可以按照老刘提供的格式进行节点设置了，如下示意配置格式，1.1.1.1换成你的vps的IP，端口就是上面config.json里面设置的端口，password后面就是里面设置的密码，sni后面就是你的域名，uuid就是config.json里面user部分，可以自己去通过相应工具生成。~~

**客户端的事，各显神通吧！**
windows下可以用v2rayN里集成的；
也可以去tuic的官方github项目下一个，下的程序是命令行程序，所以要自己做一个脚本运行它，可以静默运行，无窗口运行（cmd批处理的一些简单技巧，自己上网查）；
还可以把官方的程序注册成一个windows服务（稍高级的技巧，当然也可以上网查）。

android手机里推荐nekobox，苹果用小火箭。
当然有那种万能跨平台的客户端工具，比如clash。我觉得设置太复杂，没必要搞。我在路由器上弄的时候，从没用这些东西，路由器不就是个linux嘛，你都会架vps了，要啥可视化工具。


**至此服务器端就全部配置完成了。**

重启：

```
systemctl restart tuic
```

如果想查看服务器状态，用这个指令

```
systemctl status tuic
```

卸载：

```
systemctl stop tuic && systemctl disable --now tuic.service && rm -rf /opt/tuic
```


基本明白服务端的思路构架了吧，
caddy作为一个网站服务器，像nginx一样，监听在443/tcp端口，我们关闭了它默认的h3也就是QUIC服务，但不影响正常网页(伪装)。
caddy内集成了一个naiveproxy，我们可以正常使用naive协议。
caddy还自动帮域名申请了证书。
我们让tuic监听443/udp端口并模拟QUIC，两者完美兼容。

**补充个防火墙的事**
把443/udp端口开启，以firewalld为例（用什么防火墙随意iptables、firewalld、ufw都行）

```
firewall-cmd --add-port=443/udp
firewall-cmd --runtime-to-permanent
```




