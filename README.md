# Deploying a Secure Dockerized LAMP Stack with Amazon RDS and AWS Secrets Manager
This lab guides you through Dockerizing a LAMP stack with Amazon RDS as the database, replacing MySQL. AWS Secrets Manager is used to securely manage database credentials, enhancing security and scalability.

## Prerequisites

* AWS CLI installed and configured
* An AWS account with IAM permissions to create RDS and Secrets Manager resources
* Docker and Docker Compose installed

## Step 1: Create an Amazon RDS MySQL Instance

* Log in to AWS Management Console.
* Navigate to RDS > Create database.
* Select Standard create and choose MySQL.
* Choose Free tier for testing purposes.
* Set database credentials:
*   Username: admin
*   Password: YourSecurePassword
* Enable Public Access (for now, but security groups will restrict access).
* Configure security groups to allow access from your local machine.
* Click Create database and wait for the instance to be available.
* Note the Endpoint from the RDS console.

## Step 2: Store Database Credentials in AWS Secrets Manager

* Navigate to Secrets Manager in AWS.
* Click Store a new secret.
* Select Credentials for RDS database.
*   Enter:
*   Username: admin
*   Password: YourSecurePassword
* Choose the RDS instance created earlier.
* Click Next, name the secret as lamp-rds-secret.
* Click Store secret.

## Step 3: Update docker-compose.yml

Modify your docker-compose.yml to remove MySQL and fetch credentials from AWS Secrets Manager.

```docker-compose
version: '3.8'

services:
  lamp_php:
    build: .
    container_name: lamp_php
    volumes:
      - ./src:/var/www/html
    networks:
      - lamp_network
    environment:
      DB_HOST: "${DB_HOST}"
      DB_USER: "${DB_USER}"
      DB_PASSWORD: "${DB_PASSWORD}"
      DB_NAME: "${DB_NAME}"

  lamp_nginx:
    image: nginx:latest
    container_name: lamp_nginx
    ports:
      - "8080:80"
    volumes:
      - ./src:/var/www/html
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - lamp_php
    networks:
      - lamp_network

  lamp_phpmyadmin:
    image: phpmyadmin/phpmyadmin
    container_name: lamp_phpmyadmin
    restart: always
    ports:
      - "8081:80"
    environment:
      PMA_HOST: "${DB_HOST}"
      PMA_PORT: "3306"
      PMA_ARBITRARY: "1"
    networks:
      - lamp_network

networks:
  lamp_network:

```

## Step 4: Fetch Secrets from AWS Secrets Manager

Install AWS CLI if not installed.
Run the following command to retrieve the secrets and export them as environment variables:

```powershell
$secret = aws secretsmanager get-secret-value --secret-id lamp-rds-secret --query SecretString --output text | ConvertFrom-Json
$env:DB_HOST = "$($secret.host)"
$env:DB_USER = "$($secret.username)"
$env:DB_PASSWORD = "$($secret.password)"
$env:DB_NAME = "$($secret.dbname)"
```

Verify the environment variables:

```powershell
echo $env:DB_HOST
```
## Step 5: Start Docker Containers

Run the following command to start the updated stack:
```powershell
docker-compose up -d --build
```
## Step 6: Test the Setup
* Open http://localhost:8080 in your browser to check if the PHP application is working.
* Open http://localhost:8081 for phpMyAdmin and log in with:
*   Host: The RDS endpoint
*  Username: admin
*   Password: The stored password

## Step 7. Cleanup

If you want to remove the setup:
* Stop and remove containers:
```powershell
docker-compose down -v
```

Delete the AWS resources:
* Delete the RDS instance.
* Remove the secret from AWS Secrets Manager.
