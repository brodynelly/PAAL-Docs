# Security Measures

## Overview

The PAAL system implements comprehensive security measures to protect sensitive data and ensure secure access to the application. This document outlines the security architecture, authentication mechanisms, authorization rules, and other security practices implemented in the system.

## Security Architecture

The PAAL system follows a defense-in-depth approach with multiple layers of security:

```
┌─────────────────────────────────────────────────────────────┐
│                     Application Layer                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ Input       │  │ Business    │  │ Output              │  │
│  │ Validation  │  │ Logic       │  │ Encoding            │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│                     Access Control Layer                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ Authenti-   │  │ Authori-    │  │ Session             │  │
│  │ cation      │  │ zation      │  │ Management          │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│                     Transport Layer                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ HTTPS/TLS   │  │ API         │  │ Network             │  │
│  │ Encryption  │  │ Gateway     │  │ Segmentation        │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│                     Data Layer                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ Encryption  │  │ Backup      │  │ Data                │  │
│  │ at Rest     │  │ Security    │  │ Minimization        │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Security Principles

The system is built on the following security principles:

1. **Defense in Depth**: Multiple layers of security controls
2. **Principle of Least Privilege**: Users and processes have only the permissions they need
3. **Secure by Default**: Security is built into the system from the beginning
4. **Fail Securely**: Default to secure state on failure
5. **Complete Mediation**: Every access to resources is checked for authorization
6. **Separation of Duties**: Critical actions require multiple approvals
7. **Keep Security Simple**: Simple security mechanisms are easier to verify

## Authentication Implementation

### JWT-Based Authentication

The system uses JSON Web Tokens (JWT) for authentication:

#### Token Generation

```javascript
// Token generation during login
const generateToken = (user) => {
  return jwt.sign(
    {
      id: user._id,
      email: user.email,
      role: user.role,
      permissions: user.permissions
    },
    process.env.JWT_SECRET,
    { expiresIn: '24h' }
  );
};
```

#### Token Validation

```javascript
// Middleware for validating JWT tokens
const authenticateJWT = (req, res, next) => {
  const authHeader = req.headers.authorization;

  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Authentication required' });
  }

  const token = authHeader.split(' ')[1];

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    if (error.name === 'TokenExpiredError') {
      return res.status(401).json({ error: 'Token has expired' });
    }
    return res.status(401).json({ error: 'Invalid token' });
  }
};
```

### Password Security

Passwords are securely hashed using bcrypt:

```javascript
// Password hashing in User model
UserSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();

  try {
    const salt = await bcrypt.genSalt(10);
    this.password = await bcrypt.hash(this.password, salt);
    next();
  } catch (error) {
    next(error);
  }
});

// Password verification
UserSchema.methods.comparePassword = async function(candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};
```

### Session Management

The system implements secure session management:

- **Token Expiration**: JWTs expire after 24 hours
- **Token Storage**: Tokens are stored in localStorage on the client
- **Token Refresh**: Refresh tokens can be implemented for extended sessions
- **Session Termination**: Logout functionality clears tokens

## Authorization Rules

### Role-Based Access Control (RBAC)

The system implements role-based access control with the following roles:

| Role | Description | Access Level |
|------|-------------|-------------|
| Admin | System administrator | Full access to all resources |
| Manager | Farm manager | Access to assigned farms and all their resources |
| Farmer | Farm worker | Limited access to assigned farms |
| Viewer | Read-only user | Read-only access to assigned resources |

#### Role Middleware

```javascript
// Middleware for checking admin role
const isAdmin = (req, res, next) => {
  if (!req.user) {
    return res.status(401).json({ error: 'Authentication required' });
  }

  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }

  next();
};

// Middleware for checking manager role
const isManager = (req, res, next) => {
  if (!req.user) {
    return res.status(401).json({ error: 'Authentication required' });
  }

  if (req.user.role !== 'admin' && req.user.role !== 'manager') {
    return res.status(403).json({ error: 'Manager access required' });
  }

  next();
};
```

### Permission-Based Access Control

For more granular control, the system also implements permission-based access:

```javascript
// Middleware for checking specific permissions
const hasPermission = (permission) => {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    if (req.user.role === 'admin') {
      // Admins have all permissions
      return next();
    }

    if (!req.user.permissions || !req.user.permissions.includes(permission)) {
      return res.status(403).json({ error: `Permission '${permission}' required` });
    }

    next();
  };
};

