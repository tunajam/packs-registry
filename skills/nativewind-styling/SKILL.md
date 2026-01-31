# NativeWind Styling

Use Tailwind CSS to style React Native applications.

## When to Apply

- Rapid UI development
- Consistent design systems
- Teams familiar with Tailwind
- Cross-platform styling

## Installation

### Expo
```bash
npx expo install nativewind tailwindcss

# Initialize Tailwind
npx tailwindcss init
```

### Configure Tailwind
```js
// tailwind.config.js
module.exports = {
  content: [
    './App.{js,jsx,ts,tsx}',
    './app/**/*.{js,jsx,ts,tsx}',
    './components/**/*.{js,jsx,ts,tsx}',
  ],
  presets: [require('nativewind/preset')],
  theme: {
    extend: {},
  },
  plugins: [],
};
```

### Configure Babel
```js
// babel.config.js
module.exports = function (api) {
  api.cache(true);
  return {
    presets: [
      ['babel-preset-expo', { jsxImportSource: 'nativewind' }],
      'nativewind/babel',
    ],
  };
};
```

### Global CSS
```css
/* global.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

```tsx
// App.tsx
import './global.css';

export default function App() {
  return <RootLayout />;
}
```

### TypeScript Setup
```ts
// nativewind-env.d.ts
/// <reference types="nativewind/types" />
```

## Basic Usage

```tsx
import { View, Text, Pressable } from 'react-native';

export function MyComponent() {
  return (
    <View className="flex-1 justify-center items-center bg-white">
      <Text className="text-xl font-bold text-gray-900">
        Hello NativeWind
      </Text>
      <Pressable className="bg-blue-500 px-4 py-2 rounded-lg mt-4">
        <Text className="text-white font-medium">Press me</Text>
      </Pressable>
    </View>
  );
}
```

## Layout Classes

### Flexbox
```tsx
// Flex container
<View className="flex flex-row">
<View className="flex flex-col">
<View className="flex-1">  // flex: 1
<View className="flex-none">

// Justify content
<View className="justify-start">
<View className="justify-center">
<View className="justify-end">
<View className="justify-between">
<View className="justify-around">

// Align items
<View className="items-start">
<View className="items-center">
<View className="items-end">
<View className="items-stretch">

// Gap
<View className="gap-4">
<View className="gap-x-2 gap-y-4">
```

### Spacing
```tsx
// Padding
<View className="p-4">      // all sides
<View className="px-4">     // horizontal
<View className="py-2">     // vertical
<View className="pt-4 pb-2"> // top, bottom
<View className="pl-4 pr-2"> // left, right

// Margin
<View className="m-4">
<View className="mx-auto">  // center horizontally
<View className="mt-4 mb-2">
```

### Sizing
```tsx
// Width/Height
<View className="w-full h-full">
<View className="w-1/2 h-1/3">
<View className="w-64 h-32">     // fixed pixels
<View className="min-w-0 max-w-sm">
<View className="aspect-square">
<View className="aspect-video">
```

## Typography

```tsx
// Font size
<Text className="text-xs">    // 12px
<Text className="text-sm">    // 14px
<Text className="text-base">  // 16px
<Text className="text-lg">    // 18px
<Text className="text-xl">    // 20px
<Text className="text-2xl">   // 24px

// Font weight
<Text className="font-normal">
<Text className="font-medium">
<Text className="font-semibold">
<Text className="font-bold">

// Text alignment
<Text className="text-left">
<Text className="text-center">
<Text className="text-right">

// Line height
<Text className="leading-tight">
<Text className="leading-normal">
<Text className="leading-loose">

// Text color
<Text className="text-gray-900">
<Text className="text-blue-500">
<Text className="text-red-600">
```

## Colors & Backgrounds

```tsx
// Background color
<View className="bg-white">
<View className="bg-gray-100">
<View className="bg-blue-500">
<View className="bg-transparent">

// Opacity
<View className="bg-black/50">  // 50% opacity black
<View className="opacity-50">

// Gradients (via LinearGradient)
<LinearGradient
  className="flex-1"
  colors={['#4c669f', '#3b5998', '#192f6a']}
