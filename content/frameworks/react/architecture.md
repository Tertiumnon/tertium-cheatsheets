# React Architecture with Hooks

Modern React architecture built on functional components and hooks enables scalable, maintainable applications. This guide covers organizing large-scale React apps with clean architecture, state management, and best practices.

## Key Concepts

- **Functional Components:** Functions that return JSX (modern standard)
- **Hooks:** Functions for using state and side effects in functional components
- **Custom Hooks:** Reusable logic extraction from components
- **Context API:** Built-in solution for prop drilling and global state
- **Render Props & HOC:** Component composition patterns
- **Container vs. Presentational:** Component role separation
- **State Management:** Centralized vs. distributed state approaches

## Project Structure (Feature-Based)

Organize by features and concerns:

```
src/
├── components/                       # Reusable UI components
│   ├── common/                       # Shared components
│   │   ├── Button/
│   │   │   ├── Button.tsx
│   │   │   ├── Button.styles.ts      # Styled-components or CSS modules
│   │   │   └── Button.test.tsx
│   │   ├── Modal/
│   │   ├── Card/
│   │   ├── Spinner/
│   │   └── index.ts                  # Barrel export
│   └── layout/
│       ├── Header.tsx
│       ├── Footer.tsx
│       ├── Sidebar.tsx
│       └── Layout.tsx
├── features/                         # Feature modules
│   ├── order/                        # Order Feature
│   │   ├── components/               # Feature-specific components
│   │   │   ├── OrderList.tsx         # Smart component
│   │   │   ├── OrderCard.tsx         # Presentational component
│   │   │   ├── OrderForm.tsx
│   │   │   └── OrderSummary.tsx
│   │   ├── hooks/                    # Custom hooks
│   │   │   ├── useOrderList.ts
│   │   │   ├── useOrder.ts
│   │   │   └── useOrderForm.ts
│   │   ├── context/                  # Feature context
│   │   │   ├── OrderContext.ts
│   │   │   └── OrderProvider.tsx
│   │   ├── services/                 # API & business logic
│   │   │   ├── orderService.ts
│   │   │   └── orderMapper.ts
│   │   ├── types/                    # TypeScript types
│   │   │   └── order.types.ts
│   │   ├── utils/                    # Feature utilities
│   │   │   └── orderHelpers.ts
│   │   └── pages/                    # Page components (routes)
│   │       ├── OrderListPage.tsx
│   │       ├── OrderDetailPage.tsx
│   │       └── OrderFormPage.tsx
│   ├── customer/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── context/
│   │   ├── services/
│   │   ├── types/
│   │   └── pages/
│   └── payment/
│       └── ...
├── hooks/                            # Global custom hooks
│   ├── useAuth.ts
│   ├── useApi.ts
│   ├── useFetch.ts
│   ├── useLocalStorage.ts
│   └── useDebounce.ts
├── context/                          # Global contexts
│   ├── AuthContext.tsx
│   ├── ThemeContext.tsx
│   └── NotificationContext.tsx
├── services/                         # Core services
│   ├── api.ts                        # API client
│   ├── auth.service.ts
│   ├── storage.service.ts
│   └── logger.service.ts
├── types/                            # Global types
│   ├── index.ts
│   └── api.types.ts
├── utils/                            # Global utilities
│   ├── formatters.ts
│   ├── validators.ts
│   └── constants.ts
├── pages/                            # Route pages
│   ├── Home.tsx
│   ├── NotFound.tsx
│   └── Dashboard.tsx
├── App.tsx                           # Root component
├── App.styles.ts
├── main.tsx                          # Entry point
└── index.css
```

## Functional Components & Hooks

### Basic Component with Hooks

```typescript
// features/order/components/OrderList.tsx
import { FC, useState, useEffect } from 'react';
import { orderService } from '../services/orderService';
import { Order } from '../types/order.types';
import { OrderCard } from './OrderCard';
import * as S from './OrderList.styles';

interface OrderListProps {
  status?: 'pending' | 'completed' | 'all';
}

export const OrderList: FC<OrderListProps> = ({ status = 'all' }) => {
  const [orders, setOrders] = useState<Order[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const loadOrders = async () => {
      setLoading(true);
      setError(null);
      try {
        const data = await orderService.getOrders(status);
        setOrders(data);
      } catch (err) {
        setError(err instanceof Error ? err.message : 'Failed to load orders');
      } finally {
        setLoading(false);
      }
    };

    loadOrders();
  }, [status]);

  if (loading) return <S.LoadingSpinner />;
  if (error) return <S.ErrorMessage>{error}</S.ErrorMessage>;

  return (
    <S.Container>
      <S.Title>Orders ({orders.length})</S.Title>
      <S.GridContainer>
        {orders.map(order => (
          <OrderCard key={order.id} order={order} />
        ))}
      </S.GridContainer>
    </S.Container>
  );
};
```

