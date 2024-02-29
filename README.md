# MariaDB Master-Master Replication on Amazon Linux 2

This guide provides step-by-step instructions to set up MariaDB master-master replication on Amazon Linux 2. The replication configuration allows changes made on one server to be automatically synchronized to the other server, providing redundancy and high availability.

## Prerequisites
- Two Amazon Linux 2 instances
- SSH access to both instances
- Allow port 3306 in the security group
- Basic knowledge of working with the Linux command line

## Steps

### Step 1: Install MariaDB on both servers

1. Connect to your Amazon Linux 2 instances using SSH.
2. Update the package lists by running the command:
   ```
   sudo yum update
   ```
3. Install MariaDB by running the command:
   ```
   sudo yum install mariadb-server
   ```
4. Start the MariaDB service:
   ```
   sudo systemctl start mariadb
   ```
5. Enable MariaDB to start on boot:
   ```
   sudo systemctl enable mariadb
   ```
6. Secure your MariaDB installation by running the following command and following the prompts:
   ```
   sudo mysql_secure_installation
   ```

### Step 2: Configure MariaDB replication on the first server (Server A)

1. Edit the MariaDB configuration file by running the command:
   ```
   sudo vi /etc/my.cnf.d/server.cnf
   ```
2. Add the following lines to the `[mysqld]` section:
   ```
   server-id = 1
   log_bin = /var/log/mysql/mariadb-bin
   log_bin_index = /var/log/mysql/mariadb-bin.index
   binlog_format = mixed
   ```
3. Save and exit the file.
4. Verify that the directory `/var/log/mysql/` exists:
   ```
   sudo ls /var/log/mysql/
   ```
   If it doesn't exist, create the missing directory:
   ```
   sudo mkdir /var/log/mysql/
   sudo chown -R mysql:mysql /var/log/mysql/
   sudo systemctl restart mariadb
   ```
5. Restart the MariaDB service to apply the configuration changes:
   ```
   sudo systemctl restart mariadb
   ```
6. Log in to the MySQL shell:
   ```
   sudo mysql -u root -p
   ```
7. Create a replication user and grant the necessary privileges:
   ```sql
   CREATE USER 'replication_user'@'%' IDENTIFIED BY 'password';
   GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'%';
   FLUSH PRIVILEGES;
   ```
8. Exit the MySQL shell:
   ```
   exit;
   ```
9. Take a snapshot or backup of the database on Server A to use as an initial data copy for Server B.

### Step 3: Configure MariaDB replication on the second server (Server B)

1. Repeat Step 2 on Server B, but make the following changes in the MariaDB configuration file (`/etc/my.cnf.d/server.cnf`) for Server B:
   ```
   server-id = 2
   log_bin = /var/log/mysql/mariadb-bin
   log_bin_index = /var/log/mysql/mariadb-bin.index
   binlog_format = mixed
   ```
2. Restart the MariaDB service on Server B:
   ```
   sudo systemctl restart mariadb
   ```
3. Log in to the MySQL shell on Server B:
   ```
   sudo mysql -u root -p
   ```
4. Create a replication user and grant the necessary privileges, similar to Step 2:
   ```sql
   CREATE USER 'replication_user'@'%' IDENTIFIED BY 'password';
   GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'%';
   FLUSH PRIVILEGES;
   ```
5. Exit the MySQL shell:
   ```
   exit;
   ```
6. Restore the database snapshot or backup taken from Server A to Server B.

### Step 4: Configure replication on both servers

1. Log in to the MySQL shell on Server A:
   ```
   sudo mysql -u root -p
   ```
2. Get the current binary log file and position by running the following command:
   ```sql
   SHOW MASTER STATUS;
   ```
   Take note of the values for `File` and `Position`.
3. Configure replication on Server B by running the following commands in the MySQL shell on Server B:
   ```sql
   STOP SLAVE;
   CHANGE MASTER TO MASTER_HOST='ServerA_IP_Address', MASTER_USER='replication_user', MASTER_PASSWORD='password', MASTER_LOG_FILE='File', MASTER_LOG_POS=Position;
   START SLAVE;
   ```
   Replace `ServerA_IP_Address` with the IP address of Server A and replace `File` and `Position` with the values obtained in Step 4, point 2.
4. Log in to the MySQL shell on Server B:
   ```
   sudo mysql -u root -p
   ```
5. Get the current binary log file and positionon Server B by running the following command:
   ```sql
   SHOW MASTER STATUS;
   ```
   Take note of the values for `File` and `Position`.
6. Configure replication on Server A by running the following commands in the MySQL shell on Server A:
   ```sql
   STOP SLAVE;
   CHANGE MASTER TO MASTER_HOST='ServerB_IP_Address', MASTER_USER='replication_user', MASTER_PASSWORD='password', MASTER_LOG_FILE='File', MASTER_LOG_POS=Position;
   START SLAVE;
   ```
   Replace `ServerB_IP_Address` with the IP address of Server B and replace `File` and `Position` with the values obtained in Step 4, point 5.
7. Verify the replication status on both servers:
   - On Server A:
     ```sql
     SHOW SLAVE STATUS\G;
     ```
     Ensure that the `Slave_IO_Running` and `Slave_SQL_Running` fields show `Yes`.
   - On Server B:
     ```sql
     SHOW SLAVE STATUS\G;
     ```
     Ensure that the `Slave_IO_Running` and `Slave_SQL_Running` fields show `Yes`.

### Step 5: Test the replication

To test the replication, you can perform database operations on either server and verify that the changes are replicated to the other server. For example, you can create a new table or insert records into an existing table on Server A and check if the changes are reflected on Server B, and vice versa.

Keep in mind that replication can take some time to propagate the changes depending on the size of your database and the network speed between the servers.

Congratulations! You have successfully set up MariaDB master-master replication on Amazon Linux 2. Now you have a replicated database environment where changes made on either server are automatically synchronized to the other server.
