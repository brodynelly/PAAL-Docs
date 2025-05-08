# High-Level System Architecture

## Overview

The PAAL (Pig Activity and Analytics Logger) system is designed to monitor, track, and analyze pig behavior and health metrics in agricultural settings. The system collects data from various sensors, processes it, and presents insights through a web-based dashboard. This document provides a high-level overview of the system architecture, its major components, and how they interact.

## System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Client Devices                             │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌───────┐ │
│  │  Web Browser │   │ Mobile Device│   │ Tablet       │   │ ...   │ │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘   └───┬───┘ │
└─────────┼────────────────┬─┼────────────────┬─┼─────────────┬─┼─────┘
          │                │ │                │ │             │ │
          │                │ │                │ │             │ │
          ▼                ▼ ▼                ▼ ▼             ▼ ▼
┌─────────────────────────────────────────────────────────────────────┐
│                              NGINX Proxy                             │
└─────────────────────────────┬─────────────────────────┬─────────────┘
                              │                         │
                              │                         │
          ┌──────────────────┐│                         │┌─────────────────┐
          │                  ││                         ││                 │
          ▼                  ▼│                         │▼                 │
┌───────────────────┐ ┌──────┴─────────────┐   ┌────────┴────────┐ ┌─────────────────┐
│  Next.js Frontend │ │ Express.js Backend │   │ Socket.IO Server │ │ Static Assets   │
│  (React SPA)      │ │ (REST API)         │   │ (Real-time)      │ │ (Images, etc.)  │
└─────────┬─────────┘ └──────┬─────────────┘   └────────┬────────┘ └─────────────────┘
          │                  │                          │
          │                  │                          │
          │                  ▼                          │
          │         ┌────────────────────┐              │
          │         │ Business Logic     │              │
          │         │ - Authentication   │              │
          │         │ - Data Processing  │              │
          │         │ - Analytics        │              │
          │         └──────────┬─────────┘              │
          │                    │                        │
          │                    │                        │
          │                    ▼                        │
          │         ┌────────────────────┐              │
          └────────►│ Data Access Layer  │◄─────────────┘
                    └──────────┬─────────┘
                               │
                               │
                               ▼
                    ┌────────────────────┐
                    │     MongoDB        │
                    │   (Database)       │
                    └────────────────────┘
                               ▲
                               │
                               │
┌─────────────────────────────┴─────────────────────────────────────┐
│                        Data Collection                             │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌───────┐ │
│  │ Posture      │   │ Temperature  │   │ Health       │   │ Other │ │
│  │ Sensors      │   │ Sensors      │   │ Monitors     │   │Sensors│ │
│  └──────────────┘   └──────────────┘   └──────────────┘   └───────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

## Major Components

### 1. Client Layer

The client layer consists of various devices that users use to access the system:

- **Web Browsers**: The primary means of accessing the system through desktop or laptop computers
- **Mobile Devices**: Smartphones running the web application
- **Tablets**: Larger mobile devices often used in field settings

### 2. NGINX Proxy

NGINX serves as a reverse proxy and load balancer, handling incoming requests and routing them to the appropriate backend services:

- Routes API requests to the Express.js backend
- Routes WebSocket connections to the Socket.IO server
- Serves the Next.js frontend application
- Handles SSL termination and basic security measures

### 3. Frontend Application

The frontend is built with Next.js and React:

- **Single Page Application (SPA)**: Provides a responsive and interactive user interface
- **Component-Based Architecture**: Modular UI components for different parts of the application
- **Client-Side State Management**: Manages application state using React Context and React Query
- **Responsive Design**: Adapts to different screen sizes and devices

### 4. Backend Services

The backend consists of several services:

#### Express.js REST API

- Handles HTTP requests from the frontend
- Implements RESTful endpoints for CRUD operations
- Processes and validates user input
- Enforces authentication and authorization

#### Socket.IO Server

- Provides real-time communication capabilities
- Pushes updates to connected clients
- Enables live monitoring of pig data
- Broadcasts system events and notifications

### 5. Business Logic Layer

The business logic layer contains the core application logic:

- **Authentication and Authorization**: User management, role-based access control
- **Data Processing**: Transforming raw data into useful information
- **Analytics**: Generating insights and statistics from collected data
- **Validation**: Ensuring data integrity and consistency

### 6. Data Access Layer

The data access layer handles interactions with the database:

- **Mongoose ODM**: Object Data Modeling for MongoDB
- **CRUD Operations**: Create, Read, Update, Delete operations on data
- **Query Building**: Constructing efficient database queries
- **Data Validation**: Schema-level validation of data

### 7. Database

MongoDB serves as the primary data store:

- **Document-Oriented**: Stores data in flexible, JSON-like documents
- **Collections**: Organizes documents into collections (similar to tables)
- **Indexes**: Optimizes query performance
- **Change Streams**: Enables real-time monitoring of data changes

