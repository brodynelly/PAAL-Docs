# Testing Strategy

## Overview

The PAAL system implements a comprehensive testing strategy to ensure code quality, functionality, and reliability. This document outlines the testing approach, tools, and practices used throughout the frontend application.

## Testing Pyramid

The testing strategy follows the testing pyramid approach, with a focus on:

1. **Unit Tests**: Testing individual components and functions in isolation
2. **Integration Tests**: Testing interactions between components
3. **End-to-End Tests**: Testing complete user flows

The distribution aims for approximately:
- 70% Unit Tests
- 20% Integration Tests
- 10% End-to-End Tests

## Testing Tools and Libraries

### Core Testing Libraries

- **Jest**: JavaScript testing framework for unit and integration tests
- **React Testing Library**: Testing utilities for React components
- **Cypress**: End-to-end testing framework
- **MSW (Mock Service Worker)**: API mocking library for testing API interactions

### Additional Testing Utilities

- **@testing-library/user-event**: Simulates user interactions
- **@testing-library/jest-dom**: Custom Jest matchers for DOM testing
- **jest-axe**: Accessibility testing with Jest
- **Cypress Axe**: Accessibility testing with Cypress

## Unit Testing

Unit tests focus on testing individual components, hooks, and utility functions in isolation.

### Component Testing

Components are tested using React Testing Library to ensure they render correctly and respond to user interactions as expected.

Example component test:

```tsx
// src/components/Button.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Button } from './Button';

describe('Button', () => {
  it('renders correctly with default props', () => {
    render(<Button>Click me</Button>);
    
    const button = screen.getByRole('button', { name: /click me/i });
    expect(button).toBeInTheDocument();
    expect(button).toHaveClass('bg-primary');
  });

  it('applies variant classes correctly', () => {
    render(<Button variant="secondary">Secondary</Button>);
    
    const button = screen.getByRole('button', { name: /secondary/i });
    expect(button).toHaveClass('bg-secondary');
    expect(button).not.toHaveClass('bg-primary');
  });

  it('calls onClick handler when clicked', async () => {
    const handleClick = jest.fn();
    render(<Button onClick={handleClick}>Click me</Button>);
    
    const button = screen.getByRole('button', { name: /click me/i });
    await userEvent.click(button);
    
    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('is disabled when disabled prop is true', () => {
    render(<Button disabled>Disabled</Button>);
    
    const button = screen.getByRole('button', { name: /disabled/i });
    expect(button).toBeDisabled();
  });
});
```

### Hook Testing

Custom hooks are tested using React Testing Library's `renderHook` utility.

Example hook test:

```tsx
// src/hooks/useCounter.test.ts
import { renderHook, act } from '@testing-library/react';
import { useCounter } from './useCounter';

describe('useCounter', () => {
  it('initializes with the provided initial value', () => {
    const { result } = renderHook(() => useCounter(5));
    
    expect(result.current.count).toBe(5);
  });

  it('increments the counter when increment is called', () => {
    const { result } = renderHook(() => useCounter(0));
    
    act(() => {
      result.current.increment();
    });
    
    expect(result.current.count).toBe(1);
  });

  it('decrements the counter when decrement is called', () => {
    const { result } = renderHook(() => useCounter(10));
    
    act(() => {
      result.current.decrement();
    });
    
    expect(result.current.count).toBe(9);
  });

  it('resets the counter when reset is called', () => {
    const { result } = renderHook(() => useCounter(0));
    
    act(() => {
      result.current.increment();
      result.current.increment();
      result.current.reset();
    });
    
    expect(result.current.count).toBe(0);
  });
});
```

### Utility Function Testing

Pure utility functions are tested with straightforward Jest tests.

Example utility test:

```tsx
// src/lib/utils.test.ts
import { formatCurrency, formatDate, calculatePercentage } from './utils';

describe('formatCurrency', () => {
  it('formats numbers as USD currency', () => {
    expect(formatCurrency(1000)).toBe('$1,000.00');
    expect(formatCurrency(1000.5)).toBe('$1,000.50');
    expect(formatCurrency(0)).toBe('$0.00');
  });
});

describe('formatDate', () => {
  it('formats dates in the expected format', () => {
    const date = new Date('2023-01-15T12:00:00Z');
    expect(formatDate(date)).toBe('Jan 15, 2023');
  });

  it('returns a placeholder for invalid dates', () => {
    expect(formatDate(null)).toBe('--');
    expect(formatDate(undefined)).toBe('--');
  });
});

describe('calculatePercentage', () => {
  it('calculates percentages correctly', () => {
    expect(calculatePercentage(50, 200)).toBe(25);
    expect(calculatePercentage(200, 100)).toBe(200);
  });

  it('handles edge cases', () => {
    expect(calculatePercentage(0, 100)).toBe(0);
    expect(calculatePercentage(100, 0)).toBe(0);
  });
});
```

