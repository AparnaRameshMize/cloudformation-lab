aws> cloudformation create-stack --stack-name ec2-iam-stack --template-body file://ec2-iam.yml --capabilities CAPABILITY_IAM

___

## LAMP Stack for Wordpress

### Lab - 01. Basic Infrastructure Creation 
Some initial considerations
- Default VPC to keep our initial template simple
- S3 Bucket
- AWS RDS - MySQL (uses Default Security Group)
- EC2 Instance (No SSH access for now)
- All (3) Independent resources, no dependencies between the resources
- Logical & Physical IDs
    - Logical Id - name as mentioned in the template
    - Physical Id - the EC2 instance id for example in case of an EC2 Instance


Create the stack and verify the created resources
```bash
aws> cloudformation create-stack --stack-name wp-01 --template-body file://wordpress-01.json
```
What happens when we try to use the same template to create another stack, with a different name?
```bash
aws> cloudformation create-stack --stack-name wp-02 --template-body file://wordpress-01.json
```
What hapens when we try to use the same temlate to create a stack in a different region (not us-east-1)?
```bash
aws> cloudformation create-stack --stack-name wp-01 --template-body file://wordpress-01.json --region us-east-2
```
Cleanup
```bash
aws> cloudformation delete-stack --stack-name wp-01
aws> cloudformation delete-stack --stack-name wp-02
aws> cloudformation delete-stack --stack-name wp-01 --region us-east-2
```
Drawbacks
- Hardcoding - for example the instance sizing
- Works only once
- Works in *us-east-1* only

___
### Lab - 02. Making CF Template flexible & portable

Let us consider creating Wordpress in three distinct sizes: S (Small), M (Medium) & L (Large)

One way to achieve this is to use *Parameters* section of CloudFormation template
```JSON
"Parameters": {
    "EnvironmentSize": {
        "Type": "String",
        "Default": "t2.micro",
        "AllowedValues": [
            "t2.micro",
            "t2.small",
            "t2.medium"
        ],
        "Description": "Select Environment Size (S,M,L)"
    }
}
```

The Instance Sizing is not the same for EC2 & RDS, so environment sizing should be two separate parameters, one for EC2 sizing (            "t2.micro", "t2.small", "t2.medium") and one for RDS sizing ("db.t2.micro", "db.t2.small", "db.t2.medium"). This is not optimal!

How about defining a value called "EnvironmentSize" (SMALL, MEDIUM, LARGE that will translate into corresponding EC2 Instance & RDS sizes!. *Mappings* to rescue!

```JSON
"InstanceSize": {
    "SMALL": {
        "EC2": "t2.micro",
        "DB": "db.t2.micro"
    },
    "MEDIUM": {
        "EC2": "t2.small",
        "DB": "db.t2.small"
    },
    "LARGE": {
        "EC2": "t2.medium",
        "DB": "db.t2.medium"
    }
}
```
Now, if we couple this *Mapping* with a  EnvironmentSize *Parameter*, then the temaplte becomes more dynamic
```JSON
"Parameters": {
    "EnvironmentSize": {
        "Type": "String",
        "Default": "SMALL",
        "AllowedValues": [
            "SMALL",
            "MEDIUM",
            "LARGE"
        ],
        "Description": "Select Environment Size (S,M,L)"
    }
},

...

"DBInstanceClass": {
    "Fn::FindInMap": [
        "InstanceSize",
        {
            "Ref": "EnvironmentSize"
        },
        "DB"
    ]
},
```
To make the S3 bucket name dynamic is easy. Just do not specify the name of the bucket. An S3 bucket name is automatically derived based on the *Stack Name*, *Logical Name* and some *Random characters* (to make the Bucket name unique in the S3 global namespace).
```JSON
"S3": {
    "Type": "AWS::S3::Bucket"
}
```
**In general, relying on the ability of CloudFormation to generate a Resource name based on the Stack Name & Logical Name increases the portability of the template**

Similarly, flexibility with specifying AMD ID based on a Region can be achieved using *Mappings* and *Psuedo Paramaters*

Let us test the Stack creation with the template and clean up after verification
```bash
aws> cloudformation create-stack --stack-name wp-02 --template-body file://wordpress-02.json \
--parameters ParameterKey="EnvironmentSize",ParameterValue="SMALL"
aws> cloudformation delete-stack --stack-name wp-02
```
___

### Lab - 03. Installing Wordpress on EC2 (Userdata)
EC2 instance creation can be bootstrapped with Userdata. the following need to be installed and configured
- PHP
- MySQL
- Apache
- Wordpress

The following shell script will install the necessary components
```bash
#!/bin/bash
yum install httpd php mysql php-mysql -y
yum update -y
chkconfig httpd on
service httpd start
cd /var/www/html
wget https://wordpress.org/latest.tar.gz
tar -zxvf latest.tar.gz --strip 1
rm latest.tar.gz
cp wp-config-sample.php wp-config.php
sed -i 's/database_name_here/${DatabaseName}/g' wp-config.php
sed -i 's/localhost/${DB.Endpoint.Address}/g' wp-config.php
sed -i 's/username_here/${DatabaseUser}/g' wp-config.php
sed -i 's/password_here/${DatabasePassword}/g' wp-config.php
```
Let us test the Stack creation with the template and clean up after verification
```bash
aws> cloudformation create-stack --stack-name wp-03 --template-body file://wordpress-03.json \
--parameters ParameterKey="EnvironmentSize",ParameterValue="SMALL" \
ParameterKey="DatabaseName",ParameterValue="wordpress" \
ParameterKey="DatabaseUser",ParameterValue="wordpress" \
ParameterKey="DatabasePassword",ParameterValue="w0rdpr355"
```
- Verify that the EC2 Instance is created after the RDS creation is complete - why?
- Navigate to the EC2 instance and change the associated Default Security Group Ingress rule to allow SSH traffic from you machine/laptop [SSH, TCP, 22, Custom, your-ip/32 or 0.0.0.0/0]
- SSH into the EC2 instance and verify the wordpress config files
- Verify the Apache process is up & running using the command: ` $ ps -aux | grep httpd`
- Verify the log `cloud-init-output.log`  under `/var/log` directory and check the bootstapping log entries
- Edit the Security Group one more time to allow HTTP traffic to the EC2 instance [HTTP, TCP, 80, Custom, your-ip/32 or 0.0.0.0/0]
- Visit the wordpress site from you browser using the public ip of the EC2 Instance - http://<public-ip>
- Make sure the Security Group changes are rooledback since this is a default Security Group
```bash
aws> cloudformation delete-stack --stack-name wp-03
```

___