// Usage in routes
router.post('/farms', authenticateJWT, hasPermission('create:farms'), createFarm);
```

### Resource-Based Access Control

The system restricts access to resources based on ownership and assignment:

```javascript
// Middleware for checking farm access
const checkFarmAccess = async (req, res, next) => {
  try {
    const farmId = req.params.id;

    // Admins have access to all farms
    if (req.user.role === 'admin') {
      return next();
    }

    // Check if user is assigned to this farm
    if (req.user.assignedFarm && req.user.assignedFarm.toString() === farmId) {
      return next();
    }

    // Check if user has explicit permission for this farm
    const userFarmAccess = await UserFarmAccess.findOne({
      userId: req.user.id,
      farmId: farmId
    });

    if (userFarmAccess) {
      return next();
    }

    return res.status(403).json({ error: 'You do not have access to this farm' });
  } catch (error) {
    next(error);
  }
};
```

## Input Validation and Sanitization

### Request Validation

The system validates all input data using validation middleware:

```javascript
// Validation middleware using express-validator
const validatePigInput = [
  body('tag').notEmpty().withMessage('Tag is required'),
  body('breed').optional(),
  body('age').optional().isInt({ min: 0 }).withMessage('Age must be a positive number'),
  body('currentLocation.farmId').notEmpty().withMessage('Farm ID is required'),

  // Validation handler
  (req, res, next) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }
    next();
  }
];

// Usage in routes
router.post('/pigs', authenticateJWT, validatePigInput, createPig);
```

### Data Sanitization

Input data is sanitized to prevent injection attacks:

```javascript
// Sanitization middleware
const sanitizePigInput = [
  body('tag').trim().escape(),
  body('breed').trim().escape(),
  body('notes').trim()
];

// Usage in routes
router.post('/pigs', authenticateJWT, sanitizePigInput, validatePigInput, createPig);
```

## Protection Against Common Attacks

### CSRF Protection

Cross-Site Request Forgery protection is implemented:

```javascript
// CSRF protection middleware
const csrf = require('csurf');
const csrfProtection = csrf({ cookie: true });

// Apply to routes that handle state changes
app.post('/api/auth/login', csrfProtection, loginHandler);
app.post('/api/pigs', csrfProtection, createPigHandler);
```

### XSS Prevention

Cross-Site Scripting prevention measures:

1. **Content Security Policy**: Restricts sources of executable scripts
   ```javascript
   // CSP middleware
   app.use((req, res, next) => {
     res.setHeader(
       'Content-Security-Policy',
       "default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data:;"
     );
     next();
   });
   ```

2. **Output Encoding**: All dynamic content is properly encoded before rendering
   ```javascript
   // In frontend components
   <div>{sanitizeHtml(pig.notes)}</div>
   ```

3. **HttpOnly Cookies**: Prevents JavaScript access to cookies
   ```javascript
   // Cookie configuration
   app.use(cookieParser());
   app.use(session({
     secret: process.env.SESSION_SECRET,
     cookie: {
       httpOnly: true,
       secure: process.env.NODE_ENV === 'production',
       sameSite: 'strict'
     }
   }));
   ```

### SQL Injection Prevention

MongoDB injection is prevented through:

1. **Parameterized Queries**: Using Mongoose models and schemas
   ```javascript
   // Safe query using Mongoose
   const pig = await Pig.findOne({ pigId: pigId });
   ```

2. **Input Validation**: Validating and sanitizing all inputs
   ```javascript
   // Validate ID before using in query
   const id = parseInt(req.params.id, 10);
   if (isNaN(id)) {
     return res.status(400).json({ error: 'Invalid ID format' });
   }
   ```

3. **Schema Validation**: Enforcing data types and constraints
   ```javascript
   const PigSchema = new Schema({
     pigId: {
       type: Number,
       required: true,
       validate: {
         validator: Number.isInteger,
         message: '{VALUE} is not an integer value'
       }
     }
   });
   ```

## Security Headers

The application sets various security headers:

```javascript
// Security headers middleware
const helmet = require('helmet');
app.use(helmet());

// Additional custom headers
app.use((req, res, next) => {
  // Strict Transport Security
  res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');

  // Content Type Options
  res.setHeader('X-Content-Type-Options', 'nosniff');

  // Frame Options
  res.setHeader('X-Frame-Options', 'DENY');

  // XSS Protection
  res.setHeader('X-XSS-Protection', '1; mode=block');

  // Referrer Policy
  res.setHeader('Referrer-Policy', 'same-origin');

  next();
});
```

## Rate Limiting

Rate limiting is implemented to prevent brute force attacks:

```javascript
// Rate limiting middleware
const rateLimit = require('express-rate-limit');

// Login rate limit
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 requests per window
  message: 'Too many login attempts, please try again after 15 minutes',
  standardHeaders: true,
  legacyHeaders: false,
});

