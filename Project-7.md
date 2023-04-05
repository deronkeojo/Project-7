### DEVOPS TOOLING WEBSITE SOLUTION
### The task of the project is to implement a tooling website solution which makes access to Devops tools within the corporate infrastructure easily accessible.

The solution consists of the following components:

1. Infrastructure: AWS
2. Webserver Linux: Red Hat Enterprise Linux 8
3. Database Server: Ubuntu 20.04 + MySQL
4. Storage Server: Red Hat Enterprise Linux 8 + NFS Server
5. Programming Language: PHP
6. Code Repository: GitHub
The diagram below shows 3 stateless Web Servers sharing a common database and also accessing the same files using Network File System as a shared filed storage, which can also be used for back up in case a server crashes. As a result the content of the server is secured and safe.

![3 Tier Architecture](./images/3%20Tier%20Architecture.png)
### STEP 1 – PREPARE NFS SERVER
### Spin up a new EC2 instance with RHEL Linux 8 Operating System.

Configure Logical Volume Management on the Server.

Instead of formating the disks as ext4 you will have to format them as xfs

Attach three volumes with the the same availability

-Inspect what block devices are attached to the server lsblk

![lsblk](./images/lsblk.png)

### -Use gdisk utility to create a single partition on each of the 3 disks
`sudo gdisk /dev/xvdf`
`sudo gdisk /dev/xvdg`
`sudo gdisk /dev/xvdh`
![sudo gdisk](./images/sudo%20gdisk.png)

### -View the newly configured partition on each of the 3 disks using:
`lsblk command`
![lsblk commands](./images/lsblk%202.png)

### Install lvm2 
`sudo yum install lvm2`
![Install lvm2](./images/sudo%20yum%20install%20lvm2%20-y.png)

### Use the pvcreate utility to mark each of 3 disks as physical volumes (PVs) to be used by LVM:
`sudo pvcreate /dev/xvdf1 /dev/xvdg1 /dev/xvdh1`
![sudo pvcreate](./images/sudo%20pvcreate.png)

### Add all 3 PVs to a volume group:
`sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1`
### -Verify that the VG has been created successfully
`sudo vgs`
![vgcreate](./images/sudo%20vgcreate.png)

### Create 3 logical volumes. lv-opt lv-apps, and lv-logs.
`sudo lvcreate -n apps-lv -L 9G webdata-vg sudo lvcreate -n logs-lv -L 9G webdata-vg sudo lvcreate -n opt-lv -L 9G webdata-vg`
### Verify that the Logical Volume had been created successfully:
`sudo lvs`
![logical volumes](./images/sudo%20lvcreate.png)

### Use mkfs.xfs to format the logical volumes with xfs filesystem with the commands below:
`sudo mkfs -t xfs /dev/webdata-vg/apps-lv sudo mkfs -t xfs /dev/webdata-vg/logs-lv sudo mkfs -t xfs /dev/webdata-vg/opt-lv`
![sudo mkfs](./images/sudo%20mkfs.png)

### Create mount points on /mnt directory for the logical volumes as follow:
### Mount lv-apps on /mnt/apps – To be used by webservers Mount lv-logs on /mnt/logs – To be used by webserver logs Mount lv-opt on /mnt/opt – To be used by Jenkins server in Project 8
### Create /mnt directory for the logical volumes

`sudo mkdir /mnt/apps sudo mkdir /mnt/apps sudo mkdir /mnt/opt`
### Mount the logical volumes on the mnt directories:
`sudo mount /dev/webdata-vg/apps-lv /mnt/apps sudo mount /dev/webdata-vg/logs-lv /mnt/logs sudo mount /dev/webdata-vg/opt-lv /mnt/opt`
![sudo mount](./images/sudo%20mount.png)

### Retrieve the UUID of the devices and updated the ftsab file:
`sudo blkid sudo vi /etc/fstab`

### Confirm if the mount is okay and then reload:
`sudo mount -a sudo systemctl daemon-reload`

### Install NFS server, configure it to start on reboot and make sure it is u and running
`sudo yum -y update sudo yum install nfs-utils -y sudo systemctl start nfs-server.service sudo systemctl enable nfs-server.service sudo systemctl status nfs-server.service`
![install nfs server](./images/Install%20NFS%20server.png)
![install nfs server](./images/install%20nfs%20server%202.png)

### Export the mounts for webservers’ subnet cidr to connect as clients. For simplicity, you will install your all three Web Servers inside the same subnet, but in production set up you would probably want to separate each tier inside its own subnet for higher level of security. To check your subnet cidr – open your EC2 details in AWS web console and locate ‘Networking’ tab and open a Subnet link:
### Make sure we set up permission that will allow our Web servers to read, write and execute files on NFS:
`sudo chown -R nobody: /mnt/apps sudo chown -R nobody: /mnt/logs sudo chown -R nobody: /mnt/opt`
`sudo chmod -R 777 /mnt/apps sudo chmod -R 777 /mnt/logs sudo chmod -R 777 /mnt/opt`
`sudo systemctl restart nfs-server.service`

### -Configure access to NFS for clients within the same subnet (example of Subnet CIDR – 172.31.32.0/20 ):
`sudo vi /etc/exports`
`/mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash) /mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash) /mnt/opt <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)`
`sudo exportfs -arv`
### Check which port is used by NFS and open it using Security Groups (add new Inbound Rule)
`rpcinfo -p | grep nfs`
![install nfs server](./images/sudo%20vi%20.png)

