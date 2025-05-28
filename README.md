
cdnfly5.1.13搭建教程

交流群

https://t.me/+4BadEc7Gcg1lMjJl

宝塔新建一个站点 添加域名

auth.cdnfly.cn	

monitor.cdnfly.cn

然后 bt.fikkey.com	域名	https://dl2.lotcdn.com/

 主控机host指向这些域名 

 主控安装命令只能centos7

 
 curl -fsSL https://github.com/LoveesYe/cdnflydadao/raw/main/cdnfly/v5.1.13/master/master.sh -o master.sh && chmod +x master.sh && ./master.sh --es-dir /home/es

 
节点安装命令只能centos7

curl -fsSL https://github.com/jasontuyou/cdnfly/raw/main/agent.sh -o agent.sh  && chmod +x agent.sh && ./agent.sh --master-ver v5.1.13 --master-ip 你的ip --es-ip 你的ip --es-pwd es密码

重启主控

supervisorctl -c /opt/cdnfly/master/conf/supervisord.conf restart all


重启节点

supervisorctl -c /opt/cdnfly/agent/conf/supervisord.conf restart all

手动修改节点里的主控域名


sed -i "s/MASTER_HOST.*/MASTER_HOST = \"www.baidu.com\"/g" /opt/cdnfly/agent/conf/config.py;
supervisorctl -c /opt/cdnfly/agent/conf/supervisord.conf restart all;

如何解除用户登录受限
如果出现IP受限，或者需要邮件验证码或者短信验证码，可以登录主控服务器，执行命令

eval  `grep MYSQL_PASS /opt/cdnfly/master/conf/config.py`;
eval  `grep MYSQL_IP /opt/cdnfly/master/conf/config.py`;
eval  `grep MYSQL_PORT /opt/cdnfly/master/conf/config.py`;
eval  `grep MYSQL_DB /opt/cdnfly/master/conf/config.py`;
eval  `grep MYSQL_USER /opt/cdnfly/master/conf/config.py`;
mysql -h$MYSQL_IP -u$MYSQL_USER -p$MYSQL_PASS -P$MYSQL_PORT  $MYSQL_DB  -e 'update user set white_ip="",login_captcha="" where name="admin" ';

节点使用中转服务器连接主控

有时候主控是国外的，节点是国内的，国内节点封海外，无法连接国外的主控服务器，这时候可以搭建一个中转服务器来中转连接。

登录中转服务器，执行

mkdir -p /data/sh/;
cat > /data/sh/iptables.sh <<'EOF'
master_ip="这里替换为主控ip"
local_ip=`ip ad | grep "inet " | grep -v 127.0.0.1 | head -n 1 | awk '{print $2}' | awk -F'/' '{print $1}'`
sysctl net.ipv4.ip_forward=1
iptables -t nat -A PREROUTING -p tcp --dport 88 -j DNAT --to-destination $master_ip:88
iptables -t nat -A POSTROUTING -p tcp  --dport 88 -j SNAT --to-source $local_ip
iptables -t nat -A PREROUTING -p tcp --dport 9200 -j DNAT --to-destination $master_ip:9200
iptables -t nat -A POSTROUTING -p tcp  --dport 9200 -j SNAT --to-source $local_ip
EOF
chmod +x /data/sh/iptables.sh;
/data/sh/iptables.sh;
echo "/data/sh/iptables.sh" >> /etc/rc.local;

注意：中转服务器需要与主控和节点都相通，且开通88和9200端口

修改节点配置
登录需要配置中转的节点服务器，执行如下命令


new_master_ip="这里替换为中转服务器IP";
sed -i "s/ES_IP =.*/ES_IP = \"$new_master_ip\"/" /opt/cdnfly/agent/conf/config.py;
sed -i "s/MASTER_IP.*/MASTER_IP = \"$new_master_ip\"/g" /opt/cdnfly/agent/conf/config.py;
sed -i "s/hosts:.*/hosts: [\"$new_master_ip:9200\"]/" /opt/cdnfly/agent/conf/filebeat.yml;
chattr -i /usr/local/openresty/nginx/conf/listen_80.conf /usr/local/openresty/nginx/conf/listen_other.conf ;
chattr -i /usr/local/openresty/nginx/conf/;
sed -i "s#http://.*:88#http://$new_master_ip:88#" /usr/local/openresty/nginx/conf/listen_80.conf /usr/local/openresty/nginx/conf/listen_other.conf ;
ps aux | grep [/]usr/local/openresty/nginx/sbin/nginx | awk '{print $2}'  | xargs kill -HUP ||  true;
supervisorctl -c /opt/cdnfly/agent/conf/supervisord.conf restart filebeat;
supervisorctl -c /opt/cdnfly/agent/conf/supervisord.conf restart agent;
supervisorctl -c /opt/cdnfly/agent/conf/supervisord.conf restart task;