### 8. Data Collection

The data collection layer consists of various sensors and devices that collect data from pigs:

- **Posture Sensors**: Track pig posture (standing, sitting, lying)
- **Temperature Sensors**: Monitor environmental conditions
- **Health Monitors**: Track vital signs and health indicators
- **Other Sensors**: Additional data collection devices

## Request Flow

### Typical User Request Flow

1. **Client Initiates Request**: User interacts with the frontend application
2. **NGINX Routing**: Request is received by NGINX and routed to the appropriate service
3. **Frontend Processing**: For client-side operations, the frontend handles the request directly
4. **API Request**: For server-side operations, the frontend sends a request to the backend API
5. **Authentication**: Backend verifies the user's identity and permissions
6. **Business Logic**: Request is processed according to business rules
7. **Data Access**: Backend interacts with the database as needed
8. **Response Generation**: Backend generates a response
9. **Client Update**: Frontend updates the UI based on the response
10. **Real-time Updates**: If applicable, Socket.IO pushes updates to connected clients

### Data Collection Flow

1. **Sensor Data Collection**: Sensors collect data from pigs and their environment
2. **Data Transmission**: Data is transmitted to the backend API
3. **Data Validation**: Backend validates the incoming data
4. **Data Storage**: Valid data is stored in the database
5. **Real-time Processing**: Data is processed in real-time for immediate insights
6. **Notification**: If significant events are detected, notifications are generated
7. **Dashboard Update**: Connected clients receive updates via Socket.IO

## Authentication and Authorization

### Authentication Flow

1. **User Login**: User provides credentials (email/password)
2. **Credential Verification**: Backend verifies credentials against stored data
3. **Token Generation**: Upon successful verification, a JWT token is generated
4. **Token Storage**: Token is stored in the client (localStorage)
5. **Authenticated Requests**: Subsequent requests include the token in the Authorization header
6. **Token Verification**: Backend verifies the token for each protected request

### Authorization Model

The system implements role-based access control (RBAC) with the following roles:

- **Admin**: Full access to all system features and data
- **Manager**: Access to manage farms, barns, stalls, and view all data
- **Farmer**: Access to specific assigned farms and their data
- **Viewer**: Read-only access to specific data

## Data Flow

### Data Collection and Storage

1. **Sensor Data Collection**: Sensors collect raw data
2. **Data Transmission**: Data is sent to the backend API
3. **Data Preprocessing**: Raw data is cleaned and normalized
4. **Data Storage**: Processed data is stored in the database
5. **Aggregation**: Data is aggregated for analytics and reporting

### Data Retrieval and Presentation

1. **Data Request**: Frontend requests data from the backend
2. **Query Execution**: Backend queries the database
3. **Data Transformation**: Raw data is transformed into a client-friendly format
4. **Data Transmission**: Transformed data is sent to the frontend
5. **Data Visualization**: Frontend renders the data in charts, tables, etc.

## Scalability and Performance

### Horizontal Scalability

The system is designed to scale horizontally:

- **Stateless Backend**: Backend services can be scaled by adding more instances
- **Load Balancing**: NGINX distributes traffic across multiple backend instances
- **Database Scaling**: MongoDB can be scaled through sharding and replication

### Performance Optimization

Several strategies are employed to optimize performance:

- **Caching**: Frequently accessed data is cached
- **Indexing**: Database indexes improve query performance
- **Pagination**: Large result sets are paginated
- **Lazy Loading**: Data is loaded only when needed
- **Code Splitting**: Frontend code is split into smaller chunks

## Security Measures

The system implements several security measures:

- **Authentication**: JWT-based authentication
- **Authorization**: Role-based access control
- **Input Validation**: All user input is validated
- **HTTPS**: Encrypted communication
- **Rate Limiting**: Protection against brute force attacks
- **CORS**: Controlled cross-origin resource sharing
- **Password Hashing**: Secure password storage using bcrypt

## Monitoring and Logging

The system includes monitoring and logging capabilities:

- **Activity Logging**: User actions are logged for audit purposes
- **Error Logging**: Errors are logged for troubleshooting
- **Performance Monitoring**: System performance is monitored
- **Health Checks**: Regular checks ensure system components are functioning properly

## Deployment Architecture

The system is deployed using Docker containers:

- **Containerization**: Each component runs in a Docker container
- **Docker Compose**: Containers are orchestrated using Docker Compose
- **Environment Configuration**: Environment-specific configuration is managed through environment variables
- **CI/CD Pipeline**: Automated build and deployment process

## Conclusion

The PAAL system architecture is designed to be scalable, maintainable, and secure. It follows modern best practices for web application development, with a clear separation of concerns and modular components. The architecture supports the system's requirements for real-time data processing, analytics, and user interaction, while providing a foundation for future enhancements and scaling.
