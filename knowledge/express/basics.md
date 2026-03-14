# Express.js Basics

Express is a minimal and flexible Node.js web application framework that provides a robust set of features for web and mobile applications. This comprehensive guide covers all fundamental concepts based on official Express documentation.

---

## Table of Contents

1. [Application Setup](#1-application-setup)
2. [Routing](#2-routing)
3. [Route Parameters and Query Strings](#3-route-parameters-and-query-strings)
4. [Request Object](#4-request-object)
5. [Response Object](#5-response-object)
6. [Route Handlers](#6-route-handlers)
7. [express.Router() for Modular Routes](#7-expressrouter-for-modular-routes)
8. [Static Files](#8-static-files)
9. [Body Parsing](#9-body-parsing)
10. [Error Handling](#10-error-handling)
11. [Template Engines](#11-template-engines)
12. [Cookies and Sessions](#12-cookies-and-sessions)
13. [CORS Configuration](#13-cors-configuration)
14. [Security Best Practices](#14-security-best-practices)
15. [Project Structure Patterns](#15-project-structure-patterns)
16. [TypeScript with Express](#16-typescript-with-express)

---

## 1. Application Setup

### Creating an Express Application

The `express()` function is a top-level function exported by the Express module. It creates an Express application object.

```javascript
const express = require('express');
const app = express();
```

With ES modules:

```javascript
import express from 'express';
const app = express();
```

### Application Settings

Express provides methods to configure application behavior:

```javascript
const app = express();

// Set application settings
app.set('view engine', 'ejs');
app.set('views', './views');
app.set('case sensitive routing', true);
app.set('strict routing', false);
app.set('x-powered-by', false); // Hide Express fingerprint
app.set('trust proxy', true); // Trust reverse proxy

// Get setting value
const viewEngine = app.get('view engine');

// Enable/disable settings
app.enable('trust proxy');
app.disable('x-powered-by');

// Check if enabled
if (app.enabled('trust proxy')) {
  console.log('Trust proxy is enabled');
}
```

### Common Application Settings

| Setting | Description | Default |
|---------|-------------|---------|
| `case sensitive routing` | Enable case sensitivity | Disabled |
| `env` | Environment mode | `process.env.NODE_ENV` or "development" |
| `etag` | Set ETag response header | `weak` |
| `jsonp callback name` | JSONP callback name | `callback` |
| `json escape` | Enable escaping JSON responses | Disabled |
| `json replacer` | JSON replacer function | undefined |
| `json spaces` | JSON response indentation | Disabled |
| `query parser` | Query string parser | `extended` |
| `strict routing` | Enable strict routing | Disabled |
| `subdomain offset` | Subdomain offset | 2 |
| `trust proxy` | Trust proxy headers | Disabled |
| `views` | View template directory | `process.cwd() + '/views'` |
| `view cache` | Enable view template caching | Enabled in production |
| `view engine` | Default template engine | undefined |
| `x-powered-by` | Enable X-Powered-By header | Enabled |

### Starting the Server with app.listen()

The `app.listen()` method binds and listens for connections on the specified host and port:

```javascript
const PORT = process.env.PORT || 3000;

// Basic usage
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

// With host specification
app.listen(PORT, '0.0.0.0', () => {
  console.log(`Server running on http://0.0.0.0:${PORT}`);
});

// With backlog (maximum pending connections)
app.listen(PORT, '0.0.0.0', 511, () => {
  console.log('Server started with custom backlog');
});

// Returns http.Server instance
const server = app.listen(PORT);
server.on('error', (err) => {
  if (err.code === 'EADDRINUSE') {
    console.error(`Port ${PORT} is already in use`);
  }
});
```

### Using with HTTP/HTTPS Servers

```javascript
const http = require('http');
const https = require('https');
const fs = require('fs');

// HTTP server
const httpServer = http.createServer(app);
httpServer.listen(80);

// HTTPS server
const httpsOptions = {
  key: fs.readFileSync('server.key'),
  cert: fs.readFileSync('server.cert')
};
const httpsServer = https.createServer(httpsOptions, app);
httpsServer.listen(443);
```

### Complete Application Setup Example

```javascript
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';
import compression from 'compression';

const app = express();

// Security middleware
app.use(helmet());

// CORS configuration
app.use(cors());

// Request logging
app.use(morgan('combined'));

// Response compression
app.use(compression());

// Body parsing
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true, limit: '10mb' }));

// Disable X-Powered-By header
app.disable('x-powered-by');

// Trust proxy (for apps behind reverse proxy)
app.set('trust proxy', 1);

// Start server
const PORT = process.env.PORT || 3000;
const server = app.listen(PORT, () => {
  console.log(`Server running in ${process.env.NODE_ENV || 'development'} mode`);
  console.log(`Listening on port ${PORT}`);
});

// Graceful shutdown
process.on('SIGTERM', () => {
  console.log('SIGTERM received, shutting down gracefully');
  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
});

export default app;
```

---

## 2. Routing

Routing refers to how an application's endpoints (URIs) respond to client requests. Express supports all HTTP methods.

### HTTP Methods

```javascript
// GET - Retrieve resources
app.get('/users', (req, res) => {
  res.json({ users: [] });
});

// POST - Create resources
app.post('/users', (req, res) => {
  const newUser = req.body;
  res.status(201).json(newUser);
});

// PUT - Replace entire resource
app.put('/users/:id', (req, res) => {
  const { id } = req.params;
  const updatedUser = req.body;
  res.json({ id, ...updatedUser });
});

// PATCH - Partial update
app.patch('/users/:id', (req, res) => {
  const { id } = req.params;
  const partialUpdate = req.body;
  res.json({ id, ...partialUpdate });
});

// DELETE - Remove resource
app.delete('/users/:id', (req, res) => {
  res.status(204).send();
});

// HEAD - Same as GET without response body
app.head('/users', (req, res) => {
  res.set('X-Total-Count', '100');
  res.end();
});

// OPTIONS - Get supported methods
app.options('/users', (req, res) => {
  res.set('Allow', 'GET, POST, OPTIONS');
  res.status(204).send();
});

// ALL - Match all HTTP methods
app.all('/api/*', (req, res, next) => {
  console.log(`API request: ${req.method} ${req.path}`);
  next();
});
```

### Route Paths

Express supports various route path patterns:

```javascript
// String paths
app.get('/', handler);
app.get('/about', handler);
app.get('/users/profile', handler);

// String patterns with special characters
app.get('/ab?cd', handler);      // Matches /acd and /abcd
app.get('/ab+cd', handler);      // Matches /abcd, /abbcd, /abbbcd, etc.
app.get('/ab*cd', handler);      // Matches /abcd, /abxcd, /ab123cd, etc.
app.get('/ab(cd)?e', handler);   // Matches /abe and /abcde

// Regular expressions
app.get(/.*fly$/, handler);      // Matches butterfly, dragonfly, etc.
app.get(/^\/users\/\d+$/, handler); // Matches /users/123

// Array of paths
app.get(['/users', '/people'], handler);
```

### Route Path Parameters

```javascript
// Single parameter
app.get('/users/:userId', (req, res) => {
  res.json({ userId: req.params.userId });
});

// Multiple parameters
app.get('/users/:userId/posts/:postId', (req, res) => {
  const { userId, postId } = req.params;
  res.json({ userId, postId });
});

// Optional parameters (using regex)
app.get('/files/:name.:ext?', (req, res) => {
  const { name, ext } = req.params;
  res.json({ name, ext: ext || 'txt' });
});

// Parameter constraints with regex
app.get('/users/:id(\\d+)', (req, res) => {
  // Only matches numeric IDs
  res.json({ id: parseInt(req.params.id) });
});

// Hyphenated parameters
app.get('/flights/:from-:to', (req, res) => {
  res.json({ from: req.params.from, to: req.params.to });
});

// Dot-separated parameters
app.get('/plantae/:genus.:species', (req, res) => {
  res.json({ genus: req.params.genus, species: req.params.species });
});
```

### app.route() - Chainable Route Handlers

```javascript
app.route('/books')
  .get((req, res) => {
    res.json({ books: [] });
  })
  .post((req, res) => {
    res.status(201).json(req.body);
  })
  .put((req, res) => {
    res.json({ message: 'Books updated' });
  });

app.route('/books/:id')
  .get((req, res) => {
    res.json({ id: req.params.id });
  })
  .put((req, res) => {
    res.json({ id: req.params.id, ...req.body });
  })
  .delete((req, res) => {
    res.status(204).send();
  });
```

---

## 3. Route Parameters and Query Strings

### Route Parameters

Route parameters are named URL segments used to capture values at specific positions in the URL.

```javascript
// Basic parameter
app.get('/users/:id', (req, res) => {
  console.log(req.params); // { id: '123' }
  res.json({ userId: req.params.id });
});

// Multiple parameters
app.get('/posts/:year/:month/:day', (req, res) => {
  const { year, month, day } = req.params;
  res.json({ date: `${year}-${month}-${day}` });
});

// Parameter validation middleware
app.param('id', (req, res, next, value) => {
  // Validate ID format
  if (!/^\d+$/.test(value)) {
    return res.status(400).json({ error: 'Invalid ID format' });
  }
  req.userId = parseInt(value);
  next();
});

app.get('/users/:id', (req, res) => {
  res.json({ userId: req.userId });
});

// Custom parameter validation
app.param('userId', async (req, res, next, id) => {
  try {
    const user = await User.findById(id);
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    req.user = user;
    next();
  } catch (err) {
    next(err);
  }
});
```

### Query Strings

Query strings are parsed automatically and available via `req.query`.

```javascript
// URL: /search?q=express&page=1&limit=10
app.get('/search', (req, res) => {
  const { q, page = 1, limit = 10 } = req.query;

  res.json({
    query: q,
    page: parseInt(page),
    limit: parseInt(limit)
  });
});

// Array query parameters
// URL: /items?tags=node&tags=express&tags=api
app.get('/items', (req, res) => {
  const { tags } = req.query; // ['node', 'express', 'api']
  res.json({ tags });
});

// Nested query parameters (with extended parser)
// URL: /filter?user[name]=john&user[age]=30
app.get('/filter', (req, res) => {
  const { user } = req.query; // { name: 'john', age: '30' }
  res.json({ user });
});

// Query parser configuration
app.set('query parser', 'extended'); // Default, supports nested objects
app.set('query parser', 'simple');   // Simple parsing, no nested objects
app.set('query parser', false);      // Disable query parsing

// Custom query parser
app.set('query parser', (queryString) => {
  return new URLSearchParams(queryString);
});
```

### Combining Parameters and Query Strings

```javascript
// URL: /users/123/posts?status=published&sort=date
app.get('/users/:userId/posts', (req, res) => {
  const { userId } = req.params;
  const { status = 'all', sort = 'created' } = req.query;

  res.json({
    userId,
    filters: { status, sort }
  });
});

// Pagination with validation
app.get('/products', (req, res) => {
  let { page, limit, sort, order } = req.query;

  // Parse and validate
  page = Math.max(1, parseInt(page) || 1);
  limit = Math.min(100, Math.max(1, parseInt(limit) || 20));
  sort = ['name', 'price', 'date'].includes(sort) ? sort : 'date';
  order = order === 'asc' ? 'asc' : 'desc';

  res.json({
    pagination: { page, limit },
    sorting: { sort, order }
  });
});
```

---

## 4. Request Object

The `req` object represents the HTTP request and has properties for the request query string, parameters, body, HTTP headers, and more.

### Request Properties

```javascript
app.get('/request-info', (req, res) => {
  const info = {
    // URL information
    path: req.path,                    // '/request-info'
    originalUrl: req.originalUrl,      // '/request-info?foo=bar'
    baseUrl: req.baseUrl,              // Base URL of router mount
    url: req.url,                      // URL after baseUrl

    // HTTP method
    method: req.method,                // 'GET', 'POST', etc.

    // Protocol and security
    protocol: req.protocol,            // 'http' or 'https'
    secure: req.secure,                // true if HTTPS

    // Host information
    hostname: req.hostname,            // 'example.com'
    subdomains: req.subdomains,        // ['api'] for 'api.example.com'

    // Client information
    ip: req.ip,                        // Client IP
    ips: req.ips,                      // Proxy chain IPs (trust proxy)

    // Content type
    xhr: req.xhr,                      // true if X-Requested-With: XMLHttpRequest
    fresh: req.fresh,                  // Response still fresh
    stale: req.stale,                  // Response is stale
  };

  res.json(info);
});
```

### req.params - Route Parameters

```javascript
// Route: /users/:userId/posts/:postId
app.get('/users/:userId/posts/:postId', (req, res) => {
  // URL: /users/42/posts/123
  console.log(req.params);
  // { userId: '42', postId: '123' }

  const { userId, postId } = req.params;
  res.json({ userId, postId });
});
```

### req.query - Query String Parameters

```javascript
app.get('/search', (req, res) => {
  // URL: /search?q=express&page=2&filters[status]=active
  console.log(req.query);
  // { q: 'express', page: '2', filters: { status: 'active' } }

  const { q, page, filters } = req.query;
  res.json({ query: q, page, filters });
});
```

### req.body - Request Body

```javascript
// Requires body-parsing middleware
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

app.post('/users', (req, res) => {
  console.log(req.body);
  // { name: 'John', email: 'john@example.com' }

  const { name, email, password } = req.body;
  res.status(201).json({ name, email });
});
```

### req.headers - Request Headers

```javascript
app.get('/headers', (req, res) => {
  // All headers (lowercase keys)
  console.log(req.headers);

  // Specific headers
  const authorization = req.headers.authorization;
  const contentType = req.headers['content-type'];
  const userAgent = req.headers['user-agent'];

  // Using req.get() or req.header()
  const acceptHeader = req.get('Accept');
  const customHeader = req.header('X-Custom-Header');

  // Check content type
  if (req.is('json')) {
    console.log('Request has JSON content type');
  }

  if (req.is('application/*')) {
    console.log('Application content type');
  }

  res.json({
    authorization: authorization ? 'Present' : 'Missing',
    contentType,
    userAgent
  });
});
```

### Request Methods

```javascript
app.all('/api/*', (req, res, next) => {
  // req.accepts() - Check acceptable content types
  if (req.accepts('json')) {
    res.type('json');
  } else if (req.accepts('html')) {
    res.type('html');
  }

  // req.acceptsCharsets() - Check acceptable character sets
  const charset = req.acceptsCharsets('utf-8', 'iso-8859-1');

  // req.acceptsEncodings() - Check acceptable encodings
  const encoding = req.acceptsEncodings('gzip', 'deflate');

  // req.acceptsLanguages() - Check acceptable languages
  const lang = req.acceptsLanguages('en', 'es', 'fr');

  // req.range() - Parse Range header
  const range = req.range(1000); // File size

  next();
});
```

### Complete Request Example

```javascript
app.post('/api/users/:id/profile', (req, res) => {
  // Gather all request information
  const requestInfo = {
    // Route parameters
    params: req.params,           // { id: '123' }

    // Query string
    query: req.query,             // { include: 'posts' }

    // Request body
    body: req.body,               // { name: 'John', bio: '...' }

    // Headers
    headers: {
      authorization: req.get('Authorization'),
      contentType: req.get('Content-Type'),
      userAgent: req.get('User-Agent')
    },

    // Request metadata
    meta: {
      method: req.method,
      path: req.path,
      originalUrl: req.originalUrl,
      protocol: req.protocol,
      secure: req.secure,
      ip: req.ip,
      hostname: req.hostname
    }
  };

  res.json(requestInfo);
});
```

---

## 5. Response Object

The `res` object represents the HTTP response that an Express app sends when it gets an HTTP request.

### res.send() - Send Response

```javascript
app.get('/send-examples', (req, res) => {
  // Send string (Content-Type: text/html)
  res.send('<h1>Hello World</h1>');

  // Send object (Content-Type: application/json)
  res.send({ message: 'Hello' });

  // Send array (Content-Type: application/json)
  res.send([1, 2, 3]);

  // Send Buffer (Content-Type: application/octet-stream)
  res.send(Buffer.from('whoop'));
});
```

### res.json() - Send JSON Response

```javascript
app.get('/json-examples', (req, res) => {
  // Basic JSON
  res.json({ message: 'Hello', status: 'success' });

  // With null/undefined values
  res.json({ data: null, optional: undefined });

  // Arrays
  res.json([{ id: 1 }, { id: 2 }]);

  // JSON with custom settings
  app.set('json spaces', 2);         // Pretty print
  app.set('json replacer', null);    // Custom replacer
  app.set('json escape', true);      // Escape special characters
});
```

### res.status() - Set Status Code

```javascript
app.get('/status-examples', (req, res) => {
  // Chainable status
  res.status(200).json({ message: 'OK' });
  res.status(201).json({ message: 'Created' });
  res.status(204).send();              // No content
  res.status(400).json({ error: 'Bad Request' });
  res.status(401).json({ error: 'Unauthorized' });
  res.status(403).json({ error: 'Forbidden' });
  res.status(404).json({ error: 'Not Found' });
  res.status(500).json({ error: 'Internal Server Error' });
});

// Common status code patterns
app.post('/users', (req, res) => {
  // 201 Created - Resource created successfully
  res.status(201).json({ id: 1, ...req.body });
});

app.delete('/users/:id', (req, res) => {
  // 204 No Content - Successful deletion
  res.status(204).send();
});

app.get('/users/:id', (req, res) => {
  const user = null; // Not found
  if (!user) {
    // 404 Not Found
    return res.status(404).json({ error: 'User not found' });
  }
  res.json(user);
});
```

### res.redirect() - Redirect Response

```javascript
app.get('/redirect-examples', (req, res) => {
  // Default redirect (302 Found)
  res.redirect('/new-location');

  // Permanent redirect (301 Moved Permanently)
  res.redirect(301, '/permanent-location');

  // Temporary redirect (307 Temporary Redirect)
  res.redirect(307, '/temp-location');

  // Redirect to full URL
  res.redirect('https://example.com');

  // Redirect back to referer
  res.redirect('back');

  // Relative redirects
  res.redirect('..');           // Parent directory
  res.redirect('../login');     // Sibling route
});
```

### res.sendFile() - Send Files

```javascript
const path = require('path');

app.get('/files/:filename', (req, res) => {
  const options = {
    root: path.join(__dirname, 'public'),
    dotfiles: 'deny',
    headers: {
      'x-timestamp': Date.now(),
      'x-sent': true
    }
  };

  res.sendFile(req.params.filename, options, (err) => {
    if (err) {
      console.error('Error sending file:', err);
      res.status(err.status).end();
    }
  });
});

// Send file with absolute path
app.get('/download/:file', (req, res) => {
  const filePath = path.join(__dirname, 'files', req.params.file);
  res.sendFile(filePath);
});
```

### res.download() - Download Files

```javascript
app.get('/download/:filename', (req, res) => {
  const filePath = path.join(__dirname, 'files', req.params.filename);

  // Basic download
  res.download(filePath);

  // With custom filename
  res.download(filePath, 'custom-name.pdf');

  // With options and callback
  res.download(filePath, 'report.pdf', {
    headers: {
      'X-Custom-Header': 'value'
    }
  }, (err) => {
    if (err) {
      console.error('Download failed:', err);
    }
  });
});
```

### Response Headers

```javascript
app.get('/header-examples', (req, res) => {
  // Set single header
  res.set('X-Custom-Header', 'custom-value');
  res.setHeader('X-Another-Header', 'another-value');

  // Set multiple headers
  res.set({
    'Content-Type': 'application/json',
    'X-Request-Id': '12345',
    'Cache-Control': 'no-cache'
  });

  // Append to existing header
  res.append('Set-Cookie', 'foo=bar');
  res.append('Set-Cookie', 'baz=qux');

  // Get header value
  const contentType = res.get('Content-Type');

  // Remove header
  res.removeHeader('X-Powered-By');

  // Content type shortcuts
  res.type('json');              // application/json
  res.type('html');              // text/html
  res.type('png');               // image/png
  res.type('application/xml');   // application/xml

  res.json({ headers: 'set' });
});
```

### Response Formatting

```javascript
// res.format() - Content negotiation
app.get('/data', (req, res) => {
  res.format({
    'text/plain': () => {
      res.send('Hello World');
    },
    'text/html': () => {
      res.send('<h1>Hello World</h1>');
    },
    'application/json': () => {
      res.json({ message: 'Hello World' });
    },
    default: () => {
      res.status(406).send('Not Acceptable');
    }
  });
});

// res.links() - Set Link header
app.get('/users', (req, res) => {
  res.links({
    next: '/users?page=2',
    last: '/users?page=10'
  });
  // Link: </users?page=2>; rel="next", </users?page=10>; rel="last"

  res.json({ users: [] });
});

// res.location() - Set Location header
app.post('/users', (req, res) => {
  const userId = 123;
  res.location(`/users/${userId}`);
  res.status(201).json({ id: userId });
});
```

### res.render() - Render Templates

```javascript
app.set('view engine', 'ejs');
app.set('views', './views');

app.get('/page', (req, res) => {
  res.render('page', {
    title: 'My Page',
    user: { name: 'John' }
  });
});

// With callback
app.get('/page-callback', (req, res) => {
  res.render('page', { title: 'Test' }, (err, html) => {
    if (err) {
      return res.status(500).send('Render error');
    }
    res.send(html);
  });
});
```

---

## 6. Route Handlers

Express allows multiple approaches to handling routes.

### Single Handler

```javascript
app.get('/simple', (req, res) => {
  res.json({ message: 'Single handler' });
});
```

### Multiple Handlers (Middleware Chain)

```javascript
// Separate middleware functions
const validateUser = (req, res, next) => {
  if (!req.body.email) {
    return res.status(400).json({ error: 'Email required' });
  }
  next();
};

const checkDuplicates = async (req, res, next) => {
  const exists = await User.findByEmail(req.body.email);
  if (exists) {
    return res.status(409).json({ error: 'User already exists' });
  }
  next();
};

const createUser = async (req, res) => {
  const user = await User.create(req.body);
  res.status(201).json(user);
};

app.post('/users', validateUser, checkDuplicates, createUser);

// Inline multiple handlers
app.get('/data',
  (req, res, next) => {
    console.log('First handler');
    req.customData = { processed: true };
    next();
  },
  (req, res, next) => {
    console.log('Second handler');
    next();
  },
  (req, res) => {
    res.json({ data: req.customData });
  }
);
```

### Array of Handlers

```javascript
const logRequest = (req, res, next) => {
  console.log(`${new Date().toISOString()} - ${req.method} ${req.path}`);
  next();
};

const authenticate = (req, res, next) => {
  const token = req.headers.authorization;
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }
  // Verify token...
  next();
};

const authorize = (roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
};

// Array of middleware
const adminMiddleware = [logRequest, authenticate, authorize(['admin'])];

app.get('/admin/dashboard', adminMiddleware, (req, res) => {
  res.json({ message: 'Admin dashboard' });
});

// Combined array and individual handlers
app.post('/admin/users',
  adminMiddleware,
  validateUser,
  async (req, res) => {
    const user = await User.create(req.body);
    res.status(201).json(user);
  }
);
```

### Conditional Handlers

```javascript
// Skip middleware conditionally
app.use((req, res, next) => {
  if (req.path.startsWith('/public')) {
    return next('route'); // Skip remaining handlers
  }
  next();
});

// Route-specific conditions
app.get('/resource/:id',
  (req, res, next) => {
    if (req.query.skip === 'true') {
      return next('route');
    }
    next();
  },
  (req, res) => {
    res.json({ handler: 'first' });
  }
);

app.get('/resource/:id', (req, res) => {
  res.json({ handler: 'second (skipped to)' });
});
```

### Handler Factories

```javascript
// Create reusable handlers
const createCrudHandlers = (Model) => ({
  getAll: async (req, res) => {
    const items = await Model.find();
    res.json(items);
  },

  getById: async (req, res) => {
    const item = await Model.findById(req.params.id);
    if (!item) {
      return res.status(404).json({ error: 'Not found' });
    }
    res.json(item);
  },

  create: async (req, res) => {
    const item = await Model.create(req.body);
    res.status(201).json(item);
  },

  update: async (req, res) => {
    const item = await Model.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true }
    );
    if (!item) {
      return res.status(404).json({ error: 'Not found' });
    }
    res.json(item);
  },

  delete: async (req, res) => {
    await Model.findByIdAndDelete(req.params.id);
    res.status(204).send();
  }
});

// Usage
const userHandlers = createCrudHandlers(User);
app.get('/users', userHandlers.getAll);
app.get('/users/:id', userHandlers.getById);
app.post('/users', userHandlers.create);
app.put('/users/:id', userHandlers.update);
app.delete('/users/:id', userHandlers.delete);
```

---

## 7. express.Router() for Modular Routes

The `express.Router` class creates modular, mountable route handlers.

### Basic Router Usage

```javascript
// routes/users.js
const express = require('express');
const router = express.Router();

// Middleware specific to this router
router.use((req, res, next) => {
  console.log('Users router middleware');
  next();
});

// Define routes
router.get('/', async (req, res) => {
  const users = await User.find();
  res.json(users);
});

router.get('/:id', async (req, res) => {
  const user = await User.findById(req.params.id);
  if (!user) {
    return res.status(404).json({ error: 'User not found' });
  }
  res.json(user);
});

router.post('/', async (req, res) => {
  const user = await User.create(req.body);
  res.status(201).json(user);
});

router.put('/:id', async (req, res) => {
  const user = await User.findByIdAndUpdate(req.params.id, req.body, { new: true });
  res.json(user);
});

router.delete('/:id', async (req, res) => {
  await User.findByIdAndDelete(req.params.id);
  res.status(204).send();
});

module.exports = router;

// app.js
const usersRouter = require('./routes/users');
app.use('/api/users', usersRouter);
```

### Router Options

```javascript
const router = express.Router({
  caseSensitive: true,    // Treat /Foo and /foo differently
  mergeParams: true,      // Merge parent params into child
  strict: true            // Treat /foo and /foo/ differently
});
```

### Nested Routers

```javascript
// routes/users.js
const express = require('express');
const router = express.Router();
const postsRouter = require('./posts');

// Mount nested router
router.use('/:userId/posts', postsRouter);

router.get('/', (req, res) => {
  res.json({ users: [] });
});

module.exports = router;

// routes/posts.js
const express = require('express');
const router = express.Router({ mergeParams: true }); // Access parent params

router.get('/', (req, res) => {
  // Access userId from parent router
  const { userId } = req.params;
  res.json({ userId, posts: [] });
});

router.get('/:postId', (req, res) => {
  const { userId, postId } = req.params;
  res.json({ userId, postId });
});

module.exports = router;

// app.js
app.use('/api/users', usersRouter);
// Results in: /api/users/:userId/posts/:postId
```

### Router Middleware

```javascript
const router = express.Router();

// Apply middleware to all routes in router
router.use(authenticate);

// Apply middleware to specific path
router.use('/admin', adminOnly);

// Apply middleware to specific methods
router.get('/', publicHandler);
router.post('/', authenticate, createHandler);

// Route-specific middleware chain
router.route('/profile')
  .all(authenticate)
  .get(getProfile)
  .put(validateProfile, updateProfile);
```

### Router Parameter Middleware

```javascript
const router = express.Router();

// Process parameter before route handlers
router.param('userId', async (req, res, next, id) => {
  try {
    const user = await User.findById(id);
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    req.user = user;
    next();
  } catch (err) {
    next(err);
  }
});

// Routes automatically have user attached
router.get('/:userId', (req, res) => {
  res.json(req.user);
});

router.put('/:userId', (req, res) => {
  // req.user is already loaded
  Object.assign(req.user, req.body);
  req.user.save();
  res.json(req.user);
});
```

### Complete Router Example

```javascript
// routes/api/v1/products.js
const express = require('express');
const router = express.Router();

// Router-level middleware
router.use((req, res, next) => {
  req.requestTime = Date.now();
  next();
});

// Parameter preprocessing
router.param('productId', async (req, res, next, id) => {
  if (!isValidObjectId(id)) {
    return res.status(400).json({ error: 'Invalid product ID' });
  }

  const product = await Product.findById(id);
  if (!product) {
    return res.status(404).json({ error: 'Product not found' });
  }

  req.product = product;
  next();
});

// Routes
router.route('/')
  .get(async (req, res) => {
    const { page = 1, limit = 20, category, sort } = req.query;

    const query = category ? { category } : {};
    const options = {
      skip: (page - 1) * limit,
      limit: parseInt(limit),
      sort: sort ? { [sort]: 1 } : { createdAt: -1 }
    };

    const [products, total] = await Promise.all([
      Product.find(query, null, options),
      Product.countDocuments(query)
    ]);

    res.json({
      products,
      pagination: {
        page: parseInt(page),
        limit: parseInt(limit),
        total,
        pages: Math.ceil(total / limit)
      }
    });
  })
  .post(authenticate, authorize('admin'), async (req, res) => {
    const product = await Product.create(req.body);
    res.status(201).json(product);
  });

router.route('/:productId')
  .get((req, res) => {
    res.json(req.product);
  })
  .put(authenticate, authorize('admin'), async (req, res) => {
    Object.assign(req.product, req.body);
    await req.product.save();
    res.json(req.product);
  })
  .delete(authenticate, authorize('admin'), async (req, res) => {
    await req.product.remove();
    res.status(204).send();
  });

// Nested routes
router.get('/:productId/reviews', async (req, res) => {
  const reviews = await Review.find({ product: req.product._id });
  res.json(reviews);
});

module.exports = router;
```

---

## 8. Static Files

Express provides the `express.static` built-in middleware function to serve static files.

### Basic Static File Serving

```javascript
// Serve files from 'public' directory
app.use(express.static('public'));

// Files are accessible at:
// public/style.css -> http://localhost:3000/style.css
// public/js/app.js -> http://localhost:3000/js/app.js
```

### Virtual Path Prefix

```javascript
// Mount with virtual path
app.use('/static', express.static('public'));

// Files are accessible at:
// public/style.css -> http://localhost:3000/static/style.css
// public/js/app.js -> http://localhost:3000/static/js/app.js
```

### Absolute Path

```javascript
const path = require('path');

// Use absolute path for reliability
app.use('/static', express.static(path.join(__dirname, 'public')));
```

### Multiple Static Directories

```javascript
// Express searches in order
app.use(express.static('public'));
app.use(express.static('uploads'));
app.use(express.static('assets'));

// With different mount points
app.use('/static', express.static('public'));
app.use('/uploads', express.static('uploads'));
app.use('/assets', express.static('node_modules'));
```

### Static Options

```javascript
const options = {
  dotfiles: 'ignore',           // 'allow', 'deny', 'ignore'
  etag: true,                   // Enable ETag generation
  extensions: ['html', 'htm'],  // Extension fallbacks
  fallthrough: true,            // Pass to next middleware if not found
  immutable: true,              // Add immutable directive to Cache-Control
  index: 'index.html',          // Index file name (false to disable)
  lastModified: true,           // Set Last-Modified header
  maxAge: '1d',                 // Cache max-age (ms or string)
  redirect: true,               // Redirect to trailing / for directories
  setHeaders: (res, path, stat) => {
    res.set('x-timestamp', Date.now());
  }
};

app.use(express.static('public', options));
```

### Caching Strategies

```javascript
// Development - no caching
if (process.env.NODE_ENV === 'development') {
  app.use(express.static('public', { maxAge: 0 }));
}

// Production - aggressive caching
if (process.env.NODE_ENV === 'production') {
  // Versioned assets (long cache)
  app.use('/assets', express.static('dist/assets', {
    maxAge: '1y',
    immutable: true
  }));

  // HTML files (short cache)
  app.use(express.static('dist', {
    maxAge: '1h',
    setHeaders: (res, path) => {
      if (path.endsWith('.html')) {
        res.setHeader('Cache-Control', 'no-cache');
      }
    }
  }));
}
```

### SPA (Single Page Application) Fallback

```javascript
const path = require('path');

// Serve static files
app.use(express.static(path.join(__dirname, 'client/build')));

// API routes
app.use('/api', apiRouter);

// SPA fallback - serve index.html for all other routes
app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname, 'client/build', 'index.html'));
});
```

---

## 9. Body Parsing

Express provides built-in middleware for parsing request bodies.

### express.json()

Parses incoming requests with JSON payloads.

```javascript
// Basic usage
app.use(express.json());

// With options
app.use(express.json({
  inflate: true,              // Handle compressed bodies
  limit: '100kb',             // Maximum body size
  reviver: null,              // JSON.parse reviver function
  strict: true,               // Only accept arrays and objects
  type: 'application/json',   // Media type to parse
  verify: (req, res, buf, encoding) => {
    // Custom verification
    if (buf.includes('forbidden')) {
      throw new Error('Forbidden content');
    }
  }
}));

// Parse only for specific routes
app.post('/api/data', express.json({ limit: '10mb' }), (req, res) => {
  res.json(req.body);
});
```

### express.urlencoded()

Parses incoming requests with URL-encoded payloads.

```javascript
// Basic usage
app.use(express.urlencoded({ extended: true }));

// With options
app.use(express.urlencoded({
  extended: true,             // Use qs library (supports nested objects)
  inflate: true,              // Handle compressed bodies
  limit: '100kb',             // Maximum body size
  parameterLimit: 1000,       // Maximum parameters
  type: 'application/x-www-form-urlencoded',
  verify: undefined           // Custom verification function
}));

// extended: false uses querystring library (flat structure)
// extended: true uses qs library (nested objects)
```

### express.raw()

Parses incoming requests into a Buffer.

```javascript
// Parse all as raw
app.use(express.raw({ type: '*/*' }));

// Parse specific content types
app.use('/webhook', express.raw({ type: 'application/octet-stream' }));

// With options
app.use(express.raw({
  inflate: true,
  limit: '5mb',
  type: 'application/octet-stream'
}));

// Usage
app.post('/upload', express.raw({ limit: '50mb' }), (req, res) => {
  const buffer = req.body; // Buffer
  // Process binary data
  res.json({ size: buffer.length });
});
```

### express.text()

Parses incoming requests into a string.

```javascript
// Parse text/plain
app.use(express.text());

// With options
app.use(express.text({
  defaultCharset: 'utf-8',
  inflate: true,
  limit: '100kb',
  type: 'text/plain'
}));

// Usage
app.post('/log', express.text(), (req, res) => {
  const text = req.body; // String
  console.log('Received:', text);
  res.send('OK');
});
```

### Multipart Form Data (File Uploads)

For multipart form data, use a library like `multer`:

```javascript
const multer = require('multer');

// Memory storage
const upload = multer({ storage: multer.memoryStorage() });

// Disk storage
const diskStorage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, 'uploads/');
  },
  filename: (req, file, cb) => {
    const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1E9);
    cb(null, file.fieldname + '-' + uniqueSuffix + path.extname(file.originalname));
  }
});

