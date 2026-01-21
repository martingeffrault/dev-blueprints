# React Native & Expo (2025)

> **Last updated**: January 2026
> **Versions covered**: React Native 0.76+, Expo SDK 52+
> **Purpose**: Cross-platform mobile development with native performance

---

## Philosophy (2025-2026)

React Native in 2025 is **Expo-first**. The New Architecture (Fabric + TurboModules) is now default, and the legacy architecture may be removed in late 2025. Expo has evolved from a beginner wrapper into a robust production framework.

**Key shifts:**
- **New Architecture is default** — 75% of SDK 52+ projects use it
- **Expo Router for navigation** — File-based routing like Next.js
- **Feature-based architecture** — Group by feature, not by type
- **Zustand over Redux** — Simpler state management
- **EAS Build/Update** — Cloud builds and OTA updates

---

## TL;DR

- Use Expo SDK 52+ with New Architecture enabled
- Use Expo Router for file-based navigation
- Structure by feature, not by file type
- Use Zustand for state, TanStack Query for server state
- Functional components with hooks only
- Use `react-native-reanimated` for animations
- Test with Jest + React Native Testing Library
- Deploy with EAS Build and EAS Update

---

## Best Practices

### Project Structure (Feature-Based)

```
my-app/
├── app/                        # Expo Router screens
│   ├── (tabs)/                 # Tab navigation group
│   │   ├── _layout.tsx
│   │   ├── index.tsx           # Home tab
│   │   ├── search.tsx          # Search tab
│   │   └── profile.tsx         # Profile tab
│   ├── (auth)/                 # Auth flow group
│   │   ├── _layout.tsx
│   │   ├── login.tsx
│   │   └── register.tsx
│   ├── posts/
│   │   ├── [id].tsx            # Dynamic route
│   │   └── create.tsx
│   ├── _layout.tsx             # Root layout
│   └── +not-found.tsx          # 404 screen
├── src/
│   ├── features/               # Feature modules
│   │   ├── auth/
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── services/
│   │   │   ├── store.ts
│   │   │   └── types.ts
│   │   ├── posts/
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── services/
│   │   │   └── types.ts
│   │   └── user/
│   ├── components/             # Shared components
│   │   ├── ui/
│   │   │   ├── Button.tsx
│   │   │   ├── Input.tsx
│   │   │   └── Card.tsx
│   │   └── layout/
│   ├── hooks/                  # Shared hooks
│   ├── services/               # API clients
│   ├── stores/                 # Global stores
│   ├── utils/                  # Utilities
│   ├── constants/              # App constants
│   └── types/                  # Global types
├── assets/
│   ├── images/
│   └── fonts/
├── app.json
├── eas.json
├── tsconfig.json
└── package.json
```

### Expo Router Navigation

```tsx
// app/_layout.tsx - Root layout
import { Stack } from 'expo-router';
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from '@/services/query-client';

export default function RootLayout() {
  return (
    <QueryClientProvider client={queryClient}>
      <Stack>
        <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
        <Stack.Screen name="(auth)" options={{ headerShown: false }} />
        <Stack.Screen
          name="posts/[id]"
          options={{ title: 'Post Details' }}
        />
      </Stack>
    </QueryClientProvider>
  );
}
```

```tsx
// app/(tabs)/_layout.tsx - Tab navigation
import { Tabs } from 'expo-router';
import { Ionicons } from '@expo/vector-icons';

export default function TabLayout() {
  return (
    <Tabs screenOptions={{ tabBarActiveTintColor: '#007AFF' }}>
      <Tabs.Screen
        name="index"
        options={{
          title: 'Home',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="home" size={size} color={color} />
          ),
        }}
      />
      <Tabs.Screen
        name="search"
        options={{
          title: 'Search',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="search" size={size} color={color} />
          ),
        }}
      />
      <Tabs.Screen
        name="profile"
        options={{
          title: 'Profile',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="person" size={size} color={color} />
          ),
        }}
      />
    </Tabs>
  );
}
```

