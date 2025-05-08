# Backend Overall Architecture

## Architecture Type

The Backend is built using Node.js and Express.js. This architecture was chosen for its simplicity, ease of development, and suitability for the current scale of the application. This Approach allows for a straightforward deployment and maintenance while still providing modular organization.

## Core Technologies

### Framework and Libraries

- **Node.js**: JavaScript runtime for building the server-side application
- **Express.js**: Web application framework for handling HTTP requests and routing
- **Mongoose**: Object Data Modeling (ODM) library for MongoDB
- **Socket.IO**: Library for real-time, bidirectional communication

### Database

- **MongoDB**: NoSQL database used for storing all application data
- **MongoDB Change Streams**: Used for real-time data updates

### Authentication and Security

- **JSON Web Tokens (JWT)**: Used for authentication and authorization
- **bcrypt**: Library for password hashing
- **express-rate-limit**: Middleware for rate limiting API requests

### Development Tools

- **npm**: Package manager for managing dependencies
- **nodemon**: Development tool for automatic server restarts
- **ESLint**: Static code analysis tool for identifying problematic patterns

## Application Structure

The backend application follows a modular structure organized by feature:

```
server/
├── config/             # Configuration settings
│   └── index.js        # Exports configuration objects
├── db/                 # Database connection and models
│   ├── connection.js   # Manages MongoDB connection
│   └── models.js       # Registers all Mongoose models
├── middleware/         # Express middleware
│   ├── authMiddleware.js # Authentication middleware
│   ├── role.js         # Role-based access control
│   └── index.js        # Configures middleware
├── models/             # Mongoose models
│   ├── Pig.js          # Pig data model
│   ├── Farm.js         # Farm data model
│   ├── Barn.js         # Barn data model
│   ├── Stall.js        # Stall data model
│   ├── User.js         # User data model
│   ├── PostureData.js  # Pig posture data model
│   └── ...             # Other data models
├── routes/             # API routes
│   ├── pig.js          # Pig-related endpoints
│   ├── farm.js         # Farm-related endpoints
│   ├── auth.js         # Authentication endpoints
│   ├── user.js         # User management endpoints
│   └── ...             # Other route files
├── scripts/            # Utility scripts
│   ├── seed-all.js     # Database seeding script
│   └── ...             # Other scripts
├── services/           # Business logic services
│   ├── activityLogger.js # Activity logging service
│   └── ...             # Other services
├── socket/             # Socket.IO related code
│   ├── events.js       # Socket event handlers
│   ├── stats.js        # Statistics emission
│   └── index.js        # Socket.IO initialization
├── utils/              # Utility functions
│   └── routeValidator.js # Route validation utility
├── index.js            # Legacy entry point
└── server.js           # Main entry point
```

## Request Flow

The flow of a request through the backend system is as follows:

1. **HTTP Request**: Client sends an HTTP request to the server
2. **NGINX Proxy**: Request is received by NGINX and forwarded to the Node.js application
3. **Express Middleware**: Request passes through middleware for:
    - CORS handling 
        - **(DEVMODE: PLEASE UPDATE CORS TO RESTRICT ACCESS ON DEPLOYMENT)**
    - Body parsing
    - Authentication (if required)
    - Request logging
4. **Route Handler**: Request is routed to the appropriate handler based on the URL path
5. **Business Logic**: Route handler executes business logic, often interacting with the database
6. **Database Interaction**: Mongoose models are used to query or update the MongoDB database
7. **Response**: Handler sends an HTTP response back to the client
8. **Real-time Updates**: If the request resulted in data changes, Socket.IO may emit events to connected clients

## Authentication and Authorization

### Authentication Flow

1. **Login Request**: Client sends credentials to `/api/auth/login`
2. **Credential Verification**: Server verifies email and password
3. **Token Generation**: Server generates a JWT containing user information
4. **Token Response**: Token is sent back to the client
5. **Subsequent Requests**: Client includes token in Authorization header
6. **Token Verification**: `authenticateJWT` middleware verifies token on protected routes

### Authorization

Role-based access control is implemented using middleware:

```javascript
// middleware/authMiddleware.js
const isAdmin = (req, res, next) => {
  if (!req.user) {
    return res.status(401).json({ error: 'Not authenticated' });
  }

  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Not authorized as admin' });
  }

  next();
};
```

## Data Flow

### Database Interactions

The application uses Mongoose models to interact with MongoDB:

```javascript
// Example of database interaction in routes/pig.js
router.get('/', async (req, res) => {
  try {
    const pigs = await Pig.find({})
      .populate('currentLocation.farmId')
      .populate('currentLocation.barnId')
      .populate('currentLocation.stallId')
      .populate('healthStatus')
      .sort({ updatedAt: -1 });

    // Transform data for response
    const transformedPigs = pigs.map(pig => ({
      owner: `PIG-${pig.pigId.toString().padStart(3, '0')}`,
      status: pig.healthStatus?.status || '------',
      costs: pig.age || '------',
      region: pig.currentLocation.stallId?.name
        ? `${pig.currentLocation.stallId.name}`
        : '------',
      stability: pig.stability || 0,
      lastEdited: pig.updatedAt
        ? new Date(pig.updatedAt).toLocaleDateString('en-GB', {
          day: '2-digit',
          month: '2-digit',
          year: 'numeric',
          hour: '2-digit',
          minute: '2-digit'
        })
        : new Date().toLocaleDateString('en-GB', {
          day: '2-digit',
          month: '2-digit',
          year: 'numeric',
          hour: '2-digit',
          minute: '2-digit'
        }),
      breed: pig.breed || '------',
      active: pig.active
    }));

    res.json(transformedPigs);
  } catch (error) {
    console.error('Error fetching pigs:', error);
    res.status(500).json({ error: 'Failed to fetch pigs' });
  }
});
```

### Real-time Updates

The application uses MongoDB Change Streams and Socket.IO to provide real-time updates:

```javascript
// db/models.js
const setupChangeStreams = (emitUpdatedStats) => {
  // Watch Pig model changes
  const Pig = mongoose.model('Pig');
  const pigChangeStream = Pig.watch([], { fullDocument: 'updateLookup' });
  
  pigChangeStream.on('change', async (change) => {
    await emitUpdatedStats();
    
    // Log the activity
    if (change.operationType === 'insert') {
      const activity = await logActivity({
        type: 'pig',
        action: 'created',
        description: `Pig ${change.fullDocument.tag} was added to the system`,
        entityId: change.fullDocument._id
      });
      
      // Emit the activity to all connected clients
      emitActivity(io, activity);
    }
    // Handle other change types...
  });
  
  // Watch other model changes...
};
```

## Error Handling

The application implements consistent error handling throughout:

```javascript
// Example error handling pattern
try {
  // Operation that might fail
} catch (error) {
  console.error('Error description:', error);
  res.status(500).json({ 
    error: 'User-friendly error message',
    details: process.env.NODE_ENV === 'development' ? error.message : undefined
  });
}
```

## Configuration Management

Configuration is centralized in the `config` directory:

```javascript
// config/index.js
require('dotenv').config();

// Server configuration
const SERVER_PORT = 5005;

// Database configuration
const DATABASE_CONFIG = {
  host: process.env.DATABASE_HOST,
  port: process.env.DATABASE_PORT,
  database: process.env.MONGO_INITDB_DATABASE,
  username: process.env.MONGO_INITDB_ROOT_USERNAME,
  password: process.env.MONGO_INITDB_ROOT_PASSWORD,
};

// CORS configuration
const CORS_CONFIG = {
  origin: process.env.CORS_ORIGIN || '*',
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization'],
};

module.exports = {
  SERVER_PORT,
  DATABASE_CONFIG,
  CORS_CONFIG,
};
```

## Security Measures

The application implements several security measures:

1. **Authentication**: JWT-based authentication for API access
2. **Password Security**: Passwords are hashed using bcrypt
3. **Input Validation**: All user inputs are validated before processing
4. **Rate Limiting**: Protection against brute force attacks
5. **CORS Configuration**: Restricted cross-origin resource sharing
6. **HTTP Headers**: Security-related HTTP headers are set
7. **Environment Variables**: Sensitive information is stored in environment variables
