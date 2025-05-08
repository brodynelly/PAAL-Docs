# API Design and Documentation

## Overview

The PAAL system provides a comprehensive RESTful API that allows clients to interact with the system's resources. This document details the API design principles, authentication mechanisms, endpoint specifications, and implementation details to serve as a complete reference for developers working with the API.

## API Design Principles

The API follows these key design principles to ensure consistency, reliability, and maintainability:

### 1. RESTful Architecture

Resources are represented as URLs, and standard HTTP methods are used for operations:

- **GET**: Retrieve a resource or collection of resources
- **POST**: Create a new resource
- **PUT**: Update an existing resource (full update)
- **PATCH**: Partially update an existing resource
- **DELETE**: Remove a resource

Example resource hierarchy:
```
/farms                  # Collection of farms
/farms/{id}             # Specific farm
/farms/{id}/barns       # Collection of barns in a farm
/farms/{id}/barns/{id}  # Specific barn in a farm
```

### 2. JSON Format

All requests and responses use JSON format for data exchange:

- Request bodies must use `Content-Type: application/json`
- Responses are returned with `Content-Type: application/json`
- Date fields use ISO 8601 format (e.g., `2023-06-15T10:30:45.123Z`)
- Numeric IDs are represented as strings when used as resource identifiers

### 3. Consistent Response Structure

Responses follow a consistent structure for both success and error cases:

**Success Responses**:
```json
{
  "data": {
    // Resource data or collection
  },
  "meta": {
    // Metadata about the response (pagination, etc.)
  }
}
```

**Error Responses**:
```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": [
      // Optional additional error details
    ]
  }
}
```

### 4. Proper HTTP Status Codes

Appropriate HTTP status codes are used to indicate the result of operations:

| Code | Description | Usage |
|------|-------------|-------|
| 200 | OK | Successful GET, PUT, PATCH, or DELETE |
| 201 | Created | Successful resource creation (POST) |
| 204 | No Content | Successful operation with no response body |
| 400 | Bad Request | Invalid request format or parameters |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Authentication succeeded but insufficient permissions |
| 404 | Not Found | Resource not found |
| 409 | Conflict | Request conflicts with current state (e.g., duplicate) |
| 422 | Unprocessable Entity | Validation errors |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server-side error |

### 5. Authentication Required

Most endpoints require authentication via JWT:

- Tokens are obtained through the authentication endpoints
- Tokens must be included in the `Authorization` header
- Token expiration and refresh mechanisms are implemented
- Sensitive operations require re-authentication

### 6. Role-Based Access Control

Endpoints enforce appropriate authorization based on user roles:

| Role | Description | Access Level |
|------|-------------|-------------|
| Admin | System administrator | Full access to all resources |
| Manager | Farm manager | Access to assigned farms and all their resources |
| Farmer | Farm worker | Limited access to assigned farms |
| Viewer | Read-only user | Read-only access to assigned resources |

### 7. Pagination

Large result sets are paginated to improve performance:

- Default page size is 20 items
- Maximum page size is 100 items
- Pagination parameters: `page` (1-based) and `limit`
- Pagination metadata is included in the response

Example pagination metadata:
```json
"meta": {
  "pagination": {
    "page": 2,
    "limit": 20,
    "totalItems": 156,
    "totalPages": 8,
    "hasNextPage": true,
    "hasPrevPage": true
  }
}
```

### 8. Filtering and Sorting

Resources can be filtered and sorted using query parameters:

- Filtering: `?field=value` or `?field[operator]=value`
- Sorting: `?sort=field` (ascending) or `?sort=-field` (descending)
- Multiple filters and sorts can be combined
- Complex filtering uses JSON-encoded filter objects

Example filtering and sorting:
```
/api/pigs?breed=Yorkshire&age[gte]=12&sort=-updatedAt
```

### 9. Versioning

API versioning is supported through URL paths:

- Current version: v1 (implicit in base URL)
- Future versions will use explicit versioning: `/api/v2/...`
- Version changes are documented in the API changelog
- Multiple versions may be supported simultaneously during transitions

### 10. Rate Limiting

API endpoints are protected by rate limiting:

- Rate limits vary by endpoint sensitivity
- Rate limit headers are included in responses
- Exceeding rate limits results in 429 responses
- Backoff strategies are recommended for clients

Rate limit headers:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1623760500
```

### 11. Comprehensive Documentation

All API endpoints are thoroughly documented:

- OpenAPI/Swagger specifications
- Request/response examples
- Authentication requirements
- Error scenarios
- Rate limiting information

## Base URL

The base URL for all API endpoints is:

```
http://localhost:8080/api
```

In production environments, this would be replaced with the appropriate domain.

## Authentication

### Authentication Mechanism

The API uses JSON Web Tokens (JWT) for authentication, providing a stateless, secure method for authenticating API requests. This section details the complete authentication flow, token structure, and security considerations.

#### Authentication Flow

1. **Client Authentication**: Client submits credentials to the `/api/auth/login` endpoint
2. **Server Validation**: Server validates credentials against the database
3. **Token Generation**: Server generates a JWT token containing user information and permissions
4. **Token Response**: Server returns the token to the client
5. **Token Storage**: Client stores the token (typically in localStorage or secure cookie)
6. **Authenticated Requests**: Client includes the token in the `Authorization` header of subsequent requests
7. **Token Verification**: Server verifies the token signature and expiration
8. **Authorization**: Server checks user permissions for the requested resource
9. **Response**: Server returns the requested resource or an error

#### Token Structure

The JWT token consists of three parts: header, payload, and signature.

**Header**:
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload**:
```json
{
  "id": "60d21b4667d0d8992e610c85",
  "email": "user@example.com",
  "role": "admin",
  "permissions": ["read:farms", "write:pigs"],
  "iat": 1623760500,
  "exp": 1623846900
}
```

**Signature**:
The signature is created by encoding the header and payload with a secret key.

#### Token Usage

To access protected endpoints, clients must include the token in the `Authorization` header:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjYwZDIxYjQ2NjdkMGQ4OTkyZTYxMGM4NSIsImVtYWlsIjoidXNlckBleGFtcGxlLmNvbSIsInJvbGUiOiJhZG1pbiIsInBlcm1pc3Npb25zIjpbInJlYWQ6ZmFybXMiLCJ3cml0ZTpwaWdzIl0sImlhdCI6MTYyMzc2MDUwMCwiZXhwIjoxNjIzODQ2OTAwfQ.8yF7MJoo2RL4xMw2Sy2CGF6YHF8L2Q3CMTHfNXfBnLc
```