### Presentational Component

Reusable UI component with no business logic:

```typescript
// features/order/components/OrderCard.tsx
import { FC } from 'react';
import { Order } from '../types/order.types';
import * as S from './OrderCard.styles';

interface OrderCardProps {
  order: Order;
  onEdit?: (order: Order) => void;
  onDelete?: (id: string) => void;
}

export const OrderCard: FC<OrderCardProps> = ({ order, onEdit, onDelete }) => {
  return (
    <S.Card>
      <S.Header>
        <S.OrderId>#{order.id}</S.OrderId>
        <S.Status status={order.status}>{order.status}</S.Status>
      </S.Header>
      <S.Body>
        <S.Amount>${order.total.toFixed(2)}</S.Amount>
        <S.Date>{new Date(order.createdAt).toLocaleDateString()}</S.Date>
      </S.Body>
      <S.Footer>
        {onEdit && <S.Button onClick={() => onEdit(order)}>Edit</S.Button>}
        {onDelete && <S.Button onClick={() => onDelete(order.id)}>Delete</S.Button>}
      </S.Footer>
    </S.Card>
  );
};
```

## Custom Hooks

Extract and reuse component logic:

### Data Fetching Hook

```typescript
// hooks/useFetch.ts
import { useState, useEffect } from 'react';

interface UseFetchOptions<T> {
  onSuccess?: (data: T) => void;
  onError?: (error: Error) => void;
  dependencies?: any[];
}

export function useFetch<T>(
  url: string,
  options?: UseFetchOptions<T>
): { data: T | null; loading: boolean; error: Error | null; refetch: () => void } {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  const fetchData = async () => {
    setLoading(true);
    setError(null);
    try {
      const response = await fetch(url);
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      const result = await response.json();
      setData(result);
      options?.onSuccess?.(result);
    } catch (err) {
      const error = err instanceof Error ? err : new Error('Unknown error');
      setError(error);
      options?.onError?.(error);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchData();
  }, options?.dependencies ?? [url]);

  return { data, loading, error, refetch: fetchData };
}

// Usage in component
const { data: orders, loading, error } = useFetch<Order[]>(
  '/api/orders',
  {
    onSuccess: (data) => console.log('Orders loaded', data),
    dependencies: [status]
  }
);
```

### Form Hook

```typescript
// features/order/hooks/useOrderForm.ts
import { useState, useCallback } from 'react';
import { Order } from '../types/order.types';
import { orderService } from '../services/orderService';

interface FormErrors {
  [key: string]: string;
}

export function useOrderForm(initialOrder?: Order) {
  const [formData, setFormData] = useState<Partial<Order>>(initialOrder || {});
  const [errors, setErrors] = useState<FormErrors>({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleChange = useCallback((e: React.ChangeEvent<HTMLInputElement | HTMLTextAreaElement>) => {
    const { name, value } = e.target;
    setFormData(prev => ({ ...prev, [name]: value }));
    // Clear error for this field
    setErrors(prev => ({ ...prev, [name]: '' }));
  }, []);

  const validate = (): boolean => {
    const newErrors: FormErrors = {};
    if (!formData.id) newErrors.id = 'Order ID is required';
    if (!formData.total) newErrors.total = 'Total is required';
    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };

  const submit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!validate()) return;

    setIsSubmitting(true);
    try {
      const result = initialOrder
        ? await orderService.update(formData as Order)
        : await orderService.create(formData as Order);
      return result;
    } finally {
      setIsSubmitting(false);
    }
  };

  return {
    formData,
    errors,
    isSubmitting,
    handleChange,
    submit,
    setFormData
  };
}

// Usage
const OrderForm = () => {
  const { formData, errors, submit, handleChange } = useOrderForm();

  return (
    <form onSubmit={submit}>
      <input
        name="id"
        value={formData.id || ''}
        onChange={handleChange}
      />
      {errors.id && <span className="error">{errors.id}</span>}
      <button type="submit">Submit</button>
    </form>
  );
};
```

