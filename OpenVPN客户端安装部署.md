
**

## Mac客户端VPN部署**
1.首先下载安装客户端开源软件Tunnelblick

```
   wget https://github.com/Tunnelblick/Tunnelblick/releases/download/v3.8.1beta02/Tunnelblick_3.8.1beta02_build_5390.dmg
```

   安装运行之后，打开Tunnelblick详情，然后将我们在服务端设置的客户端证书文件ca.crt  client.crt  client.key  ta.key移动到客户端上，可以通过scp命令移动到mac桌面。

   然后在mac创建文件夹，将移动过来的4个证书放进文件夹，然后创建一个以“ .ovpn ”结尾的文件，比如client.ovpn，然后将进行文件编写，如下：
   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190926190158883.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzY4NzUyNw==,size_1,color_FFFFFF,t_70)

按照这个标准模版。
**备注**：这里有一个很容易犯的错误，每行后面不要有空格，写完要检查一下，然后最下面也不要有空格，否则会报错。
写好之后，将文件保存，这一步就算做完了。

2. 开始使用Tunnelblick

     将上面写好的配置文件client.ovpn拉取到Tunnelblick的左侧窗口，然后点击右下角的连接，就可以开始使用VPN了。
3. 检测是否成功连接VPN

     可以先通过ping OpenVPN服务端的内网IP，检测是否可以通，然后ping服务端的内网其他机器，可以通过就是正常；然后通过“sudo ssh  root@内网ip”的方式，检测是否可以连接内网机器,可以的话就是正常的。 还可以先在浏览器打开百度，搜索“ip”，检查当前ip是否为服务端的公网ip。

4. 连接上之后，我们就相当于是以服务端机器的形式操控机器了，可以连接其他内网机器做相应工作。

## windows客户端VPN部署
如果是windows系统，首先下载客户端软件在openvpn官网，需要过墙，下载后，运行，记住安装目录，可以自定义安装路径。然后将5个证书拷贝到openvpn安装目录下的config文件夹内（这个文件夹内不能有其他不相关文件），然后右键桌面上vpn图标，以管理员身份运行，vpn会在右下角生成一个图标，双击开始连接。

## 进阶：
生产中，我们需要的是更加便捷和一键操作，以下是快速生产客户端脚本，满足了生成客户端证书、打包、发送到本机。

```
#!/bin/bash
#首先客户端会生成三个证书，我们先检测有没有该证书，以免犯错。
if [ -f /etc/openvpn/easy-rsa-3.0.6/easyrsa3/pki/private/${1}.key ];then
  echo “${1}.key 已存在”
  exit
fi
if [ -f /etc/openvpn/easy-rsa-3.0.6/easyrsa3/pki/issued/${1}.crt ];then
  echo “${1}.crt 已存在”
  exit
fi
if [ -f /etc/openvpn/easy-rsa-3.0.6/easyrsa3/pki/reqs/${1}.req ];then
   echo "${1}.req 已存在"
fi

cd /etc/openvpn/easy-rsa-3.0.6/easyrsa3
#生成客户端登陆用户名和签约证书
./easyrsa build-client-full $1 nopass
#在user目录下创建该用户的指定目录
cd /etc/openvpn/user
mkdir $1
#把需要的证书都拷贝过来
cp /etc/openvpn/easy-rsa-3.0.6/easyrsa3/pki/private/${1}.key $1
cp /etc/openvpn/easy-rsa-3.0.6/easyrsa3/pki/issued/${1}.crt $1
cp /etc/openvpn/easy-rsa-3.0.6/easyrsa3/pki/ca.crt $1
cp /etc/openvpn/easy-rsa-3.0.6/easyrsa3/ta.key $1
#修改用户配置文件
cp default.ovpn $1/${1}_nei.ovpn
echo "cert ${1}.crt" >> $1/${1}_nei.ovpn
echo "key ${1}.key" >> $1/${1}_nei.ovpn
#准备打包
cd /etc/openvpn/user/$1
/usr/bin/zip -q -r ${1}.zip *
scp ${1}.zip cq@192.168.4.208:/Users/cq/Desktop/OpenVPN

```
脚本中的客户端配置文件是提前放在服务端/etc/openvpn/user目录下的，并改名为default.ovpn，user也是自己创建的目录。其中配置文件中去除了客户端的两个证书参数，用于在脚本中加入。



