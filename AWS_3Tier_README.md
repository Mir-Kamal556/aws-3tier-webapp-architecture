# AWS 3-Tier Web Application Architecture

A production-style, highly available 3-tier web application deployed on AWS — featuring multi-AZ infrastructure, Application Load Balancer, EC2 instances with Apache + PHP, phpMyAdmin, and RDS MySQL backend.

---

## Architecture Overview

```
Internet
    │
    ▼
Route 53 (DNS)
    │
    ▼
Application Load Balancer (project-alb) ← alb-sg (port 80)
    │              │
    ▼              ▼
app-server1    app-server2
(us-east-1a)   (us-east-1b)
Apache+PHP     Apache+PHP
phpMyAdmin     phpMyAdmin
    │              │
    └──────┬───────┘
           ▼
     RDS MySQL (Multi-AZ)
      db-pvt-sub1/sub2
```

---

## Infrastructure Details

### VPC & Networking (Part 1)

| Resource | Name | Details |
|---|---|---|
| VPC | awsproject-vpc | CIDR: 20.0.0.0/20 |
| Public Subnet 1 | web-pub-sub1 | 20.0.1.0/24 — us-east-1a |
| Public Subnet 2 | web-pub-sub2 | 20.0.2.0/24 — us-east-1b |
| App Subnet 1 | app-pvt-sub1 | 20.0.3.0/24 — us-east-1a |
| App Subnet 2 | app-pvt-sub2 | 20.0.4.0/24 — us-east-1b |
| DB Subnet 1 | db-pvt-sub1 | 20.0.5.0/24 — us-east-1a |
| DB Subnet 2 | db-pvt-sub2 | 20.0.6.0/24 — us-east-1b |
| Internet Gateway | my-igw | Attached to awsproject-vpc |
| NAT Gateway | my-nat | In web-pub-sub1, Elastic IP allocated |

### Route Tables

| Route Table | Subnet Association | Target |
|---|---|---|
| route-web | web-pub-sub1, web-pub-sub2 | 0.0.0.0/0 → my-igw |
| route-app | app-pvt-sub1, app-pvt-sub2 | 0.0.0.0/0 → my-nat |
| route-db | db-pvt-sub1, db-pvt-sub2 | 0.0.0.0/0 → my-nat |

### Security Groups

| Security Group | Inbound Rules |
|---|---|
| jump-sg | SSH (22), HTTP (80) — 0.0.0.0/0 |
| alb-sg | HTTP (80) — 0.0.0.0/0 |
| app-sg | SSH (22), All traffic from alb-sg |
| db-sg | MySQL/Aurora (3306) from app-sg |

---

### EC2 Instances (Part 2)

| Instance | Subnet | Public IP | Role |
|---|---|---|---|
| jump-server | web-pub-sub1 | Enabled | Bastion Host |
| app-server1 | app-pvt-sub1 | Disabled | Apache + PHP + phpMyAdmin |
| app-server2 | app-pvt-sub2 | Disabled | Apache + PHP + phpMyAdmin |

All instances: Amazon Linux 2023, t3.micro, keypair: projectkey

### App Server Setup Commands

```bash
# Update system
sudo yum update -y

# Install PHP 8.2
sudo dnf install php8.2
sudo yum install php8.2-mysqlnd

# Install Apache
sudo yum install -y httpd
sudo systemctl start httpd
sudo systemctl enable httpd

# Set permissions
sudo usermod -a -G apache ec2-user
sudo chown -R ec2-user:apache /var/www
sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
find /var/www -type f -exec sudo chmod 0664 {} \;

# Install PHP modules
sudo yum install php-mbstring php-xml -y
sudo yum install php-fpm
sudo systemctl restart httpd
sudo systemctl restart php-fpm

# Install phpMyAdmin
cd /var/www/html
wget https://www.phpmyadmin.net/downloads/phpMyAdmin-latest-all-languages.tar.gz
mkdir phpMyAdmin && tar -xvzf phpMyAdmin-latest-all-languages.tar.gz -C phpMyAdmin --strip-components 1
rm phpMyAdmin-latest-all-languages.tar.gz

# Test page
echo "PHP server 1" > /var/www/html/index.html   # on app-server1
echo "PHP server 2" > /var/www/html/index.html   # on app-server2
```

### Load Balancer

- **Name:** project-alb
- **Type:** Application Load Balancer (Internet-facing)
- **VPC:** awsproject-vpc
- **AZs:** us-east-1a, us-east-1b
- **Security Group:** alb-sg
- **Listener:** HTTP port 80 → Target group: app-tg
- **Target Group:** app-tg (app-server1 + app-server2, port 80, stickiness enabled)

✅ Verified: Refreshing the ALB DNS alternates between "PHP server 1" and "PHP server 2" — confirming load balancing is working.

---

### RDS MySQL (Part 3)

| Setting | Value |
|---|---|
| Engine | MySQL |
| Template | Free tier |
| Instance identifier | mydbproject |
| Instance class | db.t3.micro |
| Subnet group | db-subnetgroup (db-pvt-sub1 + db-pvt-sub2) |
| Public access | No |
| Security group | db-sg (port 3306 from app-sg only) |
| Availability Zone | us-east-1a |

### Connect phpMyAdmin to RDS

```bash
cd /var/www/html/phpMyAdmin
mv config.sample.inc.php config.inc.php
nano config.inc.php
# Replace 'localhost' with your RDS endpoint
```

✅ Verified: phpMyAdmin accessible via ALB DNS — successfully connected to RDS MySQL cluster.

---

## Proof of Work

| Step | Status |
|---|---|
| VPC with 6 subnets across 2 AZs | ✅ |
| Internet Gateway + NAT Gateway | ✅ |
| Route tables configured | ✅ |
| Jump server (bastion host) running | ✅ |
| app-server1 + app-server2 running | ✅ |
| Apache + PHP 8.2 installed on both servers | ✅ |
| phpMyAdmin installed and configured | ✅ |
| ALB routing traffic between both servers | ✅ |
| Load balancing verified (PHP server 1 / PHP server 2) | ✅ |
| RDS MySQL created in private subnets | ✅ |
| phpMyAdmin connected to RDS endpoint | ✅ |

---

## Technologies Used

`AWS VPC` `EC2` `RDS MySQL` `Application Load Balancer` `NAT Gateway` `Internet Gateway` `Security Groups` `Route Tables` `Apache HTTP Server` `PHP 8.2` `phpMyAdmin` `Amazon Linux 2023`

---

## Author

**Mir Kamal Uddin**  
AWS & DevOps Engineer | IT Support Engineer  
📍 Dubai, UAE  
[LinkedIn](https://www.linkedin.com/in/mirkamal-uddin41639029a/) | [GitHub](https://github.com/Mir-Kamal556)
