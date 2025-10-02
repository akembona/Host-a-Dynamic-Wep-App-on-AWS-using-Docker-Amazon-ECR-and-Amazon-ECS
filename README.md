![Alt text]()

# Host a Dynamic Web App on AWS with Docker, Amazon ECR, and Amazon ECS

This project demonstrates how to **host a dynamic web application on AWS** using **Docker, Amazon Elastic Container Registry (ECR), Amazon Elastic Container Service (ECS Fargate), and supporting AWS services**. The deployment is based on a **3-tier architecture** (Presentation, Application, and Database tiers), ensuring scalability, high availability, and security.

The repository contains:

* A **Dockerfile** to containerize the application.
* **Scripts** used for deployment on AWS.
* A **reference architecture diagram** of the solution.

---

## 🚀 Project Overview

The main objectives of this project were:

* Containerize a **PHP/MySQL-based dynamic application** using **Docker**.
* Push container images to **Amazon ECR**.
* Deploy the app to **ECS Fargate** with high availability.
* Set up **Amazon RDS MySQL** for database management.
* Secure the application with **Route 53 (DNS)** and **AWS Certificate Manager (SSL/TLS)**.
* Enable **scalability** using Auto Scaling for ECS tasks.

---

## 🛠️ Tools & Technologies

* **Docker** – build container images.
* **Git & GitHub** – source code versioning and repository management.
* **AWS CLI** – interact with AWS services via the command line.
* **Flyway** – migrate SQL data into RDS MySQL.
* **VS Code** – code editing and script development.
* **Amazon ECR** – store Docker images.
* **Amazon ECS (Fargate)** – deploy containerized applications.
* **Amazon RDS (MySQL)** – relational database for the app.
* **Amazon S3** – store environment variable files.
* **Route 53** – custom domain name and DNS management.
* **AWS Certificate Manager** – secure HTTPS connections.
* **IAM Roles** – permissions for ECS to pull images and execute tasks.
* **Application Load Balancer (ALB)** – distribute traffic to ECS tasks.
* **Auto Scaling** – handle traffic spikes by scaling ECS tasks.
* **Bastion Host** – SSH access to resources in private subnets.
* **Security Groups** – control inbound/outbound traffic.

---

## 📊 AWS Architecture

* **3-Tier VPC**:

  * **Public Subnets** → Bastion Host, NAT Gateway, ALB.
  * **Private Subnets** → ECS tasks (App tier) + RDS (DB tier).
* **Networking**: Internet Gateway, Route Tables, NAT Gateways.
* **Scaling & Load Balancing**: ECS Fargate with ALB + Auto Scaling.
* **Security**: IAM roles, Security Groups, SSL certificates.

📌 *(Architecture diagram included in repository)*

---

## 📦 Deployment Workflow

1. **Build Docker Image**

   ```bash
   docker build -t dynamic-web-app .
   ```

2. **Test Container Locally**

   ```bash
   docker run -p 80:80 dynamic-web-app
   ```

3. **Push to Amazon ECR**

   * Authenticate:

     ```bash
     aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account_id>.dkr.ecr.<region>.amazonaws.com
     ```
   * Tag & push:

     ```bash
     docker tag dynamic-web-app:latest <account_id>.dkr.ecr.<region>.amazonaws.com/dynamic-web-app:latest
     docker push <account_id>.dkr.ecr.<region>.amazonaws.com/dynamic-web-app:latest
     ```

4. **Create ECS Cluster & Task Definition**

   * Task definition specifies container image from **ECR**.
   * Fargate service runs containers in private subnets.

5. **Setup Database (Amazon RDS)**

   * MySQL RDS instance in private subnet.
   * Flyway migration applied for schema/data.

6. **Configure Load Balancer & Auto Scaling**

   * ALB distributes traffic to Fargate tasks.
   * Auto Scaling adjusts tasks based on traffic demand.

7. **Domain & Security**

   * Route 53 connects custom domain → ALB.
   * ACM issues SSL certificate for HTTPS.

---

## 📜 Dockerfile

