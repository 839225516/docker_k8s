#### 使用docker-compose部署mysql 
```shell
# 安装docker ,docker-compose
yum install docker-ce -y
yum install docker-compose -y
```

编写docker-compose文件
```shell
mkdir /data/dockerMysql/ && cd /data/dockerMysql/
mkdir -p  {db,conf,init}

cat > docker-compose.yml <<EOF
version: '3.3'

services:
  mysql:
    image: mysql:5.7
    restart: always
    environment:
      TZ: Asia/Shanghai
      MYSQL_USER: 'user'
      MYSQL_ROOT_PASSWORD: 'jlpayroot'
    ports:
      - '3306:3306'
    expose:
      - '3306'
    volumes:
      - "./db:/var/lib/mysql"
      - "./conf/my.cnf:/etc/my.cnf"
      #- "./init:/docker-entrypoint-initdb.d/"
EOF

cat > conf/my.cnf <<EOF
[mysqld]
user = mysql
default-storage-engine = INNODB
character-set-server = utf8

[mysql.server]
default-character-set = utf8
 
[mysqld_safe]
default-character-set = utf8

[client]
default-character-set=utf8

[mysql]
default-character-set=utf8
EOF

cat > init/init.sql <<EOF
use mysql;
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'test';
EOF
```

##### 启动数据库
```shell
# 拉镜像
docker-compose pull

# 启动mysql
docker-compose up -d

```


安装MariaDB-client
```shell
cat > /etc/yum.repos.d/MariaDB.repo <<EOF
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.2.4/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
EOF

yum install MariaDB-client
```