const uploadDisk = multer({
  storage: diskStorage,
  limits: {
    fileSize: 5 * 1024 * 1024, // 5MB
    files: 5
  },
  fileFilter: (req, file, cb) => {
    const allowedTypes = ['image/jpeg', 'image/png', 'image/gif'];
    if (allowedTypes.includes(file.mimetype)) {
      cb(null, true);
    } else {
      cb(new Error('Invalid file type'));
    }
  }
});

// Single file
app.post('/upload', uploadDisk.single('avatar'), (req, res) => {
  res.json({ file: req.file });
});

// Multiple files (same field)
app.post('/gallery', uploadDisk.array('photos', 10), (req, res) => {
  res.json({ files: req.files });
});

// Multiple fields
app.post('/profile', uploadDisk.fields([
  { name: 'avatar', maxCount: 1 },
  { name: 'documents', maxCount: 5 }
]), (req, res) => {
  res.json({
    avatar: req.files['avatar'],
    documents: req.files['documents']
  });
});
```

### Body Parser Configuration Examples

```javascript
// API server configuration
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true, limit: '10mb' }));

// Webhook endpoint with raw body
app.post('/webhook/stripe',
  express.raw({ type: 'application/json' }),
  (req, res) => {
    const sig = req.headers['stripe-signature'];
    const event = stripe.webhooks.constructEvent(req.body, sig, webhookSecret);
    res.json({ received: true });
  }
);

