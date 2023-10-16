# mysql_slave_master_configuration
Mysql configuration of master and slave
https://dev.mysql.com/doc/refman/8.0/en/multiple-data-directories.html
https://dev.mysql.com/doc/refman/8.0/en/multiple-unix-servers.html
https://dev.mysql.com/doc/refman/8.0/en/replication-howto-masterstatus.html
https://www.toptal.com/mysql/mysql-master-slave-replication-tutorial
https://dev.mysql.com/doc/refman/8.0/en/log-file-maintenance.html

PRE-REPLICATION HOUSEKEEPING
----------------------------
1. Remove skip-log-bin from my.cnf, restart mysql.
	- SELECT @@log_bin

2. SHOW MASTER STATUS;

3. Slave can be used as a backup as well now.
	- SET PERSIST binlog_expire_logs_seconds=259200; (i.e. 3 days will do)
	- FLUSH LOGS; -- flush logs shld be part of preventive maintenance

4. SELECT @@innodb_undo_log_truncate;
	- 1 (i.e. truncation on undo tablespace enabled)

5. SELECT @@innodb_max_undo_log_size;
	- 1073741824 (i.e. 1 GB)
	
6. Ensure autocommit is in place, else u need to self-commit to write into binary logs
	- SET PERSIST autocommit=1;

