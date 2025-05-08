# Service Decomposition

## Overview

The Backend System is currently organized in a modular way that follows service-oriented principles. This document outlines the routes and endpoints for the application, their responsibilities, and how they interact with each other.

## System Orginization

Here is the outline of the services operated within the backend for the system

1. **Authentication Service**
2. **User Management Service**
3. **Farm Management Service**
4. **Pig Management Service**
5. **Data Collection Service**
6. **Analytics Service**
7. **Notification Service**
8. **Activity Logging Service**

## Authentication

### Responsibilities

- User authentication (login/logout)
- Token generation and validation
- Password management (reset, change)
- Session management

### Key Components

- **Routes**: `/api/auth/*`
- **Models**: `User`
- **Middleware**: `authMiddleware.js`

### API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/auth/login` | POST | Authenticate user and generate token |
| `/api/auth/token` | GET | Verify token and return user info |
| `/api/auth/register` | POST | Register a new user (admin only) |

### Communication between services

- **User Management Service**: Shares the User model for user information
- **Activity Logging Service**: Logs authentication events

### Code Example

```javascript
// routes/auth.js (excerpt)
router.post('/login', loginLimiter, async (req, res) => {
  try {
    const { email, password } = req.body;

    // Validate input
    if (!email || !password) {
      return res.status(400).json({ error: 'Email and password are required' });
    }

    // Find user
    const user = await User.findOne({ email: email.toLowerCase() });
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    // Check if user is active
    if (!user.isActive) {
      return res.status(401).json({ error: 'Account is inactive' });
    }

    // Check password
    const isMatch = await user.comparePassword(password);
    if (!isMatch) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    // Generate JWT token
    const token = jwt.sign(
      {
        id: user._id,
        role: user.role,
        email: user.email
      },
      process.env.JWT_SECRET || 'fallback_secret',
      { expiresIn: '1d' }
    );

    // Update last login
    user.lastLogin = new Date();
    await User.updateOne({ email: user.email }, { lastLogin: new Date() });

    // Log the login activity
    const activity = await logActivity({
      type: 'user',
      action: 'login',
      description: `User "${user.email}" logged in`,
      userId: user._id,
      ipAddress: req.ip,
      metadata: {
        email: user.email,
        role: user.role,
        name: `${user.firstName} ${user.lastName}`
      }
    });

    // Return user info and token
    const userResponse = user.toObject();
    delete userResponse.password;

    res.json({
      token,
      user: userResponse
    });
  } catch (error) {
    console.error('Login error:', error);
    res.status(500).json({ error: 'Server error' });
  }
});
```

## User Management Service

### Responsibilities

- User CRUD operations
- Role and permission management
- User profile management

### Key Components

- **Routes**: `/api/users/*`
- **Models**: `User`
- **Middleware**: `authMiddleware.js`, `role.js`

### API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/users` | GET | Get all users (admin only) |
| `/api/users/:id` | GET | Get user by ID |
| `/api/users/:id` | PUT | Update user |
| `/api/users/:id` | DELETE | Delete user |
| `/api/users/:id/permissions` | PUT | Update user permissions |

### Interaction with Other Services

- **Authentication Service**: Shares the User model
- **Farm Management Service**: Users can be assigned to farms
- **Activity Logging Service**: Logs user management activities

### Code Example

