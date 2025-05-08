# Business Logic Layer

## Overview

The business logic layer of the PAAL system contains the core application logic that processes data, enforces business rules, and coordinates interactions between different parts of the system. This document outlines the key business logic components, their responsibilities, and how they are implemented.

## Core Business Logic Components

The business logic in the PAAL system is primarily implemented in the following components:

1. **Route Handlers**: Process API requests and implement business rules
2. **Service Modules**: Encapsulate reusable business logic
3. **Model Methods**: Implement entity-specific business logic
4. **Middleware**: Enforce cross-cutting concerns like authentication and authorization

## Pig Management Logic

### Pig Data Processing

The system processes pig data to track health, location, and various metrics:

```javascript
// routes/pig.js (excerpt)
router.get('/', async (req, res) => {
  try {
    const pigs = await Pig.find({})
      .populate('currentLocation.farmId')
      .populate('currentLocation.barnId')
      .populate('currentLocation.stallId')
      .populate('healthStatus')
      .sort({ updatedAt: -1 });

    // Transform data for response - this is business logic that formats
    // raw database data into a client-friendly format
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

### Posture Data Aggregation

The system aggregates posture data to provide insights into pig behavior:

```javascript
// routes/pig.js (excerpt)
router.get('/:id/posture/aggregated', async (req, res) => {
  try {
    const id = parseInt(req.params.id, 10);
    if (isNaN(id)) {
      return res.status(400).json({ error: 'Invalid pig id' });
    }

    // Get date range from query parameters
    const { start, end } = req.query;
    
    // Default to last 7 days if no date range provided
    const startDate = start ? new Date(start) : new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
    const endDate = end ? new Date(end) : new Date();
    
    // Set end date to end of day
    endDate.setHours(23, 59, 59, 999);
    
    // Validate date range
    if (isNaN(startDate.getTime()) || isNaN(endDate.getTime())) {
      return res.status(400).json({ error: 'Invalid date range' });
    }
    
    // Aggregate posture data by day
    const postureData = await PigPosture.aggregate([
      {
        $match: {
          pigId: id,
          timestamp: { $gte: startDate, $lte: endDate }
        }
      },
      {
        $group: {
          _id: { $dateToString: { format: '%Y-%m-%d', date: '$timestamp' } },
          scores: { $push: '$score' }
        }
      },
      {
        $project: {
          _id: 0,
          date: '$_id',
          // Calculate percentage of each posture type
          standing: {
            $multiply: [
              {
                $divide: [
                  { $size: { $filter: { input: '$scores', as: 'score', cond: { $eq: ['$$score', 0] } } } },
                  { $size: '$scores' }
                ]
              },
              100
            ]
          },
          sitting: {
            $multiply: [
              {
                $divide: [
                  { $size: { $filter: { input: '$scores', as: 'score', cond: { $eq: ['$$score', 1] } } } },
                  { $size: '$scores' }
                ]
              },
              100
            ]
          },
          lying: {
            $multiply: [
              {
                $divide: [
                  { $size: { $filter: { input: '$scores', as: 'score', cond: { $eq: ['$$score', 2] } } } },
                  { $size: '$scores' }
                ]
              },
              100
            ]
          }
        }
      },
      {
        $sort: { date: 1 }
      }
    ]);
    
    res.json(postureData);
  } catch (error) {
    console.error('Error fetching aggregated posture data:', error);
    res.status(500).json({ error: 'Failed to fetch posture data' });
  }
});
```

### Health Risk Calculation

The system calculates health risk scores based on various factors:

```javascript
// services/healthRiskCalculator.js
/**
 * Calculate health risk score for a pig
 * @param {Object} pigData - Pig data including health metrics
 * @returns {number} Risk score between 0 and 1
 */