// Mixed body types
app.use('/api', express.json());
app.use('/form', express.urlencoded({ extended: true }));
app.use('/raw', express.raw({ type: '*/*' }));
```

---

## 10. Error Handling

Error handling in Express is done through middleware functions with four arguments.

### Basic Error Handler

```javascript
// Error handling middleware (must have 4 parameters)
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: 'Something went wrong!' });
});
```

### Custom Error Classes

```javascript
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
    this.status = `${statusCode}`.startsWith('4') ? 'fail' : 'error';
    this.isOperational = true;

    Error.captureStackTrace(this, this.constructor);
  }
}

class NotFoundError extends AppError {
  constructor(message = 'Resource not found') {
    super(message, 404);
  }
}

class ValidationError extends AppError {
  constructor(message = 'Validation failed', errors = []) {
    super(message, 400);
    this.errors = errors;
  }
}

class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(message, 401);
  }
}

class ForbiddenError extends AppError {
  constructor(message = 'Forbidden') {
    super(message, 403);
  }
}
```

### Async Error Handling

```javascript
// Async handler wrapper
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// Usage with async/await
app.get('/users/:id', asyncHandler(async (req, res) => {
  const user = await User.findById(req.params.id);
  if (!user) {
    throw new NotFoundError('User not found');
  }
  res.json(user);
}));