```javascript
// routes/user.js (excerpt)
router.get('/', authenticateJWT, isAdmin, async (req, res) => {
  try {
    const users = await User.find({})
      .select('-password')
      .sort({ lastName: 1, firstName: 1 });
    
    res.json(users);
  } catch (error) {
    console.error('Error fetching users:', error);
    res.status(500).json({ error: 'Failed to fetch users' });
  }
});

router.put('/:id', authenticateJWT, isAdmin, async (req, res) => {
  try {
    const { firstName, lastName, email, role, isActive, assignedFarm } = req.body;
    
    // Find user
    const user = await User.findById(req.params.id);
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    // Update user fields
    if (firstName) user.firstName = firstName;
    if (lastName) user.lastName = lastName;
    if (email) user.email = email.toLowerCase();
    if (role) user.role = role;
    if (isActive !== undefined) user.isActive = isActive;
    if (assignedFarm) user.assignedFarm = assignedFarm;
    
    await user.save();
    
    // Log activity
    await logActivity({
      type: 'user',
      action: 'updated',
      description: `User "${user.email}" was updated`,
      userId: req.user.id,
      entityId: user._id,
      metadata: {
        email: user.email,
        role: user.role,
        name: `${user.firstName} ${user.lastName}`
      }
    });
    
    // Return updated user without password
    const userResponse = user.toObject();
    delete userResponse.password;
    
    res.json(userResponse);
  } catch (error) {
    console.error('Error updating user:', error);
    res.status(500).json({ error: 'Failed to update user' });
  }
});
```

## Farm Management Service

### Responsibilities

- Farm CRUD operations
- Barn and stall management
- Location hierarchy management

### Key Components

- **Routes**: `/api/farms/*`, `/api/barns/*`, `/api/stalls/*`
- **Models**: `Farm`, `Barn`, `Stall`

### API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/farms` | GET | Get all farms |
| `/api/farms` | POST | Create a new farm |
| `/api/farms/:id` | GET | Get farm by ID |
| `/api/farms/:id` | PUT | Update farm |
| `/api/farms/:id` | DELETE | Delete farm |
| `/api/barns/farm/:farmId` | GET | Get barns by farm ID |
| `/api/stalls/barn/:barnId` | GET | Get stalls by barn ID |

### Interaction with Other Services

- **User Management Service**: Farms can be assigned to users
- **Pig Management Service**: Pigs are located in stalls
- **Activity Logging Service**: Logs farm management activities

### Code Example

```javascript
// routes/farm.js (excerpt)
router.get('/', authenticateJWT, async (req, res) => {
  try {
    // If user is a farmer, they can only see their assigned farm
    if (req.user.role === 'farmer' && req.user.assignedFarm) {
      const farmId = req.user.assignedFarm;
      
      // Filter to only show the assigned farm
      const farm = await Farm.findById(farmId);
      if (!farm) {
        return res.status(404).json({ error: 'Farm not found' });
      }
      
      // Get counts for this farm
      const [barns, stalls, pigs] = await Promise.all([
        Barn.countDocuments({ farmId: farm._id }),
        Stall.countDocuments({ farmId: farm._id }),
        Pig.countDocuments({ 'currentLocation.farmId': farm._id })
      ]);
      
      const farmWithCounts = {
        ...farm.toObject(),
        counts: {
          barns,
          stalls,
          pigs
        }
      };
      
      return res.json([farmWithCounts]);
    }
    
    // For admins, get all farms with counts
    const farms = await Farm.find({});
    
    // Get counts for each farm
    const farmsWithCounts = await Promise.all(
      farms.map(async (farm) => {
        const [barns, stalls, pigs] = await Promise.all([
          Barn.countDocuments({ farmId: farm._id }),
          Stall.countDocuments({ farmId: farm._id }),
          Pig.countDocuments({ 'currentLocation.farmId': farm._id })
        ]);
        
        return {
          ...farm.toObject(),
          counts: {
            barns,
            stalls,
            pigs
          }
        };
      })
    );
    
    res.json(farmsWithCounts);
  } catch (error) {
    console.error('Error fetching farms:', error);
    res.status(500).json({ error: 'Failed to fetch farms' });
  }
});
```

## Pig Management Service

### Responsibilities

- Pig CRUD operations
- Pig health record management
- Pig location management

### Key Components

- **Routes**: `/api/pigs/*`
- **Models**: `Pig`, `PigHealthStatus`, `PigBCS`

### API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/pigs` | GET | Get all pigs |
| `/api/pigs` | POST | Create a new pig |
| `/api/pigs/:id` | GET | Get pig by ID |
| `/api/pigs/:id` | PUT | Update pig |
| `/api/pigs/:id` | DELETE | Delete pig |
| `/api/pigs/:id/health` | POST | Add health record |
| `/api/pigs/:id/bcs` | GET | Get body condition score history |

