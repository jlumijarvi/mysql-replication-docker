## Mysql replication into a slave in docker

### Master configuration

Open the config file.

```
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Replace the bind address with the IP address of the server. In local environments it can be left as 127.0.0.1.

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

```
mysql -u root -p
```

If needed create a new readonly db user in in order to access the slave database outside its container.

```
GRANT SELECT on *.* to 'readonly'@'%' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
```

Grant replication privileges to the slave user, lock the databases and print the master's status.

```
GRANT REPLICATION SLAVE ON *.* TO 'slave_user'@'%' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
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

In the original shell window unlock the databases. Keep the master status in the console.

```
UNLOCK TABLES;
EXIT;
```

### Slave configuration

Login to the slave server. Make sure [Docker](https://docs.docker.com/install/) is installed. Clone this repo.

Docker containers bypass firewalls by default so in case using ufw or similar create custom deamon configuration.

```
sudo nano /etc/docker/daemon.js
```

```
{
        "iptables": false
}
```

```
sudo service docker restart
```

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

In other than local environments where a firewall such us afw is used allow MySQL access from the slave to the master.

If the slave is in another server get the server's public IP address.

If the slave in the same server check the slave container network's IP address. 

```
docker inspect slave_db --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

Add ufw rule.

```
sudo ufw allow from <slave IP address> to any port 3306
sudo ufw reload
```
