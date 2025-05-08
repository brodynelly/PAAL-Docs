# Data Access Layer

## Overview

The data access layer is responsible for interacting with the MongoDB database. This document outlines the data access patterns, models, and techniques used in the system.

## MongoDB and Mongoose

The PAAL system uses MongoDB as its primary database and Mongoose as the Object Data Modeling (ODM) library. Mongoose provides a schema-based solution to model application data and includes built-in type casting, validation, query building, and logic hooks.

### Database Connection

The database connection is established in the `db/connection.js` file:

```javascript
// db/connection.js
const mongoose = require('mongoose');
const { DATABASE_CONFIG } = require('../config');

/**
 * Connect to MongoDB database
 * @returns {Promise<void>}
 */
const connectToDatabase = async () => {
  try {
    // Construct connection string
    const connectionString = `mongodb://${DATABASE_CONFIG.username}:${DATABASE_CONFIG.password}@${DATABASE_CONFIG.host}:${DATABASE_CONFIG.port}/${DATABASE_CONFIG.database}?authSource=admin`;
    
    // Connect to MongoDB
    await mongoose.connect(connectionString, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    
    console.log('Connected to MongoDB');
    
    // Register models after connection is established
    require('./models');
  } catch (error) {
    console.error('Failed to connect to MongoDB:', error);
    throw error;
  }
};

module.exports = {
  connectToDatabase
};
```

### Model Registration

Models are registered in the `db/models.js` file:

```javascript
// db/models.js
const mongoose = require('mongoose');

/**
 * Register all models
 */
const registerModels = () => {
  // Define all models to register
  const models = [
    '../models/Pig',
    '../models/PigBCS',
    '../models/PostureData',
    '../models/PigHeatStatus',
    '../models/PigHealthStatus',
    '../models/PigVulvaSwelling',
    '../models/PigBreathRate',
    '../models/Device',
    '../models/TemperatureData',
    '../models/Farm',
    '../models/Barn',
    '../models/Stall',
    '../models/User',
    '../models/ActivityLog'
  ];

  // Register each model
  models.forEach(modelPath => {
    try {
      require(modelPath);
    } catch (error) {
      console.error(`Error loading model from ${modelPath}:`, error);
    }
  });

  console.log(`Registered ${models.length} models successfully`);
};

/**
 * Set up MongoDB change streams for real-time updates
 * @param {Function} emitUpdatedStats - Function to emit updated stats
 */
const setupChangeStreams = (emitUpdatedStats) => {
  // Watch Pig model changes
  const Pig = mongoose.model('Pig');
  const pigChangeStream = Pig.watch([], { fullDocument: 'updateLookup' });
  
  pigChangeStream.on('change', async (change) => {
    await emitUpdatedStats();
    
    // Additional change stream handling...
  });
  
  // Watch other model changes...
};

module.exports = {
  registerModels,
  setupChangeStreams
};
```

## Data Models

### Core Models

The system includes several core data models:

#### Pig Model

```javascript
// models/Pig.js
const mongoose = require('mongoose');
const { Schema } = mongoose;

const PigSchema = new Schema({
  pigId: {
    type: Number,
    required: true,
    unique: true,
    index: true,
    validate: {
      validator: Number.isInteger,
      message: '{VALUE} is not an integer value'
    }
  },
  tag: { type: String, required: true, unique: true },
  breed: { type: String },
  age: { type: Number, min: 0 },
  currentLocation: {
    farmId: { type: Schema.Types.ObjectId, ref: 'Farm' },
    barnId: { type: Schema.Types.ObjectId, ref: 'Barn'},
    stallId: { type: Schema.Types.ObjectId, ref: 'Stall' }
  },
  active: { type: Boolean, default: true }
}, { 
  timestamps: true,
  toJSON: { virtuals: true },
  toObject: { virtuals: true }
});

// Indexes for efficient queries
PigSchema.index({ 'currentLocation.farmId': 1 });
PigSchema.index({ 'currentLocation.barnId': 1 });
PigSchema.index({ 'currentLocation.stallId': 1 });

module.exports = mongoose.model('Pig', PigSchema);
```

#### Farm Model

```javascript
// models/Farm.js
const mongoose = require('mongoose');
const { Schema } = mongoose;

const FarmSchema = new Schema({
  name: { 
    type: String, 
    required: true,
    trim: true
  },
  location: { 
    type: String, 
    required: true,
    trim: true
  },
  description: { 
    type: String,
    default: ''
  },
  isActive: { 
    type: Boolean, 
    default: true 
  }
}, { 
  timestamps: true 
});

