# automated-mysql-backup-with-mydumper

This project includes how to use mydumper to back up a MySQL database, including manual backup methods and automated scripts. It also includes a solution for restoring a large MySQL database using myloader.

# Installation of mydumper

The installation of mydumper is very simple. There are a lot of tutorials on the Internet, which will not be introduced here.

## Manual backup

```mydumper -h 10.2.14.26 -P 3306 -u dump_user -p dump_user -t 8 -c -G -E -R -B db_name1,db_name2,db_name3 -o /data/mysql_backup/10.2.14.26/$(date +%Y%m%d%H%M%S)```

Description of key parameters in the command:

-t: number of threads

-G -E -R: backup triggers, events time, stored procedures & functions

-B: enumerate the libraries to be backed up, separated by commas.

-o: directory location where the backup files are stored!

### Export a table from a database

 ```mydumper -h 10.2.18.132 -u root -P 3309 -p password -t 2 -c -G -E -R -S /data/mysql_single/mysql.sock --regex '^(db_name\.table$)' -o /data/tmp```

### Export different tables from different databases

 ```mydumper -h 10.2.18.132 -u root -P 3309 -p password -t 2 -c -G -E -R -S /data/mysql_single/mysql.sock --regex '^(db_name1\.table$|db_name2\.table$)' -o /data/tmp```

## Automatic backup

Backup script:

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
databasename=db_name1,db_name2,db_name3
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

Save the above automatic backup source code as a .sh file and upload it to the backup server. The script will connect to the MySQL service according to the link information filled in and run the backup process.

## Recovering the Database

```myloader -h 10.2.18.23 -P 3306 -u username -p password -e -t 8 -o -d /data/mysql_backup/10.2.14.26/20241107131527```

Key parameter descriptions in the command:

-t: number of threads

-o: if the table to be restored exists, drop it first

-v: output mode, 0 = silent, 1 = errors, 2 = warnings, 3 = info, default is 2

-B: which database to restore to (target database)

-s (lowercase): select the database to be restored (source database)

-d: specify the backup file directory to be restored

-e: enable binlog during recovery (must be enabled when restoring to a cluster environment)

### Restore the specified database from the full backup

myloader -h 10.2.18.23 -P 3306 -u proxysql -p proxysql -e -t 8 -s oms-returned -o -d /data/mydumper/20241107131527

### Restore a database backup to another database (a new database will be created if the target database does not exist)

myloader -h 10.2.18.23 -P 3306 -u proxysql -p proxysql -e -t 8 -s from_dbname -B to_dbname -o -d /data/mydumper/20241107131527（全备目录）

myloader -h 10.2.18.23 -P 3306 -u proxysql -p proxysql -e -t 8 -B to_dbname -o -d /data/mydumper/single_db（单库备份目录）

## Optimization:

When myloader imports a large table (20 million records), there are many import threads at the beginning, but only one or two threads are left later. The import speed is extremely slow. According to local tests, it takes about 5 hours to import a large database with 20 million records.

### Backup optimization

/usr/bin/mydumper -h 10.2.14.26 -u username -p password -P 3306 -t 10 -c -G -E -R -F 500 -o /backup/mysql_backup/10.2.14.26/202407321334 >> /backup/mysql_backup/10.2.14.26/202407321334/log.txt

The -F parameter splits large tables into blocks of a specified size (in MB) to improve the concurrency of recovery.

### Recovery optimization

myloader -h 10.2.18.134 -P 3306 -u username -p password -t 32 --innodb-optimize-keys AFTER_IMPORT_ALL_TABLES -s oms-workflow --queries-per-transaction 10 -o -v 3 -d /backup/mysql_backup/10.2.14.26/202407321334

Important parameter adjustment instructions:

The --innodb-optimize-keys parameter is used to specify the timing of creating indexes when importing data. The value: AFTER_IMPORT_ALL_TABLES means that the index will be created after the table data is imported.

The --queries-per-transaction parameter is used to specify the transaction size during import. If not specified, the default value is 1000, which means that 1000 insert statements constitute a transaction. Because there are 2000 records in an insert statement during export, there were 2 million records in a previous transaction, which was too large. After optimization, this value is changed to 10, so that each transaction has 20,000 data records, which can be quickly submitted when importing data, improving concurrency.
