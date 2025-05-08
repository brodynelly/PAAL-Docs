# UI/UX Design System

## Overview

The PAAL system implements a cohesive design system that ensures consistency across the application while providing a modern, accessible user interface. This document outlines the principles, components, and guidelines that make up the design system.

## Design Principles

### 1. Consistency

All UI elements follow consistent patterns in terms of spacing, typography, color usage, and interaction behaviors. This consistency helps users learn the interface quickly and reduces cognitive load.

### 2. Hierarchy

Visual hierarchy guides users through the interface, highlighting important information and actions while keeping secondary elements accessible but less prominent.

### 3. Accessibility

The design system prioritizes accessibility, ensuring that the application is usable by people with diverse abilities. This includes sufficient color contrast, keyboard navigation support, and screen reader compatibility.

### 4. Responsiveness

UI components adapt to different screen sizes and devices, providing an optimal experience across desktop, tablet, and mobile views.

### 5. Efficiency

The interface is designed to help users accomplish tasks with minimal effort, using clear labeling, intuitive patterns, and efficient workflows.

## Typography

### Font Families

The application uses a custom font setup with Geist Sans and Geist Mono:

```css
/* Base font configuration */
:root {
  --font-geist-sans: 'Geist Sans', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  --font-geist-mono: 'Geist Mono', Menlo, Monaco, Consolas, 'Liberation Mono', monospace;
}
```

### Type Scale

The type scale follows a consistent progression:

| Element | Size | Weight | Line Height |
|---------|------|--------|-------------|
| h1 | 2rem (32px) | 700 | 1.2 |
| h2 | 1.5rem (24px) | 700 | 1.2 |
| h3 | 1.25rem (20px) | 600 | 1.3 |
| h4 | 1.125rem (18px) | 600 | 1.4 |
| Body | 1rem (16px) | 400 | 1.5 |
| Small | 0.875rem (14px) | 400 | 1.5 |
| Caption | 0.75rem (12px) | 400 | 1.5 |

## Color System

### Primary Colors

The color palette is centered around a blue primary color with supporting neutrals:

```css
:root {
  /* Primary */
  --color-primary-50: #eef2ff;
  --color-primary-100: #e0e7ff;
  --color-primary-200: #c7d2fe;
  --color-primary-300: #a5b4fc;
  --color-primary-400: #818cf8;
  --color-primary-500: #6366f1;
  --color-primary-600: #4f46e5;
  --color-primary-700: #4338ca;
  --color-primary-800: #3730a3;
  --color-primary-900: #312e81;
  --color-primary-950: #1e1b4b;
  
  /* Neutrals */
  --color-gray-50: #f9fafb;
  --color-gray-100: #f3f4f6;
  --color-gray-200: #e5e7eb;
  --color-gray-300: #d1d5db;
  --color-gray-400: #9ca3af;
  --color-gray-500: #6b7280;
  --color-gray-600: #4b5563;
  --color-gray-700: #374151;
  --color-gray-800: #1f2937;
  --color-gray-900: #111827;
  --color-gray-950: #030712;
}
```

### Semantic Colors

Semantic colors convey meaning:

```css
:root {
  /* Success */
  --color-success-50: #f0fdf4;
  --color-success-500: #22c55e;
  --color-success-700: #15803d;
  
  /* Warning */
  --color-warning-50: #fffbeb;
  --color-warning-500: #f59e0b;
  --color-warning-700: #b45309;
  
  /* Danger */
  --color-danger-50: #fef2f2;
  --color-danger-500: #ef4444;
  --color-danger-700: #b91c1c;
  
  /* Info */
  --color-info-50: #eff6ff;
  --color-info-500: #3b82f6;
  --color-info-700: #1d4ed8;
}
```

### Color Usage Guidelines

- **Primary Colors**: Used for primary actions, navigation, and key UI elements
- **Neutrals**: Used for text, backgrounds, borders, and secondary UI elements
- **Semantic Colors**: Used to convey status, feedback, and alerts
- **Background Colors**: Light backgrounds in light mode, dark backgrounds in dark mode
- **Text Colors**: Dark text on light backgrounds, light text on dark backgrounds

## Spacing System

The spacing system uses a consistent scale based on 4px increments:

```css
:root {
  --space-1: 0.25rem; /* 4px */
  --space-2: 0.5rem;  /* 8px */
  --space-3: 0.75rem; /* 12px */
  --space-4: 1rem;    /* 16px */
  --space-5: 1.25rem; /* 20px */
  --space-6: 1.5rem;  /* 24px */
  --space-8: 2rem;    /* 32px */
  --space-10: 2.5rem; /* 40px */
  --space-12: 3rem;   /* 48px */
  --space-16: 4rem;   /* 64px */
  --space-20: 5rem;   /* 80px */
  --space-24: 6rem;   /* 96px */
}
```

## Component Library

The application uses a combination of custom components and components from Shadcn/UI and Tremor.

### Core Components

#### Button

Buttons follow a consistent style with multiple variants:

```tsx
<Button variant="primary">Primary Action</Button>
<Button variant="secondary">Secondary Action</Button>
<Button variant="outline">Outline</Button>
<Button variant="ghost">Ghost</Button>
<Button variant="link">Link</Button>
<Button variant="destructive">Destructive Action</Button>
```

Button sizes:

```tsx
<Button size="sm">Small</Button>
<Button size="default">Default</Button>
<Button size="lg">Large</Button>
```

#### Card

Cards are used to group related content:

```tsx
<Card>
  <CardHeader>
    <CardTitle>Card Title</CardTitle>
    <CardDescription>Card description text</CardDescription>
  </CardHeader>
  <CardContent>
    Main content goes here
  </CardContent>
  <CardFooter>
    Footer content or actions
  </CardFooter>
</Card>
```

#### Form Elements

Form elements include inputs, selects, checkboxes, and more:

```tsx
<Input placeholder="Enter text" />
<Select>
  <SelectTrigger>
    <SelectValue placeholder="Select an option" />
  </SelectTrigger>
  <SelectContent>
    <SelectItem value="option1">Option 1</SelectItem>
    <SelectItem value="option2">Option 2</SelectItem>
  </SelectContent>
</Select>
<Checkbox id="terms" />
<label htmlFor="terms">I agree to the terms</label>
```

#### Navigation

Navigation components include the sidebar, tabs, and breadcrumbs:

```tsx
<Tabs defaultValue="tab1">
  <TabsList>
    <TabsTrigger value="tab1">Tab 1</TabsTrigger>
    <TabsTrigger value="tab2">Tab 2</TabsTrigger>
  </TabsList>
  <TabsContent value="tab1">Tab 1 content</TabsContent>
  <TabsContent value="tab2">Tab 2 content</TabsContent>
</Tabs>
```

### Data Visualization Components

Data visualization components from Tremor are used for charts and metrics:

```tsx
<Card>
  <CardHeader>
    <CardTitle>Daily Posture</CardTitle>
  </CardHeader>
  <CardContent>
    <BarChart
      data={data}
      index="date"
      categories={["standing", "sitting", "lying"]}
      colors={["blue", "teal", "amber"]}
      valueFormatter={(value) => `${value}%`}
      yAxisWidth={48}
    />
  </CardContent>
</Card>
```

### Loading States

Loading states use skeleton components to indicate content is loading:

```tsx
<Skeleton className="h-12 w-full rounded-md" />
<Skeleton className="h-4 w-3/4 rounded-md mt-4" />
<Skeleton className="h-4 w-1/2 rounded-md mt-2" />
```

## Layout System

### Grid System

The application uses a responsive grid system based on CSS Grid and Flexbox:

```tsx
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
  <Card>Content 1</Card>
  <Card>Content 2</Card>
  <Card>Content 3</Card>
</div>
```

### Responsive Breakpoints

The application uses the following breakpoints:

```css
/* Tailwind CSS breakpoints */
sm: 640px   /* Small devices like phones */
md: 768px   /* Medium devices like tablets */
lg: 1024px  /* Large devices like laptops */
xl: 1280px  /* Extra large devices like desktops */
2xl: 1536px /* Very large screens */
```

## Icons

The application uses Lucide icons for a consistent icon set:

```tsx
import { Home, Settings, User, Bell } from "lucide-react";

<Button>
  <Home className="w-4 h-4 mr-2" />
  Home
</Button>
```

## Animation and Transitions

Animations and transitions are used sparingly to enhance the user experience:

```css
/* Base transition */
.transition-base {
  transition-property: color, background-color, border-color, text-decoration-color, fill, stroke, opacity, box-shadow, transform, filter, backdrop-filter;
  transition-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
  transition-duration: 150ms;
}

/* Fade in animation */
.animate-fade-in {
  animation: fadeIn 0.3s ease-in-out;
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}
```

## Dark Mode Support

The application supports both light and dark modes:

```tsx
<ThemeProvider defaultTheme="light" attribute="class">
  <App />
</ThemeProvider>
```

CSS variables adapt to the current theme:

```css
:root {
  --background: 0 0% 100%;
  --foreground: 222.2 84% 4.9%;
  --card: 0 0% 100%;
  --card-foreground: 222.2 84% 4.9%;
  --popover: 0 0% 100%;
  --popover-foreground: 222.2 84% 4.9%;
  --primary: 221.2 83.2% 53.3%;
  --primary-foreground: 210 40% 98%;
  --secondary: 210 40% 96.1%;
  --secondary-foreground: 222.2 47.4% 11.2%;
  --muted: 210 40% 96.1%;
  --muted-foreground: 215.4 16.3% 46.9%;
  --accent: 210 40% 96.1%;
  --accent-foreground: 222.2 47.4% 11.2%;
  --destructive: 0 84.2% 60.2%;
  --destructive-foreground: 210 40% 98%;
  --border: 214.3 31.8% 91.4%;
  --input: 214.3 31.8% 91.4%;
  --ring: 221.2 83.2% 53.3%;
}

.dark {
  --background: 222.2 84% 4.9%;
  --foreground: 210 40% 98%;
  --card: 222.2 84% 4.9%;
  --card-foreground: 210 40% 98%;
  --popover: 222.2 84% 4.9%;
  --popover-foreground: 210 40% 98%;
  --primary: 217.2 91.2% 59.8%;
  --primary-foreground: 222.2 47.4% 11.2%;
  --secondary: 217.2 32.6% 17.5%;
  --secondary-foreground: 210 40% 98%;
  --muted: 217.2 32.6% 17.5%;
  --muted-foreground: 215 20.2% 65.1%;
  --accent: 217.2 32.6% 17.5%;
  --accent-foreground: 210 40% 98%;
  --destructive: 0 62.8% 30.6%;
  --destructive-foreground: 210 40% 98%;
  --border: 217.2 32.6% 17.5%;
  --input: 217.2 32.6% 17.5%;
  --ring: 224.3 76.3% 48%;
}
```

## Accessibility Guidelines

### Color Contrast

All text and interactive elements maintain a minimum contrast ratio of 4.5:1 for normal text and 3:1 for large text and UI components.

### Keyboard Navigation

All interactive elements are keyboard accessible:
- Focusable with Tab key
- Operable with Enter/Space keys
- Visible focus indicators

### Screen Reader Support

- Semantic HTML elements are used appropriately
- ARIA attributes are added where necessary
- Alternative text is provided for images
- Form elements have associated labels

### Reduced Motion

Users with motion sensitivity can opt for reduced animations:

```css
@media (prefers-reduced-motion: reduce) {
  * {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

## Implementation Guidelines

### Component Usage

1. **Use existing components**: Always use existing components from the design system rather than creating new ones
2. **Extend, don't override**: When customization is needed, extend existing components rather than overriding their styles
3. **Maintain consistency**: Follow established patterns for similar UI elements

### Styling Approach

The application uses Tailwind CSS for styling:

```tsx
<div className="p-4 bg-white dark:bg-gray-800 rounded-lg shadow">
  <h2 className="text-xl font-semibold text-gray-900 dark:text-gray-100">
    Component Title
  </h2>
  <p className="mt-2 text-gray-600 dark:text-gray-400">
    Component description
  </p>
</div>
```

For complex components, the `cn` utility function combines class names:

```tsx
import { cn } from "@/lib/utils";

function Button({ className, variant, size, ...props }) {
  return (
    <button
      className={cn(
        "rounded-md font-medium transition-colors focus-visible:outline-none focus-visible:ring-2",
        {
          "bg-primary text-white hover:bg-primary-600": variant === "primary",
          "bg-secondary text-secondary-foreground hover:bg-secondary/80": variant === "secondary",
          // More variants...
        },
        {
          "h-9 px-3 text-sm": size === "sm",
          "h-10 px-4": size === "default",
          "h-11 px-6": size === "lg",
        },
        className
      )}
      {...props}
    />
  );
}
```
