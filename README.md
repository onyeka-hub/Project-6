# PROJECT 6

## TASK

Prepare storage infrastructure on two Linux servers and implement a basic web solution using WordPress. WordPress is a free and open-source content management system written in PHP and paired with MySQL or MariaDB as its backend Relational Database Management System (RDBMS).

## Project 6 consists of two parts:

Configure storage subsystem for Web and Database servers based on Linux OS. The focus of this part is to give you practical experience of working with disks, partitions and volumes in Linux.

Install WordPress and connect it to a remote MySQL database server. This part of the project will solidify your skills of deploying Web and DB tiers of Web solution.

## Three-tier Architecture

Generally, web, or mobile solutions are implemented based on what is called the Three-tier Architecture.

### Three-tier Architecture is a client-server software architecture pattern that comprise of 3 separate layers.

1. Presentation Layer (PL): This is the user interface such as the client server or browser on your laptop.

2. Business Layer (BL): This is the backend program that implements business logic. Application or Webserver

3. Data Access or Management Layer (DAL): This is the layer for computer data storage and data access. Database Server or File System Server such as FTP server, or NFS Server

## Your 3-Tier Setup

1. A Laptop or PC to serve as a client

2. An EC2 Linux Server as a web server (This is where you will install WordPress)

3. An EC2 Linux server as a database (DB) server (This is where you will install myqsl-server)

## Step 1 — Prepare a Web Server

1. Launch an EC2 instance that will serve as "Web Server". Create 3 volumes in the same AZ as your Web Server EC2, each of 10 GiB. 

2. Attach all three volumes one by one to your Web Server EC2 instance

3. Open up the Linux terminal to begin configuration

4. Use `lsblk` command to inspect what block devices are attached to the server. Notice names of your newly created devices. All devices in Linux reside in /dev/ directory. Inspect it with ls /dev/ and make sure you see all 3 newly created block devices there – their names will likely be nvme1n1, nvme2n1, nvme3n1.