module.exports = mongoose.model('Farm', FarmSchema);
```

#### User Model

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcrypt');
const { Schema } = mongoose;

const UserSchema = new Schema({
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true
  },
  password: {
    type: String,
    required: true
  },
  firstName: {
    type: String,
    required: true,
    trim: true
  },
  lastName: {
    type: String,
    required: true,
    trim: true
  },
  role: {
    type: String,
    enum: ['admin', 'manager', 'farmer', 'viewer'],
    default: 'farmer'
  },
  isActive: {
    type: Boolean,
    default: true
  },
  lastLogin: {
    type: Date
  },
  assignedFarm: {
    type: Schema.Types.ObjectId,
    ref: 'Farm'
  },
  permissions: {
    type: [String],
    default: []
  }
}, {
  timestamps: true
});

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

module.exports = mongoose.model('User', UserSchema);
```

### Data Collection Models

The system includes several models for collecting and storing sensor data:

#### Posture Data Model

```javascript
// models/PostureData.js
const mongoose = require('mongoose');
const { Schema } = mongoose;

const PigPostureSchema = new Schema({
  pigId: { type: Number, required: true },
  timestamp: { type: Date, default: Date.now },
  score: { type: Number, min: 0, max: 5, required: true }
}, {
  toJSON: { virtuals: true, getters: true },
  toObject: { virtuals: true, getters: true }
});

// Add a virtual property for formatted date
PigPostureSchema.virtual('formattedDate').get(function() {
  return this.timestamp ? this.timestamp.toISOString().split('T')[0] : '';
});

// Indexes for efficient queries
PigPostureSchema.index({ pigId: 1 });
PigPostureSchema.index({ timestamp: -1 });
PigPostureSchema.index({ pigId: 1, timestamp: -1 });

module.exports = mongoose.model('PigPosture', PigPostureSchema);
```

## Data Access Patterns

### Basic CRUD Operations

The system uses Mongoose's built-in methods for basic CRUD operations:

#### Create

```javascript
// Example: Creating a new pig
const newPig = await Pig.create({
  pigId: 1001,
  tag: 'PIG-1001',
  breed: 'Yorkshire',
  age: 12,
  currentLocation: {
    farmId: farmId,
    barnId: barnId,
    stallId: stallId
  },
  active: true
});
```

#### Read

```javascript
// Example: Finding a pig by ID
const pig = await Pig.findOne({ pigId: id })
  .populate('currentLocation.farmId')
  .populate('currentLocation.barnId')
  .populate('currentLocation.stallId');

// Example: Finding all pigs with pagination
const page = parseInt(req.query.page) || 1;
const limit = parseInt(req.query.limit) || 10;
const skip = (page - 1) * limit;

const pigs = await Pig.find({ active: true })
  .sort({ updatedAt: -1 })
  .skip(skip)
  .limit(limit);
```

#### Update

```javascript
// Example: Updating a pig
const updatedPig = await Pig.findOneAndUpdate(
  { pigId: id },
  {
    breed: req.body.breed,
    age: req.body.age,
    currentLocation: {
      farmId: req.body.currentLocation.farmId,
      barnId: req.body.currentLocation.barnId,
      stallId: req.body.currentLocation.stallId
    }
  },
  { new: true, runValidators: true }
);
```

#### Delete

```javascript
// Example: Soft delete (marking as inactive)
await Pig.findOneAndUpdate(
  { pigId: id },
  { active: false },
  { new: true }
);

// Example: Hard delete (removing from database)
await Pig.findOneAndDelete({ pigId: id });
```

### Advanced Query Patterns

#### Aggregation Pipeline

The system uses MongoDB's aggregation pipeline for complex data transformations:

```javascript
// Example: Aggregating posture data by day
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
```

#### Population and Joins

The system uses Mongoose's populate method to perform "joins" between collections:

```javascript
// Example: Populating related data
const farm = await Farm.findById(req.params.id);

const barns = await Barn.find({ farmId: farm._id })
  .populate({
    path: 'stalls',
    select: 'name occupancy'
  });

const pigs = await Pig.find({ 'currentLocation.farmId': farm._id })
  .populate('currentLocation.stallId')
  .populate({
    path: 'healthStatus',
    options: { sort: { timestamp: -1 }, limit: 1 }
  });
```

### Transactions

For operations that require atomicity, the system uses MongoDB transactions:

