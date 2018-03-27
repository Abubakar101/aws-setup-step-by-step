## Process involving managing AWS EC2

1. Signing into AWS account
2. Creating a group for permissions
   - As good practices: create a group with certain privileges/roles and assign users. Avoid access the account as the account owner. The account owner has access to every single resource including all the billing, including credit card. This information needs to be protected.
3. Create an IAM user, assign to the group role, give it a password and create an access alias to the console.
  - User: Bruno
  - Password: password
  - alias: awsbrunoisadmin
  - https://awsbrunoisadmin.signin.aws.amazon.com/console/
4. Use a public key cryptography to encrypt and decrypt login information. Before create an EC2 instance create a key par. Go to EC2 and on the left, use key pair. Try to rotate it every so often. You might create for a development as well as production environment. Also come with a convention for these that works for you and your organization.
  - Once it's created, it also downloads the file to your system. You don't want to lose this file.
  - Since you'll be using SSH to login to the instances, it needs to ensure proper permissions are set on this PIM file.
  - chmod 0400 ~/Desktop/Exercises/dev-05678017.pem
    That ensures that's only available for reading to the user.
5. Create Security Group

Public -- HTTP 80 --> <ELB> -- HTTP 80 --> [ Web Server ] -- MySQL/TCP 3306 --> (RDS)

  - We have public traffic coming in to our load balancer, so we want to allow for that. Then, we have the load balancer sending traffic to our web instances, so we need to allow for this, but we should restrict this access to only traffic from the load balancer. Then our web servers need to be able to connect to the database server, but again, we only want to allow the connection from our web servers.
  - 2 security groups to handle this:

### Security Group

  - Load Balancer
    * Allow HTTP on port 80 for everyone
  - Web tier
    * Allow HTTP on port 80 only from ELB
    * Allow MySQL/TCP on port 3306 only from web servers
  - The security groups can be set up at EC2, left menu EC2 and create a new security group:
    * Security group name: load-balancer
    * Description: allow http traffic on 80 in from public
    * VPC
    * Add inbound rule: type: http; Protocol: TCP; Port: 80; Source: Anywhere;

The next thing is to create a new security group for our web tier. It has been mentioned, what we want to do for the web tier, is we want to restrict http traffic on port 80 in from only the load balancer.

The way that we do that is we're going to specify the security group ID in the source when we set up that rule. (Get the group ID before start creating a new security group. it's located on the bottom of the page.):
    * Security group name: web-tier
    * Description: allow http 80 from elb
    * VPC
    * Add inbound rule: type: http; Protocol: TCP; Port: 80; Source: Custom: (add Load Balancer Group ID)
    * Once it's created, before adding a new rule for MySQL, get the Web-Tier Group ID, and at inbound, select edit and add new rule:
    * type: MySQL; Protocol: TCP; Port: 3306; Source: Custom: (add Web Tier Group ID);

6. Creating an ELB
  - Choose between new load balancer (recommended) or classic. the new load balancer is improved.
  - name: web-tier-load-balancer
  - Choose between internet-facing load balance or internal. In the case of a VPC it can be made public which means it will be given a DNS name using the public IP address o r it can be made private which is what internal refers to here. It needs to be public to receive public traffic.
  - If you have a certificate, you can also choose to connect via HTTP and HTTPS as well.
  - Choose an existing security group:
    * load-balancer
  - Review
  - Create

7. Launching an EC2
  - Choose EC2
  - Launch EC2
  - Choose an AMI (Amazon Machine Image)
    * Amazon Linux
  - Choose an instance type (the one which is best fit for your application and organization) - The rule of thumb here is to start small as possible and then increase as you find limitations along the way.
  - Configure Instance Details
    Configuring Apache and PHP (LAMP) with user data:
    * At Advance Details you can use User Data to bootstrap your instances by using Cloud-init to set up the initial configuration needed for a LAMP stack web server. This way, when the box spins up, basically when this instance spins up, it will already be configured as a web application:

    #! /bin/bash -ex
    yum update -y
    yum groupinstall -y "Web Server" "MySQL Database" "PHP Support"
    service httpd start
    chkconfig httpd on

    what's going to happen is all of this is going to run as soon as we launch the instance, and in essence, we'll automatically have our web server set up for us, without having to SSH and do all this explicitly on the server.

  - Add additional storage if needed
  - Add tags to instance to better categorize and add metadata to our instances. ex. distinguish between development vs production envs or QA env or the role that this resource is playing at the company.
    Name: web server 1
    role: webserver
    env: dev
  - Configure a Security Group
    Since we already created a security group, we choose the one we have.
  You get a warning stating that port 22 isn't open. and you can come back later and add when and if needed.
  - Review your choices and launch
  - Specify the keypair - dev-05678017 one
  - Launch - View instance

