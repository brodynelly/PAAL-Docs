# Frontend Overall Architecture

## Architecture Type

The PAAL system frontend is built as a **monolithic single-page application (SPA)** using Next.js, a React framework that enables server-side rendering and static site generation. This architecture provides a balance between developer experience, performance, and SEO capabilities.

## Core Technologies

### Framework and Libraries

- **Next.js 14**: Provides the foundation for the application with features like file-based routing, API routes, and server components
- **React 18**: Component-based UI library for building the user interface
- **TypeScript**: Adds static typing to JavaScript for improved developer experience and code quality
- **Tailwind CSS**: Utility-first CSS framework for styling components

### State Management

- **React Context API**: Used for global state management across the application
- **React Query**: Handles server state, data fetching, caching, and synchronization
- **Local component state**: Managed using React's useState and useReducer hooks for component-specific state

### UI Component Libraries

- **Tremor**: Used for data visualization components like charts and dashboards
- **Shadcn/UI**: Collection of reusable UI components built with Radix UI and Tailwind CSS
- **Lucide Icons**: Icon library used throughout the application

### Build and Development Tools

- **pnpm**: Package manager for managing dependencies
- **ESLint**: Static code analysis tool for identifying problematic patterns
- **Prettier**: Code formatter for consistent code style

## Application Structure

The frontend application follows Next.js's file-based routing structure:

```
src/
├── app/                 # Next.js app directory (routes and pages)
│   ├── (auth)/          # Authentication-related routes
│   ├── (main)/          # Main application routes
│   │   ├── overview/    # Dashboard overview page
│   │   ├── details/     # Detailed data views
│   │   ├── admin/       # Admin panel pages
│   │   └── pigs/        # Pig management pages
│   ├── api/             # API routes for server-side operations
│   └── layout.tsx       # Root layout component
├── components/          # Reusable UI components
│   ├── ui/              # Basic UI components
│   └── [feature]/       # Feature-specific components
├── lib/                 # Utility functions and shared code
│   ├── axios.ts         # API client configuration
│   └── utils.ts         # General utility functions
├── hooks/               # Custom React hooks
└── types/               # TypeScript type definitions
```

## Rendering Strategy

The application uses a hybrid rendering approach:

- **Server-side rendering (SSR)**: For pages that require dynamic data on initial load
- **Static site generation (SSG)**: For pages with static content that can be pre-rendered
- **Client-side rendering**: For highly interactive components after initial page load

## Data Flow

1. **Data Fetching**: API requests are made using the axios client configured in `lib/axios.ts`
2. **State Management**: Data is stored in React Query cache or Context API stores
3. **Component Rendering**: Components consume data from these stores and render the UI
4. **User Interactions**: User actions trigger state updates or API calls
5. **Real-time Updates**: Socket.io is used for real-time data updates from the server

## Performance Considerations

- Code splitting via Next.js's automatic chunking
- Image optimization using Next.js Image component
- Lazy loading of components and routes
- Memoization of expensive calculations and renders

## Browser Compatibility

The application is designed to work on modern browsers including:
- Chrome (latest 2 versions)
- Firefox (latest 2 versions)
- Safari (latest 2 versions)
- Edge (latest 2 versions)

## Accessibility

The application aims to meet WCAG 2.1 AA standards with:
- Semantic HTML
- Keyboard navigation support
- Screen reader compatibility
- Sufficient color contrast
- Focus management
