## Mysql replication into a slave in docker

### Master configuration

Open the config file.

```
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Replace the bind address with the IP address of the server. If the slave is in the same server as the master set it to 127.0.0.1.

```
bind-address            = 127.0.0.1
```

Enable binary logging by uncommenting the following lines.

```
server-id               = 1
log_bin                 = /var/log/mysql/mysql-bin.log 
```

Restart the mysql server.

```
sudo service mysql restart
```

Open the MySQL shell. Grant replication priviledges to the slave use, lock the databases and print the master's status.

```
mysql -u root -p
GRANT REPLICATION SLAVE ON *.* TO 'slave_user'@'%' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;
```

Open a new shell window and export the databases with mysqldump or use other tool.

```
mysqldump -u root -p --all-databases > dump.sql
```

Unlock the databases in the original shell window. Keep the master status in the screen.

```
UNLOCK TABLES;
EXIT;
```

### Slave configuration

Login to the slave server. Make sure docker is installed. Clone this repo.

Move the master database dump into the /docker-entrypoint-initdb.d directory and rename it to dump.sql.

Create a file start.sql in /docker-entrypoint-initdb.d with the following content. Replace values for MASTER_LOG_FILE and MASTER_LOG_POS with the values from the master db status. Replace MASTER_HOST with the host's name or IP address. If the master and the slave are in the same server use host.docker.internal.

```
CHANGE MASTER TO MASTER_HOST='host.docker.internal', MASTER_USER='slave_user', MASTER_PASSWORD='password', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=154;
START SLAVE;
```

Start the slave container.

```
docker-compose up -d
```

If needed create a new db user in the master database in order to access the slave database outside its container.

```
GRANT SELECT on *.* to 'readonly'@'%' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
```
