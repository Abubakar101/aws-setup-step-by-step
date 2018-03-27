## Launching an Instance in an IAM role

### EC2 in an IAM Role

  * S3, SNS, SQS, can be interacted with directly
  * You can also use the AWS SDK
  * This project used the PHP SDK

### Providing Credentials to the SDK

  * Passing credentials into a client factory method
  * Setting credentials after installation
  * Using temporary credentials from AWS STS
  * Using credentials from environment variables
  * Using IAM roles for EC2 instances (Amazon recommends)

### Launching an Instance in IAM Role

  * Create IAM role
  * Define which accounts or services can assume the role
  * Define which API actions and resources the app can use
  * Specify the role when you launch your instances

  - Go to IAM
  - Choose Roles
  - Create new role
  - Select a Role type
    * Amazon EC2
    * PowerUserAccess
  - Role Name: webserver

  Back to the Dashboard

  - EC2
  - AMIs
  - right click on initial-dev-image: launch
  - stick to the default
  - 3. Configure Instance
    IAM role: webserver
  - Tag Instance
    Name: webserver-in-role
    role: webserver
    env: dev
  - 6. Configure Security Group -> Select an existing security group -> web-tier
  - Review and launch
  - View instances and to verify the role copy the public IP
  - go to the terminal and log in with the new IP

  ssh -i ~/Desktop/Exercises/dev-05678017.pem 54.191.78.0 -l ec2-user

  Once logged in to verify the role

  curl http://169.254.169.254/latest/meta-data/iam/security-credentials/webserver

## Installing the SDK

  * Installing PHP SDK via Composer
  * Phar
  * PEAR
  * ZIP download

  We are going to use Composer

  cd /var/www/html

  - In order to use Composer, is to create a composer.json file. It instructs Composer on what dependencies to install.

  touch composer.json

  vim composer.json

  ```json
  {
    "require": {
      "aws/aws-sdk-php":"2.*"
    }
  }
  ```

  curl -sS https://getcomposer.org/installer | php

  php composer.phar install

  - Create a bucket using PHP SDK on the terminal

  touch createbucket.php

  vim createbucket.php

  ```PHP
  <?php
  require 'vendor/autoload.php';
  use Aws\S3\S3Client;
  try {
    $s3Client = S3Client::factory();
    $s3Client -> createBucket(array(
      'Bucket'=>'aws-essential-training-2014-test'
    ));
  }
  catch(\Aws\S3\Exception\S3Excpetion $e) {
    echo $e->getMessage();
  }
  echo "All Done";
  ```
