# React Components

> Official Documentation: https://react.dev/reference/react/components

Components are the fundamental building blocks of React applications. They let you split the UI into independent, reusable pieces and think about each piece in isolation.

---

## 1. Function Components

Function components are the modern standard for writing React components. They are simpler, more concise, and support hooks.

```tsx
// Basic function component
function Welcome({ name }: { name: string }) {
  return <h1>Hello, {name}</h1>;
}

// Arrow function component
const Greeting: React.FC<{ message: string }> = ({ message }) => {
  return <p>{message}</p>;
};

// Component with multiple elements (implicit fragment)
function UserCard({ name, email }: { name: string; email: string }) {
  return (
    <>
      <h2>{name}</h2>
      <p>{email}</p>
    </>
  );
}

// Component with internal state and effects
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    document.title = `Count: ${count}`;
  }, [count]);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

---

## 2. Props and TypeScript Typing

Props are the mechanism for passing data from parent to child components. TypeScript provides strong typing for props.

```tsx
// Interface-based props (recommended)
interface ButtonProps {
  label: string;
  onClick: () => void;
  disabled?: boolean;        // Optional prop
  variant?: 'primary' | 'secondary' | 'danger';  // Union type
  size?: 'sm' | 'md' | 'lg';
}

function Button({
  label,
  onClick,
  disabled = false,          // Default value
  variant = 'primary',
  size = 'md'
}: ButtonProps) {
  return (
    <button
      className={`btn btn-${variant} btn-${size}`}
      onClick={onClick}
      disabled={disabled}
    >
      {label}
    </button>
  );
}

// Type alias approach
type CardProps = {
  title: string;
  subtitle?: string;
  footer?: React.ReactNode;
};

// Extending HTML element props
interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
  error?: string;
}

function Input({ label, error, ...inputProps }: InputProps) {
  return (
    <div className="input-wrapper">
      <label>{label}</label>
      <input {...inputProps} />
      {error && <span className="error">{error}</span>}
    </div>
  );
}

// Generic component props
interface ListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
  keyExtractor: (item: T) => string | number;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>
      {items.map((item, index) => (
        <li key={keyExtractor(item)}>{renderItem(item, index)}</li>
      ))}
    </ul>
  );
}

// Usage with generic
interface User { id: number; name: string; }
<List<User>
  items={users}
  renderItem={(user) => <span>{user.name}</span>}
  keyExtractor={(user) => user.id}
/>
```

---

## 3. Children Prop Patterns

The `children` prop is a special prop for passing nested elements to components.

```tsx
// React.ReactNode - accepts anything renderable
interface ContainerProps {
  children: React.ReactNode;
}

function Container({ children }: ContainerProps) {
  return <div className="container">{children}</div>;
}

// React.ReactElement - only React elements (no strings/numbers)
interface WrapperProps {
  children: React.ReactElement;
}

// Array of specific elements
interface TabListProps {
  children: React.ReactElement<TabProps>[];
}

// Function as children (render prop pattern)
interface MouseTrackerProps {
  children: (position: { x: number; y: number }) => React.ReactNode;
}

function MouseTracker({ children }: MouseTrackerProps) {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  const handleMouseMove = (e: React.MouseEvent) => {
    setPosition({ x: e.clientX, y: e.clientY });
  };

  return (
    <div onMouseMove={handleMouseMove} style={{ height: '100vh' }}>
      {children(position)}
    </div>
  );
}

// Usage
<MouseTracker>
  {({ x, y }) => <p>Mouse at: {x}, {y}</p>}
</MouseTracker>

// Manipulating children with React.Children utilities
interface LayoutProps {
  children: React.ReactNode;
}

function Layout({ children }: LayoutProps) {
  return (
    <div className="layout">
      {React.Children.map(children, (child, index) => (
        <div key={index} className="layout-item">
          {child}
        </div>
      ))}
    </div>
  );
}

// Counting children
const childCount = React.Children.count(children);

// Converting to array
const childArray = React.Children.toArray(children);

// Ensuring single child
const onlyChild = React.Children.only(children);
```

---

## 4. Composition Patterns

Composition is the primary pattern for building complex UIs from simple components.

```tsx
// Specialization - creating specific versions of generic components
function Dialog({ title, children }: { title: string; children: React.ReactNode }) {
  return (
    <div className="dialog">
      <h2>{title}</h2>
      <div className="dialog-content">{children}</div>
    </div>
  );
}

