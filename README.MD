
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

---

## Overview
PacMail is a web-based email application with features for user login, sign-up, inbox view, and email sending. This repository automates the deployment process using Docker, Docker Compose, and GitHub Actions CI/CD pipeline.

## Prerequisites
- **Server**: A public-accessible server (e.g., VPS) for hosting the application.
- **Domain**: A domain name for public access.
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
version: '3'
services:
  frontend:
    image: node:latest
    container_name: frontend-container
    working_dir: /frontend
    volumes:
      - ./frontend:/frontend
    depends_on:
      - backend
      - database
    restart: always
    network_mode: "host"
    command: npm start
  
  backend:
    build:
      context: ./backend
    container_name: backend-container
    hostname: backend
    volumes:
      - ./backend:/app
    depends_on:
      - database
    restart: always
    network_mode: "host"
  
  database:
    image: postgres:13
    container_name: database-container
    hostname: database
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: always
    network_mode: "host"

volumes:
  postgres_data:
```

### 3. CI/CD Pipeline Configuration

#### Branch Protection
1. Protect the main branch in the repository settings.
2. Only allow merges through approved pull requests.

#### Continuous Integration Workflow
Create a GitHub Actions workflow file (`.github/workflows/ci.yml`) for the CI process:
```yaml
name: CI Workflow
on:
  pull_request:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v2

      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.10'

      - name: Install Backend Requirements
        run: |
          cd backend
          pip install -r requirements.txt

  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v2

      - name: Install Testing Requirements
        run: |
          pip install -r testing/requirements.txt

      - name: Run Tests
        run: |
          python3 testing/test_signup.py
```

#### Continuous Delivery & Deployment Workflow
Create a GitHub Actions workflow file (`.github/workflows/cd.yml`) for the CD process:
```yaml
name: CD Workflow
on:
  push:
    branches:
      - main

jobs:
  delivery:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v2

      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and Push Docker Images
        run: |
          docker-compose build
          docker-compose push

  deployment:
    runs-on: self-hosted
    steps:
      - name: SSH into Server and Deploy
        run: |
          ssh -o StrictHostKeyChecking=no user@server 'docker-compose pull && docker-compose up -d'
```

### 4. Domain and Nginx Configuration

1. **Set Up Domain**: Register a domain name for accessing the application.
2. **Nginx Reverse Proxy Configuration**:
   - Create an Nginx configuration to redirect traffic from port 80 to the frontend container on localhost:3000.
   - Example `nginx.conf`:
     ```nginx
     server {
         listen 80;
         server_name your-domain.com;

         location / {
             proxy_pass http://localhost:3000;
             proxy_set_header Host $host;
             proxy_set_header X-Real-IP $remote_addr;
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
             proxy_set_header X-Forwarded-Proto $scheme;
         }
     }
     ```

## Conclusion
Following these steps will set up automated deployment for PacMail, ensuring smooth updates and public availability. For any additional configurations, refer to the specific documentation for Docker, Nginx, and GitHub Actions.
```