```dockerfile
# Use Amazon Linux 2 base image
FROM amazonlinux:2

# Update and install dependencies
RUN yum update -y \
 && yum install -y unzip wget httpd git \
 && amazon-linux-extras enable php7.4 \
 && yum clean metadata \
 && yum install -y php php-common php-pear php-cgi php-curl php-mbstring \
    php-gd php-mysqlnd php-gettext php-json php-xml php-fpm php-intl php-zip \
 && wget https://repo.mysql.com/mysql80-community-release-el7-3.noarch.rpm \
 && rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023 \
 && yum localinstall -y mysql80-community-release-el7-3.noarch.rpm \
 && yum install -y mysql-community-server

WORKDIR /var/www/html

# Build arguments and environment variables
ARG PERSONAL_ACCESS_TOKEN
ARG GITHUB_USERNAME
ARG REPOSITORY_NAME
ARG WEB_FILE_ZIP
ARG WEB_FILE_UNZIP
ARG DOMAIN_NAME
ARG RDS_ENDPOINT
ARG RDS_DB_NAME
ARG RDS_MASTER_USERNAME
ARG RDS_DB_PASSWORD

ENV PERSONAL_ACCESS_TOKEN=$PERSONAL_ACCESS_TOKEN \
    GITHUB_USERNAME=$GITHUB_USERNAME \
    REPOSITORY_NAME=$REPOSITORY_NAME \
    WEB_FILE_ZIP=$WEB_FILE_ZIP \
    WEB_FILE_UNZIP=$WEB_FILE_UNZIP \
    DOMAIN_NAME=$DOMAIN_NAME \
    RDS_ENDPOINT=$RDS_ENDPOINT \
    RDS_DB_NAME=$RDS_DB_NAME \
    RDS_MASTER_USERNAME=$RDS_MASTER_USERNAME \
    RDS_DB_PASSWORD=$RDS_DB_PASSWORD

# Clone app, configure .env, and set permissions
RUN git clone https://$PERSONAL_ACCESS_TOKEN@github.com/$GITHUB_USERNAME/$REPOSITORY_NAME.git \
 && unzip $REPOSITORY_NAME/$WEB_FILE_ZIP -d $REPOSITORY_NAME/ \
 && cp -av $REPOSITORY_NAME/$WEB_FILE_UNZIP/. /var/www/html \
 && rm -rf $REPOSITORY_NAME \
 && sed -i '/<Directory "\/var\/www\/html">/,/<\/Directory>/ s/AllowOverride None/AllowOverride All/' /etc/httpd/conf/httpd.conf \
 && chmod -R 777 /var/www/html storage/ \
 && sed -i '/^APP_ENV=/ s/=.*$/=production/' .env \
 && sed -i "/^APP_URL=/ s/=.*$/=https:\/\/$DOMAIN_NAME\//" .env \
 && sed -i "/^DB_HOST=/ s/=.*$/=$RDS_ENDPOINT/" .env \
 && sed -i "/^DB_DATABASE=/ s/=.*$/=$RDS_DB_NAME/" .env \
 && sed -i "/^DB_USERNAME=/ s/=.*$/=$RDS_MASTER_USERNAME/" .env \
 && sed -i "/^DB_PASSWORD=/ s/=.*$/=$RDS_DB_PASSWORD/" .env

COPY AppServiceProvider.php app/Providers/AppServiceProvider.php

EXPOSE 80 3306

ENTRYPOINT ["/usr/sbin/httpd", "-D", "FOREGROUND"]
```

---

## ✅ Key Features

* **Dynamic web application containerized with Docker**
* **Secure 3-tier AWS architecture** (Web, App, DB)
* **Amazon ECS Fargate** for serverless container orchestration
* **Amazon RDS (MySQL)** for persistent data storage
* **Scalability** with Auto Scaling Group
* **Secure communication** via ACM SSL certificate
* **Custom domain** with Route 53

---

## 📌 Repository Structure

```
.
├── Dockerfile
├── scripts/
│   ├── ecr-push.sh
│   ├── ecs-deploy.sh
│   └── flyway-migrate.sh
├── architecture-diagram.png
└── README.md
```

---

## 👨‍💻 Author

**Emmanuel Azure**
Cloud & DevOps Engineer | AWS | Docker | CI/CD
