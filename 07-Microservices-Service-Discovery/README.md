# Microservices Deployment on AWS Fargate & ECS Clusters

# Module - 1: Introduction
## What are we going to learn in this section?
- Step-1: 
- Step-2: 
- Step-3:

## Architecture Diagram about our Buildout

## List of Docker Images used in this section
| Application Name                 | Docker Image Name                          |
| ------------------------------- | --------------------------------------------- |
| User Management Microservice | stacksimplify/usermanagement-microservice:1.0.0 |
| Notifications Microservice | tacksimplify/notifications-microservice:1.0.0 |

## Step-1: Pre-requisite -1: Create AWS RDS Database
- Create a RDS MySQL Database
- Select Enginer
    - Engine Options: MySQL
    - Only enable options eligible for RDS Free Usage Tier: Check this box
- Specify DB Details
    - DB Instance Identifier: microservicesdb
    - Master Username: dbadmin
    - Master Password: **Choose the password as you like and make a note of it**
    - Confirm Password: **Choose the password as you like and make a note of it**
    - Database Options
- Configure Advance Settings
    - Network & Security: leave defaults with default VPC 
    - Database Options
        - Database Name: usermgmt
        - Port: 3306
    - Backup
        - Backup Retention Period: 0 days

- Gather the following details from RDS MySQL database to provide them as environment variables in our ECS Task Definition
```
AWS_RDS_HOSTNAME=<gather db endpoint from Connectivity & security --> Endpoint & port section>
microservicesdb.cxojydmxwly6.us-east-1.rds.amazonaws.com
AWS_RDS_PORT=3306
AWS_RDS_DB_NAME=usermgmt
AWS_RDS_USERNAME=dbadmin
AWS_RDS_PASSWORD=*****
```

## Step-2: Pre-requisite-2: Create Simple Email Service - SES SMTP Credentials
### SMTP Credentials
- SMTP Settings --> Create My SMTP Credentials
- IAM User Name: append the default generated name with microservice or something so we have a reference of this IAM user created for our ECS Microservice deployment
- Download the credentials and update the same for below environment variables which you are going to provide in container definition section of Task Definition. 
```
AWS_MAIL_SERVER_HOST=email-smtp.us-east-1.amazonaws.com
AWS_MAIL_SERVER_USERNAME=
AWS_MAIL_SERVER_PASSWORD=
AWS_MAIL_SERVER_FROM_ADDRESS= can-be-anything@gmail.com
```

### Verfiy Email Addresses to which notifications we need to send. 
-  Email Addresses
    - Verify a New Email Address
    - Email Address Verification Request will be sent to that address, click on link to verify your email. 

## Step-3: Pre-requisite-3: Create Application Load Balancer & DNS Register it in Route53
- Create Application Load Balancer
    - Name: microservices-alb
    - Availability Zones: ecs-vpc:  us-east-1a, us-east-1b
    - Security Group: microservices-sg-alb (Inbound: Port 80)
    - Target Group: temp-tg-micro (Rest all defaults)
- DNS register the LB URL with a custom domain name something like "services.stacksimplify.com" in Route53. 
    - Gather the DNS Name of "microservices-alb"
    - Create a Record set in Hosted Zone
        - Record Set Name: services.stacksimplify.com
        - Alias: Yes
        - Alias Name: microservices-alb url

# Module - 2: Deploy Notification Service
## Notification Microservice - Create Task Definition
- Configure Task Definition
    - Task Definition Name: notification-microservice
    - Task Role: ecsTaskExecutionRole
    - Network Mode: awsvpc
    - Task Execution Role: ecsTaskExecutionRole
    - Task Memory: 2GB
    - Task CPU: 1 vCPU
- Configure Container Definition    
    - Container Name: notification-microservice
    - Image: stacksimplify/notification-microservice:1.0.0
    - Container Port: 8095
    - Environment Variables
        - AWS_MAIL_SERVER_HOST
        - AWS_MAIL_SERVER_USERNAME
        - AWS_MAIL_SERVER_PASSWORD
        - AWS_MAIL_SERVER_FROM_ADDRESS

## Notification Microservice - Create Service
- Configure Service
    - Launch Type: Fargate
    - Task Definition:
        - Family: notification-microservice
        - Version: 1(latest) 
    - Service Name: svc-notification-microservice
    - Number of Tasks: 1
