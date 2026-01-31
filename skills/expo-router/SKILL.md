# Expo Router

File-based routing for universal Expo apps.

## When to Apply

- Starting new Expo projects
- Building universal (mobile + web) apps
- Wanting Next.js-like routing in React Native
- Simplifying navigation structure

## Setup

### New Project
```bash
npx create-expo-app@latest --template tabs
```

### Existing Project
```bash
npx expo install expo-router expo-linking expo-constants
```

### Configuration
```json
// app.json
{
  "expo": {
    "scheme": "myapp",
    "web": {
      "bundler": "metro"
    }
  }
}
```

```json
// package.json
{
  "main": "expo-router/entry"
}
```

## File Structure

```
app/
├── _layout.tsx        # Root layout
├── index.tsx          # Home (/)
├── about.tsx          # /about
├── settings/
│   ├── _layout.tsx    # Settings layout
│   ├── index.tsx      # /settings
│   └── profile.tsx    # /settings/profile
├── [id].tsx           # Dynamic route (/123)
├── [...rest].tsx      # Catch-all route
└── (tabs)/
    ├── _layout.tsx    # Tab layout
    ├── home.tsx       # Tab: home
    └── search.tsx     # Tab: search
```

## Layouts

### Root Layout
```tsx
// app/_layout.tsx
import { Stack } from 'expo-router';

export default function RootLayout() {
  return (
    <Stack>
      <Stack.Screen name="index" options={{ title: 'Home' }} />
      <Stack.Screen name="about" options={{ title: 'About' }} />
      <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
    </Stack>
  );
}
```

### Tab Layout
```tsx
// app/(tabs)/_layout.tsx
import { Tabs } from 'expo-router';
import { Ionicons } from '@expo/vector-icons';

export default function TabLayout() {
  return (
    <Tabs screenOptions={{ tabBarActiveTintColor: '#007AFF' }}>
      <Tabs.Screen
        name="home"
        options={{
          title: 'Home',
          tabBarIcon: ({ color }) => (
            <Ionicons name="home" size={24} color={color} />
          ),
        }}
      />
      <Tabs.Screen
        name="search"
        options={{
          title: 'Search',
          tabBarIcon: ({ color }) => (
            <Ionicons name="search" size={24} color={color} />
          ),
        }}
      />
    </Tabs>
  );
}
```

## Navigation

### Link Component
```tsx
import { Link } from 'expo-router';

// Basic link
<Link href="/about">About</Link>

// With params
<Link href="/user/123">User Profile</Link>

// Object href
<Link href={{
  pathname: '/user/[id]',
  params: { id: '123' }
}}>
  User Profile
</Link>

// Replace instead of push
<Link href="/home" replace>Home</Link>

// As button
<Link href="/settings" asChild>
  <Pressable>
    <Text>Settings</Text>
  </Pressable>
</Link>
```

### useRouter Hook
```tsx
import { useRouter } from 'expo-router';

function MyComponent() {
  const router = useRouter();

  return (
    <View>
      <Button
        title="Go to Profile"
        onPress={() => router.push('/profile')}
      />
      
      <Button
        title="Replace with Home"
        onPress={() => router.replace('/home')}
      />
      
      <Button
        title="Go Back"
        onPress={() => router.back()}
      />
      
      <Button
        title="Navigate with params"
        onPress={() => router.push({
          pathname: '/user/[id]',
          params: { id: '456' }
        })}
      />
    </View>
  );
}
```

## Dynamic Routes

### Single Parameter
```tsx
// app/user/[id].tsx
import { useLocalSearchParams } from 'expo-router';

export default function UserScreen() {
  const { id } = useLocalSearchParams<{ id: string }>();
  
  return <Text>User ID: {id}</Text>;
}

// Navigate: /user/123
```

### Multiple Parameters
```tsx
// app/post/[category]/[id].tsx
import { useLocalSearchParams } from 'expo-router';

export default function PostScreen() {
  const { category, id } = useLocalSearchParams<{
    category: string;
    id: string;
  }>();
  
  return <Text>Post {id} in {category}</Text>;
}

// Navigate: /post/tech/456
```

