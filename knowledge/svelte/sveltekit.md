# SvelteKit

> Official Documentation: https://kit.svelte.dev/

## Overview

SvelteKit is the official application framework for Svelte, providing routing, server-side rendering, API routes, and build optimization out of the box.

Key features:
- **File-based routing** - Routes from filesystem structure
- **SSR/SSG/CSR** - Multiple rendering modes
- **API routes** - Server endpoints alongside pages
- **Form actions** - Server-side form handling
- **Adapters** - Deploy anywhere (Node, Vercel, Cloudflare, etc.)

---

## Project Structure

```
my-app/
├── src/
│   ├── lib/                 # Reusable code ($lib alias)
│   │   ├── components/
│   │   ├── server/          # Server-only code ($lib/server)
│   │   └── utils/
│   ├── routes/              # Route definitions
│   │   ├── +page.svelte     # Home page
│   │   ├── +page.ts         # Page load function
│   │   ├── +layout.svelte   # Root layout
│   │   ├── +error.svelte    # Error page
│   │   ├── about/
│   │   │   └── +page.svelte
│   │   └── api/
│   │       └── users/
│   │           └── +server.ts
│   ├── app.html             # HTML template
│   ├── app.d.ts             # TypeScript declarations
│   └── hooks.server.ts      # Server hooks
├── static/                  # Static assets
├── svelte.config.js
├── vite.config.ts
└── package.json
```

---

## Routing

### Page Routes

```svelte
<!-- src/routes/+page.svelte -->
<h1>Home Page</h1>

<!-- src/routes/about/+page.svelte -->
<h1>About Page</h1>

<!-- src/routes/blog/[slug]/+page.svelte -->
<script lang="ts">
  let { data } = $props();
</script>

<h1>{data.post.title}</h1>
<p>{data.post.content}</p>
```

### Dynamic Routes

```
src/routes/
├── blog/
│   ├── [slug]/           # /blog/hello-world
│   │   └── +page.svelte
│   └── [...rest]/        # /blog/2024/01/post (catch-all)
│       └── +page.svelte
├── users/
│   └── [[optional]]/     # /users or /users/123 (optional param)
│       └── +page.svelte
└── (group)/              # Route group (no URL segment)
    ├── login/
    │   └── +page.svelte
    └── register/
        └── +page.svelte
```

### Route Groups

```
src/routes/
├── (marketing)/           # Grouped routes share layout
│   ├── +layout.svelte     # Marketing layout
│   ├── about/
│   └── pricing/
├── (app)/                 # App routes
│   ├── +layout.svelte     # App layout (authenticated)
│   ├── dashboard/
│   └── settings/
└── +layout.svelte         # Root layout (all routes)
```

---

## Load Functions

### Universal Load (+page.ts)

Runs on server (SSR) and client (navigation):

```typescript
// src/routes/blog/[slug]/+page.ts
import type { PageLoad } from './$types';

export const load: PageLoad = async ({ params, fetch, parent }) => {
  const { slug } = params;

  // Wait for parent data if needed
  const parentData = await parent();

  // Fetch data (uses SvelteKit's enhanced fetch)
  const response = await fetch(`/api/posts/${slug}`);
  const post = await response.json();

  return {
    post,
    seo: {
      title: post.title,
      description: post.excerpt
    }
  };
};
```

### Server Load (+page.server.ts)

Runs only on server (access to secrets, database):

```typescript
// src/routes/dashboard/+page.server.ts
import type { PageServerLoad } from './$types';
import { db } from '$lib/server/database';
import { error, redirect } from '@sveltejs/kit';

export const load: PageServerLoad = async ({ locals, cookies }) => {
  // Access session from hooks
  if (!locals.user) {
    throw redirect(303, '/login');
  }

  const userId = locals.user.id;

  // Direct database access
  const stats = await db.query('SELECT * FROM stats WHERE user_id = $1', [userId]);

  if (!stats) {
    throw error(404, 'Stats not found');
  }

  return {
    stats,
    user: locals.user
  };
};
```

### Layout Load

```typescript
// src/routes/+layout.server.ts
import type { LayoutServerLoad } from './$types';

export const load: LayoutServerLoad = async ({ locals }) => {
  return {
    user: locals.user
  };
};

// src/routes/+layout.ts
import type { LayoutLoad } from './$types';

export const load: LayoutLoad = async ({ data }) => {
  return {
    ...data,
    theme: 'dark'
  };
};
```

---

## Form Actions

### Basic Actions