// Alternative: express-async-handler package
const asyncHandlerPkg = require('express-async-handler');

app.get('/posts', asyncHandlerPkg(async (req, res) => {
  const posts = await Post.find();
  res.json(posts);
}));
```

### Comprehensive Error Handler

```javascript
// Development error handler
const developmentErrorHandler = (err, req, res, next) => {
  console.error('Error:', err);

  res.status(err.statusCode || 500).json({
    status: err.status || 'error',
    message: err.message,
    stack: err.stack,
    error: err
  });
};

// Production error handler
const productionErrorHandler = (err, req, res, next) => {
  // Operational, trusted error
  if (err.isOperational) {
    return res.status(err.statusCode).json({
      status: err.status,
      message: err.message,
      ...(err.errors && { errors: err.errors })
    });
  }

  // Programming or unknown error
  console.error('ERROR:', err);

  res.status(500).json({
    status: 'error',
    message: 'Something went wrong'
  });
};

// Error handler selector
app.use((err, req, res, next) => {
  err.statusCode = err.statusCode || 500;
  err.status = err.status || 'error';

  if (process.env.NODE_ENV === 'development') {
    developmentErrorHandler(err, req, res, next);
  } else {
    productionErrorHandler(err, req, res, next);
  }
});
```

### Handling Specific Errors

```javascript
// Handle specific error types
const errorHandler = (err, req, res, next) => {
  // Mongoose validation error
  if (err.name === 'ValidationError') {
    const errors = Object.values(err.errors).map(e => e.message);
    return res.status(400).json({
      status: 'fail',
      message: 'Validation failed',
      errors
    });
  }

  // Mongoose cast error (invalid ObjectId)
  if (err.name === 'CastError') {
    return res.status(400).json({
      status: 'fail',
      message: `Invalid ${err.path}: ${err.value}`
    });
  }

  // Mongoose duplicate key error
  if (err.code === 11000) {
    const field = Object.keys(err.keyValue)[0];
    return res.status(409).json({
      status: 'fail',
      message: `Duplicate field value: ${field}`
    });
  }

  // JWT errors
  if (err.name === 'JsonWebTokenError') {
    return res.status(401).json({
      status: 'fail',
      message: 'Invalid token'
    });
  }

  if (err.name === 'TokenExpiredError') {
    return res.status(401).json({
      status: 'fail',
      message: 'Token expired'
    });
  }

  // Default error
  res.status(err.statusCode || 500).json({
    status: err.status || 'error',
    message: err.message || 'Internal server error'
  });
};