### Debounce Hook

```typescript
// hooks/useDebounce.ts
import { useState, useEffect } from 'react';

export function useDebounce<T>(value: T, delay: number = 500): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(handler);
  }, [value, delay]);

  return debouncedValue;
}

// Usage: Search component
const SearchOrders = () => {
  const [searchTerm, setSearchTerm] = useState('');
  const debouncedSearchTerm = useDebounce(searchTerm, 300);
  const { data: results } = useFetch(`/api/orders?search=${debouncedSearchTerm}`, {
    dependencies: [debouncedSearchTerm]
  });

  return (
    <>
      <input
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
        placeholder="Search orders..."
      />
      {/* Display results */}
    </>
  );
};
```

## Context API & Providers

Global state management without Redux:

### Authentication Context

```typescript
// context/AuthContext.tsx
import { createContext, useContext, useState, useCallback, ReactNode } from 'react';
import { authService } from '../services/auth.service';

interface User {
  id: string;
  email: string;
  name: string;
  role: 'admin' | 'user';
}

interface AuthContextType {
  user: User | null;
  loading: boolean;
  isAuthenticated: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  signup: (data: SignupData) => Promise<void>;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export const AuthProvider = ({ children }: { children: ReactNode }) => {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  const login = useCallback(async (email: string, password: string) => {
    setLoading(true);
    try {
      const userData = await authService.login(email, password);
      setUser(userData);
      localStorage.setItem('token', userData.token);
    } finally {
      setLoading(false);
    }
  }, []);

  const logout = useCallback(() => {
    setUser(null);
    localStorage.removeItem('token');
    authService.logout();
  }, []);

  const signup = useCallback(async (data: SignupData) => {
    setLoading(true);
    try {
      const userData = await authService.signup(data);
      setUser(userData);
      localStorage.setItem('token', userData.token);
    } finally {
      setLoading(false);
    }
  }, []);

  return (
    <AuthContext.Provider value={{ user, loading, isAuthenticated: !!user, login, logout, signup }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
};
```

### Feature Context

```typescript
// features/order/context/OrderContext.tsx
import { createContext, useContext, useState, useCallback, ReactNode } from 'react';
import { Order } from '../types/order.types';
import { orderService } from '../services/orderService';

interface OrderContextType {
  orders: Order[];
  selectedOrder: Order | null;
  loading: boolean;
  error: string | null;
  loadOrders: () => Promise<void>;
  selectOrder: (order: Order) => void;
  createOrder: (order: Omit<Order, 'id'>) => Promise<void>;
  updateOrder: (order: Order) => Promise<void>;
  deleteOrder: (id: string) => Promise<void>;
  clearError: () => void;
}

const OrderContext = createContext<OrderContextType | undefined>(undefined);

export const OrderProvider = ({ children }: { children: ReactNode }) => {
  const [orders, setOrders] = useState<Order[]>([]);
  const [selectedOrder, setSelectedOrder] = useState<Order | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const loadOrders = useCallback(async () => {
    setLoading(true);
    setError(null);
    try {
      const data = await orderService.getOrders();
      setOrders(data);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to load orders');
    } finally {
      setLoading(false);
    }
  }, []);

  const createOrder = useCallback(async (newOrder: Omit<Order, 'id'>) => {
    try {
      const created = await orderService.create(newOrder);
      setOrders(prev => [...prev, created]);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to create order');
      throw err;
    }
  }, []);

  const updateOrder = useCallback(async (updated: Order) => {
    try {
      await orderService.update(updated);
      setOrders(prev => prev.map(o => o.id === updated.id ? updated : o));
      if (selectedOrder?.id === updated.id) {
        setSelectedOrder(updated);
      }
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to update order');
      throw err;
    }
  }, [selectedOrder]);

  const deleteOrder = useCallback(async (id: string) => {
    try {
      await orderService.delete(id);
      setOrders(prev => prev.filter(o => o.id !== id));
      if (selectedOrder?.id === id) {
        setSelectedOrder(null);
      }
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to delete order');
      throw err;
    }
  }, [selectedOrder]);

  const value: OrderContextType = {
    orders,
    selectedOrder,
    loading,
    error,
    loadOrders,
    selectOrder: setSelectedOrder,
    createOrder,
    updateOrder,
    deleteOrder,
    clearError: () => setError(null)
  };

  return (
    <OrderContext.Provider value={value}>
      {children}
    </OrderContext.Provider>
  );
};

export const useOrder = () => {
  const context = useContext(OrderContext);
  if (!context) {
    throw new Error('useOrder must be used within OrderProvider');
  }
  return context;
};
```