#### Token Expiration and Refresh

Tokens have a limited lifespan for security reasons:

- **Access Tokens**: Valid for 24 hours
- **Refresh Tokens**: Valid for 7 days (when implemented)

When a token expires, clients can:

1. Redirect the user to the login page
2. Use a refresh token to obtain a new access token (when implemented)
3. Implement silent refresh before token expiration

#### Token Refresh Flow

1. Client detects token expiration (or proactively refreshes before expiration)
2. Client sends refresh token to `/api/auth/refresh` endpoint
3. Server validates refresh token
4. Server issues new access token
5. Client updates stored access token

#### Security Considerations

The authentication system implements several security measures:

1. **HTTPS Only**: All authentication requests must use HTTPS
2. **Token Expiration**: Tokens have a limited lifespan
3. **Secure Storage**: Clients should store tokens securely
4. **CSRF Protection**: Implemented for browser-based clients
5. **Rate Limiting**: Login and token endpoints are rate-limited
6. **Token Revocation**: Administrators can revoke tokens
7. **Password Policies**: Strong password requirements are enforced
8. **Brute Force Protection**: Account lockout after multiple failed attempts

#### Implementation Details

The authentication system is implemented using the following components:

1. **jsonwebtoken**: Library for generating and verifying JWTs
2. **bcrypt**: Library for password hashing
3. **express-rate-limit**: Middleware for rate limiting
4. **helmet**: Middleware for security headers
5. **Custom Middleware**: For token verification and authorization

```javascript
// Example token verification middleware
const authenticateJWT = (req, res, next) => {
  const authHeader = req.headers.authorization;

  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({
      error: {
        code: 'UNAUTHORIZED',
        message: 'Authentication required'
      }
    });
  }

  const token = authHeader.split(' ')[1];

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    if (error.name === 'TokenExpiredError') {
      return res.status(401).json({
        error: {
          code: 'TOKEN_EXPIRED',
          message: 'Token has expired'
        }
      });
    }

    return res.status(401).json({
      error: {
        code: 'INVALID_TOKEN',
        message: 'Invalid token'
      }
    });
  }
};
```

### Authentication Endpoints

#### Login

Authenticates a user with email and password credentials and returns a JWT token.

- **URL**: `/api/auth/login`
- **Method**: `POST`
- **Auth Required**: No
- **Rate Limit**: 5 requests per minute per IP address
- **Content-Type**: `application/json`

##### Request Parameters

**Request Body**:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `email` | String | Yes | User's email address |
| `password` | String | Yes | User's password (min 8 characters) |
| `rememberMe` | Boolean | No | Whether to extend token expiration (default: false) |

**Example Request**:
```json
{
  "email": "user@example.com",
  "password": "password123",
  "rememberMe": true
}
```

##### Response Parameters

**Success Response (200 OK)**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `token` | String | JWT authentication token |
| `refreshToken` | String | JWT refresh token (only if rememberMe is true) |
| `expiresIn` | Number | Token expiration time in seconds |
| `user` | Object | User information |
| `user.id` | String | User's unique identifier |
| `user.email` | String | User's email address |
| `user.firstName` | String | User's first name |
| `user.lastName` | String | User's last name |
| `user.role` | String | User's role (admin, manager, farmer, viewer) |
| `user.lastLogin` | String | ISO 8601 timestamp of last login |
| `user.assignedFarm` | Object | Farm assigned to the user (if applicable) |
| `user.permissions` | Array | List of user permissions |