```typescript
// src/routes/login/+page.server.ts
import type { Actions } from './$types';
import { fail, redirect } from '@sveltejs/kit';
import { db } from '$lib/server/database';

export const actions: Actions = {
  default: async ({ request, cookies }) => {
    const data = await request.formData();
    const email = data.get('email')?.toString();
    const password = data.get('password')?.toString();

    if (!email || !password) {
      return fail(400, { email, missing: true });
    }

    const user = await db.authenticate(email, password);

    if (!user) {
      return fail(401, { email, incorrect: true });
    }

    cookies.set('session', user.sessionId, {
      path: '/',
      httpOnly: true,
      sameSite: 'strict',
      secure: process.env.NODE_ENV === 'production',
      maxAge: 60 * 60 * 24 * 7 // 1 week
    });

    throw redirect(303, '/dashboard');
  }
};
```

```svelte
<!-- src/routes/login/+page.svelte -->
<script lang="ts">
  import { enhance } from '$app/forms';
  import type { ActionData } from './$types';

  let { form }: { form: ActionData } = $props();
</script>

<form method="POST" use:enhance>
  {#if form?.missing}
    <p class="error">Please fill in all fields</p>
  {/if}
  {#if form?.incorrect}
    <p class="error">Invalid credentials</p>
  {/if}

  <label>
    Email
    <input name="email" type="email" value={form?.email ?? ''} />
  </label>

  <label>
    Password
    <input name="password" type="password" />
  </label>

  <button type="submit">Log in</button>
</form>
```

### Named Actions

```typescript
// src/routes/todos/+page.server.ts
import type { Actions } from './$types';

export const actions: Actions = {
  create: async ({ request }) => {
    const data = await request.formData();
    const text = data.get('text');
    // Create todo...
    return { success: true };
  },

  delete: async ({ request }) => {
    const data = await request.formData();
    const id = data.get('id');
    // Delete todo...
    return { success: true };
  }
};
```

```svelte
<!-- Using named actions -->
<form method="POST" action="?/create" use:enhance>
  <input name="text" />
  <button>Add Todo</button>
</form>

<form method="POST" action="?/delete" use:enhance>
  <input type="hidden" name="id" value={todo.id} />
  <button>Delete</button>
</form>
```

### Progressive Enhancement

```svelte
<script lang="ts">
  import { enhance } from '$app/forms';
  import type { SubmitFunction } from '@sveltejs/kit';

  let loading = $state(false);

  const handleSubmit: SubmitFunction = () => {
    loading = true;

    return async ({ result, update }) => {
      loading = false;

      if (result.type === 'success') {
        // Custom success handling
        await update();
      } else if (result.type === 'failure') {
        // Custom error handling
      }
    };
  };
</script>

<form method="POST" use:enhance={handleSubmit}>
  <button disabled={loading}>
    {loading ? 'Submitting...' : 'Submit'}
  </button>
</form>
```

---

## API Routes (+server.ts)

```typescript
// src/routes/api/users/+server.ts
import type { RequestHandler } from './$types';
import { json, error } from '@sveltejs/kit';
import { db } from '$lib/server/database';

export const GET: RequestHandler = async ({ url }) => {
  const page = Number(url.searchParams.get('page')) || 1;
  const limit = Number(url.searchParams.get('limit')) || 10;

  const users = await db.getUsers(page, limit);

  return json(users);
};

export const POST: RequestHandler = async ({ request, locals }) => {
  if (!locals.user?.isAdmin) {
    throw error(403, 'Forbidden');
  }

  const body = await request.json();
  const user = await db.createUser(body);

  return json(user, { status: 201 });
};

// src/routes/api/users/[id]/+server.ts
export const GET: RequestHandler = async ({ params }) => {
  const user = await db.getUser(params.id);

  if (!user) {
    throw error(404, 'User not found');
  }

  return json(user);
};

export const DELETE: RequestHandler = async ({ params }) => {
  await db.deleteUser(params.id);
  return new Response(null, { status: 204 });
};
```

---

## Hooks

### Server Hooks