// API rate limit
const apiLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 100, // 100 requests per minute
  message: 'Too many requests, please try again later',
  standardHeaders: true,
  legacyHeaders: false,
});

// Apply to routes
app.post('/api/auth/login', loginLimiter, loginHandler);
app.use('/api/', apiLimiter);
```

## Encryption Implementation

### Data Encryption

Sensitive data is encrypted in the database:

```javascript
// Encryption utility
const crypto = require('crypto');

const encrypt = (text) => {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv(
    'aes-256-cbc',
    Buffer.from(process.env.ENCRYPTION_KEY, 'hex'),
    iv
  );

  let encrypted = cipher.update(text);
  encrypted = Buffer.concat([encrypted, cipher.final()]);

  return iv.toString('hex') + ':' + encrypted.toString('hex');
};

const decrypt = (text) => {
  const parts = text.split(':');
  const iv = Buffer.from(parts[0], 'hex');
  const encryptedText = Buffer.from(parts[1], 'hex');

  const decipher = crypto.createDecipheriv(
    'aes-256-cbc',
    Buffer.from(process.env.ENCRYPTION_KEY, 'hex'),
    iv
  );

  let decrypted = decipher.update(encryptedText);
  decrypted = Buffer.concat([decrypted, decipher.final()]);

  return decrypted.toString();
};

// Usage in schema
const PigSchema = new Schema({
  // Other fields...
  medicalNotes: {
    type: String,
    set: function(value) {
      if (!value) return value;
      return encrypt(value);
    },
    get: function(value) {
      if (!value) return value;
      return decrypt(value);
    }
  }
});
```

### Transport Layer Security

All communication is secured using HTTPS:

```javascript
// HTTPS server setup
const https = require('https');
const fs = require('fs');

const options = {
  key: fs.readFileSync('path/to/private.key'),
  cert: fs.readFileSync('path/to/certificate.crt')
};

const server = https.createServer(options, app);
server.listen(443, () => {
  console.log('HTTPS server running on port 443');
});
```

## Logging and Monitoring

### Security Logging

Security-relevant events are logged:

```javascript
// Security logger
const winston = require('winston');

const securityLogger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  defaultMeta: { service: 'security-service' },
  transports: [
    new winston.transports.File({ filename: 'logs/security.log' })
  ]
});

// Log authentication attempts
const logAuthAttempt = (req, success, userId = null) => {
  securityLogger.info({
    event: 'authentication_attempt',
    success,
    userId,
    ip: req.ip,
    userAgent: req.headers['user-agent'],
    timestamp: new Date().toISOString()
  });
};

// Log access to sensitive resources
const logResourceAccess = (req, resourceType, resourceId) => {
  securityLogger.info({
    event: 'resource_access',
    userId: req.user?.id,
    resourceType,
    resourceId,
    method: req.method,
    ip: req.ip,
    timestamp: new Date().toISOString()
  });
};
```

### Security Monitoring

The system implements security monitoring:

1. **Failed Login Monitoring**: Alerts on multiple failed login attempts
2. **Unusual Access Patterns**: Detects and alerts on unusual access patterns
3. **Error Rate Monitoring**: Monitors for spikes in error rates
4. **Dependency Vulnerability Scanning**: Regular scanning of dependencies for vulnerabilities

## Security Configuration

Security settings are managed through environment variables:

```
# .env.example
# JWT Configuration
JWT_SECRET=your-secret-key-at-least-32-chars-long
JWT_EXPIRATION=24h

# Encryption
ENCRYPTION_KEY=32-char-hex-encryption-key

# Security Headers
ENABLE_CSP=true
CSP_REPORT_URI=https://example.com/csp-report

# Rate Limiting
RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX_REQUESTS=100

# CORS Configuration
CORS_ORIGIN=https://example.com
```

## Security Best Practices

1. **Principle of Least Privilege**: Users and processes have only the permissions they need
2. **Defense in Depth**: Multiple layers of security controls
3. **Fail Securely**: Default to secure state on failure
4. **Complete Mediation**: Every access to resources is checked for authorization
5. **Separation of Duties**: Critical actions require multiple approvals
6. **Keep Security Simple**: Simple security mechanisms are easier to verify
7. **Security by Design**: Security built into the system from the beginning

## Security Testing

The system undergoes regular security testing:

1. **Automated Security Scanning**: Regular automated security scans
2. **Penetration Testing**: Periodic penetration testing by security professionals
3. **Dependency Scanning**: Regular scanning of dependencies for vulnerabilities
4. **Code Reviews**: Security-focused code reviews for all changes
