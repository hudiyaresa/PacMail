
# PacMail Deployment

This repository contains the configuration and scripts needed to deploy the PacMail web application with assistance from Pacmann AI. This guide walks you through setting up the infrastructure, building Docker images, configuring CI/CD pipelines, and deploying the app to be publicly accessible.

## Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
  - [1. Server Preparation](#1-server-preparation)
  - [2. Docker Configuration](#2-docker-configuration)
    - [Create Custom Docker Image for Backend](#create-custom-docker-image-for-backend)
    - [Docker Compose Configuration](#docker-compose-configuration)
  - [3. CI/CD Pipeline Configuration](#3-cicd-pipeline-configuration)
    - [Branch Protection](#branch-protection)
    - [Continuous Integration Workflow](#continuous-integration-workflow)
    - [Continuous Delivery & Deployment Workflow](#continuous-delivery--deployment-workflow)
  - [4. Domain and Nginx Configuration](#4-domain-and-nginx-configuration)
- [Conclusion](#conclusion)
- [References](#references)

---

## Overview
### Overview

PacMail is a web-based email application providing essential features such as user login, sign-up, inbox management, and email sending. However, the deployment process previously faced key challenges, including the lack of automation, potential downtime during updates disrupting user experience, and limited public accessibility via a dedicated domain.

To address these issues, this repository implements a comprehensive solution using Docker, Docker Compose, and a robust GitHub Actions CI/CD pipeline. The implementation ensures seamless updates, uninterrupted service, and public availability of PacMail. The deployment process is streamlined into four key steps:

1. **Server Preparation:** Setting up a cloud-hosted server to host the application.  
2. **Application Composition:** Packaging the application into containers using Docker Compose.  
3. **CI/CD Pipeline:** Automating deployment with GitHub Actions and self-hosted runners.  
4. **Domain & Reverse Proxy Configuration:** Configuring Nginx, Certbot, and DNS to ensure secure public accessibility.  


## Prerequisites
- **Server**: A public-accessible server (e.g., VPS) for hosting the application.
- **Domain**: A domain name for public access (freeDNS).
- **Software**:
  - Docker & Docker Compose
  - Nginx
  - Firewall configured for required ports
  - GitHub repository with secrets for sensitive information (e.g., Docker Hub credentials, database credentials)

## Setup Instructions

### 1. Server Preparation
1. **Install Docker**: Install Docker and Docker Compose on your server.
2. **Install Nginx**: Install Nginx for handling reverse proxy to route traffic to the application.
3. **Configure Firewall**: Open necessary ports (e.g., HTTP - 80, HTTPS - 443).

### 2. Docker Configuration

#### Create Custom Docker Image for Backend
In the `backend` directory, create a `Dockerfile`:
```dockerfile
FROM python:3.10
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
EXPOSE 5000
CMD ["python3", "api.py"]
```

#### Docker Compose Configuration
Configure `docker-compose.yml` for all services (frontend, backend, and database).

```yaml
version: "3.8"

services:
  frontend-pacmail:
    image: node:latest
    container_name: frontend-container
    working_dir: /frontend
    restart: always
    depends_on:
      - backend-pacmail
      - database-pacmail
    volumes:
      - ./frontend:/frontend
    command: npm start
    network_mode: "host"
    env_file: 
      - ./frontend/.env
  
  backend-pacmail:
    build:
      context: ./backend
    container_name: backend-container
    hostname: backend-pacmail
    volumes:
      - ./backend:/app
    depends_on:
      - database-pacmail
    restart: always
    network_mode: "host"
    env_file: 
      - ./backend/.env
  
  database-pacmail:
    image: postgres:13
    container_name: database-container
    hostname: database-pacmail
    restart: always
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    network_mode: "host"
    ports:
      - 5432:5432

volumes:
  postgres_data:

```

### 3. CI/CD Pipeline Configuration

#### Branch Protection
1. Protect the main branch in the repository settings.
2. Only allow merges through approved pull requests.
![main branch protection](https://github.com/hudiyaresa/PacMail/blob/main/demo_screenshots/branch%20protection%20role.png)

#### Continuous Integration Workflow
Create a GitHub Actions workflow file ([`.github/workflows/ci.yml`](https://github.com/hudiyaresa/PacMail/blob/70b12cb7b1c99e948d4179cedc81343f7c5e5f17/.github/workflows/ci.yml)) for the CI process. A CI process will be triggered for any pull requests targeting the main branch. This process includes automated testing to validate application functionality:
![CI Process](https://github.com/hudiyaresa/PacMail/blob/main/demo_screenshots/01.%20pacmail%20build%20and%20test.png)

#### Continuous Delivery & Deployment Workflow
##### Staging
Create a GitHub Actions workflow file ([`.github/workflows/cd-staging.yml`](https://github.com/hudiyaresa/PacMail/blob/70b12cb7b1c99e948d4179cedc81343f7c5e5f17/.github/workflows/cd-staging.yml)) for the CD process. This process will be triggered by pushes or updates to the main branch, leading to application deployment to the staging server. This enables the QA team to test the application before production deployment:
![CD Process Staging](https://github.com/hudiyaresa/PacMail/blob/main/demo_screenshots/02.%20deploy%20staging.png)

##### Push to Registry
Create a GitHub Actions workflow file ([`.github/workflows/cd-push-registry.yml`](https://github.com/hudiyaresa/PacMail/blob/70b12cb7b1c99e948d4179cedc81343f7c5e5f17/.github/workflows/cd-push-registry.yml)) for the CD process. Once testing on the staging server is complete and the application is deemed ready for release, the Docker image used will be saved to Docker Hub. This process is triggered by a new tag on the main branch:
![CD Process Push Registry](https://github.com/hudiyaresa/PacMail/blob/main/demo_screenshots/03.%20push%20to%20registry.png)

##### Deploy to Production
Create a GitHub Actions workflow file ([`.github/workflows/cd-prod.yml`](https://github.com/hudiyaresa/PacMail/blob/70b12cb7b1c99e948d4179cedc81343f7c5e5f17/.github/workflows/cd-prod.yml)) for the CD process. A new tag release will initiate the deployment process, making the application available to end-users on the production server:
![CD Process Production](https://github.com/hudiyaresa/PacMail/blob/main/demo_screenshots/04.%20deploy%20production.png)


### 4. Domain and Nginx Configuration

1. **Set Up Domain**: Register a domain name for accessing the application.
2. **Nginx Reverse Proxy Configuration**:
   - Create an Nginx configuration to redirect traffic from port 80 to the frontend container on localhost:3000.

     ```
     cd /etc/nginx/sites-available
     sudo nano pacmail.groups.id
     ```

     ```nginx
    server {
        listen 80;
        server_name pacmail.groups.id;

        location / {
            proxy_pass http://public-ip:3000;
        }

        location /api {
            proxy_pass http://public-ip:5000;
        }
    }
     ```

     - creating a symlink to the sites-enabled directory
        ```
        sudo ln -s /etc/nginx/sites-available/pacmail.groups.id /etc/nginx/sites-enabled
        ```

     - check the syntax and test to ensure there are no incorrect configurations
        ```
        sudo nginx -t
        ```

### 5. Create Domain Name and Install SSL Certificate

To optimize user experience and security, we'll implement a domain name and SSL certificate. This will allow users to access the application via a user-friendly URL and a secure HTTPS connection. To simplify the SSL certificate process, we will utilize Certbot, a powerful automation tool

```sudo apt install certbot python3-certbot-nginx```

With our Nginx configuration, we'll automate the SSL certificate process with Certbot

```sudo certbot --nginx -d "pacmail.groups.id"```

With the SSL certificate generated and installed, the application is now securely accessible via HTTPS. You can access it at [https://pacmail.groups.id](https://pacmail.groups.id)


![Pacmail HTTPS](https://github.com/hudiyaresa/PacMail/blob/main/demo_screenshots/pacmail%20landingpage.png)
![Pacmail Test](https://github.com/hudiyaresa/PacMail/blob/main/demo_screenshots/pacmail%20test.png)


### Conclusion

By implementing a robust CI/CD pipeline, PacMail achieves several key benefits: increased deployment frequency, improved code quality, reduced deployment time, enhanced application reliability, and greater scalability. This approach ensures that PacMail can deliver a reliable, scalable, and accessible email solution to meet the evolving needs of the company and its users.

Following the outlined steps simplifies automated deployment for PacMail, ensuring smooth updates and public availability. For any additional configurations, refer to the specific documentation for Docker, Nginx, and GitHub Actions.


### References
- [Docker Documentation](https://docs.docker.com)
- [Nginx Reverse Proxy Setup](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Benefits of CI/CD](https://www.redhat.com/en/topics/devops/what-is-ci-cd)