### Components

```tsx
// src/components/ui/Button.tsx
import { Pressable, Text, StyleSheet, ActivityIndicator } from 'react-native';
import Animated, {
  useAnimatedStyle,
  useSharedValue,
  withSpring
} from 'react-native-reanimated';

interface ButtonProps {
  title: string;
  onPress: () => void;
  variant?: 'primary' | 'secondary' | 'outline';
  isLoading?: boolean;
  disabled?: boolean;
}

const AnimatedPressable = Animated.createAnimatedComponent(Pressable);

export function Button({
  title,
  onPress,
  variant = 'primary',
  isLoading = false,
  disabled = false,
}: ButtonProps) {
  const scale = useSharedValue(1);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
  }));

  const handlePressIn = () => {
    scale.value = withSpring(0.95);
  };

  const handlePressOut = () => {
    scale.value = withSpring(1);
  };

  return (
    <AnimatedPressable
      style={[styles.button, styles[variant], animatedStyle]}
      onPress={onPress}
      onPressIn={handlePressIn}
      onPressOut={handlePressOut}
      disabled={disabled || isLoading}
    >
      {isLoading ? (
        <ActivityIndicator color="#fff" />
      ) : (
        <Text style={[styles.text, styles[`${variant}Text`]]}>{title}</Text>
      )}
    </AnimatedPressable>
  );
}

const styles = StyleSheet.create({
  button: {
    paddingVertical: 12,
    paddingHorizontal: 24,
    borderRadius: 8,
    alignItems: 'center',
    justifyContent: 'center',
  },
  primary: {
    backgroundColor: '#007AFF',
  },
  secondary: {
    backgroundColor: '#5856D6',
  },
  outline: {
    backgroundColor: 'transparent',
    borderWidth: 1,
    borderColor: '#007AFF',
  },
  text: {
    fontSize: 16,
    fontWeight: '600',
  },
  primaryText: {
    color: '#fff',
  },
  secondaryText: {
    color: '#fff',
  },
  outlineText: {
    color: '#007AFF',
  },
});
```

### State Management with Zustand

```tsx
// src/stores/auth-store.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

interface User {
  id: string;
  email: string;
  name: string;
}

interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  setUser: (user: User, token: string) => void;
  logout: () => void;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      user: null,
      token: null,
      isAuthenticated: false,
      isLoading: false,
      setUser: (user, token) =>
        set({ user, token, isAuthenticated: true }),
      logout: () =>
        set({ user: null, token: null, isAuthenticated: false }),
    }),
    {
      name: 'auth-storage',
      storage: createJSONStorage(() => AsyncStorage),
    }
  )
);
```

### Data Fetching with TanStack Query

```tsx
// src/features/posts/hooks/use-posts.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { postsService } from '../services/posts-service';
import type { Post, CreatePostInput } from '../types';

export function usePosts() {
  return useQuery({
    queryKey: ['posts'],
    queryFn: postsService.getAll,
  });
}

export function usePost(id: string) {
  return useQuery({
    queryKey: ['posts', id],
    queryFn: () => postsService.getById(id),
    enabled: !!id,
  });
}

export function useCreatePost() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: CreatePostInput) => postsService.create(data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['posts'] });
    },
  });
}
```

```tsx
// src/features/posts/services/posts-service.ts
import { api } from '@/services/api';
import type { Post, CreatePostInput } from '../types';

export const postsService = {
  getAll: async (): Promise<Post[]> => {
    const response = await api.get('/posts');
    return response.data;
  },

  getById: async (id: string): Promise<Post> => {
    const response = await api.get(`/posts/${id}`);
    return response.data;
  },

  create: async (data: CreatePostInput): Promise<Post> => {
    const response = await api.post('/posts', data);
    return response.data;
  },
};
```

### API Client with Interceptors

