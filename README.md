# automated-mysql-backup-with-mydumper

This project includes how to use mydumper to back up a MySQL database, including manual backup methods and automated scripts. It also includes a solution for restoring a large MySQL database using myloader.

# mydumper的安装

mydumper的安装的安装非常简单，在互联网上有大量的教程，这里不做介绍。

## 手动备份

```mydumper -h 10.2.14.26 -P 3306 -u dump_user -p dump_user -t 8 -c -G -E -R -B oms-returned,oms-sap-syncer,oms-system,oms-workflow -o /data/mysql_backup/10.2.14.26/$(date +%Y%m%d%H%M%S)```

命令中的重点参数说明：

-t：线程数

-G -E -R：备份触发器、events时间、存储过程&函数

-B：枚举要备份的库，用逗号隔开。

-o：备份文件存放的目录位置!

### 导出某个数据库的某张表

 ```mydumper -h 10.2.18.132 -u root -P 3309 -p Mgr_test@123 -t 2 -c -G -E -R -S /data/mysql_single/mysql.sock --regex '^(oms-order\.oms_order_trace$)' -o /data/tmp```

### 导出不同数据库的不同表

 ```mydumper -h 10.2.18.132 -u root -P 3309 -p Mgr_test@123 -t 2 -c -G -E -R -S /data/mysql_single/mysql.sock --regex '^(oms-order\.oms_order_trace$|db2\.table2$)' -o /data/tmp```

## 自动备份

备份脚本：

```
#!/bin/bash

number=5
backupdir=/data/yangfan_test
dd=`date +%Y%m%d%H%M%S`
logtime=`date +%Y%m%d-%H:%M:%S.%3N`
tool=/usr/bin/mydumper
dbhost=192.168.10.1
dbport=3306
username=yourusername
password=yourpassword
threads=8
databasename=oms-contract,oms-data
echo -e "\n==============This is a new backup process===============\n$(date +%Y%m%d-%H:%M:%S.%3N) The current number of backups to retain is:$number，The backup will be run using $threads threads\nStart the backup process=====>" >> $backupdir/log.txt
$tool -h $dbhost -P $dbport -u $username -p $password -t $threads -G -E -R -B $databasename -o $backupdir/$dd
echo -e "$(date +%Y%m%d-%H:%M:%S.%3N) Backup process completed! Backup created: $backupdir/$dd" >> $backupdir/log.txt
count=`ls -l -crt $backupdir | awk '{print $9 }'| grep [0-9] | wc -l` #Find directories that meet the statistical conditions and match the directory name in digital format in the 9th column of the result set of the "ls" command
while(($count > $number)) #This "while" will loop to determine whether the number of backup files in the current backup path is greater than the set threshold. If it is greater, it will be deleted in a loop until it reaches the set threshold.
do
delfolder=`ls -l -crt $backupdir | awk '{print $9 }' |grep [0-9]| head -1`
echo -e "$(date +%Y%m%d-%H:%M:%S.%3N) The oldest backup currently is: $backupdir/$delfolder" >> $backupdir/log.txt
echo -e "$(date +%Y%m%d-%H:%M:%S.%3N) The number of backups $count has exceeded the threshold $number，The oldest backup will be deleted $backupdir/$delfolder" >> $backupdir/log.txt
rm -rf $backupdir/$delfolder
echo -e "$(date +%Y%m%d-%H:%M:%S.%3N) Deleted: $backupdir/$delfolder" >> $backupdir/log.txt
count=`ls -l -crt $backupdir | awk '{print $9 }'| grep [0-9] | wc -l`
done

```

将以上自动备份源代码保存为.sh文件，并上传到备份服务器，脚本会根据填入到链接信息链接到mysql服务并运行备份进程。

## 恢复数据库

```myloader -h 10.2.18.23 -P 3306 -u proxysql -p proxysql -e -t 8 -o -d /data/mysql_backup/10.2.14.26/20221107131527```

命令中的重点参数说明：

-t：线程数

-o：如果要恢复的表存在，则先drop掉该表

-v：输出模式, 0 = silent, 1 = errors, 2 = warnings, 3 = info, 默认为2

-B：要还原到哪个数据库（目标库）

-s（小写）：选择被还原的数据库（源库）

-d：指定待恢复的备份文件目录

-e：在恢复时开启binlog（还原到集群环境时必须开启）

### 从全备中恢复指定库

myloader -h 10.2.18.23 -P 3306 -u proxysql -p proxysql -e -t 8 -s oms-returned -o -d /data/mydumper/20221107131527

### 将某个数据库备份还原到另一个数据库中（目标库不存在则会新建）

myloader -h 10.2.18.23 -P 3306 -u proxysql -p proxysql -e -t 8 -s from_dbname -B to_dbname -o -d /data/mydumper/20221107131527（全备目录）

myloader -h 10.2.18.23 -P 3306 -u proxysql -p proxysql -e -t 8 -B to_dbname -o -d /data/mydumper/single_db（单库备份目录）

## 优化：

myloader在导入大表（2000万记录）的时候一开始导入线程还挺多，但到了后面就剩一两个线程了，导入速度极慢，经本地测试，2000万记录的大库需要导5个小时左右。

### 备份优化

/usr/bin/mydumper -h 10.2.14.26 -u dump_user -p dump_user -P 3306 -t 10 -c -G -E -R -F 500 -o /backup/mysql_backup/10.2.14.26/202407321334 >> /backup/mysql_backup/10.2.14.26/202407321334/log.txt

-F参数将大表分割成指定大小（单位为MB）的块，以提高恢复时的并发能力。

### 恢复优化

myloader -h 10.2.18.134 -P 3306 -u root -p test -t 32 --innodb-optimize-keys AFTER_IMPORT_ALL_TABLES -s oms-workflow --queries-per-transaction 10 -o -v 3 -d /backup/mysql_backup/10.2.14.26/202407321334

重要参数调整说明：

--innodb-optimize-keys 参数用来指定导入数据时创建索引的时机， 值：AFTER_IMPORT_ALL_TABLES代表导入完表数据后再创建索引。

--queries-per-transaction 参数用来指定导入时的事务大小，如果不指定，默认值是1000，也就是说1000个insert语句构成一个事务。因为我们导出时的一个insert语句中有2000条记录，所以之前的一个事务中有200万条记录，太大了，优化后把此值改为10，这样每个事务中就是2万条数据，导入数据时可以快速提交，提高并发能力。
