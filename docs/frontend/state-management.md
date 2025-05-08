# State Management

## Overview

The PAAL frontend uses a combination of state management approaches to handle different types of state. This document outlines those said states: 

## State Categories

The application's state is divided into several categories:

1. **Server State**: Data fetched from the backend API
2. **UI State**: Visual state like open/closed modals, active tabs, etc.
3. **Form State**: User input in forms and validation state
4. **Authentication State**: User authentication information
5. **Application State**: Cross-cutting concerns like theme preferences

## Server State Management

### React Query

React Query is the primary tool for managing server state. It provides:

- Data fetching with automatic caching
- Background refetching for stale data
- Pagination and infinite scrolling support
- Mutation capabilities for updating data
- Optimistic updates for a responsive UI

#### Example Usage:

```tsx
import { useQuery, useMutation, useQueryClient } from 'react-query';
import api from '@/lib/axios';

// Fetch pigs
const usePigs = (filters) => {
  return useQuery(
    ['pigs', filters],
    async () => {
      const response = await api.get('/pigs', { params: filters });
      return response.data;
    },
    {
      staleTime: 5 * 60 * 1000, // 5 minutes
      refetchOnWindowFocus: true,
    }
  );
};

// Update a pig
const useUpdatePig = () => {
  const queryClient = useQueryClient();

  return useMutation(
    async (pigData) => {
      const response = await api.put(`/pigs/${pigData.pigId}`, pigData);
      return response.data;
    },
    {
      onSuccess: (data, variables) => {
        // Invalidate and refetch
        queryClient.invalidateQueries(['pigs']);
        queryClient.invalidateQueries(['pig', variables.pigId]);
      },
    }
  );
};
```

### Custom Hooks for API Interactions

For complex data requirements, custom hooks encapsulate the data fetching logic:

```tsx
// src/hooks/usePigDetails.ts
import { useQuery } from 'react-query';
import api from '@/lib/axios';

export const usePigDetails = (pigId) => {
  const { data: pig, isLoading: pigLoading, error: pigError } = useQuery(
    ['pig', pigId],
    async () => {
      const response = await api.get(`/pigs/${pigId}`);
      return response.data;
    },
    {
      enabled: !!pigId,
    }
  );

  const { data: postureData, isLoading: postureLoading, error: postureError } = useQuery(
    ['pig-posture', pigId],
    async () => {
      const response = await api.get(`/pigs/${pigId}/posture/aggregated`);
      return response.data;
    },
    {
      enabled: !!pigId,
    }
  );

  return {
    pig,
    postureData,
    isLoading: pigLoading || postureLoading,
    error: pigError || postureError,
  };
};
```

## UI State Management

### React Context API

For UI state that needs to be shared across components, React Context is used:

```tsx
// src/components/Sidebar/SidebarContext.tsx
import React, { createContext, useContext, useState } from 'react';

type SidebarContextType = {
  isOpen: boolean;
  toggle: () => void;
  close: () => void;
  open: () => void;
};

const SidebarContext = createContext<SidebarContextType | undefined>(undefined);

export const SidebarProvider: React.FC<{
  children: React.ReactNode;
  defaultOpen?: boolean;
}> = ({ children, defaultOpen = false }) => {
  const [isOpen, setIsOpen] = useState(defaultOpen);

  const toggle = () => setIsOpen(!isOpen);
  const close = () => setIsOpen(false);
  const open = () => setIsOpen(true);

  return (
    <SidebarContext.Provider value={{ isOpen, toggle, close, open }}>
      {children}
    </SidebarContext.Provider>
  );
};

export const useSidebar = () => {
  const context = useContext(SidebarContext);
  if (context === undefined) {
    throw new Error('useSidebar must be used within a SidebarProvider');
  }
  return context;
};
```

### Component-Level State

For UI state specific to a component, React's `useState` and `useReducer` hooks are used:

```tsx
// Tab state example
const [activeTab, setActiveTab] = useState('overview');

// Modal state example
const [isModalOpen, setIsModalOpen] = useState(false);

// Complex state with useReducer
type State = {
  isLoading: boolean;
  error: string | null;
  data: any[];
  page: number;
};

type Action =
  | { type: 'FETCH_START' }
  | { type: 'FETCH_SUCCESS'; payload: any[] }
  | { type: 'FETCH_ERROR'; payload: string }
  | { type: 'SET_PAGE'; payload: number };

const reducer = (state: State, action: Action): State => {
  switch (action.type) {
    case 'FETCH_START':
      return { ...state, isLoading: true, error: null };
    case 'FETCH_SUCCESS':
      return { ...state, isLoading: false, data: action.payload };
    case 'FETCH_ERROR':
      return { ...state, isLoading: false, error: action.payload };
    case 'SET_PAGE':
      return { ...state, page: action.payload };
    default:
      return state;
  }
};

const [state, dispatch] = useReducer(reducer, {
  isLoading: false,
  error: null,
  data: [],
  page: 1,
});
```

## Form State Management

### React Hook Form

For managing form state, validation, and submission, React Hook Form is used:

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import * as z from 'zod';

// Define validation schema
const schema = z.object({
  name: z.string().min(1, 'Name is required'),
  location: z.string().min(1, 'Location is required'),
  description: z.string().optional(),
});

type FormValues = z.infer<typeof schema>;

const CreateFarmForm = () => {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
    reset,
  } = useForm<FormValues>({
    resolver: zodResolver(schema),
    defaultValues: {
      name: '',
      location: '',
      description: '',
    },
  });

  const onSubmit = async (data: FormValues) => {
    try {
      await api.post('/farms', data);
      reset();
      // Show success notification
    } catch (error) {
      // Handle error
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* Form fields */}
    </form>
  );
};
```

## Authentication State

Authentication state is managed using a custom AuthProvider context:

```tsx
// src/components/AuthProvider.tsx
import React, { createContext, useContext, useEffect, useState } from 'react';
import { useRouter } from 'next/navigation';
import api from '@/lib/axios';

type User = {
  id: string;
  email: string;
  firstName: string;
  lastName: string;
  role: string;
};

type AuthContextType = {
  user: User | null;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  isAuthenticated: boolean;
};

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export const AuthProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const router = useRouter();

  // Check if user is authenticated on mount
  useEffect(() => {
    const checkAuth = async () => {
      try {
        const token = localStorage.getItem('token');
        if (!token) {
          setIsLoading(false);
          return;
        }

        const response = await api.get('/auth/token');
        setUser(response.data.user);
      } catch (error) {
        localStorage.removeItem('token');
        localStorage.removeItem('user');
      } finally {
        setIsLoading(false);
      }
    };

    checkAuth();
  }, []);

  const login = async (email: string, password: string) => {
    setIsLoading(true);
    try {
      const response = await api.post('/auth/login', { email, password });
      const { token, user } = response.data;

      localStorage.setItem('token', token);
      localStorage.setItem('user', JSON.stringify(user));

      setUser(user);
      router.push('/overview');
    } catch (error) {
      throw error;
    } finally {
      setIsLoading(false);
    }
  };

  const logout = () => {
    localStorage.removeItem('token');
    localStorage.removeItem('user');
    setUser(null);
    router.push('/login');
  };

  return (
    <AuthContext.Provider
      value={{
        user,
        isLoading,
        login,
        logout,
        isAuthenticated: !!user,
      }}
    >
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (context === undefined) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
};
```

## Application State

### Theme and Preferences

Theme and other user preferences are managed using the `next-themes` library and local storage:

```tsx
// src/app/layout.tsx
import { ThemeProvider } from 'next-themes';

export default function RootLayout({ children }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body>
        <ThemeProvider
          attribute="class"
          defaultTheme="light"
          disableTransitionOnChange
        >
          {children}
        </ThemeProvider>
      </body>
    </html>
  );
}

// Usage in components
import { useTheme } from 'next-themes';

const ThemeToggle = () => {
  const { theme, setTheme } = useTheme();

  return (
    <button onClick={() => setTheme(theme === 'dark' ? 'light' : 'dark')}>
      Toggle Theme
    </button>
  );
};
```

### URL State

For state that should be reflected in the URL (for bookmarking and sharing), the `useSearchParams` hook and `nuqs` library are used:

```tsx
import { useSearchParams, usePathname, useRouter } from 'next/navigation';
import { useQueryState } from 'nuqs';

