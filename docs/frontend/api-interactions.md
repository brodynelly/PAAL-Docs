# API Interactions

## Overview

The frontend interacts with the backend through a RESTful API and WebSocket connections. This document details the API endpoints used, request/response formats, and the purpose of each interaction.

## API Client Configuration

The application uses Axios for HTTP requests, configured in `src/lib/axios.ts`:

```typescript
import axios from 'axios';

const api = axios.create({
  baseURL: `${process.env.NEXT_PUBLIC_API_URL}/api`,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add a request interceptor to include auth token
api.interceptors.request.use((config) => {
  // Only add token for browser environment
  if (typeof window !== 'undefined') {
    // Get token from localStorage
    const token = localStorage.getItem('token');

    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
  }

  return config;
});

// Add a response interceptor to handle authentication errors
api.interceptors.response.use(
  (response) => {
    return response;
  },
  (error) => {
    // If the error is due to authentication (401) or authorization (403)
    if (error.response && (error.response.status === 401 || error.response.status === 403)) {
      // Redirect to login page if not already there
      if (typeof window !== 'undefined' && !window.location.pathname.includes('/login')) {
        localStorage.removeItem('token');
        localStorage.removeItem('user');
        window.location.href = '/login';
      }
    }
    return Promise.reject(error);
  }
);

export default api;
```

## Authentication Endpoints

### Login

- **Endpoint**: `POST /api/auth/login`
- **Purpose**: Authenticate a user and receive an access token
- **Request**:
  ```json
  {
    "email": "user@example.com",
    "password": "password123"
  }
  ```
- **Response**:
  ```json
  {
    "token": "jwt-token-string",
    "user": {
      "id": "user-id",
      "email": "user@example.com",
      "firstName": "John",
      "lastName": "Doe",
      "role": "admin"
    }
  }
  ```

### Verify Token

- **Endpoint**: `GET /api/auth/token`
- **Purpose**: Verify if the current token is valid and get user information
- **Headers**: `Authorization: Bearer {token}`
- **Response**:
  ```json
  {
    "user": {
      "id": "user-id",
      "email": "user@example.com",
      "firstName": "John",
      "lastName": "Doe",
      "role": "admin",
      "assignedFarm": {
        "id": "farm-id",
        "name": "Farm Name"
      }
    },
    "token": "jwt-token-string"
  }
  ```

## Pig Management Endpoints

### Get All Pigs

- **Endpoint**: `GET /api/pigs`
- **Purpose**: Retrieve a list of all pigs in the system
- **Headers**: `Authorization: Bearer {token}`
- **Response**:
  ```json
  [
    {
      "owner": "PIG-001",
      "status": "healthy",
      "costs": "12",
      "region": "Stall A",
      "stability": 0.8,
      "lastEdited": "01/01/2023, 12:00",
      "breed": "Yorkshire",
      "active": true
    },
    // More pigs...
  ]
  ```

### Get Pig Overview

- **Endpoint**: `GET /api/pigs/overview`
- **Purpose**: Get aggregated pig data for dashboard
- **Headers**: `Authorization: Bearer {token}`
- **Query Parameters**: 
  - `filter`: Optional filter (e.g., "breeding", "new", "healthy")
- **Response**:
  ```json
  {
    "success": true,
    "data": [
      {
        "name": "Breeding",
        "Farm 1": 10,
        "Farm 2": 5,
        "averageWeight": 120
      },
      // More categories...
    ],
    "stats": {
      "totalPigs": 150,
      "categories": ["Breeding", "Healthy", "At Risk"],
      "locations": ["Farm 1", "Farm 2"]
    }
  }
  ```

### Get Single Pig

- **Endpoint**: `GET /api/pigs/{id}`
- **Purpose**: Get detailed information about a specific pig
- **Headers**: `Authorization: Bearer {token}`
- **Response**:
  ```json
  {
    "pigId": 123,
    "tag": "PIG-123",
    "breed": "Yorkshire",
    "age": 12,
    "currentLocation": {
      "farmId": {
        "_id": "farm-id",
        "name": "Farm 1"
      },
      "barnId": {
        "_id": "barn-id",
        "name": "Barn A"
      },
      "stallId": {
        "_id": "stall-id",
        "name": "Stall 1"
      }
    },
    "active": true
  }
  ```

### Get Pig Posture Data

- **Endpoint**: `GET /api/pigs/{id}/posture/aggregated`
- **Purpose**: Get aggregated posture data for a specific pig
- **Headers**: `Authorization: Bearer {token}`
- **Query Parameters**:
  - `start`: Start date (YYYY-MM-DD)
  - `end`: End date (YYYY-MM-DD)
- **Response**:
  ```json
  [
    {
      "date": "2022-08-01",
      "standing": 45,
      "sitting": 30,
      "lying": 25
    },
    // More dates...
  ]
  ```

### Create Pig

- **Endpoint**: `POST /api/pigs`
- **Purpose**: Create a new pig in the system
- **Headers**: `Authorization: Bearer {token}`
- **Request**:
  ```json
  {
    "pigId": 123,
    "tag": "PIG-123",
    "breed": "Yorkshire",
    "age": 12,
    "currentLocation": {
      "farmId": "farm-id",
      "barnId": "barn-id",
      "stallId": "stall-id"
    }
  }
  ```