function ConfirmDialog({ onConfirm, onCancel }: { onConfirm: () => void; onCancel: () => void }) {
  return (
    <Dialog title="Confirm Action">
      <p>Are you sure you want to proceed?</p>
      <button onClick={onConfirm}>Confirm</button>
      <button onClick={onCancel}>Cancel</button>
    </Dialog>
  );
}

// Compound Components Pattern
const TabsContext = React.createContext<{
  activeTab: number;
  setActiveTab: (index: number) => void;
} | null>(null);

function Tabs({ children, defaultTab = 0 }: { children: React.ReactNode; defaultTab?: number }) {
  const [activeTab, setActiveTab] = useState(defaultTab);

  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

function TabList({ children }: { children: React.ReactNode }) {
  return <div className="tab-list" role="tablist">{children}</div>;
}

function Tab({ children, index }: { children: React.ReactNode; index: number }) {
  const context = useContext(TabsContext);
  if (!context) throw new Error('Tab must be used within Tabs');

  const { activeTab, setActiveTab } = context;
  return (
    <button
      role="tab"
      aria-selected={activeTab === index}
      onClick={() => setActiveTab(index)}
      className={activeTab === index ? 'active' : ''}
    >
      {children}
    </button>
  );
}

function TabPanels({ children }: { children: React.ReactNode }) {
  const context = useContext(TabsContext);
  if (!context) throw new Error('TabPanels must be used within Tabs');

  const childArray = React.Children.toArray(children);
  return <div className="tab-panels">{childArray[context.activeTab]}</div>;
}

function TabPanel({ children }: { children: React.ReactNode }) {
  return <div role="tabpanel">{children}</div>;
}

// Attach sub-components
Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panels = TabPanels;
Tabs.Panel = TabPanel;

// Usage
<Tabs defaultTab={0}>
  <Tabs.List>
    <Tabs.Tab index={0}>Profile</Tabs.Tab>
    <Tabs.Tab index={1}>Settings</Tabs.Tab>
    <Tabs.Tab index={2}>Notifications</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panels>
    <Tabs.Panel>Profile content</Tabs.Panel>
    <Tabs.Panel>Settings content</Tabs.Panel>
    <Tabs.Panel>Notifications content</Tabs.Panel>
  </Tabs.Panels>
</Tabs>

// Slot Pattern
interface PageLayoutProps {
  header: React.ReactNode;
  sidebar: React.ReactNode;
  children: React.ReactNode;
  footer?: React.ReactNode;
}

function PageLayout({ header, sidebar, children, footer }: PageLayoutProps) {
  return (
    <div className="page-layout">
      <header>{header}</header>
      <aside>{sidebar}</aside>
      <main>{children}</main>
      {footer && <footer>{footer}</footer>}
    </div>
  );
}
```

---

## 5. Render Props

Render props is a pattern for sharing code between components using a prop whose value is a function.

```tsx
// Basic render prop
interface DataFetcherProps<T> {
  url: string;
  render: (data: T | null, loading: boolean, error: Error | null) => React.ReactNode;
}

function DataFetcher<T>({ url, render }: DataFetcherProps<T>) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    setLoading(true);
    fetch(url)
      .then((res) => res.json())
      .then((data) => {
        setData(data);
        setLoading(false);
      })
      .catch((err) => {
        setError(err);
        setLoading(false);
      });
  }, [url]);

  return <>{render(data, loading, error)}</>;
}

// Usage
<DataFetcher<User[]>
  url="/api/users"
  render={(data, loading, error) => {
    if (loading) return <Spinner />;
    if (error) return <Error message={error.message} />;
    return <UserList users={data!} />;
  }}
/>

// Toggle render prop
interface ToggleProps {
  children: (props: { on: boolean; toggle: () => void }) => React.ReactNode;
}

function Toggle({ children }: ToggleProps) {
  const [on, setOn] = useState(false);
  const toggle = () => setOn((prev) => !prev);

  return <>{children({ on, toggle })}</>;
}

// Usage
<Toggle>
  {({ on, toggle }) => (
    <div>
      <button onClick={toggle}>{on ? 'Hide' : 'Show'}</button>
      {on && <div>Toggled content!</div>}
    </div>
  )}
