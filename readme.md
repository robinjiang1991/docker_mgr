### 安装Mysql（5.7）
1.1 通过Docker安装
```
# 建立my-net docker 网络，用于各容器间的互联
docker network create -d bridge my-net
# 拉取mysql镜像
docker pull mysql:5.7
# 下面命令123456为数据库密码，自行修改
docker run --network my-net --name mysql5.7 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7
```

1.2 根据JIRA要求修改mysql配置
[Connecting Jira applications to MySQL 5.7](https://confluence.atlassian.com/adminjiraserver/connecting-jira-applications-to-mysql-5-7-966063305.html#ConnectingJiraapplicationstoMySQL5.7-configuringmysql)，具体操作如下：
查看容器ID根据容器ID进入Docker容器

```
[root@localhost ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
bb3b1dc59b3c        mysql:5.7           "docker-entrypoint..."   About an hour ago   Up 13 minutes       3306/tcp, 33060/tcp      mysql5.7
```



执行以下命令进行配置修改

```ruby
docker cp mysql5.7:/etc/mysql/mysql.conf.d/mysqld.cnf /root/ # 将容器中配置文件拷出宿主机进行修改
vim /root/mysqld.cnf
```

在mysqld.cnf 最后增加以下内容

```csharp
# Jira
default-storage-engine=INNODB
character_set_server=utf8mb4
innodb_default_row_format=DYNAMIC
innodb_large_prefix=ON
innodb_file_format=Barracuda
innodb_log_file_size=2G
#Confluence
collation-server=utf8mb4_bin
max_allowed_packet=256M
transaction-isolation=READ-COMMITTED
binlog_format=row
```

保存返回，拷进容器原位置

```ruby
docker cp /root/mysqld.cnf mysql5.7:/etc/mysql/mysql.conf.d/
```

根据容器ID重启mysql后重新进入容器

```csharp
[root@localhost ~]# docker restart bb3b1dc59b3c
[root@localhost ~]# docker exec -it bb3b1dc59b3c /bin/bash
```

```csharp
mysql -uroot -p123456
# 若不安装JIRA可忽略
CREATE DATABASE jiradb CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,DROP,REFERENCES,ALTER,INDEX on jiradb.* TO 'jira'@'%' IDENTIFIED BY 'jira';
GRANT ALL PRIVILEGES ON jiradb.* TO 'jira'@'%' IDENTIFIED BY 'jira';
# 若不安装Confluence可忽略
CREATE DATABASE confluencedb CHARACTER SET utf8 COLLATE utf8_bin;
GRANT ALL PRIVILEGES ON confluencedb.* TO 'confluence'@'%' IDENTIFIED BY 'confluence';
flush privileges;
```



### 安装JIRA（8.4.0)
2.1 编写Dockerfile文件
```
#截至2019年9月11日，最新版本为8.4.0，后期出现新版本可指定8.4.0进行安装。
FROM cptactionhank/atlassian-jira-software:latest

USER root

# 将代理破解包加入容器
COPY "atlassian-agent.jar" /opt/atlassian/jira/

# 设置启动加载代理包
RUN echo 'export CATALINA_OPTS="-javaagent:/opt/atlassian/jira/atlassian-agent.jar ${CATALINA_OPTS}"' >> /opt/atlassian/jira/bin/setenv.sh

```
2.2 下载atlassian-agent.jar文件，放置在Dockerfile同目录下，例如：
```
--JIRA
  --Dockerfile
  --atlassian-agent.jar
```
 构建镜像，执行命令
```
 docker build -t jira/jira:8.4.0 .
```

 2.4 启动容器，执行命令：
```
[root@localhost JIRA]# docker run --detach --publish 8080:8080 --network my-net jira/jira:8.4.0
6fad2372fd58cd23bed937ba0b134a124e696c54a594c64af6cf41166c94c318
[root@localhost JIRA]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
6fad2372fd58        jira/jira:8.4.0     "/docker-entrypoin..."   4 seconds ago       Up 3 seconds        0.0.0.0:8080->8080/tcp   vibrant_spence
```

# 破解重点

1.复制服务器ID:BY9B-GWD1-1C78-K2DE
2.在本地存放"atlassian-agent.jar"的目录下执行命令，生成许可证
```
# 需替换邮箱（liangjiangji@dongriaf.com）、名称（J）、
# 访问地址（http://10.0.5.36）、服务器ID（B2CR-L5A6-WSC0-LVO9）
# 为你的信息

[root@localhost JIRA]# java -jar atlassian-agent.jar -d -m liangjiangji@dongriaf.com -n J -p jira -o http://10.0.5.36  -s B2CR-L5A6-WSC0-LVO9

```


3.安装Confluence
3.1 编写Dockerfile文件
#截至2019年9月11日，最新版本为8.4.0，后期出现新版本可指定8.4.0进行安装。
```
FROM cptactionhank/atlassian-confluence:latest 
USER root
```

# 将代理破解包加入容器
COPY "atlassian-agent.jar" /opt/atlassian/confluence/

# 设置启动加载代理包
RUN echo 'export CATALINA_OPTS="-javaagent:/opt/atlassian/confluence/atlassian-agent.jar ${CATALINA_OPTS}"' >> /opt/atlassian/confluence/bin/setenv.sh
3.2 下载atlassian-agent.jar文件（提取密码：88bq），放置在Dockerfile同目录下，例如：
--CONF
  --Dockerfile
  --atlassian-agent.jar

3.3 构建镜像，执行命令
``` 
docker build -t confluence/confluence:7.0.0 .
```
结果如下： 
3.4 启动容器，执行命令：
```
[root@localhost JIRA]# docker run --detach --publish 8090:8090 --network my-net confluence/confluence:7.0.0 
```
3.5 访问http://ip:8090，参照前面JIRA配置流程进行设置，安装过程可与JIRA关联
3.6 生成授权码
# 设置产品类型：-p conf， 详情可执行：java -jar atlassian-agent.jar 
```
java -jar atlassian-agent.jar -d -m liangjiangji@dongriaf.com -n j -p conf -o http://10.0.5.36 -s BLFI-DH5F-3QKA-1921
```
3.7进入Confluence容器，并新建/home/confluence/文件夹

```
docker exec -it b86712dbede9 /bin/bash
cd /home
mkdir confluence 
```