- Configure Network
    - VPC: ecs-vpc
    - Subnets: us-east-1a, us-east-1b
    - Security Groups: ecs-microservices-inbound 
        - Allow traffic from ALB security group or anywhere
        - Inbound Port 8095 and 8096 (for both notification & usermanagement service)
    - Health Check Grace Period: 147
    - Load Balancing
        - Load Balancer Type: Application Load Balancer
        - Load Balancer Name: microservices-alb        
        - Container to Load Balance: notification-microservice : 8096  **Click Add to Load Balancer **
        - Target Group Name: tg-notification-microservice
        - Path Pattern: /notification*
        - Evaluation Order: 1
        - Health Check path: /notification/health-status
- **Important Note:** Disable Service Discovery for now

## Notification Microservice - Verify the deployment and access the service. 
- Verify tasks are in **RUNNING** state in Service
- Verify ALB - Target Groups section, Registered Tagets status to confirm if health checks are succeeding. If healthy, you shoud see
    - Status: Healthy
    - Description: This target is currently passing target group's health checks.
- **Troubleshooting Tip**
    - If we get **Request Timed out** as status, cross check ECS security group assigned to **svc-notificaiton-microservice** whether port 8096 is open for inbound traffic. 
- Verify using Load Balancer URL or DNS registered URL
```
http://services.stacksimplify.com/notification/health-status
```


# Module - 3: Deploy User Management Service
## User Management Service - Create Task Definition
- Configure Task Definition
    - Task Definition Name: usermgmt-microservice
    - Task Role: ecsTaskExecutionRole
    - Network Mode: awsvpc
    - Task Execution Role: ecsTaskExecutionRole
    - Task Memory: 2GB
    - Task CPU: 1 vCPU
- Configure Container Definition    
    - Container Name: usermanagement-microservice
    - Image: stacksimplify/usermanagement-microservice:1.0.0
    - Container Port: 8095
    - Environment Variables
        - AWS_RDS_HOSTNAME=microservicesdb.cxojydmxwly6.us-east-1.rds.amazonaws.com
        - AWS_RDS_PORT=3306
        - AWS_RDS_DB_NAME=usermgmt
        - AWS_RDS_USERNAME=dbadmin
        - AWS_RDS_PASSWORD=*****
        - NOTIFICATION_SERVICE_HOST=services.stacksimplify.com [or] ALB DNS Name
        - NOTIFICATION_SERVICE_PORT=80

## User Management Service - Create Service
- Configure Service
    - Launch Type: Fargate
    - Task Definition:
        - Family: usermgmt-microservice
        - Version: 1(latest) 
    - Service Name: svc-usermgmt-microservice
    - Number of Tasks: 1
- Configure Network
    - VPC: ecs-vpc
    - Subnets: us-east-1a, us-east-1b
    - Security Groups: ecs-microservices-inbound (already configured during notification service)
        - Allow traffic from ALB security group or anywhere
        - Inbound Port 8095 and 8096 (for both notification & usermanagement service)
    - Health Check Grace Period: 147
    - Load Balancing
        - Load Balancer Type: Application Load Balancer
        - Load Balancer Name: microservices-alb        
        - Container to Load Balance: usermgmt-microservice : 8095  **Click Add to Load Balancer **
        - Target Group Name: tg-usermgmt-microservice
        - Path Pattern: /usermgmt*
        - Evaluation Order: 1
        - Health Check path: /usermgmt/health-status
- **Important Note:** Disable Service Discovery for now


## User Management Microservice - Verify the deployment and access the service. 
- Verify tasks are in **RUNNING** state in Service
- Verify ALB - Target Groups section, Registered Tagets status to confirm if health checks are succeeding. If healthy, you shoud see
    - Status: Healthy
    - Description: This target is currently passing target group's health checks.
- **Troubleshooting Tip**
    - If we get **Request Timed out** as status, cross check ECS security group assigned to **svc-notificaiton-microservice** whether port 8096 is open for inbound traffic. 
- Verify using Load Balancer URL or DNS registered URL
```
http://services.stacksimplify.com/usermgmt/health-status
```

## Test both Microservices using Postman
- Import postman project
- Add Environment URL or update environment url
- Test **Create User service** and verify the email id to confirm account creation email received.
- Test **Send Notification service** and verify the email id to confirm test email received. 