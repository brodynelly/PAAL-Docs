# PAAL Monitoring System

PAAL (Precision Agriculture and Animal Livestock) Monitoring System is a comprehensive web application designed for agricultural operations management. This system provides real-time monitoring, data analytics, and management tools for farms, barns, stalls, and livestock.

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [System Requirements](#system-requirements)
- [Installation Guide](#installation-guide)
  - [Prerequisites](#prerequisites)
  - [Step-by-Step Installation](#step-by-step-installation)
- [Usage Guide](#usage-guide)
  - [Logging In](#logging-in)
  - [Dashboard](#dashboard)
  - [Farm Management](#farm-management)
  - [System Settings](#system-settings)
- [Troubleshooting](#troubleshooting)
- [Maintenance](#maintenance)
- [Support](#support)
- [Developer Documentation](#developer-documentation)

## Overview

PAAL Monitoring System is an enterprise-grade application that helps farm owners and managers monitor and manage their agricultural operations efficiently. The system provides a user-friendly interface for tracking livestock health, farm conditions, and operational metrics.

## Tech Stack

### Frontend

*   Next.js 14
*   React
*   TailwindCSS
*   Recharts for data visualization
*   Radix UI for accessible components

### Backend

*   Node.js
*   Express
*   MongoDB (configured as a replica set for high availability & authentication)
*   Mongoose

## System Requirements

- **Operating System**: Windows 10/11, macOS 10.15+, or Linux (Ubuntu 20.04+ recommended)
- **Processor**: 2 GHz dual-core processor or better
- **Memory**: 4 GB RAM minimum (8 GB recommended)
- **Storage**: 10 GB available space
- **Internet Connection**: Broadband internet connection
- **Software**: Docker and Docker Compose

## Prerequisites

*   Node.js 18.x or higher
*   MongoDB (no need for a local install if using Docker)
*   npm or pnpm
*   Docker and Docker Compose

**Note:** This repository includes a Docker Compose setup for MongoDB configured as a replica set (even in a single-node setup) along with a key file for secure internal authentication. It also includes automation scripts for backing up and restoring the database across machines, ensuring consistency without exposing sensitive production data.

Getting Started
---------------

### 1\. Clone the Repository

    git clone https://github.com/brodynelly/paal.git
    cd paal

### 2\. Create and Switch to Your Specified `localDev` Branch

    git checkout -b <user_name>localDev

This command creates a new branch named `localDev` and switches you to it. Remember to replace `<user_name>` with your own git username. All changes you make now will be isolated from the main branch.

### 3\. Install Local Dependancies with PNPM

    npm install
    pnpm install


this command allows you to install the local depos for the web application. This is crucial in actually running the application and not running into compile time errors with docker


### 4\. Set Up Environment Variables

Create a `.env` file in the root directory with the following content (adjust as needed):

    # MongoDB Initialization Variables
    MONGO_INITDB_ROOT_USERNAME=PAAL
    MONGO_INITDB_ROOT_PASSWORD=PAAL
    MONGO_INITDB_DATABASE=paalab
    
    # MongoDB Connection Settings
    DATABASE_HOST=mongo-c
    DATABASE_PORT=27017
    DATABASE_COLLECTION=paalab
    
    # Backend & Server Variables
    SERVER_HOST=server-c
    SERVER_PORT=5005
    PORT=3000
    
    # Clerk URLs
    NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
    NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up
    NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL=/overview
    NEXT_PUBLIC_CLERK_AFTER_SIGN_UP_URL=/overview
    
    # API URL for React App
    REACT_APP_API_URL=http://localhost:8080
    NEXT_PUBLIC_API_URL=http://localhost:8080
    NEXT_PUBLIC_BASE_URL=http://localhost:8080
    
    # Authentication
    JWT_SECRET=your_jwt_secret_key_change_this_in_production
    CLERK_WEBHOOK_SECRET=your_clerk_webhook_secret
    
    # Environment
    NODE_ENV=development


Docker Compose automatically loads a file named `.env` from the root directory when you run `docker compose up`.


### 4.1 \. Creating the Authentication SSL Key for `security.keyFile`

To enable **internal authentication** for MongoDB using a **key file**, follow these steps:

#### Step 1: Create the Directory for the Key File

Ensure that the directory structure exists on your host machine:

    mkdir -p ./database/sslkey

This will create the `sslkey` directory inside `database/`.

### Step 2: Generate the Key File

Run the following command to create a **random key** and save it to `security.keyFile`:

    openssl rand -base64 756 > ./database/sslkey/security.keyFile

This generates a **756-byte** base64-encoded key (recommended by MongoDB) and saves it to `security.keyFile`.

### Step 3: Set Proper Permissions


MongoDB requires that the key file is **only readable** by the owner (`600` permissions):

    chmod 600 ./database/sslkey/security.keyFile

This ensures that only the owner can read and write the key file.

### Step 4: Verify the Key File

You can check the contents of the key file to confirm it was generated correctly:

    cat ./database/sslkey/security.keyFile

You should see a long base64-encoded string.


### 5\. Starting the Mongo Replicaset

### 1\. Start the Mongo Docker

    docker compose up --build mongo -d

### 2\. Initiate the MongoDB Replica Set

#### 2.1\.Once MongoDB is running, connect to it with authentication from the admin database (this is a seperate database from PAAL that sets the user Auth) :

    mongosh "mongodb://PAAL:PAAL@127.0.0.1:27017/admin?authSource=admin"

or if mongosh isn't installed on machine. Load through docker container using command:

    docker exec -it mongo-c mongosh -u PAAL -p PAAL --authenticationDatabase admin

#### 2.2\.Then, initiate the replica set:

    rs.initiate({
      _id: "rs0",
      members: [
        { _id: 0, host: "mongo-c:27017" }
      ]
    })

#### 2.3\. (OPTIONAL) Verify the configuration:

    rs.status()

## Usage Guide

### Logging In

1. On the login page, use one of the following test accounts:
   - Admin: `admin@test.com` / `admin123`
   - Farmer: `farmer@test.com` / `farmer123`

2. Click the "Sign In" button

### Dashboard

The dashboard provides an overview of your farm operations:

- **Admin Dashboard**: Shows system information, user statistics, and farm metrics
- **Farmer Dashboard**: Shows farm-specific information and livestock metrics

### Farm Management

1. Navigate to the "Farms" section in the sidebar
2. View the list of farms
3. Click on a farm to view details
4. Use the "Edit" button to modify farm information

### System Settings

**Admin users only:**

1. Navigate to the "System" section in the sidebar
2. Access various system settings:
   - **Backup & Restore**: Create and restore system backups
   - **Maintenance**: Perform system maintenance tasks
   - **System Logs**: View system activity logs

## Starting the Application

### 1\. closing the mongo Docker Image after Initiating rs0 image

    docker compose down

this will close the docker image that you created the rs0 image in

### 2\. Starting the web service

    docker compose up --build -d

this with start the entire ecosystem for this design where you can access everything.

### 3\. Start the Development Servers

Make sure your frontend is running on port `8080` and your backend on port `8080/api`. Access the frontend via `http://localhost:8080/`.

### 4\. Verify Connectivity

*   **Test Application:** Run the application to ensure it connects to MongoDB correctly.
*   **Check Containers:** Use `docker ps` to verify that all containers are running.
*   **Logs:** For troubleshooting, view logs with:

        docker-compose logs mongo


### 6\. Seed the Database (Optional)

Now, when you first load the DB there will be no data. Lets populate it with some prop data, run:

    docker exec -it server-c /bin/bash
    npm run seed

*We first load into the Docker CLI to execute our seed command, because the connection to the database is over the docker-network*

Connecting to Database Via Compass
---------------------

## Using mongodb Compass.

### 1\. Open compass connection window and enter this connection string

    mongodb://PAAL:PAAL@localhost:27017/?directConnection=true

This will allow you to connect directly connect to the database via compass to see visual changes and updated to the database.


## Troubleshooting

### Common Issues

#### Application Not Starting

1. Ensure Docker is running
2. Check if the ports are available (8080 for the frontend, 27017 for MongoDB)
3. Check the logs:
   ```
   docker-compose logs
   ```

#### Login Issues

1. Ensure you're using the correct credentials
2. Check if the database is properly initialized
3. Run the fix-database script:
   ```
   ./fix-database.sh
   ```

#### Database Connection Issues

1. Check if the MongoDB container is running:
   ```
   docker-compose ps
   ```
2. Ensure the database credentials in the `.env` file match those in the application

### Restarting the Application

If you encounter issues, try restarting the application:

```bash
docker-compose down
docker-compose up -d
```

## Backup & Restore

### Backup Script Overview

We include an automated script (`backup_to_github.sh`) that runs `mongodump` inside the MongoDB container to export the database, archive it, and commit the backup to GitHub. This helps ensure that all developers work with the same data across local machines.

#### How to Use the Backup Script:

1.  Make the script executable:

        chmod +x backup_to_github.sh

2.  Run the script:

        ./backup_to_github.sh


### Restoring Backups

*   Pull the latest commit from the `localDev` branch.
*   If necessary, run `mongorestore` (or a provided script) to restore the database from the backup.

*    TODO:
*        I will bind the docker `mongodb` image to a volume with backup data stored in this github repo for percistant data flow.

Project Structure
-----------------

    paal-test/
    ├── server/                   # Backend server files
    │   ├── models/               # MongoDB models
    │   ├── routes/               # API routes
    │   └── index.js              # Server entry point
    ├── src/                      # Frontend source code
    │   ├── app/                 # Next.js pages and routes
    │   ├── components/          # React components
    │   ├── lib/                 # Utility functions
    │   └── types/               # TypeScript type definitions
    ├── docker-compose.yml        # Docker Compose configuration
    ├── .env                      # Environment variables for Docker Compose
    ├── Dockerfile.frontend       # Dockerfile for frontend build
    ├── backup_to_github.sh       # Database backup automation script
    └── README.md                 # This file

Snippet into Docker Compose
---------------------------

### 4\. Docker Compose Setup for MongoDB with Replica Set

Our Docker Compose file sets up MongoDB with a replica set (`rs0`) and internal authentication using a key file.

*   **Replica Set:** Even a single node runs as a replica set for future scalability.
*   **Key File:** The key file is mounted from `./database/sslkey/security.keyFile` into the container at `/etc/secrets/security.keyFile`.
*   **Authentication:** The `--auth` flag is enabled, and the admin user is automatically created from environment variables.

Snippet from `docker-compose.yml`:

    version: "3.8"
    services:
      mongo:
        image: mongo:latest
        container_name: ${DATABASE_HOST}
        restart: always
        env_file: .env
        environment:
          MONGO_INITDB_ROOT_USERNAME: ${MONGO_INITDB_ROOT_USERNAME}
          MONGO_INITDB_ROOT_PASSWORD: ${MONGO_INITDB_ROOT_PASSWORD}
          MONGO_INITDB_DATABASE: ${DATABASE_COLLECTION}
        ports:
          - ${DATABASE_PORT}:27017
        networks:
          - app-net
        volumes:
          - database-v:/data/db
          - ./database/sslkey/security.keyFile:/etc/secrets/security.keyFile:rw
        command: [ "mongod", "--replSet", "rs0", "--auth", "--keyFile", "/etc/secrets/security.keyFile" ]


After starting the container, connect to MongoDB and run the replica set initiation command (see RUNNING THE APPLICATION SECTION).

### 5\. Frontend and Backend Services

The backend service runs the Node.js/Express server, the frontend service runs Next.js, and the Mongo service hosts the database. Each container operates in an isolated network environment. If you were to check your router, each Docker container would register as its own device. These containers communicate through a Docker network bridge, which provides a secure link between them. This setup is particularly useful in production, as it helps restrict communication gateways, reducing exposure to external threats and minimizing potential internal bugs.

Now, let's examine what makes our Docker environment function. Everything is defined as services within our docker-compose.yml file. Each service is started based on its Dockerfile, which provides specific instructions for launching the server. And each service is operated with our 'DockerFile' identifier for specific instructions to start our server. There are two sets of DockerFiles in the main directory of the project. one is for production and has the suffix `.production`, and the other is for development with the suffix `.development`. For now, we will focus on the development build, which uses the Next.js development command that enables live reloading!!

**Example Docker Compose entry for Frontend (development override):**

    services:
      frontend:
        build:
          context: .
          dockerfile: Dockerfile.frontend
        container_name: frontend
        environment:
          - NODE_ENV=development
        ports:
          - "3000:3000"
        volumes:
          - ./:/usr/src/app
          - /usr/src/app/node_modules
        command: ["npm", "run", "dev"]

You can use a separate override file (e.g., `docker-compose.override.yml`) to differentiate between production and development setups.


## Support

For additional support:

- Check the documentation in the `docs` folder
- Contact system administrator
- Email support at: support@paalmonitoring.com

## Developer Documentation

### How It All Works

*   **Docker & Environment Variables:** Docker Compose automatically loads your `.env` file to replace placeholders in the configuration. This ensures consistency across environments and levels of production. Simply by changing our env variables, we can have a production build in less than a week. This also makes it easy to adjust credentials, ports, and other settings.
*   **MongoDB Replica Set:** MongoDB is configured to run as a replica set even if it's a single node. A key file is mounted for secure inter-node authentication, and the container automatically creates a root user based on your `.env` variables. After container startup, you manually initiate the replica set using `rs.initiate()`.
*   **Development Workflow:** The frontend service is set up for development with live reloading by mounting your source code directory. The backend service connects to MongoDB using a connection string that includes the replica set and authentication parameters.
*   **Backup and Restore:** The backup script leverages MongoDB’s `mongodump` and `mongorestore` commands to maintain a consistent dataset across local machines, ensuring every developer works with the same data.
*   **Version Control:** Always commit your changes to your local development branch (`localDev`) and avoid pushing directly to `main` or the default