## State Management Patterns

### Using Context + useReducer for Complex State

```typescript
// features/order/hooks/useOrderState.ts
import { useReducer, useCallback } from 'react';
import { Order } from '../types/order.types';

type OrderAction =
  | { type: 'LOAD_START' }
  | { type: 'LOAD_SUCCESS'; payload: Order[] }
  | { type: 'LOAD_ERROR'; payload: string }
  | { type: 'ADD_ORDER'; payload: Order }
  | { type: 'UPDATE_ORDER'; payload: Order }
  | { type: 'DELETE_ORDER'; payload: string }
  | { type: 'CLEAR_ERROR' };

interface OrderState {
  orders: Order[];
  loading: boolean;
  error: string | null;
}

const initialState: OrderState = {
  orders: [],
  loading: false,
  error: null
};

function orderReducer(state: OrderState, action: OrderAction): OrderState {
  switch (action.type) {
    case 'LOAD_START':
      return { ...state, loading: true, error: null };
    case 'LOAD_SUCCESS':
      return { ...state, orders: action.payload, loading: false };
    case 'LOAD_ERROR':
      return { ...state, error: action.payload, loading: false };
    case 'ADD_ORDER':
      return { ...state, orders: [...state.orders, action.payload] };
    case 'UPDATE_ORDER':
      return {
        ...state,
        orders: state.orders.map(o => o.id === action.payload.id ? action.payload : o)
      };
    case 'DELETE_ORDER':
      return { ...state, orders: state.orders.filter(o => o.id !== action.payload) };
    case 'CLEAR_ERROR':
      return { ...state, error: null };
    default:
      return state;
  }
}

export function useOrderState() {
  const [state, dispatch] = useReducer(orderReducer, initialState);

  return {
    ...state,
    loadStart: useCallback(() => dispatch({ type: 'LOAD_START' }), []),
    loadSuccess: useCallback((orders: Order[]) => 
      dispatch({ type: 'LOAD_SUCCESS', payload: orders }), []),
    loadError: useCallback((error: string) => 
      dispatch({ type: 'LOAD_ERROR', payload: error }), []),
    addOrder: useCallback((order: Order) => 
      dispatch({ type: 'ADD_ORDER', payload: order }), []),
    updateOrder: useCallback((order: Order) => 
      dispatch({ type: 'UPDATE_ORDER', payload: order }), []),
    deleteOrder: useCallback((id: string) => 
      dispatch({ type: 'DELETE_ORDER', payload: id }), []),
    clearError: useCallback(() => dispatch({ type: 'CLEAR_ERROR' }), [])
  };
}
```

## Routing Structure

```typescript
// App.tsx
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { AuthProvider } from './context/AuthContext';
import { OrderProvider } from './features/order/context/OrderContext';
import { Layout } from './components/layout/Layout';
import { ProtectedRoute } from './components/ProtectedRoute';
import { LoginPage } from './features/auth/pages/LoginPage';
import { OrderListPage } from './features/order/pages/OrderListPage';
import { OrderDetailPage } from './features/order/pages/OrderDetailPage';

export const App = () => {
  return (
    <BrowserRouter>
      <AuthProvider>
        <Routes>
          <Route path="/login" element={<LoginPage />} />
          <Route element={<Layout />}>
            <Route
              path="/"
              element={
                <ProtectedRoute>
                  <Navigate to="/orders" />
                </ProtectedRoute>
              }
            />
            <Route
              path="/orders"
              element={
                <ProtectedRoute>
                  <OrderProvider>
                    <OrderListPage />
                  </OrderProvider>
                </ProtectedRoute>
              }
            />
            <Route
              path="/orders/:id"
              element={
                <ProtectedRoute>
                  <OrderProvider>
                    <OrderDetailPage />
                  </OrderProvider>
                </ProtectedRoute>
              }
            />
          </Route>
          <Route path="*" element={<NotFoundPage />} />
        </Routes>
      </AuthProvider>
    </BrowserRouter>
  );
};
```

### Protected Route Component