如何更换Linux DNS

国内节点推荐使用：

echo "nameserver 223.5.5.5" > /etc/resolv.conf 
echo "nameserver 114.114.114.114" >> /etc/resolv.conf 

国外节点推荐使用：

echo "nameserver 8.8.8.8" > /etc/resolv.conf 
echo "nameserver 8.8.4.4" >> /etc/resolv.conf 

节点报空间不足如何处理

首先尝试清空访问日志，执行如下命令

bash <(curl -sSLk us.centos.bz/cdnfly/clean_log.sh) 

之后执行df -h命令查询分区/占用情况，如果还没有下降，再执行清空缓存的命令，如下：

bash <(curl -sSLk us.centos.bz/cdnfly/clean_cache.sh) 

主控后台没有监控数据怎么办
1. 首先检查主控的9200端口是否已经开放

可以登录节点，执行如下命令检查

curl -v 主控ip:9200

2. 检查主控硬盘
3. 
如果主控硬盘使用率超过90%，elasticsearch是不会处理新数据了的。这时候可以减少存储天数，或者扩大硬盘，或者初始化elasticsearch。
减少存储天数，是在系统管理-》系统设置，数据清理的网站访问日志 (ES)和节点监控数据 (ES)，可以改为1天，

或者初始化elasticsearch,http://doc.cdnfly.cn/FAQ.html#%E5%A6%82%E4%BD%95%E5%88%9D%E5%A7%8B%E5%8C%96elasticsearch

3. 检查filebeat日志


日志在节点的/var/log/filebeat/filebeat路径

4. 初始化elasticsearch

如果上面的都没有问题，尝试初始化elasticsearch


云端节点监控功能说明：

监控默认是使用云端服务器去请求CDN节点，因此要保持云端和CDN节点之间的网络畅通。另外如果是用宝塔面板，php不要安装bt_safe扩展，否则无法使用tcp类型监控；如果要用ping类型监控，还需要允许exec函数。
支持多节点监控（和官方一样），要添加其它监控节点，可以编辑config.php配置文件，根据里面的注释说明添加。

修改为你自身安装节点,或使用默认的github节点安装
/opt/cdnfly/master/panel/src/views/system/update/index.html


主控登录地址为: http://主控IP/



管理员账号和密码： admin/cdnfly
普通用户账号和密码： jason/cdnfly



服务器配置要求

主控
1.内存 - 因为主控安装有Elasticsearch，推荐16G及以上，如果网站访问量比较小，8G也行，至少4G。
2.硬盘 - 建议固态硬盘, 同样考虑访问日志大小，推荐100G及以上，量小的话都可以。
3.CPU - CPU至少2核
4.开放80 88 9200端口
节点

1.内存 - 至少2G及以上
2.硬盘 - 根据网站缓存的大小配置
3.CPU - Nginx主要是跑CPU，所以要想访问性能好，CPU尽量好点。
4.开放80 443 5000端口
系统
支持Centos-7---Ubuntu-16.04

官方最新公共
尊敬的cdnfly用户:
目前发现登录安全漏洞，需要及时按照如下方法来临时修复。找-个只有你知道的域名,这个域名用于管理员登录。
如的域名，不用带http://,路径为:系统管理--->系统设置--->用户相关，限制管理员只能从此域名登录


搬迁主控
注意：下面的迁移步骤不包括迁移elasticsearch的数据
1 备份旧主控数据
在旧主控执行如下命令开始备份（注意：备份前会停止旧主控的进程）

cd /root

curl http://us.centos.bz/cdnfly/backup_master.sh -o backup_master.sh

chmod +x backup_master.sh

./backup_master.sh


这时候将在目录/root下，打包生成cdn.sql.gz文件，请把这个文件传输到新主控的/root/目录下，可以使用scp命令，命令如下：

cd /root
scp cdn.sql.gz   root@新主控IP:/root/
2 在新机器安装好主控程序
首先登录cdnfly.cn，更新授权为新主控ip，并清空机器码
登录旧主控机器，执行如下命令查看版本:

grep VERSION_NAME /opt/cdnfly/master/conf/config.py

如下图，版本为v4.1.6：


登录新机器，执行如下命令安装:

curl http://dl.cdnfly.cn/cdnfly/master.sh -o master.sh

chmod +x master.sh

./master.sh --ver v4.1.60

其中v4.1.60替换成自己的主控版本号
3 登录新主控，恢复备份
执行如下命令恢复

cd /root
curl http://us.centos.bz/cdnfly/restore_master.sh -o restore_master.sh

chmod +x restore_master.sh

./restore_master.sh

从旧主控下载/opt/cdnfly/master/conf/config.py上传到新主控覆盖
然后在新主控初始化es,重启新主控
执行如下命令初始化:

cd /tmp

wget us.centos.bz/cdnfly/int_es.sh -O int_es.sh

chmod +x int_es.sh

./int_es.sh /home/es

supervisorctl restart all

其中/var/lib/elasticsearch为es的数据目录，可以更改成其它的，比如/home/es

4 替换节点里的主控IP
一个个登录节点，执行如下命令替换

new_master_ip="这里替换为新主控IP"
sed -i "s/ES_IP =.*/ES_IP = \"$new_master_ip\"/" /opt/cdnfly/agent/conf/config.py
sed -i "s/MASTER_IP.*/MASTER_IP = \"$new_master_ip\"/g" /opt/cdnfly/agent/conf/config.py
sed -i "s/hosts:.*/hosts: [\"$new_master_ip:9200\"]/" /opt/cdnfly/agent/conf/filebeat.yml
logs_path=`awk '/error_log/{print $2}'  /usr/local/openresty/nginx/conf/nginx.conf | sed 's/error.log//'`
if [[ `echo $logs_path | grep ^/ ` != ""  ]];then
    sed -i "s#.*access.log#    - $logs_path/access.log#" /opt/cdnfly/agent/conf/filebeat.yml
    sed -i "s#.*stream.log#    - $logs_path/stream.log#" /opt/cdnfly/agent/conf/filebeat.yml
fi
sed -i "s#http://.*:88#http://$new_master_ip:88#" /usr/local/openresty/nginx/conf/listen_80.conf /usr/local/openresty/nginx/conf/listen_other.conf 
ps aux | grep [/]usr/local/openresty/nginx/sbin/nginx | awk '{print $2}'  | xargs kill -HUP ||  true
supervisorctl restart filebeat
supervisorctl restart agent
supervisorctl restart task

主控更换ip后节点修改命令

new_master_ip="这里替换为主控IP"
(后台系统升级里查看es_pwd密码)
es_pwd="这里替换为es密码"

sed -i "s/ES_IP =.*/ES_IP = \"$new_master_ip\"/" /opt/cdnfly/agent/conf/config.py

sed -i "s/MASTER_IP.*/MASTER_IP = \"$new_master_ip\"/g" /opt/cdnfly/agent/conf/config.py

sed -i "s/hosts:.*/hosts: [\"$new_master_ip:9200\"]/" /opt/cdnfly/agent/conf/filebeat.yml

chattr -i /usr/local/openresty/nginx/conf/ /usr/local/openresty/nginx/conf/listen_80.conf /usr/local/openresty/nginx/conf/listen_other.conf

sed -i "s#http://.*:88#http://$new_master_ip:88#" /usr/local/openresty/nginx/conf/listen_80.conf /usr/local/openresty/nginx/conf/listen_other.conf

chattr +i /usr/local/openresty/nginx/conf/ /usr/local/openresty/nginx/conf/listen_80.conf /usr/local/openresty/nginx/conf/listen_other.conf

sed -i "s/ES_PWD =.*/ES_PWD = \"$es_pwd\"/" /opt/cdnfly/agent/conf/config.py

sed -i "s/password:.*/password: \"$es_pwd\"/" /opt/cdnfly/agent/conf/filebeat.yml

sed -i "s/agent-pwd:.*/agent-pwd: \"$es_pwd\"/" /opt/cdnfly/agent/conf/filebeat.yml

ps aux | grep [/]usr/local/openresty/nginx/sbin/nginx | awk '{print $2}'  | xargs kill -HUP ||  true

supervisorctl -c /opt/cdnfly/agent/conf/supervisord.conf restart filebeat

supervisorctl -c /opt/cdnfly/agent/conf/supervisord.conf restart agent

supervisorctl -c /opt/cdnfly/agent/conf/supervisord.conf restart task
