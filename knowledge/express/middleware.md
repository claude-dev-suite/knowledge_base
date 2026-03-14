# Express.js Middleware - Comprehensive Guide

This comprehensive guide covers all aspects of Express.js middleware, from fundamentals to advanced patterns, based on official Express documentation and industry best practices.

---

## Table of Contents

1. [Middleware Fundamentals](#1-middleware-fundamentals)
2. [Application-level Middleware](#2-application-level-middleware)
3. [Router-level Middleware](#3-router-level-middleware)
4. [Error-handling Middleware](#4-error-handling-middleware)
5. [Built-in Middleware](#5-built-in-middleware)
6. [Third-party Middleware](#6-third-party-middleware)
7. [Custom Middleware Patterns](#7-custom-middleware-patterns)
8. [Async Middleware](#8-async-middleware)
9. [Middleware Order and Execution Flow](#9-middleware-order-and-execution-flow)
10. [Conditional Middleware](#10-conditional-middleware)
11. [Middleware Factories](#11-middleware-factories)
12. [Authentication Middleware](#12-authentication-middleware)
13. [Validation Middleware](#13-validation-middleware)
14. [Logging Middleware](#14-logging-middleware)
15. [Rate Limiting Middleware](#15-rate-limiting-middleware)
16. [TypeScript Middleware Types](#16-typescript-middleware-types)
17. [Best Practices](#17-best-practices)

---

## 1. Middleware Fundamentals

Middleware functions are functions that have access to the request object (`req`), the response object (`res`), and the next middleware function in the application's request-response cycle. The next middleware function is commonly denoted by a variable named `next`.

### Basic Middleware Structure

```typescript
import { Request, Response, NextFunction } from 'express';

// Basic middleware signature
const middleware = (req: Request, res: Response, next: NextFunction) => {
  // Middleware logic here
  next(); // Pass control to the next middleware
};
```

### The Request Object (req)

The request object represents the HTTP request and contains properties for the request query string, parameters, body, HTTP headers, and more.

```typescript
const requestInspector = (req: Request, res: Response, next: NextFunction) => {
  // Request properties
  console.log('Method:', req.method);           // GET, POST, PUT, DELETE, etc.
  console.log('URL:', req.url);                 // /users/123
  console.log('Original URL:', req.originalUrl); // Full original URL
  console.log('Base URL:', req.baseUrl);        // Router mount path
  console.log('Path:', req.path);               // Path portion of URL
  console.log('Protocol:', req.protocol);       // http or https
  console.log('Hostname:', req.hostname);       // example.com
  console.log('IP:', req.ip);                   // Client IP address
  console.log('Secure:', req.secure);           // true if HTTPS
  console.log('Fresh:', req.fresh);             // Cache freshness check
  console.log('Stale:', req.stale);             // Opposite of fresh
  console.log('XHR:', req.xhr);                 // true if AJAX request

  // Request data
  console.log('Params:', req.params);           // Route parameters { id: '123' }
  console.log('Query:', req.query);             // Query string { page: '1' }
  console.log('Body:', req.body);               // Parsed request body
  console.log('Cookies:', req.cookies);         // Parsed cookies
  console.log('Headers:', req.headers);         // HTTP headers

  // Header access methods
  console.log('Content-Type:', req.get('Content-Type'));
  console.log('Accepts JSON:', req.accepts('json'));
  console.log('Is JSON:', req.is('json'));

  next();
};
```

### The Response Object (res)

The response object represents the HTTP response that an Express app sends when it gets an HTTP request.

```typescript
const responseDemo = (req: Request, res: Response, next: NextFunction) => {
  // Status codes
  res.status(200);                    // Set status code
  res.sendStatus(200);                // Set and send status

  // Sending responses
  res.send('Hello World');            // Send string/buffer/object
  res.json({ message: 'Hello' });     // Send JSON
  res.jsonp({ message: 'Hello' });    // Send JSONP
  res.end();                          // End response without data

  // Headers
  res.set('X-Custom-Header', 'value');
  res.setHeader('X-Custom-Header', 'value');
  res.get('X-Custom-Header');         // Get response header
  res.type('json');                   // Set Content-Type
  res.contentType('application/json');

  // Cookies
  res.cookie('name', 'value', { httpOnly: true });
  res.clearCookie('name');

  // Redirects
  res.redirect('/new-path');
  res.redirect(301, '/permanent-redirect');

  // File responses
  res.sendFile('/path/to/file');
  res.download('/path/to/file', 'filename.pdf');
  res.attachment('filename.pdf');

  // Templates
  res.render('template', { data: 'value' });

  // Chaining
  res.status(201).json({ created: true });
};
```

### The Next Function

The `next()` function is used to pass control to the next middleware function. If you don't call `next()`, the request will hang.

```typescript
// Passing to next middleware
const loggerMiddleware = (req: Request, res: Response, next: NextFunction) => {
  console.log(`${req.method} ${req.url}`);
  next(); // Continue to next middleware
};

// Skipping remaining route middleware
const skipMiddleware = (req: Request, res: Response, next: NextFunction) => {
  if (req.query.skip) {
    next('route'); // Skip remaining middleware in this route
  } else {
    next();
  }
};

// Passing errors
const errorMiddleware = (req: Request, res: Response, next: NextFunction) => {
  try {
    // Some operation that might fail
    throw new Error('Something went wrong');
  } catch (error) {
    next(error); // Pass error to error-handling middleware
  }
};
```

---

## 2. Application-level Middleware

Application-level middleware is bound to an instance of the `app` object using `app.use()` or `app.METHOD()`.

### Global Middleware

```typescript
import express, { Application, Request, Response, NextFunction } from 'express';

const app: Application = express();

// Applied to ALL requests
app.use((req: Request, res: Response, next: NextFunction) => {
  console.log('Time:', Date.now());
  next();
});

// Applied to ALL requests starting with /api
app.use('/api', (req: Request, res: Response, next: NextFunction) => {
  console.log('API Request');
  next();
});
```

### Method-specific Middleware

```typescript
// Applied only to GET requests
app.get('/users', (req: Request, res: Response, next: NextFunction) => {
  console.log('GET /users');
  next();
});

// Applied only to POST requests
app.post('/users', (req: Request, res: Response, next: NextFunction) => {
  console.log('POST /users');
  next();
});

// Applied to all methods for a specific path
app.all('/secret', (req: Request, res: Response, next: NextFunction) => {
  console.log('Accessing the secret section...');
  next();
});
```

### Multiple Middleware Functions

```typescript
// Array of middleware
const middleware1 = (req: Request, res: Response, next: NextFunction) => {
  console.log('Middleware 1');
  next();
};

const middleware2 = (req: Request, res: Response, next: NextFunction) => {
  console.log('Middleware 2');
  next();
};

// Using array
app.use('/users', [middleware1, middleware2]);

// Using multiple arguments
app.use('/users', middleware1, middleware2);

// Mixed approach
app.get('/users/:id', [middleware1], middleware2, (req, res) => {
  res.json({ id: req.params.id });
});
```

### Mounting at Specific Paths

```typescript
// Mount middleware at /admin path
app.use('/admin', (req: Request, res: Response, next: NextFunction) => {
  // req.baseUrl will be '/admin'
  // req.path will be the rest of the URL after /admin
  console.log(`Admin access: ${req.baseUrl}${req.path}`);
  next();
});

// Multiple mount paths
app.use(['/api', '/api-v2'], (req: Request, res: Response, next: NextFunction) => {
  console.log('API request on:', req.baseUrl);
  next();
});

// Path patterns with wildcards
app.use('/api/*', (req: Request, res: Response, next: NextFunction) => {
  console.log('API wildcard match');
  next();
});
```

---

## 3. Router-level Middleware

Router-level middleware works exactly like application-level middleware, except it is bound to an instance of `express.Router()`.

### Basic Router Middleware

```typescript
import express, { Router, Request, Response, NextFunction } from 'express';

const router: Router = express.Router();

// Router-level middleware for all routes in this router
router.use((req: Request, res: Response, next: NextFunction) => {
  console.log('Router Time:', Date.now());
  next();
});

// Middleware for specific path within router
router.use('/user/:id', (req: Request, res: Response, next: NextFunction) => {
  console.log('Request URL:', req.originalUrl);
  next();
}, (req: Request, res: Response, next: NextFunction) => {
  console.log('Request Type:', req.method);
  next();
});

// Route handlers
router.get('/user/:id', (req: Request, res: Response) => {
  res.json({ id: req.params.id });
});

// Mount router on app
const app = express();
app.use('/api', router);
```

### Modular Route Organization

```typescript
// routes/users.ts
import { Router, Request, Response, NextFunction } from 'express';

const usersRouter = Router();

// Router-specific middleware
const validateUserId = (req: Request, res: Response, next: NextFunction) => {
  const id = parseInt(req.params.id);
  if (isNaN(id) || id < 1) {
    return res.status(400).json({ error: 'Invalid user ID' });
  }
  next();
};

usersRouter.use('/:id', validateUserId);

usersRouter.get('/', (req, res) => {
  res.json({ users: [] });
});

usersRouter.get('/:id', (req, res) => {
  res.json({ user: { id: req.params.id } });
});

usersRouter.post('/', (req, res) => {
  res.status(201).json({ user: req.body });
});

export default usersRouter;

// routes/posts.ts
import { Router } from 'express';

const postsRouter = Router();

postsRouter.get('/', (req, res) => {
  res.json({ posts: [] });
});

export default postsRouter;

// app.ts
import express from 'express';
import usersRouter from './routes/users';
import postsRouter from './routes/posts';

const app = express();

app.use('/api/users', usersRouter);
app.use('/api/posts', postsRouter);
```

### Router.param() Middleware

```typescript
const router = Router();

// Pre-process parameter
router.param('userId', async (req: Request, res: Response, next: NextFunction, id: string) => {
  try {
    const user = await User.findById(id);
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    (req as any).user = user;
    next();
  } catch (error) {
    next(error);
  }
});

// Routes using the parameter
router.get('/users/:userId', (req, res) => {
  res.json({ user: (req as any).user });
});

router.put('/users/:userId', (req, res) => {
  res.json({ user: (req as any).user });
});
```

### Nested Routers

```typescript
const apiRouter = Router();
const v1Router = Router();
const v2Router = Router();

// V1 routes
v1Router.get('/users', (req, res) => {
  res.json({ version: 1, users: [] });
});

// V2 routes with different response structure
v2Router.get('/users', (req, res) => {
  res.json({
    version: 2,
    data: { users: [] },
    meta: { total: 0 }
  });
});

// Mount version routers
apiRouter.use('/v1', v1Router);
apiRouter.use('/v2', v2Router);

// Mount API router
app.use('/api', apiRouter);

// Results in:
// GET /api/v1/users
// GET /api/v2/users
```

---

## 4. Error-handling Middleware

Error-handling middleware functions have four arguments: `(err, req, res, next)`. This signature tells Express that this is an error-handling middleware.

### Basic Error Handler

```typescript
import { Request, Response, NextFunction } from 'express';

// Error interface
interface AppError extends Error {
  statusCode?: number;
  status?: string;
  isOperational?: boolean;
}

// Basic error handler
const errorHandler = (
  err: AppError,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  console.error('Error:', err);

  const statusCode = err.statusCode || 500;
  const message = err.message || 'Internal Server Error';

  res.status(statusCode).json({
    status: 'error',
    statusCode,
    message,
  });
};

// Register error handler (must be last)
app.use(errorHandler);
```

### Custom Error Classes

```typescript
// errors/AppError.ts
export class AppError extends Error {
  public statusCode: number;
  public status: string;
  public isOperational: boolean;

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

export class BadRequestError extends AppError {
  constructor(message = 'Bad request') {
    super(message, 400);
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(message, 401);
  }
}

export class ForbiddenError extends AppError {
  constructor(message = 'Forbidden') {
    super(message, 403);
  }
}

export class ValidationError extends AppError {
  public errors: any[];

  constructor(message = 'Validation failed', errors: any[] = []) {
    super(message, 400);
    this.errors = errors;
  }
}
```

### Comprehensive Error Handler

```typescript
import { Request, Response, NextFunction } from 'express';
import { AppError, ValidationError } from './errors/AppError';

// Development error response
const sendErrorDev = (err: AppError, res: Response) => {
  res.status(err.statusCode || 500).json({
    status: err.status,
    error: err,
    message: err.message,
    stack: err.stack,
  });
};

// Production error response
const sendErrorProd = (err: AppError, res: Response) => {
  // Operational, trusted error: send message to client
  if (err.isOperational) {
    res.status(err.statusCode || 500).json({
      status: err.status,
      message: err.message,
      ...(err instanceof ValidationError && { errors: err.errors }),
    });
  } else {
    // Programming or unknown error: don't leak error details
    console.error('ERROR:', err);
    res.status(500).json({
      status: 'error',
      message: 'Something went wrong',
    });
  }
};

// Handle specific error types
const handleCastErrorDB = (err: any): AppError => {
  const message = `Invalid ${err.path}: ${err.value}`;
  return new AppError(message, 400);
};

const handleDuplicateFieldsDB = (err: any): AppError => {
  const value = err.errmsg?.match(/(["'])(\\?.)*?\1/)?.[0];
  const message = `Duplicate field value: ${value}. Please use another value.`;
  return new AppError(message, 400);
};

const handleValidationErrorDB = (err: any): AppError => {
  const errors = Object.values(err.errors).map((el: any) => el.message);
  const message = `Invalid input data. ${errors.join('. ')}`;
  return new AppError(message, 400);
};

const handleJWTError = (): AppError =>
  new AppError('Invalid token. Please log in again.', 401);

const handleJWTExpiredError = (): AppError =>
  new AppError('Your token has expired. Please log in again.', 401);

// Global error handler
export const globalErrorHandler = (
  err: any,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  err.statusCode = err.statusCode || 500;
  err.status = err.status || 'error';

  if (process.env.NODE_ENV === 'development') {
    sendErrorDev(err, res);
  } else {
    let error = { ...err };
    error.message = err.message;

    // MongoDB errors
    if (err.name === 'CastError') error = handleCastErrorDB(error);
    if (err.code === 11000) error = handleDuplicateFieldsDB(error);
    if (err.name === 'ValidationError') error = handleValidationErrorDB(error);

    // JWT errors
    if (err.name === 'JsonWebTokenError') error = handleJWTError();
    if (err.name === 'TokenExpiredError') error = handleJWTExpiredError();

    sendErrorProd(error, res);
  }
};
```

### 404 Not Found Handler

```typescript
// Handle unmatched routes
const notFoundHandler = (req: Request, res: Response, next: NextFunction) => {
  const err = new AppError(
    `Cannot find ${req.originalUrl} on this server`,
    404
  );
  next(err);
};

// Register before error handler
app.use(notFoundHandler);
app.use(globalErrorHandler);
```

### Async Error Wrapper

```typescript
// Wrap async route handlers to catch errors
const catchAsync = (fn: Function) => {
  return (req: Request, res: Response, next: NextFunction) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
};

// Usage
app.get('/users/:id', catchAsync(async (req: Request, res: Response) => {
  const user = await User.findById(req.params.id);
  if (!user) {
    throw new NotFoundError('User not found');
  }
  res.json({ user });
}));
```

---

## 5. Built-in Middleware

Express has several built-in middleware functions that handle common tasks.

### express.json()

Parses incoming requests with JSON payloads.

```typescript
import express from 'express';

const app = express();

// Basic usage
app.use(express.json());

// With options
app.use(express.json({
  limit: '10kb',              // Maximum request body size
  strict: true,               // Only accept arrays and objects
  type: 'application/json',   // Content-Type to parse
  verify: (req, res, buf, encoding) => {
    // Custom verification function
    // Store raw body for webhook signature verification
    (req as any).rawBody = buf;
  },
  reviver: (key, value) => {
    // JSON.parse reviver function
    // Convert date strings to Date objects
    if (typeof value === 'string' && /^\d{4}-\d{2}-\d{2}T/.test(value)) {
      return new Date(value);
    }
    return value;
  },
}));

// Route-specific JSON parsing with different options
app.post('/webhooks/stripe',
  express.json({
    type: 'application/json',
    verify: (req, res, buf) => {
      (req as any).rawBody = buf.toString();
    },
  }),
  (req, res) => {
    // Verify Stripe signature using rawBody
    res.json({ received: true });
  }
);
```

### express.urlencoded()

Parses incoming requests with URL-encoded payloads (form data).

```typescript
// Basic usage
app.use(express.urlencoded({ extended: true }));

// With options
app.use(express.urlencoded({
  extended: true,             // Use qs library (allows nested objects)
  limit: '10kb',              // Maximum request body size
  parameterLimit: 1000,       // Maximum number of parameters
  type: 'application/x-www-form-urlencoded',
  verify: (req, res, buf, encoding) => {
    // Custom verification
  },
}));

// Difference between extended: true and false
// extended: false - uses querystring library (flat data)
// POST: name=John&age=30 -> { name: 'John', age: '30' }

// extended: true - uses qs library (nested data)
// POST: user[name]=John&user[age]=30 -> { user: { name: 'John', age: '30' } }
// POST: items[0]=a&items[1]=b -> { items: ['a', 'b'] }
```

### express.static()

Serves static files such as images, CSS, and JavaScript.

```typescript
import path from 'path';

// Basic usage - serve files from 'public' directory
app.use(express.static('public'));
// GET /style.css serves public/style.css

// With virtual path prefix
app.use('/static', express.static('public'));
// GET /static/style.css serves public/style.css

// Absolute path (recommended)
app.use(express.static(path.join(__dirname, 'public')));

// Multiple static directories
app.use(express.static('public'));
app.use(express.static('files'));
// Express looks in order: public first, then files

// With options
app.use(express.static('public', {
  dotfiles: 'ignore',         // 'allow', 'deny', 'ignore' for dotfiles
  etag: true,                 // Enable ETag generation
  extensions: ['html', 'htm'], // File extensions to look for
  fallthrough: true,          // Pass to next middleware if file not found
  immutable: true,            // Add immutable directive to Cache-Control
  index: 'index.html',        // Default file to serve for directories
  lastModified: true,         // Set Last-Modified header
  maxAge: '1d',               // Cache-Control max-age (ms or string)
  redirect: true,             // Redirect to trailing / for directories
  setHeaders: (res, path, stat) => {
    // Custom headers for served files
    if (path.endsWith('.html')) {
      res.set('Cache-Control', 'no-cache');
    }
    if (path.endsWith('.js') || path.endsWith('.css')) {
      res.set('Cache-Control', 'public, max-age=31536000, immutable');
    }
  },
}));
```

### express.raw()

Parses incoming request payloads into a Buffer.

```typescript
// Parse all content types as Buffer
app.use(express.raw());

// Parse specific content types
app.use(express.raw({
  type: 'application/octet-stream',
  limit: '5mb',
}));

// For webhook payloads
app.post('/webhooks',
  express.raw({ type: 'application/json' }),
  (req, res) => {
    const rawBody = req.body; // Buffer
    // Verify signature using raw body
    res.sendStatus(200);
  }
);
```

### express.text()

Parses incoming request payloads into a string.

```typescript
// Parse as text
app.use(express.text());

// With options
app.use(express.text({
  type: 'text/plain',
  limit: '100kb',
  defaultCharset: 'utf-8',
}));

// For XML payloads
app.post('/xml',
  express.text({ type: 'application/xml' }),
  (req, res) => {
    const xmlString = req.body;
    // Parse XML string
    res.json({ received: true });
  }
);
```

---

## 6. Third-party Middleware

### Helmet - Security Headers

```typescript
import helmet from 'helmet';

// Use all defaults (recommended)
app.use(helmet());

// Custom configuration
app.use(helmet({
  // Content Security Policy
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'", 'https://fonts.googleapis.com'],
      fontSrc: ["'self'", 'https://fonts.gstatic.com'],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", 'data:', 'https:'],
      connectSrc: ["'self'", 'https://api.example.com'],
      frameSrc: ["'none'"],
      objectSrc: ["'none'"],
      upgradeInsecureRequests: [],
    },
  },
  // Cross-Origin settings
  crossOriginEmbedderPolicy: false,
  crossOriginOpenerPolicy: { policy: 'same-origin' },
  crossOriginResourcePolicy: { policy: 'same-origin' },
  // Other headers
  dnsPrefetchControl: { allow: false },
  frameguard: { action: 'deny' },
  hidePoweredBy: true,
  hsts: { maxAge: 31536000, includeSubDomains: true, preload: true },
  ieNoOpen: true,
  noSniff: true,
  originAgentCluster: true,
  permittedCrossDomainPolicies: { permittedPolicies: 'none' },
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
  xssFilter: true,
}));

// Disable specific middleware
app.use(helmet({
  contentSecurityPolicy: false, // Disable CSP
}));
```

### CORS - Cross-Origin Resource Sharing

```typescript
import cors, { CorsOptions } from 'cors';

// Allow all origins (not recommended for production)
app.use(cors());

// Specific origins
const corsOptions: CorsOptions = {
  origin: ['http://localhost:3000', 'https://myapp.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Requested-With'],
  exposedHeaders: ['X-Total-Count', 'X-Page-Count'],
  credentials: true,              // Allow cookies
  maxAge: 86400,                  // Preflight cache duration (24 hours)
  preflightContinue: false,       // Pass preflight to next handler
  optionsSuccessStatus: 204,      // Status for successful OPTIONS requests
};

app.use(cors(corsOptions));

// Dynamic origin
app.use(cors({
  origin: (origin, callback) => {
    const allowedOrigins = [
      'http://localhost:3000',
      'https://myapp.com',
      /\.myapp\.com$/,  // Allow subdomains
    ];

    // Allow requests with no origin (mobile apps, curl, etc.)
    if (!origin) return callback(null, true);

    const isAllowed = allowedOrigins.some(allowed => {
      if (allowed instanceof RegExp) {
        return allowed.test(origin);
      }
      return allowed === origin;
    });

    if (isAllowed) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
}));

// Route-specific CORS
app.options('/api/*', cors()); // Enable preflight for API routes
app.get('/public', cors(), (req, res) => {
  res.json({ data: 'public' });
});
```

### Morgan - HTTP Request Logger

```typescript
import morgan from 'morgan';
import { createWriteStream } from 'fs';
import path from 'path';

// Predefined formats
app.use(morgan('combined'));  // Apache combined format
app.use(morgan('common'));    // Apache common format
app.use(morgan('dev'));       // Colored dev output
app.use(morgan('short'));     // Shorter than default
app.use(morgan('tiny'));      // Minimal output

// Custom format string
app.use(morgan(':method :url :status :res[content-length] - :response-time ms'));

// Custom tokens
morgan.token('id', (req: Request) => (req as any).id);
morgan.token('user', (req: Request) => (req as any).user?.email || 'anonymous');
morgan.token('body', (req: Request) => {
  const body = { ...req.body };
  // Redact sensitive fields
  if (body.password) body.password = '[REDACTED]';
  if (body.token) body.token = '[REDACTED]';
  return JSON.stringify(body);
});

app.use(morgan(':id :user :method :url :status :response-time ms - :body'));

// Write to file
const accessLogStream = createWriteStream(
  path.join(__dirname, 'logs', 'access.log'),
  { flags: 'a' }
);

app.use(morgan('combined', { stream: accessLogStream }));

// Conditional logging
app.use(morgan('dev', {
  skip: (req, res) => res.statusCode < 400, // Only log errors
}));

// Different logging for different environments
if (process.env.NODE_ENV === 'development') {
  app.use(morgan('dev'));
} else {
  app.use(morgan('combined', { stream: accessLogStream }));
}
```

### Compression

```typescript
import compression from 'compression';

// Basic usage
app.use(compression());

// With options
app.use(compression({
  level: 6,                    // Compression level (0-9, -1 for default)
  threshold: 1024,             // Only compress responses > 1KB
  memLevel: 8,                 // Memory level (1-9)
  chunkSize: 16384,            // Chunk size for streaming
  windowBits: 15,              // Window size
  strategy: 0,                 // Compression strategy
  filter: (req, res) => {
    // Don't compress if x-no-compression header is present
    if (req.headers['x-no-compression']) {
      return false;
    }
    // Use default filter (compresses text/* and application/json, etc.)
    return compression.filter(req, res);
  },
}));

// Skip compression for certain routes
app.use('/api/stream', (req, res, next) => {
  // Disable compression for streaming endpoints
  res.setHeader('X-No-Compression', 'true');
  next();
});
```

### Cookie Parser

```typescript
import cookieParser from 'cookie-parser';

// Basic usage
app.use(cookieParser());

// With secret for signed cookies
app.use(cookieParser('my-secret-key'));

// With options
app.use(cookieParser('my-secret-key', {
  decode: (value) => decodeURIComponent(value),
}));

// Accessing cookies
app.get('/dashboard', (req, res) => {
  // Regular cookies
  const sessionId = req.cookies.sessionId;

  // Signed cookies (verified with secret)
  const userId = req.signedCookies.userId;

  res.json({ sessionId, userId });
});

// Setting cookies
app.post('/login', (req, res) => {
  // Regular cookie
  res.cookie('sessionId', 'abc123', {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 24 * 60 * 60 * 1000, // 24 hours
  });

  // Signed cookie
  res.cookie('userId', 'user123', {
    signed: true,
    httpOnly: true,
  });

  res.json({ success: true });
});
```

### Express Session

```typescript
import session from 'express-session';
import RedisStore from 'connect-redis';
import { createClient } from 'redis';

// Redis client
const redisClient = createClient({ url: process.env.REDIS_URL });
redisClient.connect();

// Session configuration
app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET!,
  name: 'sessionId',           // Cookie name (default: connect.sid)
  resave: false,               // Don't save if not modified
  saveUninitialized: false,    // Don't save empty sessions
  rolling: true,               // Reset expiry on each request
  cookie: {
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
    sameSite: 'strict',
    maxAge: 24 * 60 * 60 * 1000, // 24 hours
  },
}));

// Using sessions
app.post('/login', (req, res) => {
  req.session.userId = 'user123';
  req.session.role = 'admin';
  res.json({ success: true });
});

app.get('/profile', (req, res) => {
  if (!req.session.userId) {
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
    res.json({ success: true });
  });
});
```

---

## 7. Custom Middleware Patterns

### Request ID Middleware

```typescript
import { v4 as uuidv4 } from 'uuid';

const requestId = (req: Request, res: Response, next: NextFunction) => {
  const id = req.headers['x-request-id'] as string || uuidv4();
  (req as any).id = id;
  res.setHeader('X-Request-ID', id);
  next();
};

app.use(requestId);
```

### Response Time Middleware

```typescript
const responseTime = (req: Request, res: Response, next: NextFunction) => {
  const start = process.hrtime();

  res.on('finish', () => {
    const [seconds, nanoseconds] = process.hrtime(start);
    const duration = seconds * 1000 + nanoseconds / 1000000;
    console.log(`${req.method} ${req.url} - ${duration.toFixed(2)}ms`);
  });

  next();
};

app.use(responseTime);
```

### Request Sanitization Middleware

```typescript
import sanitizeHtml from 'sanitize-html';

const sanitizeBody = (req: Request, res: Response, next: NextFunction) => {
  if (req.body && typeof req.body === 'object') {
    const sanitize = (obj: any): any => {
      if (typeof obj === 'string') {
        return sanitizeHtml(obj, {
          allowedTags: [],
          allowedAttributes: {},
        });
      }
      if (Array.isArray(obj)) {
        return obj.map(sanitize);
      }
      if (typeof obj === 'object' && obj !== null) {
        const sanitized: any = {};
        for (const [key, value] of Object.entries(obj)) {
          sanitized[key] = sanitize(value);
        }
        return sanitized;
      }
      return obj;
    };

    req.body = sanitize(req.body);
  }
  next();
};

app.use(express.json());
app.use(sanitizeBody);
```

### Caching Middleware

```typescript
interface CacheOptions {
  ttl: number;
  key?: (req: Request) => string;
}

const cache = new Map<string, { data: any; expires: number }>();

const cacheMiddleware = (options: CacheOptions) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const key = options.key?.(req) || `${req.method}:${req.originalUrl}`;
    const cached = cache.get(key);

    if (cached && cached.expires > Date.now()) {
      return res.json(cached.data);
    }

    // Override res.json to cache the response
    const originalJson = res.json.bind(res);
    res.json = (data: any) => {
      cache.set(key, {
        data,
        expires: Date.now() + options.ttl,
      });
      return originalJson(data);
    };

    next();
  };
};

// Usage
app.get('/api/products',
  cacheMiddleware({ ttl: 60000 }), // Cache for 1 minute
  async (req, res) => {
    const products = await Product.findAll();
    res.json(products);
  }
);
```

### IP Whitelist/Blacklist Middleware

```typescript
interface IPFilterOptions {
  whitelist?: string[];
  blacklist?: string[];
  message?: string;
}

const ipFilter = (options: IPFilterOptions) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const clientIP = req.ip || req.socket.remoteAddress || '';

    if (options.blacklist?.includes(clientIP)) {
      return res.status(403).json({
        error: options.message || 'Access denied'
      });
    }

    if (options.whitelist && !options.whitelist.includes(clientIP)) {
      return res.status(403).json({
        error: options.message || 'Access denied'
      });
    }

    next();
  };
};

// Usage
app.use('/admin', ipFilter({
  whitelist: ['127.0.0.1', '::1', '192.168.1.100'],
  message: 'Admin access restricted',
}));
```

### Request Timeout Middleware

```typescript
const timeout = (ms: number) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const timeoutId = setTimeout(() => {
      if (!res.headersSent) {
        res.status(408).json({ error: 'Request timeout' });
      }
    }, ms);

    res.on('finish', () => clearTimeout(timeoutId));
    res.on('close', () => clearTimeout(timeoutId));

    next();
  };
};

app.use(timeout(30000)); // 30 second timeout
```

---

## 8. Async Middleware

### Basic Async Wrapper

```typescript
type AsyncHandler = (
  req: Request,
  res: Response,
  next: NextFunction
) => Promise<any>;

const asyncHandler = (fn: AsyncHandler) => {
  return (req: Request, res: Response, next: NextFunction) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
};

// Usage
app.get('/users', asyncHandler(async (req, res) => {
  const users = await User.findAll();
  res.json(users);
}));

app.get('/users/:id', asyncHandler(async (req, res) => {
  const user = await User.findById(req.params.id);
  if (!user) {
    throw new NotFoundError('User not found');
  }
  res.json(user);
}));
```

### Express-async-errors Package

```typescript
// Automatically wraps all async middleware
import 'express-async-errors';

// No need for asyncHandler wrapper
app.get('/users', async (req, res) => {
  const users = await User.findAll();
  res.json(users);
});

// Errors are automatically passed to error handler
app.get('/users/:id', async (req, res) => {
  const user = await User.findById(req.params.id);
  if (!user) {
    throw new NotFoundError('User not found');
  }
  res.json(user);
});
```

### Async Middleware with Timeout

```typescript
const asyncWithTimeout = (fn: AsyncHandler, timeoutMs: number) => {
  return async (req: Request, res: Response, next: NextFunction) => {
    const timeoutPromise = new Promise((_, reject) => {
      setTimeout(() => reject(new Error('Operation timed out')), timeoutMs);
    });

    try {
      await Promise.race([fn(req, res, next), timeoutPromise]);
    } catch (error) {
      next(error);
    }
  };
};

// Usage
app.get('/slow-operation', asyncWithTimeout(async (req, res) => {
  const result = await slowDatabaseQuery();
  res.json(result);
}, 5000));
```

### Parallel Async Operations

```typescript
app.get('/dashboard', asyncHandler(async (req, res) => {
  // Run operations in parallel
  const [users, products, orders] = await Promise.all([
    User.count(),
    Product.count(),
    Order.findRecent(),
  ]);

  res.json({
    stats: {
      userCount: users,
      productCount: products,
      recentOrders: orders,
    },
  });
}));
```

### Sequential Async Middleware

```typescript
const loadUser = asyncHandler(async (req: Request, res: Response, next: NextFunction) => {
  const user = await User.findById(req.params.userId);
  if (!user) {
    throw new NotFoundError('User not found');
  }
  (req as any).user = user;
  next();
});

const loadUserPosts = asyncHandler(async (req: Request, res: Response, next: NextFunction) => {
  const posts = await Post.findByUserId((req as any).user.id);
  (req as any).posts = posts;
  next();
});

app.get('/users/:userId/dashboard',
  loadUser,
  loadUserPosts,
  (req, res) => {
    res.json({
      user: (req as any).user,
      posts: (req as any).posts,
    });
  }
);
```

---

## 9. Middleware Order and Execution Flow

### Standard Middleware Order

```typescript
import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import compression from 'compression';
import morgan from 'morgan';
import cookieParser from 'cookie-parser';

const app = express();

// 1. Trust proxy (if behind reverse proxy)
app.set('trust proxy', 1);

// 2. Security headers (first for all responses)
app.use(helmet());

// 3. CORS (before routes, to handle preflight)
app.use(cors());

// 4. Compression (before body parsing)
app.use(compression());

// 5. Request logging
app.use(morgan('combined'));

// 6. Body parsing
app.use(express.json({ limit: '10kb' }));
app.use(express.urlencoded({ extended: true, limit: '10kb' }));

// 7. Cookie parsing
app.use(cookieParser());

// 8. Static files (before routes)
app.use(express.static('public'));

// 9. Custom middleware (request ID, etc.)
app.use(requestIdMiddleware);

// 10. Rate limiting
app.use(rateLimiter);

// 11. API routes
app.use('/api/v1', apiRoutes);

// 12. 404 handler (after all routes)
app.use(notFoundHandler);

// 13. Error handler (always last)
app.use(errorHandler);
```

### Execution Flow Visualization

```typescript
// Middleware execution is like an onion
// Request flows in -> Response flows out

const first = (req: Request, res: Response, next: NextFunction) => {
  console.log('1. First - Before');
  next();
  console.log('6. First - After');
};

const second = (req: Request, res: Response, next: NextFunction) => {
  console.log('2. Second - Before');
  next();
  console.log('5. Second - After');
};

const third = (req: Request, res: Response, next: NextFunction) => {
  console.log('3. Third - Before');
  next();
  console.log('4. Third - After');
};

app.use(first);
app.use(second);
app.use(third);

app.get('/', (req, res) => {
  console.log('Handler');
  res.send('OK');
});

// Output order:
// 1. First - Before
// 2. Second - Before
// 3. Third - Before
// Handler
// 4. Third - After
// 5. Second - After
// 6. First - After
```

### Short-circuiting Middleware

```typescript
// Middleware that stops execution
const authMiddleware = (req: Request, res: Response, next: NextFunction) => {
  if (!req.headers.authorization) {
    // Response sent, no next() called - chain stops
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next(); // Continue to next middleware
};

// Skip to next route
const conditionalSkip = (req: Request, res: Response, next: NextFunction) => {
  if (req.query.skip) {
    return next('route'); // Skip remaining middleware for this route
  }
  next();
};

// Skip to error handler
const errorSkip = (req: Request, res: Response, next: NextFunction) => {
  const error = new Error('Something wrong');
  next(error); // Jumps directly to error handler
};
```

### Route-specific vs Global Middleware

```typescript
// Global - applies to all routes
app.use(morgan('dev'));

// Path-specific global
app.use('/api', apiMiddleware);

// Route-specific
app.get('/protected', authMiddleware, (req, res) => {
  res.json({ secret: 'data' });
});

// Multiple route-specific
app.get('/admin/users',
  authMiddleware,
  requireRole('admin'),
  async (req, res) => {
    const users = await User.findAll();
    res.json(users);
  }
);
```

---

## 10. Conditional Middleware

### Environment-based Middleware

```typescript
const isDevelopment = process.env.NODE_ENV === 'development';
const isProduction = process.env.NODE_ENV === 'production';

// Only in development
if (isDevelopment) {
  app.use(morgan('dev'));
  app.use('/debug', debugRouter);
}

// Only in production
if (isProduction) {
  app.use(helmet());
  app.use(compression());
}

// Conditional middleware factory
const when = (
  condition: boolean | ((req: Request) => boolean),
  middleware: express.RequestHandler
) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const shouldApply = typeof condition === 'function'
      ? condition(req)
      : condition;

    if (shouldApply) {
      return middleware(req, res, next);
    }
    next();
  };
};

// Usage
app.use(when(isProduction, compression()));
app.use(when((req) => req.path.startsWith('/api'), apiLogger));
```

### Request-based Conditional Middleware

```typescript
// Based on content type
const jsonOnly = (req: Request, res: Response, next: NextFunction) => {
  if (req.is('json')) {
    return express.json()(req, res, next);
  }
  next();
};

// Based on headers
const adminOnly = (req: Request, res: Response, next: NextFunction) => {
  const isAdmin = req.headers['x-admin-key'] === process.env.ADMIN_KEY;
  if (!isAdmin) {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

// Based on query parameters
const withPagination = (req: Request, res: Response, next: NextFunction) => {
  if (req.query.page || req.query.limit) {
    (req as any).pagination = {
      page: parseInt(req.query.page as string) || 1,
      limit: Math.min(parseInt(req.query.limit as string) || 10, 100),
    };
  }
  next();
};

// Based on path pattern
const apiVersioning = (req: Request, res: Response, next: NextFunction) => {
  const versionMatch = req.path.match(/^\/api\/v(\d+)/);
  if (versionMatch) {
    (req as any).apiVersion = parseInt(versionMatch[1]);
  }
  next();
};
```

### Feature Flag Middleware

```typescript
interface FeatureFlags {
  [key: string]: boolean;
}

const featureFlags: FeatureFlags = {
  newAuth: true,
  betaFeatures: false,
  analytics: true,
};

const requireFeature = (feature: string) => {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!featureFlags[feature]) {
      return res.status(404).json({ error: 'Feature not available' });
    }
    next();
  };
};

// Usage
app.use('/beta', requireFeature('betaFeatures'), betaRoutes);

// Dynamic feature check
const ifFeature = (feature: string, middleware: express.RequestHandler) => {
  return (req: Request, res: Response, next: NextFunction) => {
    if (featureFlags[feature]) {
      return middleware(req, res, next);
    }
    next();
  };
};

app.use(ifFeature('analytics', analyticsMiddleware));
```

### User-based Conditional Middleware

```typescript
// Based on user role
const conditionalAuth = (req: Request, res: Response, next: NextFunction) => {
  // Public routes don't need auth
  const publicRoutes = ['/health', '/login', '/register'];
  if (publicRoutes.includes(req.path)) {
    return next();
  }
  return authMiddleware(req, res, next);
};

// Based on subscription level
const premiumOnly = (req: Request, res: Response, next: NextFunction) => {
  const user = (req as any).user;
  if (!user || user.subscription !== 'premium') {
    return res.status(403).json({
      error: 'Premium subscription required',
      upgradeUrl: '/pricing',
    });
  }
  next();
};
```

---

## 11. Middleware Factories

### Basic Factory Pattern

```typescript
interface LoggerOptions {
  level: 'debug' | 'info' | 'warn' | 'error';
  format?: 'json' | 'text';
  includeBody?: boolean;
}

const createLogger = (options: LoggerOptions) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const log = {
      timestamp: new Date().toISOString(),
      method: req.method,
      url: req.url,
      ip: req.ip,
      ...(options.includeBody && { body: req.body }),
    };

    if (options.format === 'json') {
      console[options.level](JSON.stringify(log));
    } else {
      console[options.level](`${log.timestamp} ${log.method} ${log.url}`);
    }

    next();
  };
};

// Usage
app.use(createLogger({ level: 'info', format: 'json', includeBody: true }));
```

### Configurable Rate Limiter Factory

```typescript
interface RateLimitConfig {
  windowMs: number;
  maxRequests: number;
  message?: string;
  keyGenerator?: (req: Request) => string;
  skip?: (req: Request) => boolean;
  onLimitReached?: (req: Request) => void;
}

const createRateLimiter = (config: RateLimitConfig) => {
  const hits = new Map<string, { count: number; resetTime: number }>();

  return (req: Request, res: Response, next: NextFunction) => {
    // Skip if configured
    if (config.skip?.(req)) {
      return next();
    }

    const key = config.keyGenerator?.(req) || req.ip || 'unknown';
    const now = Date.now();
    const record = hits.get(key);

    // Reset if window expired
    if (!record || now > record.resetTime) {
      hits.set(key, { count: 1, resetTime: now + config.windowMs });
      return next();
    }

    // Increment hit count
    record.count++;

    // Check limit
    if (record.count > config.maxRequests) {
      config.onLimitReached?.(req);

      res.set('Retry-After', String(Math.ceil((record.resetTime - now) / 1000)));
      return res.status(429).json({
        error: config.message || 'Too many requests',
        retryAfter: Math.ceil((record.resetTime - now) / 1000),
      });
    }

    next();
  };
};

// Usage
const apiLimiter = createRateLimiter({
  windowMs: 15 * 60 * 1000, // 15 minutes
  maxRequests: 100,
  keyGenerator: (req) => req.headers['x-api-key'] as string || req.ip || 'unknown',
  skip: (req) => req.path === '/health',
  onLimitReached: (req) => {
    console.warn(`Rate limit exceeded for ${req.ip}`);
  },
});

app.use('/api', apiLimiter);
```

### Validation Middleware Factory

```typescript
import { z, ZodSchema, ZodError } from 'zod';

interface ValidationOptions {
  body?: ZodSchema;
  query?: ZodSchema;
  params?: ZodSchema;
  stripUnknown?: boolean;
}

const validate = (options: ValidationOptions) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const errors: { location: string; errors: any[] }[] = [];

    try {
      if (options.body) {
        req.body = options.body.parse(req.body);
      }
    } catch (error) {
      if (error instanceof ZodError) {
        errors.push({ location: 'body', errors: error.errors });
      }
    }

    try {
      if (options.query) {
        req.query = options.query.parse(req.query) as any;
      }
    } catch (error) {
      if (error instanceof ZodError) {
        errors.push({ location: 'query', errors: error.errors });
      }
    }

    try {
      if (options.params) {
        req.params = options.params.parse(req.params);
      }
    } catch (error) {
      if (error instanceof ZodError) {
        errors.push({ location: 'params', errors: error.errors });
      }
    }

    if (errors.length > 0) {
      return res.status(400).json({
        error: 'Validation failed',
        details: errors,
      });
    }

    next();
  };
};

// Schemas
const createUserBody = z.object({
  name: z.string().min(2).max(100),
  email: z.string().email(),
  password: z.string().min(8),
});

const userIdParams = z.object({
  id: z.string().uuid(),
});

const paginationQuery = z.object({
  page: z.string().regex(/^\d+$/).transform(Number).optional(),
  limit: z.string().regex(/^\d+$/).transform(Number).optional(),
});

// Usage
app.post('/users',
  validate({ body: createUserBody }),
  async (req, res) => {
    const user = await User.create(req.body);
    res.status(201).json(user);
  }
);

app.get('/users/:id',
  validate({ params: userIdParams }),
  async (req, res) => {
    const user = await User.findById(req.params.id);
    res.json(user);
  }
);

app.get('/users',
  validate({ query: paginationQuery }),
  async (req, res) => {
    const { page = 1, limit = 10 } = req.query;
    const users = await User.findPaginated(page, limit);
    res.json(users);
  }
);
```

### Permission Middleware Factory

```typescript
type Permission = string;
type PermissionChecker = (user: any, req: Request) => boolean;

const hasPermission = (
  permission: Permission | Permission[] | PermissionChecker
) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const user = (req as any).user;

    if (!user) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    let hasAccess = false;

    if (typeof permission === 'function') {
      hasAccess = permission(user, req);
    } else if (Array.isArray(permission)) {
      hasAccess = permission.some(p => user.permissions?.includes(p));
    } else {
      hasAccess = user.permissions?.includes(permission);
    }

    if (!hasAccess) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }

    next();
  };
};

// Usage
app.delete('/users/:id',
  authMiddleware,
  hasPermission('users:delete'),
  async (req, res) => {
    await User.delete(req.params.id);
    res.status(204).send();
  }
);

// With custom checker
app.put('/posts/:id',
  authMiddleware,
  hasPermission((user, req) => {
    // User can edit their own posts or has admin role
    return user.id === req.params.authorId || user.role === 'admin';
  }),
  async (req, res) => {
    const post = await Post.update(req.params.id, req.body);
    res.json(post);
  }
);
```

---

## 12. Authentication Middleware

### JWT Authentication

```typescript
import jwt from 'jsonwebtoken';
import { Request, Response, NextFunction } from 'express';

interface JWTPayload {
  id: string;
  email: string;
  role: string;
  iat: number;
  exp: number;
}

interface AuthenticatedRequest extends Request {
  user?: JWTPayload;
}

const authenticateJWT = async (
  req: AuthenticatedRequest,
  res: Response,
  next: NextFunction
) => {
  try {
    const authHeader = req.headers.authorization;

    if (!authHeader?.startsWith('Bearer ')) {
      return res.status(401).json({ error: 'No token provided' });
    }

    const token = authHeader.split(' ')[1];

    const decoded = jwt.verify(
      token,
      process.env.JWT_SECRET!
    ) as JWTPayload;

    // Optional: Check if user still exists
    const user = await User.findById(decoded.id);
    if (!user) {
      return res.status(401).json({ error: 'User no longer exists' });
    }

    // Optional: Check if password changed after token issued
    if (user.passwordChangedAt) {
      const changedTimestamp = Math.floor(user.passwordChangedAt.getTime() / 1000);
      if (decoded.iat < changedTimestamp) {
        return res.status(401).json({ error: 'Password recently changed' });
      }
    }

    req.user = decoded;
    next();
  } catch (error) {
    if (error instanceof jwt.TokenExpiredError) {
      return res.status(401).json({ error: 'Token expired' });
    }
    if (error instanceof jwt.JsonWebTokenError) {
      return res.status(401).json({ error: 'Invalid token' });
    }
    next(error);
  }
};
```

### Role-based Authorization

```typescript
type Role = 'user' | 'moderator' | 'admin' | 'superadmin';

const requireRole = (...allowedRoles: Role[]) => {
  return (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Not authenticated' });
    }

    if (!allowedRoles.includes(req.user.role as Role)) {
      return res.status(403).json({
        error: 'Insufficient permissions',
        required: allowedRoles,
        current: req.user.role,
      });
    }

    next();
  };
};

// Usage
app.delete('/users/:id',
  authenticateJWT,
  requireRole('admin', 'superadmin'),
  async (req, res) => {
    await User.delete(req.params.id);
    res.status(204).send();
  }
);

// Hierarchical roles
const roleHierarchy: Record<Role, number> = {
  user: 1,
  moderator: 2,
  admin: 3,
  superadmin: 4,
};

const requireMinRole = (minRole: Role) => {
  return (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Not authenticated' });
    }

    const userLevel = roleHierarchy[req.user.role as Role] || 0;
    const requiredLevel = roleHierarchy[minRole];

    if (userLevel < requiredLevel) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }

    next();
  };
};
```

### API Key Authentication

```typescript
const authenticateApiKey = async (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  const apiKey = req.headers['x-api-key'] as string || req.query.apiKey as string;

  if (!apiKey) {
    return res.status(401).json({ error: 'API key required' });
  }

  try {
    const keyRecord = await ApiKey.findByKey(apiKey);

    if (!keyRecord) {
      return res.status(401).json({ error: 'Invalid API key' });
    }

    if (keyRecord.expiresAt && keyRecord.expiresAt < new Date()) {
      return res.status(401).json({ error: 'API key expired' });
    }

    // Track usage
    await ApiKey.incrementUsage(keyRecord.id);

    (req as any).apiKey = keyRecord;
    next();
  } catch (error) {
    next(error);
  }
};
```

### OAuth2/Session Authentication

```typescript
import passport from 'passport';
import { Strategy as GoogleStrategy } from 'passport-google-oauth20';

passport.use(new GoogleStrategy({
  clientID: process.env.GOOGLE_CLIENT_ID!,
  clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
  callbackURL: '/auth/google/callback',
}, async (accessToken, refreshToken, profile, done) => {
  try {
    let user = await User.findByGoogleId(profile.id);

    if (!user) {
      user = await User.create({
        googleId: profile.id,
        email: profile.emails?.[0].value,
        name: profile.displayName,
      });
    }

    done(null, user);
  } catch (error) {
    done(error);
  }
}));

passport.serializeUser((user: any, done) => {
  done(null, user.id);
});

passport.deserializeUser(async (id: string, done) => {
  try {
    const user = await User.findById(id);
    done(null, user);
  } catch (error) {
    done(error);
  }
});

// Middleware
app.use(passport.initialize());
app.use(passport.session());

// Routes
app.get('/auth/google',
  passport.authenticate('google', { scope: ['profile', 'email'] })
);

app.get('/auth/google/callback',
  passport.authenticate('google', { failureRedirect: '/login' }),
  (req, res) => {
    res.redirect('/dashboard');
  }
);

// Protected route
const ensureAuthenticated = (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  if (req.isAuthenticated()) {
    return next();
  }
  res.redirect('/login');
};

app.get('/dashboard', ensureAuthenticated, (req, res) => {
  res.render('dashboard', { user: req.user });
});
```

---

## 13. Validation Middleware

### Zod Validation

```typescript
import { z, ZodSchema, ZodError } from 'zod';

// Generic validator
const validate = <T extends ZodSchema>(schema: T) => {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      schema.parse({
        body: req.body,
        query: req.query,
        params: req.params,
      });
      next();
    } catch (error) {
      if (error instanceof ZodError) {
        return res.status(400).json({
          error: 'Validation failed',
          details: error.errors.map(err => ({
            path: err.path.join('.'),
            message: err.message,
          })),
        });
      }
      next(error);
    }
  };
};

// Schema definitions
const createUserSchema = z.object({
  body: z.object({
    name: z.string()
      .min(2, 'Name must be at least 2 characters')
      .max(100, 'Name must be at most 100 characters'),
    email: z.string()
      .email('Invalid email format'),
    password: z.string()
      .min(8, 'Password must be at least 8 characters')
      .regex(/[A-Z]/, 'Password must contain uppercase letter')
      .regex(/[a-z]/, 'Password must contain lowercase letter')
      .regex(/[0-9]/, 'Password must contain number'),
    age: z.number()
      .int('Age must be an integer')
      .min(18, 'Must be 18 or older')
      .optional(),
  }),
});

const updateUserSchema = z.object({
  params: z.object({
    id: z.string().uuid('Invalid user ID'),
  }),
  body: z.object({
    name: z.string().min(2).max(100).optional(),
    email: z.string().email().optional(),
  }),
});

// Usage
app.post('/users', validate(createUserSchema), async (req, res) => {
  const user = await User.create(req.body);
  res.status(201).json(user);
});

app.patch('/users/:id', validate(updateUserSchema), async (req, res) => {
  const user = await User.update(req.params.id, req.body);
  res.json(user);
});
```

### Joi Validation

```typescript
import Joi, { Schema, ValidationError } from 'joi';

const validateJoi = (schema: Schema, property: 'body' | 'query' | 'params' = 'body') => {
  return (req: Request, res: Response, next: NextFunction) => {
    const { error, value } = schema.validate(req[property], {
      abortEarly: false,
      stripUnknown: true,
    });

    if (error) {
      return res.status(400).json({
        error: 'Validation failed',
        details: error.details.map(detail => ({
          path: detail.path.join('.'),
          message: detail.message,
        })),
      });
    }

    req[property] = value;
    next();
  };
};

// Schema
const userSchema = Joi.object({
  name: Joi.string().min(2).max(100).required(),
  email: Joi.string().email().required(),
  password: Joi.string().min(8).pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/).required(),
  confirmPassword: Joi.ref('password'),
}).with('password', 'confirmPassword');

// Usage
app.post('/users', validateJoi(userSchema), async (req, res) => {
  const user = await User.create(req.body);
  res.status(201).json(user);
});
```

### Custom Validation Middleware

```typescript
interface ValidationRule {
  field: string;
  validate: (value: any) => boolean;
  message: string;
}

const customValidate = (rules: ValidationRule[]) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const errors: { field: string; message: string }[] = [];

    for (const rule of rules) {
      const value = rule.field.split('.').reduce((obj, key) => obj?.[key], req.body);

      if (!rule.validate(value)) {
        errors.push({ field: rule.field, message: rule.message });
      }
    }

    if (errors.length > 0) {
      return res.status(400).json({ error: 'Validation failed', details: errors });
    }

    next();
  };
};

// Usage
app.post('/products',
  customValidate([
    {
      field: 'name',
      validate: (v) => typeof v === 'string' && v.length >= 2,
      message: 'Name must be at least 2 characters',
    },
    {
      field: 'price',
      validate: (v) => typeof v === 'number' && v > 0,
      message: 'Price must be a positive number',
    },
    {
      field: 'category',
      validate: (v) => ['electronics', 'clothing', 'books'].includes(v),
      message: 'Invalid category',
    },
  ]),
  async (req, res) => {
    const product = await Product.create(req.body);
    res.status(201).json(product);
  }
);
```

### File Validation Middleware

```typescript
import multer from 'multer';

const upload = multer({
  storage: multer.memoryStorage(),
  limits: {
    fileSize: 5 * 1024 * 1024, // 5MB
    files: 5,
  },
  fileFilter: (req, file, cb) => {
    const allowedTypes = ['image/jpeg', 'image/png', 'image/gif'];

    if (!allowedTypes.includes(file.mimetype)) {
      return cb(new Error('Invalid file type'));
    }

    cb(null, true);
  },
});

// Single file
app.post('/avatar',
  upload.single('avatar'),
  (req, res) => {
    res.json({ file: req.file });
  }
);

// Multiple files
app.post('/gallery',
  upload.array('images', 10),
  (req, res) => {
    res.json({ files: req.files });
  }
);

// Multiple fields
app.post('/document',
  upload.fields([
    { name: 'document', maxCount: 1 },
    { name: 'attachments', maxCount: 5 },
  ]),
  (req, res) => {
    res.json({ files: req.files });
  }
);
```

---

## 14. Logging Middleware

### Structured Logging with Winston

```typescript
import winston from 'winston';
import { Request, Response, NextFunction } from 'express';

// Create logger
const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: 'api' },
  transports: [
    new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
    new winston.transports.File({ filename: 'logs/combined.log' }),
  ],
});

if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.combine(
      winston.format.colorize(),
      winston.format.simple()
    ),
  }));
}

// Logging middleware
const requestLogger = (req: Request, res: Response, next: NextFunction) => {
  const start = Date.now();
  const requestId = (req as any).id || 'unknown';

  // Log request
  logger.info('Incoming request', {
    requestId,
    method: req.method,
    url: req.originalUrl,
    ip: req.ip,
    userAgent: req.headers['user-agent'],
  });

  // Log response
  res.on('finish', () => {
    const duration = Date.now() - start;
    const logLevel = res.statusCode >= 400 ? 'error' : 'info';

    logger[logLevel]('Request completed', {
      requestId,
      method: req.method,
      url: req.originalUrl,
      statusCode: res.statusCode,
      duration: `${duration}ms`,
      contentLength: res.get('Content-Length'),
    });
  });

  next();
};

app.use(requestLogger);
```

### Pino Logger

```typescript
import pino from 'pino';
import pinoHttp from 'pino-http';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport: process.env.NODE_ENV === 'development'
    ? { target: 'pino-pretty' }
    : undefined,
});

// HTTP logging middleware
app.use(pinoHttp({
  logger,
  customLogLevel: (req, res, err) => {
    if (res.statusCode >= 500 || err) return 'error';
    if (res.statusCode >= 400) return 'warn';
    return 'info';
  },
  customSuccessMessage: (req, res) => {
    return `${req.method} ${req.url} completed`;
  },
  customErrorMessage: (req, res, err) => {
    return `${req.method} ${req.url} failed: ${err.message}`;
  },
  customAttributeKeys: {
    req: 'request',
    res: 'response',
    err: 'error',
    responseTime: 'duration',
  },
  serializers: {
    req: (req) => ({
      method: req.method,
      url: req.url,
      headers: {
        host: req.headers.host,
        'user-agent': req.headers['user-agent'],
      },
    }),
    res: (res) => ({
      statusCode: res.statusCode,
    }),
  },
  redact: ['req.headers.authorization', 'req.body.password'],
}));
```

### Audit Logging

```typescript
interface AuditLog {
  userId: string;
  action: string;
  resource: string;
  resourceId?: string;
  details?: any;
  ip: string;
  timestamp: Date;
}

const auditLogger = (action: string, resource: string) => {
  return async (req: Request, res: Response, next: NextFunction) => {
    const originalJson = res.json.bind(res);

    res.json = function(data: any) {
      // Log after successful response
      if (res.statusCode < 400) {
        const auditLog: AuditLog = {
          userId: (req as any).user?.id || 'anonymous',
          action,
          resource,
          resourceId: req.params.id,
          details: {
            method: req.method,
            path: req.path,
            body: req.body,
            response: data,
          },
          ip: req.ip || 'unknown',
          timestamp: new Date(),
        };

        // Save to database or send to audit service
        AuditService.log(auditLog).catch(console.error);
      }

      return originalJson(data);
    };

    next();
  };
};

// Usage
app.delete('/users/:id',
  authenticateJWT,
  requireRole('admin'),
  auditLogger('DELETE', 'user'),
  async (req, res) => {
    await User.delete(req.params.id);
    res.json({ deleted: true });
  }
);
```

---

## 15. Rate Limiting Middleware

### Express Rate Limit

```typescript
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import { createClient } from 'redis';

const redisClient = createClient({ url: process.env.REDIS_URL });
redisClient.connect();

// Global rate limiter
const globalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,                 // 100 requests per window
  message: {
    error: 'Too many requests',
    retryAfter: 900,        // seconds
  },
  standardHeaders: true,    // Return rate limit info in headers
  legacyHeaders: false,     // Disable X-RateLimit-* headers
  store: new RedisStore({
    sendCommand: (...args: string[]) => redisClient.sendCommand(args),
  }),
  keyGenerator: (req) => req.ip || 'unknown',
  skip: (req) => {
    // Skip rate limiting for health checks
    return req.path === '/health';
  },
  handler: (req, res, next, options) => {
    res.status(429).json({
      error: options.message,
      retryAfter: Math.ceil(options.windowMs / 1000),
    });
  },
});

app.use(globalLimiter);

// Strict limiter for authentication
const authLimiter = rateLimit({
  windowMs: 60 * 60 * 1000,  // 1 hour
  max: 5,                     // 5 attempts per hour
  message: { error: 'Too many login attempts' },
  skipSuccessfulRequests: true, // Don't count successful logins
});

app.post('/login', authLimiter, loginHandler);

// API-specific limiter
const apiLimiter = rateLimit({
  windowMs: 60 * 1000,       // 1 minute
  max: 60,                    // 60 requests per minute
  keyGenerator: (req) => {
    // Rate limit by API key if present, otherwise by IP
    return req.headers['x-api-key'] as string || req.ip || 'unknown';
  },
});

app.use('/api', apiLimiter);
```

### Sliding Window Rate Limiter

```typescript
interface SlidingWindowOptions {
  windowMs: number;
  maxRequests: number;
}

const slidingWindowLimiter = (options: SlidingWindowOptions) => {
  const requests = new Map<string, number[]>();

  return (req: Request, res: Response, next: NextFunction) => {
    const key = req.ip || 'unknown';
    const now = Date.now();
    const windowStart = now - options.windowMs;

    // Get existing timestamps for this key
    let timestamps = requests.get(key) || [];

    // Filter out old timestamps
    timestamps = timestamps.filter(ts => ts > windowStart);

    // Check if limit exceeded
    if (timestamps.length >= options.maxRequests) {
      const oldestRequest = timestamps[0];
      const retryAfter = Math.ceil((oldestRequest + options.windowMs - now) / 1000);

      res.set('Retry-After', String(retryAfter));
      return res.status(429).json({
        error: 'Too many requests',
        retryAfter,
      });
    }

    // Add current timestamp
    timestamps.push(now);
    requests.set(key, timestamps);

    // Set rate limit headers
    res.set('X-RateLimit-Limit', String(options.maxRequests));
    res.set('X-RateLimit-Remaining', String(options.maxRequests - timestamps.length));
    res.set('X-RateLimit-Reset', String(Math.ceil((now + options.windowMs) / 1000)));

    next();
  };
};

app.use(slidingWindowLimiter({
  windowMs: 60 * 1000,
  maxRequests: 100,
}));
```

### Token Bucket Rate Limiter

```typescript
interface TokenBucket {
  tokens: number;
  lastRefill: number;
}

interface TokenBucketOptions {
  bucketSize: number;        // Maximum tokens
  refillRate: number;        // Tokens per second
  tokensPerRequest?: number; // Tokens consumed per request
}

const tokenBucketLimiter = (options: TokenBucketOptions) => {
  const buckets = new Map<string, TokenBucket>();
  const tokensPerRequest = options.tokensPerRequest || 1;

  return (req: Request, res: Response, next: NextFunction) => {
    const key = req.ip || 'unknown';
    const now = Date.now();

    let bucket = buckets.get(key);

    if (!bucket) {
      bucket = { tokens: options.bucketSize, lastRefill: now };
      buckets.set(key, bucket);
    }

    // Refill tokens based on time elapsed
    const elapsed = (now - bucket.lastRefill) / 1000;
    const refillAmount = elapsed * options.refillRate;
    bucket.tokens = Math.min(options.bucketSize, bucket.tokens + refillAmount);
    bucket.lastRefill = now;

    // Check if enough tokens
    if (bucket.tokens < tokensPerRequest) {
      const waitTime = (tokensPerRequest - bucket.tokens) / options.refillRate;
      res.set('Retry-After', String(Math.ceil(waitTime)));
      return res.status(429).json({
        error: 'Rate limit exceeded',
        retryAfter: Math.ceil(waitTime),
      });
    }

    // Consume tokens
    bucket.tokens -= tokensPerRequest;

    // Set headers
    res.set('X-RateLimit-Limit', String(options.bucketSize));
    res.set('X-RateLimit-Remaining', String(Math.floor(bucket.tokens)));

    next();
  };
};

app.use(tokenBucketLimiter({
  bucketSize: 100,
  refillRate: 10, // 10 tokens per second
}));
```

---

## 16. TypeScript Middleware Types

### Basic Types

```typescript
import { Request, Response, NextFunction, RequestHandler, ErrorRequestHandler } from 'express';

// Standard middleware type
type Middleware = (req: Request, res: Response, next: NextFunction) => void;

// Async middleware type
type AsyncMiddleware = (req: Request, res: Response, next: NextFunction) => Promise<void>;

// Error middleware type
type ErrorMiddleware = (err: Error, req: Request, res: Response, next: NextFunction) => void;

// Using built-in types
const middleware: RequestHandler = (req, res, next) => {
  next();
};

const errorHandler: ErrorRequestHandler = (err, req, res, next) => {
  res.status(500).json({ error: err.message });
};
```

### Extended Request Types

```typescript
import { Request, Response, NextFunction } from 'express';

// Extend Request with custom properties
interface AuthenticatedRequest extends Request {
  user: {
    id: string;
    email: string;
    role: string;
  };
}

interface PaginatedRequest extends Request {
  pagination: {
    page: number;
    limit: number;
    offset: number;
  };
}

interface RequestWithFile extends Request {
  file: Express.Multer.File;
  files: Express.Multer.File[];
}

// Combine interfaces
interface FullRequest extends AuthenticatedRequest, PaginatedRequest {
  requestId: string;
}

// Use in middleware
const authMiddleware = (
  req: Request,
  res: Response,
  next: NextFunction
): void => {
  // After verification, cast to AuthenticatedRequest
  (req as AuthenticatedRequest).user = {
    id: '123',
    email: 'user@example.com',
    role: 'admin',
  };
  next();
};

// Use in route handler
const getUser = (req: AuthenticatedRequest, res: Response): void => {
  res.json({ user: req.user });
};
```

### Generic Middleware Types

```typescript
// Generic middleware factory type
type MiddlewareFactory<T> = (options: T) => RequestHandler;

// Example implementation
interface CacheOptions {
  ttl: number;
  key?: string;
}

const cacheMiddleware: MiddlewareFactory<CacheOptions> = (options) => {
  return (req, res, next) => {
    // Implementation
    next();
  };
};

// Generic validation middleware
type ValidationMiddleware<T> = (
  schema: T
) => RequestHandler;

const validate: ValidationMiddleware<ZodSchema> = (schema) => {
  return (req, res, next) => {
    // Implementation
    next();
  };
};
```

### Type-safe Middleware Chain

```typescript
import { Router, RequestHandler } from 'express';

// Type-safe route builder
class TypedRouter {
  private router: Router;

  constructor() {
    this.router = Router();
  }

  get<TReq extends Request = Request>(
    path: string,
    ...handlers: Array<(req: TReq, res: Response, next: NextFunction) => void>
  ) {
    this.router.get(path, ...(handlers as RequestHandler[]));
    return this;
  }

  post<TReq extends Request = Request>(
    path: string,
    ...handlers: Array<(req: TReq, res: Response, next: NextFunction) => void>
  ) {
    this.router.post(path, ...(handlers as RequestHandler[]));
    return this;
  }

  getRouter() {
    return this.router;
  }
}

// Usage
const typedRouter = new TypedRouter();

typedRouter.get<AuthenticatedRequest>(
  '/profile',
  authMiddleware,
  (req, res) => {
    // req.user is properly typed
    res.json({ user: req.user });
  }
);
```

### Module Augmentation

```typescript
// types/express.d.ts
import { User } from '../models/User';

declare global {
  namespace Express {
    interface Request {
      user?: User;
      requestId: string;
      startTime: number;
    }

    interface Response {
      success: (data: any, statusCode?: number) => void;
      error: (message: string, statusCode?: number) => void;
    }
  }
}

export {};

// Now these properties are available on all Request/Response objects
const middleware = (req: Request, res: Response, next: NextFunction) => {
  req.requestId = uuidv4();
  req.startTime = Date.now();

  res.success = (data, statusCode = 200) => {
    res.status(statusCode).json({ success: true, data });
  };

  res.error = (message, statusCode = 500) => {
    res.status(statusCode).json({ success: false, error: message });
  };

  next();
};
```

---

## 17. Best Practices

### 1. Always Call next() or Send Response

```typescript
// BAD - Request hangs
const badMiddleware = (req: Request, res: Response, next: NextFunction) => {
  console.log('Logging...');
  // Missing next() call
};

// GOOD - Calls next
const goodMiddleware = (req: Request, res: Response, next: NextFunction) => {
  console.log('Logging...');
  next();
};

// GOOD - Sends response
const responseMiddleware = (req: Request, res: Response, next: NextFunction) => {
  if (!req.user) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next();
};
```

### 2. Use Async Error Handling

```typescript
// BAD - Errors in async code are not caught
app.get('/users', async (req, res, next) => {
  const users = await User.findAll(); // If this throws, error is not handled
  res.json(users);
});

// GOOD - Using try/catch
app.get('/users', async (req, res, next) => {
  try {
    const users = await User.findAll();
    res.json(users);
  } catch (error) {
    next(error);
  }
});

// GOOD - Using async wrapper
const asyncHandler = (fn: AsyncHandler) => (req: Request, res: Response, next: NextFunction) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

app.get('/users', asyncHandler(async (req, res) => {
  const users = await User.findAll();
  res.json(users);
}));
```

### 3. Order Middleware Correctly

```typescript
// Correct middleware order
const app = express();

// 1. Security middleware first
app.use(helmet());
app.use(cors());

// 2. Request parsing
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// 3. Logging
app.use(morgan('combined'));

// 4. Static files (before authentication if public)
app.use('/public', express.static('public'));

// 5. Rate limiting
app.use(rateLimiter);

// 6. Authentication (if global)
app.use(authenticateJWT);

// 7. Routes
app.use('/api', apiRoutes);

// 8. 404 handler
app.use(notFoundHandler);

// 9. Error handler (MUST be last)
app.use(errorHandler);
```

### 4. Keep Middleware Focused

```typescript
// BAD - Middleware doing too much
const doEverything = (req: Request, res: Response, next: NextFunction) => {
  // Logging
  console.log(req.method, req.url);

  // Authentication
  const token = req.headers.authorization;
  // ... verify token

  // Rate limiting
  // ... check rate limit

  // Input sanitization
  // ... sanitize input

  next();
};

// GOOD - Single responsibility
const logger = (req: Request, res: Response, next: NextFunction) => {
  console.log(req.method, req.url);
  next();
};

const authenticate = (req: Request, res: Response, next: NextFunction) => {
  // Only authentication logic
  next();
};

const rateLimit = (req: Request, res: Response, next: NextFunction) => {
  // Only rate limiting logic
  next();
};

// Compose middleware
app.use(logger);
app.use(authenticate);
app.use(rateLimit);
```

### 5. Use Middleware Factories for Configuration

```typescript
// BAD - Hardcoded values
const loggerMiddleware = (req: Request, res: Response, next: NextFunction) => {
  console.log(`[INFO] ${req.method} ${req.url}`);
  next();
};

// GOOD - Configurable via factory
interface LoggerOptions {
  level: 'debug' | 'info' | 'warn' | 'error';
  format: 'json' | 'text';
}

const createLogger = (options: LoggerOptions) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const log = { level: options.level, method: req.method, url: req.url };

    if (options.format === 'json') {
      console.log(JSON.stringify(log));
    } else {
      console.log(`[${options.level.toUpperCase()}] ${req.method} ${req.url}`);
    }

    next();
  };
};

app.use(createLogger({ level: 'info', format: 'json' }));
```

### 6. Handle Errors Gracefully

```typescript
// Comprehensive error handling
const errorHandler = (
  err: any,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  // Log error
  console.error('Error:', {
    message: err.message,
    stack: err.stack,
    path: req.path,
    method: req.method,
  });

  // Don't leak error details in production
  const response = {
    error: process.env.NODE_ENV === 'production'
      ? 'Internal server error'
      : err.message,
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack }),
  };

  // Set appropriate status code
  const statusCode = err.statusCode || err.status || 500;

  res.status(statusCode).json(response);
};
```

### 7. Avoid Modifying Request/Response Prototypes

```typescript
// BAD - Modifying prototypes
Request.prototype.getUser = function() {
  return this.user;
};

// GOOD - Add properties to individual requests
const addHelpers = (req: Request, res: Response, next: NextFunction) => {
  (req as any).getUser = () => (req as any).user;
  next();
};

// BETTER - Use type augmentation
declare global {
  namespace Express {
    interface Request {
      getUser(): User | undefined;
    }
  }
}
```

### 8. Test Middleware Independently

```typescript
import { Request, Response, NextFunction } from 'express';
import { authMiddleware } from './middleware/auth';

describe('Auth Middleware', () => {
  let mockRequest: Partial<Request>;
  let mockResponse: Partial<Response>;
  let nextFunction: NextFunction;

  beforeEach(() => {
    mockRequest = {
      headers: {},
    };
    mockResponse = {
      status: jest.fn().mockReturnThis(),
      json: jest.fn(),
    };
    nextFunction = jest.fn();
  });

  it('should return 401 if no token provided', () => {
    authMiddleware(
      mockRequest as Request,
      mockResponse as Response,
      nextFunction
    );

    expect(mockResponse.status).toHaveBeenCalledWith(401);
    expect(mockResponse.json).toHaveBeenCalledWith({
      error: 'No token provided',
    });
    expect(nextFunction).not.toHaveBeenCalled();
  });

  it('should call next with valid token', async () => {
    mockRequest.headers = {
      authorization: 'Bearer valid-token',
    };

    await authMiddleware(
      mockRequest as Request,
      mockResponse as Response,
      nextFunction
    );

    expect(nextFunction).toHaveBeenCalled();
    expect(mockRequest).toHaveProperty('user');
  });
});
```

### 9. Use Environment-specific Configuration

```typescript
// config/middleware.ts
interface MiddlewareConfig {
  compression: boolean;
  helmet: boolean;
  morgan: {
    format: string;
    skip?: boolean;
  };
  rateLimit: {
    windowMs: number;
    max: number;
  };
}

const config: Record<string, MiddlewareConfig> = {
  development: {
    compression: false,
    helmet: false,
    morgan: { format: 'dev' },
    rateLimit: { windowMs: 60000, max: 1000 },
  },
  production: {
    compression: true,
    helmet: true,
    morgan: { format: 'combined', skip: true },
    rateLimit: { windowMs: 60000, max: 100 },
  },
  test: {
    compression: false,
    helmet: false,
    morgan: { format: 'tiny' },
    rateLimit: { windowMs: 60000, max: 10000 },
  },
};

export const middlewareConfig = config[process.env.NODE_ENV || 'development'];
```

### 10. Document Your Middleware

```typescript
/**
 * Authentication middleware that validates JWT tokens
 *
 * @description
 * Extracts the JWT token from the Authorization header (Bearer scheme),
 * validates it against the JWT_SECRET environment variable, and attaches
 * the decoded user information to the request object.
 *
 * @example
 * // Protected route
 * app.get('/profile', authenticateJWT, profileHandler);
 *
 * @example
 * // Protected router
 * router.use(authenticateJWT);
 *
 * @throws {401} When no token is provided
 * @throws {401} When token is invalid or expired
 *
 * @requires process.env.JWT_SECRET - The secret key for JWT verification
 *
 * @see https://jwt.io/ for JWT documentation
 */
const authenticateJWT = async (
  req: AuthenticatedRequest,
  res: Response,
  next: NextFunction
): Promise<void> => {
  // Implementation
};
```

---

## Quick Reference

### Middleware Types Summary

| Type | Signature | Purpose |
|------|-----------|---------|
| Application-level | `app.use(fn)` | Global middleware |
| Router-level | `router.use(fn)` | Router-specific middleware |
| Error-handling | `(err, req, res, next)` | Error processing |
| Built-in | `express.json()` | Body parsing, static files |
| Third-party | `helmet()`, `cors()` | Security, functionality |

### Common HTTP Status Codes

| Code | Meaning | When to Use |
|------|---------|-------------|
| 200 | OK | Successful GET/PUT |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Validation errors |
| 401 | Unauthorized | Missing/invalid auth |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource not found |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server errors |

### Essential Middleware Stack

```typescript
app.use(helmet());                    // Security headers
app.use(cors(corsOptions));           // CORS
app.use(compression());               // Compression
app.use(morgan('combined'));          // Logging
app.use(express.json());              // JSON parsing
app.use(express.urlencoded());        // Form parsing
app.use(cookieParser());              // Cookie parsing
app.use(rateLimiter);                 // Rate limiting
app.use('/api', apiRouter);           // Routes
app.use(notFoundHandler);             // 404
app.use(errorHandler);                // Errors
```

---

## Additional Resources

- [Express.js Official Documentation](https://expressjs.com/)
- [Express Middleware Guide](https://expressjs.com/en/guide/using-middleware.html)
- [Express Error Handling](https://expressjs.com/en/guide/error-handling.html)
- [Security Best Practices](https://expressjs.com/en/advanced/best-practice-security.html)
- [Performance Best Practices](https://expressjs.com/en/advanced/best-practice-performance.html)