</Toggle>

// Downshift-style render props with state reducer
interface SelectRenderProps<T> {
  isOpen: boolean;
  selectedItem: T | null;
  highlightedIndex: number;
  getToggleProps: () => React.ButtonHTMLAttributes<HTMLButtonElement>;
  getItemProps: (options: { item: T; index: number }) => React.LiHTMLAttributes<HTMLLIElement>;
}
```

---

## 6. Higher-Order Components (HOC)

HOCs are functions that take a component and return a new component with enhanced functionality.

```tsx
// Basic HOC pattern
interface WithLoadingProps {
  isLoading: boolean;
}

function withLoading<P extends object>(
  WrappedComponent: React.ComponentType<P>
): React.FC<P & WithLoadingProps> {
  return function WithLoadingComponent({ isLoading, ...props }: P & WithLoadingProps) {
    if (isLoading) {
      return <div className="spinner">Loading...</div>;
    }
    return <WrappedComponent {...(props as P)} />;
  };
}

// Usage
const UserListWithLoading = withLoading(UserList);
<UserListWithLoading isLoading={loading} users={users} />

// HOC with configuration
function withAuth<P extends object>(
  WrappedComponent: React.ComponentType<P>,
  redirectTo: string = '/login'
): React.FC<P> {
  return function AuthenticatedComponent(props: P) {
    const { user, isAuthenticated } = useAuth();

    useEffect(() => {
      if (!isAuthenticated) {
        window.location.href = redirectTo;
      }
    }, [isAuthenticated]);

    if (!isAuthenticated) {
      return null;
    }

    return <WrappedComponent {...props} user={user} />;
  };
}

// HOC that injects props
interface WithThemeProps {
  theme: 'light' | 'dark';
}

function withTheme<P extends WithThemeProps>(
  WrappedComponent: React.ComponentType<P>
): React.FC<Omit<P, keyof WithThemeProps>> {
  return function ThemedComponent(props: Omit<P, keyof WithThemeProps>) {
    const theme = useContext(ThemeContext);
    return <WrappedComponent {...(props as P)} theme={theme} />;
  };
}

// Composing multiple HOCs
const EnhancedComponent = withAuth(withLoading(withTheme(MyComponent)));

// Better: use compose utility
import { compose } from 'lodash/fp';
const EnhancedComponent = compose(
  withAuth,
  withLoading,
  withTheme
)(MyComponent);
```

---

## 7. Controlled vs Uncontrolled Components

Understanding the difference between controlled and uncontrolled components is essential for form handling.

```tsx
// CONTROLLED: React state is the single source of truth
function ControlledForm() {
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    message: ''
  });

  const handleChange = (e: React.ChangeEvent<HTMLInputElement | HTMLTextAreaElement>) => {
    const { name, value } = e.target;
    setFormData((prev) => ({ ...prev, [name]: value }));
  };

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    console.log('Submitted:', formData);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        name="name"
        value={formData.name}
        onChange={handleChange}
        placeholder="Name"
      />
      <input
        name="email"
        type="email"
        value={formData.email}
        onChange={handleChange}
        placeholder="Email"
      />
      <textarea
        name="message"
        value={formData.message}
        onChange={handleChange}
        placeholder="Message"
      />
      <button type="submit">Submit</button>
    </form>
  );
}