```typescript
// src/hooks.server.ts
import type { Handle, HandleServerError } from '@sveltejs/kit';
import { db } from '$lib/server/database';

export const handle: Handle = async ({ event, resolve }) => {
  // Get session from cookie
  const sessionId = event.cookies.get('session');

  if (sessionId) {
    const user = await db.getUserBySession(sessionId);
    event.locals.user = user;
  }

  // Add security headers
  const response = await resolve(event, {
    transformPageChunk: ({ html }) => html.replace('%sveltekit.nonce%', crypto.randomUUID())
  });

  response.headers.set('X-Frame-Options', 'DENY');
  response.headers.set('X-Content-Type-Options', 'nosniff');

  return response;
};

export const handleError: HandleServerError = async ({ error, event }) => {
  console.error('Server error:', error);

  // Report to error tracking service
  // await reportError(error, event);

  return {
    message: 'An unexpected error occurred',
    code: 'INTERNAL_ERROR'
  };
};
```

### Client Hooks

```typescript
// src/hooks.client.ts
import type { HandleClientError } from '@sveltejs/kit';

export const handleError: HandleClientError = async ({ error, event }) => {
  console.error('Client error:', error);

  return {
    message: 'Something went wrong'
  };
};
```

---

## Page Options

```typescript
// src/routes/about/+page.ts
export const prerender = true;  // Generate static HTML at build
export const ssr = true;        // Enable server-side rendering
export const csr = true;        // Enable client-side rendering

// Prerender specific paths
export const entries = () => {
  return ['/about', '/about/team', '/about/contact'];
};

// src/routes/dashboard/+page.ts
export const prerender = false;  // Don't prerender (dynamic)
export const ssr = true;         // Render on server
export const csr = true;         // Hydrate on client

// src/routes/admin/+page.ts
export const ssr = false;  // Client-only (SPA mode)
```

---

## Environment Variables

```typescript
// .env
PUBLIC_API_URL=https://api.example.com
DATABASE_URL=postgres://...
SECRET_KEY=...

// Access in server code
import { DATABASE_URL, SECRET_KEY } from '$env/static/private';
import { env } from '$env/dynamic/private';

const db = connect(DATABASE_URL);
const secret = env.SECRET_KEY;

// Access in client/server code
import { PUBLIC_API_URL } from '$env/static/public';
import { env } from '$env/dynamic/public';

const apiUrl = PUBLIC_API_URL;
```

---

## Navigation

### Programmatic Navigation

```svelte
<script lang="ts">
  import { goto, invalidate, invalidateAll } from '$app/navigation';
  import { page } from '$app/stores';

  async function handleClick() {
    // Navigate
    await goto('/dashboard');

    // With options
    await goto('/login', {
      replaceState: true,
      invalidateAll: true
    });
  }

  async function refresh() {
    // Refresh specific data
    await invalidate('/api/user');

    // Refresh all load functions
    await invalidateAll();
  }

  // Reactive to route changes
  $effect(() => {
    console.log('Current path:', $page.url.pathname);
  });
</script>
```

### Link Prefetching

```svelte
<!-- Prefetch on hover (default) -->
<a href="/about">About</a>

<!-- Prefetch on viewport enter -->
<a href="/about" data-sveltekit-preload-data="hover">About</a>

<!-- Disable prefetching -->
<a href="/about" data-sveltekit-preload-data="off">About</a>

<!-- Preload code only (not data) -->
<a href="/about" data-sveltekit-preload-code>About</a>
```

---

## Adapters

```javascript
// svelte.config.js
import adapter from '@sveltejs/adapter-auto';      // Auto-detect platform
import adapterNode from '@sveltejs/adapter-node';  // Node.js server
import adapterStatic from '@sveltejs/adapter-static'; // Static site
import adapterVercel from '@sveltejs/adapter-vercel';
import adapterCloudflare from '@sveltejs/adapter-cloudflare';

export default {
  kit: {
    adapter: adapterNode({
      out: 'build',
      precompress: true,
      envPrefix: 'MY_APP_'
    })
  }
};

// Static adapter for fully static sites
export default {
  kit: {
    adapter: adapterStatic({
      pages: 'build',
      assets: 'build',
      fallback: '404.html',
      precompress: false,
      strict: true
    })
  }
};
```

---

## TypeScript

```typescript
// src/app.d.ts
declare global {
  namespace App {
    interface Error {
      message: string;
      code?: string;
    }
    interface Locals {
      user?: {
        id: string;
        email: string;
        isAdmin: boolean;
      };
    }
    interface PageData {
      user?: App.Locals['user'];
    }
    interface PageState {}
    interface Platform {}
  }
}

export {};
```

---

## Related Topics

- [Svelte SKILL](../../frontend-frameworks/svelte/SKILL.md) - Svelte core patterns
- [Skeleton Basics](../skeleton/basics.md) - Skeleton UI with SvelteKit
- [Tailwind Utilities](../tailwind/utilities.md) - Tailwind CSS reference