### Interaction with Other Services

- **Farm Management Service**: Pigs are located in stalls
- **Data Collection Service**: Collects posture and health data for pigs
- **Activity Logging Service**: Logs pig management activities

### Code Example

```javascript
// routes/pig.js (excerpt)
router.get('/:id', async (req, res) => {
  try {
    const id = parseInt(req.params.id, 10);
    if (isNaN(id)) {
      return res.status(400).json({ error: 'Invalid pig id' });
    }

    const pig = await Pig.findOne({ pigId: id })
      .populate('currentLocation.farmId')
      .populate('currentLocation.barnId')
      .populate('currentLocation.stallId');

    if (!pig) {
      return res.status(404).json({ error: 'Pig not found' });
    }

    res.json(pig);
  } catch (error) {
    console.error('Error fetching pig:', error);
    res.status(500).json({ error: 'Failed to fetch pig' });
  }
});

router.post('/', async (req, res) => {
  try {
    const { pigId, tag, breed, age, currentLocation } = req.body;

    // Validate pigId
    if (typeof pigId !== 'string' && typeof pigId !== 'number') {
      return res.status(400).json({ error: 'Invalid pigId' });
    }

    // Check if pig with this ID already exists
    const existingPig = await Pig.findOne({ pigId: { $eq: pigId } });
    if (existingPig) {
      return res.status(400).json({ error: 'Pig with this ID already exists' });
    }

    // Create new pig
    const newPig = await Pig.create({
      pigId: pigId,
      tag: tag,
      breed: breed,
      age: age,
      currentLocation: {
        farmId: currentLocation.farmId,
        barnId: currentLocation.barnId,
        stallId: currentLocation.stallId
      },
      active: true
    });

    res.status(201).json({
      success: true,
      pig: newPig
    });
  } catch (error) {
    console.error('Error creating pig:', error);
    res.status(500).json({ error: 'Failed to create pig' });
  }
});
```

## Analytics Service

### Responsibilities

- Generating statistics and insights
- Aggregating data for dashboards
- Providing time-series analysis

### Key Components

- **Routes**: `/api/stats/*`
- **Socket**: `socket/stats.js`

### API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/stats` | GET | Get system-wide statistics |
| `/api/stats/pigs` | GET | Get pig-related statistics |
| `/api/stats/farms` | GET | Get farm-related statistics |

### Interaction with Other Services

- **Data Collection Service**: Uses collected data for analysis
- **Pig Management Service**: Analyzes pig health and posture data
- **Farm Management Service**: Analyzes farm performance data

### Code Example

```javascript
// routes/stats.js (excerpt)
router.get('/', async (req, res) => {
  try {
    // Get counts
    const [pigCount, farmCount, barnCount, stallCount, deviceCount] = await Promise.all([
      Pig.countDocuments({ active: true }),
      Farm.countDocuments({ isActive: true }),
      Barn.countDocuments({}),
      Stall.countDocuments({}),
      Device.countDocuments({ isActive: true })
    ]);
    
    // Get health status distribution
    const healthStatusAggregation = await PigHealthStatus.aggregate([
      {
        $sort: { timestamp: -1 }
      },
      {
        $group: {
          _id: '$pigId',
          status: { $first: '$status' },
          timestamp: { $first: '$timestamp' }
        }
      },
      {
        $group: {
          _id: '$status',
          count: { $sum: 1 }
        }
      }
    ]);
    
    const healthStatus = healthStatusAggregation.reduce((acc, item) => {
      acc[item._id] = item.count;
      return acc;
    }, {
      healthy: 0,
      'at risk': 0,
      critical: 0,
      'no movement': 0
    });
    
    // Get recent activity
    const recentActivity = await ActivityLog.find({})
      .sort({ timestamp: -1 })
      .limit(5)
      .populate('userId', 'firstName lastName email');
    
    res.json({
      counts: {
        pigs: pigCount,
        farms: farmCount,
        barns: barnCount,
        stalls: stallCount,
        devices: deviceCount
      },
      healthStatus,
      recentActivity
    });
  } catch (error) {
    console.error('Error fetching stats:', error);
    res.status(500).json({ error: 'Failed to fetch stats' });
  }
});
```

