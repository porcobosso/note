# CDH安装回顾

## 准备：
- 安装JDK **all**<br>
**JDK版本 最好能到cloudera文档上参考**
```bash
tar zxvf jdk-8u144-linux-x64.tar.gz -C /lib/jvm/
echo 'export JAVA_HOME=/lib/jvm/jdk1.8.0_144' >> /etc/profile
echo 'export JRE_HOME=${JAVA_HOME}/jre' >> /etc/profile
echo 'export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib' >> /etc/profile
echo 'export PATH=${JAVA_HOME}/bin:$PATH' >> /etc/profile
source /etc/profile
# ***需要检查是否输入JDK1.8字样***
java -version
```
- 安装ntp ntpdate **all**
- `/etc/hosts`文件修改 **all**
```bash
# ***需要根据实际情况修改***
echo 192.168.1.103 node1>>/etc/hosts
echo 192.168.1.104 node2>>/etc/hosts
echo 192.168.1.107 node3>>/etc/hosts
```
- 配置无密码访问
```bash
# ***需要交互，一路键入回车即可***
ssh-keygen -t rsa -P ''
# **需要根据实际情况修改**
for a in {1..3}; do ssh root@node$a cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys; done
for a in {2..3}; do scp /root/.ssh/authorized_keys root@node$a:/root/.ssh/authorized_keys ; done
```
- 解压缩 cloudera-manager **all**
```bash
tar xvzf cloudera-manager*.tar.gz -C /opt/cloudera-manager
# 添加cdh用户
useradd --system --home=/opt/cloudera-manager/cm-5.12.1/run/cloudera-scm-server/ --no-create-home --shell=/bin/false --comment "Cloudera SCM User" cloudera-scm
# 修改master配置
sed -i 's/server_host=localhost/server_host=node0/g' /opt/cloudera-manager/cm-5.12.1/etc/cloudera-scm-agent/config.ini
mkdir -p /opt/cloudera/parcels
chown cloudera-scm:cloudera-scm /opt/cloudera/parcels
# 拷贝mysql连接jar包
cd /opt/cloudera-manager/cm-5.12.1/share/cmf/lib
cp /root/mysql-connector-java-5.1.38.jar .
cp /root/mysql-connector-java-5.1.38.jar /usr/share/java/mysql-connector-java.jar
```

## 正题
- node1上安装mysql，配置可远程访问
```sql
ALTER USER 'root'@'localhost' identified by 'mima';
flush privileges;
grant all privileges on *.* to 'root'@'%' identified by 'mima' with grant optio
n;
flush privileges;
create database hive DEFAULT CHARACTER SET utf8;
grant all on hive.* TO 'hive'@'%' IDENTIFIED BY 'hive';
create database oozie DEFAULT CHARACTER SET utf8;
grant all on oozie.* TO 'oozie'@'%' IDENTIFIED BY 'oozie';
create database hue DEFAULT CHARACTER SET utf8;
grant all on hue.* TO 'hue'@'%' IDENTIFIED BY 'hue';
exit;
```
- 初始化cloudera数据库
```bash
cd /opt/cloudera-manager/cm-5.12.1/share/cmf/schema/
./scm_prepare_database.sh -uroot -pmima mysql cm scm scm
```
- 启动manager
```bash
mkdir -p /opt/cloudera/parcel-repo
chown cloudera-scm:cloudera-scm /opt/cloudera/parcel-repo
cd /opt/cloudera/parcel-repo
cp /root/CDH* .
cp /root/manifest.json .
/opt/cloudera-manager/cm-5.12.1/etc/init.d/cloudera-scm-server start
```
**需要去 ./log/cloudera-scm-server/cloudera-scm-server.log 查看是否报错**<br>
**如果重新安装，请删除cloudera对应数据库**
- 启动agent
```bash
/opt/cloudera-manager/cm-5.12.1/etc/init.d/cloudera-scm-agent start
```
**需要去 ./log/cloudera-scm-agent/cloudera-scm-agent.log 查看是否报错**<br>

## 若干脚本
中途出错，可参考以下：
- 停止server,agent
- 删除/var/lib下集群相关文件夹
- 删除数据库中集群数据库
- 删除 percels percels-cache文件夹
- 删除日志
```bash
/opt/cloudera-manager/cm-5.12.1/etc/init.d/cloudera-scm-agent stop&/opt/cloudera-manager/cm-5.12.1/etc/init.d/cloudera-scm-server stop
rm -f /opt/cloudera-manager/cm-5.12.1/log/cloudera-scm-agent/*.log /opt/cloudera-manager/cm-5.12.1/log/cloudera-scm-server/*.log /opt/cloudera-manager/cm-5.12.1/lib/cloudera-scm-agent/cm_guid
```
-重新初始化数据库
-重新启动server agent