## Integration Testing

Integration tests focus on how components work together and interact with external services.

### Component Integration

Testing how multiple components interact when composed together.

Example integration test:

```tsx
// src/features/PigDetails/PigDetails.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { rest } from 'msw';
import { setupServer } from 'msw/node';
import { PigDetails } from './PigDetails';
import { QueryClientProvider, QueryClient } from 'react-query';

// Mock API responses
const server = setupServer(
  rest.get('/api/pigs/123', (req, res, ctx) => {
    return res(
      ctx.json({
        pigId: 123,
        tag: 'PIG-123',
        breed: 'Yorkshire',
        age: 12,
        currentLocation: {
          farmId: { _id: 'farm1', name: 'Farm 1' },
          barnId: { _id: 'barn1', name: 'Barn A' },
          stallId: { _id: 'stall1', name: 'Stall 1' }
        }
      })
    );
  }),
  
  rest.get('/api/pigs/123/posture/aggregated', (req, res, ctx) => {
    return res(
      ctx.json([
        { date: '2023-01-01', standing: 40, sitting: 30, lying: 30 },
        { date: '2023-01-02', standing: 45, sitting: 25, lying: 30 }
      ])
    );
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

// Create a wrapper with necessary providers
const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,
      },
    },
  });
  
  return ({ children }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
};

describe('PigDetails', () => {
  it('loads and displays pig details', async () => {
    render(<PigDetails pigId="123" />, { wrapper: createWrapper() });
    
    // Check loading state
    expect(screen.getByText(/loading/i)).toBeInTheDocument();
    
    // Wait for data to load
    await waitFor(() => {
      expect(screen.queryByText(/loading/i)).not.toBeInTheDocument();
    });
    
    // Check pig details are displayed
    expect(screen.getByText('PIG-123')).toBeInTheDocument();
    expect(screen.getByText('Yorkshire')).toBeInTheDocument();
    expect(screen.getByText('12')).toBeInTheDocument();
    expect(screen.getByText('Farm 1')).toBeInTheDocument();
    
    // Check tabs work
    const postureTab = screen.getByRole('tab', { name: /posture/i });
    await userEvent.click(postureTab);
    
    // Check posture data is displayed
    expect(screen.getByText('Standing')).toBeInTheDocument();
    expect(screen.getByText('40%')).toBeInTheDocument();
  });
  
  it('handles API errors gracefully', async () => {
    // Override the handler to return an error
    server.use(
      rest.get('/api/pigs/123', (req, res, ctx) => {
        return res(ctx.status(500), ctx.json({ error: 'Server error' }));
      })
    );
    
    render(<PigDetails pigId="123" />, { wrapper: createWrapper() });
    
    // Wait for error state
    await waitFor(() => {
      expect(screen.getByText(/failed to load pig data/i)).toBeInTheDocument();
    });
  });
});
```

### API Integration

Testing components that interact with APIs using MSW to mock API responses.

Example API integration test:

```tsx
// src/features/FarmList/FarmList.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import { rest } from 'msw';
import { setupServer } from 'msw/node';
import { FarmList } from './FarmList';
import { QueryClientProvider, QueryClient } from 'react-query';

const farms = [
  {
    _id: 'farm1',
    name: 'Farm 1',
    location: 'Location 1',
    counts: { barns: 5, stalls: 20, pigs: 100 }
  },
  {
    _id: 'farm2',
    name: 'Farm 2',
    location: 'Location 2',
    counts: { barns: 3, stalls: 12, pigs: 60 }
  }
];

const server = setupServer(
  rest.get('/api/farms', (req, res, ctx) => {
    return res(ctx.json(farms));
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,
      },
    },
  });
  
  return ({ children }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
};

describe('FarmList', () => {
  it('loads and displays farms', async () => {
    render(<FarmList />, { wrapper: createWrapper() });
    
    // Check loading state
    expect(screen.getByText(/loading farms/i)).toBeInTheDocument();
    
    // Wait for data to load
    await waitFor(() => {
      expect(screen.queryByText(/loading farms/i)).not.toBeInTheDocument();
    });
    
    // Check farm data is displayed
    expect(screen.getByText('Farm 1')).toBeInTheDocument();
    expect(screen.getByText('Farm 2')).toBeInTheDocument();
    expect(screen.getByText('Location 1')).toBeInTheDocument();
    expect(screen.getByText('100 pigs')).toBeInTheDocument();
  });
  
  it('displays empty state when no farms are available', async () => {
    // Override the handler to return an empty array
    server.use(
      rest.get('/api/farms', (req, res, ctx) => {
        return res(ctx.json([]));
      })
    );
    
    render(<FarmList />, { wrapper: createWrapper() });
    
    // Wait for empty state
    await waitFor(() => {
      expect(screen.getByText(/no farms found/i)).toBeInTheDocument();
    });
  });
});
```