8. Trying to access the EC2 via HTTP
  - if you get the EC2 instant public IP it won't work because the rules for the ELB are set to prohibit public traffic.
  - Go to the Security Group and change the rules for web-tier and test it by having access via HTTP using My IP
  - Try use the public IP again and it should work

9. Trying to access the EC2 via SSH
  - Now you need to set the security group to allow inbound traffic on port 22 via SSH
  - choose type SSH, port 22 and my IP
  * Best practice: instead of use your own IP address to access EC2 via SSH, is to set up a Bastion host, which keeps things secure and only allow SSH in from that specific bastion host, rather than form a specific IP
  - Go to terminal on your computer
  ssh -i ~/Desktop/Exercises/dev-05678017.pem 54.218.11.189 -l ec2-user
  - It will ask for the authenticity of the host
  - Now that you are in, you can create the heartbeat file to determine where the instance is healthy or not.
  - cd /var/www/html/
  To allow EC2 users to manipulate files in this directory, we need to modify the ownership and permissions of this directory. One way of doing this is:
  - Add the www group to my instance and give that group ownership of the var/www directory and add right permissions for that group. So any members of that group will be able to add, delete, and modify files for the web server:

  sudo groupadd www

  - Add EC2 user to this newly created www group:

  sudo usermod -a -G www ec2-user

  - Add that group to have permissions to be able to add and delete files in this directory. for changes to effect you need to log out and log back in.

  exit

  - One you log back in, check whether EC2 user is part of the www group:

  groups

  - Change group ownership of var/www, which is our doc root and its contents, to be the www group.

  sudo chown -R root:www /var/www

  - Change the directory permissions of var/www and its subdirectories to add group right permissions and se the group ID on all future subdirectories. This will make easy to add files.

  sudo chmod 2775 /var/www
  find /var/www -type d -exec sudo chmod 2775 {} +
  find /var/www -type f -exec sudo chmod 0664 {} +

  The command above it will work recursively change the file permissions of var/www and its subdirectories to add group right permissions.

  Now EC2 users and any future members of the www group can add, delete, and edit files in the Apache document root.

  - Ready to add a content: whether a PHP website or a php application.
  - cd /var/www/html
  - add heartbeat.php file

  touch heartbeat.php
  echo "<?php echo 'OK';" > heartbeat.php

  - Go to the browser and after the ip address add /heartbeat.php you should see OK
  We got back 200, and we get back ok, and this is actually would be fine for our load balancer. It would send the indication that it's alive, and it's healthy, and this is basically all we need for the heartbeat. So, it demonstrates also that php is installed and working and basically puts us in a place where we can use our load balancer.

  Add another script that will tell us the instance identifier of the web server we are hitting. At the moment, we only have one web server, but we'll be adding an additional one in order to follow our design for failure best practice and complete the architecture we outlined:

  - Create an instance.php file and use the instance metadata to get the instance identifier.

  touch instance.php
  vim instance.php

  ```php
  <?php
    $this_instance_id = file_get_contents('http://169.254.169.254/latest/meta-data/instance-id');
    if(!empty($this_instance_id))
        echo (string)($this_instance_id);
    else
        echo "Sorry, instance id unknown!";
  ```

  if you run the file on http://ipaddress/instance.php it should return the Instance id

  Now, we have everything in place to add our instance to the load balancer. the heartbeat.php in place, a way to check what instance we are hitting.

  At the console go to load-balancer and look at the instances.

  (This differs from classic load-balancing process)

  To register you EC2 instance on Application Load Balancer, you have to go to the Target Groups and register your existing EC2 instance

  - Register your EC2 Instance at the Target Groups

  Now to test to see if the EC2 is running behind the load balancer, just go at the load balance and get the public address (DNS):
  - Get DNS on the Description of the Load Balancer

  web-tier-load-balancer-577080776.us-west-2.elb.amazonaws.com (A Record)

  - To Check to see if instance is healthy from CLI, you have to go where Apache log files are:

  cd /etc/httpd/

  sudo tail -f logs/access_log

  If you get this:
  172.31.29.30 - - [24/Apr/2017:02:27:07 +0000] "GET /heartb