const FilterComponent = () => {
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();

  // Using Next.js built-in hooks
  const handleFilterChange = (filter) => {
    const params = new URLSearchParams(searchParams.toString());
    params.set('filter', filter);
    router.push(`${pathname}?${params.toString()}`);
  };

  // Using nuqs for type-safe URL state
  const [dateRange, setDateRange] = useQueryState('dateRange', {
    defaultValue: '30d',
    parse: (value) => value as '7d' | '30d' | '90d',
  });

  return (
    <div>
      <select
        value={searchParams.get('filter') || ''}
        onChange={(e) => handleFilterChange(e.target.value)}
      >
        <option value="">All</option>
        <option value="healthy">Healthy</option>
        <option value="at-risk">At Risk</option>
      </select>

      <select
        value={dateRange}
        onChange={(e) => setDateRange(e.target.value as any)}
      >
        <option value="7d">Last 7 days</option>
        <option value="30d">Last 30 days</option>
        <option value="90d">Last 90 days</option>
      </select>
    </div>
  );
};
```

## Cache Invalidation Strategies

Effective cache invalidation is crucial for maintaining data consistency while optimizing performance. The PAAL system implements several cache invalidation strategies:

### Time-Based Invalidation

Data is automatically considered stale after a specified time period:

```typescript
// Configure stale time for queries
const { data: pigs } = useQuery({
  queryKey: ['pigs'],
  queryFn: fetchPigs,
  staleTime: 5 * 60 * 1000, // Data becomes stale after 5 minutes
  cacheTime: 30 * 60 * 1000 // Cached data is removed after 30 minutes of inactivity
});
```

Different data types have different staleness configurations:

| Data Type | Stale Time | Cache Time | Rationale |
|-----------|------------|------------|-----------|
| Reference Data (farms, barns) | 1 hour | 24 hours | Changes infrequently |
| Entity Data (pigs, users) | 5 minutes | 1 hour | Moderate change frequency |
| Real-time Data (posture readings) | 30 seconds | 5 minutes | Changes frequently |
| User Preferences | 1 hour | 24 hours | User-specific, changes infrequently |

### Event-Based Invalidation

Cache is invalidated when specific events occur:

```typescript
// Invalidate cache after mutation
const queryClient = useQueryClient();

const mutation = useMutation({
  mutationFn: createPig,
  onSuccess: (newPig) => {
    // Invalidate all pig queries
    queryClient.invalidateQueries({ queryKey: ['pigs'] });

    // Update farm counts
    queryClient.invalidateQueries({
      queryKey: ['farms', newPig.currentLocation.farmId]
    });
  }
});
```

### Selective Updates

Instead of invalidating entire queries, specific items in the cache can be updated:

```typescript
const updatePigMutation = useMutation({
  mutationFn: updatePig,
  onSuccess: (updatedPig) => {
    // Update the specific pig in the cache
    queryClient.setQueryData(['pig', updatedPig.pigId], updatedPig);

    // Update the pig in list queries
    queryClient.setQueriesData(['pigs'], (oldData) => {
      if (!oldData) return oldData;
      return oldData.map(pig =>
        pig.pigId === updatedPig.pigId ? updatedPig : pig
      );
    });
  }
});
```

### Optimistic Updates

UI is updated immediately before server confirmation to improve perceived performance:

```typescript
const addPigMutation = useMutation({
  mutationFn: createPig,
  // Optimistically update the cache
  onMutate: async (newPig) => {
    // Cancel outgoing refetches
    await queryClient.cancelQueries({ queryKey: ['pigs'] });

    // Snapshot the previous value
    const previousPigs = queryClient.getQueryData(['pigs']);

    // Optimistically update the cache
    queryClient.setQueryData(['pigs'], (old) => {
      return [...(old || []), { ...newPig, id: 'temp-id' }];
    });

    // Return context with the snapshotted value
    return { previousPigs };
  },
  // If the mutation fails, use the context we returned above
  onError: (err, newPig, context) => {
    queryClient.setQueryData(['pigs'], context.previousPigs);
  },
  // Always refetch after error or success
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['pigs'] });
  }
});
```

### Cache Persistence

Cache is persisted to localStorage to improve performance across page refreshes:

```typescript
// src/lib/queryClient.ts
import { QueryClient } from 'react-query';
import { createWebStoragePersistor } from 'react-query/persistQueryClient-experimental';
import { persistQueryClient } from 'react-query/persistQueryClient-experimental';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,
      cacheTime: 30 * 60 * 1000,
    },
  },
});