### In order for NFS server to be accessible from the client, open the following ports and allow access from the web subcidrs: TCP 111, UDP 111, UDP 2049

![inbound rules](./images/inbound%20rules.png)

### STEP 2 — CONFIGURE THE DATABASE SERVER
### Install mysql-server:
`sudo apt install mysql-server sudo systemctl start mysql sudo systemctl enable mysql sudo systemctl status mysql sudo mysql_secure_installation`

### Create a database called tooling, a database user (webaccess) and grant permission to webaccess user on tooling database to do anything only from the webservers subnet cidr.

`Create database tooling;`
`CREATE USER 'webaccess'@'172.31.16.0/20' IDENTIFIED BY 'password';`
`grant all privileges on tooling.* to 'webaccess' @'172.31.16.0/20';`
`flush privileges;`


### Screenshot showing database has been created

![Screenshot](./images/create%20database.png)
![show databases](./images/show%20databases.png)

### Set the bind address to 0000 in mysql.cnf

`sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf`

### Restart mysql
`sudo systemctl restart mysql`
![sudo systemctl restart](./images/show%20databases%201.png)

![tooling](./images/tooling.png)

### Open mysql/aurora in the security group

## STEP 3 -PREPARE THE WEBSERVER
### Open mysql/aurora in the security group
### During this process you will do the following:
### Configure NFS client on 3 web servers
### Deploy a Tooling application to your Web Servers into a shared NFS folder
### Configure the Web Servers to work with a single MySQL database
### Launch 3 new EC2 instance with RHEL 8 Operating System, update the respositries and install NFS.

`sudo yum install nfs-utils nfs4-acl-tools -y sudo systemctl start nfs-server sudo systemctl enable nfs-server sudo systemctl status nfs-server`

### Create www directory
`sudo mkdir /var/www`

### Mount /var/www:
`sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/apps /var/www`

### Verify that NFS was mounted succesfully:
`df -h`

![verify](./images/sc.png)

### To ensure that the changes will persist on Web Server after reboot open the fstab file:
`sudo vi /etc/fstab`

### Enter in the below and save:
`<NFS-Server-Private-IP-Address>:/mnt/apps /var/www nfs defaults 0 0`

### Reload server with below command:
`sudo systemctl daemon-reload`

### Repeat the steps above on the 2 Web Servers
### Install Remi’s repository, Apache and PHP:
`sudo yum install httpd -y`
`sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm`
`sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm`
`sudo dnf module reset php`
`sudo dnf module enable php:remi-7.4`
`sudo dnf install php php-opcache php-gd php-curl php-mysqlnd`
`sudo systemctl start php-fpm`
`sudo systemctl enable php-fpm`
`sudo setsebool -P httpd_execmem 1`


### To verify whether NFS was mounted correctly create a new file called test.md from your web server and check whether it is visible within your NFS server:
`sudo touch /var/www/test.md`

### The log file for Apache is located on the Web Server. The folder contains content and once mounted this will be deleted. In order to avoid this move the content and create a new httpd with the below command:
`sudo mv /var/log/httpd /var/log/httpd.bak`
`sudo mkdir /var/log/httpd`

### Mount to the NFS server’s export for logs to var/log/httpd, updated the fstab to ensure changes will persist on web server after reboot and reloaded.
`sudo mount -t nfs -o rw,nosuid <NFS Private IP address>:/mnt/logs /var/log/httpd`

### To make sure changes persist after reboot update the fstab
`sudo vi /etc/fstab`

### #Update with the below
### 72.31.30.141:/mnt/logs /var/log/httpd nfs defaults 0 0

### Reload the system
`sudo systemctl daemon-reload`

### Copy the content of httpd.bak into httpd folder since mount has already taken place 
`sudo cp -R /var/log/httpd.bak/. /var/log/httpd`
![Reload](./images/screenshot%201.png)

### Install git and clone tooling source code:
`sudo yum install git`

### Fork the tooling source code from https://github.com/darey-io/tooling to your Github account
### Clone the directory , this will create a folder named tooling.
`git clone https://github.com/Mariam-Lawal5/tooling.git`

### Deploy the content of the html folder of the Tooling directory to /var/www/html
sudo cp -R ~/tooling/html/. /var/www/html`

`sudo cp -R /var/log/httpd.bak/. /var/log/httpd`
![git](./images/git.png)

### Disable Apache default page.
`sudo mv /etc/httpd/conf.d/welcome.conf /etc/httpd/conf.d/welcome.conf_backup`

### And restart httpd
`sudo systemctl restart httpd`

### If unable to restart httpd do the below:
### Disable SELinux sudo setenforce 0 and to make this change permanent – open config file /etc/sysconfig/selinux and set SELINUX=disabled. Restart and check apache status
`sudo setenforce 0`
`sudo vi /etc/sysconfig/selinux`

![screen](./images/screen.png)

### -Install mysql server
`sudo yum install mysql-server`

### Update the website’s configuration with tooling script to connect to the database /var/www/html/functions.php file
`sudo vi /var/www/html/functions.php file`

![connect to database](./images/connect%20to%20database.png)

### Connect with the database using the tooling-db.sql script within the tooling directory t using below command
`mysql -h <databse-private-ip> -u <db-username> -p tooling < tooling-db.sql`
### Access website with the the public ip address of all webservers and login
![Login wordpress](./images/wordpress.png)