![new-attached-vols](https://user-images.githubusercontent.com/83009045/160480713-3e8cc073-fc8a-4f55-acdb-870bbd24bd34.JPG)

5. Use `df -h` command to see all mounts and free space on your server

6. Use `gdisk` utility to create a single partition on each of the 3 disks

```
	sudo gdisk /dev/nvme1n1
```


7. Use `lsblk` utility to view the newly configured partition on each of the 3 disks

![new-partitons](https://user-images.githubusercontent.com/83009045/160480840-c9353358-405c-4d43-a207-1266488ee54b.JPG)

8. Install `lvm2` package using `sudo yum install lvm2`. 

 Run `sudo lvmdiskscan` command to check for available partitions.

![available-partition](https://user-images.githubusercontent.com/83009045/160480931-d5a00e68-3318-416b-9c28-aeef2cf455b7.JPG)

9. Use `pvcreate` utility to mark each of 3 disks as physical volumes (PVs) to be used by LVM

```
	sudo pvcreate /dev/nvme1n1p1
	sudo pvcreate /dev/nvme2n1p1
	sudo pvcreate /dev/nvme3n1p1
```


10. Verify that your Physical volume has been created successfully by running `sudo pvs`

![physical-vols](https://user-images.githubusercontent.com/83009045/160481066-8b985790-594c-493a-a2bc-b114bea2acb6.JPG)

10. Use `vgcreate` utility to add all 3 PVs to a volume group (VG). Name the VG webdata-vg


	`sudo vgcreate webdata-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1`
	
![vol-group](https://user-images.githubusercontent.com/83009045/160481118-31c8314f-d1be-40d5-bd7e-8cbb13591191.JPG)

11. Use `lvcreate` utility to create 2 logical volumes. apps-lv (Use half of the PV size), and logs-lv Use the remaining space of the PV size. NOTE: apps-lv will be used to store data for the Website while, logs-lv will be used to store data for logs.


```
	sudo lvcreate -n apps-lv -L 14G webdata-vg

	sudo lvcreate -n logs-lv -L 14G webdata-vg
```

12. Verify that your Logical Volume has been created successfully by running `sudo lvs`

![logical-vol](https://user-images.githubusercontent.com/83009045/160481199-007d1a8a-06ca-4fff-aa45-2058d153290d.JPG)

13. Verify the entire setup

```
	sudo vgdisplay -v #view complete setup - VG, PV, and LV

	sudo lsblk 
```

***

14. Use `mkfs.ext4` to format the logical volumes with ext4 filesystem

```
	sudo mkfs -t ext4 /dev/webdata-vg/apps-lv

	sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```

15. Create /var/www/html directory to store website files

```
	sudo mkdir -p /var/www/html
```

16. Create /home/recovery/logs to store backup of log data

```
	sudo mkdir -p /home/recovery/logs
```

17. Mount /var/www/html on apps-lv logical volume

```
	sudo mount /dev/webdata-vg/apps-lv /var/www/html/
```

18. Use `rsync` utility to backup all the files in the log directory /var/log into /home/recovery/logs (This is required before mounting the file system)

```
	sudo rsync -av /var/log/. /home/recovery/logs/
```

19. Mount /var/log on logs-lv logical volume. (Note that all the existing data on /var/log will be deleted. That is why step 15 above is very
important)

```
	sudo mount /dev/webdata-vg/logs-lv /var/log
```

20. Restore log files back into /var/log directory

```
	sudo rsync -av /home/recovery/logs/. /var/log
```

![mounted-vol](https://user-images.githubusercontent.com/83009045/160481292-4b8a6757-61e6-4b1e-ae65-7d5703ac7d97.JPG)

21. Update /etc/fstab file so that the mount configuration will persist after restart of the server.

### The UUID of the device will be used to update the /etc/fstab file;

`sudo blkid`

***

`sudo vi /etc/fstab`

### Update /etc/fstab in this format using your own UUID

![updated-fstab](https://user-images.githubusercontent.com/83009045/160481427-11237902-026c-4089-96f4-2f8bab515554.JPG)


22. Test the configuration and reload the daemon

```
	sudo mount -a

	sudo systemctl daemon-reload
```

23. Verify your setup by running `df -h`

![updated-fstab2](https://user-images.githubusercontent.com/83009045/160481504-74bf16ea-440f-4d1c-94e6-f884b3cbdd16.JPG)

## Step 2 — Prepare the Database Server

Launch a second RedHat EC2 instance that will have a role – ‘DB Server’

Repeat the same steps as for the Web Server, but instead of apps-lv create db-lv and mount it to /db directory instead of /var/www/html/.

## Step 3 — Install MySQL on your DB Server EC2

```
	sudo yum update

	sudo yum install mysql-server
```

Verify that the service is up and running by using `sudo systemctl status mysqld`, if it is not running, restart the service and enable it so it will be running even after reboot:

```
	sudo systemctl restart mysqld

	sudo systemctl enable mysqld
```

Create the database user that the wordpress server will use to connect to the db server

```
	sudo mysql

	CREATE DATABASE wordpress;

	CREATE USER `onyeka-user`@`<Web-Server-Private-IP-Address>` IDENTIFIED BY 'onyeka12345';

	GRANT ALL ON wordpress.* TO 'onyeka-user'@'<Web-Server-Private-IP-Address>';

	FLUSH PRIVILEGES;

	SHOW DATABASES;

	exit
```

To show the users in a database do

`SELECT User, Host FROM mysql.user;`

To remove a user do 

`DROP USER 'username'@'host';`

![mysql-status](https://user-images.githubusercontent.com/83009045/160481848-687f4768-2b77-4769-a249-484a4c4b8ee2.JPG)


## Step 4 — Install WordPress on your Web Server EC2

1. Update the repository

	`sudo yum -y update`

2. Install wget, Apache and it’s dependencies

	`sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json`

3. Start Apache

```
	sudo systemctl enable httpd

	sudo systemctl start httpd
```


4. To install PHP and it’s depemdencies

```
	sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

	sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm

	sudo yum module list php

	sudo yum module reset php

	sudo yum module enable php:remi-7.4

	sudo yum install php php-opcache php-gd php-curl php-mysqlnd
	
	sudo systemctl start php-fpm

	sudo systemctl enable php-fpm

	sudo setsebool -P httpd_execmem 1

	sudo systemctl restart httpd
```

5. Download wordpress and copy wordpress to var/www/html

```
	mkdir wordpress

	cd   wordpress

	sudo wget http://wordpress.org/latest.tar.gz
 
	sudo tar -xzvf latest.tar.gz
 
	sudo rm -rf latest.tar.gz
 
	sudo cp -R wordpress /var/www/html/

	sudo systemctl restart httpd
```

### cd into /var/www/html directory

```
	cd wordpress

	
	sudo cp wp-config-sample.php wp-config.php
```

**where** 

		`sudo wget http://wordpress.org/latest.tar.gz` downloads the file

		`sudo tar -xzvf latest.tar.gz` uncompresses the file

		`sudo rm -rf latest.tar.gz` deletes the compressed file and its content

		`sudo cp -R wordpress/ /var/www/html/` will copy the directory wordpress recursively to /var/www/html directory. 

		`cp wp-config-sample.php wp-config.php` this is creating another file wp-config.php which 
		contains the scipt which wordpress uses for installation.This contains database setting where to put the values 
		of the database user name, password, host name so that the webserver can be able to connect to the database server. 

**Note** that the host name here should be the private ip address of the database server, if both the webserver and database server are in the same local network.


## Step 5 — Configure WordPress to connect to remote database.


**Hint**: Do not forget to open MySQL port 3306 on DB Server EC2. For extra security, you shall allow access to the DB server ONLY from your Web Server’s IP address, so in the Inbound Rule configuration specify source as /32

1. Install MySQL client and test that you can connect from your Web Server to your DB server by using mysql-client

```
	sudo yum install mysql

	sudo mysql -u onyeka-user -p -h <DB-Server-Private-IP-address>
```


2. Verify if you can successfully execute `SHOW DATABASES;` command and see a list of existing databases.

![db-from-webserver](https://user-images.githubusercontent.com/83009045/160481941-11cebbb3-68cd-4690-8b2a-18691ba0c118.JPG)

6. Fill out your DB credentials:

![wp-config php](https://user-images.githubusercontent.com/83009045/160481667-46675a84-49b3-4b11-8924-50e1ca838017.JPG)

![wordpress2](https://user-images.githubusercontent.com/83009045/160482101-a760692a-82fc-479f-b067-2d07043ba6f3.JPG)


**Note** that at this point with the public ip address, the webserver will be serving the redhat page on the browser and will serve the wordpress page when you add /wordpress to public ip on the browser.

But we want the webserver to serve the wordpress page with only the public ip on the browser.

To this, we have to edit the  apache http server configuration file **httpd.conf** located at **/etc/httpd/conf/httpd.conf** file. This contains the configuration directives that gives the server its instructions.

This contains the document root which is the directory out of which you will server your documents. 

 Change it from **"/var/www/html"** to **"/var/www/html/wordpress"**

![document-root](https://user-images.githubusercontent.com/83009045/160481767-818ac519-6d18-4a66-8a83-0814eaf8b0ea.JPG)

6. Configure SELinux Policies

```
	sudo chown -R apache:apache /var/www/html/wordpress

	sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R  
	
	sudo setsebool -P httpd_can_network_connect=1
```


3. Change permissions and configuration so Apache could use WordPress:

4. Enable TCP port 80 in Inbound Rules configuration for your Web Server EC2 (enable from everywhere 0.0.0.0/0 or from your workstation’s IP)

5. Try to access from your browser the link to your WordPress `http://<Web-Server-Public-IP-Address>/wordpress/` or `http://<Web-Server-Public-IP-Address>`

![wordpress1](https://user-images.githubusercontent.com/83009045/160482015-294c7b60-64c6-4d38-ba0f-1ec35e5e920d.JPG)



7. With this message – it means your WordPress has successfully connected to your remote MySQL database and you can starting working on wordpress.

![wordpr![wordpress3](https://user-images.githubusercontent.com/83009045/160482265-c8d9baf2-dd53-422a-802f-7d86df9bfd40.JPG)

## End of project 6
