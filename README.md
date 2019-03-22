> 我在原作者的基础上做了一点优化，如下：
> 1. 把 iptables.sh 的执行顺序换了下，以加快可用性。原先 USG 启动至少3分钟后才能翻，现在1分钟左右即可——主要是更新 chnlist 比较耗时
> 2. 建议在每一次设置完 ss server 后，把 iptables.sh 第4行前的#去掉，且改为SERVER_IP=你的 ss server IP
> 3. 依次执行的内容有细化，方便和我一样的小白:)

# ubnt-router-shadowsocks  
在 usg，edgerouter 等设备上安装 ss 的脚本，包括 dnsmasq 配置，以及 iptables，国内 ip 走直连，除此以外默认走代理。

# 说明

此脚本执行后将处理的任务：  

1. 要求填写服务器 ip，端口，密码，以及加密方式，然后生成 supervisord 配置文件。  

2. 自动配置 iptables 和 [dnsmasq-china-list](https://github.com/felixonmars/dnsmasq-china-list.git)

3. 将配置信息写入启动脚本 rc.local，默认重启后重写 iptables。

# 安装步骤以及要求  
以 USG 举例
1. 先设置 apt-get 的 source，然后安装必备的软件
[EdgeRouter-Add-other-Debian-packages-to-EdgeOS](https://help.ubnt.com/hc/en-us/articles/205202560-EdgeRouter-Add-other-Debian-packages-to-EdgeOS)  
```
configure
set system package repository stretch components 'main contrib non-free' 
set system package repository stretch distribution stretch
set system package repository stretch url http://http.us.debian.org/debian
commit ; save
sudo -i
apt-get update
apt-get install git wget supervisor
```
> 这里要注意下，建议将 http://http.us.debian.org/debian 替换为国内源（我在这里浪费很多时间等待）

> 源一[中科大]： http://ftp.cn.debian.org/debian/

> 源二[清华]： http://ftp2.cn.debian.org/debian/

> 更多参见 https://www.debian.org/mirror/list.zh-cn.html

2. 下载仓库文件并执行脚本  
```
git clone https://github.com/imMMX/ubnt-router-shadowsocks.git
```  
依次执行 
```
cd ubnt-router-shadowsocks
./install.sh
./dnsmasqchn.sh
./iptables.sh add_rules
cp tproxy/erl3/xt_TPROXY.ko /lib/modules/`uname -r`/extra/
depmod
modprobe xt_TPROXY
```
> 这里要注意下，**原作者install.sh中第15行aes-256-cf )需要修改为为aes-256-cfb )，否则选项6始终通不过**

> 另外，**我实测发现原作者 @imMMX 放在tproxy/usg/下的 tx_TPROXY 在 USG 下不能用，erl3 的反倒可以！**

> 再：网上说tproxy内核模块其实edgeos系统自带，在/lib/modules/3.10.20-UBNT/kernel/net/netfilter/nf_tproxy_core.ko ，但系统没有自动加载，需要自己modprobe —— 所以我理解modprobe nf_tproxy_core也可以，具体自行尝试

修改 /etc/rc.local，确保重启后 1.自动更新 iptabless 2.创建 supervisor 的日志目录（重启后消失） 3.加载 UDP 模块
```
mkdir -p /var/log/supervisor/
supervisord

sleep 10
/root/ubnt-router-shadowsocks/iptables.sh add_rules
wait
modprobe xt_TPROXY
```
按照提示填入服务器信息，等待脚本执行完毕后确认 /etc/dnsmasq.d/ 有相关文件，确认 iptables 写入成功

```
iptables -t nat -L
```

* iptables 样例  

```shell
Chain BYPASSLIST (1 references)
target     prot opt source               destination
RETURN     all  --  anywhere             123.123.123.123
RETURN     all  --  anywhere             0.0.0.0/8
RETURN     all  --  anywhere             10.0.0.0/8
RETURN     all  --  anywhere             127.0.0.0/8
RETURN     all  --  anywhere             link-local/16
RETURN     all  --  anywhere             172.16.0.0/12
RETURN     all  --  anywhere             192.168.0.0/16
RETURN     all  --  anywhere             base-address.mcast.net/4
RETURN     all  --  anywhere             240.0.0.0/4
RETURN     tcp  --  anywhere             anywhere             match-set chnlist dst
```

3. 安装 shadowsocks-libev  
架构是 mips 和 mips64 的可以去我这个仓库 [ubnt-mips-shadowsocks-libev](https://github.com/imMMX/ubnt-mips-shadowsocks-libev) 下载或者自己编译。解压缩后移动到 /usr/bin/ 中或者 /usr/local/bin/ 确保 ss-redir ss-tunnel 可以直接执行。不知道自己是什么架构的输入 uname -a 查看

```
uname -a
wget https://github.com/imMMX/ubnt-mips-shadowsocks-libev/releases/download/3.2.0/ss-mips64.zip
unzip ss-mips64.zip
cp ss-bin/bin/ss-redir ss-bin/bin/ss-tunnel /usr/bin/
```

4. 启动 supervisord  
```
supervisorctl
reload
```