## End-to-End Testing

End-to-end tests use Cypress to test complete user flows through the application.

Example E2E test:

```javascript
// cypress/e2e/login.cy.js
describe('Login Flow', () => {
  beforeEach(() => {
    // Visit the login page before each test
    cy.visit('/login');
  });

  it('displays the login form', () => {
    cy.get('h1').should('contain', 'Sign In');
    cy.get('input[name="email"]').should('exist');
    cy.get('input[name="password"]').should('exist');
    cy.get('button[type="submit"]').should('exist');
  });

  it('shows validation errors for empty fields', () => {
    cy.get('button[type="submit"]').click();
    cy.get('form').contains('Email is required');
    cy.get('form').contains('Password is required');
  });

  it('shows an error message for invalid credentials', () => {
    cy.get('input[name="email"]').type('invalid@example.com');
    cy.get('input[name="password"]').type('wrongpassword');
    cy.get('button[type="submit"]').click();
    
    cy.contains('Invalid credentials').should('be.visible');
  });

  it('redirects to dashboard after successful login', () => {
    // Intercept the login API call
    cy.intercept('POST', '/api/auth/login', {
      statusCode: 200,
      body: {
        token: 'fake-jwt-token',
        user: {
          id: 'user1',
          email: 'user@example.com',
          firstName: 'Test',
          lastName: 'User',
          role: 'admin'
        }
      }
    }).as('loginRequest');
    
    cy.get('input[name="email"]').type('user@example.com');
    cy.get('input[name="password"]').type('password123');
    cy.get('button[type="submit"]').click();
    
    cy.wait('@loginRequest');
    cy.url().should('include', '/overview');
    cy.contains('Welcome, Test').should('be.visible');
  });
});
```

Example dashboard E2E test:

```javascript
// cypress/e2e/dashboard.cy.js
describe('Dashboard', () => {
  beforeEach(() => {
    // Login and set token before each test
    cy.login('admin@example.com', 'password123');
    cy.visit('/overview');
  });

  it('displays the dashboard with key metrics', () => {
    cy.get('h1').should('contain', 'Dashboard');
    
    // Check for key metrics
    cy.contains('Total Pigs').should('exist');
    cy.contains('Healthy').should('exist');
    cy.contains('At Risk').should('exist');
    
    // Check for charts
    cy.get('[data-testid="pig-distribution-chart"]').should('exist');
    cy.get('[data-testid="health-status-chart"]').should('exist');
  });

  it('filters data when date range is changed', () => {
    // Intercept the API call that will be made when filter changes
    cy.intercept('GET', '/api/stats*').as('statsRequest');
    
    // Open date picker
    cy.get('[data-testid="date-range-selector"]').click();
    
    // Select last 30 days
    cy.get('[data-value="30d"]').click();
    
    // Wait for API request to complete
    cy.wait('@statsRequest');
    
    // Verify the chart has updated
    cy.get('[data-testid="pig-distribution-chart"]')
      .should('contain', 'Last 30 days');
  });

  it('navigates to pig details when clicking on a pig', () => {
    // Click on a pig in the table
    cy.get('[data-testid="pig-table"]')
      .contains('tr', 'PIG-001')
      .click();
    
    // Verify navigation to pig details page
    cy.url().should('include', '/pigs/');
    cy.contains('Pig Details').should('exist');
  });
});
```

## Accessibility Testing

Accessibility testing is integrated into both component tests and end-to-end tests.

### Component-Level Accessibility Testing

Using jest-axe to test components for accessibility violations:

```tsx
// src/components/Form.test.tsx
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';
import { Form } from './Form';

expect.extend(toHaveNoViolations);

describe('Form', () => {
  it('should not have accessibility violations', async () => {
    const { container } = render(
      <Form>
        <label htmlFor="name">Name</label>
        <input id="name" name="name" type="text" />
        <button type="submit">Submit</button>
      </Form>
    );
    
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });
});
```

### End-to-End Accessibility Testing

Using Cypress-axe to test pages for accessibility violations:

```javascript
// cypress/e2e/accessibility.cy.js
describe('Accessibility Tests', () => {
  beforeEach(() => {
    cy.login('admin@example.com', 'password123');
    cy.injectAxe();
  });

  it('Dashboard page has no detectable accessibility violations', () => {
    cy.visit('/overview');
    cy.checkA11y();
  });

  it('Pig details page has no detectable accessibility violations', () => {
    cy.visit('/pigs/123');
    cy.checkA11y();
  });

  it('Farm management page has no detectable accessibility violations', () => {
    cy.visit('/admin/farms');
    cy.checkA11y();
  });
});
```

## Visual Regression Testing

Visual regression testing is implemented using Cypress and Percy to detect unintended visual changes.

```javascript
// cypress/e2e/visual.cy.js
describe('Visual Regression Tests', () => {
  beforeEach(() => {
    cy.login('admin@example.com', 'password123');
  });

  it('Dashboard looks correct', () => {
    cy.visit('/overview');
    cy.wait(1000); // Wait for animations to complete
    cy.percySnapshot('Dashboard');
  });

  it('Pig details page looks correct', () => {
    cy.visit('/pigs/123');
    cy.wait(1000);
    cy.percySnapshot('Pig Details');
  });

  it('Farm management page looks correct', () => {
    cy.visit('/admin/farms');
    cy.wait(1000);
    cy.percySnapshot('Farm Management');
  });
});
```

## Performance Testing

Performance testing focuses on key metrics like load time, time to interactive, and bundle size.

### Lighthouse CI

Lighthouse CI is integrated into the CI/CD pipeline to measure performance metrics:

```yaml
# .github/workflows/lighthouse.yml
name: Lighthouse CI
on: [push]
jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 16
      - run: npm ci
      - run: npm run build
      - name: Run Lighthouse CI
        uses: treosh/lighthouse-ci-action@v9
        with:
          urls: |
            https://staging.example.com/
            https://staging.example.com/login
            https://staging.example.com/overview
          budgetPath: ./lighthouse-budget.json
          uploadArtifacts: true
```

### Bundle Analysis

Webpack Bundle Analyzer is used to monitor bundle size:

```bash
# Package.json script
"analyze": "ANALYZE=true next build"
```

## Test Coverage

Jest is configured to collect test coverage information:

```json
// jest.config.js
{
  "collectCoverage": true,
  "collectCoverageFrom": [
    "src/**/*.{js,jsx,ts,tsx}",
    "!src/**/*.d.ts",
    "!src/**/*.stories.{js,jsx,ts,tsx}",
    "!src/pages/_app.tsx",
    "!src/pages/_document.tsx"
  ],
  "coverageThreshold": {
    "global": {
      "branches": 80,
      "functions": 80,
      "lines": 80,
      "statements": 80
    }
  }
}
```

## Continuous Integration

Tests are run automatically in the CI/CD pipeline:

```yaml
# .github/workflows/test.yml
name: Test
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 16
      - run: npm ci
      - run: npm run lint
      - run: npm test
      - name: Upload coverage
        uses: codecov/codecov-action@v3
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 16
      - run: npm ci
      - name: Cypress run
        uses: cypress-io/github-action@v5
        with:
          build: npm run build
          start: npm start
```

## Testing Best Practices

1. **Write tests as you code**: Tests should be written alongside the code they test
2. **Test behavior, not implementation**: Focus on what the component does, not how it does it
3. **Use realistic data**: Test with data that resembles what will be used in production
4. **Test edge cases**: Include tests for error states, empty states, and boundary conditions
5. **Keep tests independent**: Each test should be able to run independently of others
6. **Avoid testing third-party code**: Focus on testing your own code, not libraries
7. **Use test-driven development when appropriate**: For complex logic, consider writing tests first
8. **Maintain test quality**: Refactor tests as the codebase evolves
9. **Test accessibility**: Include accessibility checks in your testing strategy
10. **Monitor test coverage**: Use coverage reports to identify untested code