**Example Success Response**:
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjYwZDIxYjQ2NjdkMGQ4OTkyZTYxMGM4NSIsImVtYWlsIjoidXNlckBleGFtcGxlLmNvbSIsInJvbGUiOiJhZG1pbiIsInBlcm1pc3Npb25zIjpbInJlYWQ6ZmFybXMiLCJ3cml0ZTpwaWdzIl0sImlhdCI6MTYyMzc2MDUwMCwiZXhwIjoxNjIzODQ2OTAwfQ.8yF7MJoo2RL4xMw2Sy2CGF6YHF8L2Q3CMTHfNXfBnLc",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjYwZDIxYjQ2NjdkMGQ4OTkyZTYxMGM4NSIsInR5cGUiOiJyZWZyZXNoIiwiaWF0IjoxNjIzNzYwNTAwLCJleHAiOjE2MjQzNjUzMDB9.7UKIzA3sLVRaJaT_FIM9w7G_ZxH9bS1JV2cv7dxsQxE",
  "expiresIn": 86400,
  "user": {
    "id": "60d21b4667d0d8992e610c85",
    "email": "user@example.com",
    "firstName": "John",
    "lastName": "Doe",
    "role": "admin",
    "lastLogin": "2023-06-15T10:30:45.123Z",
    "assignedFarm": {
      "id": "60d21b4667d0d8992e610c87",
      "name": "Farm 1"
    },
    "permissions": [
      "read:farms",
      "write:farms",
      "read:barns",
      "write:barns",
      "read:stalls",
      "write:stalls",
      "read:pigs",
      "write:pigs",
      "read:users",
      "write:users"
    ]
  }
}
```

##### Error Responses

| Status Code | Error Code | Description | Possible Cause |
|-------------|------------|-------------|----------------|
| 400 | `MISSING_CREDENTIALS` | Missing email or password | Required fields not provided |
| 400 | `INVALID_EMAIL_FORMAT` | Invalid email format | Email doesn't match expected format |
| 401 | `INVALID_CREDENTIALS` | Invalid credentials | Email or password is incorrect |
| 403 | `ACCOUNT_INACTIVE` | Account is inactive | User account has been deactivated |
| 403 | `ACCOUNT_LOCKED` | Account is locked | Too many failed login attempts |
| 429 | `RATE_LIMIT_EXCEEDED` | Too many requests | Rate limit exceeded |

**Example Error Response (401 Unauthorized)**:
```json
{
  "error": {
    "code": "INVALID_CREDENTIALS",
    "message": "Invalid email or password",
    "details": {
      "remainingAttempts": 4
    }
  }
}
```

##### Implementation Notes

1. **Password Validation**:
   - Passwords are hashed using bcrypt before comparison
   - Failed login attempts are tracked and limited

2. **Token Generation**:
   - Tokens include user ID, role, and permissions
   - Standard tokens expire in 24 hours
   - "Remember me" tokens expire in 7 days

3. **Security Measures**:
   - Login attempts are rate-limited to prevent brute force attacks
   - Account is temporarily locked after 5 failed attempts
   - All login attempts are logged for security auditing

##### Code Example

```javascript
// routes/auth.js
router.post('/login', loginLimiter, async (req, res) => {
  try {
    const { email, password, rememberMe = false } = req.body;

    // Validate input
    if (!email || !password) {
      return res.status(400).json({
        error: {
          code: 'MISSING_CREDENTIALS',
          message: 'Email and password are required'
        }
      });
    }

    // Validate email format
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(email)) {
      return res.status(400).json({
        error: {
          code: 'INVALID_EMAIL_FORMAT',
          message: 'Invalid email format'
        }
      });
    }

    // Find user
    const user = await User.findOne({ email: email.toLowerCase() });
    if (!user) {
      // Log failed attempt
      await LoginAttempt.create({
        email: email.toLowerCase(),
        ipAddress: req.ip,
        successful: false,
        timestamp: new Date()
      });

      return res.status(401).json({
        error: {
          code: 'INVALID_CREDENTIALS',
          message: 'Invalid email or password'
        }
      });
    }

    // Check if account is locked
    const recentFailedAttempts = await LoginAttempt.countDocuments({
      email: email.toLowerCase(),
      successful: false,
      timestamp: { $gt: new Date(Date.now() - 15 * 60 * 1000) } // Last 15 minutes
    });

    if (recentFailedAttempts >= 5) {
      return res.status(403).json({
        error: {
          code: 'ACCOUNT_LOCKED',
          message: 'Account is locked due to too many failed attempts',
          details: {
            unlockTime: new Date(Date.now() + 15 * 60 * 1000)
          }
        }
      });
    }

    // Check if user is active
    if (!user.isActive) {
      return res.status(403).json({
        error: {
          code: 'ACCOUNT_INACTIVE',
          message: 'Account is inactive'
        }
      });
    }

    // Check password
    const isMatch = await user.comparePassword(password);
    if (!isMatch) {
      // Log failed attempt
      await LoginAttempt.create({
        email: email.toLowerCase(),
        ipAddress: req.ip,
        successful: false,
        timestamp: new Date(),
        userId: user._id
      });

      return res.status(401).json({
        error: {
          code: 'INVALID_CREDENTIALS',
          message: 'Invalid email or password',
          details: {
            remainingAttempts: 5 - recentFailedAttempts - 1
          }
        }
      });
    }

    // Get user permissions
    const permissions = await getUserPermissions(user);

    // Generate JWT token
    const tokenExpiration = rememberMe ? '7d' : '1d';
    const expiresIn = rememberMe ? 7 * 24 * 60 * 60 : 24 * 60 * 60; // in seconds

    const token = jwt.sign(
      {
        id: user._id,
        role: user.role,
        email: user.email,
        permissions
      },
      process.env.JWT_SECRET,
      { expiresIn: tokenExpiration }
    );

    // Generate refresh token if rememberMe is true
    let refreshToken = null;
    if (rememberMe) {
      refreshToken = jwt.sign(
        {
          id: user._id,
          type: 'refresh'
        },
        process.env.JWT_REFRESH_SECRET,
        { expiresIn: '7d' }
      );

      // Store refresh token hash in database
      const refreshTokenHash = await bcrypt.hash(refreshToken, 10);
      await RefreshToken.create({
        userId: user._id,
        token: refreshTokenHash,
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
      });
    }

    // Update last login
    user.lastLogin = new Date();
    await user.save();

    // Log successful login
    await LoginAttempt.create({
      email: email.toLowerCase(),
      ipAddress: req.ip,
      successful: true,
      timestamp: new Date(),
      userId: user._id
    });

    // Prepare user response (exclude sensitive data)
    const userResponse = {
      id: user._id,
      email: user.email,
      firstName: user.firstName,
      lastName: user.lastName,
      role: user.role,
      lastLogin: user.lastLogin,
      permissions
    };

    // Include assigned farm if exists
    if (user.assignedFarm) {
      const farm = await Farm.findById(user.assignedFarm);
      if (farm) {
        userResponse.assignedFarm = {
          id: farm._id,
          name: farm.name
        };
      }
    }

    // Return response
    const response = {
      token,
      expiresIn
    };

    if (refreshToken) {
      response.refreshToken = refreshToken;
    }

    response.user = userResponse;

    res.json(response);
  } catch (error) {
    console.error('Login error:', error);
    res.status(500).json({
      error: {
        code: 'SERVER_ERROR',
        message: 'An unexpected error occurred'
      }
    });
  }
});
```

#### Verify Token

Verifies a token and returns the associated user information.

- **URL**: `/api/auth/token`
- **Method**: `GET`
- **Auth Required**: Yes

**Success Response (200 OK)**:
```json
{
  "user": {
    "id": "60d21b4667d0d8992e610c85",
    "email": "user@example.com",
    "firstName": "John",
    "lastName": "Doe",
    "role": "admin",
    "assignedFarm": {
      "id": "60d21b4667d0d8992e610c86",
      "name": "Farm 1"
    }
  },
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Error Responses**:
- `401 Unauthorized`: Invalid or expired token
- `404 Not Found`: User not found

## User Management API

### Get All Users

Retrieves a list of all users (admin only).

- **URL**: `/api/users`
- **Method**: `GET`
- **Auth Required**: Yes (Admin)

**Query Parameters**:
- `limit` (optional): Maximum number of users to return (default: 100)
- `page` (optional): Page number for pagination (default: 1)
- `sort` (optional): Field to sort by (default: "lastName")
- `order` (optional): Sort order, "asc" or "desc" (default: "asc")
- `search` (optional): Search term for filtering users

**Success Response (200 OK)**:
```json
[
  {
    "_id": "60d21b4667d0d8992e610c85",
    "email": "admin@example.com",
    "firstName": "Admin",
    "lastName": "User",
    "role": "admin",
    "isActive": true,
    "lastLogin": "2023-06-15T10:30:45.123Z"
  },
  {
    "_id": "60d21b4667d0d8992e610c86",
    "email": "farmer@example.com",
    "firstName": "Farmer",
    "lastName": "User",
    "role": "farmer",
    "isActive": true,
    "lastLogin": "2023-06-14T08:15:30.456Z",
    "assignedFarm": "60d21b4667d0d8992e610c87"
  }
]
```

**Error Responses**:
- `401 Unauthorized`: Not authenticated
- `403 Forbidden`: Not authorized as admin
- `500 Internal Server Error`: Server error

### Get User by ID

Retrieves a specific user by ID.

- **URL**: `/api/users/:id`
- **Method**: `GET`
- **Auth Required**: Yes (Admin or self)

**Success Response (200 OK)**:
```json
{
  "_id": "60d21b4667d0d8992e610c85",
  "email": "user@example.com",
  "firstName": "John",
  "lastName": "Doe",
  "role": "farmer",
  "isActive": true,
  "lastLogin": "2023-06-15T10:30:45.123Z",
  "assignedFarm": {
    "_id": "60d21b4667d0d8992e610c87",
    "name": "Farm 1"
  },
  "permissions": ["read:farms", "write:pigs"]
}
```

**Error Responses**:
- `401 Unauthorized`: Not authenticated
- `403 Forbidden`: Not authorized to view this user
- `404 Not Found`: User not found
- `500 Internal Server Error`: Server error

### Create User

Creates a new user (admin only).

- **URL**: `/api/auth/register`
- **Method**: `POST`
- **Auth Required**: Yes (Admin)

**Request Body**:
```json
{
  "email": "newuser@example.com",
  "password": "password123",
  "firstName": "New",
  "lastName": "User",
  "role": "farmer",
  "permissions": ["read:farms", "write:pigs"],
  "assignedFarm": "60d21b4667d0d8992e610c87"
}
```

**Success Response (201 Created)**:
```json
{
  "_id": "60d21b4667d0d8992e610c88",
  "email": "newuser@example.com",
  "firstName": "New",
  "lastName": "User",
  "role": "farmer",
  "isActive": true,
  "permissions": ["read:farms", "write:pigs"],
  "assignedFarm": "60d21b4667d0d8992e610c87"
}
```

**Error Responses**:
- `400 Bad Request`: Invalid input or user already exists
- `401 Unauthorized`: Not authenticated
- `403 Forbidden`: Not authorized as admin
- `500 Internal Server Error`: Server error

### Update User

Updates an existing user.

- **URL**: `/api/users/:id`
- **Method**: `PUT`
- **Auth Required**: Yes (Admin or self)

**Request Body**:
```json
{
  "firstName": "Updated",
  "lastName": "Name",
  "role": "farmer",
  "isActive": true,
  "assignedFarm": "60d21b4667d0d8992e610c87"
}
```

**Success Response (200 OK)**:
```json
{
  "_id": "60d21b4667d0d8992e610c85",
  "email": "user@example.com",
  "firstName": "Updated",
  "lastName": "Name",
  "role": "farmer",
  "isActive": true,
  "lastLogin": "2023-06-15T10:30:45.123Z",
  "assignedFarm": "60d21b4667d0d8992e610c87"
}
```

**Error Responses**:
- `400 Bad Request`: Invalid input
- `401 Unauthorized`: Not authenticated
- `403 Forbidden`: Not authorized to update this user
- `404 Not Found`: User not found
- `500 Internal Server Error`: Server error

### Delete User

Deletes a user (admin only).

- **URL**: `/api/users/:id`
- **Method**: `DELETE`
- **Auth Required**: Yes (Admin)

**Success Response (200 OK)**:
```json
{
  "message": "User deleted successfully"
}
```

**Error Responses**:
- `401 Unauthorized`: Not authenticated
- `403 Forbidden`: Not authorized as admin
- `404 Not Found`: User not found
- `500 Internal Server Error`: Server error

## Farm Management API

### Get All Farms

Retrieves a list of all farms.

- **URL**: `/api/farms`
- **Method**: `GET`
- **Auth Required**: Yes

**Success Response (200 OK)**:
```json
[
  {
    "_id": "60d21b4667d0d8992e610c87",
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
  {
    "_id": "60d21b4667d0d8992e610c88",
    "name": "Farm 2",
    "location": "Location 2",
    "description": "Description for Farm 2",
    "isActive": true,
    "counts": {
      "barns": 3,
      "stalls": 12,
      "pigs": 60
    }
  }
]
```

**Error Responses**:
- `401 Unauthorized`: Not authenticated
- `500 Internal Server Error`: Server error

### Get Farm by ID

Retrieves a specific farm by ID.

- **URL**: `/api/farms/:id`
- **Method**: `GET`
- **Auth Required**: Yes

**Success Response (200 OK)**:
```json
{
  "_id": "60d21b4667d0d8992e610c87",
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
    {
      "_id": "60d21b4667d0d8992e610c89",
      "name": "Barn A",
      "farmId": "60d21b4667d0d8992e610c87",
      "stallCount": 8
    },
    {
      "_id": "60d21b4667d0d8992e610c90",
      "name": "Barn B",
      "farmId": "60d21b4667d0d8992e610c87",
      "stallCount": 12
    }
  ],
  "recentPigs": [
    {
      "pigId": 1001,
      "tag": "PIG-1001",
      "breed": "Yorkshire",
      "age": 12
    }
  ],
  "devices": [
    {
      "_id": "60d21b4667d0d8992e610c91",
      "deviceId": "DEV-001",
      "type": "temperature",
      "status": "active"
    }
  ]
}
```

**Error Responses**:
- `401 Unauthorized`: Not authenticated
- `403 Forbidden`: Not authorized to access this farm
- `404 Not Found`: Farm not found
- `500 Internal Server Error`: Server error

### Create Farm

Creates a new farm (admin only).

- **URL**: `/api/farms`
- **Method**: `POST`
- **Auth Required**: Yes (Admin)

**Request Body**:
```json
{
  "name": "New Farm",
  "location": "New Location",
  "description": "Description for New Farm",
  "isActive": true
}
```

**Success Response (201 Created)**:
```json
{
  "_id": "60d21b4667d0d8992e610c92",
  "name": "New Farm",
  "location": "New Location",
  "description": "Description for New Farm",
  "isActive": true
}
```

**Error Responses**:
- `400 Bad Request`: Invalid input
- `401 Unauthorized`: Not authenticated
- `403 Forbidden`: Not authorized as admin
- `500 Internal Server Error`: Server error

### Update Farm

Updates an existing farm (admin only).

- **URL**: `/api/farms/:id`
- **Method**: `PUT`
- **Auth Required**: Yes (Admin)

**Request Body**:
```json
{
  "name": "Updated Farm",
  "location": "Updated Location",
  "description": "Updated description",
  "isActive": true
}
```

**Success Response (200 OK)**:
```json
{
  "_id": "60d21b4667d0d8992e610c87",
  "name": "Updated Farm",
  "location": "Updated Location",
  "description": "Updated description",
  "isActive": true
}
```

**Error Responses**:
- `400 Bad Request`: Invalid input
- `401 Unauthorized`: Not authenticated
- `403 Forbidden`: Not authorized as admin
- `404 Not Found`: Farm not found
- `500 Internal Server Error`: Server error

### Delete Farm

Deletes a farm (admin only).

- **URL**: `/api/farms/:id`
- **Method**: `DELETE`
- **Auth Required**: Yes (Admin)

**Query Parameters**:
- `cascade` (optional): If "true", deletes all associated barns, stalls, and pigs

**Success Response (200 OK)**:
```json
{
  "message": "Farm deleted successfully"
}
```

**Error Responses**:
- `401 Unauthorized`: Not authenticated
- `403 Forbidden`: Not authorized as admin
- `404 Not Found`: Farm not found
- `500 Internal Server Error`: Server error

## Pig Management API

### Get All Pigs

Retrieves a list of all pigs.

- **URL**: `/api/pigs`
- **Method**: `GET`
- **Auth Required**: Yes

**Success Response (200 OK)**:
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
  {
    "owner": "PIG-002",
    "status": "at risk",
    "costs": "8",
    "region": "Stall B",
    "stability": 0.4,
    "lastEdited": "02/01/2023, 14:30",
    "breed": "Duroc",
    "active": true
  }
]
```

**Error Responses**:
- `401 Unauthorized`: Not authenticated
- `500 Internal Server Error`: Server error

### Get Pig Overview

Retrieves aggregated pig data for the dashboard.

- **URL**: `/api/pigs/overview`
- **Method**: `GET`
- **Auth Required**: Yes

**Query Parameters**:
- `filter` (optional): Filter by category (e.g., "breeding", "new", "healthy")

**Success Response (200 OK)**:
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
    {
      "name": "Healthy",
      "Farm 1": 80,
      "Farm 2": 40,
      "averageWeight": 100
    },
    {
      "name": "At Risk",
      "Farm 1": 15,
      "Farm 2": 10,
      "averageWeight": 90
    }
  ],
  "stats": {
    "totalPigs": 160,
    "categories": ["Breeding", "Healthy", "At Risk"],
    "locations": ["Farm 1", "Farm 2"]
  }
}
```