/>
```

## Borders

```tsx
// Border width
<View className="border">       // 1px all sides
<View className="border-2">     // 2px
<View className="border-t">     // top only
<View className="border-b-2">   // bottom 2px

// Border color
<View className="border-gray-300">
<View className="border-blue-500">

// Border radius
<View className="rounded">      // 4px
<View className="rounded-md">   // 6px
<View className="rounded-lg">   // 8px
<View className="rounded-xl">   // 12px
<View className="rounded-full"> // circular
```

## Shadows

```tsx
<View className="shadow-sm">
<View className="shadow">
<View className="shadow-md">
<View className="shadow-lg">
<View className="shadow-xl">
```

## Responsive Design

```tsx
// Platform-specific (not Tailwind breakpoints)
import { Platform } from 'react-native';

<View className={Platform.OS === 'ios' ? 'pt-12' : 'pt-8'}>

// Screen size with useWindowDimensions
import { useWindowDimensions } from 'react-native';

function ResponsiveLayout() {
  const { width } = useWindowDimensions();
  const isLargeScreen = width >= 768;
  
  return (
    <View className={`flex-1 ${isLargeScreen ? 'flex-row' : 'flex-col'}`}>
      {/* content */}
    </View>
  );
}
```

## Dark Mode

```tsx
// Enable dark mode
// tailwind.config.js
module.exports = {
  darkMode: 'class',
  // ...
};

// Usage
<View className="bg-white dark:bg-gray-900">
  <Text className="text-black dark:text-white">
    Adapts to dark mode
  </Text>
</View>
```

### Toggle Dark Mode
```tsx
import { useColorScheme } from 'nativewind';

function ThemeToggle() {
  const { colorScheme, toggleColorScheme } = useColorScheme();
  
  return (
    <Pressable onPress={toggleColorScheme}>
      <Text>{colorScheme === 'dark' ? '🌙' : '☀️'}</Text>
    </Pressable>
  );
}
```

## Custom Theme

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#eff6ff',
          500: '#3b82f6',
          900: '#1e3a8a',
        },
        brand: '#FF6B00',
      },
      fontFamily: {
        sans: ['Inter'],
        mono: ['FiraCode'],
      },
      spacing: {
        '18': '4.5rem',
        '112': '28rem',
      },
    },
  },
};
```

## Component Patterns

### Button Component
```tsx
interface ButtonProps {
  variant?: 'primary' | 'secondary' | 'outline';
  size?: 'sm' | 'md' | 'lg';
  children: React.ReactNode;
  onPress: () => void;
}

export function Button({ 
  variant = 'primary', 
  size = 'md',
  children, 
  onPress 
}: ButtonProps) {
  const baseClasses = 'rounded-lg items-center justify-center';
  
  const variants = {
    primary: 'bg-blue-500 active:bg-blue-600',
    secondary: 'bg-gray-200 active:bg-gray-300',
    outline: 'border border-blue-500 bg-transparent',
  };
  
  const sizes = {
    sm: 'px-3 py-1.5',
    md: 'px-4 py-2',
    lg: 'px-6 py-3',
  };
  
  const textColors = {
    primary: 'text-white',
    secondary: 'text-gray-900',
    outline: 'text-blue-500',
  };

  return (
    <Pressable 
      className={`${baseClasses} ${variants[variant]} ${sizes[size]}`}
      onPress={onPress}
    >
      <Text className={`font-medium ${textColors[variant]}`}>
        {children}
      </Text>
    </Pressable>
  );
}
```

### Card Component
```tsx
export function Card({ children, className = '' }) {
  return (
    <View className={`bg-white rounded-xl shadow-md p-4 ${className}`}>
      {children}
    </View>
  );
}
```

## Best Practices

1. **Use consistent spacing** - Stick to the spacing scale (4, 8, 12, 16...)
2. **Extract components** - Reuse common patterns
3. **Leverage dark mode** - Design for both themes
4. **Test on device** - Shadows render differently
5. **Keep className simple** - Break into components if too long
6. **Use custom theme** - Define brand colors once