app.use(errorHandler);
```

### 404 Handler

```javascript
// 404 handler (place before error handler)
app.use((req, res, next) => {
  next(new NotFoundError(`Cannot ${req.method} ${req.originalUrl}`));
});

// Or without custom error class
app.use((req, res) => {
  res.status(404).json({
    status: 'fail',
    message: `Route ${req.originalUrl} not found`
  });
});
```

### Unhandled Rejections and Exceptions

```javascript
// Handle unhandled promise rejections
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason);
  // Application specific logging
  // Optionally exit process
  // server.close(() => process.exit(1));
});

// Handle uncaught exceptions
process.on('uncaughtException', (err) => {
  console.error('Uncaught Exception:', err);
  // Exit process as state may be corrupted
  process.exit(1);
});
```

---

## 11. Template Engines

Express supports various template engines for rendering dynamic HTML.

### Setting Up Template Engine

```javascript
// EJS
app.set('view engine', 'ejs');
app.set('views', './views');

// Pug (Jade)
app.set('view engine', 'pug');

// Handlebars
const exphbs = require('express-handlebars');
app.engine('handlebars', exphbs.engine());
app.set('view engine', 'handlebars');

// Nunjucks
const nunjucks = require('nunjucks');
nunjucks.configure('views', {
  autoescape: true,
  express: app
});
```

### Rendering Views

```javascript
// Basic render
app.get('/', (req, res) => {
  res.render('index', { title: 'Home Page' });
});