const localStoragePersistor = createWebStoragePersistor({
  storage: window.localStorage,
  key: 'paal-cache',
});

persistQueryClient({
  queryClient,
  persistor: localStoragePersistor,
  maxAge: 24 * 60 * 60 * 1000, // 24 hours
});

export default queryClient;
```

## Performance Optimization Techniques

The PAAL system implements several performance optimization techniques:

### Component Optimization

#### Memoization

Components are memoized to prevent unnecessary re-renders:

```tsx
// Memoize a component
const PigCard = React.memo(({ pig }) => {
  return (
    <div className="card">
      <h3>{pig.tag}</h3>
      <p>Breed: {pig.breed}</p>
      <p>Age: {pig.age}</p>
    </div>
  );
});

// Memoize expensive calculations
const PigStatistics = ({ pigs }) => {
  // This calculation only runs when pigs array changes
  const statistics = useMemo(() => {
    return {
      total: pigs.length,
      averageAge: pigs.reduce((sum, pig) => sum + pig.age, 0) / pigs.length,
      breedDistribution: pigs.reduce((acc, pig) => {
        acc[pig.breed] = (acc[pig.breed] || 0) + 1;
        return acc;
      }, {})
    };
  }, [pigs]);

  return (
    <div>
      <p>Total: {statistics.total}</p>
      <p>Average Age: {statistics.averageAge.toFixed(1)}</p>
      {/* Render breed distribution */}
    </div>
  );
};
```

#### Callback Memoization

Event handlers are memoized to prevent unnecessary re-renders:

```tsx
const PigActions = ({ pigId, onView, onEdit, onDelete }) => {
  // These callbacks only change when pigId changes
  const handleView = useCallback(() => {
    onView(pigId);
  }, [pigId, onView]);

  const handleEdit = useCallback(() => {
    onEdit(pigId);
  }, [pigId, onEdit]);

  const handleDelete = useCallback(() => {
    onDelete(pigId);
  }, [pigId, onDelete]);

  return (
    <div className="actions">
      <button onClick={handleView}>View</button>
      <button onClick={handleEdit}>Edit</button>
      <button onClick={handleDelete}>Delete</button>
    </div>
  );
};
```

### Rendering Optimization

#### Virtualized Lists

Long lists are rendered using virtualization to improve performance:

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';

const VirtualizedPigList = ({ pigs }) => {
  const parentRef = useRef(null);

  const rowVirtualizer = useVirtualizer({
    count: pigs.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 60, // estimated row height
    overscan: 5, // number of items to render outside of the visible area
  });

  return (
    <div
      ref={parentRef}
      className="list-container"
      style={{ height: '500px', overflow: 'auto' }}
    >
      <div
        style={{
          height: `${rowVirtualizer.getTotalSize()}px`,
          width: '100%',
          position: 'relative',
        }}
      >
        {rowVirtualizer.getVirtualItems().map((virtualRow) => (
          <div
            key={virtualRow.index}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualRow.size}px`,
              transform: `translateY(${virtualRow.start}px)`,
            }}
          >
            <PigCard pig={pigs[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  );
};
```

#### Windowed Pagination

For large datasets, windowed pagination is used instead of loading all data at once:

```tsx
const PigTable = ({ farmId }) => {
  const [page, setPage] = useState(1);
  const [pageSize, setPageSize] = useState(20);

  const { data, isLoading } = useQuery({
    queryKey: ['pigs', { farmId, page, pageSize }],
    queryFn: () => fetchPigs({ farmId, page, pageSize }),
    keepPreviousData: true, // Keep previous data while loading new data
  });

  return (
    <div>
      <table>{/* Render table */}</table>

      <Pagination
        currentPage={page}
        totalPages={data?.totalPages || 1}
        onPageChange={setPage}
      />
    </div>
  );
};
```

### Data Fetching Optimization

#### Parallel Queries

Related data is fetched in parallel to reduce loading time:

```tsx
const FarmDetails = ({ farmId }) => {
  // Fetch farm details and pigs in parallel
  const { data: farm } = useQuery({
    queryKey: ['farm', farmId],
    queryFn: () => fetchFarm(farmId),
  });

  const { data: pigs } = useQuery({
    queryKey: ['pigs', { farmId }],
    queryFn: () => fetchPigs({ farmId }),
  });

  const { data: barns } = useQuery({
    queryKey: ['barns', { farmId }],
    queryFn: () => fetchBarns({ farmId }),
  });

  // Render farm details, pigs, and barns
};
```

#### Prefetching

Data is prefetched to improve perceived performance:

```tsx
const FarmList = ({ farms }) => {
  const queryClient = useQueryClient();

  // Prefetch farm details when hovering over a farm
  const prefetchFarm = (farmId) => {
    queryClient.prefetchQuery({
      queryKey: ['farm', farmId],
      queryFn: () => fetchFarm(farmId),
    });
  };

  return (
    <ul>
      {farms.map((farm) => (
        <li
          key={farm.id}
          onMouseEnter={() => prefetchFarm(farm.id)}
        >
          <Link to={`/farms/${farm.id}`}>{farm.name}</Link>
        </li>
      ))}
    </ul>
  );
};
```

#### Selective Data Loading

Only necessary data is loaded to reduce payload size:

```tsx
// API supports field selection
const { data: pigs } = useQuery({
  queryKey: ['pigs', { fields: 'pigId,tag,breed,age' }],
  queryFn: () => fetchPigs({ fields: 'pigId,tag,breed,age' }),
});
```

### Code Splitting

The application uses code splitting to reduce initial bundle size:

```tsx
// Dynamic import for code splitting
import { lazy, Suspense } from 'react';