REPLICATION
-----------
1. Connect (SIT):
	10.0.0.192
	root/h411..
	
	(MySQL SIT root password is mySQLP@55w0rd, but I don't think we need that kind of privileges)

2. vi /etc/my.cnf, and update as below 
	(NOTE: due to space issue, move data directory to root (i.e. /mysql) instead of /var/lib/mysql):
	
	[mysqld@master]
	server-id=1
	datadir=/var/lib/mysql
	socket=/var/lib/mysql/mysql.sock
	log-error=/var/log/mysqld.log
	pid-file=/var/run/mysqld/mysqld.pid
	bind-address=10.0.0.192
	port=3306
	innodb_flush_log_at_trx_commit=1
	sync_binlog=1

	[mysqld@slave]
	server-id=2
	datadir=/var/lib/mysql-slave
	socket=/var/lib/mysql-slave/mysql_slave.sock
	log-error=/var/log/mysqld_slave.log
	pid-file=/var/run/mysqld/mysqld_slave.pid
	bind-address=10.0.0.192
	port=3307
	loose_mysqlx_port=33061
	mysqlx_socket=/var/run/mysqld/mysqlx_slave.sock
	super_read_only=1

3. Open port 3307 (and 3306 if not done so)

	firewall-cmd --zone=public --permanent --add-port=3306/tcp
	firewall-cmd --zone=public --permanent --add-port=3307/tcp
	firewall-cmd --reload
	firewall-cmd --list-all
	
	Note: If encounter error like "FirewallD is not running", do "systemctl restart firewalld"
	
4. Stop the existing (master) database, so that the copy of its data directory is clean

	service mysqld stop
	service mysqld status
	
5. duplicate a copy of master data directory (remove slave's auto.cnf which contain the master generated uuid, file will regen)

	cp -R /var/lib/mysql /root/mysql
	chmod -R --reference=/var/lib/mysql /root/mysql
	chown -R --reference=/var/lib/mysql /root/mysql


	cp -R /var/lib/mysql /var/lib/mysql-slave
	chmod -R --reference=/var/lib/mysql /var/lib/mysql-slave
	chown -R --reference=/var/lib/mysql /var/lib/mysql-slave
	cd /var/lib/mysql-slave
	rm -f auto.cnf

6. start both services & verify their statuses

	service mysqld@master start
	service mysqld@slave start
	service mysqld@master status
	service mysqld@slave status

7. login to master db, create replication user, flush all commits and lock

	mysql -u tag_user -p --host=10.0.0.192 --port=3306
	
	CREATE USER 'replication'@'%' IDENTIFIED WITH sha256_password BY 'P@ssw0rd';
	
	GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
	
	FLUSH TABLES WITH READ LOCK;
		-- If you only can use 1x session, then no choice, skip the locking. Ensure no one (or app) is meddling with your db
		-- In order to maintain lock, don't quit client. 
		
	SHOW MASTER STATUS;
		-- File: mysql-bin.000010
		-- Position: 863
	
8. open another putty session, login to slave db, setup replication

	mysql -u tag_user -p --host=10.0.0.192 --port=3307
	
	CHANGE MASTER TO MASTER_HOST='10.0.0.192', MASTER_PORT=3306, MASTER_USER='replication', MASTER_PASSWORD='P@ssw0rd', 
		MASTER_LOG_FILE='binlog.000021', MASTER_LOG_POS=19358, MASTER_AUTO_POSITION=0, MASTER_SSL=0;
		
	START SLAVE;
	
	SHOW SLAVE STATUS \G
	
		Slave_IO_State: Waiting for master to send event
		Slave_IO_Running: Yes
		Slave_SQL_Running: Yes
	
9. go back to master db and unlock (if you did not skip locking)
	
	UNLOCK TABLES;

10. Note the following approach of logging into the db. In order to access the correct instance, u have to connect via TCP/IP.
	
	mysql -u tag_user -p --host=10.0.0.192 --port=3306
	mysql -u tag_user -p --host=10.0.0.192 --port=3307

	root user connection can only connect via localhost (socket), which will ignore --port (only for TCP/IP), and always connect to default port 3306
	if you want to do root permission stuff on other port, goanna create a user with GRANT ALL permission to it (standard app user is GRANT SELECT,INSERT,UPDATE,DELETE permissions
		or maybe even ALTER permission for hibernate db update)
	
	mysql -u root -p
	(need to putty to 10.0.0.192 first, password is "mySQLP@55w0rd" for SIT DB)
	
	mysql -S /mysql/mysql.sock -u root -p
	(NOTE: due to space issue, need to specify the correct .sock to use as data directory moved to root (i.e. /mysql) instead of /var/lib/mysql):
			
11. verify CRUD in master is replicated to slave

	C: insert alerts (id, createdBy) values (1234567, 'test');
	R: select * from alerts where id = 1234567;
	U: update alerts set createdBy='REPLICATED' where id = 1234567;
	D: delete from alerts where id = 1234567;

12. update the services to start during boot from mysqld to mysqld@master and mysqld@slave

	systemctl disable mysqld@service
	systemctl edit mysqld@master.service --full
		Under [Unit], add "Requires=mysqld@slave.service" and "After=mysqld@slave.service", save with ":wq"

	systemctl enable mysqld@master.service
	reboot 							(<-- to restart server)

	systemctl status mysqld@master.service 			(<-- should be started automatically)
	systemctl status mysqld@slave.service 			(<-- should be started by its master automatically)
	mysql -u tag_user -p --host=10.0.0.192 --port=3306 	(<-- login to master)
	show master status; 					(<-- take note of the binary log file and position)
	exit
	mysql -u tag_user -p --host=10.0.0.192 --port=3307	(<-- login to slave)
	show slave status \G					(<-- Master_Log_File and Pos should tally. Status should be "Waiting for master to send event")
	exit

----------------------------------------
FYI, if you want to do the export/import way:

1. export dump from master (pls wait...)
	
	cd /var/lib
	mysqldump -u tag_user -p --host=10.0.0.192 --port=3306 --all-databases --master-data=2 > replicationdump.sql
		
2. find out master log pos from the generated replicationdump.sql (due to master-data=2)
		
	less replicationdump.sql
		and look for "-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000004', MASTER_LOG_POS=155;"
	
	verify in master
		show master status;
		
3. import dump into slave (pls wait...)

	mysql -u tag_user -p --host=10.0.0.192 --port=3307 < replicationdump.sql
	rm replicationdump.sql

POST REPLICATION
----------------
- UAT: remove original /var/lib/mysql datadir
- PROD: remove original /var/lib/mysql datadir, remove binlog bakup


Connect to root user in Production DB
-----------------------------------------
mysql -S /mysql/mysql.sock -u root -p
