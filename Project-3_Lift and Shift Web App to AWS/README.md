﻿# Project-3:Lift and Shift Web App to AWS

[*Project Source*](https://www.udemy.com/course/devopsprojects/?src=sac&kw=devops+projects)

## Prerequisites:
 * AWS Account
 * Registered DNS Name
 * Maven
 * JDK8
 * AWS CLI

### Architecture of Web App:
<div align="center">
   <div align="center">
    <img src="images/Project3_Multi Tier Web Application Stack Setup Locally.jpg" width='700'/>
  </div>
</div>
</br>

### Architecture on AWS:
<div align="center">
   <div align="center">
    <img src="images/Project3_AWS.jpg" width='700'/>
  </div>
</div>
</br>

### Step-1: Create Security Groups for Services

<div align="center">
   <div align="center">
    <img src="images/SecurityGroups.jpg" width='700'/>
  </div>
</div>
</br>


- We will create `App_ELB_SG` first. We will configure `Inbound` rules to Allow both `HTTP` and `HTTPS` on port `80` and `443` respectively  from Anywhere `IPv4` and `IPv6`.
- Next we will create `App_Server_SG`. We will open port `8080` to accept connections from `APP_ELb_SG`.
- Finally, we will create `Backend_Services_SG`. WE need to open port `3306` for `MySQL`, `11211` for `Memcached` and `5672` for `RabbitMQ` server. We can check whcih ports needed fro aplication services to communicate each other from `application.properties` file under `src/main/resources` directory.We also need to open commucation `AllTraffic` from own SecGrp for backend services to communicate with each other.

### Step-2: Create KeyPair to Connect EC2 instances

- We will create a Keypair to connect our instances via SSH. We will use this keypair to connect our instances via SSH. 

### Step-3: Provision Backend EC2 instances with UserData script

##### DB Instance:

- Create DB instance with below details.We will also add Inbound rule to `Backend_Services_SG` for `SSH` on port `22`  from `MyIP` to be able to connect our db instance via SSH.
```sh
Name: app-db01
Project: app
AMI: Centos 7
InstanceType: t2.micro
SecGrp: vprofile-backend-SG
UserData: mysql.sh
```
- Once our instance is ready, we can SSH into the server and check if userdata script is executed.We can also check status of mariadb.
```sh
ssh -i app-prod-key.pem centos@<public_ip_of_instance>
sudo su -
curl http://169.254.169.254/latest/user-data
systemctl status mariadb
```

##### Memcached Instance:

- Create Memcached instance with below details.
```sh
Name: app-mc01
Project: app
AMI: Centos 7
InstanceType: t2.micro
SecGrp: Backend_Services_SG
UserData: memcache.sh
```
- Once our instance is ready, we can SSH into the server and check if userdata script is executed.We can also check status of memcache service and if it is listening on port 11211.
```sh
ssh -i app-prod-key.pem centos@<public_ip_of_instance>
sudo su -
curl http://169.254.169.254/latest/user-data
systemctl status memcached.service
ss -tunpl | grep 11211
```

##### RabbitMQ Instance:

- Create RabbitMQ instance with below details.
```sh
Name: app-rmq01
Project: app
AMI: Centos 7
InstanceType: t2.micro
SecGrp: Backend_Services_SG
UserData: rabbitmq.sh
```
- Once our instance is ready, we can SSH into the server and check if userdata script is executed.We can also check status of rabbitmq service.
```sh
ssh -i app-prod-key.pem centos@<public_ip_of_instance>
sudo su -
curl http://169.254.169.254/latest/user-data
systemctl status rabbitmq-server
```

_Note: It may take some time to run userdata script after you connect to server. You can check the process `ps -ef` to see if the process start for service. If not wait sometime and check with `systemctl status <service_name>` command again._

### Step-3: Create Private Hosted Zone in Route53 

- Our backend stack is running. Next we will update Private IP of our backend services in Route53 Private DNS Zone.Lets note down Private IP addresses.
```sh
rmq01 172.31.80.20
db01 172.31.22.178
mc01 172.31.87.132
```
- Create `app_web.in` Private Hosted zone in Route53. we will pick `Default VPC` in `N.Virginia` region.
- Now we will create records for our backend services. The purpose of this activity is we will use these record names in our `application.properties` file. Even if IP address of the services, our application won't need to change the config file.  
```sh
Simple Routing -> Define Simple Record
Value/Route traffic to: IP address or another value
```
<div align="center">
   <div align="center">
    <img src="images/route53.jpg" width='700'/>
  </div>
</div>
</br>

### Step-4: Provision Application EC2 instances with UserData script

- Create Tomcat instance with below details.We will also add Inbound rule to `App_Server_SG` for `SSH` on port `22`  from `MyIP` to be able to connect our db instance via SSH.
```sh
Name: app01
Project: app
AMI: Ubuntu 18.04
InstanceType: t2.micro
SecGrp: App_Server_SG
UserData: tomcat_ubuntu.sh
```

### Step-5: Create Artifact Locally with MAVEN

- Clone the repository.
```sh
git clone https://github.com/rumeysakdogan/vprofile-project.git
```

- Before we create our artifact, we need to do changes to our `application.properties` file under `/src/main/resources` directory for below lines.
```sh
jdbc.url=jdbc:mysql://db01.vprofile.in:3306/accounts?useUnicode=true&

memcached.active.host=mc01.vprofile.in

rabbitmq.address=rmq01.vprofile.in
```
- We will go to `vprofile-project` root directory to the same level pom.xml exists. Then we will execute below command to create our artifact `vprofile-v2.war`:

```sh
mvn install
```

### Step-6: Create S3 bucket using AWS CLI, copy artifact

- We will upload our artifact to s3 bucket from AWS CLI and our Tomcat server will get the same artifact from s3 bucket.

- We will create an IAM user for authentication to be used from AWS CLI.
```sh
name: app-s3-admin
Access key - Programmatic access
Policy: s3FullAccess
```
<div align="center">
   <div align="center">
    <img src="images/s3_user.jpg" width='700'/>
  </div>
</div>
</br>

- Next we will configure our `aws cli` to use iam user credentials.
```sh
aws configure
AccessKeyID: 
SecretAccessKey:
region: us-east-1
format: json
```
- Create bucket. Note: S3 buckets are global so the naming must be UNIQUE!
```sh
aws s3 mb s3://app-artifact-storage-rd 
```
- Go to target directory and copy the artifact to bucket with below command. Then verify by listing objects in the bucket.
```sh
aws s3 cp vprofile-v2.war s3://app-artifact-storage-rd
aws s3 ls app-artifact-storage-rd
```
- We can verify the same from AWS Console.
<div align="center">
   <div align="center">
    <img src="images/artifact.jpg" width='700'/>
  </div>
</div>
</br>

### Step-7: Download Artifact to Tomcat server from S3

- In order to download our artifact onto Tomcat server, we need to create IAM role for Tomcat. Once role is created we will attach it to our `app01` server.
```sh
Type: EC2
Name: app-artifact-storage-role
Policy: s3FullAccess
```
- Before we login to our server, we need to add SSH access on port 22 to our `App_Server_SG`.

- Then connect to `app011` Ubuntu server.
```sh
ssh -i "app-prod-key.pem" ubuntu@<public_ip_of_server>
sudo su -
systemctl status tomcat8
```

- We will delete `ROOT` (where default tomcat app files stored) directory under `/var/lib/tomcat8/webapps/`. Before deleting it we need to stop Tomcat server. 
```sh
cd /var/lib/tomcat8/webapps/
systemctl stop tomcat8
rm -rf ROOT
```
- Next we will download our artifact from s3 using aws cli commands. First we need to install `aws cli`. We will initially download our artifact to `/tmp` directory, then we will copy it under `/var/lib/tomcat8/webapps/` directory as `ROOT.war`. Since this is the default app directory, Tomcat will extract the compressed file.
```sh
apt install awscli -y
aws s3 ls s3://app-artifact-storage-rd
aws s3 cp s3://app-artifact-storage-rd/vprofile-v2.war /tmp/vprofile-v2.war
cd /tmp
cp vprofile-v2.war /var/lib/tomcat8/webapps/ROOT.war
systemctl start tomcat8
```

- We can also verify `application.properties` file has the latest changes.
```sh
cat /var/lib/tomcat8/webapps/ROOT/WEB-INF/classes/application.properties
```

- We can validate network connectivity from server using `telnet`.
```sh
apt install telnet
telnet db01.vprofile.in 3306
```

### Step-8: Setup LoadBalancer

- Before creating LoadBalancer , first we need to create Target Group.
```sh
Intances
Target Grp Name: APP-ELb-TG
protocol-port: HTTP:8080
healtcheck path : /login
Advanced health check settings
Override: 8080
Healthy threshold: 3
available instance: app01 (Include as pending below)
```
<div align="center">
   <div align="center">
    <img src="images/target_group.jpg" width='700'/>
  </div>
</div>
</br>

- Now we will create our Load Balancer.
```sh
app-prod-elb
Internet Facing
Select all AZs
SecGrp: APP_ELb_SG
Listeners: HTTP, HTTPS
Select the certificate for HTTPS
```

### Step-9: Create Route53 record for ELB endpoint

- We will create an A record with alias to ALB so that we can use our domain name to reach our application.

- Lets check our application using our DNS. We can securely connect to our application!
<div align="center">
   <div align="center">
    <img src="images/web_app.jpg" width='700'/>
  </div>
</div>
</br>

### Step-10: Configure AutoScaling Group for Application Instances

- We will create an AMI from our App Instance.

![](images/vprofile-ami.png)

- Next we will create a Launch template using the AMI created in above step for our ASG.
```sh
Name: app-LT
AMI: app-image
InstanceType: t2.micro
IAM Profile: app-artifact-storage-role
SecGrp: App_Server_SG
KeyPair: app-prod-key
```

- Our Launch template is ready, now we can create our ASG.
```sh
Name: app-ASG
ELB healthcheck
Add ELB
Min:1
Desired:2
Max:4
Target Tracking-CPU Utilization 50
```
<div align="center">
   <div align="center">
    <img src="images/autoscaling.jpg" width='700'/>
  </div>
</div>
</br>

- If we terminate any instances we will see ASG will create a new one using LT that we created.

### Step-11: Clean-up

- Delete all resources we created to avoid any charges from AWS.


## Connect with Me 🤝🏻 &nbsp;

<h3 align="center">
<a href="https://linkedin.com/in/dayo-sasanya"><img src="https://img.shields.io/badge/-Dayo%20Sasanya-0077B5?style=for-the-badge&logo=Linkedin&logoColor=white"/></a>


