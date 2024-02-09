
## Infrastructure Diagram

- **AWS Infras**
![image](https://drive.google.com/uc?export=view&id=1tvrUAoCU6QafY6VFrh_CkxDEsPqmb7R7)

---
## Provision the environment and review tasks

Click the link to deploy the **Base-infras deployment:** [Stack deployment](https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/quickcreate?templateURL=https://aws-labs-resources-kng318.s3.amazonaws.com/cfn-templates/DemoVPC.yaml&stackName=DemoVPC)

After the stack is in `CREATE_COMPLETE` state, this stack will create necessary infrastracture for this demo

---
### RDS Setup

#### Create RDS Subnet Group

A subnet group is what allows RDS to select from a range of subnets to put its databases inside  
In this case you will give it a selection of 3 subnets sn-db-A / B and C  
RDS can then decide freely which to use.

Move to the RDS Console <https://console.aws.amazon.com/rds/home?region=us-east-1#>  
Click `Subnet Groups`  
Click `Create DB Subnet Group`  
Under `Name` enter `WordPressRDSSNG`  
Under `Description` enter `RDS Subnet Group for WordPress`  
Under `VPC` select `DemoVPC`

Under `Add subnets` In `Availability Zones` select `us-east-1a` & `us-east-1b` & `us-east-1c`  
Under `Subnets` check the box next to

- 10\.16.16.0/20 (this is sn-db-A)
- 10\.16.80.0/20 (this is sn-db-B)
- 10\.16.144.0/20 (this is sn-db-C)

Click `Create`

#### Create RDS Instance

In this sub stage of the demo, you are going to provision an RDS instance using the subnet group to control placement within the VPC.  
In this labs, the instruction below will guide you to deploy single AZ RDS instance, to keep costs low. But for production you would normally use Multi AZ instead.

Click `Databases`  
Click `Create Database`  
Click `Standard Create`  
Click `MySql`  
Under `Version` select `MySQL 8.0.32`

Scroll down and select `Free Tier` under templates *this ensures there will be no costs for the database but it will be single AZ only*

Under `Db instance identifier` enter `DemoWordPressDB` under `Master Username` enter `demowpuser`  
Under `Master Password` and `Confirm Password` enter `Dem0W0rdPressPassW0rD`

Under `DB Instance size`, then `DB instance class`, then `Burstable classes (includes t classes)` make sure db.t3.micro or db.t2.micro or db.t4g.micro is selected

Scroll down, under `Connectivity`, `Compute resource` select `Don’t connect to an EC2 compute resource`  
under `Connectivity`, `Network type` select `IPv4`  
under `Connectivity`, `Virtual private cloud (VPC)` select `DemoVPC`  
under `Subnet group` that `wordpressrdssng` is selected, which is the name of the rds subnet group you created above  
Make sure `Public Access` is set to `No`  
Under `VPC security groups` make sure `choose existing` is selected, remove `default` and add `DemoVPC-SG-Database`  
Under `Availability Zone` set `us-east-1a`

**IMPORTANT .. DON'T MISS THIS STEP** Scroll down past `Database Authentication` & `Monitoring` and expand `Additional configuration`  
in the `Initial database name` box enter `demowpuserdb`  
Scroll to the bottom and click `create Database`

\*\* this will take anywhere up to 30 minutes to create ... you can go to create the EFS while waiting or go make some coffee  !!!! \*\*

After process complete, click the `DemoWordPressDB` instance  
Note down the `endpoint` for the next setup

---
### Create EFS File System

Move to the EFS Console <https://console.aws.amazon.com/efs/home?region=us-east-1#/get-started>  
Click on `Create file System`  
We're going to step through the full configuration options, so click on `Customize`

#### File System Settings

For `Name` type `Demo-WordPress-Media-EFS`  
This is critical data so for `Storage Class` leave this set to `Regional` and ensure `Enable Automatic Backups` is enabled.  
for `LifeCycle management` ...  
for `Transition into IA` set to `30 days since last access`  
for `Transition out of IA` set to `None`  
Untick `Enable encryption of data at rest` .. in production you would leave this on, but for this demo which focusses on architecture it simplifies the implementation.

Scroll down into the `Performance settings`  
For `throughput modes` choose `Bursting`  
Expand `Additional Settings` and ensure `Performance Mode` is set to `General Purpose`  
Click `Next`

#### Network Settings

In this part you will be configuing the EFS `Mount Targets` which are the network interfaces in the VPC which your instances will connect with.

In the `Virtual Private Cloud (VPC)` dropdown select `DemoVPC`  
You should see 3 rows in `Mount targets`.  
Make sure `us-east-1a`, `us-east-1b` & `us-east-1c` are selected in each row.  
In `us-east-1a` row, select `sn-app-A` in the subnet ID dropdown, and in the security groups dropdown select `DemoVPC-SGEFS` & remove the default security group  
In `us-east-1b` row, select `sn-app-B` in the subnet ID dropdown, and in the security groups dropdown select `DemoVPC-SGEFS` & remove the default security group  
In `us-east-1c` row, select `sn-app-C` in the subnet ID dropdown, and in the security groups dropdown select `DemoVPC-SGEFS` & remove the default security group
Click `next`

#### File system policy

Leave all these options in the as default and click `next`

We wont be setting a file system policy so click `Create`

The file system will start in the `Creating` State and then move to `Available` once it does..  
Click on the file system to enter it and click `Network`  
Scroll down and all the mount points will show as `creating` keep hitting refresh and wait for all 3 to show as available before moving on.

Note down the `fs-XXXXXXXX` or `DNS name` (either will work) once visible at the top of this screen, you will need it in the next step.

---
### Create the load balancer

Move to the EC2 console <https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Home>:  
Click `Load Balancers` under `Load Balancing`  
Click `Create Load Balancer`  
Click `Create` under `Application Load Balancer`  
Under name enter `DemoWordPressALB`  
Ensure `internet-facing` is selected  
Ensure `ipv4` selected for `IP address type`

Under `Network Mapping` select `DemoVPC` in the `VPC Dropdown`  
Check the boxes next to `us-east-1a` `us-east-1b` and `us-east-1c`  
Select `sn-pub-A`, `sn-pub-B` and `sn-pub-C` for each.

Scroll down and under `Security groups` remove `default` and select the `DemoVPC-SGLoadBalancer` from the dropdown.

Under `Listener and Routing` Ensure `Protocol` is set to `HTTP` and `Port` is set to `80`.  
Click `Create target group` which will open a new tab  
for `Target Type` choose `Instances` for Target group name choose `DemoWordPressALBTG`  
For `Protocol` choose `HTTP`  
For `Port` choose `80`  
Make sure the `VPC` is set to `DemoVPC`  
Check that `Protocol Version` is set to `HTTP1`

Under `Health checks` for `Protocol` choose `HTTP` and for `Path` choose `/`  
Click `Next`  
We wont register any right now so click `Create target Group`  
Go back to the previous tab where you are creating the Load Balancer Click the `Refresh Icon` and select the `DemoWordPressALBTG` item in the dropdown.  
Scroll down to the bottom and click `Create load balancer`  
Click `View Load Balancer` and select the load balancer you are creating.  
Scroll down and copy the `DNS Name` into your clipboard

---
### Parameter Store

After we've done setup the RDS, EFS, ALB, now we'll store all the nessesary parameters into the Parameter Store for secure used, In the LT we wont enter directly these value into the User Data, we'll retrieve it for securely and conviniently

Move to the Systems Manager service: https://us-east-1.console.aws.amazon.com/systems-manager/home?region=us-east-1#

##### Create Parameter - DBUser (the login for the specific wordpress DB)

Click `Create Parameter` 
Set `Name` to `/Demo/Wordpress/DBUser` 
Set `Description` to `Wordpress Database User`  
Set `Tier` to `Standard`  
Set `Type` to `String`  
Set `Data type` to `text`  
Set `Value` to `demowpuser`  
Click `Create parameter`

##### Create Parameter - DBName (the name of the wordpress database)

Click `Create Parameter` 
Set `Name` to `/Demo/Wordpress/DBName` 
Set `Description` to `Wordpress Database Name`  
Set `Tier` to `Standard`  
Set `Type` to `String`  
Set `Data type` to `text`  
Set `Value` to `demowpuserdb`  
Click `Create parameter`

##### Create Parameter - DBEndpoint (the endpoint for the wordpress DB .. )

Click `Create Parameter` 
Set Name to `/Demo/Wordpress/DBEndpoint`  
Set `Descripton` enter `WordPress DB Endpoint Name`  
Set `Tier` select `Standard`  
Set `Type` select `String`  
Set `Data Type` select `text`  
Set `Value` enter the RDS endpoint you have copied
Click `Create Parameter`

##### Create Parameter - DBPassword (the password for the DBUser)

Click `Create Parameter` 
Set `Name` to `/Demo/Wordpress/DBPassword` 
Set `Description` to `Wordpress DB Password`  
Set `Tier` to `Standard`  
Set `Type` to `SecureString`  
Set `KMS Key Source` to `My Current Account`  
Leave `KMS Key ID` as default (should be `alias/aws/ssm`).  
Set `Value` to `Dem0W0rdPressPassW0rD`  
Click `Create parameter`

##### Create Parameter - DBRootPassword (the password for the database root user, used for self-managed admin)

Click `Create Parameter` 
Set `Name` to `/Demo/Wordpress/DBRootPassword` 
Set `Description` to `Wordpress DBRoot Password`  
Set `Tier` to `Standard`  
Set `Type` to `SecureString`  
Set `KMS Key Source` to `My Current Account`  
Leave `KMS Key ID` as default (should be `alias/aws/ssm`).  
Set `Value` to `Dem0W0rdPressPassW0rD`  
Click `Create parameter`

##### Create Parameter - EFSFSID (The ID of the EFS that using for mount into the EC2 instance to storing Wordpress content)

Click `Create Parameter`
Set `Name` enter `/Demo/Wordpress/EFSFSID`
Set `Description` enter `File System ID for Wordpress Content (wp-content)`  
Set `Tier` set `Standard`  
Set `Type` set `String`  
Set `Data Type` set `text`  
Set `Value` set the file system ID `fs-XXXXXXX` which you just noted down (use your own file system ID)  
Click `Create Parameter`

##### Create Parameter - ALBDNSNAME (The DNS Endpoint of the Application Load Balancer)

Click `Create Parameter`
Set `Name` enter `/Demo/Wordpress/ALBDNSNAME` 
Set `Description` enter `DNS Name of the Application Load Balancer for wordpress`  
Set `Tier` set `Standard`  
Set `Type` set `String`  
Set `Data Type` set `text`  
Set `Value` set the DNS name of the load balancer you copied into your clipboard
Click `Create Parameter`

---

### Create Launch Template for ASG

Now we need to prepare the LT for the ASG to automatically scaling our wordpress instance

Move to the EC2 console <https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Home>:

Click `Launch Templates` under `Instances` on the left menu  
Click `Create Launch Template`  
Under `Launch Template Name` enter `Wordpress`  
Under `Template version description` enter `App only, uses EFS for storing media contents, uses ALB endpoint for WP home, uses RDS for database`  
Check the `Provide guidance to help me set up a template that I can use with EC2 Auto Scaling` box

Under `Application and OS Images (Amazon Machine Image)` click `Quick Start`  
Click `Amazon Linux`  
in the `Amazon Machine Image` dropdown, locate `Amazon Linux 2023 AMI` and set the Architecture to `64-bit (x86)`  
Under `Instance Type` select whichever instance is free tier eligible from either `t3.micro` and `t2.micro`  
Under `Key pair (login)` select `Don't include in launch template`  
Under `networking Settings` `select existing security group` and choose `DemoVPC-SGWordpress` Leave storage volumes unchanged  
Leave Resource Tags Unchanged  
Expand `Advanced Details` Under `IAM instance profile` select `DemoVPC-WordpressInstanceProfile` there will be some random at the end, thats ok!  
Under `Credit specification` select `Standard`

#### Add Userdata

At this point we need to add the configuration which will build the instance Enter the user data below into the `User Data` box

```
#!/bin/bash -xe

EFSFSID=$(aws ssm get-parameters --region us-east-1 --names /Demo/Wordpress/EFSFSID --query Parameters[0].Value)
EFSFSID=`echo $EFSFSID | sed -e 's/^"//' -e 's/"$//'`

ALBDNSNAME=$(aws ssm get-parameters --region us-east-1 --names /Demo/Wordpress/ALBDNSNAME --query Parameters[0].Value)
ALBDNSNAME=`echo $ALBDNSNAME | sed -e 's/^"//' -e 's/"$//'`

DBPassword=$(aws ssm get-parameters --region us-east-1 --names /Demo/Wordpress/DBPassword --with-decryption --query Parameters[0].Value)
DBPassword=`echo $DBPassword | sed -e 's/^"//' -e 's/"$//'`

DBRootPassword=$(aws ssm get-parameters --region us-east-1 --names /Demo/Wordpress/DBRootPassword --with-decryption --query Parameters[0].Value)
DBRootPassword=`echo $DBRootPassword | sed -e 's/^"//' -e 's/"$//'`

DBUser=$(aws ssm get-parameters --region us-east-1 --names /Demo/Wordpress/DBUser --query Parameters[0].Value)
DBUser=`echo $DBUser | sed -e 's/^"//' -e 's/"$//'`

DBName=$(aws ssm get-parameters --region us-east-1 --names /Demo/Wordpress/DBName --query Parameters[0].Value)
DBName=`echo $DBName | sed -e 's/^"//' -e 's/"$//'`

DBEndpoint=$(aws ssm get-parameters --region us-east-1 --names /Demo/Wordpress/DBEndpoint --query Parameters[0].Value)
DBEndpoint=`echo $DBEndpoint | sed -e 's/^"//' -e 's/"$//'`

dnf -y update

dnf install wget php-mysqlnd httpd php-fpm php-mysqli mariadb105-server php-json php php-devel stress amazon-efs-utils -y

systemctl enable httpd
systemctl start httpd

mkdir -p /var/www/html/wp-content
chown -R ec2-user:apache /var/www/
echo -e "$EFSFSID:/ /var/www/html/wp-content efs _netdev,tls,iam 0 0" >> /etc/fstab
mount -a -t efs defaults

wget http://wordpress.org/latest.tar.gz -P /var/www/html
cd /var/www/html
tar -zxvf latest.tar.gz
cp -rvf wordpress/* .
rm -R wordpress
rm latest.tar.gz

sudo cp ./wp-config-sample.php ./wp-config.php
sed -i "s/'database_name_here'/'$DBName'/g" wp-config.php
sed -i "s/'username_here'/'$DBUser'/g" wp-config.php
sed -i "s/'password_here'/'$DBPassword'/g" wp-config.php
sed -i "s/'localhost'/'$DBEndpoint'/g" wp-config.php

usermod -a -G apache ec2-user   
chown -R ec2-user:apache /var/www
chmod 2775 /var/www
find /var/www -type d -exec chmod 2775 {} \;
find /var/www -type f -exec chmod 0664 {} \;

cat >> /home/ec2-user/update_wp_ip.sh<< 'EOF'
#!/bin/bash
source <(php -r 'require("/var/www/html/wp-config.php"); echo("DB_NAME=".DB_NAME."; DB_USER=".DB_USER."; DB_PASSWORD=".DB_PASSWORD."; DB_HOST=".DB_HOST); ')
SQL_COMMAND="mysql -u $DB_USER -h $DB_HOST -p$DB_PASSWORD $DB_NAME -e"
OLD_URL=$(mysql -u $DB_USER -h $DB_HOST -p$DB_PASSWORD $DB_NAME -e 'select option_value from wp_options where option_id = 1;' | grep http)

ALBDNSNAME=$(aws ssm get-parameters --region us-east-1 --names /Demo/Wordpress/ALBDNSNAME --query Parameters[0].Value)
ALBDNSNAME=`echo $ALBDNSNAME | sed -e 's/^"//' -e 's/"$//'`

$SQL_COMMAND "UPDATE wp_options SET option_value = replace(option_value, '$OLD_URL', 'http://$ALBDNSNAME') WHERE option_name = 'home' OR option_name = 'siteurl';"
$SQL_COMMAND "UPDATE wp_posts SET guid = replace(guid, '$OLD_URL','http://$ALBDNSNAME');"
$SQL_COMMAND "UPDATE wp_posts SET post_content = replace(post_content, '$OLD_URL', 'http://$ALBDNSNAME');"
$SQL_COMMAND "UPDATE wp_postmeta SET meta_value = replace(meta_value,'$OLD_URL','http://$ALBDNSNAME');"
EOF

chmod 755 /home/ec2-user/update_wp_ip.sh
echo "/home/ec2-user/update_wp_ip.sh" >> /etc/rc.local
/home/ec2-user/update_wp_ip.sh
```

Ensure to leave a blank line at the end  
Click `Create Launch Template`  
Click `View launch templates`

---
### Create ASG

Now we're starting to initialize the ASG for dynamically scaling
##### Create an auto scaling group (no scaling yet)

Move to the EC2 console: https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Home

Under `Auto Scaling`  
Click `Auto Scaling Groups`  
Click `Create an Auto Scaling Group`  
For `Auto Scaling group name` enter `DemoWordPressASG`  
Under `Launch Template` select `Wordpress`  
Under `Version` select `Latest`  
Scroll down and click `Next`  
For `Network` `VPC` select `DemoVPC`  
For `Subnets` select `sn-Pub-A`, `sn-pub-B` and `sn-pub-C`  
Click `next`

##### Integrate ASG and ALB

Its here where we integrate the ASG with the Load Balancer. Load balancers actually work (for EC2) with static instance registrations. What ASG does, it link with a target group, any instances provisioned by the ASG are added to the target group, anything terminated is removed.

Check the `Attach to an existing Load balancer` box  
Ensure `Choose from your load balancer target groups` is selected.  
for `existing load balancer targer groups` select `DemoWordPressALBTG`  
don't make any changes to `VPC Lattice integration options`

Under `Health Checks` check `Turn on Elastic Load Balancing health checks`  
Under `Additional Settings` check `Enable group metrics collection within CloudWatch`

Scroll down and click `Next`

For now leave `Desired` `Mininum` and `Maximum` at `1`  
For `Automatic scaling - optional` leave it on `No scaling policies`  
For `Instance maintenance policy` leave it on `No policy`
Make sure `Enable instance scale-in protection` is **NOT** checked  
Click `Next`  
We wont be adding notifications so click `Next` Again  
Click `Add Tag`  
for `Key` enter `Name` and for `Value` enter `Wordpress-ASG` make sure `Tag New instances` is checked Click `Next` Click `Create Auto Scaling Group`

Right click on instances and open in a new tab, you should see a new instance being created... `Wordpress-ASG` this is the one created automatically by the ASG using the launch template - this is because the desired capacity is set to `1` and we currently have `0`

---
### Add scaling policy

Move to the AWS Console <https://console.aws.amazon.com/ec2/autoscaling/home?region=us-east-1#AutoScalingGroups>:  
Click `Auto Scaling Groups`  
Click the `DemoWordPressASG` ASG  
Click the `Automatic SCaling` Tab

We're going to add two policies, scale in and scale out.

##### SCALEOUT when CPU usage on average is above 40%

Click `Create dynamic Scaling Policy`  
For policy `type` select `Simple scaling`  
for `Scaling Policy name` enter `HIGHCPU`  
Click `Create a CloudWatch Alarm`  
Click `Select Metric`  
Click `EC2`  
Click `By Auto Scaling Group` (if this doesn't show in your list, wait a while and refresh)  
Check `DemoWordPressASG CPU Utilization` (if this doesn't show in your list, wait a while and refresh)  
Click `Select Metric`  
Scroll Down... select `Threashhold style` `static`, select `Greater` and enter `40` in the `than` box and click `Next` Click `Remove` next to notification if you see anything listed here Click `Next` Enter `WordpressHIGHCPU` in `Alarm Name`  
Click `Next`  
Click `Create Alarm`  
Go back to the AutoScalingGroup tab and click the `Refresh SYmbol` next to Cloudwatch Alarm  
Click the dropdown and select `WordpressHIGHCPU`  
For `Take the action` choose `Add` `1` Capacity units  
Click `Create`

##### SCALEIN when CPU usage on average is below 40%

Click `Create Dynamic Scaling Policy`  
For policy `type` select `Simple scaling`  
for `Scaling Policy name` enter `LOWCPU`  
Click `Create a CloudWatch Alarm`  
Click `Select Metric`  
Click `EC2`  
Click `By Auto Scaling Group` Check `DemoWordPressASG CPU Utilization`  
Click `Select Metric`  
Scroll Down... select `Static`, `Lower` and enter `40` in the `than` box and click `next` Click `Remove` next to notification Click `Next` Enter `WordpressLOWCPU` in `Alarm Name`  
Click `Next`  
Click `Create Alarm`  
Go back to the AutoScalingGroup tab and click the `Refresh Symbol` next to Cloudwatch Alarm  
Click the dropdown and select `WordpressLOWCPU`  
For `Take the action` choose `Remove` `1` Capacity units  
Click `Create`

##### ADJUST ASG Values

Click `Details Tab`  
Under `Group Details` click `Edit`  
Set `Desired 1`, Minimum `1` and Maximum `3`  
Click `Update`

##### Test Scaling & Self Healing

Open Auto Scaling Groups in a new tab <https://console.aws.amazon.com/ec2autoscaling/home?region=us-east-1#/details>  
Open that Auto scaling group in that tab and click on `Activity` tab Go to running instances in the EC2 Console <https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances:sort=desc:tag:Name> in a new tab

Simulate some load on the wordpress instance

Select the/one running `Wordpress-ASG` instance, right click, `Connect`, Select `Session Manager` and click `Connect`  
type `sudo bash` and press enter  
type `cd` and press enter  
type `clear` and press enter

run `stress -c 2 -v -t 3000`

this stresses the CPU on the instance, while running go to the ASG tag, and refresh the activities tab. it might take a few minutes, but the ASG will detect high CPU load and begin provisioning a new EC2 instance. if you want to see the monitoring stats, change to the monitoring tag on the ASG Console Check `enable` next to Auto Scaling Group Metrics Collection

At some point another instance will be added. This will be auto built based on the launch template, connect to the RDS instance and EFS file system and add another instance of capacity to the platform. Try terminating one of the EC2 instances ... Watch what happens in the activity tab of the auto scaling group console. This is an example of self-healing, a new instance is provisioned to take the old ones place.

