# Wordpress-website-on-AWS
Wordpress website on AWS
- Lab Overview And High Level Design

I have deployed the word press website using various AWS services, like VPC with public and private subnets EC2, Security groups, Nat Gateways, RDS, Application Load Balancer, Auto scaling group, EFS and more

Build a Three-Tier AWS Network VPC from Scratch:
The infrastructure here is divided into three sections :
1) Tier 1: Public Subnet - The public subnet has the NAT Gateway, Load Balancer and the Bastion host
2) Tier 2: Private Subnet 1 - This subnet has the web servers(EC2 instances).
3) Tier 3: Private Subnet 2 - This subnet has the database.

For high availability , these subnets are duplicated in another Availability zone as well.

4) An Internet Gateway is used for the communication between the internet and the VPC. The Public Route table is associated with the public subnets and routes teh traffic through the Internet Gateway. The Main route table is associated with the private subnets.


Setup
 Create the VPC :
 1) I have created the VPC in N.Virginia region and named it as Dev VPC . The CIDR block is 10.0.0.0/16. Other settings arre not changed while creating the VPC. Once VPC is created, enable the DNS hostnames, we do that so that the instances with public IPs will get the corresponding public DNS hostnames.
 2) Next thing is to create an Internet Gateway for the VPC, to allow VPC communication with the internet. I named it as Dev Internet Gateway , once created , attach it to the created Dev VPC. One Internet Gateway can only be attached to one VPC at a time.
 3) Create the public subnet in the first Availability Zone. I named the first subnet as 'Public Subnet AZ1' in us east1 a AZ and chose the CIDR block 10.0.0.0/24.
 4) Create the public subnet in the second Availability Zone. I named the second subnet as 'Public Subnet AZ2' in us east1 b AZ and chose the CIDR block 10.0.1.0/24.
 5) Once public subnets are created. Enable auto-assign public ip settings in the public subnets, so that the EC2 instances launched in the public subnets will have the public ip4s assigned to them. This can be done by selecting the subnet and going to Actions.
 6) Next, create a Route table and I named it as Public route table . Whenever a VPC is created, a route table is created automatically thats called the Main route table, its private by default Once the new rouet table is created, create a public route to the route table, this will help in routing traffic to the internet.
 7) To add the route to the internet, edit routes and add a new route, with destination as 0.0.0.0/0 and target select the Dev Internet Gateway.
 8) Next, associate the two subnets with the public route table. This can be done by selecting subnet associations on the public route table page, select Edit subnet associations and select the two public subnets created and save.
 9) Now, create 4 private subnets in the Dev VPC. First subnet is PrivateAppSubnet AZ1 in us-east1-a, with ipv4 cidr 10.0.2.0/24. Second subnet is PrivateAppSubnet AZ2 in us-east1-b, with ipv4 cidr 10.0.3.0/24. Third subnet is PrivateDataSubnet AZ1 in us-east1-a, with ipv4 cidr 10.0.4.0/24. Fourth subnet is PrivateDataSubnet AZ2 in us-east1-b, with ipv4 cidr 10.0.5.0/24. I did not associate these private subnets to the public route table, by default they are associated to the main route table.


NAT Gateway
The NAT Gateway allows the instances in the private app and data subnets to access the internet.
1) I have created the NAT Gateway first in the public subnet in AZ1 and a private route table in AZ1 . A route has been then added to this route table to route traffic to the internet through the NAT Gateway. Then the route table is associated with private subnets in the AZ1.
2) Same process is repeated in the AZ2, a NAT Gateway, a private route table are created in the AZ2 and the route has been then added to this route table to route traffic to the internet through the NAT Gateway. Then the route table is associated with private subnets in the AZ2.
3) To add route, destination is 0.0.0.0/0 and target will be the NAT Gateway in the respective AZs.