// UNCONTROLLED: DOM is the source of truth
function UncontrolledForm() {
  const nameRef = useRef<HTMLInputElement>(null);
  const emailRef = useRef<HTMLInputElement>(null);
  const messageRef = useRef<HTMLTextAreaElement>(null);

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    console.log('Submitted:', {
      name: nameRef.current?.value,
      email: emailRef.current?.value,
      message: messageRef.current?.value
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input ref={nameRef} name="name" defaultValue="" placeholder="Name" />
      <input ref={emailRef} name="email" type="email" defaultValue="" placeholder="Email" />
      <textarea ref={messageRef} name="message" defaultValue="" placeholder="Message" />
      <button type="submit">Submit</button>
    </form>
  );
}

// Hybrid: Component that supports both controlled and uncontrolled modes
interface FlexibleInputProps {
  value?: string;
  defaultValue?: string;
  onChange?: (value: string) => void;
}

function FlexibleInput({ value, defaultValue, onChange }: FlexibleInputProps) {
  const [internalValue, setInternalValue] = useState(defaultValue || '');
  const isControlled = value !== undefined;

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const newValue = e.target.value;
    if (!isControlled) {
      setInternalValue(newValue);
    }
    onChange?.(newValue);
  };

  return (
    <input
      value={isControlled ? value : internalValue}
      onChange={handleChange}
    />
  );
}
```

---

## 8. Error Boundaries

Error boundaries catch JavaScript errors in their child component tree and display a fallback UI.

```tsx
// Class-based Error Boundary (required - no hook equivalent)
interface ErrorBoundaryProps {
  children: React.ReactNode;
  fallback?: React.ReactNode;
  onError?: (error: Error, errorInfo: React.ErrorInfo) => void;
}

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends React.Component<ErrorBoundaryProps, ErrorBoundaryState> {
  constructor(props: ErrorBoundaryProps) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('Error caught by boundary:', error, errorInfo);
    this.props.onError?.(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || (
        <div className="error-fallback">
          <h2>Something went wrong</h2>
          <details>
            <summary>Error details</summary>
            <pre>{this.state.error?.message}</pre>
          </details>
          <button onClick={() => this.setState({ hasError: false, error: null })}>
            Try again
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}

// Usage
<ErrorBoundary
  fallback={<div>Error loading widget</div>}
  onError={(error) => logErrorToService(error)}
>
  <MyWidget />
</ErrorBoundary>

// Reusable error boundary with reset capability
interface ErrorBoundaryWithResetProps extends ErrorBoundaryProps {
  resetKeys?: unknown[];
}

class ErrorBoundaryWithReset extends React.Component<
  ErrorBoundaryWithResetProps,
  ErrorBoundaryState
> {
  constructor(props: ErrorBoundaryWithResetProps) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  componentDidUpdate(prevProps: ErrorBoundaryWithResetProps) {
    if (this.state.hasError && prevProps.resetKeys !== this.props.resetKeys) {
      this.setState({ hasError: false, error: null });
    }
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || <div>Error occurred</div>;
    }
    return this.props.children;
  }
}
```

---

## 9. Suspense and Lazy Loading

Suspense lets components wait for something before rendering, commonly used with lazy-loaded components.

```tsx
import React, { Suspense, lazy } from 'react';

// Lazy loading components
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));
const Profile = lazy(() => import('./pages/Profile'));

// Named export lazy loading
const UserList = lazy(() =>
  import('./components/UserList').then((module) => ({
    default: module.UserList
  }))
);

// Suspense wrapper
function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <Routes>
        <Route path="/" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
        <Route path="/profile" element={<Profile />} />
      </Routes>
    </Suspense>
  );
}

// Granular suspense boundaries
function DashboardPage() {
  return (
    <div className="dashboard">
      <Suspense fallback={<HeaderSkeleton />}>
        <Header />
      </Suspense>

      <div className="dashboard-content">
        <Suspense fallback={<SidebarSkeleton />}>
          <Sidebar />
        </Suspense>

        <Suspense fallback={<MainContentSkeleton />}>
          <MainContent />
        </Suspense>
      </div>
    </div>
  );
}

// Preloading components
const PreloadedComponent = lazy(() => import('./HeavyComponent'));

// Trigger preload on hover
function NavigationLink({ to, children }: { to: string; children: React.ReactNode }) {
  const handleMouseEnter = () => {
    // Preload the component when user hovers
    import('./pages' + to);
  };

  return (
    <Link to={to} onMouseEnter={handleMouseEnter}>
      {children}
    </Link>
  );
}

// Suspense with data fetching (React 18+)
// Using use() hook (experimental)
function UserProfile({ userId }: { userId: string }) {
  const user = use(fetchUser(userId)); // Suspends until resolved
  return <div>{user.name}</div>;
}
```

---

## 10. Portals

Portals provide a way to render children into a DOM node outside the parent component hierarchy.

```tsx
import { createPortal } from 'react-dom';

// Basic portal for modals
interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  children: React.ReactNode;
}

function Modal({ isOpen, onClose, children }: ModalProps) {
  if (!isOpen) return null;

  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        <button className="modal-close" onClick={onClose}>
          &times;
        </button>
        {children}
      </div>
    </div>,
    document.getElementById('modal-root')!
  );
}