// With data
app.get('/profile', (req, res) => {
  res.render('profile', {
    user: {
      name: 'John Doe',
      email: 'john@example.com'
    },
    posts: [
      { title: 'Post 1', content: 'Content 1' },
      { title: 'Post 2', content: 'Content 2' }
    ]
  });
});

// Render with callback
app.get('/email', (req, res) => {
  res.render('email-template', { data: {} }, (err, html) => {
    if (err) {
      return res.status(500).send('Render error');
    }
    // Use HTML (e.g., send as email)
    sendEmail(html);
    res.send('Email sent');
  });
});
```

### EJS Examples

```ejs
<!-- views/layout.ejs -->
<!DOCTYPE html>
<html>
<head>
  <title><%= title %></title>
</head>
<body>
  <%- include('partials/header') %>

  <main>
    <%- body %>
  </main>

  <%- include('partials/footer') %>
</body>
</html>

<!-- views/index.ejs -->
<h1><%= title %></h1>

<!-- Unescaped output -->
<%- htmlContent %>

<!-- Control flow -->
<% if (user) { %>
  <p>Welcome, <%= user.name %></p>
<% } else { %>
  <p>Please log in</p>
<% } %>

<!-- Loops -->
<ul>
<% posts.forEach(post => { %>
  <li><%= post.title %></li>
<% }); %>
</ul>

<!-- Include partials -->
<%- include('partials/sidebar', { items: menuItems }) %>
```

### Pug Examples

```pug
//- views/layout.pug
doctype html
html
  head
    title= title
  body
    include partials/header

    main
      block content

    include partials/footer

//- views/index.pug
extends layout

block content
  h1= title

  if user
    p Welcome, #{user.name}
  else
    p Please log in

  ul
    each post in posts
      li= post.title
```

### Handlebars Examples

```handlebars
<!-- views/layouts/main.handlebars -->
<!DOCTYPE html>
<html>
<head>
  <title>{{title}}</title>
</head>
<body>
  {{> header}}

  <main>
    {{{body}}}
  </main>

  {{> footer}}
</body>
</html>

<!-- views/index.handlebars -->
<h1>{{title}}</h1>

{{#if user}}
  <p>Welcome, {{user.name}}</p>
{{else}}
  <p>Please log in</p>
{{/if}}

<ul>
{{#each posts}}
  <li>{{this.title}}</li>
{{/each}}
</ul>
```

### Custom Template Engine

```javascript
const fs = require('fs');

app.engine('ntl', (filePath, options, callback) => {
  fs.readFile(filePath, (err, content) => {
    if (err) return callback(err);

    // Simple template replacement
    const rendered = content.toString()
      .replace(/\{\{(\w+)\}\}/g, (match, key) => {
        return options[key] || '';
      });

    return callback(null, rendered);
  });
});

app.set('views', './views');
app.set('view engine', 'ntl');
```

---

## 12. Cookies and Sessions

### Cookies with cookie-parser

```javascript
const cookieParser = require('cookie-parser');

// Parse cookies
app.use(cookieParser());

// With signed cookies
app.use(cookieParser('your-secret-key'));

// Setting cookies
app.get('/set-cookie', (req, res) => {
  // Basic cookie
  res.cookie('name', 'value');

  // Cookie with options
  res.cookie('user', 'john', {
    maxAge: 900000,           // Expires in 15 minutes (ms)
    httpOnly: true,           // Not accessible via JavaScript
    secure: true,             // Only over HTTPS
    sameSite: 'strict',       // CSRF protection
    path: '/',                // Cookie path
    domain: '.example.com'    // Cookie domain
  });

  // Signed cookie
  res.cookie('session', 'data', { signed: true });

  res.send('Cookies set');
});

// Reading cookies
app.get('/get-cookie', (req, res) => {
  const name = req.cookies.name;
  const signedSession = req.signedCookies.session;

  res.json({ name, signedSession });
});

// Clearing cookies
app.get('/clear-cookie', (req, res) => {
  res.clearCookie('name');
  res.clearCookie('user', { path: '/', domain: '.example.com' });
  res.send('Cookies cleared');
});
```

### Sessions with express-session

```javascript
const session = require('express-session');

app.use(session({
  secret: 'your-secret-key',
  resave: false,
  saveUninitialized: false,
  name: 'sessionId',              // Cookie name (default: connect.sid)
  cookie: {
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
    maxAge: 24 * 60 * 60 * 1000,  // 24 hours
    sameSite: 'lax'
  }
}));

// Using sessions
app.post('/login', (req, res) => {
  // Authenticate user...
  req.session.userId = user.id;
  req.session.isAuthenticated = true;
  res.json({ message: 'Logged in' });
});

app.get('/profile', (req, res) => {
  if (!req.session.isAuthenticated) {
    return res.status(401).json({ error: 'Not authenticated' });
  }
  res.json({ userId: req.session.userId });
});

app.post('/logout', (req, res) => {
  req.session.destroy((err) => {
    if (err) {
      return res.status(500).json({ error: 'Logout failed' });
    }
    res.clearCookie('sessionId');
    res.json({ message: 'Logged out' });
  });
});

// Regenerate session (after login for security)
app.post('/login', (req, res) => {
  // Authenticate...
  req.session.regenerate((err) => {
    if (err) return next(err);
    req.session.userId = user.id;
    res.json({ message: 'Logged in' });
  });
});
```

### Session Stores

```javascript
// Redis store
const RedisStore = require('connect-redis').default;
const { createClient } = require('redis');

const redisClient = createClient({ url: 'redis://localhost:6379' });
redisClient.connect();

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: 'your-secret',
  resave: false,
  saveUninitialized: false
}));

// MongoDB store
const MongoStore = require('connect-mongo');

app.use(session({
  store: MongoStore.create({
    mongoUrl: 'mongodb://localhost/sessions',
    ttl: 24 * 60 * 60 // 1 day
  }),
  secret: 'your-secret',
  resave: false,
  saveUninitialized: false
}));

// PostgreSQL store
const pgSession = require('connect-pg-simple')(session);
const { Pool } = require('pg');

const pool = new Pool({
  connectionString: process.env.DATABASE_URL
});

app.use(session({
  store: new pgSession({
    pool: pool,
    tableName: 'session'
  }),
  secret: 'your-secret',
  resave: false,
  saveUninitialized: false
}));
```

---

## 13. CORS Configuration

Cross-Origin Resource Sharing (CORS) configuration using the cors middleware.

### Basic CORS Setup

```javascript
const cors = require('cors');

// Enable all CORS requests
app.use(cors());

// Enable for specific origins
app.use(cors({
  origin: 'http://localhost:3000'
}));
```

### Advanced CORS Configuration

```javascript
const corsOptions = {
  // Origin configuration
  origin: [
    'http://localhost:3000',
    'https://example.com',
    /\.example\.com$/       // Regex for subdomains
  ],

  // Dynamic origin
  origin: (origin, callback) => {
    const allowedOrigins = ['http://localhost:3000', 'https://example.com'];

    // Allow requests with no origin (mobile apps, curl, etc.)
    if (!origin) return callback(null, true);

    if (allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },

  // Allowed methods
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],

  // Allowed headers
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Requested-With'],

  // Exposed headers (accessible to client)
  exposedHeaders: ['X-Total-Count', 'X-Page-Count'],

  // Allow credentials (cookies, authorization headers)
  credentials: true,

  // Preflight cache duration (seconds)
  maxAge: 86400,

  // Pass preflight response to next handler
  preflightContinue: false,

  // Status code for successful OPTIONS requests
  optionsSuccessStatus: 204
};

app.use(cors(corsOptions));
```

### Route-Specific CORS

```javascript
// Enable CORS for specific route
app.get('/api/public', cors(), (req, res) => {
  res.json({ data: 'public' });
});

// Different CORS options per route
const publicCors = cors({ origin: '*' });
const privateCors = cors({
  origin: 'https://admin.example.com',
  credentials: true
});

app.get('/api/public', publicCors, publicHandler);
app.get('/api/admin', privateCors, adminHandler);
```

### Manual CORS Implementation

```javascript
// Without cors package
app.use((req, res, next) => {
  const allowedOrigins = ['http://localhost:3000', 'https://example.com'];
  const origin = req.headers.origin;

  if (allowedOrigins.includes(origin)) {
    res.setHeader('Access-Control-Allow-Origin', origin);
  }

  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
  res.setHeader('Access-Control-Allow-Credentials', 'true');
  res.setHeader('Access-Control-Max-Age', '86400');

  // Handle preflight
  if (req.method === 'OPTIONS') {
    return res.status(204).end();
  }

  next();
});
```

---

## 14. Security Best Practices

### Using Helmet

```javascript
const helmet = require('helmet');

// Use all default protections
app.use(helmet());

// Configure specific protections
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", 'data:', 'https:'],
      connectSrc: ["'self'", 'https://api.example.com'],
      fontSrc: ["'self'", 'https://fonts.gstatic.com'],
      objectSrc: ["'none'"],
      upgradeInsecureRequests: []
    }
  },
  crossOriginEmbedderPolicy: true,
  crossOriginOpenerPolicy: true,
  crossOriginResourcePolicy: { policy: 'same-site' },
  dnsPrefetchControl: { allow: false },
  frameguard: { action: 'deny' },
  hidePoweredBy: true,
  hsts: { maxAge: 31536000, includeSubDomains: true, preload: true },
  ieNoOpen: true,
  noSniff: true,
  originAgentCluster: true,
  permittedCrossDomainPolicies: { permittedPolicies: 'none' },
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
  xssFilter: true
}));
```

### Rate Limiting

```javascript
const rateLimit = require('express-rate-limit');