**Error Responses**:
- `401 Unauthorized`: Not authenticated
- `500 Internal Server Error`: Server error

### Get Pig by ID

Retrieves a specific pig by ID.

- **URL**: `/api/pigs/:id`
- **Method**: `GET`
- **Auth Required**: Yes

**Success Response (200 OK)**:
```json
{
  "pigId": 1001,
  "tag": "PIG-1001",
  "breed": "Yorkshire",
  "age": 12,
  "currentLocation": {
    "farmId": {
      "_id": "60d21b4667d0d8992e610c87",
      "name": "Farm 1"
    },
    "barnId": {
      "_id": "60d21b4667d0d8992e610c89",
      "name": "Barn A"
    },
    "stallId": {
      "_id": "60d21b4667d0d8992e610c93",
      "name": "Stall 1"
    }
  },
  "active": true
}
```

**Error Responses**:
- `400 Bad Request`: Invalid pig ID
- `401 Unauthorized`: Not authenticated
- `404 Not Found`: Pig not found
- `500 Internal Server Error`: Server error

### Get Pig Posture Data

Retrieves posture data for a specific pig.

- **URL**: `/api/pigs/:id/posture/aggregated`
- **Method**: `GET`
- **Auth Required**: Yes

**Query Parameters**:
- `start` (optional): Start date (YYYY-MM-DD)
- `end` (optional): End date (YYYY-MM-DD)

