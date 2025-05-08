# Code Structure and Conventions

## Overview

This document outlines the code organization, naming conventions, and coding style guidelines for the PAAL system. Following these conventions ensures consistency across the codebase and makes it easier for developers to understand and maintain the code.

## Repository Structure

The PAAL system is organized as a monorepo with separate directories for the frontend and backend components:

```
paal/
├── documentation/        # Project documentation
├── server/               # Backend server code
└── src/                  # Frontend application code
```

## Backend Code Structure

The backend code is organized in a modular structure by feature and responsibility:

```
server/
├── config/               # Configuration settings
│   └── index.js          # Exports configuration objects
├── db/                   # Database connection and models
│   ├── connection.js     # Manages MongoDB connection
│   └── models.js         # Registers all Mongoose models
├── middleware/           # Express middleware
│   ├── authMiddleware.js # Authentication middleware
│   ├── role.js           # Role-based access control
│   └── index.js          # Configures middleware
├── models/               # Mongoose models
│   ├── Pig.js            # Pig data model
│   ├── Farm.js           # Farm data model
│   ├── Barn.js           # Barn data model
│   ├── Stall.js          # Stall data model
│   ├── User.js           # User data model
│   ├── PostureData.js    # Pig posture data model
│   └── ...               # Other data models
├── routes/               # API routes
│   ├── pig.js            # Pig-related endpoints
│   ├── farm.js           # Farm-related endpoints
│   ├── auth.js           # Authentication endpoints
│   ├── user.js           # User management endpoints
│   └── ...               # Other route files
├── scripts/              # Utility scripts
│   ├── seed-all.js       # Database seeding script
│   └── ...               # Other scripts
├── services/             # Business logic services
│   ├── activityLogger.js # Activity logging service
│   └── ...               # Other services
├── socket/               # Socket.IO related code
│   ├── events.js         # Socket event handlers
│   ├── stats.js          # Statistics emission
│   └── index.js          # Socket.IO initialization
├── utils/                # Utility functions
│   └── routeValidator.js # Route validation utility
├── .env                  # Environment variables (not in version control)
├── .env.example          # Example environment variables
├── index.js              # Legacy entry point
├── server.js             # Main entry point
├── package.json          # Node.js dependencies
└── README.md             # Backend documentation
```

## Frontend Code Structure

The frontend code follows Next.js's file-based routing structure with additional organization by feature:

```
src/
├── app/                  # Next.js app directory (routes and pages)
│   ├── (auth)/           # Authentication-related routes
│   │   ├── login/        # Login page
│   │   └── ...           # Other auth pages
│   ├── (main)/           # Main application routes
│   │   ├── overview/     # Dashboard overview page
│   │   ├── details/      # Detailed data views
│   │   ├── admin/        # Admin panel pages
│   │   ├── pigs/         # Pig management pages
│   │   │   ├── [id]/     # Dynamic route for pig details
│   │   │   └── ...       # Other pig pages
│   │   └── ...           # Other main pages
│   ├── api/              # API routes for server-side operations
│   └── layout.tsx        # Root layout component
├── components/           # Reusable UI components
│   ├── ui/               # Basic UI components
│   │   ├── Button.tsx    # Button component
│   │   ├── Card.tsx      # Card component
│   │   └── ...           # Other UI components
│   └── [feature]/        # Feature-specific components
│       ├── PigCard.tsx   # Pig card component
│       └── ...           # Other feature components
├── hooks/                # Custom React hooks
│   ├── useAuth.ts        # Authentication hook
│   └── ...               # Other hooks
├── lib/                  # Utility functions and shared code
│   ├── axios.ts          # API client configuration
│   └── utils.ts          # General utility functions
├── types/                # TypeScript type definitions
│   ├── pig.ts            # Pig-related types
│   └── ...               # Other type definitions
├── public/               # Static assets
├── .env.local            # Environment variables (not in version control)
├── .env.example          # Example environment variables
├── next.config.mjs       # Next.js configuration
├── package.json          # Node.js dependencies
└── README.md             # Frontend documentation
```

## Naming Conventions

### Files and Directories

- **Backend**:
  - Use PascalCase for model files: `Pig.js`, `Farm.js`
  - Use camelCase for route files: `pig.js`, `farm.js`
  - Use camelCase for utility and service files: `activityLogger.js`

- **Frontend**:
  - Use PascalCase for component files: `Button.tsx`, `PigCard.tsx`
  - Use camelCase for utility and hook files: `useAuth.ts`, `axios.ts`
  - Use kebab-case for CSS module files: `button-styles.module.css`