const calculateHealthRisk = (pigData) => {
  // Initialize with base risk
  let riskScore = 0;
  
  // Factor 1: Recent posture data
  if (pigData.recentPosture) {
    // Higher risk if pig is lying down too much
    const lyingPercentage = pigData.recentPosture.lying || 0;
    if (lyingPercentage > 70) {
      riskScore += 0.3;
    } else if (lyingPercentage > 50) {
      riskScore += 0.1;
    }
  }
  
  // Factor 2: Body condition score
  if (pigData.bcsScore) {
    // Higher risk if BCS is too low or too high
    if (pigData.bcsScore < 2 || pigData.bcsScore > 4) {
      riskScore += 0.2;
    }
  }
  
  // Factor 3: Recent health status
  if (pigData.healthStatus) {
    if (pigData.healthStatus === 'critical') {
      riskScore += 0.5;
    } else if (pigData.healthStatus === 'at risk') {
      riskScore += 0.3;
    }
  }
  
  // Factor 4: Age
  if (pigData.age) {
    if (pigData.age > 5) {
      riskScore += 0.1;
    }
  }
  
  // Cap risk score at 1
  return Math.min(riskScore, 1);
};

module.exports = {
  calculateHealthRisk
};
```

## Farm Management Logic

### Farm Data Aggregation

The system aggregates data about farms, barns, stalls, and pigs:

```javascript
// routes/farm.js (excerpt)
router.get('/:id', authenticateJWT, async (req, res) => {
  try {
    // If user is a farmer, they can only access their assigned farm
    if (req.user.role === 'farmer' && req.user.assignedFarm) {
      const farmId = req.user.assignedFarm.toString();
      const requestedFarmId = req.params.id;
      
      if (farmId !== requestedFarmId) {
        return res.status(403).json({ error: 'You are not authorized to access this farm' });
      }
    }
    
    const farm = await Farm.findById(req.params.id);
    if (!farm) {
      return res.status(404).json({ error: 'Farm not found' });
    }
    
    // Get related data
    const [barns, stalls, pigs, devices] = await Promise.all([
      Barn.find({ farmId: farm._id }),
      Stall.find({ farmId: farm._id }),
      Pig.find({ 'currentLocation.farmId': farm._id })
        .sort({ updatedAt: -1 })
        .limit(10),
      Device.find({ farmId: farm._id })
        .sort({ updatedAt: -1 })
        .limit(10)
    ]);
    
    // Get health status counts
    const healthStatusCount = {
      healthy: 0,
      atRisk: 0,
      critical: 0
    };
    
    // Get the most recent health status for each pig
    const pigHealthStatuses = await PigHealthStatus.aggregate([
      {
        $match: {
          pigId: { $in: pigs.map(pig => pig.pigId) }
        }
      },
      {
        $sort: { timestamp: -1 }
      },
      {
        $group: {
          _id: '$pigId',
          status: { $first: '$status' }
        }
      }
    ]);
    
    // Count pigs by health status
    pigHealthStatuses.forEach(status => {
      if (status.status === 'healthy') {
        healthStatusCount.healthy++;
      } else if (status.status === 'at risk') {
        healthStatusCount.atRisk++;
      } else if (status.status === 'critical') {
        healthStatusCount.critical++;
      }
    });
    
    res.json({
      ...farm.toObject(),
      counts: {
        barns: barns.length,
        stalls: stalls.length,
        pigs: pigs.length,
        devices: devices.length
      },
      healthStatus: healthStatusCount,
      barns: barns,
      recentPigs: pigs,
      devices: devices
    });
  } catch (error) {
    console.error('Error fetching farm:', error);
    res.status(500).json({ error: 'Failed to fetch farm' });
  }
});
```

### Farm Access Control

The system enforces access control rules for farms:

```javascript
// middleware/farmAccessMiddleware.js
/**
 * Middleware to check if user has access to a farm
 * @param {Object} req - Express request object
 * @param {Object} res - Express response object
 * @param {Function} next - Express next function
 */
const checkFarmAccess = async (req, res, next) => {
  try {
    // Admin has access to all farms
    if (req.user.role === 'admin') {
      return next();
    }
    
    // Get farm ID from request
    const farmId = req.params.farmId || req.params.id || req.body.farmId;
    
    if (!farmId) {
      return next();
    }
    
    // Check if user is assigned to this farm
    if (req.user.assignedFarm && req.user.assignedFarm.toString() === farmId.toString()) {
      return next();
    }
    
    // Check if user has permission to access this farm
    if (req.user.permissions && req.user.permissions.includes('access:all-farms')) {
      return next();
    }
    
    // User doesn't have access to this farm
    return res.status(403).json({ error: 'You do not have access to this farm' });
  } catch (error) {
    console.error('Error checking farm access:', error);
    return res.status(500).json({ error: 'Server error' });
  }
};

