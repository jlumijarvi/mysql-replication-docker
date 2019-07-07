## Mysql replication into a slave in docker

The tutorial assumes that there is a running master database server. The steps have been tested with MySQL version 5.7.

### Master configuration

Open the config file.

```
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Replace the bind address with the public IP address of the server. In local environments it can be left as 127.0.0.1.

```
bind-address            = 127.0.0.1
```

Enable binary logging by uncommenting the following lines.

```
server-id               = 1
log_bin                 = /var/log/mysql/mysql-bin.log 
```

Restart the MySQL server.

```
sudo service mysql restart
```

Open the MySQL shell. 

```
mysql -u root -p
```

Grant replication privileges to the slave user. Create also a readonly user. Grant LOCK TABLES privilege if the database export will be done in the slave server.

```
GRANT REPLICATION SLAVE ON *.* TO 'slave_user'@'%' IDENTIFIED BY 'password';
GRANT SELECT, LOCK TABLES on *.* to 'readonly'@'%' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
```

Lock the databases and print the master's status.

```
FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;
```

Open a new shell window and export the databases with mysqldump or use other tool.

```
mysqldump -u root -p --all-databases > dump.sql
```

or gzipped.

```
mysqldump -u root -p --all-databases | gzip > dump.gz
```

Export can be done also in the slave server. Follow the steps in [Master server firewall configuration](#master-server-firewall-configuration) to allow access from the slave to the master.

Login to the slave server and export the master databases with mysqldump.

```
mysqldump -h <master IP address> -u readonly -p --all-databases > dump.sql
```

In the original shell window unlock the databases. If LOCK TABLES privilege was granted to the readonly user, remove it now. Exit the MySQL shell. Keep the master status visible in the console.

```
UNLOCK TABLES;
GRANT SELECT on *.* to 'readonly'@'%' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
EXIT;
```

### Slave configuration

Login to the slave server. Make sure [Docker](https://docs.docker.com/install/) and [Docker Compose](https://docs.docker.com/compose/install/) are installed. Clone this repo.

Move the master database dump into the /docker-entrypoint-initdb.d directory, unzip if needed and rename it to dump.sql.

Copy .env.example into .env and set the variables in the file.

- Set the host's name or IP address into MASTER_HOST. In local environments with Mac or Windows set it to host.docker.internal.
- Set MASTER_USER and MASTER_PASSWORD according to the master db's slave user credentials configured in the earlier step.
- Set MASTER_LOG_FILE and MASTER_LOG_POS based on the values from the master status.

Start the container. Slave starts running at port 3307. The container's MySQL data directory /var/lib/mysql will be mounted into /data. Note that the MYSQL_ROOT_PASSWORD defined in .env is not valid anymore. Use the master db's credentials to access the slave.

```
docker-compose up -d
```

### Master server firewall configuration

In other than local environments the firewall should be configured to allow MySQL access from the slave to the master.

If the slave is in another server get the server's public IP address.

If the slave is in the same server check the slave container network's IP address. 

```
docker inspect slave_db --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

Add ufw rule.

```
sudo ufw allow from <slave IP address> to any port 3306
sudo ufw reload
```