**Success Response (200 OK)**:
```json
[
  {
    "date": "2022-08-01",
    "standing": 45,
    "sitting": 30,
    "lying": 25
  },
  {
    "date": "2022-08-02",
    "standing": 40,
    "sitting": 35,
    "lying": 25
  },
  {
    "date": "2022-08-03",
    "standing": 50,
    "sitting": 25,
    "lying": 25
  }
]
```

**Error Responses**:
- `400 Bad Request`: Invalid pig ID or date range
- `401 Unauthorized`: Not authenticated
- `404 Not Found`: Pig not found
- `500 Internal Server Error`: Server error

### Create Pig

Creates a new pig.

- **URL**: `/api/pigs`
- **Method**: `POST`
- **Auth Required**: Yes

**Request Body**:
```json
{
  "pigId": 1003,
  "tag": "PIG-1003",
  "breed": "Yorkshire",
  "age": 10,
  "currentLocation": {
    "farmId": "60d21b4667d0d8992e610c87",
    "barnId": "60d21b4667d0d8992e610c89",
    "stallId": "60d21b4667d0d8992e610c93"
  }
}
```

**Success Response (201 Created)**:
```json
{
  "success": true,
  "pig": {
    "pigId": 1003,
    "tag": "PIG-1003",
    "breed": "Yorkshire",
    "age": 10,
    "currentLocation": {
      "farmId": "60d21b4667d0d8992e610c87",
      "barnId": "60d21b4667d0d8992e610c89",
      "stallId": "60d21b4667d0d8992e610c93"
    },
    "active": true,
    "_id": "60d21b4667d0d8992e610c94"
  }
}
```