// General rate limiter
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,   // 15 minutes
  max: 100,                    // Limit each IP to 100 requests per window
  message: { error: 'Too many requests, please try again later' },
  standardHeaders: true,       // Return rate limit info in headers
  legacyHeaders: false
});

app.use(limiter);

// Stricter limiter for auth routes
const authLimiter = rateLimit({
  windowMs: 60 * 60 * 1000,   // 1 hour
  max: 5,                      // 5 attempts per hour
  message: { error: 'Too many login attempts' },
  skipSuccessfulRequests: true
});

app.use('/api/auth', authLimiter);

// API rate limiter with custom key
const apiLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: 60,
  keyGenerator: (req) => {
    return req.headers['x-api-key'] || req.ip;
  }
});
```

### Input Validation

```javascript
const { body, param, query, validationResult } = require('express-validator');

// Validation middleware
const validate = (validations) => {
  return async (req, res, next) => {
    await Promise.all(validations.map(validation => validation.run(req)));

    const errors = validationResult(req);
    if (errors.isEmpty()) {
      return next();
    }

    res.status(400).json({
      errors: errors.array().map(err => ({
        field: err.path,
        message: err.msg
      }))
    });
  };
};

// Usage
app.post('/users',
  validate([
    body('email')
      .isEmail()
      .normalizeEmail()
      .withMessage('Valid email required'),
    body('password')
      .isLength({ min: 8 })
      .matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
      .withMessage('Password must be 8+ chars with uppercase, lowercase, and number'),
    body('name')
      .trim()
      .escape()
      .isLength({ min: 2, max: 50 })
      .withMessage('Name must be 2-50 characters')
  ]),
  createUser
);
```

### SQL Injection Prevention

```javascript
// Use parameterized queries
const mysql = require('mysql2/promise');

// BAD - vulnerable to SQL injection
const badQuery = `SELECT * FROM users WHERE id = ${req.params.id}`;

// GOOD - parameterized query
const [rows] = await connection.execute(
  'SELECT * FROM users WHERE id = ?',
  [req.params.id]
);

// With ORM (Sequelize, Prisma, etc.)
const user = await User.findByPk(req.params.id);
```

### Additional Security Measures

```javascript
// Disable x-powered-by header
app.disable('x-powered-by');

// Trust proxy (behind reverse proxy)
app.set('trust proxy', 1);

// Secure cookies
app.use(session({
  cookie: {
    secure: true,
    httpOnly: true,
    sameSite: 'strict'
  }
}));

// Request size limits
app.use(express.json({ limit: '10kb' }));
app.use(express.urlencoded({ limit: '10kb', extended: true }));

// HPP - HTTP Parameter Pollution protection
const hpp = require('hpp');
app.use(hpp());

// Mongo sanitize (prevent NoSQL injection)
const mongoSanitize = require('express-mongo-sanitize');
app.use(mongoSanitize());

// XSS clean
const xss = require('xss-clean');
app.use(xss());
```

---

## 15. Project Structure Patterns

### Basic Structure

```
project/
├── src/
│   ├── app.js              # Express app setup
│   ├── server.js           # Server startup
│   ├── routes/
│   │   ├── index.js        # Route aggregator
│   │   ├── users.js
│   │   └── products.js
│   ├── controllers/
│   │   ├── userController.js
│   │   └── productController.js
│   ├── middleware/
│   │   ├── auth.js
│   │   ├── validate.js
│   │   └── errorHandler.js
│   ├── models/
│   │   ├── User.js
│   │   └── Product.js
│   └── utils/
│       ├── helpers.js
│       └── constants.js
├── config/
│   └── database.js
├── tests/
├── .env
└── package.json
```

### Feature-Based Structure

```
project/
├── src/
│   ├── app.js
│   ├── server.js
│   ├── features/
│   │   ├── users/
│   │   │   ├── user.routes.js
│   │   │   ├── user.controller.js
│   │   │   ├── user.service.js
│   │   │   ├── user.model.js
│   │   │   ├── user.validation.js
│   │   │   └── user.test.js
│   │   ├── products/
│   │   │   ├── product.routes.js
│   │   │   ├── product.controller.js
│   │   │   ├── product.service.js
│   │   │   └── product.model.js
│   │   └── auth/
│   │       ├── auth.routes.js
│   │       ├── auth.controller.js
│   │       └── auth.service.js
│   ├── shared/
│   │   ├── middleware/
│   │   ├── utils/
│   │   └── errors/
│   └── config/
├── tests/
└── package.json
```

### Implementation Examples

```javascript
// src/app.js
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const morgan = require('morgan');
const routes = require('./routes');
const errorHandler = require('./middleware/errorHandler');

const app = express();

// Middleware
app.use(helmet());
app.use(cors());
app.use(morgan('dev'));
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Routes
app.use('/api', routes);

// Error handling
app.use(errorHandler);

module.exports = app;

// src/server.js
const app = require('./app');
const config = require('./config');
const connectDB = require('./config/database');

const startServer = async () => {
  await connectDB();

  const server = app.listen(config.port, () => {
    console.log(`Server running on port ${config.port}`);
  });

  // Graceful shutdown
  process.on('SIGTERM', () => {
    server.close(() => {
      console.log('Server closed');
      process.exit(0);
    });
  });
};

startServer();

// src/routes/index.js
const express = require('express');
const userRoutes = require('./users');
const productRoutes = require('./products');
const authRoutes = require('./auth');

const router = express.Router();

router.use('/auth', authRoutes);
router.use('/users', userRoutes);
router.use('/products', productRoutes);

module.exports = router;

// src/controllers/userController.js
const userService = require('../services/userService');
const asyncHandler = require('../utils/asyncHandler');

exports.getUsers = asyncHandler(async (req, res) => {
  const users = await userService.findAll(req.query);
  res.json(users);
});

exports.getUserById = asyncHandler(async (req, res) => {
  const user = await userService.findById(req.params.id);
  res.json(user);
});

exports.createUser = asyncHandler(async (req, res) => {
  const user = await userService.create(req.body);
  res.status(201).json(user);
});

// src/services/userService.js
const User = require('../models/User');
const { NotFoundError } = require('../utils/errors');

exports.findAll = async (query) => {
  const { page = 1, limit = 20 } = query;
  return User.find()
    .skip((page - 1) * limit)
    .limit(limit);
};

exports.findById = async (id) => {
  const user = await User.findById(id);
  if (!user) {
    throw new NotFoundError('User not found');
  }
  return user;
};

exports.create = async (data) => {
  return User.create(data);
};
```

---

## 16. TypeScript with Express

### Setup

```bash
npm install typescript @types/node @types/express ts-node-dev
npx tsc --init
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### Basic Express Application

```typescript
// src/app.ts
import express, { Express, Request, Response, NextFunction } from 'express';
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';

const app: Express = express();

app.use(helmet());
app.use(cors());
app.use(morgan('dev'));
app.use(express.json());

export default app;
```

### Custom Type Definitions

