## 1.准备工具：

apt-get install keepalived mysql-server
pip版本要求为9.0.1 如果版本过低，先执行python –m install –upgrade pip
删除/usr/lib/python2.7/dist-packages/requests 
apt-get autoremove
which pip  ln –s xxx /usr/bin/pip

## 2.系统配置：
VIP=192.168.127.200
执行如下命令，进行ARP抑制以实现DR模式，否则无法进行负载均衡。
ifconfig lo:0 $VIP netmask 255.255.255.255 broadcast $VIP
/sbin/route add -host $VIP dev lo:0               
echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
sysctl -p >/dev/null 2>&1

## 3.搭建mysql双主集群
1.每台机器修改/etc/mysql/my.cnf文件，注意server-id一台为1一台为2，两台均以2为步数递增。
log-bin=mysql-bin
server-id=1
binlog-format=ROW
log-slave-updates = true
auto_increment_offset = 1
auto_increment_increment = 2

2.每台重启mysql服务，登录。执行show master status;

3.记录File和Position两项，每台执行命令
change master to master_host='ip',master_port=3306,master_user='jumpserver',master_password='password',master_log_file='mysql-bin.000006',master_log_pos=107;
完毕后 start slave;
show slave status \G
 
看到Slave_IO_Running和Slave_SQL_Running为yes证明双主集群搭建完毕。

## 4.搭建mysql-proxy实现对两台机器的代理

1.安装lua-5.3.4.tar.gz包，执行make linux && make install按提示安装需要的依赖。

2.安装mysql-proxy：

apt-get install mysql-proxy

3.默认启动方式

mysql-proxy --proxy-address=:3307 --admin-username=jumpserver --admin-password=123456 --proxy-backend-addresses=192.168.127.131:3306 --proxy-backend-addresses=192.168.127.130:3306 --admin-lua-script=/usr/share/mysql-proxy/admin.lua&

4.开启读写分离方式
mysql-proxy --proxy-address=:3307 --admin-username=jumpserver --admin-password=123456 --proxy-read-only-backend-addresses=192.168.127.131:3306 --proxy-backend-addresses=192.168.127.130:3306 --admin-lua-script=/usr/share/mysql-proxy/admin.lua --proxy-lua-script=/usr/share/mysql-proxy/rw-splitting.lua&

5.至此mysql双主+mysql-proxy集群搭建完成。

## 5.搭建jumpserver+keepalived

1.两台机器分别安装Jumpserver需要的python依赖包
pip install -r requirements.txt -i http://pypi.douban.com/simple
如果安装失败可能是pip版本过低，需要升级为pip9.0.1，升级方法参考链接
http://blog.csdn.net/sinat_21302587/article/details/74279098

2.选择一台主机进行install.py脚本执行，具体搭建参考jumpserver搭建文档。

3.登录单台测试成功

 
4.记录jumpserver.conf文件的key值，保证两台key值及数据库配置相同。
 
5.另一台配置好后启动Jumpserver:
./service.sh start
6.安装keepalived并配置
apt-get install keepalived
确保前面系统配置无误，输入ip a检查lo:0网卡ip
 

修改/etc/keepalived/keepalived.conf文件
 
配置好后重启keepalived进程，浏览器输入VIP：192.168.127.200测试
 