**Error Responses**:
- `400 Bad Request`: Invalid input or pig ID already exists
- `401 Unauthorized`: Not authenticated
- `500 Internal Server Error`: Server error

## Statistics API

### Get System Statistics

Retrieves system-wide statistics.

- **URL**: `/api/stats`
- **Method**: `GET`
- **Auth Required**: Yes

**Success Response (200 OK)**:
```json
{
  "counts": {
    "pigs": 160,
    "farms": 2,
    "barns": 8,
    "stalls": 32,
    "devices": 15
  },
  "healthStatus": {
    "healthy": 120,
    "at risk": 25,
    "critical": 10,
    "no movement": 5
  },
  "recentActivity": [
    {
      "_id": "60d21b4667d0d8992e610c95",
      "type": "pig",
      "action": "created",
      "description": "Pig PIG-1003 was added to the system",
      "timestamp": "2023-06-15T14:30:45.123Z",
      "userId": {
        "firstName": "John",
        "lastName": "Doe",
        "email": "john@example.com"
      }
    }
  ]
}
```

**Error Responses**:
- `401 Unauthorized`: Not authenticated
- `500 Internal Server Error`: Server error

## Activity Logging API

### Get Recent Activities

Retrieves recent system activities.

- **URL**: `/api/activities`
- **Method**: `GET`
- **Auth Required**: Yes

**Query Parameters**:
- `limit` (optional): Maximum number of activities to return (default: 10)
- `type` (optional): Filter by activity type (e.g., "user", "pig", "farm")
- `userId` (optional): Filter by user ID

**Success Response (200 OK)**:
```json
[
  {
    "_id": "60d21b4667d0d8992e610c95",
    "type": "pig",
    "action": "created",
    "description": "Pig PIG-1003 was added to the system",
    "timestamp": "2023-06-15T14:30:45.123Z",
    "userId": {
      "_id": "60d21b4667d0d8992e610c85",
      "firstName": "John",
      "lastName": "Doe",
      "email": "john@example.com"
    },
    "entityId": "60d21b4667d0d8992e610c94",
    "metadata": {
      "pigId": 1003,
      "tag": "PIG-1003"
    }
  },
  {
    "_id": "60d21b4667d0d8992e610c96",
    "type": "user",
    "action": "login",
    "description": "User \"john@example.com\" logged in",
    "timestamp": "2023-06-15T10:30:45.123Z",
    "userId": {
      "_id": "60d21b4667d0d8992e610c85",
      "firstName": "John",
      "lastName": "Doe",
      "email": "john@example.com"
    },
    "metadata": {
      "email": "john@example.com",
      "role": "admin"
    }
  }
]
```

**Error Responses**:
- `401 Unauthorized`: Not authenticated
- `500 Internal Server Error`: Server error

## Data Upload API

### Upload Posture Data

Uploads posture data for a pig.

- **URL**: `/api/upload/postureupload`
- **Method**: `POST`
- **Auth Required**: Yes

**Request Body**:
```json
{
  "pigId": 1001,
  "timestamp": "2023-06-15T12:00:00.000Z",
  "score": 3
}
```

**Success Response (201 Created)**:
```json
{
  "success": true,
  "data": {
    "pigId": 1001,
    "timestamp": "2023-06-15T12:00:00.000Z",
    "score": 3,
    "_id": "60d21b4667d0d8992e610c97"
  }
}
```

**Error Responses**:
- `400 Bad Request`: Invalid input
- `401 Unauthorized`: Not authenticated
- `404 Not Found`: Pig not found
- `500 Internal Server Error`: Server error

## Real-time Communication

In addition to the RESTful API, the system provides real-time updates via Socket.IO.

### Socket.IO Events

#### Connection

Clients connect to the Socket.IO server:

```javascript
// Client-side code
import { io } from 'socket.io-client';

const socket = io('http://localhost:8080', {
  path: '/socket.io',
  transports: ['websocket']
});

socket.on('connect', () => {
  console.log('Connected to Socket.IO server');

  // Join rooms based on user role
  socket.emit('join', {
    userId: 'user-id',
    role: 'admin',
    assignedFarm: 'farm-id'
  });
});
```

#### Stats Update

Clients receive updated statistics:

```javascript
// Client-side code
socket.on('stats_update', (stats) => {
  console.log('Received updated stats:', stats);
  // Update UI with new stats
});
```

#### Activity Update

Clients receive new activity logs:

```javascript
// Client-side code
socket.on('activity', (activity) => {
  console.log('New activity:', activity);
  // Update activity feed in UI
});
```

## Rate Limiting

To protect the API from abuse, rate limiting is implemented on certain endpoints:

- `/api/auth/login`: 5 requests per minute per IP
- Most other endpoints: 100 requests per 15 minutes per IP

When a rate limit is exceeded, the API returns a `429 Too Many Requests` response with information about when the limit will reset.

## Error Handling

The PAAL API implements a comprehensive and consistent error handling strategy to provide clear, actionable feedback to clients. This section details the error response format, error categorization, and implementation details.

### Error Response Format

All API error responses follow a consistent structure:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {
      // Additional context-specific error details
    },
    "path": "/api/resource/123",
    "timestamp": "2023-06-15T10:30:45.123Z",
    "requestId": "req-123456"
  }
}
```

#### Error Response Fields

| Field | Type | Description | Always Present |
|-------|------|-------------|----------------|
| `error.code` | String | Machine-readable error code | Yes |
| `error.message` | String | Human-readable error description | Yes |
| `error.details` | Object | Additional error context | No |
| `error.path` | String | API endpoint path that generated the error | Yes |
| `error.timestamp` | String | ISO 8601 timestamp when the error occurred | Yes |
| `error.requestId` | String | Unique identifier for the request (for support/debugging) | Yes |

### HTTP Status Codes

The API uses appropriate HTTP status codes to indicate the nature of errors:

| Status Code | Category | Description | Example Scenarios |
|-------------|----------|-------------|-------------------|
| 400 | Bad Request | The request contains invalid parameters or is malformed | Missing required fields, invalid data format |
| 401 | Unauthorized | Authentication is required or has failed | Missing token, expired token, invalid credentials |
| 403 | Forbidden | The authenticated user lacks permission | Attempting to access restricted resources |
| 404 | Not Found | The requested resource does not exist | Invalid ID, deleted resource |
| 409 | Conflict | The request conflicts with the current state | Duplicate entry, concurrent modification |
| 422 | Unprocessable Entity | The request is well-formed but contains semantic errors | Validation errors |
| 429 | Too Many Requests | The client has sent too many requests | Rate limit exceeded |
| 500 | Internal Server Error | An unexpected error occurred on the server | Database errors, unhandled exceptions |
| 503 | Service Unavailable | The service is temporarily unavailable | Maintenance, overload |

### Error Codes

The API uses standardized error codes to provide machine-readable error information:

#### Authentication Errors

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| `UNAUTHORIZED` | 401 | Authentication is required |
| `INVALID_CREDENTIALS` | 401 | Invalid username or password |
| `INVALID_TOKEN` | 401 | The provided token is invalid |
| `TOKEN_EXPIRED` | 401 | The provided token has expired |
| `ACCOUNT_INACTIVE` | 403 | The user account is inactive |
| `ACCOUNT_LOCKED` | 403 | The account is temporarily locked |
| `INSUFFICIENT_PERMISSIONS` | 403 | The user lacks required permissions |

#### Validation Errors

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| `VALIDATION_ERROR` | 422 | The request contains validation errors |
| `MISSING_REQUIRED_FIELD` | 400 | A required field is missing |
| `INVALID_FIELD_FORMAT` | 400 | A field has an invalid format |
| `INVALID_FIELD_VALUE` | 400 | A field has an invalid value |
| `INVALID_QUERY_PARAMETER` | 400 | A query parameter is invalid |

#### Resource Errors

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| `RESOURCE_NOT_FOUND` | 404 | The requested resource does not exist |
| `RESOURCE_ALREADY_EXISTS` | 409 | The resource already exists |
| `RESOURCE_CONFLICT` | 409 | The request conflicts with the current state |
| `RESOURCE_GONE` | 410 | The resource is no longer available |

#### Rate Limiting Errors

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| `RATE_LIMIT_EXCEEDED` | 429 | The rate limit has been exceeded |

#### Server Errors

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| `INTERNAL_SERVER_ERROR` | 500 | An unexpected error occurred |
| `SERVICE_UNAVAILABLE` | 503 | The service is temporarily unavailable |
| `DATABASE_ERROR` | 500 | A database error occurred |

### Error Handling Implementation

The API implements error handling using middleware and utility functions:

#### Error Middleware

```javascript
// middleware/errorHandler.js
const errorHandler = (err, req, res, next) => {
  // Default to 500 Internal Server Error
  let statusCode = err.statusCode || 500;
  let errorCode = err.code || 'INTERNAL_SERVER_ERROR';
  let message = err.message || 'An unexpected error occurred';
  let details = err.details || {};

  // Handle specific error types
  if (err.name === 'ValidationError') {
    // Mongoose validation error
    statusCode = 422;
    errorCode = 'VALIDATION_ERROR';
    message = 'Validation failed';
    details = Object.keys(err.errors).reduce((acc, key) => {
      acc[key] = err.errors[key].message;
      return acc;
    }, {});
  } else if (err.name === 'CastError') {
    // Mongoose cast error (invalid ID)
    statusCode = 400;
    errorCode = 'INVALID_FIELD_VALUE';
    message = `Invalid ${err.path}`;
  } else if (err.code === 11000) {
    // MongoDB duplicate key error
    statusCode = 409;
    errorCode = 'RESOURCE_ALREADY_EXISTS';
    message = 'Duplicate entry';
    const field = Object.keys(err.keyValue)[0];
    details = {
      field,
      value: err.keyValue[field]
    };
  }

  // Log server errors
  if (statusCode >= 500) {
    console.error('Server error:', err);
  }

  // Send error response
  res.status(statusCode).json({
    error: {
      code: errorCode,
      message,
      details,
      path: req.path,
      timestamp: new Date().toISOString(),
      requestId: req.id // Added by request ID middleware
    }
  });
};