```tsx
// src/services/api.ts
import axios from 'axios';
import { useAuthStore } from '@/stores/auth-store';

export const api = axios.create({
  baseURL: process.env.EXPO_PUBLIC_API_URL,
  timeout: 10000,
});

// Request interceptor
api.interceptors.request.use((config) => {
  const token = useAuthStore.getState().token;
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Response interceptor
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      useAuthStore.getState().logout();
    }
    return Promise.reject(error);
  }
);
```

### Form Handling with Validation

```tsx
// src/features/auth/components/LoginForm.tsx
import { View, StyleSheet } from 'react-native';
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { Input } from '@/components/ui/Input';
import { Button } from '@/components/ui/Button';
import { useLogin } from '../hooks/use-login';

const loginSchema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
});

type LoginForm = z.infer<typeof loginSchema>;

export function LoginForm() {
  const { mutate: login, isPending } = useLogin();

  const { control, handleSubmit, formState: { errors } } = useForm<LoginForm>({
    resolver: zodResolver(loginSchema),
    defaultValues: {
      email: '',
      password: '',
    },
  });

  const onSubmit = (data: LoginForm) => {
    login(data);
  };

  return (
    <View style={styles.container}>
      <Controller
        control={control}
        name="email"
        render={({ field: { onChange, onBlur, value } }) => (
          <Input
            placeholder="Email"
            keyboardType="email-address"
            autoCapitalize="none"
            value={value}
            onChangeText={onChange}
            onBlur={onBlur}
            error={errors.email?.message}
          />
        )}
      />

      <Controller
        control={control}
        name="password"
        render={({ field: { onChange, onBlur, value } }) => (
          <Input
            placeholder="Password"
            secureTextEntry
            value={value}
            onChangeText={onChange}
            onBlur={onBlur}
            error={errors.password?.message}
          />
        )}
      />

      <Button
        title="Login"
        onPress={handleSubmit(onSubmit)}
        isLoading={isPending}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    gap: 16,
  },
});
```

### Optimized FlatList

```tsx
// src/features/posts/components/PostList.tsx
import { FlatList, StyleSheet, RefreshControl } from 'react-native';
import { useCallback, useMemo } from 'react';
import { PostCard } from './PostCard';
import { usePosts } from '../hooks/use-posts';
import type { Post } from '../types';

export function PostList() {
  const { data: posts, isLoading, refetch, isRefetching } = usePosts();

  // Stable renderItem reference
  const renderItem = useCallback(
    ({ item }: { item: Post }) => <PostCard post={item} />,
    []
  );

  // Stable keyExtractor
  const keyExtractor = useCallback((item: Post) => item.id, []);

  // Memoized item separator
  const ItemSeparator = useMemo(
    () => () => <View style={styles.separator} />,
    []
  );

  return (
    <FlatList
      data={posts}
      renderItem={renderItem}
      keyExtractor={keyExtractor}
      ItemSeparatorComponent={ItemSeparator}
      refreshControl={
        <RefreshControl refreshing={isRefetching} onRefresh={refetch} />
      }
      // Performance optimizations
      removeClippedSubviews
      maxToRenderPerBatch={10}
      windowSize={5}
      initialNumToRender={10}
      getItemLayout={(_, index) => ({
        length: ITEM_HEIGHT,
        offset: ITEM_HEIGHT * index,
        index,
      })}
    />
  );
}

const ITEM_HEIGHT = 120;

const styles = StyleSheet.create({
  separator: {
    height: 12,
  },
});
```

### Animations with Reanimated