// Tooltip portal
interface TooltipProps {
  content: string;
  children: React.ReactElement;
}

function Tooltip({ content, children }: TooltipProps) {
  const [show, setShow] = useState(false);
  const [position, setPosition] = useState({ x: 0, y: 0 });
  const triggerRef = useRef<HTMLElement>(null);

  const handleMouseEnter = () => {
    if (triggerRef.current) {
      const rect = triggerRef.current.getBoundingClientRect();
      setPosition({
        x: rect.left + rect.width / 2,
        y: rect.top - 10
      });
    }
    setShow(true);
  };

  return (
    <>
      {React.cloneElement(children, {
        ref: triggerRef,
        onMouseEnter: handleMouseEnter,
        onMouseLeave: () => setShow(false)
      })}
      {show &&
        createPortal(
          <div
            className="tooltip"
            style={{
              position: 'fixed',
              left: position.x,
              top: position.y,
              transform: 'translate(-50%, -100%)'
            }}
          >
            {content}
          </div>,
          document.body
        )}
    </>
  );
}

// Dropdown portal (escapes overflow: hidden)
function DropdownMenu({ trigger, items }: DropdownMenuProps) {
  const [isOpen, setIsOpen] = useState(false);
  const triggerRef = useRef<HTMLButtonElement>(null);

  const menuPosition = triggerRef.current?.getBoundingClientRect();

  return (
    <>
      <button ref={triggerRef} onClick={() => setIsOpen(!isOpen)}>
        {trigger}
      </button>
      {isOpen &&
        createPortal(
          <ul
            className="dropdown-menu"
            style={{
              position: 'fixed',
              top: (menuPosition?.bottom ?? 0) + 4,
              left: menuPosition?.left ?? 0
            }}
          >
            {items.map((item) => (
              <li key={item.id} onClick={item.onClick}>
                {item.label}
              </li>
            ))}
          </ul>,
          document.body
        )}
    </>
  );
}
```

---

## 11. Forwarding Refs

Ref forwarding lets components pass a ref through to a child DOM element.

```tsx
import React, { forwardRef, useRef, useImperativeHandle } from 'react';

// Basic ref forwarding
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'danger';
}

const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ variant = 'primary', children, ...props }, ref) => (
    <button ref={ref} className={`btn btn-${variant}`} {...props}>
      {children}
    </button>
  )
);
Button.displayName = 'Button';

// Usage
function Form() {
  const buttonRef = useRef<HTMLButtonElement>(null);

  useEffect(() => {
    buttonRef.current?.focus();
  }, []);

  return <Button ref={buttonRef}>Submit</Button>;
}

// Forwarding refs with input components
interface TextInputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
  error?: string;
}

const TextInput = forwardRef<HTMLInputElement, TextInputProps>(
  ({ label, error, ...props }, ref) => (
    <div className="input-group">
      <label>{label}</label>
      <input ref={ref} {...props} />
      {error && <span className="error">{error}</span>}
    </div>
  )
);
TextInput.displayName = 'TextInput';

// useImperativeHandle - customize exposed ref value
interface VideoPlayerRef {
  play: () => void;
  pause: () => void;
  seek: (time: number) => void;
}

interface VideoPlayerProps {
  src: string;
}

const VideoPlayer = forwardRef<VideoPlayerRef, VideoPlayerProps>(
  ({ src }, ref) => {
    const videoRef = useRef<HTMLVideoElement>(null);

    useImperativeHandle(ref, () => ({
      play: () => videoRef.current?.play(),
      pause: () => videoRef.current?.pause(),
      seek: (time: number) => {
        if (videoRef.current) {
          videoRef.current.currentTime = time;
        }
      }
    }));

    return <video ref={videoRef} src={src} />;
  }
);
VideoPlayer.displayName = 'VideoPlayer';

// Usage
function VideoPage() {
  const playerRef = useRef<VideoPlayerRef>(null);

  return (
    <div>
      <VideoPlayer ref={playerRef} src="/video.mp4" />
      <button onClick={() => playerRef.current?.play()}>Play</button>
      <button onClick={() => playerRef.current?.pause()}>Pause</button>
      <button onClick={() => playerRef.current?.seek(30)}>Skip to 30s</button>
    </div>
  );
}
```

---

## 12. Fragments

Fragments let you group children without adding extra nodes to the DOM.

```tsx
import React, { Fragment } from 'react';

