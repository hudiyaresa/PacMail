
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
    environment:
      - REACT_APP_API_URL=${REACT_APP_API_URL}
  
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
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
  
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

#### Continuous Integration Workflow
Create a GitHub Actions workflow file (`.github/workflows/ci.yml`) for the CI process:
```yaml
name: Dev Testing 🔎 

on:
  pull_request:
    branches: [ "main" ]

jobs:
  build-testing:
    name: Build and Testing
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2
      
      - name: Create .env file
        run: |
          echo "POSTGRES_USER=${{ secrets.POSTGRES_USER }}" > .env
          echo "POSTGRES_PASSWORD=${{ secrets.POSTGRES_PASSWORD }}" >> .env
          echo "POSTGRES_DB=${{ secrets.POSTGRES_DB }}" >> .env
          echo "REACT_APP_API_URL: ${{ secrets.REACT_APP_API_URL }}" >> .env 

      - name: Build and Run Container
        run: |
          sudo docker compose up database-pacmail backend-pacmail frontend-pacmail --build --detach

      - name: Hit Endpoint
        run: |
          sleep 10
          curl ${{ secrets.PROD_URL_1 }}
          curl ${{ secrets.PROD_URL_2 }}
          
      - name: Testing
        run: |
          pip install -r testing/requirements.txt
          python3 testing/test_signup.py
```

#### Continuous Delivery & Deployment Workflow
Create a GitHub Actions workflow file (`.github/workflows/cd.yml`) for the CD process:
```yaml
name: Deploy Production 🚀

on:
  release:
    types:
      - published
      - edited

jobs:
  deploy-production:
    name: Deploy to production server 🚀
    runs-on: ubuntu-latest

    steps:
      - name: Execute deployment command
        uses: appleboy/ssh-action@v1.0.3
        env:
          APP_PATH_PROD: ${{ secrets.APP_PATH_PROD }}
          GIT_URL: ${{ secrets.GIT_URL }}
          REACT_APP_API_URL: ${{ secrets.REACT_APP_API_URL }}
          POSTGRES_USER: ${{ secrets.POSTGRES_USER }}
          POSTGRES_PASSWORD: ${{ secrets.POSTGRES_PASSWORD }}
          POSTGRES_DB: ${{ secrets.POSTGRES_DB }}
          POSTGRES_HOST: ${{ secrets.POSTGRES_HOST }}
          DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
          DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
          APP_NAME: ${{ vars.APP_NAME }}
          APP_TAG: ${{ github.event.release.tag_name }}

        with:
          host: ${{ secrets.SSH_HOST_PROD }}
          username: ${{ secrets.SSH_USER_NAME_PROD }}
          key: ${{ secrets.SSH_PRIVATE_KEY_PROD }}
          envs: APP_PATH_PROD, GIT_URL, REACT_APP_API_URL, POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB, POSTGRES_HOST, DOCKERHUB_USERNAME, DOCKERHUB_TOKEN, APP_NAME, APP_TAG
          script: |

            sudo docker login -u $DOCKERHUB_USERNAME -p $DOCKERHUB_TOKEN

            if [[ -d "${APP_PATH_PROD}" ]]; then 
              cd $APP_PATH_PROD
              sudo docker compose down
              git pull --rebase
            else
              ssh-keyscan github.com > ~/.ssh/known_hosts
              git clone $GIT_URL $APP_PATH_PROD
              cd $APP_PATH_PROD
            fi

            # If there are any envars update
            echo "POSTGRES_USER=$POSTGRES_USER" > .env
            echo "POSTGRES_PASSWORD=$POSTGRES_PASSWORD" >> .env
            echo "POSTGRES_DB=$POSTGRES_DB" >> .env
            echo "POSTGRES_HOST=$POSTGRES_HOST" >> .env
            echo "POSTGRES_PORT=$POSTGRES_PORT" >> .env
            echo "APP_IMAGE=${DOCKERHUB_USERNAME}/${APP_NAME}" >> .env
            echo "APP_TAG=$APP_TAG" >> .env

            # Run app
            sudo docker compose up database-pacmail backend-pacmail frontend-pacmail --build --detach

      - name: Hit Endpoint
        run: |
          sleep 10
          curl ${{ vars.PROD_URL_1 }}
```

### 4. Domain and Nginx Configuration

1. **Set Up Domain**: Register a domain name for accessing the application.
2. **Nginx Reverse Proxy Configuration**:
   - Create an Nginx configuration to redirect traffic from port 80 to the frontend container on localhost:3000.
   - Example `nginx.conf`:
     ```nginx
     server {
         listen 80;
         server_name pacmail.groups.id;

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