### Variables and Functions

- Use camelCase for variables and function names:
  ```javascript
  const pigCount = 10;
  function calculateTotal() { ... }
  ```

- Use PascalCase for React components and classes:
  ```javascript
  function PigCard() { ... }
  class PigModel { ... }
  ```

- Use UPPER_SNAKE_CASE for constants:
  ```javascript
  const MAX_PIGS_PER_PAGE = 20;
  const API_BASE_URL = 'http://localhost:8080/api';
  ```

### Database

- Use PascalCase for collection names: `Pigs`, `Farms`
- Use camelCase for field names: `pigId`, `currentLocation`

## Coding Style Guidelines

### JavaScript/TypeScript

- Use ES6+ features where appropriate
- Prefer `const` over `let` when variables are not reassigned
- Avoid using `var`
- Use arrow functions for callbacks and anonymous functions
- Use async/await for asynchronous operations
- Use destructuring for object and array access
- Use template literals for string interpolation
- Use optional chaining and nullish coalescing operators

```javascript
// Good
const { name, age } = pig;
const formattedName = `Pig: ${name}`;
const displayAge = age ?? 'Unknown';
const location = pig?.currentLocation?.stallId?.name || 'Unknown Location';

// Avoid
const name = pig.name;
const age = pig.age;
const formattedName = 'Pig: ' + name;
const displayAge = age !== null && age !== undefined ? age : 'Unknown';
const location = pig && pig.currentLocation && pig.currentLocation.stallId && pig.currentLocation.stallId.name ? pig.currentLocation.stallId.name : 'Unknown Location';
```

### React

- Use functional components with hooks
- Use the React Context API for global state management
- Use React Query for server state management
- Break down complex components into smaller, reusable components
- Use prop destructuring in component parameters
- Use the `useCallback` hook for memoizing functions
- Use the `useMemo` hook for memoizing expensive calculations
- Use the `useEffect` hook for side effects

```jsx
// Good
function PigCard({ pig, onEdit }) {
  const { name, age, breed } = pig;
  
  const handleEdit = useCallback(() => {
    onEdit(pig.id);
  }, [onEdit, pig.id]);
  
  return (
    <Card>
      <CardHeader>{name}</CardHeader>
      <CardBody>
        <p>Age: {age}</p>
        <p>Breed: {breed}</p>
      </CardBody>
      <CardFooter>
        <Button onClick={handleEdit}>Edit</Button>
      </CardFooter>
    </Card>
  );
}
```

### CSS/Styling

- Use Tailwind CSS for styling components
- Use CSS modules for component-specific styles
- Follow a mobile-first approach for responsive design
- Use CSS variables for theme colors and spacing

```jsx
// Using Tailwind CSS
function Button({ children, variant = 'primary' }) {
  const baseClasses = 'px-4 py-2 rounded font-medium focus:outline-none';
  const variantClasses = {
    primary: 'bg-blue-500 text-white hover:bg-blue-600',
    secondary: 'bg-gray-200 text-gray-800 hover:bg-gray-300',
    danger: 'bg-red-500 text-white hover:bg-red-600'
  };
  
  return (
    <button className={`${baseClasses} ${variantClasses[variant]}`}>
      {children}
    </button>
  );
}
```

### Backend

- Use async/await for asynchronous operations
- Use try/catch blocks for error handling
- Use middleware for cross-cutting concerns
- Use environment variables for configuration
- Use meaningful variable and function names
- Add comments for complex logic

```javascript
// Good
router.get('/:id', async (req, res) => {
  try {
    const id = parseInt(req.params.id, 10);
    
    // Validate ID
    if (isNaN(id)) {
      return res.status(400).json({ error: 'Invalid pig ID' });
    }
    
    // Find pig by ID
    const pig = await Pig.findOne({ pigId: id })
      .populate('currentLocation.farmId')
      .populate('currentLocation.barnId')
      .populate('currentLocation.stallId');
    
    // Check if pig exists
    if (!pig) {
      return res.status(404).json({ error: 'Pig not found' });
    }
    
    res.json(pig);
  } catch (error) {
    console.error('Error fetching pig:', error);
    res.status(500).json({ error: 'Failed to fetch pig' });
  }
});
```

## Documentation Guidelines

### Code Comments

- Use JSDoc comments for functions and classes
- Add inline comments for complex logic
- Keep comments up-to-date with code changes
- Focus on why, not what (the code should be self-explanatory)