// Short syntax
function Columns() {
  return (
    <>
      <td>Column 1</td>
      <td>Column 2</td>
      <td>Column 3</td>
    </>
  );
}

// Explicit Fragment (required when using keys)
function Glossary({ items }: { items: Array<{ term: string; description: string }> }) {
  return (
    <dl>
      {items.map((item) => (
        <Fragment key={item.term}>
          <dt>{item.term}</dt>
          <dd>{item.description}</dd>
        </Fragment>
      ))}
    </dl>
  );
}

// Avoiding unnecessary wrapper divs
function UserInfo({ user }: { user: User }) {
  // Bad: adds unnecessary div
  // return (
  //   <div>
  //     <span>{user.name}</span>
  //     <span>{user.email}</span>
  //   </div>
  // );

  // Good: no extra DOM node
  return (
    <>
      <span>{user.name}</span>
      <span>{user.email}</span>
    </>
  );
}

// Conditional fragments
function ConditionalContent({ showExtra }: { showExtra: boolean }) {
  return (
    <div>
      <h1>Title</h1>
      {showExtra && (
        <>
          <p>Extra paragraph 1</p>
          <p>Extra paragraph 2</p>
        </>
      )}
    </div>
  );
}
```

---

## 13. Context for Prop Drilling

Context provides a way to pass data through the component tree without prop drilling.

```tsx
import React, { createContext, useContext, useState, useMemo } from 'react';

// Theme context example
interface Theme {
  mode: 'light' | 'dark';
  colors: {
    primary: string;
    background: string;
    text: string;
  };
}

interface ThemeContextType {
  theme: Theme;
  toggleTheme: () => void;
}

const ThemeContext = createContext<ThemeContextType | null>(null);

// Custom hook for consuming context
function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within a ThemeProvider');
  }
  return context;
}

// Provider component
function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [mode, setMode] = useState<'light' | 'dark'>('light');

  const theme = useMemo<Theme>(() => ({
    mode,
    colors: mode === 'light'
      ? { primary: '#007bff', background: '#ffffff', text: '#333333' }
      : { primary: '#66b3ff', background: '#1a1a1a', text: '#ffffff' }
  }), [mode]);

  const toggleTheme = () => setMode((prev) => (prev === 'light' ? 'dark' : 'light'));

  const value = useMemo(() => ({ theme, toggleTheme }), [theme]);

  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
}

// Multi-context pattern for complex state
interface User {
  id: string;
  name: string;
  email: string;
}

const UserContext = createContext<User | null>(null);
const UserDispatchContext = createContext<React.Dispatch<UserAction> | null>(null);

type UserAction =
  | { type: 'SET_USER'; payload: User }
  | { type: 'UPDATE_NAME'; payload: string }
  | { type: 'LOGOUT' };

function userReducer(state: User | null, action: UserAction): User | null {
  switch (action.type) {
    case 'SET_USER':
      return action.payload;
    case 'UPDATE_NAME':
      return state ? { ...state, name: action.payload } : null;
    case 'LOGOUT':
      return null;
    default:
      return state;
  }
}

function UserProvider({ children }: { children: React.ReactNode }) {
  const [user, dispatch] = useReducer(userReducer, null);

  return (
    <UserContext.Provider value={user}>
      <UserDispatchContext.Provider value={dispatch}>
        {children}
      </UserDispatchContext.Provider>
    </UserContext.Provider>
  );
}

// Consuming hooks
function useUser() {
  return useContext(UserContext);
}

function useUserDispatch() {
  const dispatch = useContext(UserDispatchContext);
  if (!dispatch) {
    throw new Error('useUserDispatch must be used within UserProvider');
  }
  return dispatch;
}

// Composing providers
function AppProviders({ children }: { children: React.ReactNode }) {
  return (
    <ThemeProvider>
      <UserProvider>
        <NotificationProvider>
          {children}
        </NotificationProvider>
      </UserProvider>
    </ThemeProvider>
  );
}
```

---

## 14. Best Practices and Common Pitfalls

### Best Practices

```tsx
// 1. Keep components small and focused
// Bad: monolithic component
function BadUserDashboard() {
  // 500 lines of mixed concerns...
}