const PigDetails = lazy(() => import('./PigDetails'));
const FarmDetails = lazy(() => import('./FarmDetails'));

const App = () => {
  return (
    <Routes>
      <Route path="/" element={<Dashboard />} />
      <Route
        path="/pigs/:id"
        element={
          <Suspense fallback={<LoadingSpinner />}>
            <PigDetails />
          </Suspense>
        }
      />
      <Route
        path="/farms/:id"
        element={
          <Suspense fallback={<LoadingSpinner />}>
            <FarmDetails />
          </Suspense>
        }
      />
    </Routes>
  );
};
```

## State Management Best Practices

1. **Colocation**: Keep state as close as possible to where it's used
   - Use component state for component-specific state
   - Use context for shared state within a subtree
   - Use global state only when necessary

2. **Single Source of Truth**: Avoid duplicating state across different stores
   - Define clear ownership of each piece of state
   - Derive state from a single source when needed in multiple places

3. **Immutability**: Always update state immutably to prevent bugs
   - Use spread operators or libraries like Immer for immutable updates
   - Avoid direct mutation of objects and arrays

4. **Derived State**: Calculate derived state on-the-fly rather than storing it
   - Use selectors or memoized functions to compute derived state
   - Avoid storing calculated values that can be derived from existing state

5. **Persistence**: Use localStorage for state that should persist across sessions
   - Only persist essential state (user preferences, authentication tokens)
   - Implement proper serialization and deserialization

6. **Separation of Concerns**: Separate UI state from server state
   - Use React Query for server state
   - Use React Context or component state for UI state

7. **Error Handling**: Include error states in all data fetching operations
   - Handle loading, success, and error states consistently
   - Provide meaningful error messages to users

8. **Optimistic Updates**: Update UI before server confirmation for better UX
   - Implement proper rollback mechanisms for failed operations
   - Show pending state for ongoing operations

9. **Minimize Re-renders**: Optimize components to prevent unnecessary re-renders
   - Use memoization techniques (React.memo, useMemo, useCallback)
   - Structure component tree to isolate frequent updates

10. **State Documentation**: Document state structure and management patterns
    - Create clear interfaces for state objects
    - Document state update patterns for complex state
