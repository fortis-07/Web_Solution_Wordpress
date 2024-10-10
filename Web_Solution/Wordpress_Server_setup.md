# Step 1: WebServer Setup

Launch an EC2 instance that will serve as 'web server'. Create 3 volumes in the same AZ as your web server EC2, each of 10 Gib

![WhatsApp Image 2024-10-07 at 09 08 52_9b651b33](https://github.com/user-attachments/assets/017cf88c-d742-495c-b509-8ad66b853600)


add 3 volumes here, so they'll be 4, ie, root volume and the other 3

![WhatsApp Image 2024-10-10 at 09 49 35_600c44ce](https://github.com/user-attachments/assets/6608bdef-4c87-49c7-aff5-197660e65d21)


Then launch your instance.

Confirm your volumes are up and in use.



Open your Linux terminal to begin configurations. SSH into your instance

```powershell
ssh -i keypair.pem ec2-user@public-ip
```

List attached volumes

```powershell
lsblk
```

![WhatsApp Image 2024-10-07 at 10 37 15_46b22aa9](https://github.com/user-attachments/assets/44c0d497-b4b7-45c2-af7c-7aa62a52657a)


Use `df -h` to see all mounted drives and free spaces

Create a single partition on each of the 3 disks

```powershell
sudo gdisk /dev/xvdb
```

Type n to create a new partition.  
Enter the partition number (default is 1).  
Enter the first sector (default is fine).  
Enter the last sector or size. 
Then choose p to create
Type w to write the partition table and exit.

![WhatsApp Image 2024-10-07 at 10 29 06_0baf9af0](https://github.com/user-attachments/assets/7cd5f904-b7a2-4bbc-a6d1-0b5673fbe04a)

![WhatsApp Image 2024-10-07 at 10 38 49_c73b1e90](https://github.com/user-attachments/assets/46226e2d-937f-4129-927d-d485ec23657e)

Use the `lsblk` utility to see all newly created partitions for the 3 volumes.

![WhatsApp Image 2024-10-07 at 10 41 52_b1c8e284](https://github.com/user-attachments/assets/13ae1b66-66a3-446e-8054-23fce2c54a12)


Install lvm2 package. Lvm2 is used for managing disk drives and other storage devices

```powershell
sudo yum install lvm2
```

![Screenshot (338)](https://github.com/user-attachments/assets/8b1f1eda-afb5-4661-928f-55cf1c98150c)


Run `sudo lvmdiskscan` to check for available partitions.  
Use the pvcreate utility tool to mark each of the volumes as physical volumes

```powershell
sudo pvcreate /dev/xvdb
sudo pvcreate /dev/xvdb
sudo pvcreate /dev/xvdb
```

Verify that the physical volume has been created `sudo pvs`

![WhatsApp Image 2024-10-07 at 11 11 14_7f116046](https://github.com/user-attachments/assets/bfec0b56-b219-4741-977d-9849308a07d6)

![WhatsApp Image 2024-10-07 at 11 14 14_57d71394](https://github.com/user-attachments/assets/0fd163e5-5552-464b-840c-110f0c1d97d2)

Add all 3 PVs to a volume group called `webdata-vg`

```powershell
sudo vgcreate webdata-vg /dev/nvme1n1p1  /dev/nvme2n1p1  /dev/nvme3n1p1
```

Verify the setup by running `sudo vgs`

![WhatsApp Image 2024-10-07 at 11 17 32_dcd23a05](https://github.com/user-attachments/assets/bf0ee406-9f2d-4d1e-9fe1-2748da227df9)


Create 2 logical volumes; app-lv and logs-lv.

```powershell
sudo lvcreate -n app-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg
```

Verify that the logical volumes has been created `sudo lvs`

![WhatsApp Image 2024-10-07 at 11 21 07_930fcd1b](https://github.com/user-attachments/assets/2668a0ad-ae14-415b-a23f-33c3835c0643)


Verify the entire setup to be sure all has been configured properly

```powershell
sudo vgdisplay -v #view complete setup - VG, PV, and LV
sudo lsblk
```

![WhatsApp Image 2024-10-07 at 11 22 47_54976b76](https://github.com/user-attachments/assets/5670317a-706e-4696-a775-8b40f0a3905f)

![WhatsApp Image 2024-10-07 at 11 23 19_232a5c67](https://github.com/user-attachments/assets/ccb565dd-da9f-4931-bca9-51ffe2b6286b)

![WhatsApp Image 2024-10-07 at 11 25 41_27cb09fe](https://github.com/user-attachments/assets/bb2d0ce5-e553-49c5-8f0f-751726d05490)

format the logical volumes using ext4 filesystems

```powershell
sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```
![WhatsApp Image 2024-10-07 at 11 28 37_7b7bd2dd](https://github.com/user-attachments/assets/4bccd943-7785-45a3-979a-be02f9034acd)

Create a directory to store website file

```powershell
sudo mkdir -p /var/www/html
```
![WhatsApp Image 2024-10-07 at 11 25 41_27cb09fe](https://github.com/user-attachments/assets/0b50e373-f7ef-4764-8267-d1a148747e14)

Create another directory for the log files

```powershell
sudo mkdir -p /home/recovery/logs
```

Mount the newly created directory for website files on the app logical volume we earlier created

```powershell
sudo mount /dev/webdata-vg/apps-lv/ /var/www/html/
```

Back up all the files on the logs logical volume before mounting, this is done using rsync utility

```powershell
sudo rsync -av /var/log/. /home/recovery/logs/
```

![WhatsApp Image 2024-10-07 at 11 33 57_6fb3e802](https://github.com/user-attachments/assets/41b6a3d9-61b5-454f-80e2-cba2433326f3)


Mount the .var/logs on the log-lv

```powershell
sudo mount /dev/webdata-vg/logs-lv/   /var/log/
```

Restore the log files back into /var/log/ directory

```powershell
sudo rsync -av /home/recovery/logs/. /var/log
```
![WhatsApp Image 2024-10-07 at 11 48 06_0475a806](https://github.com/user-attachments/assets/969a824f-1505-4288-ab37-97127e3c6d8f)

Ensure that the mount configurations persist after server restart, this can be done by updating the UUID of the /etc/fstab

```powershell
sudo blkid
```
![WhatsApp Image 2024-10-07 at 11 48 52_3290daf1](https://github.com/user-attachments/assets/c6b7cf59-ec95-40d4-bb6c-4a38a985fa1f)


Copy the logs and apps UUID then eplace the UUID for the log-lv with the one you copied , save and exit.

```powershell
sudo vi /etc/fstab
```

test the configuration

```powershell
sudo mount -a
```

Reload the daemom

```powershell
sudo systemctl reload daemon
```

Verify the setup

```powershell
df -h
```

![WhatsApp Image 2024-10-07 at 12 05 12_6f8d1a7f](https://github.com/user-attachments/assets/cd6d08f9-34d6-4dfc-b2a9-9ae81c055cac)

# Step 2 - Prepare the Database Server
Launch a second RedHat EC2 instance that will have a role - DB Server. Repeat the same steps as for the Web Server, but instead of apps-lv, create dv-lv and mount it to /db directory.
follow the same process for apps, server one, instaead of apps create a db aserver and create a db diirectory to save the server

![WhatsApp Image 2024-10-07 at 18 45 35_1e98b620](https://github.com/user-attachments/assets/1c875c23-8e33-4d95-905d-1dbcfb0aad3f)

# Step 3 - Install WordPress on the Web Server EC2
1. Update the repository

```bash
sudo yum -y update
```

2. Install wget apache , and all its dependencies

```bash
sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
```

3. Enable, and start apache

```bash
sudo systemctl enable httpd
```

```bash
sudo systemctl start httpd
```

![WhatsApp Image 2024-10-10 at 10 01 34_69918271](https://github.com/user-attachments/assets/9d6f8a9e-3e36-4e99-8142-ee6ce26ab203)

4. Install the latest version of PHP and it's dependencies using the Remi repository

```bash
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm

sudo yum module list php sudo yum module reset php

sudo yum module enable php:remi-7.4

sudo yum install php php-opcache php-gd php-curl php-mysqlnd

sudo systemctl start php-fpm

sudo systemctl enable php-fpm setsebool -P httpd_execmem 1
```

![WhatsApp Image 2024-10-10 at 10 10 27_ee410abe](https://github.com/user-attachments/assets/797a08b9-f4dd-4281-a6a5-856cea6c0ab7)


![WhatsApp Image 2024-10-10 at 10 16 07_a7e201c5](https://github.com/user-attachments/assets/f9f0b4ef-aea5-42bb-9d42-51e636f1ec99)


![WhatsApp Image 2024-10-10 at 15 21 24_5e73ed3f](https://github.com/user-attachments/assets/5eeadc42-794f-460f-9f9e-50364998f8d2)


GO to the browswer and check the Apache setup

![WhatsApp Image 2024-10-10 at 15 45 52_935f5735](https://github.com/user-attachments/assets/3e74f19a-1350-4d34-9a70-7d670db63895)

5. Download and Copy wordpress to the /var/www/html directory

```bash
sudo wget http://wordpress.org/latest.tar.gz
```

```bash
sudo tar xzvf latest.tar.gz
```

```bash
sudo rm -rf latest.tar.gz
```

```bash
sudo cp wordpress/wp-config-sample.php wordpress/wp-config.php
```
```bash
configure SElinux policies
```

```bash
sudo chown -R apache:apache /var/www/html/wordpress
```
```bash
sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
```

```bash
sudo setsebool -P httpd_can_network_connect=1
```

![WhatsApp Image 2024-10-10 at 15 05 47_fd06c66d](https://github.com/user-attachments/assets/1de941f1-ace5-45e1-a0e2-60c5335e3e15)

6. Start and Enable MySQL
Ensure that the MySQL service is running and is set to start on boot:

```sudo systemctl start mysqld
sudo systemctl enable mysqld
sudo systemctl status mysqld
```

![WhatsApp Image 2024-10-10 at 16 13 13_8d990784](https://github.com/user-attachments/assets/6524cb88-90d1-452c-bef9-a05b1ef17dd6)

7. Configure the Database for WordPress
Secure MySQL Installation
Run the MySQL secure installation script to set up a root password and configure security settings:

```sudo mysql_secure_installation
```

![image](https://github.com/user-attachments/assets/02d4c6ac-4fd0-4715-9989-afea7322e3a2)

Create the WordPress Database and User
Connect to MySQL and create the necessary database and user for WordPress:

```sudo mysql -u root -p
```

```CREATE DATABASE wordpress_db;
CREATE USER 'wordpress'@'IP Address ' IDENTIFIED WITH mysql_native_password BY 'P@ssw0rd1';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wordpress'@'Ip Address' WITH GRANT OPTION;
FLUSH PRIVILEGES;
show databases;
exit
```

![image](https://github.com/user-attachments/assets/862762fb-3038-4c42-8885-cafac13a700b)

6. Configure MySQL Bind Address
For security reasons, set the MySQL bind address to the private IP address of the DB server rather than allowing connections from all IP addresses (0.0.0.0):

```sudo vi /etc/my.cnf
```

Update the bind address in the configuration file, then restart MySQL:

```sudo systemctl restart mysqld
```

Configure WordPress to connect to remote database

Open MySQL port 3306 on the DB Server EC2.
For extra security, access to the DB Server is allowed only from the Web Server IP address. In the inbound rule, /32 is configured as source.

Install mysql server on the Web Server EC2.
WordPress has its own database, therefore it needs a database server to store it's information such as: Username, Email, Passwords, First name and Last name of the users on the wordpress website on a database.

```sudo yum install mysql-server
```

![WhatsApp Image 2024-10-10 at 16 25 07_f68475d2](https://github.com/user-attachments/assets/bb681eba-dc80-4d94-9859-7823ec91535d)

```sudo systemctl start mysqld
sudo systemctl enable mysqld
sudo systemctl status mysqld
```

Open wp-config.php file and edit the database information
```cd /var/www/html
```

```sudo vi wp-config.php
```

```sudo systemctl restart httpd
```

The private IP address of the DB Server is set as the DB_HOST because the DB Server and the Web Server resides in the same subnet which makes it possible for them to communicate directly. The private IP address is not an internet routable address.

To connect to the MySQL database from the web server, use the following command:

```sudo mysql -h <IP Address> -u wordpress -p
```
Once connected, you can verify that the database is accessible by running:

show databases;
exit;


![WhatsApp Image 2024-10-10 at 16 28 53_2600ebc5](https://github.com/user-attachments/assets/ddaba7a9-e85d-4601-ba6a-90a969c280f5)

Access WordPress Installation
Now, access the WordPress installation by navigating to the public IP address of the web server in a web browser. Follow the prompts to complete the WordPress installation process.


![WhatsApp Image 2024-10-10 at 16 35 17_17e30fbf](https://github.com/user-attachments/assets/a90e9fb9-3d45-4322-ba36-97f471b717b3)


![Screenshot (85)](https://github.com/user-attachments/assets/a9587d92-6595-426c-886c-5d74d0041acb)


![image](https://github.com/user-attachments/assets/f33f7950-d715-4849-94df-6089b18f9ab7)


![image](https://github.com/user-attachments/assets/5d7487a9-92c3-4d52-a2dc-6e512a4652b7)