// Good: composed from smaller components
function GoodUserDashboard() {
  return (
    <DashboardLayout>
      <UserHeader />
      <UserStats />
      <RecentActivity />
      <QuickActions />
    </DashboardLayout>
  );
}

// 2. Lift state up only when necessary
// State should live in the lowest common ancestor that needs it

// 3. Use composition over prop drilling
// Instead of passing props through many levels, use composition or context

// 4. Memoize expensive computations and callbacks
function ExpensiveList({ items, filter }: Props) {
  const filteredItems = useMemo(
    () => items.filter((item) => item.name.includes(filter)),
    [items, filter]
  );

  const handleClick = useCallback((id: string) => {
    console.log('Clicked:', id);
  }, []);

  return (
    <ul>
      {filteredItems.map((item) => (
        <ListItem key={item.id} item={item} onClick={handleClick} />
      ))}
    </ul>
  );
}

// 5. Use React.memo for expensive child components
const ExpensiveChild = React.memo(function ExpensiveChild({ data }: Props) {
  // Expensive rendering logic
  return <div>{/* ... */}</div>;
});

// 6. Always provide displayName for forwardRef and memo components
const Button = forwardRef<HTMLButtonElement, ButtonProps>((props, ref) => {
  return <button ref={ref} {...props} />;
});
Button.displayName = 'Button';
```

### Common Pitfalls

```tsx
// PITFALL 1: Creating new objects/arrays in render
// Bad: creates new array every render, causing child re-renders
function BadParent() {
  return <Child items={[1, 2, 3]} />; // New array reference each render
}

// Good: stable reference
const ITEMS = [1, 2, 3];
function GoodParent() {
  return <Child items={ITEMS} />;
}

// PITFALL 2: Inline function definitions breaking memoization
// Bad: new function every render
function BadList({ items }: Props) {
  return items.map((item) => (
    <MemoizedItem
      key={item.id}
      onClick={() => handleClick(item.id)} // New function each render!
    />
  ));
}

// Good: useCallback with proper dependencies
function GoodList({ items }: Props) {
  const handleClick = useCallback((id: string) => {
    console.log('Clicked:', id);
  }, []);

  return items.map((item) => (
    <MemoizedItem key={item.id} id={item.id} onClick={handleClick} />
  ));
}

// PITFALL 3: Missing keys or using index as key for dynamic lists
// Bad: using index as key
{items.map((item, index) => <Item key={index} {...item} />)}

// Good: using stable unique identifier
{items.map((item) => <Item key={item.id} {...item} />)}

// PITFALL 4: Mutating state directly
// Bad: direct mutation
const handleAdd = () => {
  items.push(newItem); // Mutating!
  setItems(items);
};

// Good: create new reference
const handleAdd = () => {
  setItems([...items, newItem]);
};

// PITFALL 5: Not cleaning up effects
// Bad: memory leak
useEffect(() => {
  const subscription = api.subscribe(handleUpdate);
  // Missing cleanup!
}, []);

// Good: proper cleanup
useEffect(() => {
  const subscription = api.subscribe(handleUpdate);
  return () => subscription.unsubscribe();
}, []);

// PITFALL 6: Overusing context for frequently changing values
// Bad: causes all consumers to re-render
const BadContext = createContext({ x: 0, y: 0 }); // Mouse position

// Good: split contexts or use state management library
const MouseXContext = createContext(0);
const MouseYContext = createContext(0);

// PITFALL 7: Not handling loading and error states
// Bad: assumes data always exists
function BadDataDisplay({ data }) {
  return <div>{data.name}</div>; // Crashes if data is undefined
}

// Good: handle all states
function GoodDataDisplay({ data, loading, error }) {
  if (loading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  if (!data) return <EmptyState />;
  return <div>{data.name}</div>;
}
```

---

## Quick Reference

| Pattern | Use Case |
|---------|----------|
| Function Components | Default for all components |
| forwardRef | Exposing DOM refs to parent components |
| Compound Components | Related components that share state |
| Render Props | Sharing behavior with flexible rendering |
| HOCs | Cross-cutting concerns (auth, logging) |
| Context | Avoiding prop drilling for global state |
| Portals | Rendering outside DOM hierarchy (modals) |
| Error Boundaries | Graceful error handling in UI |
| Suspense | Code splitting and async loading |
| Fragments | Grouping without extra DOM nodes |
