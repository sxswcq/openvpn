OpenVPN服务端搭建，必须首先确定合适的openvpn版本和easy-rsa版本。

系统平台：Centos7

OpenVPN版本：2.4.7

easy-rsa版本：3.0.6

安装部署大致步骤：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190926172533360.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzY4NzUyNw==,size_1,color_FFFFFF,t_70)
一、依赖环境安装

```
  yum -y install openssl openssl-devel pam pam-devel epel-release iptables-services gcc gcc-c++
```
 

OpenVPN软件包不在CentOS标准Yum源里，所以需要安装先通过yum安装epel源才能正常安装相关软件包！由于是C语言编写的，所以gcc要装。iptables用来做防火墙，不用firewalld。

二、软件下载

   下载OpenVPN的tar包，由于需要外网下载，可以先下载到本机，再传到相应机器上。

```
   wget https://swupdate.openvpn.org/community/releases/openvpn-2.4.7.tar.gz
   #把van换成vpn
```

  下载Easy-rsa

       wget  https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.6/EasyRSA-unix-v3.0.6.tgz

或者直接访问我的github下载，上面都有了。
**https://github.com/sxswcq/openvpn/tree/master**
三、安装服务端
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019110416065784.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzY4NzUyNw==,size_1,color_FFFFFF,t_70)
四、安装Easy-rsa，并制作证书    

        tar -xf EasyRSA-3.0.1.tgz 

       cp -r EasyRSA-3.0.1 /etc/openvpn/easy-rsa     #把整个文件拷贝过去并改名

       cd  /etc/openvpn/easy-rsa/ 

       cp vars.example vars       #如果没有这个文件，就自己新建一个

       vim vars  #制作证书信息

   填入如下内容：

            export EASYRSA_REQ_COUNTRY="CN"     国家
            export EASYRSA_REQ_PROVINCE="ZJ“     省份
            export EASYRSA_REQ_CITY="Hangzhou"     城市
            export EASYRSA_REQ_ORG="cq"      自己起名字
            export EASYRSA_REQ_EMAIL="cq@abc.com"     邮箱
            export EASYRSA_REQ_OU="cq"        自己起个名字

**生成CA证书信息**

```
 ./easyrsa init-pki #初始化 pki 相关目录
 ./easyrsa build-ca #生成 CA 根证书, 先自定义填写密码，记做密码1，     Common Name处写名字，名字随便起，记做名称1。如果不希望给CA证书增设密码，则可以在命令后面加上nopass参数。
```

**生成服务端证书和密钥**

```
 ./easyrsa build-server-full server nopass     #其中第一个server为证书名称，记做名称2.

```
**生成Diffie-Hellman算法需要的密钥文件**
```

./easyrsa gen-dh       #创建Diffie-Hellman，这可能得等一小会儿

```
**生成tls-auth这个key**
这个key主要用于防止DoS和TLS攻击，这一步是可选的。****

```
 /usr/local/openvpn/sbin/openvpn  --genkey  --secret  ta.key
```

到此，服务端所需证书完毕，接下来生成客户端需要的证书和密钥。



**生成客户端登陆用户名和密码**

```
./easyrsa gen-req client    #其中client是自定义用户名，记做名称3；需要输入一个密码，记做密码2。也可以加参数nopass免去密码。
```


**生成签约证书**

```
./easyrsa sign client client    #后面的client为名称3.
```
         

  到此为止，我们需要的证书文件都生成完毕，分别为：

  服务端：ca.crt  server.crt  server.key  dh.pem  ta.key

  客户端： ca.crt  client.crt  client.key  ta.key