```typescript
// src/types/index.ts
import { Request } from 'express';

export interface User {
  id: string;
  email: string;
  name: string;
  role: 'user' | 'admin';
}

export interface AuthenticatedRequest extends Request {
  user?: User;
}

export interface PaginationQuery {
  page?: string;
  limit?: string;
  sort?: string;
  order?: 'asc' | 'desc';
}

export interface ApiResponse<T> {
  success: boolean;
  data?: T;
  error?: string;
  pagination?: {
    page: number;
    limit: number;
    total: number;
    pages: number;
  };
}
```

### Typed Controllers

```typescript
// src/controllers/userController.ts
import { Request, Response, NextFunction } from 'express';
import { User, AuthenticatedRequest, ApiResponse } from '../types';
import * as userService from '../services/userService';

export const getUsers = async (
  req: Request,
  res: Response<ApiResponse<User[]>>,
  next: NextFunction
): Promise<void> => {
  try {
    const users = await userService.findAll();
    res.json({ success: true, data: users });
  } catch (error) {
    next(error);
  }
};

export const getUserById = async (
  req: Request<{ id: string }>,
  res: Response<ApiResponse<User>>,
  next: NextFunction
): Promise<void> => {
  try {
    const user = await userService.findById(req.params.id);
    res.json({ success: true, data: user });
  } catch (error) {
    next(error);
  }
};

export const createUser = async (
  req: Request<{}, {}, Omit<User, 'id'>>,
  res: Response<ApiResponse<User>>,
  next: NextFunction
): Promise<void> => {
  try {
    const user = await userService.create(req.body);
    res.status(201).json({ success: true, data: user });
  } catch (error) {
    next(error);
  }
};
```

### Typed Middleware

```typescript
// src/middleware/auth.ts
import { Response, NextFunction } from 'express';
import { AuthenticatedRequest, User } from '../types';
import jwt from 'jsonwebtoken';

export const authenticate = (
  req: AuthenticatedRequest,
  res: Response,
  next: NextFunction
): void => {
  const token = req.headers.authorization?.split(' ')[1];

  if (!token) {
    res.status(401).json({ error: 'No token provided' });
    return;
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!) as User;
    req.user = decoded;
    next();
  } catch {
    res.status(401).json({ error: 'Invalid token' });
  }
};

export const authorize = (...roles: User['role'][]) => {
  return (req: AuthenticatedRequest, res: Response, next: NextFunction): void => {
    if (!req.user || !roles.includes(req.user.role)) {
      res.status(403).json({ error: 'Forbidden' });
      return;
    }
    next();
  };
};
```

### Typed Router

```typescript
// src/routes/users.ts
import { Router } from 'express';
import * as userController from '../controllers/userController';
import { authenticate, authorize } from '../middleware/auth';
import { validateUser } from '../middleware/validation';

const router = Router();

router.get('/', userController.getUsers);
router.get('/:id', userController.getUserById);
router.post('/', authenticate, validateUser, userController.createUser);
router.put('/:id', authenticate, validateUser, userController.updateUser);
router.delete('/:id', authenticate, authorize('admin'), userController.deleteUser);

export default router;
```

### Custom Error Classes

```typescript
// src/utils/errors.ts
export class AppError extends Error {
  statusCode: number;
  status: string;
  isOperational: boolean;

  constructor(message: string, statusCode: number) {
    super(message);
    this.statusCode = statusCode;
    this.status = `${statusCode}`.startsWith('4') ? 'fail' : 'error';
    this.isOperational = true;

    Error.captureStackTrace(this, this.constructor);
  }
}

export class NotFoundError extends AppError {
  constructor(message = 'Resource not found') {
    super(message, 404);
  }
}

export class ValidationError extends AppError {
  errors: { field: string; message: string }[];

  constructor(message = 'Validation failed', errors: { field: string; message: string }[] = []) {
    super(message, 400);
    this.errors = errors;
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(message, 401);
  }
}
```

### Typed Error Handler

```typescript
// src/middleware/errorHandler.ts
import { Request, Response, NextFunction } from 'express';
import { AppError, ValidationError } from '../utils/errors';

interface ErrorResponse {
  status: string;
  message: string;
  errors?: { field: string; message: string }[];
  stack?: string;
}

export const errorHandler = (
  err: Error | AppError,
  req: Request,
  res: Response<ErrorResponse>,
  next: NextFunction
): void => {
  if (err instanceof AppError) {
    const response: ErrorResponse = {
      status: err.status,
      message: err.message
    };

    if (err instanceof ValidationError) {
      response.errors = err.errors;
    }

    if (process.env.NODE_ENV === 'development') {
      response.stack = err.stack;
    }

    res.status(err.statusCode).json(response);
    return;
  }

  // Unknown error
  console.error('ERROR:', err);
  res.status(500).json({
    status: 'error',
    message: 'Internal server error'
  });
};
```

### Async Handler Utility

```typescript
// src/utils/asyncHandler.ts
import { Request, Response, NextFunction, RequestHandler } from 'express';

type AsyncRequestHandler<P = {}, ResBody = any, ReqBody = any, ReqQuery = {}> = (
  req: Request<P, ResBody, ReqBody, ReqQuery>,
  res: Response<ResBody>,
  next: NextFunction
) => Promise<void>;

export const asyncHandler = <P = {}, ResBody = any, ReqBody = any, ReqQuery = {}>(
  fn: AsyncRequestHandler<P, ResBody, ReqBody, ReqQuery>
): RequestHandler<P, ResBody, ReqBody, ReqQuery> => {
  return (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
};
```

### Complete TypeScript Application

```typescript
// src/app.ts
import express, { Express } from 'express';
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';
import compression from 'compression';

import routes from './routes';
import { errorHandler } from './middleware/errorHandler';
import { NotFoundError } from './utils/errors';

const app: Express = express();

// Security
app.use(helmet());
app.use(cors({
  origin: process.env.CORS_ORIGIN || '*',
  credentials: true
}));

// Logging
if (process.env.NODE_ENV !== 'test') {
  app.use(morgan('dev'));
}

// Body parsing
app.use(express.json({ limit: '10kb' }));
app.use(express.urlencoded({ extended: true, limit: '10kb' }));

// Compression
app.use(compression());

// Routes
app.use('/api/v1', routes);

// 404 handler
app.use((req, res, next) => {
  next(new NotFoundError(`Cannot ${req.method} ${req.originalUrl}`));
});

// Error handler
app.use(errorHandler);

export default app;

// src/server.ts
import app from './app';
import { connectDatabase } from './config/database';

const PORT = process.env.PORT || 3000;

const startServer = async (): Promise<void> => {
  try {
    await connectDatabase();

    const server = app.listen(PORT, () => {
      console.log(`Server running on port ${PORT}`);
    });

    // Graceful shutdown
    const shutdown = (): void => {
      console.log('Shutting down gracefully...');
      server.close(() => {
        console.log('Server closed');
        process.exit(0);
      });
    };

    process.on('SIGTERM', shutdown);
    process.on('SIGINT', shutdown);

  } catch (error) {
    console.error('Failed to start server:', error);
    process.exit(1);
  }
};

startServer();
```

---

## Quick Reference

### HTTP Status Codes

| Code | Name | Usage |
|------|------|-------|
| 200 | OK | Successful GET, PUT, PATCH |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Invalid request data |
| 401 | Unauthorized | Missing/invalid authentication |
| 403 | Forbidden | Authenticated but not authorized |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Resource conflict (duplicate) |
| 422 | Unprocessable Entity | Validation errors |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server error |

### Common Middleware Order

```javascript
// 1. Security headers
app.use(helmet());

// 2. CORS
app.use(cors());

// 3. Request logging
app.use(morgan('dev'));

// 4. Body parsing
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// 5. Cookie parsing
app.use(cookieParser());

// 6. Session handling
app.use(session({ ... }));

// 7. Static files
app.use(express.static('public'));

// 8. Custom middleware
app.use(customMiddleware);

// 9. Routes
app.use('/api', routes);

// 10. 404 handler
app.use(notFoundHandler);

// 11. Error handler (always last)
app.use(errorHandler);
```

---

## Additional Resources

- [Express Official Documentation](https://expressjs.com/)
- [Express API Reference](https://expressjs.com/en/4x/api.html)
- [Express Security Best Practices](https://expressjs.com/en/advanced/best-practice-security.html)
- [Express Performance Best Practices](https://expressjs.com/en/advanced/best-practice-performance.html)