## Activity Logging Service

### Responsibilities

- Logging system activities
- Providing activity history
- Supporting audit trails

### Key Components

- **Routes**: `/api/activities/*`
- **Models**: `ActivityLog`
- **Services**: `services/activityLogger.js`

### API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/activities` | GET | Get recent activities |
| `/api/activities` | POST | Log a new activity |

### Interaction with Other Services

- **All Services**: Receives activity logs from all services
- **Notification Service**: Sends activities for real-time broadcasting
- **User Management Service**: Links activities to users

### Code Example

```javascript
// services/activityLogger.js
const ActivityLog = require('../models/ActivityLog');

/**
 * Log a system activity
 * @param {Object} activityData - Activity data
 * @param {string} activityData.type - Activity type (e.g., 'user', 'pig', 'farm')
 * @param {string} activityData.action - Action performed (e.g., 'created', 'updated', 'deleted')
 * @param {string} activityData.description - Human-readable description
 * @param {string} [activityData.userId] - ID of the user who performed the action
 * @param {string} [activityData.entityId] - ID of the entity affected
 * @param {Object} [activityData.metadata] - Additional metadata
 * @returns {Promise<Object>} Created activity log
 */
const logActivity = async (activityData) => {
  try {
    const activity = new ActivityLog({
      type: activityData.type,
      action: activityData.action,
      description: activityData.description,
      userId: activityData.userId,
      entityId: activityData.entityId,
      metadata: activityData.metadata,
      ipAddress: activityData.ipAddress
    });
    
    await activity.save();
    return activity;
  } catch (error) {
    console.error('Error logging activity:', error);
    // Still return something even if logging fails
    return {
      type: activityData.type,
      action: activityData.action,
      description: activityData.description,
      timestamp: new Date()
    };
  }
};

/**
 * Get recent activities
 * @param {Object} options - Query options
 * @param {number} [options.limit=10] - Maximum number of activities to return
 * @param {string} [options.type] - Filter by activity type
 * @param {string} [options.userId] - Filter by user ID
 * @returns {Promise<Array>} Array of activity logs
 */
const getRecentActivities = async (options = {}) => {
  try {
    const { limit = 10, type, userId } = options;
    
    const query = {};
    if (type) query.type = type;
    if (userId) query.userId = userId;
    
    const activities = await ActivityLog.find(query)
      .sort({ timestamp: -1 })
      .limit(limit)
      .populate('userId', 'firstName lastName email')
      .lean();
    
    return activities;
  } catch (error) {
    console.error('Error getting recent activities:', error);
    return [];
  }
};

module.exports = {
  logActivity,
  getRecentActivities
};
```

## **API ENDPOINT COMUNICATION**

### Direct Method Calls

Services communicate primarily through direct method calls:

```javascript
// Example of direct service communication
const { logActivity } = require('../services/activityLogger');

// In a route handler
await logActivity({
  type: 'pig',
  action: 'created',
  description: `Pig ${newPig.tag} was added to the system`,
  userId: req.user.id,
  entityId: newPig._id
});
```

### Database-Mediated Communication

Services also communicate indirectly through the database:

1. **Service A** writes data to the database
2. **MongoDB Change Streams** detect the change
3. **Service B** reacts to the change

### Socket.IO for Real-time Updates

For real-time communication with clients, Socket.IO is used:

```javascript
// socket/stats.js
const emitUpdatedStats = async () => {
  try {
    const stats = await calculateStats();
    io.emit('stats_update', stats);
    return stats;
  } catch (error) {
    console.error('Error emitting updated stats:', error);
  }
};
```