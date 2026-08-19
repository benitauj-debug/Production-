# AWS Two-Tier High Availability Infrastructure

## Project Overview

This project demonstrates the design and deployment of a two-tier highly available web infrastructure on AWS.

The infrastructure was designed across two Availability Zones using public and private subnets. The public tier contains the internet-facing components, while the private tier contains the web servers.

The architecture also includes Amazon EFS for shared storage, an Application Load Balancer for traffic distribution, an Auto Scaling Group for high availability and scalability, a bastion/jump server for administrative access, Route 53 for DNS management, AWS Certificate Manager for HTTPS, and Datadog with Slack integration for monitoring and alerting.

---

## Table of Contents

1. [Project Objectives](#1-project-objectives)
2. [Architecture Overview](#2-architecture-overview)
3. [Availability Zone Design](#3-availability-zone-design)
4. [Create the VPC](#4-create-the-vpc)
5. [Create the Public Subnets](#5-create-the-public-subnets)
6. [Create the Private Subnets](#6-create-the-private-subnets)
7. [Internet Gateway](#7-internet-gateway)
8. [Public Route Table](#8-public-route-table)
9. [NAT Gateway](#9-nat-gateway)
10. [Private Route Table](#10-private-route-table)
11. [Security Groups](#11-security-groups)
12. [Create the Jump/Bastion Server](#12-create-the-jumpbastion-server)
13. [Create the SSH Key Pair](#13-create-the-ssh-key-pair)
14. [Connect to the Jump Server Using MobaXterm](#14-connect-to-the-jump-server-using-mobaxterm)
15. [Create Amazon EFS](#15-create-amazon-efs)
16. [Mount EFS](#16-mount-efs)
17. [Upload Website Content to EFS](#17-upload-website-content-to-efs)
18. [Install and Configure Nginx](#18-install-and-configure-nginx)
19. [Create the EC2 Launch Template](#19-create-the-ec2-launch-template)
20. [Auto Scaling Group](#20-auto-scaling-group)
21. [Initial Web Server Deployment](#21-initial-web-server-deployment)
22. [Application Load Balancer](#22-application-load-balancer)
23. [Create the Target Group](#23-create-the-target-group)
24. [Connect Auto Scaling to the Load Balancer](#24-connect-auto-scaling-to-the-load-balancer)
25. [Configure Load Balancer Listeners](#25-configure-load-balancer-listeners)
26. [Configure DNS with Route 53](#26-configure-dns-with-route-53)
27. [Configure AWS Certificate Manager](#27-configure-aws-certificate-manager)
28. [Install Datadog](#28-install-datadog)
29. [Configure Datadog Monitoring](#29-configure-datadog-monitoring)
30. [Configure CPU Monitoring](#30-configure-cpu-monitoring)
31. [Integrate Datadog with Slack](#31-integrate-datadog-with-slack)
32. [Complete Architecture](#32-complete-architecture)
33. [High Availability Mechanism](#33-high-availability-mechanism)
34. [Failure Scenario](#34-failure-scenario)
35. [Auto Scaling Scenario](#35-auto-scaling-scenario)
36. [Security Architecture](#36-security-architecture)
37. [DNS Architecture](#37-dns-architecture)
38. [Monitoring Architecture](#38-monitoring-architecture)
39. [Testing and Validation](#39-testing-and-validation)
40. [Project Evidence](#40-project-evidence)
41. [Project Screenshots](#41-project-screenshots)
42. [Project Directory for GitHub](#42-project-directory-for-github)
43. [Important Security Note for GitHub](#43-important-security-note-for-github)
44. [Production Improvements](#44-production-improvements)
45. [Final Architecture Summary](#45-final-architecture-summary)
46. [Key Skills Demonstrated](#46-key-skills-demonstrated)
47. [Conclusion](#47-conclusion)

---

## 1. Project Objectives

The main objectives of this project were to:

- Build a custom AWS VPC.
- Create a highly available network across two Availability Zones.
- Separate public and private resources.
- Provide internet access to public resources through an Internet Gateway.
- Provide outbound internet access to private resources through a NAT Gateway.
- Deploy web servers in private subnets.
- Use Amazon EFS for shared website storage.
- Deploy Nginx as the web server.
- Automate server configuration using an EC2 Launch Template.
- Use an Auto Scaling Group to maintain multiple web servers.
- Distribute traffic using an Application Load Balancer.
- Provide administrative access through a bastion/jump server.
- Manage the domain using Dynadot and Amazon Route 53.
- Secure the website using AWS Certificate Manager.
- Monitor infrastructure using Datadog.
- Send monitoring alerts to Slack.

---

## 2. Architecture Overview

The infrastructure was divided into two main tiers:

### Public Tier

The public tier contains resources that need internet connectivity.

These include:

- Application Load Balancer
- Bastion/Jump Server
- NAT Gateway

The public resources are located in public subnets.

### Private Tier

The private tier contains the web servers.

The web servers are located in private subnets and do not have direct inbound access from the internet.

They receive application traffic through the Application Load Balancer.

The private servers use the NAT Gateway when they need outbound internet access.

---

## 3. Availability Zone Design

The infrastructure was distributed across two Availability Zones.

The subnet structure was:

```
Availability Zone A
├── Public Subnet 01
└── Private Subnet 01

Availability Zone B
├── Public Subnet 02
└── Private Subnet 02
```

This design provides redundancy.

If one Availability Zone experiences a failure, the resources in the second Availability Zone can continue supporting the application.

---

## 4. Create the VPC

The first step was creating a custom VPC.

### Configuration

| Setting | Value |
|---|---|
| VPC Name | `benita-cloud` |
| IPv4 CIDR | `192.168.0.0/16` |

The VPC provides the isolated networking environment for all AWS resources used in the project.

### AWS Console Steps

1. Open the AWS Management Console.
2. Navigate to VPC.
3. Select Your VPCs.
4. Select Create VPC.
5. Choose VPC only.
6. Enter the VPC name: `benita-cloud`
7. Enter the IPv4 CIDR: `192.168.0.0/16`
8. Create the VPC.

---

## 5. Create the Public Subnets

Two public subnets were created across two Availability Zones.

**Public Subnet 01** — Availability Zone A

**Public Subnet 02** — Availability Zone B

The two public subnets allow public-facing resources such as the Application Load Balancer and jump server to be distributed across multiple Availability Zones.

Public Subnet 01 sits in the same Availability Zone as Private Subnet 01 (Availability Zone A), and Public Subnet 02 sits in the same Availability Zone as Private Subnet 02 (Availability Zone B). Each AZ therefore has its own self-contained public/private subnet pair.

### AWS Console Steps

1. Open the VPC console.
2. Select Subnets.
3. Select Create subnet.
4. Select the `benita-cloud` VPC.
5. Create Public Subnet 01.
6. Select the first Availability Zone.
7. Assign an appropriate CIDR block.
8. Create Public Subnet 02.
9. Select the second Availability Zone.
10. Assign another non-overlapping CIDR block.
11. Create both subnets.

---

## 6. Create the Private Subnets

Two private subnets were created across the same two Availability Zones.

**Private Subnet 01** — Availability Zone A

**Private Subnet 02** — Availability Zone B

The private subnets contain the Nginx web servers.

Private Subnet 01 is paired with Public Subnet 01 in Availability Zone A, and Private Subnet 02 is paired with Public Subnet 02 in Availability Zone B — matching the pairing described in [Section 3, Availability Zone Design](#3-availability-zone-design).

The private subnets do not have a direct route to the Internet Gateway.

---

## 7. Internet Gateway

An Internet Gateway was created and attached to the VPC.

The Internet Gateway provides internet connectivity for resources in the public subnets.

### Steps

1. Open the VPC console.
2. Select Internet Gateways.
3. Select Create internet gateway.
4. Give it a descriptive name.
5. Create the Internet Gateway.
6. Select the newly created Internet Gateway.
7. Select Actions.
8. Choose Attach to a VPC.
9. Select `benita-cloud`.
10. Attach the Internet Gateway.

---

## 8. Public Route Table

A public route table was created for the public subnets.

The public route table contains a default route to the Internet Gateway.

| Destination | Target |
|---|---|
| `0.0.0.0/0` | Internet Gateway |

### Steps

1. Open Route Tables.
2. Select Create route table.
3. Name it: `public-route-table`
4. Select the `benita-cloud` VPC.
5. Create the route table.
6. Open the Routes tab.
7. Select Edit routes.
8. Add: `0.0.0.0/0` → Internet Gateway
9. Save the route.
10. Open Subnet Associations.
11. Associate Public Subnet 01.
12. Associate Public Subnet 02.

---

## 9. NAT Gateway

A NAT Gateway was created to allow resources in the private subnets to access the internet for outbound connections.

Private servers can use the NAT Gateway for activities such as:

- Installing Ubuntu updates
- Installing Nginx
- Installing EFS dependencies
- Installing the Datadog agent
- Downloading required packages

The NAT Gateway does not provide direct inbound internet access to the private servers.

Traffic flow:

```
Private Web Server
        |
        v
Private Route Table
        |
        v
    NAT Gateway
        |
        v
Internet Gateway
        |
        v
    Internet
```

### Steps

1. Open the VPC console.
2. Select NAT Gateways.
3. Select Create NAT Gateway.
4. Select a public subnet.
5. Allocate an Elastic IP address.
6. Create the NAT Gateway.
7. Wait until the NAT Gateway becomes available.

> For stronger production-grade multi-AZ resilience, a NAT Gateway can be deployed in each Availability Zone.

---

## 10. Private Route Table

A separate private route table was created for the private subnets.

The default route points to the NAT Gateway.

| Destination | Target |
|---|---|
| `0.0.0.0/0` | NAT Gateway |

### Steps

1. Open Route Tables.
2. Select Create route table.
3. Name it: `private-route-table`
4. Select the `benita-cloud` VPC.
5. Create the route table.
6. Open the Routes tab.
7. Add: `0.0.0.0/0` → NAT Gateway
8. Save the route.
9. Open Subnet Associations.
10. Associate Private Subnet 01.
11. Associate Private Subnet 02.

---

## 11. Security Groups

Security groups were created to control traffic between the different components.

The architecture uses separate security groups for public and private resources.

### Public Security Group

The public security group controls access to public-facing resources.

Required traffic may include:

| Port | Purpose |
|---|---|
| 80 | HTTP |
| 443 | HTTPS |
| 22 | SSH |

SSH should be restricted to a trusted administrator IP where possible.

### Private Security Group

The private security group protects the web servers.

The web servers should receive application traffic from the Application Load Balancer rather than directly from the internet.

The private security group can also allow SSH from the jump server security group.

```
ALB Security Group
       |
       | HTTP/HTTPS
       v
Private Web Server

Jump Server Security Group
       |
       | SSH
       v
Private Web Server
```

---

## 12. Create the Jump/Bastion Server

A jump server was created in one of the public subnets.

The purpose of the jump server is to provide controlled administrative access to resources inside the private network.

The jump server has a public IP address.

The private web servers remain inside the private subnets.

---

## 13. Create the SSH Key Pair

An EC2 key pair was created for SSH authentication.

AWS stores the **public key** associated with an EC2 key pair. The **private key** is generated at creation time and must be kept securely by the user — it is not retained by AWS.

The private key used for this project was the **"4036" private key**.

The private key was securely saved and used by MobaXterm to authenticate the SSH connection.

> **Note:** The private key should never be uploaded to GitHub or included in project documentation.

---

## 14. Connect to the Jump Server Using MobaXterm

MobaXterm was used as the SSH client.

The connection was established using:

- Jump server public IP address
- Username: `ubuntu`
- Private SSH key: `4036`
- SSH protocol

### Connection Process

1. Open MobaXterm.
2. Select Session.
3. Select SSH.
4. Enter the public IP address of the jump server.
5. Enter the username: `ubuntu`
6. Enable private key authentication.
7. Select the `4036` private key.
8. Start the SSH session.
9. Authenticate with the private key.
10. Confirm that the Ubuntu terminal opens successfully.

The connection path is:

```
Administrator
      |
      v
  MobaXterm
      |
      | SSH
      | ubuntu
      | 4036 private key
      v
Jump Server
      |
      v
Private Web Servers
```

The jump server therefore acts as the controlled administrative gateway into the private environment.

---

## 15. Create Amazon EFS

Amazon Elastic File System was created to provide shared storage for the web servers.

The reason for using EFS is that multiple EC2 instances need access to the same website files.

Instead of storing website content separately on each server, the website content is stored on EFS.

```
             EFS
            /   \
           /     \
          v       v
    Web Server 1  Web Server 2
```

### Steps

1. Open the AWS console.
2. Navigate to EFS.
3. Select Create file system.
4. Select the VPC: `benita-cloud`
5. Configure the required Availability Zones.
6. Configure EFS mount targets.
7. Apply the appropriate security group.
8. Create the file system.

> For a multi-AZ architecture, EFS mount targets should be available in the Availability Zones where the web servers operate.

---

## 16. Mount EFS

The EFS client and required dependencies were installed on the web server.

The EFS file system was then mounted on the server.

The mount allowed the web server to access the shared website files.

The website content was uploaded to the EFS-mounted directory.

The final structure was:

```
EFS
 |
 +-- Website files
 |
 +-- HTML files
 |
 +-- CSS
 |
 +-- JavaScript
 |
 +-- Images
```

---

## 17. Upload Website Content to EFS

After mounting EFS, the website content was uploaded to the EFS file system.

The website files became accessible to the Nginx web server.

This approach means that if an EC2 instance is terminated, the website files remain available on EFS.

---

## 18. Install and Configure Nginx

Nginx was selected as the web server.

The web server configuration included:

- Nginx installation
- Nginx service startup
- Website directory configuration
- EFS integration
- Website content

Nginx serves the website files stored on the EFS-mounted directory.

---

## 19. Create the EC2 Launch Template

A Launch Template was created to automate the configuration of the web servers.

The Launch Template contains the configuration required whenever a new EC2 instance is launched.

The configuration included:

1. Ubuntu operating system.
2. Nginx installation.
3. EFS client installation.
4. Required dependencies.
5. EFS mount configuration.
6. Datadog agent installation.
7. Datadog API key configuration.
8. Required startup commands.

The purpose of the Launch Template is to ensure that every new web server is configured consistently.

---

## 20. Auto Scaling Group

An Auto Scaling Group was created to manage the web server instances.

The configuration was:

| Setting | Value |
|---|---|
| Minimum capacity | 2 |
| Desired capacity | 2 |
| Maximum capacity | 20 |

The Auto Scaling Group launches the web servers using the Launch Template.

The web servers are placed in the private subnets.

The Auto Scaling Group provides:

- High availability
- Automatic instance replacement
- Horizontal scaling
- Consistent server configuration

---

## 21. Initial Web Server Deployment

After creating the Auto Scaling Group, two web server instances were launched.

The instances were deployed across the private subnets.

The initial architecture was:

```
Availability Zone A
Private Subnet 01
      |
      v
Web Server 1


Availability Zone B
Private Subnet 02
      |
      v
Web Server 2
```

This provides redundancy across Availability Zones.

---

## 22. Application Load Balancer

An Application Load Balancer was created to distribute incoming traffic across the web servers.

The ALB was deployed across the public subnets.

The ALB provides the public entry point for the application.

Traffic flow:

```
Internet User
      |
      v
Application Load Balancer
      |
      v
  Target Group
     /      \
    /        \
   v          v
Web 1        Web 2
```

---

## 23. Create the Target Group

A target group was created for the web servers.

The target group contains the EC2 instances launched by the Auto Scaling Group.

Health checks were configured to determine whether the web servers were healthy.

If a server fails its health check, the Application Load Balancer stops sending traffic to it.

---

## 24. Connect Auto Scaling to the Load Balancer

The Auto Scaling Group was associated with the Application Load Balancer target group.

This means that instances launched by the Auto Scaling Group can automatically become targets of the load balancer.

The resulting flow is:

```
Auto Scaling Group
       |
       v
  Target Group
       |
       v
Application Load Balancer
```

When a new instance is launched, it can be registered with the target group.

---

## 25. Configure Load Balancer Listeners

The Application Load Balancer was configured to listen for web traffic.

The listeners include:

| Port | Protocol |
|---|---|
| 80 | HTTP |
| 443 | HTTPS |

The HTTPS listener is associated with the ACM certificate.

HTTP can be configured to redirect users to HTTPS.

---

## 26. Configure DNS with Route 53

The domain was registered through Dynadot.

Amazon Route 53 was used to manage the DNS hosted zone.

### Process

1. Create a hosted zone in Route 53.
2. Enter the domain name.
3. Create the hosted zone.
4. Copy the Route 53 name servers.
5. Open Dynadot.
6. Update the domain's name server configuration.
7. Replace the existing name servers with the Route 53 name servers.
8. Allow DNS propagation.
9. Configure the required DNS record in Route 53.
10. Point the domain to the Application Load Balancer.

The traffic flow becomes:

```
User
 |
 v
Domain
 |
 v
Route 53
 |
 v
Application Load Balancer
 |
 v
Private Web Servers
```

---

## 27. Configure AWS Certificate Manager

AWS Certificate Manager was used to secure the website with HTTPS.

### Steps

1. Open AWS Certificate Manager.
2. Select Request a certificate.
3. Select a public certificate.
4. Enter the application domain.
5. Select DNS validation.
6. Create the certificate request.
7. Complete DNS validation through Route 53.
8. Wait for the certificate to become issued.
9. Open the Application Load Balancer.
10. Select the HTTPS listener.
11. Attach the ACM certificate.
12. Save the configuration.

The final HTTPS flow is:

```
User
 |
 | HTTPS :443
 v
Application Load Balancer
 |
 v
Private Web Servers
```

---

## 28. Install Datadog

Datadog was integrated into the infrastructure for monitoring.

The Datadog agent was installed on:

- Web Server 1
- Web Server 2
- Jump Server

The Datadog configuration was included in the web server Launch Template so that new instances could automatically install the monitoring agent.

This is important for Auto Scaling because new instances can automatically become visible in Datadog.

---

## 29. Configure Datadog Monitoring

After installing the Datadog agent, the EC2 instances appeared in Datadog.

The monitored infrastructure included:

- Jump Server
- Web Server 1
- Web Server 2

Datadog was used to monitor infrastructure metrics such as:

- CPU utilization
- Memory utilization
- Disk utilization
- Network activity
- Host availability

---

## 30. Configure CPU Monitoring

A CPU monitor was created in Datadog.

The purpose of the monitor was to identify abnormal CPU utilization.

The monitoring workflow is:

```
EC2 Web Server
      |
      v
Datadog Agent
      |
      v
   Datadog
      |
      v
 CPU Monitor
      |
      v
    Alert
```

This allows infrastructure problems to be identified before they significantly affect application availability.

---

## 31. Integrate Datadog with Slack

Datadog was integrated with Slack to receive infrastructure alerts.

The workflow is:

```
Web Server
    |
    v
CPU Usage
    |
    v
Datadog Monitor
    |
    v
Alert Triggered
    |
    v
   Slack
```

This allows the infrastructure team to receive monitoring alerts without constantly checking the Datadog dashboard.

---

## 32. Complete Architecture

The completed architecture can be represented as follows:

```
                         INTERNET
                             |
                             v
                          DOMAIN
                             |
                             v
                         ROUTE 53
                             |
                             v
                APPLICATION LOAD BALANCER
                  /                      \
                 /                        \
                v                          v
        PUBLIC SUBNET 01          PUBLIC SUBNET 02
                |                         |
                |                    ALB / Public
                |
          JUMP SERVER
                |
                | SSH
                v
        PRIVATE SUBNET 01        PRIVATE SUBNET 02
                |                         |
                v                         v
          WEB SERVER 1              WEB SERVER 2
                \                         /
                 \                       /
                  +--------- EFS --------+
                           |
                    Shared Web Content
```

```
Private Web Servers
        |
        v
Private Route Table
        |
        v
    NAT Gateway
        |
        v
Internet Gateway
        |
        v
    Internet
```

Monitoring:

```
Web Servers + Jump Server
          |
          v
      Datadog
          |
          v
  CPU Monitoring
          |
          v
        Slack
```

---

## 33. High Availability Mechanism

The architecture achieves high availability through several components working together.

**Multiple Availability Zones**
The infrastructure is distributed across two Availability Zones.

**Multiple Web Servers**
At least two web servers are maintained by the Auto Scaling Group.

**Application Load Balancer**
The ALB distributes traffic across healthy web servers.

**Auto Scaling Group**
The Auto Scaling Group automatically replaces failed instances and can scale the number of instances based on configured policies.

**Amazon EFS**
EFS provides shared persistent storage accessible by multiple web servers.

**NAT Gateway**
Private servers can access the internet for outbound operations without being directly exposed to inbound internet traffic.

---

## 34. Failure Scenario

If Web Server 1 fails, the architecture is designed to continue serving traffic through Web Server 2.

```
             Application Load Balancer
                       |
                 Health Check
                       |
              +--------+--------+
              |                 |
              v                 v
        Web Server 1        Web Server 2
         UNHEALTHY             HEALTHY
              X                  |
                                 |
                                 v
                              USERS
```

The Auto Scaling Group can then launch a replacement instance.

The replacement instance uses the Launch Template to automatically:

1. Launch Ubuntu.
2. Install dependencies.
3. Install EFS client.
4. Mount EFS.
5. Install Nginx.
6. Install Datadog.
7. Join the target group after becoming healthy.

This reduces the amount of manual intervention required during instance failure.

---

## 35. Auto Scaling Scenario

If application demand increases, additional instances can be launched.

For example:

**Normal Traffic**

```
Web Server 1
Web Server 2
```

**During increased demand:**

```
Web Server 1
Web Server 2
Web Server 3
Web Server 4
...
```

The Auto Scaling Group can scale up to the configured maximum of **20 instances**.

The Application Load Balancer distributes traffic across the healthy instances.

---

## 36. Security Architecture

The infrastructure follows a layered security model.

**Network Isolation**
The VPC separates public and private resources.

**Private Web Servers**
The web servers do not require public IP addresses.

**Security Groups**
Security groups control communication between the ALB, jump server, and private web servers.

**Bastion Server**
Administrative SSH access is routed through the jump server.

**HTTPS**
The application uses HTTPS through the Application Load Balancer and AWS Certificate Manager.

**Shared Storage Protection**
EFS access is controlled through its associated security group.

**Private Internet Access**
Private servers use the NAT Gateway for outbound internet access.

---

## 37. DNS Architecture

The DNS architecture consists of:

```
Dynadot
    |
    | Domain Registration
    v
Route 53
    |
    | DNS Resolution
    v
Application Load Balancer
    |
    v
Private Web Servers
```

Dynadot remains the domain registrar, while Route 53 handles the DNS hosted zone after the domain's name servers were updated.

---

## 38. Monitoring Architecture

The monitoring architecture consists of:

```
EC2 Web Server 1
        |
EC2 Web Server 2
        |
   Jump Server
        |
        v
 Datadog Agent
        |
        v
     Datadog
        |
        v
  CPU Monitor
        |
        v
      Slack
```

This provides centralized visibility into the infrastructure.

---

## 39. Testing and Validation

After completing the infrastructure, the following tests were performed.

**VPC Test**
Confirmed that the VPC was available and contained: `192.168.0.0/16`

**Subnet Test**
Confirmed that the VPC contained:

- Public Subnet 01
- Private Subnet 01
- Public Subnet 02
- Private Subnet 02

**Routing Test**
Confirmed that:

- Public Subnets → Internet Gateway
- Private Subnets → NAT Gateway

**SSH Test**
Confirmed that the jump server could be accessed through MobaXterm using:

- Username: `ubuntu`
- Private Key: `4036`

**EFS Test**
Confirmed that the EFS file system was mounted and that the website files were accessible.

**Nginx Test**
Confirmed that Nginx was installed and running on the web servers.

**Load Balancer Test**
Confirmed that the web server targets were registered with the Application Load Balancer.

**Health Check Test**
Confirmed that the ALB could determine whether the web servers were healthy.

**Auto Scaling Test**
Confirmed that the Auto Scaling Group maintained the required web server capacity.

**DNS Test**
Confirmed that the domain resolved through Route 53.

**HTTPS Test**
Confirmed that the website could be accessed securely through HTTPS.

**Datadog Test**
Confirmed that the web servers and jump server appeared in Datadog.

**Slack Test**
Confirmed that the configured Datadog monitoring alert could send notifications to Slack.

---

## 40. Project Evidence

Screenshots were captured during the deployment process to document the implementation.

Important evidence includes:

1. VPC configuration.
2. Subnet configuration.
3. Route tables.
4. Internet Gateway.
5. NAT Gateway.
6. Security Groups.
7. Jump Server.
8. MobaXterm SSH connection.
9. EFS configuration.
10. EFS mounted on the server.
11. Website files stored on EFS.
12. Launch Template.
13. Auto Scaling Group.
14. EC2 instances.
15. Application Load Balancer.
16. Target Group health checks.
17. Route 53 hosted zone.
18. Dynadot name server configuration.
19. AWS Certificate Manager certificate.
20. Datadog hosts.
21. Datadog CPU monitor.
22. Slack integration and alerts.

---

## 41. Project Screenshots

The VPC screenshot shows the custom VPC and subnet architecture.

**VPC**

| Field | Value |
|---|---|
| VPC Name | `benita-cloud` |
| CIDR | `192.168.0.0/16` |

**Subnets**

- Public Subnet 01
- Private Subnet 01
- Public Subnet 02
- Private Subnet 02

The screenshot also shows the association between the subnets and the public/private route tables.

The architecture diagram illustrates the relationship between:

- Internet
- Load Balancer
- Public tier
- Private tier
- Jump server
- Web servers
- EFS
- NAT Gateway
- Monitoring

---

## 42. Project Directory for GitHub

The GitHub repository can be organized as follows:

---

## 44. Production Improvements

Although the infrastructure is highly available, several improvements can make it more production-ready.

**NAT Gateway Per Availability Zone**
Deploy one NAT Gateway in each Availability Zone and route each private subnet through its local NAT Gateway.

**Secrets Management**
Move the Datadog API key out of the Launch Template and use AWS Secrets Manager or Systems Manager Parameter Store.

**IAM Roles**
Use IAM roles for EC2 instances instead of storing AWS credentials on the servers.

**Restricted SSH**
Allow SSH to the jump server only from trusted administrator IP addresses. Allow SSH from the jump server to private servers rather than opening SSH to the internet.

**Logging**
Centralize Nginx and system logs using CloudWatch or Datadog.

**Auto Scaling Policies**
Configure CPU-based or application-based scaling policies to automatically increase or decrease the number of instances based on demand.

---

## 45. Final Architecture Summary

The completed AWS infrastructure consists of a highly available two-tier architecture distributed across two Availability Zones.

The public tier contains the Application Load Balancer, NAT Gateway, and bastion/jump server.

The private tier contains the Nginx web servers managed by an Auto Scaling Group.

The Auto Scaling Group maintains a minimum of two instances and can scale to a maximum of twenty instances.

Amazon EFS provides shared persistent storage for the website content.

The Application Load Balancer distributes incoming traffic to healthy private web servers.

Private web servers access the internet only through the NAT Gateway.

The jump server provides controlled SSH access to the private environment using MobaXterm, the Ubuntu username, the jump server public IP address, and the `4036` private key.

Route 53 manages DNS resolution, while Dynadot remains the domain registrar.

AWS Certificate Manager provides the SSL/TLS certificate used by the Application Load Balancer to secure HTTPS traffic.

Datadog monitors the web servers and jump server, while Slack receives configured infrastructure alerts.

The overall architecture provides:

- High availability
- Fault tolerance
- Horizontal scalability
- Network segmentation
- Secure administrative access
- Shared persistent storage
- Load balancing
- HTTPS encryption
- Infrastructure monitoring
- Automated server deployment

---

## 46. Key Skills Demonstrated

This project demonstrates practical experience with:

Amazon VPC, IPv4 CIDR planning, public and private subnets, Availability Zones, Route Tables, Internet Gateway, NAT Gateway, Security Groups, Amazon EC2, Bastion/Jump Servers, SSH, MobaXterm, Ubuntu, Nginx, Amazon EFS, EC2 Launch Templates, Auto Scaling Groups, Application Load Balancer, Target Groups, Route 53, Dynadot DNS, AWS Certificate Manager, HTTPS, Datadog, Slack, High Availability, Fault Tolerance, Infrastructure Monitoring, Cloud Security, and Infrastructure Automation.

---

## 47. Conclusion

This project provided hands-on experience designing and implementing a highly available AWS web infrastructure.

The final environment separates public-facing resources from private web servers, distributes resources across multiple Availability Zones, uses EFS for shared storage, uses an Application Load Balancer for traffic distribution, and uses an Auto Scaling Group to maintain application availability and scalability.

The addition of Route 53, AWS Certificate Manager, Datadog, and Slack completed the infrastructure by providing DNS management, HTTPS security, monitoring, and alerting.

The project demonstrates how multiple AWS services can be combined to build a secure, scalable, monitored, and highly available production-style web environment.