Security Groups
I have created the following security groups for this project :
1) ALB SG : For the application load balancer, with following rules :
 ![image](https://user-images.githubusercontent.com/31481968/231736465-d131388f-55f4-40de-a726-7b4b78747eb5.png)
2) SSH SG : To allow the SSH traffic from my own IP:
 ![image](https://user-images.githubusercontent.com/31481968/231736997-77c3c009-e4f0-4ba0-870e-b03c5208f3a6.png)
3) Webserver SG : To allow the traffic from application load balancer :
 ![image](https://user-images.githubusercontent.com/31481968/231737708-be5bc331-c49f-400e-bb9f-f912231ee042.png)
4) Database SG : To allow the traffic from the webserver:
 ![image](https://user-images.githubusercontent.com/31481968/231738084-5497bc20-8a9f-4f43-82ad-5cac12c45301.png)
5) EFS SG  : To allow the NFS traffic from webserver , SSH traffic from my own IP and NFS traffic from the EFS SG itself. By allowing the security group to contact itself,  a permission is created for the resources within the security group to communicate with the EFS file system. 
 ![image](https://user-images.githubusercontent.com/31481968/231739071-ccf32264-08f8-4f43-8a1a-0c45893ca849.png)

      
   RDS 
   
   RDS instances need to be created in the private data subnets.
   1) First I created a subnet group, which defines in which subnet I want my RDS instances. While creating that, select the AZs and the private data subnets in the respective AZs.
   2) Then create database using MySQL 5.7.37. I did not want to go out of free tier hence I chose not to create a stand by database and chose the Single DB instance option. For the DB instance class option, I chose burstable class option.
   3)  Choose the dev VPC and the correct subnet group created and also remove all other SGs if attached and only attach the DB SG. For the availability zone, I chose us-east1b to create my master DB isntance. 
   4)  Then, go to additional configuration and give the db a name.
   
   
   Elastic File System
   I have created an EFS called the Dev EFS, so that the web servers can have access to the shared files. The EFS mount targets are in each AZ . The webserves will use the mount target to connect to the EFS.
   This EFS used, will allow the webservers to pull the application code and configuration files from the same location.
   To create, cgo yo EFS, click on create EFS and then select customize.
   While mounting targets, select the Dev VPC and the two AZs and private datasubnets of the respective AZs. Remove the default security groups and attach the EFS security groups to the targets.
   
   SSH Into EC2 Instances
   For this, I created a key pair from the AWS console, as I was using a Windows machine, I used Putty, so I created a ppk file and got a public key and a private key.
   
   Launch the WordPress setup server
  1) I created the EC2 instance called Setup server, in the public subnet in AZ1 to install the wordpress website and move the files to EFS.
  2) I selected a t2.micro and chose the  key pair created in the previous step.
  3) In the network settings, select the Dev VPC and the public subnet in AZ1.
  4) For the security groups, select the already created security groups, select SSH SG, ALB SG and the Webserver SG.

  Install Wordpress
  To do that, first I needed to SSH into the setup server using Putty. For that, copy the public Ip4 from the launched EC2 and then in putty, I entered the hostname as   ec2-user@publipv4 and added the downloaded private key to the private key file for authentication.
  Once I SSH into my EC2, I ran the below commands to install the server.
  1) Create the html directory :
     sudo su
     yum update -y
     mkdir -p /var/www/html
 2) Mount efs to the directory
    First go to the efs created, click Attach and copy the mount information just the below part:
    
    Then execute the below command :
    sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-03c9b3354880b36a6.efs.us-east-1.amazonaws.com:/ /var/www/html.
    
    The copied part needs to be replaced withe respective efs, mine was fs-03c9b3354880b36a6.efs.us-east-1.amazonaws.com.
    Mounting can be verified by executing the command df -h, if mounted properly the efs file system information will be visible.
    
  3) Install apache web server 
     sudo yum install -y httpd httpd-tools mod_ssl
     sudo systemctl enable httpd 
     sudo systemctl start httpd
     
  4)install php v7.4
    sudo amazon-linux-extras enable php7.4
    sudo yum clean metadata
    sudo yum install php php-common php-pear -y
    sudo yum install php-{cgi,curl,mbstring,gd,mysqlnd,gettext,json,xml,fpm,intl,zip} -y
    
   5)Install mysql5.7
    sudo rpm -Uvh https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
    sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
    sudo yum install mysql-community-server -y
    sudo systemctl enable mysqld
    sudo systemctl start mysql

   6)Set permissions
    sudo usermod -a -G apache ec2-user : Give sudo granted 
    sudo chown -R ec2-user:apache /var/www : Change ownership
    sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
    sudo find /var/www -type f -exec sudo chmod 0664 {} \;
    chown apache:apache -R /var/www/html 
    
    7)Download wordpress files and move them to html directory
    wget https://wordpress.org/latest.tar.gz
    tar -xzf latest.tar.gz
    cp -r wordpress/* /var/www/html/
    
    8)create the wp-config.php file
    cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
    
    9)edit the wp-config.php file to enter the information of the RDS instance
    nano /var/www/html/wp-config.php
    Go to the RDS instance in the console, copy the database name, db username and password then enter in the file in the respective fields
    For DB hostname field in the file, copy the RDS endpoint from the Connectivity and Security tab.
    
    Press CtrlX, Y, then Enter, this will save the file and exit.
    
    10) Restart the webserver
    service httpd restart
    
    Now on pasting my ec2 public ip4 on a new tab , I could see the wordpress site.
    
    
    Launch EC2 instances in private subnet:
    1. Create EC2s in Private subnet 1 and 2 with webserver SG , with following user data script. The user data script has the same commands which were used to install wordpress on the setup server:
    #!/bin/bash
    yum update -y
    sudo yum install -y httpd httpd-tools mod_ssl
    sudo systemctl enable httpd 
    sudo systemctl start httpd
    sudo amazon-linux-extras enable php7.4
    sudo yum clean metadata
    sudo yum install php php-common php-pear -y
    sudo yum install php-{cgi,curl,mbstring,gd,mysqlnd,gettext,json,xml,fpm,intl,zip} -y
    sudo rpm -Uvh https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
    sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
    sudo yum install mysql-community-server -y
    sudo systemctl enable mysqld
    sudo systemctl start mysqld
    echo "fs-03c9b3354880b36a6.efs.us-east-1.amazonaws.com:/ /var/www/html nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 0 0" >> /etc/fstab
    mount -a
    chown apache:apache -R /var/www/html
    sudo service httpd restart
    
    I named the instances as Webserver AZ1 and Webserver AZ2.
    Also select the created key pair while launching the instances.
    
    Create Application Load Balancer:
    1. Create Target Group called Dev TG, select the protocol as HTTP and choose Dev VPC, register the two instances (Webserver AZ1 and Webserver AZ2) as the targets.
    2. Now create the application load balancer, I named it Dev ALB. Its an internet facing type with ip address type ipv4.
    3. Select the Dev VPC and the availability zones us-east1-a (then the public subnet AZ1 ) and us-east1-b (then the public subnet AZ2 ).
    4. Select the ALB SG for the security groups.
    5. Add HTTp listener and select the Dec TG for the target group.
    
    Once the ALB is provisioned, the website can now be accessed by the DNS name of the ALB. This domain name can be  updated in the word press settings as well by    appending wp-admin to the domain name on the address bar.
   Now, the set up server in the public subnet can be terminated.
   
   Create an Autoscaling Group
   
   This is done to dynamically scale the infrastructure. 
   1)For this, first I terminated the EC2 instamces called Webserver AZ1 and Webserver AZ2.
   2)Create Launch template, I named it as Dev launch template and select Amazon linux 2 AMI and t2.micro as instance type.
   3)Select the created key pair and for security group, select the Webserver SG.
   4)In the user data section,I added the same script I used while creating the webservers.
   5)Once launch template is created, create the Auto scaling group.
   6)I named the group as Dev ASG and selected the created Dev launch template.
   7)Select the Dev VPC and for subnets select the Private AppSubnet AZ1 and Private AppSubnet AZ2.
   8) I then attached the group to the load balancer and selected the Dev TG.
   9) For health checks, select the health check type as ELB and timeout 300 seconds.
   10) For the group canacity, I selected the desired as 2, minimum as 1 and maximum as 4.
   11) Next, I added a notification as well and created an SNS topic and left all the event types checked.
   12) Also, add the tag as ASG-Webservers.
   13) Once the group is successfully created, I had two EC2 instances running , launched by the auto scaling group, can be verified by the tag name ASG-Webservers.

Now the website is up and running again.
   
    

    
  
