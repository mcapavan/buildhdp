# buildhdp
Quick commands to build cluster

#### Execute on all nodes

```bash
#setup OS:

echo "***************** unzip and wget"
yum install -y wget
yum install -y mlocate
updatedb

cp -p /etc/security/limits.conf /etc/security/limits.conf.ORIG

sed -i '$ a *    soft    nofile 65536' /etc/security/limits.conf
sed -i '$ a *    hard    nofile 65536' /etc/security/limits.conf
sed -i '$ a *    soft    nproc 65536' /etc/security/limits.conf
sed -i '$ a *    hard    nproc 65536' /etc/security/limits.conf

echo ******************" Installing ntp"
yum install -y ntp
systemctl enable ntpd
systemctl start ntpd

service ntpd start


echo "***************** Check if NTP service is active:"
systemctl status ntpd

service ntpd status

echo "***************** Disable SELinux running the following:"
setenforce 0

echo "*****************  Make a backup of the config file located in /etc/selinux"
cd /etc/selinux
cp -p config config.ORIG

sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

# Change the behaviour in the creation of new files
umask 0022
echo umask 0022 >> /etc/profile

echo "************** restart"
shutdown -r now
```

#### Login back to all nodes and run the below on all nodes

```bash
wget -nv http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.6.2.0/ambari.repo -O /etc/yum.repos.d/ambari.repo

yum install -y ambari-agent
sed -i 's/hostname=localhost/hostname=pchalla0.field.hortonworks.com/g' /etc/ambari-agent/conf/ambari-agent.ini
ambari-agent start
```

#### On Ambari Server node

```bash
yum install -y ambari-server
yum install -y mysql-connector-java
ambari-server setup --jdbc-db=mysql --jdbc-driver=/usr/share/java/mysql-connector-java.jar
```

#### On MySQL node, Install mysql and create Ambari, Hive and Oozie databases

```bash
wget http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm
rpm -ivh mysql-community-release-el7-5.noarch.rpm
ls -1 /etc/yum.repos.d/mysql-community*

yum install -y mysql-connector-java
yum install -y mysql-server
systemctl start mysqld
systemctl status mysqld

#secure installation
mysql_secure_installation

```

#### Once the password is configured for mysql root user id, run the below

```bash
mysql -u root -phadoop

create database hive_db;
GRANT ALL PRIVILEGES ON *.* TO hive_db;
GRANT ALL PRIVILEGES ON *.* TO 'hive'@'%' IDENTIFIED BY 'hadoop';
FLUSH PRIVILEGES;

CREATE DATABASE ambari_db;
GRANT ALL PRIVILEGES ON *.* TO ambari_db;
GRANT ALL PRIVILEGES ON *.* TO 'ambari'@'%' IDENTIFIED BY 'hadoop';
FLUSH PRIVILEGES;

CREATE DATABASE oozie_db;
GRANT ALL PRIVILEGES ON *.* TO oozie_db;
GRANT ALL PRIVILEGES ON *.* TO 'oozie'@'%' IDENTIFIED BY 'hadoop';
FLUSH PRIVILEGES;
```

#### On Ambari Server Node

```bash
yum install -y mysql
mysql -u ambari -phadoop -h pchalla1.field.hortonworks.com ambari_db

USE ambari;
SOURCE /var/lib/ambari-server/resources/Ambari-DDL-MySQL-CREATE.sql;
\q
```

```bash
ambari-server setup
```

```bash
echo "ambari.post.user.creation.hook.enabled=true" >> /etc/ambari-server/conf/ambari.properties
echo "ambari.post.user.creation.hook=/var/lib/ambari-server/resources/scripts/post-user-creation-hook.sh" >> /etc/ambari-server/conf/ambari.properties
ambari-server restart
```

#### Create sample tables

```bash
sudo -u hdfs hdfs dfs -mkdir /user/root
sudo -u hdfs hdfs dfs -chown root:hadoop /user/root
sudo -u hdfs hdfs dfs -mkdir /user/admin
sudo -u hdfs hdfs dfs -chown admin:hadoop /user/admin


cd /tmp
wget https://raw.githubusercontent.com/abajwa-hw/security-workshops/master/data/sample_07.csv
wget https://raw.githubusercontent.com/abajwa-hw/security-workshops/master/data/sample_08.csv

hive

CREATE TABLE sample_07 (
code string ,
description string ,
total_emp int ,
salary int )
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' STORED AS TextFile;
load data local inpath '/tmp/sample_07.csv' into table sample_07;

CREATE TABLE sample_08 (
code string ,
description string ,
total_emp int ,
salary int )
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' STORED AS TextFile;
load data local inpath '/tmp/sample_08.csv' into table sample_08;
```