```javascript
// Example: Using transactions for atomic operations
const session = await mongoose.startSession();
session.startTransaction();

try {
  // Create a new farm
  const farm = await Farm.create([{
    name: req.body.name,
    location: req.body.location,
    description: req.body.description,
    isActive: true
  }], { session });
  
  // Create a default barn for the farm
  const barn = await Barn.create([{
    name: 'Default Barn',
    farmId: farm[0]._id,
    description: 'Default barn created with farm'
  }], { session });
  
  // Create a default stall for the barn
  await Stall.create([{
    name: 'Default Stall',
    farmId: farm[0]._id,
    barnId: barn[0]._id,
    capacity: 10,
    description: 'Default stall created with farm'
  }], { session });
  
  // Commit the transaction
  await session.commitTransaction();
  session.endSession();
  
  res.status(201).json(farm[0]);
} catch (error) {
  // Abort the transaction on error
  await session.abortTransaction();
  session.endSession();
  
  console.error('Error creating farm with default structures:', error);
  res.status(500).json({ error: 'Failed to create farm' });
}
```

## Indexing Strategy

The system uses indexes to improve query performance:

### Single Field Indexes

```javascript
// Example: Single field index on pigId
PigSchema.index({ pigId: 1 });

// Example: Single field index on timestamp with descending order
PigPostureSchema.index({ timestamp: -1 });
```

### Compound Indexes

```javascript
// Example: Compound index on pigId and timestamp
PigPostureSchema.index({ pigId: 1, timestamp: -1 });

// Example: Compound index on location fields
PigSchema.index({ 'currentLocation.farmId': 1, 'currentLocation.barnId': 1 });
```

### Text Indexes

```javascript
// Example: Text index for search functionality
FarmSchema.index({ name: 'text', location: 'text', description: 'text' });
```

## Data Validation

### Schema-Level Validation

The system uses Mongoose's built-in validation features:

```javascript
// Example: Validation in Pig schema
const PigSchema = new Schema({
  pigId: {
    type: Number,
    required: true,
    unique: true,
    validate: {
      validator: Number.isInteger,
      message: '{VALUE} is not an integer value'
    }
  },
  age: { 
    type: Number, 
    min: 0 
  },
  // Other fields...
});
```

### Custom Validators

```javascript
// Example: Custom validator for email
const UserSchema = new Schema({
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true,
    validate: {
      validator: function(v) {
        return /^\w+([\.-]?\w+)*@\w+([\.-]?\w+)*(\.\w{2,3})+$/.test(v);
      },
      message: props => `${props.value} is not a valid email address!`
    }
  },
  // Other fields...
});
```

## Change Streams

The system uses MongoDB Change Streams to monitor and react to data changes in real-time:

```javascript
// Example: Setting up a change stream for the Pig model
const setupPigChangeStream = (io) => {
  const Pig = mongoose.model('Pig');
  const pigChangeStream = Pig.watch([], { fullDocument: 'updateLookup' });
  
  pigChangeStream.on('change', async (change) => {
    // React to changes
    if (change.operationType === 'insert') {
      // A new pig was added
      io.emit('pig_added', change.fullDocument);
      
      // Log the activity
      await logActivity({
        type: 'pig',
        action: 'created',
        description: `Pig ${change.fullDocument.tag} was added to the system`,
        entityId: change.fullDocument._id
      });
    } else if (change.operationType === 'update') {
      // A pig was updated
      io.emit('pig_updated', change.fullDocument);
      
      // Log the activity
      await logActivity({
        type: 'pig',
        action: 'updated',
        description: `Pig ${change.fullDocument.tag} was updated`,
        entityId: change.fullDocument._id
      });
    } else if (change.operationType === 'delete') {
      // A pig was deleted
      io.emit('pig_deleted', change.documentKey._id);
      
      // Log the activity
      await logActivity({
        type: 'pig',
        action: 'deleted',
        description: `A pig was removed from the system`,
        entityId: change.documentKey._id
      });
    }
  });
  
  return pigChangeStream;
};
```

## Error Handling

The system implements consistent error handling for database operations:

```javascript
// Example: Error handling for database operations
try {
  const pig = await Pig.findOne({ pigId: id });
  
  if (!pig) {
    return res.status(404).json({ error: 'Pig not found' });
  }
  
  // Process pig data...
} catch (error) {
  console.error('Error fetching pig:', error);
  
  // Determine appropriate error response
  if (error.name === 'CastError') {
    return res.status(400).json({ error: 'Invalid pig ID format' });
  } else if (error.name === 'ValidationError') {
    return res.status(400).json({ error: error.message });
  } else {
    return res.status(500).json({ error: 'Database error' });
  }
}
```

