# React Router v6+ Reference

## Basic Setup

```tsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/users/:userId" element={<UserProfile />} />
        <Route path="*" element={<NotFound />} />
      </Routes>
    </BrowserRouter>
  );
}
```

## Navigation

```tsx
import { Link, NavLink, useNavigate } from 'react-router-dom';

function Navigation() {
  const navigate = useNavigate();

  return (
    <nav>
      <Link to="/">Home</Link>

      <NavLink
        to="/about"
        className={({ isActive }) => isActive ? 'active' : ''}
      >
        About
      </NavLink>

      <button onClick={() => navigate('/dashboard')}>Dashboard</button>
      <button onClick={() => navigate(-1)}>Back</button>
    </nav>
  );
}
```

## Route Parameters

```tsx
import { useParams, useSearchParams } from 'react-router-dom';

// /users/:userId
function UserProfile() {
  const { userId } = useParams<{ userId: string }>();
  return <div>User ID: {userId}</div>;
}

// /products?category=electronics&page=2
function ProductList() {
  const [searchParams, setSearchParams] = useSearchParams();

  const category = searchParams.get('category') || 'all';
  const page = parseInt(searchParams.get('page') || '1');

  const nextPage = () => {
    setSearchParams(prev => {
      prev.set('page', String(page + 1));
      return prev;
    });
  };

  return <div>...</div>;
}
```

## Nested Routes

```tsx
<Routes>
  <Route path="/dashboard" element={<DashboardLayout />}>
    <Route index element={<DashboardHome />} />
    <Route path="analytics" element={<Analytics />} />
    <Route path="users">
      <Route index element={<UserList />} />
      <Route path=":userId" element={<UserDetail />} />
    </Route>
  </Route>
</Routes>

// Parent layout with Outlet
import { Outlet, Link } from 'react-router-dom';

function DashboardLayout() {
  return (
    <div className="dashboard">
      <nav>
        <Link to="/dashboard">Home</Link>
        <Link to="/dashboard/analytics">Analytics</Link>
      </nav>
      <main>
        <Outlet />
      </main>
    </div>
  );
}
```

## Protected Routes

```tsx
import { Navigate, Outlet, useLocation } from 'react-router-dom';

function ProtectedRoute() {
  const { user, isLoading } = useAuth();
  const location = useLocation();

  if (isLoading) return <LoadingSpinner />;

  if (!user) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  return <Outlet />;
}

// Usage
<Routes>
  <Route path="/login" element={<Login />} />
  <Route element={<ProtectedRoute />}>
    <Route path="/dashboard" element={<Dashboard />} />
    <Route path="/settings" element={<Settings />} />
  </Route>
</Routes>
```

## Data Loading (v6.4+)

```tsx
import {
  createBrowserRouter,
  RouterProvider,
  useLoaderData,
} from 'react-router-dom';

async function userLoader({ params }) {
  const response = await fetch(`/api/users/${params.userId}`);
  if (!response.ok) throw new Response('Not found', { status: 404 });
  return response.json();
}

const router = createBrowserRouter([
  {
    path: '/users/:userId',
    element: <UserProfile />,
    loader: userLoader,
    errorElement: <UserError />,
  },
]);

function UserProfile() {
  const user = useLoaderData() as User;
  return <div>{user.name}</div>;
}

function App() {
  return <RouterProvider router={router} />;
}
```

## Actions for Mutations

```tsx
async function updateUserAction({ request, params }) {
  const formData = await request.formData();

  const response = await fetch(`/api/users/${params.userId}`, {
    method: 'PUT',
    body: formData,
  });

  if (!response.ok) return { error: 'Failed to update' };
  return redirect('/users');
}

// Component with Form
import { Form, useActionData, useNavigation } from 'react-router-dom';

function EditUser() {
  const user = useLoaderData() as User;
  const actionData = useActionData() as { error?: string };
  const navigation = useNavigation();
  const isSubmitting = navigation.state === 'submitting';

  return (
    <Form method="post">
      <input name="name" defaultValue={user.name} />
      {actionData?.error && <p className="error">{actionData.error}</p>}
      <button disabled={isSubmitting}>
        {isSubmitting ? 'Saving...' : 'Save'}
      </button>
    </Form>
  );
}
```

## Error Handling

```tsx
import { useRouteError, isRouteErrorResponse } from 'react-router-dom';

function ErrorPage() {
  const error = useRouteError();

  if (isRouteErrorResponse(error)) {
    if (error.status === 404) {
      return <div>Page not found</div>;
    }
    return <div>Error {error.status}: {error.statusText}</div>;
  }

  return <div>Something went wrong</div>;
}
```

## Scroll Restoration

```tsx
import { ScrollRestoration } from 'react-router-dom';

function RootLayout() {
  return (
    <>
      <Header />
      <Outlet />
      <ScrollRestoration />
    </>
  );
}
```

## Relative Links

```tsx
function UserDetail() {
  const { userId } = useParams();

  return (
    <div>
      <h1>User {userId}</h1>
      <Link to="edit">Edit</Link>        {/* → /users/:userId/edit */}
      <Link to="../">Back to List</Link>  {/* → /users */}
    </div>
  );
}
```

## Route Configuration Object

```tsx
const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <RootError />,
    children: [
      { index: true, element: <Home /> },
      { path: 'about', element: <About /> },
      {
        path: 'products',
        element: <ProductsLayout />,
        children: [
          { index: true, element: <ProductList />, loader: productsLoader },
          { path: ':productId', element: <ProductDetail />, loader: productLoader },
        ],
      },
    ],
  },
]);
```