module.exports = errorHandler;
```

#### Custom Error Classes

```javascript
// utils/errors.js
class ApiError extends Error {
  constructor(statusCode, code, message, details = {}) {
    super(message);
    this.statusCode = statusCode;
    this.code = code;
    this.details = details;
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

class BadRequestError extends ApiError {
  constructor(message, code = 'BAD_REQUEST', details = {}) {
    super(400, code, message, details);
  }
}

class UnauthorizedError extends ApiError {
  constructor(message = 'Authentication required', code = 'UNAUTHORIZED', details = {}) {
    super(401, code, message, details);
  }
}

class ForbiddenError extends ApiError {
  constructor(message = 'Access denied', code = 'FORBIDDEN', details = {}) {
    super(403, code, message, details);
  }
}

class NotFoundError extends ApiError {
  constructor(resource = 'Resource', code = 'RESOURCE_NOT_FOUND', details = {}) {
    super(404, code, `${resource} not found`, details);
  }
}

class ConflictError extends ApiError {
  constructor(message = 'Resource conflict', code = 'RESOURCE_CONFLICT', details = {}) {
    super(409, code, message, details);
  }
}

class ValidationError extends ApiError {
  constructor(message = 'Validation failed', details = {}) {
    super(422, 'VALIDATION_ERROR', message, details);
  }
}

class RateLimitError extends ApiError {
  constructor(message = 'Rate limit exceeded', details = {}) {
    super(429, 'RATE_LIMIT_EXCEEDED', message, details);
  }
}

module.exports = {
  ApiError,
  BadRequestError,
  UnauthorizedError,
  ForbiddenError,
  NotFoundError,
  ConflictError,
  ValidationError,
  RateLimitError
};
```

#### Usage in Route Handlers

```javascript
// Example route handler with error handling
const { NotFoundError, ValidationError } = require('../utils/errors');

router.get('/pigs/:id', async (req, res, next) => {
  try {
    const id = parseInt(req.params.id, 10);

    // Validate ID
    if (isNaN(id)) {
      throw new ValidationError('Invalid pig ID', {
        id: 'Must be a number'
      });
    }

    // Find pig by ID
    const pig = await Pig.findOne({ pigId: id });

    // Check if pig exists
    if (!pig) {
      throw new NotFoundError('Pig', 'PIG_NOT_FOUND', {
        pigId: id
      });
    }

    // Return pig data
    res.json(pig);
  } catch (error) {
    // Pass error to error handling middleware
    next(error);
  }
});
```

### Error Logging and Monitoring

All errors are logged and monitored:

1. **Development Environment**:
   - Detailed error information is logged to the console
   - Stack traces are included in logs

2. **Production Environment**:
   - Errors are logged to a structured logging system
   - Critical errors trigger alerts
   - Error metrics are collected for monitoring

3. **Error Tracking**:
   - Integration with error tracking services (e.g., Sentry)
   - Grouping of similar errors
   - Tracking of error frequency and impact

### Client-Side Error Handling Recommendations

Clients should implement the following error handling strategies:

1. **HTTP Status Code Handling**:
   - Handle different status codes appropriately
   - Implement retry logic for 5xx errors

2. **Error Code Parsing**:
   - Parse the `error.code` field for programmatic handling
   - Display user-friendly messages based on error codes

3. **Validation Error Handling**:
   - Map validation errors to form fields
   - Display field-specific error messages

4. **Rate Limit Handling**:
   - Implement exponential backoff for rate limit errors
   - Parse rate limit headers to adjust request timing

5. **Authentication Error Handling**:
   - Redirect to login page for 401 errors
   - Clear invalid tokens from storage

### Example Error Responses

#### Validation Error (422 Unprocessable Entity)

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": {
      "name": "Name is required",
      "age": "Age must be a positive number",
      "email": "Invalid email format"
    },
    "path": "/api/pigs",
    "timestamp": "2023-06-15T10:30:45.123Z",
    "requestId": "req-123456"
  }
}
```

#### Resource Not Found (404 Not Found)

```json
{
  "error": {
    "code": "PIG_NOT_FOUND",
    "message": "Pig not found",
    "details": {
      "pigId": 1001
    },
    "path": "/api/pigs/1001",
    "timestamp": "2023-06-15T10:30:45.123Z",
    "requestId": "req-123456"
  }
}
```

#### Authentication Error (401 Unauthorized)

```json
{
  "error": {
    "code": "TOKEN_EXPIRED",
    "message": "Token has expired",
    "path": "/api/farms",
    "timestamp": "2023-06-15T10:30:45.123Z",
    "requestId": "req-123456"
  }
}
```

#### Rate Limit Error (429 Too Many Requests)

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded",
    "details": {
      "retryAfter": 60,
      "limit": 100,
      "remaining": 0,
      "reset": 1623760800
    },
    "path": "/api/auth/login",
    "timestamp": "2023-06-15T10:30:45.123Z",
    "requestId": "req-123456"
  }
}
```

## API Versioning

The current API version is v1, which is implicit in the base URL. Future versions would be accessed via:

```
http://localhost:8080/api/v2/...
```

## Cross-Origin Resource Sharing (CORS)

The API supports CORS with the following configuration:

```javascript
const CORS_CONFIG = {
  origin: process.env.CORS_ORIGIN || '*',
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization']
};
```

In production, the `origin` would be restricted to specific domains.