```typescript
// components/ProtectedRoute.tsx
import { ReactNode } from 'react';
import { Navigate } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

interface ProtectedRouteProps {
  children: ReactNode;
  requiredRole?: 'admin' | 'user';
}

export const ProtectedRoute = ({ children, requiredRole }: ProtectedRouteProps) => {
  const { isAuthenticated, user, loading } = useAuth();

  if (loading) return <LoadingPage />;

  if (!isAuthenticated) {
    return <Navigate to="/login" replace />;
  }

  if (requiredRole && user?.role !== requiredRole) {
    return <Navigate to="/" replace />;
  }

  return <>{children}</>;
};
```

## Performance Optimization

### React.memo for Presentational Components

```typescript
// features/order/components/OrderCard.tsx
import { memo, FC } from 'react';
import { Order } from '../types/order.types';

interface OrderCardProps {
  order: Order;
  onEdit?: (order: Order) => void;
}

export const OrderCard = memo<OrderCardProps>(({ order, onEdit }) => {
  return (
    <div>
      {/* Component JSX */}
    </div>
  );
}, (prev, next) => {
  // Custom comparison for memoization
  return prev.order.id === next.order.id;
});

OrderCard.displayName = 'OrderCard';
```

### useCallback for Stable Function References

```typescript
// features/order/components/OrderList.tsx
import { useCallback, useMemo } from 'react';

export const OrderList = ({ orders }: OrderListProps) => {
  const handleEdit = useCallback((order: Order) => {
    // Edit logic
  }, []);

  const sortedOrders = useMemo(() => {
    return [...orders].sort((a, b) => b.createdAt - a.createdAt);
  }, [orders]);

  return (
    <div>
      {sortedOrders.map(order => (
        <OrderCard
          key={order.id}
          order={order}
          onEdit={handleEdit}
        />
      ))}
    </div>
  );
};
```

## Best Practices

- **One responsibility per component:** Small, focused components
- **Lift state up:** Share state only when necessary
- **Custom hooks for logic:** Extract reusable component logic
- **Use TypeScript:** Strong typing prevents bugs
- **Memo for performance:** Wrap expensive components
- **Lazy load routes:** Code splitting for better performance
- **Avoid prop drilling:** Use Context API for deeply nested props
- **Error boundaries:** Handle errors gracefully
- **Accessible components:** ARIA labels and semantic HTML
- **Environment variables:** Use .env files for sensitive data

## File Naming Conventions

```
Components:         ComponentName.tsx   (OrderCard.tsx)
Custom Hooks:       useHookName.ts      (useOrderList.ts)
Contexts:           NameContext.tsx     (OrderContext.tsx)
Services:           serviceName.ts      (orderService.ts)
Types/Interfaces:   name.types.ts       (order.types.ts)
Tests:              Component.test.tsx  (OrderCard.test.tsx)
Styles:             Component.styles.ts (OrderCard.styles.ts)
Utils:              utilName.ts         (orderHelpers.ts)
Hooks:              useName.ts          (useAuth.ts)
```

## Related Concepts

- **React Query:** Server state management library
- **Redux/Redux Toolkit:** Centralized state management
- **Zustand:** Lightweight state management
- **Recoil:** Atomic state management
- **SWR:** Data fetching with React
- **TypeScript React:** Type-safe component development

## References & Sources

### Official React Documentation
- React Official Guide: https://react.dev/
- React Hooks API Reference: https://react.dev/reference/react/hooks
- React Context API: https://react.dev/reference/react/useContext
- React Performance Optimization: https://react.dev/reference/react/memo
- React Router: https://reactrouter.com/
- React TypeScript Guide: https://react-typescript-cheatsheet.netlify.app/

### Articles & Best Practices
- React Team Blog: https://react.dev/blog
- "Presentational and Container Components" - Dan Abramov: https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0
- "Compound Component Pattern" - React Patterns
- "Rules of Hooks" - React Documentation: https://react.dev/reference/rules/rules-of-hooks

### State Management Libraries
- **React Query (TanStack Query):** https://tanstack.com/query/latest
- **Redux Toolkit:** https://redux-toolkit.js.org/
- **Zustand:** https://github.com/pmndrs/zustand
- **Recoil:** https://recoiljs.org/
- **MobX:** https://mobx.js.org/

### Related Technologies
- **TypeScript:** https://www.typescriptlang.org/
- **Styled Components:** https://styled-components.com/
- **CSS Modules:** https://github.com/css-modules/css-modules
- **Testing Library:** https://testing-library.com/

### Note
This guide reflects **modern React best practices** using functional components and hooks (React 16.8+). The structure and patterns shown are widely adopted in production applications but can be adapted based on project requirements.
