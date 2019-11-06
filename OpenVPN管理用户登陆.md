工作中，如果oepnvpn使用人数达到一定量，这时候我们就需要做一些管控，让大家在有需要的时候申请使用vpn，然后再分配开启使用权限，这样可以再使用的时候一定程度上起到监控作用。
1.首先是如何删除用户

```powershell
cd /etc/openvpn/easy-rsa-3.0.6/easyrsa3/    #来到easyrsa目录下
./easyrsa revoke “用户名”
./easyrsa gen-crl    #执行这两步，可以删除指定用户，并生成文件crl.pem
crl-verify /etc/openvpn/easy-rsa-3.0.6/easyrsa3/pki/crl.pem  #然后在配置文件server.conf中加入这条，指定一下crl.pem的路径，就会在启动的时候去读取文件，禁止该用户登陆。
nohup /usr/local/openvpn/sbin/openvpn --config /etc/openvpn/server.conf &        #然后重启服务就生效了，我的server.conf是在/etc/openvpn下，大家根据实际情况来。
```
2.有了上面的理论，下面有我自己的一个笨方法来控制用户登陆
首先配置文件server.conf中添加：
crl-verify /etc/openvpn/easy-rsa-3.0.6/easyrsa3/pki/diaoxiao/yonghu/crl.pem
crl-verify /etc/openvpn/easy-rsa-3.0.6/easyrsa3/pki/diaoxiao/yonghu1/crl.pem
#这两行的意思是在pki目录下创建一个目录diaoxiao，专门用来放所有用户的删除后生成的文件crl.pem，对应的每个用户再单独创建目录，如yonghu和yonghu1，然后在下面放删除该用户后生成的crl.pem就好。
可能有点乱，以下是全部步骤：生成用户-->发送用户的证书等给对应用户保存-->删除用户证书，也就是上面写的删除方法--->每删除一个用户，就把生成的文件mv移动到对应用户目录，切记不要连续删除用户而忘记移动文件。
当然，如果有多个用户，就需要在配置文件中继续添加，然后默认不注释。
以下是我的脚本diaoxiao.sh：

```powershell
#!/bin/bash

usage(){
    cat <<EOF
模板:
    $0 [用户名] [开启on/关闭off]
例子:
    $0 wenyuan on
EOF
    exit 1
}

if [ $# -lt 2 ]; then
   echo "================================================== "
   echo "            操作错误，请看以下范例 "
   echo "================================================== "
   usage
   exit
fi
上面的是指引如何执行脚本的内容，如果是自己用可以不写。
process="pkill -9 openvpn"   #杀掉openvpn进程
#sed -i '19s/^#//' server.conf #取消注释
#sed -i '20s/^/#/' server.conf #开启注释
if [ $2 == on ];then    #$2是关闭或者开启，on/off
  case $1 in           #$1是用户名，指定对哪个用户操作
      yonghu)          #用户名
          sed -i '19s/^/#/' server.conf   #这里的sed是在配置文件中crl-verify这行开头加上注释，就可以开启指定用户的访问权限了，也就是不去读取该用户对应的那个文件了。19就是某用户所在行数。
          echo " 《用户》 已开启" 
      ;;
  esac
elif [ $2 == off ];then
  case $1 in
      yonghu)
          sed -i '19s/^#//' server.conf  #同上，这里是把#号取消掉，也就是关闭该用户的访问权限
          echo " 《用户》 已关闭"
      ;;
  esac
fi
$process   #执行杀死进程
sleep 2
echo "openvpn进程已杀死"
nohup /usr/local/openvpn/sbin/openvpn --config /etc/openvpn/server.conf &   #执行开启进程
sleep 1
echo "oepnvpn服务已开启"
```
有多个用户，就在case中继续添加就好。
记得chmod +x 添加权限。
之后执行放开用户和关闭用户的时候，如下操作：
./diaoxiao.sh  yonghu  on    开启
./diaoxiao.sh  yonghu  off   关闭

比较繁琐，而且随着人数增多，配置文件也会越写越多，所以是个笨方法，大家如果有好的方法欢迎分享。