```tsx
// src/components/AnimatedCard.tsx
import { StyleSheet } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  FadeInDown,
  Layout,
} from 'react-native-reanimated';
import { Gesture, GestureDetector } from 'react-native-gesture-handler';

interface AnimatedCardProps {
  index: number;
  children: React.ReactNode;
  onDelete?: () => void;
}

export function AnimatedCard({ index, children, onDelete }: AnimatedCardProps) {
  const translateX = useSharedValue(0);

  const panGesture = Gesture.Pan()
    .onUpdate((event) => {
      translateX.value = event.translationX;
    })
    .onEnd(() => {
      if (translateX.value < -100) {
        onDelete?.();
      } else {
        translateX.value = withSpring(0);
      }
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: translateX.value }],
  }));

  return (
    <GestureDetector gesture={panGesture}>
      <Animated.View
        entering={FadeInDown.delay(index * 100).springify()}
        layout={Layout.springify()}
        style={[styles.card, animatedStyle]}
      >
        {children}
      </Animated.View>
    </GestureDetector>
  );
}

const styles = StyleSheet.create({
  card: {
    backgroundColor: '#fff',
    borderRadius: 12,
    padding: 16,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 3,
  },
});
```

### Error Boundaries

```tsx
// src/components/ErrorBoundary.tsx
import { Component, ReactNode } from 'react';
import { View, Text, StyleSheet } from 'react-native';
import * as Sentry from '@sentry/react-native';
import { Button } from './ui/Button';

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
}

interface State {
  hasError: boolean;
}

export class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false };

  static getDerivedStateFromError(): State {
    return { hasError: true };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    Sentry.captureException(error, { extra: errorInfo });
  }

  handleRetry = () => {
    this.setState({ hasError: false });
  };

  render() {
    if (this.state.hasError) {
      return this.props.fallback || (
        <View style={styles.container}>
          <Text style={styles.title}>Something went wrong</Text>
          <Button title="Try Again" onPress={this.handleRetry} />
        </View>
      );
    }

    return this.props.children;
  }
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 20,
  },
  title: {
    fontSize: 18,
    fontWeight: '600',
    marginBottom: 16,
  },
});
```

### Testing

```tsx
// src/features/posts/components/__tests__/PostCard.test.tsx
import { render, screen, fireEvent } from '@testing-library/react-native';
import { PostCard } from '../PostCard';

const mockPost = {
  id: '1',
  title: 'Test Post',
  content: 'Test content',
  author: 'John Doe',
};

describe('PostCard', () => {
  it('renders post information', () => {
    render(<PostCard post={mockPost} />);

    expect(screen.getByText('Test Post')).toBeTruthy();
    expect(screen.getByText('Test content')).toBeTruthy();
    expect(screen.getByText('John Doe')).toBeTruthy();
  });

  it('calls onPress when tapped', () => {
    const onPress = jest.fn();
    render(<PostCard post={mockPost} onPress={onPress} />);

    fireEvent.press(screen.getByText('Test Post'));
    expect(onPress).toHaveBeenCalledWith(mockPost);
  });
});
```

### EAS Configuration

```json
// eas.json
{
  "cli": {
    "version": ">= 12.0.0"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    },
    "preview": {
      "distribution": "internal",
      "channel": "preview"
    },
    "production": {
      "channel": "production",
      "autoIncrement": true
    }
  },
  "submit": {
    "production": {}
  }
}
```

```json
// app.json
{
  "expo": {
    "name": "MyApp",
    "slug": "my-app",
    "version": "1.0.0",
    "orientation": "portrait",
    "icon": "./assets/icon.png",
    "newArchEnabled": true,
    "splash": {
      "image": "./assets/splash.png",
      "resizeMode": "contain",
      "backgroundColor": "#ffffff"
    },
    "ios": {
      "supportsTablet": true,
      "bundleIdentifier": "com.example.myapp"
    },
    "android": {
      "adaptiveIcon": {
        "foregroundImage": "./assets/adaptive-icon.png",
        "backgroundColor": "#ffffff"
      },
      "package": "com.example.myapp"
    },
    "plugins": [
      "expo-router",
      "expo-secure-store",
      [
        "expo-build-properties",
        {
          "android": {
            "newArchEnabled": true
          },
          "ios": {
            "newArchEnabled": true
          }
        }
      ]
    ],
    "experiments": {
      "typedRoutes": true
    }
  }
}
```

