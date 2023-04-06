# Wordpress-website-on-AWS
Wordpress website on AWS
- Lab Overview And High Level Design

I have deployed the word press website using various AWS services, like VPC with public and private subnets EC2, Security groups, Nat Gateways, RDS, Application Load Balancer, Auto scaling group, EFS and more

1. Build a Three-Tier AWS Network VPC from Scratch:
The infrastructure here is divided into three sections :
1) Tier 1: Public Subnet - The public subnet has the NAT Gateway, Load Balancer and the Bastion host
2) Tier 2: Private Subnet 1 - This subnet has the web servers(EC2 instances).
3) Tier 3: Private Subnet 2 - This subnet has the database.

For high availability , these subnets are duplicated in another Availability zone as well.

4) An Internet Gateway is used for the communication between the internet and the VPC. The Public Route table is associated with the public subnets and routes teh traffic through the Internet Gateway. The Main route table is associated with the private subnets.


2. Setup
 Create the VPC :
 1) I have created the VPC in N.Virginia region and named it as Dev VPC . The CIDR block is 10.0.0.0/16. Other settings arre not changed while creating the VPC. Once VPC is created, enable the DNS hostnames, we do that so that the instances with public IPs will get corresponding public DNS hostnames.
 2) Next thing is to create an Internet Gateway for the VPC, to allow VPC communication with the internet. I named it as Dev Internet Gateway , once created , attach it to the created Dev VPC. One Interner Gateway can only be attached to one VPC at a time.
 3) Create the public subnet in the first Availability Zone. I named the first subnet as 'Public Subnet AZ1' in us east1 a AZ and chose the CIDR block 10.0.0.0/24.
 4) Create the public subnet in the second Availability Zone. I named the second subnet as 'Public Subnet AZ2' in us east1 b AZ and chose the CIDR block 10.0.1.0/24.
 5) Once public subnets are created. Enable auto-assign public ip settings in the public subnets, so that the EC2 instances launched in the public subnets will have the public ip4s assigned to them. This can be done by selecting the subnet and going to Actions.
 6) Next, create a Route table and I named it as Public route table . Whenever a VPC is created, a rout table is created automatically thats called the Main route table, its private by default Once the new rout table is created, create a public route to the route table, this will help in routing traffic to the internet.
 7) To add the route to the internet, edit routes and add a new route, with destination as 0.0.0.0/0 and target select the Dev Internet Gateway.
 8) Next, associate the two subnets with the public route table. This can be done by selecting subnet associations on the public route table page, selecy Edit subnet associations and select the two public subnets created and save.
 9) Now, create 4 private subnets in the Dev VPC. First subnet is PrivateAppSubnet AZ1 in us-east1-a, with ipv4 cidr 10.0.2.0/24. Second subnet is PrivateAppSubnet AZ2 in us-east1-b, with ipv4 cidr 10.0.3.0/24. Third subnet is PrivateDataSubnet AZ1 in us-east1-a, with ipv4 cidr 10.0.4.0/24. Fourth subnet is PrivateDataSubnet AZ2 in us-east1-b, with ipv4 cidr 10.0.5.0/24. I did not associate these private subnets to the public route table, by default they are associated to the main route table.


NAT Gateway
The NAT Gateway allows the instances in the private app and data subnets to access the internet.
1) I have created the NAT Gateway first in the public subnet in AZ1 and a private route table in AZ1 and AZ2. A route has been then added to this route table to route traffic to the internet through the NAT Gateway. Then the route table is associated with private subnets in the AZ1.
2) Same process is repeated in the AZ2, a NAT Gateway, a private route table are created in the AZ2 and the route has been then added to this route table to route traffic to the internet through the NAT Gateway. Then the route table is associated with private subnets in the AZ2.
3) To add route, destination is 0.0.0.0/0 and target will be the NAT Gateway in the respective AZs.


Security Groups
I have created the following security groups for this project :
1) ALB SG : For the application load balancer, with following rules :
Insert picture
2) SSH SG : To allow the SSH traffic from my own IP:
  Insert
3) Webserver SG : To allow the traffic from application load balancer :
   Insert
4) Database SG : To allow the traffic from the webserver:
   Insert
5) EFS SG  : To allow the NFS traffic from webserver , SSH traffic from my own IP and NFS traffic from the EFS SG itself. By allowing the security group to contact itself,  a permission is created for the resources within the security group to communicate with the EFS file system. 
   Insert
   
   
   RDS 
   RDS instances need to be created in the private data subnets.
   1) First I craeted a subnet group, which defines in which subnet I want my RDS inatances. While creating that, select oth the AZs and the private data subnets in the respective AZs.
   2) Then create database using MySQL 5.7.36. I did not want to go out of free tier hence I chose not to create a stand by database and chose the Single DB instance option. For the DB instance class option, I chose burstable class option.
   3)  Choose the dev VPC and the correct subnet group created and also remove all other SGs if attached and only attach the DB SG. For the availability zone, O chose us-east1b to create my master DB isntance. 
   4)  Then, go to additional configuration and give the db a name.
   
   
   Elastic File System
   I have created an EFS called the Dev EFS, so that the web servers can have access to the shared files. The EFS mount targets are in each AZ . The webserves will use the mount target to connect to the EFS.
   While mounting targets, select the Dev VPC and the two AZs and private datasubnets of teh respective AZs. Remove the default security groups and attach the EFS security groups to the targets.