```javascript
/**
 * Calculate health risk score for a pig
 * @param {Object} pigData - Pig data including health metrics
 * @returns {number} Risk score between 0 and 1
 */
function calculateHealthRisk(pigData) {
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
  
  // Cap risk score at 1
  return Math.min(riskScore, 1);
}
```

### README Files

- Each major directory should have a README.md file
- READMEs should explain the purpose of the directory and its contents
- Include setup instructions, usage examples, and other relevant information

## Error Handling

### Frontend

- Use try/catch blocks for async operations
- Use error boundaries for React components
- Display user-friendly error messages
- Log errors to the console in development

```jsx
async function fetchPigData(id) {
  try {
    const response = await api.get(`/pigs/${id}`);
    return response.data;
  } catch (error) {
    console.error('Error fetching pig data:', error);
    
    // Return user-friendly error message
    if (error.response) {
      if (error.response.status === 404) {
        throw new Error('Pig not found');
      } else if (error.response.status === 401) {
        throw new Error('You must be logged in to view this information');
      }
    }
    
    throw new Error('Failed to load pig data. Please try again later.');
  }
}
```

### Backend

- Use try/catch blocks for async operations
- Return appropriate HTTP status codes
- Provide meaningful error messages
- Log errors with context information

```javascript
router.post('/', async (req, res) => {
  try {
    const { pigId, tag, breed, age, currentLocation } = req.body;

    // Validate input
    if (!pigId || !tag) {
      return res.status(400).json({ error: 'Pig ID and tag are required' });
    }

    // Create pig
    const newPig = await Pig.create({
      pigId,
      tag,
      breed,
      age,
      currentLocation,
      active: true
    });

    res.status(201).json({
      success: true,
      pig: newPig
    });
  } catch (error) {
    console.error('Error creating pig:', error);
    
    // Handle specific errors
    if (error.name === 'ValidationError') {
      return res.status(400).json({ error: error.message });
    } else if (error.code === 11000) {
      return res.status(409).json({ error: 'Pig with this ID or tag already exists' });
    }
    
    res.status(500).json({ error: 'Failed to create pig' });
  }
});
```

## Testing Conventions

### Test File Organization

- Place test files adjacent to the files they test
- Use `.test.js` or `.spec.js` suffix for test files
- Group tests by feature or component

```
components/
├── Button.tsx
├── Button.test.tsx
├── Card.tsx
└── Card.test.tsx
```

### Test Naming

- Use descriptive test names that explain what is being tested
- Follow the pattern: `describe('Component/Function', () => { it('should do something', () => {}) })`

```javascript
describe('PigCard', () => {
  it('should render pig name and breed', () => {
    // Test code
  });
  
  it('should call onEdit when edit button is clicked', () => {
    // Test code
  });
});
```

### Test Structure

- Follow the Arrange-Act-Assert pattern
- Use setup and teardown functions for common test setup
- Mock external dependencies

```javascript
describe('PigService', () => {
  // Setup
  beforeEach(() => {
    jest.clearAllMocks();
  });
  
  it('should fetch pig by ID', async () => {
    // Arrange
    const mockPig = { pigId: 1, name: 'Test Pig' };
    jest.spyOn(Pig, 'findOne').mockResolvedValue(mockPig);
    
    // Act
    const result = await PigService.getPigById(1);
    
    // Assert
    expect(Pig.findOne).toHaveBeenCalledWith({ pigId: 1 });
    expect(result).toEqual(mockPig);
  });
});
```

## Git Workflow

### Branch Naming

- Use feature branches for new features: `feature/add-pig-filter`
- Use bugfix branches for bug fixes: `bugfix/fix-login-error`
- Use hotfix branches for urgent fixes: `hotfix/critical-security-fix`

### Commit Messages

- Use present tense, imperative style: "Add feature" not "Added feature"
- Start with a capital letter
- Keep the first line under 50 characters
- Add more details in the commit body if needed
- Reference issue numbers when applicable

```
Add pig filtering by health status

- Add filter component to pig list page
- Implement filter logic in API endpoint
- Update tests for new filtering functionality

Fixes #123
```

### Pull Requests

- Use descriptive titles
- Include a summary of changes
- Reference related issues
- Add screenshots for UI changes
- Ensure all tests pass before requesting review

## Conclusion

Following these code structure and convention guidelines ensures consistency across the PAAL codebase and makes it easier for developers to understand, maintain, and extend the system. These conventions should be followed for all new code and applied to existing code during refactoring.
