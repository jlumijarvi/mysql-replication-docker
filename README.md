## Mysql replication into a slave in docker

### Master configuration

Open the config file.

```
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Replace the bind address with the IP address of the server. If the slave is in the same server as the master leave it as 127.0.0.1.

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

Open the MySQL shell. 

If needed create a new readonly db user in in order to access the slave database outside its container.

```
GRANT SELECT on *.* to 'readonly'@'%' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
```

Grant replication priviledges to the slave user, lock the databases and print the master's status.

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

Login to the slave server. Make sure [Docker](https://docs.docker.com/install/) is installed. Clone this repo.

Move the master database dump into the /docker-entrypoint-initdb.d directory and rename it to dump.sql.

Copy .env.example into .env and set the variables in the file.

Set MASTER_USER and MASTER_PASSWORD according to the master db's slave user credentials configured in the earlier step.

Set MASTER_LOG_FILE and MASTER_LOG_POS based on the values from the master db status.

Set the host's name or IP address into MASTER_HOST. If the master and the slave are in the same server use host.docker.internal.

Start the slave container. Slave starts running at port 3307. The container's MySQL data directory /var/lib/mysql will be mounted into /data.

```
docker-compose up -d
```