eat.php HTTP/1.1" 200 2 "-" "ELB-HealthChecker/2.0"

  You know your health checker is working. It's getting what it needs to return healthy. Our traffic is routing through the load balancer. It's hitting our web server, and we are getting our default Apache webpage. So, everything is looking good so far.

10. Creating a MySQL RDS database

  - Create a new RDS console
    Go to service and choose RDS (Relational Data Service) and on the right choose
  - Create Instance
  - choose MySQL (according with business necessities) but for this case:
    * Dev/Test - MySQL (...)
    * DB Engine: mysql  
    * License Model: general-public-license
    * DB Engine Version: MySQL 5.6.27
    * DB Instance Class: db.t2.micro -- 1 vCPU, 1GiB RAM
    * Multi-AZ Deployment: No
    * Storage Type: General Purpose (SSD)
    * Allocated Storage: 5 GB
    * DB Instance Identifier: dev-db
    * Master Username: admin
    * Master Password: password
    * Confirm Password: password
    After you hit next
    * VPC Security Groups: Web-tier (VPC)
    * Database Name: essential_training
    Hit launch

    Once the database is ready and built you can click on details and get the endpoint to test if the database was created using the terminal

    cd /etc/httpd

    mysql -h dev-db.c2fqow8ngppt.us-west-2.rds.amazonaws.com -u admin -p

    Once you log in the MySQL you can test:

    show databases;

    and essential_training should be there.

11. Creating a custom server image
  - Create an AMI so that we can easily recreate these web tier resources.
  - Go to EC2 and then instances
  - Select the instance and with right click choose:
  - Create Image
  - select a name that will be a convention, perhaps including the date that was created:
  initial-dev-image
  - give it a description: initial web server
  - Allow it to reboot when it's created.
  - Done! If you want to check just go to Images - AMIs and you will see AMI being created. It will take a while.
  - Create a new instance from this image but IN A DIFFERENT AVAILABLE ZONE from the previous.
  - Go to AMI, select the instance and with right click, choose launch
  - On step 3, change the Subnet part for a different location from the previous instance.
  - Click next to add storage (you won't add any, click next)
  - Step 5: Add Tags
  - Key: name, value: web server 2
  - Key: role, value: webserver
  - Key: env, value: dev
  - Security group: choose existing security group: web tier
  - review and launch
  - Identify the key pair and acknowledge it
  - Now that it's working add behind the load balancer.
  - Load Balancer -> Target Groups, Register a target (web server 2)
  - Once it's running to test, just go to the load balancer and get the DNS and hit instance

  web-tier-load-balancer-577080776.us-west-2.elb.amazonaws.com/instance.php

  if you refresh you will be toggling back and forth the two instances

12. Auto Scaling
  * Launch configuration - what to scale
  * Auto Scaling Group - how to scale
  - EC2 -> Launch Configurations
  - Create Auto Scaling Group
  - Create Launch Configuration
  Because we're defining the what we want to scale, we're basically defining the EC2 instance properties in details to have it launch when it goes to a launch cycle either scaling out or scaling in.
  Choose AMI we've already created. The initial-dev-image:
  - My AMI
  - Choose Instance Type
  - Configure details
    name: web-server-launch-config
  - Configure Security Group
    Select an existing security group -> web-tier
  - review
  - Create launch configuration
  - Choose the key pair and acknowledge it

  Since this is our first launch configuration, it takes us right into creating an auto scaling group. Think of this as HOW to do our scaling.

  Typically, we use auto scaling to either scale out, add more servers or scale in, remove servers based on some event happening. If our server becomes unhealthy, this event would be used to scale out and add more capacity.

  We could set up an event such that if our servers perhaps drop below certain usage, indicators say like low CPU or low memory, could reduce the number of servers, therefore reducing the bill.

  - Give the group a name: web-dev-scaling
  - Group size: Start with 2 instances (it will always have AT LEAST 2 instances)
  - Launch default VPC
  - Subnet: Add both of the instances
  Advanced details
  - check receive traffic from ELB
  - Choose target groups
  - Health Check Type: ELB
  - Keep this group at its initial size
  - If you want you can add Notification
  - Configure tags:
    Name: web-server-auto-scaled-instances
    role: webserver
    env: dev