- **Response**:
  ```json
  {
    "success": true,
    "pig": {
      "pigId": 123,
      "tag": "PIG-123",
      "breed": "Yorkshire",
      "age": 12,
      "currentLocation": {
        "farmId": "farm-id",
        "barnId": "barn-id",
        "stallId": "stall-id"
      },
      "active": true,
      "_id": "pig-mongodb-id"
    }
  }
  ```

## Farm Management Endpoints

### Get All Farms

- **Endpoint**: `GET /api/farms`
- **Purpose**: Retrieve a list of all farms
- **Headers**: `Authorization: Bearer {token}`
- **Response**:
  ```json
  [
    {
      "_id": "farm-id-1",
      "name": "Farm 1",
      "location": "Location 1",
      "description": "Description for Farm 1",
      "isActive": true,
      "counts": {
        "barns": 5,
        "stalls": 20,
        "pigs": 100
      }
    },
    // More farms...
  ]
  ```

### Get Single Farm

- **Endpoint**: `GET /api/farms/{id}`
- **Purpose**: Get detailed information about a specific farm
- **Headers**: `Authorization: Bearer {token}`
- **Response**:
  ```json
  {
    "_id": "farm-id-1",
    "name": "Farm 1",
    "location": "Location 1",
    "description": "Description for Farm 1",
    "isActive": true,
    "counts": {
      "barns": 5,
      "stalls": 20,
      "pigs": 100,
      "devices": 15
    },
    "healthStatus": {
      "healthy": 80,
      "atRisk": 15,
      "critical": 5
    },
    "barns": [
      // Barn objects...
    ],
    "recentPigs": [
      // Recent pig objects...
    ],
    "devices": [
      // Device objects...
    ]
  }
  ```

## User Management Endpoints

### Get All Users

- **Endpoint**: `GET /api/users`
- **Purpose**: Retrieve a list of all users
- **Headers**: `Authorization: Bearer {token}`
- **Response**:
  ```json
  [
    {
      "_id": "user-id-1",
      "email": "user1@example.com",
      "firstName": "John",
      "lastName": "Doe",
      "role": "admin",
      "isActive": true,
      "lastLogin": "2023-01-01T12:00:00.000Z"
    },
    // More users...
  ]
  ```

### Create User

- **Endpoint**: `POST /api/auth/register`
- **Purpose**: Create a new user
- **Headers**: `Authorization: Bearer {token}`
- **Request**:
  ```json
  {
    "email": "newuser@example.com",
    "password": "password123",
    "firstName": "Jane",
    "lastName": "Smith",
    "role": "farmer",
    "permissions": ["read:farms", "write:pigs"],
    "restrictedFarms": ["farm-id-1"],
    "restrictedStalls": ["stall-id-1"]
  }
  ```
- **Response**:
  ```json
  {
    "_id": "new-user-id",
    "email": "newuser@example.com",
    "firstName": "Jane",
    "lastName": "Smith",
    "role": "farmer",
    "permissions": ["read:farms", "write:pigs"],
    "restrictedFarms": ["farm-id-1"],
    "restrictedStalls": ["stall-id-1"],
    "isActive": true
  }
  ```

## Real-time Data with Socket.IO

The application uses Socket.IO for real-time updates, configured in `src/lib/socket.ts`:

```typescript
import { io } from 'socket.io-client'

const SOCKET_URL = `http://localhost:8080`

export const socket = io(SOCKET_URL, {
  path: "/socket.io",
  transports: ['websocket'],
  autoConnect: true
})

export const subscribeToStats = (callback: (stats: any) => void) => {
  socket.on('stats_update', callback)
  return () => {
    socket.off('stats_update', callback)
  }
}

export const subscribeToDevices = (callback: (devices: any) => void) => {
  socket.on('devices_update', callback)
  return () => {
    socket.off('devices_update', callback)
  }
}

export const subscribeToPigs = (callback: (pigs: any) => void) => {
  socket.on('pigs_update', callback)
  return () => {
    socket.off('pigs_update', callback)
  }
}

export const subscribeToActivities = (callback: (activities: any) => void) => {
  socket.on('recent_activities', callback)
  return () => {
    socket.off('recent_activities', callback)
  }
}

export const subscribeToNewActivity = (callback: (activity: any) => void) => {
  socket.on('activity', callback)
  return () => {
    socket.off('activity', callback)
  }
}
```

### Socket Events

- **stats_update**: Receives updated system statistics
- **devices_update**: Receives updates about device status changes
- **pigs_update**: Receives updates about pig data changes
- **recent_activities**: Receives a list of recent system activities
- **activity**: Receives a single new activity event

## Error Handling

API errors are handled consistently throughout the application:

1. HTTP status codes are used to indicate the type of error
2. Error responses include a descriptive message
3. The API client's interceptor handles authentication errors automatically
4. Component-level error states display appropriate messages to users
