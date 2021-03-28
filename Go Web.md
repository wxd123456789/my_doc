# 基本概念

**socket**

**webSocket**

**RPC**

**REST**



# 安全与加密

。。。



# 备份恢复

项目中的备份恢复的文件，流程？？？？容灾倒换的流程？？？？

备份mysql的数据、redis的数据、用户上传的文件、、、

文件同步的方式rsync：服务端开启、客户端同步（可以设置crontab，保持rsync每多长时间同步 ）

**mysql备份：**

MySQL的备份有两种方式：热备份和冷备份，热备份目前主要是采用master/slave方式（master/slave方式的同步目前主要用于数据库读写分离，也可以用于热备份数据）。冷备份的话就是数据有一定的延迟，但是可以保证该时间段之前的数据完整，例如有些时候可能我们的误操作引起了数据的丢失，那么master/slave模式是无法找回丢失数据的，但是通过冷备份可以部分恢复数据。

冷备份一般使用shell脚本来实现定时备份数据库，然后通过上面介绍rsync同步非本地机房的一台服务器。

下面这个是定时备份mysql的备份脚本，我们使用了mysqldump程序，这个命令可以把数据库导出到一个文件中。

```bash
#!/bin/bash

# 以下配置信息请自己修改
mysql_user="USER" #MySQL备份用户
mysql_password="PASSWORD" #MySQL备份用户的密码
mysql_host="localhost"
mysql_port="3306"
mysql_charset="utf8" #MySQL编码
backup_db_arr=("db1" "db2") #要备份的数据库名称，多个用空格分开隔开 如("db1" "db2" "db3")
backup_location=/var/www/mysql  #备份数据存放位置，末尾请不要带"/",此项可以保持默认，程序会自动创建文件夹
expire_backup_delete="ON" #是否开启过期备份删除 ON为开启 OFF为关闭
expire_days=3 #过期时间天数 默认为三天，此项只有在expire_backup_delete开启时有效

# 本行开始以下不需要修改
backup_time=`date +%Y%m%d%H%M`  #定义备份详细时间
backup_Ymd=`date +%Y-%m-%d` #定义备份目录中的年月日时间
backup_3ago=`date -d '3 days ago' +%Y-%m-%d` #3天之前的日期
backup_dir=$backup_location/$backup_Ymd  #备份文件夹全路径
welcome_msg="Welcome to use MySQL backup tools!" #欢迎语

# 判断MYSQL是否启动,mysql没有启动则备份退出
mysql_ps=`ps -ef |grep mysql |wc -l`
mysql_listen=`netstat -an |grep LISTEN |grep $mysql_port|wc -l`
if [ [$mysql_ps == 0] -o [$mysql_listen == 0] ]; then
        echo "ERROR:MySQL is not running! backup stop!"
        exit
else
        echo $welcome_msg
fi

# 连接到mysql数据库，无法连接则备份退出
mysql -h$mysql_host -P$mysql_port -u$mysql_user -p$mysql_password <<end
use mysql;
select host,user from user where user='root' and host='localhost';
exit
end

flag=`echo $?`
if [ $flag != "0" ]; then
        echo "ERROR:Can't connect mysql server! backup stop!"
        exit
else
        echo "MySQL connect ok! Please wait......"
        # 判断有没有定义备份的数据库，如果定义则开始备份，否则退出备份
        if [ "$backup_db_arr" != "" ];then
                #dbnames=$(cut -d ',' -f1-5 $backup_database)
                #echo "arr is (${backup_db_arr[@]})"
                for dbname in ${backup_db_arr[@]}
                do
                        echo "database $dbname backup start..."
                        `mkdir -p $backup_dir`
                        `mysqldump -h$mysql_host -P$mysql_port -u$mysql_user -p$mysql_password $dbname --default-character-set=$mysql_charset | gzip > $backup_dir/$dbname-$backup_time.sql.gz`
                        flag=`echo $?`
                        if [ $flag == "0" ];then
                                echo "database $dbname success backup to $backup_dir/$dbname-$backup_time.sql.gz"
                        else
                                echo "database $dbname backup fail!"
                        fi

                done
        else
                echo "ERROR:No database to backup! backup stop"
                exit
        fi
        # 如果开启了删除过期备份，则进行删除操作
        if [ "$expire_backup_delete" == "ON" -a  "$backup_location" != "" ];then
                 #`find $backup_location/ -type d -o -type f -ctime +$expire_days -exec rm -rf {} \;`
                 `find $backup_location/ -type d -mtime +$expire_days | xargs rm -rf`
                 echo "Expired backup data delete complete!"
        fi
        echo "All database backup success! Thank you!"
        exit
fi
```

修改shell脚本的属性：

```
chmod 600 /root/mysql_backup.sh
chmod +x /root/mysql_backup.sh
```

设置好属性之后，把命令加入crontab，我们设置了每天00:00定时自动备份，然后把备份的脚本目录/var/www/mysql设置为rsync同步目录。

```
00 00 * * * /root/mysql_backup.sh
```

MySQL恢复

前面介绍MySQL备份分为热备份和冷备份，热备份主要的目的是为了能够实时的恢复，例如应用服务器出现了硬盘故障，那么我们可以通过修改配置文件把数据库的读取和写入改成slave，这样就可以尽量少时间的中断服务。

但是有时候我们需要通过冷备份的SQL来进行数据恢复，既然有了数据库的备份，就可以通过命令导入：

```
mysql -u username -p databse < backup.sql
```

可以看到，导出和导入数据库数据都是相当简单，不过如果还需要管理权限，或者其他的一些字符集的设置的话，可能会稍微复杂一些，但是这些都是可以通过一些命令来完成的。



**redis备份：**

redis的备份也分为两种：热备份和冷备份，redis也支持master/slave模式，所以我们的热备份可以通过这种方式实现，相应的配置大家可以参考官方的文档配置，相当的简单。我们这里介绍冷备份的方式：redis其实会定时的把内存里面的缓存数据保存到数据库文件里面，我们备份只要备份相应的文件就可以，就是利用前面介绍的rsync备份到非本地机房就可以实现。redis恢复：redis的恢复分为热备份恢复和冷备份恢复，热备份恢复的目的和方法同MySQL的恢复一样，只要修改应用的相应的数据库连接即可。

但是有时候我们需要根据冷备份来恢复数据，redis的冷备份恢复其实就是只要把保存的数据库文件copy到redis的工作目录，然后启动redis就可以了，redis在启动的时候会自动加载数据库文件到内存中，启动的速度根据数据库的文件大小来决定。



# 开发web框架

![img](https://astaxie.gitbooks.io/build-web-application-with-golang/content/zh/images/navi13.png?raw=true) 



# 集成组件

**Casbin** 

Oauth和Token

Redis

Mysql

Kafka





