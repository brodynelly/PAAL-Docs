# Key Components and Structure

## Overview

The PAAL frontend is organized into several key functional areas, each composed of multiple components with specific responsibilities. This document outlines the major areas, their component hierarchies, and how they interact.

## Dashboard Overview

The dashboard provides a high-level view of the system's status and key metrics.

### Component Hierarchy

```
Overview (page.tsx)
├── Filterbar
│   ├── DateRangeSelector
│   └── GroupSelector
├── ChartCard
│   ├── BarChart
│   └── LineChart
├── ProgressBarCard
├── FertilityProgressCard
├── HeatProgressCard
└── BarnStallCard
```

### Responsibilities

- **Overview**: Container component that manages data fetching and state
- **Filterbar**: Handles filtering by date range and group (farm/barn)
- **ChartCard**: Displays visualizations of pig data
- **ProgressBarCard**: Shows progress metrics with visual indicators
- **FertilityProgressCard**: Displays fertility-specific metrics
- **HeatProgressCard**: Shows heat cycle information
- **BarnStallCard**: Displays barn and stall occupancy information

### Data Flow

1. The Overview component fetches data from `/api/pigs/overview` and `/api/stats`
2. Data is passed to child components as props
3. Filter components emit events when filters change
4. The Overview component re-fetches data with updated filter parameters

## Pig Details Page

Displays detailed information about individual pigs, including health records and posture data.

### Component Hierarchy

```
PigDashboard (page.tsx)
├── PigHeader
│   ├── Badge (status)
│   └── ActionButtons
├── Tabs
│   ├── OverviewTab
│   │   ├── HealthMetricCard
│   │   ├── LocationCard
│   │   └── HistoryTimeline
│   ├── PostureTab
│   │   ├── DateRangeSelectorWithApply
│   │   ├── TransactionChart
│   │   └── PostureDataTable
│   └── HealthTab
│       ├── BCSChart
│       └── HealthRecordsTable
├── AddHealthRecordDrawer
└── EditPigDrawer
```

### Responsibilities

- **PigDashboard**: Container component that manages data fetching and state
- **PigHeader**: Displays pig identification and status information
- **ActionButtons**: Provides actions like editing pig details and adding health records
- **OverviewTab**: Shows summary information about the pig
- **PostureTab**: Displays posture data visualizations and tables
- **HealthTab**: Shows health records and body condition score data
- **Drawers**: Modal interfaces for adding/editing data

### Data Flow

1. The PigDashboard fetches pig data from `/api/pigs/{id}`
2. Additional data is fetched from endpoints like `/api/pigs/{id}/posture/aggregated` and `/api/pigs/{id}/bcs`
3. Data is passed to tab components based on the active tab
4. User actions in drawers trigger API calls to update data
5. After successful updates, the main data is refreshed

## Admin Farm Management

Manages farms, barns, and stalls in the system.

### Component Hierarchy

```
FarmManagement (page.tsx)
├── AdminPageHeader
├── Tabs
│   ├── GridView
│   │   └── AdminFarmCard
│   └── TableView
│       └── FarmTable
├── CreateFarmModal
└── FarmDetailsModal
    ├── BarnList
    │   └── BarnCard
    ├── StallList
    │   └── StallCard
    └── DeviceList
        └── DeviceCard
```

### Responsibilities

- **FarmManagement**: Container component that manages data fetching and state
- **AdminPageHeader**: Displays page title and action buttons
- **GridView/TableView**: Different visualizations of farm data
- **AdminFarmCard**: Card component displaying farm information
- **CreateFarmModal**: Interface for adding new farms
- **FarmDetailsModal**: Detailed view of a farm with related entities

### Data Flow

1. The FarmManagement component fetches farm data from `/api/farms`
2. When a farm is selected, detailed data is fetched from `/api/farms/{id}`
3. Create/update actions trigger API calls to modify data
4. After successful operations, the farm list is refreshed

## User Management

Manages user accounts and permissions.

### Component Hierarchy

```
UserManagement (page.tsx)
├── AdminPageHeader
├── UserFilters
├── UserTable
│   └── UserRow
├── CreateUserModal
│   ├── UserForm
│   └── RoleSelector
└── EditUserModal
    ├── UserForm
    ├── RoleSelector
    └── PermissionsForm
```

### Responsibilities

- **UserManagement**: Container component that manages data fetching and state
- **UserFilters**: Filtering options for the user list
- **UserTable**: Displays user information in a tabular format
- **CreateUserModal**: Interface for adding new users
- **EditUserModal**: Interface for modifying existing users

### Data Flow

1. The UserManagement component fetches user data from `/api/users`
2. When a user is selected for editing, detailed data is fetched from `/api/users/{id}`
3. Create/update actions trigger API calls to modify data
4. After successful operations, the user list is refreshed

## Common Components

Several reusable components are used throughout the application:

### UI Components

- **Card**: Container component with consistent styling
- **Button**: Various button styles (primary, secondary, etc.)
- **Tabs**: Tabbed interface component
- **Select**: Dropdown selection component
- **DatePicker**: Date selection component
- **Table**: Data table component with sorting and pagination
- **Modal/Drawer**: Overlay components for forms and detailed views
- **Skeleton**: Loading placeholder components

### Layout Components

- **Sidebar**: Main navigation sidebar
- **Header**: Page header with user information and actions
- **ConditionalLayout**: Layout wrapper that adapts based on authentication state
- **AuthProvider**: Authentication context provider

### Data Visualization

- **BarChart**: Bar chart visualization
- **LineChart**: Line chart visualization
- **ProgressCircle**: Circular progress indicator
- **CategoryBar**: Horizontal category-based progress bar
