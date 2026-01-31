# React Navigation Patterns

Comprehensive patterns for React Navigation in React Native apps.

## When to Apply

- Setting up navigation structure
- Implementing deep linking
- Creating authenticated flows
- Building complex navigation hierarchies
- TypeScript navigation typing

## Navigator Types

### Native Stack (Recommended)
```tsx
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const Stack = createNativeStackNavigator<RootStackParamList>();

function RootStack() {
  return (
    <Stack.Navigator>
      <Stack.Screen name="Home" component={HomeScreen} />
      <Stack.Screen name="Details" component={DetailsScreen} />
    </Stack.Navigator>
  );
}
```

**Why Native Stack?** Uses native navigation primitives, better performance than JS-based stack.

### Bottom Tabs
```tsx
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';

const Tab = createBottomTabNavigator<TabParamList>();

function TabNavigator() {
  return (
    <Tab.Navigator
      screenOptions={({ route }) => ({
        tabBarIcon: ({ focused, color, size }) => {
          const icons = { Home: 'home', Settings: 'settings' };
          return <Icon name={icons[route.name]} size={size} color={color} />;
        },
      })}
    >
      <Tab.Screen name="Home" component={HomeStack} />
      <Tab.Screen name="Settings" component={SettingsScreen} />
    </Tab.Navigator>
  );
}
```

### Drawer
```tsx
import { createDrawerNavigator } from '@react-navigation/drawer';

const Drawer = createDrawerNavigator();

function DrawerNavigator() {
  return (
    <Drawer.Navigator
      drawerContent={(props) => <CustomDrawer {...props} />}
    >
      <Drawer.Screen name="Home" component={HomeScreen} />
    </Drawer.Navigator>
  );
}
```

## TypeScript Setup

### Define Param Lists
```tsx
// types/navigation.ts
export type RootStackParamList = {
  Home: undefined;
  Details: { itemId: string };
  Profile: { userId: string; name?: string };
};

export type TabParamList = {
  Feed: undefined;
  Search: { query?: string };
  Profile: undefined;
};

// Declare globally for useNavigation hook
declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}
```

### Typed Screen Props
```tsx
import { NativeStackScreenProps } from '@react-navigation/native-stack';

type Props = NativeStackScreenProps<RootStackParamList, 'Details'>;

function DetailsScreen({ route, navigation }: Props) {
  const { itemId } = route.params;
  // navigation.navigate('Home') - fully typed
}
```

### Typed Navigation Hook
```tsx
import { useNavigation, useRoute } from '@react-navigation/native';
import { NativeStackNavigationProp } from '@react-navigation/native-stack';

type NavigationProp = NativeStackNavigationProp<RootStackParamList>;

function MyComponent() {
  const navigation = useNavigation<NavigationProp>();
  navigation.navigate('Details', { itemId: '123' });
}
```

## Deep Linking

### Configure Linking
```tsx
const linking = {
  prefixes: ['myapp://', 'https://myapp.com'],
  config: {
    screens: {
      Home: '',
      Details: 'details/:itemId',
      Profile: {
        path: 'user/:userId',
        parse: {
          userId: (userId: string) => userId,
        },
      },
      NotFound: '*',
    },
  },
};

function App() {
  return (
    <NavigationContainer linking={linking} fallback={<Loading />}>
      <RootStack />
    </NavigationContainer>
  );
}
```

### Handle Deep Links
```tsx
// Listen to incoming links
import { Linking } from 'react-native';

useEffect(() => {
  const handleDeepLink = ({ url }: { url: string }) => {
    // Handle the URL
  };

  Linking.addEventListener('url', handleDeepLink);
  
  // Handle app opened via deep link
  Linking.getInitialURL().then((url) => {
    if (url) handleDeepLink({ url });
  });
}, []);
```

## Authentication Flow

### Auth Context Pattern
```tsx
const AuthContext = createContext<{
  signIn: (token: string) => void;
  signOut: () => void;
  isLoading: boolean;
  userToken: string | null;
}>({} as any);

function RootNavigator() {
  const { userToken, isLoading } = useContext(AuthContext);

  if (isLoading) {
    return <SplashScreen />;
  }

  return (
    <Stack.Navigator screenOptions={{ headerShown: false }}>
      {userToken == null ? (
        <Stack.Screen name="Auth" component={AuthStack} />
      ) : (
        <Stack.Screen name="App" component={AppStack} />
      )}
    </Stack.Navigator>
  );
}
```

## Modal Patterns

### Presentation Modes
```tsx
<Stack.Navigator>
  <Stack.Group>
    <Stack.Screen name="Home" component={HomeScreen} />
  </Stack.Group>
  
  <Stack.Group screenOptions={{ presentation: 'modal' }}>
    <Stack.Screen name="CreatePost" component={CreatePostScreen} />
  </Stack.Group>
  
  <Stack.Group screenOptions={{ presentation: 'transparentModal' }}>
    <Stack.Screen name="Alert" component={AlertScreen} />
  </Stack.Group>
</Stack.Navigator>
```

## Navigation State Persistence

```tsx
const PERSISTENCE_KEY = 'NAVIGATION_STATE';

function App() {
  const [isReady, setIsReady] = useState(false);
  const [initialState, setInitialState] = useState();

  useEffect(() => {
    const restoreState = async () => {
      const savedState = await AsyncStorage.getItem(PERSISTENCE_KEY);
      if (savedState) {
        setInitialState(JSON.parse(savedState));
      }
      setIsReady(true);
    };

    if (!isReady) restoreState();
  }, [isReady]);

  if (!isReady) return null;

  return (
    <NavigationContainer
      initialState={initialState}
      onStateChange={(state) =>
        AsyncStorage.setItem(PERSISTENCE_KEY, JSON.stringify(state))
      }
    >
      <RootStack />
    </NavigationContainer>
  );
}
```

## Best Practices

1. **Always use Native Stack** - Better performance than JS stack
2. **Type everything** - Catch navigation errors at compile time
3. **Split navigators** - Don't put all screens in one navigator
4. **Lazy load screens** - Use React.lazy for heavy screens
5. **Handle deep links** - Test on both platforms
6. **Reset navigation** - Use `navigation.reset()` for auth flows