五、制定服务端配置文件server.conf

  5.1  在/etc/openvpn下新建配置文件server.conf ，填入如下内容：

      

    port 443 #监听都端口
    proto udp #服务度用的协议，建议udp
    dev tun
    #以下是各个证书文件所在绝对路径
    ca /etc/openvpn/easy-rsa-3.0.6/easyrsa3/pki/ca.crt #ca证书
    cert /etc/openvpn/easy-rsa-3.0.6/easyrsa3/pki/issued/server.crt #服务器证书路径
    key /etc/openvpn/easy-rsa-3.0.6/easyrsa3/pki/private/server.key #服务器密钥路径
    dh /etc/openvpn/easy-rsa-3.0.6/easyrsa3/pki/dh.pem #算法密钥文件路径
    tls-auth /etc/openvpn/easy-rsa-3.0.6/easyrsa3/ta.key 0 #参数0可以省略，如果不省略，那么客户端配置相应都参数该配成1，如果省略，客户端就不需要配置。
    push "route 192.168.5.239 255.255.255.0" #定义路由策略，比如我这里是阿里云机器，vpc里面有两个网段，5网段和6网段，而我服务端是在5网段，如果需要客户端访问过来也可以访问5网段其他服务器和6网段机器，那么这里就要做路由，这两行就是做这个操作的。
    push "route 192.168.6.0 255.255.255.0"
    push "dhcp-option DNS 8.8.8.8" #默认都DNS服务器配置，也可以根据自己需要自定义
    server 10.8.0.0 255.255.255.0 #该网段为openvpn都虚拟网卡段，不要和内网网段冲突即可
    ifconfig-pool-persist ipp.txt
    keepalive 10 120
    persist-key
    persist-tun
    duplicate-cn #允许一个用户多个终端连接
    crl-verify /etc/openvpn/easy-rsa-3.0.6/easyrsa3/pki/crl.pem    #删除用户后，需要加上这个参数，确保每次都会检查是否是可用的用户
    client-connect /etc/openvpn/vpn         #下面有讲
    client-disconnect /etc/openvpn/vpnd     #下面有讲
    script-security 2    #添加用户访问日志，就需要加这个参数
    status /var/log/openvpn/openvpn-status.log #状态日志文件
    log /var/log/openvpn/openvpn.log #指定日志文件路径
    log-append /var/log/openvpn/openvpn.log
    verb 3
    explicit-exit-notify 1
            
   


  5.2  定义好配置文件之后，选择禁用Centos7的firewalld，使用经典的iptables防火墙管理软件。


    systemctl  stop  firewalld

    systemctl  mask  firewalld    #注销服务

  5.3  禁用SELinux

    sentenforce 0    #立刻生效

    sed -i ‘s/SELINUX=enforcing/SELINUX=disabled/g’ /etc/selinux/config    #永久关闭，重启生效

   5.4  启用iptables防火墙,添加规则
         

```
systemctl  enable iptables
systemctl   start  iptables
iptables    -F                  #清理所有防火墙规则，根据自己情况定
iptables -t -nat  -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j SNAT --to-source 192.168.5.239              #根据自己情况定
iptables -t -nat  -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j SNAT --to-source 192.168.6.0
iptables-save > /etc/sysconfig/iptables # iptables       #规则持久化保存
systemctl  restart   iptables     #重启防火墙
systemctl   status   iptables     #查看防火墙状态
```

   5.5   Linux服务器启用地址转发

    echo  net.ipv4.ip_forward = 1 >> /etc/sysctl.conf

    sysctl  -p     #让配置立即生效

   5.6   启动OpenVPN

    nohup /usr/local/openvpn/sbin/openvpn --config /etc/openvpn/server.conf &            #执行之后可以ps -ef查看是否正常启动



  **至此，服务端的部署配置结束。**

## **备注：**

然后我们看一下/etc/openvpn/vpn和/etc/openvpn/vpnd这两个文件是做什么的：比如我们管理员需要了解每天谁登陆过，又是什么时候登陆什么时候下线的，登陆ip是什么，那么就需要这两个脚本文件来写日志。脚本内容如下。
**vqn文件：**

```
#!/bin/bash
day=`date +%F`
if [ -f /var/log/openvpn/log$day ];then
echo "`date '+%F %H:%M:%S'` User $common_name IP $trusted_ip is logged in" >> /var/log/openvpn/log$day
else
touch /var/log/openvpn/log$day
echo "`date '+%F %H:%M:%S'` User $common_name IP $trusted_ip is logged in" >> /var/log/openvpn/log$day
fi
```
**vqnd文件：**

```
#!/bin/bash
day=`date +%F`
if [ -f /var/log/openvpn/log$day ];then
echo "`date '+%F %H:%M:%S'` User $common_name IP $trusted_ip is logged off" >>/var/log/openvpn/log$day
else
touch /var/log/openvpn/log$day
echo "`date '+%F %H:%M:%S'` User $common_name IP $trusted_ip is logged off" >>/var/log/openvpn/log$day
fi
```
配置文件中还定义了该参数crl-verify，检测该用户是否被删除。工作中如果有离职员工，需要我们主动收回vpn权限，那么可以直接删除对他的授权就好，具体如下：
首先在/etc/openvpn/easy-rsa-3.0.6/easyrsa3  路径下执行删除命令：

```
./easyrsa revoke username      #删除用户username
./easyrsa gen-crl              #生成和写入crl.pem
```
然后重启服务就好了。