### Catch-All Routes
```tsx
// app/[...rest].tsx
import { useLocalSearchParams } from 'expo-router';

export default function CatchAll() {
  const { rest } = useLocalSearchParams<{ rest: string[] }>();
  
  return <Text>Path: {rest?.join('/')}</Text>;
}

// Matches: /anything/goes/here
```

## Route Groups

Groups organize routes without affecting URLs:

```
app/
├── (auth)/
│   ├── _layout.tsx
│   ├── login.tsx      # /login
│   └── register.tsx   # /register
├── (main)/
│   ├── _layout.tsx
│   ├── home.tsx       # /home
│   └── profile.tsx    # /profile
└── _layout.tsx
```

### Conditional Layouts
```tsx
// app/_layout.tsx
import { useAuth } from '../hooks/useAuth';

export default function RootLayout() {
  const { isAuthenticated } = useAuth();

  return (
    <Stack>
      {!isAuthenticated ? (
        <Stack.Screen name="(auth)" options={{ headerShown: false }} />
      ) : (
        <Stack.Screen name="(main)" options={{ headerShown: false }} />
      )}
    </Stack>
  );
}
```

## Modals

```tsx
// app/_layout.tsx
export default function RootLayout() {
  return (
    <Stack>
      <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
      <Stack.Screen
        name="modal"
        options={{
          presentation: 'modal',
          title: 'Modal',
        }}
      />
    </Stack>
  );
}

// Navigate to modal
router.push('/modal');
```

## Deep Linking

```json
// app.json
{
  "expo": {
    "scheme": "myapp"
  }
}
```

Routes automatically support deep links:
- `myapp://home` → `/home`
- `myapp://user/123` → `/user/123`

### Handle External Links
```tsx
import { useURL } from 'expo-linking';
import { useRouter } from 'expo-router';

function DeepLinkHandler() {
  const url = useURL();
  const router = useRouter();

  useEffect(() => {
    if (url) {
      router.push(url);
    }
  }, [url]);

  return null;
}
```

## Error Handling

### Not Found
```tsx
// app/+not-found.tsx
import { Link, Stack } from 'expo-router';

export default function NotFoundScreen() {
  return (
    <>
      <Stack.Screen options={{ title: 'Oops!' }} />
      <View style={styles.container}>
        <Text>This screen doesn't exist.</Text>
        <Link href="/" style={styles.link}>
          Go to home screen
        </Link>
      </View>
    </>
  );
}
```

### Error Boundary
```tsx
// app/_layout.tsx
import { ErrorBoundary } from 'expo-router';

export function unstable_settings() {
  return {
    initialRouteName: 'index',
  };
}

export { ErrorBoundary };

export default function RootLayout() {
  return <Stack />;
}
```

## Loading States

```tsx
// app/user/[id].tsx
import { useLocalSearchParams } from 'expo-router';

export default function UserScreen() {
  const { id } = useLocalSearchParams();
  const { data, isLoading } = useUser(id);

  if (isLoading) {
    return <LoadingScreen />;
  }

  return <UserProfile user={data} />;
}
```

## Protected Routes

```tsx
// app/(protected)/_layout.tsx
import { Redirect, Stack } from 'expo-router';
import { useAuth } from '../../hooks/useAuth';

export default function ProtectedLayout() {
  const { isAuthenticated, isLoading } = useAuth();

  if (isLoading) {
    return <LoadingScreen />;
  }

  if (!isAuthenticated) {
    return <Redirect href="/login" />;
  }

  return <Stack />;
}
```

## TypeScript

### Typed Routes
```tsx
// Auto-generated types in .expo/types/router.d.ts
import { Href } from 'expo-router';

// Type-safe navigation
const href: Href = '/user/123';
const href2: Href = { pathname: '/user/[id]', params: { id: '123' } };
```

### Generate Types
```bash
npx expo customize tsconfig.json
```

## Best Practices

1. **Use route groups** - Organize by feature/auth state
2. **Keep layouts simple** - Put logic in components
3. **Type your params** - Avoid runtime errors
4. **Handle loading states** - Show skeletons/spinners
5. **Protect routes** - Redirect unauthenticated users
6. **Test deep links** - On both platforms
7. **Use Link component** - For accessible navigation