module.exports = {
  checkFarmAccess
};
```

## User Management Logic

### User Authentication

The system authenticates users and generates JWT tokens:

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

### Password Hashing

The system securely hashes passwords using bcrypt:

```javascript
// models/User.js (excerpt)
// Pre-save hook to hash password
UserSchema.pre('save', async function(next) {
  const user = this;
  
  // Only hash the password if it has been modified or is new
  if (!user.isModified('password')) {
    return next();
  }
  
  try {
    // Generate salt
    const salt = await bcrypt.genSalt(10);
    
    // Hash password
    const hash = await bcrypt.hash(user.password, salt);
    
    // Replace plain text password with hash
    user.password = hash;
    next();
  } catch (error) {
    return next(error);
  }
});

// Method to compare password
UserSchema.methods.comparePassword = async function(candidatePassword) {
  try {
    return await bcrypt.compare(candidatePassword, this.password);
  } catch (error) {
    throw error;
  }
};
```

## Activity Logging Logic

The system logs activities for audit and tracking purposes:

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
```

## Statistics Generation Logic

The system generates statistics for dashboards and reports:

```javascript
// socket/stats.js (excerpt)
/**
 * Calculate system statistics
 * @returns {Promise<Object>} System statistics
 */
const calculateStats = async () => {
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
    
    return {
      counts: {
        pigs: pigCount,
        farms: farmCount,
        barns: barnCount,
        stalls: stallCount,
        devices: deviceCount
      },
      healthStatus,
      recentActivity
    };
  } catch (error) {
    console.error('Error calculating stats:', error);
    throw error;
  }
};
```

## Business Rules and Validation

### Input Validation

The system validates input data before processing:

```javascript
// routes/pig.js (excerpt)
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

    // Validate tag
    if (!tag || typeof tag !== 'string') {
      return res.status(400).json({ error: 'Valid tag is required' });
    }

    // Validate age if provided
    if (age !== undefined && (isNaN(age) || age < 0)) {
      return res.status(400).json({ error: 'Age must be a non-negative number' });
    }

    // Validate location
    if (!currentLocation || !currentLocation.farmId) {
      return res.status(400).json({ error: 'Farm ID is required' });
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

### Business Rules Enforcement

The system enforces business rules such as role-based access control:

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

## Error Handling

The system implements consistent error handling:

```javascript
// Generic error handling pattern used throughout the application
try {
  // Operation that might fail
} catch (error) {
  console.error('Error description:', error);
  
  // Determine appropriate status code
  let statusCode = 500;
  let errorMessage = 'An unexpected error occurred';
  
  if (error.name === 'ValidationError') {
    statusCode = 400;
    errorMessage = 'Validation error';
  } else if (error.name === 'CastError') {
    statusCode = 400;
    errorMessage = 'Invalid ID format';
  } else if (error.code === 11000) {
    statusCode = 409;
    errorMessage = 'Duplicate entry';
  }
  
  // Send error response
  res.status(statusCode).json({ 
    error: errorMessage,
    details: process.env.NODE_ENV === 'development' ? error.message : undefined
  });
}
```

## Transaction Management

For operations that require atomicity, the system uses MongoDB transactions:

```javascript
// Example of transaction usage
const session = await mongoose.startSession();
session.startTransaction();

try {
  // Perform multiple operations that should be atomic
  const farm = await Farm.create([{ name: 'New Farm', location: 'Location' }], { session });
  const barn = await Barn.create([{ name: 'New Barn', farmId: farm[0]._id }], { session });
  
  // If all operations succeed, commit the transaction
  await session.commitTransaction();
  session.endSession();
  
  return { farm: farm[0], barn: barn[0] };
} catch (error) {
  // If any operation fails, abort the transaction
  await session.abortTransaction();
  session.endSession();
  throw error;
}
```

## Conclusion

The business logic layer of the PAAL system is designed to be modular, maintainable, and secure. It enforces business rules, processes data, and coordinates interactions between different parts of the system. The logic is primarily implemented in route handlers, service modules, model methods, and middleware, with a focus on separation of concerns and reusability.