---

## Anti-Patterns

### ❌ Class Components

```tsx
// ❌ DON'T — Outdated pattern
class MyComponent extends React.Component {
  render() {
    return <Text>{this.props.title}</Text>;
  }
}

// ✅ DO — Functional components
function MyComponent({ title }: { title: string }) {
  return <Text>{title}</Text>;
}
```

### ❌ Anonymous Functions in Lists

```tsx
// ❌ DON'T — Causes re-renders
<FlatList
  data={items}
  renderItem={({ item }) => <ItemCard item={item} />}
  keyExtractor={(item) => item.id}
/>

// ✅ DO — Stable references
const renderItem = useCallback(
  ({ item }) => <ItemCard item={item} />,
  []
);
const keyExtractor = useCallback((item) => item.id, []);

<FlatList
  data={items}
  renderItem={renderItem}
  keyExtractor={keyExtractor}
/>
```

### ❌ Inline Styles

```tsx
// ❌ DON'T — Creates new object every render
<View style={{ padding: 20, backgroundColor: '#fff' }} />

// ✅ DO — StyleSheet (memoized)
const styles = StyleSheet.create({
  container: {
    padding: 20,
    backgroundColor: '#fff',
  },
});

<View style={styles.container} />
```

### ❌ Blocking the JS Thread

```tsx
// ❌ DON'T — Heavy sync operation
const processedData = expensiveCalculation(rawData);

// ✅ DO — Use InteractionManager
import { InteractionManager } from 'react-native';

useEffect(() => {
  InteractionManager.runAfterInteractions(() => {
    const result = expensiveCalculation(rawData);
    setProcessedData(result);
  });
}, [rawData]);
```

### ❌ Ignoring Platform Differences

```tsx
// ❌ DON'T — Assume same behavior
<View style={{ elevation: 5 }} />

// ✅ DO — Platform-specific styles
import { Platform, StyleSheet } from 'react-native';

const styles = StyleSheet.create({
  card: {
    ...Platform.select({
      ios: {
        shadowColor: '#000',
        shadowOffset: { width: 0, height: 2 },
        shadowOpacity: 0.1,
        shadowRadius: 4,
      },
      android: {
        elevation: 4,
      },
    }),
  },
});
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| RN 0.76 | Nov 2024 | New Architecture default |
| Expo SDK 52 | Jan 2025 | New Architecture 75% adoption |
| Expo Router 4 | 2025 | API routes, typed routes |
| Fabric | 2024-25 | New rendering engine |

### Key Updates

- **New Architecture default** — Legacy may be removed late 2025
- **Expo Router** — File-based routing standard
- **React 19 features** — Use hook, Server Components experimentation
- **Hermes default** — JavaScript engine for both platforms
- **EAS Update** — OTA updates without app store review

---

## Quick Reference

| Command | Purpose |
|---------|---------|
| `npx create-expo-app@latest` | Create new project |
| `npx expo start` | Start dev server |
| `npx expo start --clear` | Clear cache and start |
| `eas build --profile development` | Development build |
| `eas build --profile production` | Production build |
| `eas update --branch preview` | OTA update |

| Library | Purpose |
|---------|---------|
| `expo-router` | File-based navigation |
| `@tanstack/react-query` | Server state |
| `zustand` | Client state |
| `react-native-reanimated` | Animations |
| `react-native-gesture-handler` | Gestures |
| `expo-secure-store` | Secure storage |
| `@sentry/react-native` | Error tracking |

---

## Resources

- [Expo Documentation](https://docs.expo.dev/)
- [New Architecture Guide](https://docs.expo.dev/guides/new-architecture/)
- [Expo Router Documentation](https://docs.expo.dev/router/introduction/)
- [React Native Directory](https://reactnative.directory/)
- [React Native Performance](https://reactnative.dev/docs/performance)
