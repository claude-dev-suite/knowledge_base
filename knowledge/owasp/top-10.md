# OWASP Top 10 Security Vulnerabilities

Comprehensive guide to the OWASP Top 10 web application security risks with prevention strategies.

**Official Documentation:** https://owasp.org/Top10/

---

## Table of Contents

1. [A01 - Broken Access Control](#a01---broken-access-control)
2. [A02 - Cryptographic Failures](#a02---cryptographic-failures)
3. [A03 - Injection](#a03---injection)
4. [A04 - Insecure Design](#a04---insecure-design)
5. [A05 - Security Misconfiguration](#a05---security-misconfiguration)
6. [A06 - Vulnerable Components](#a06---vulnerable-and-outdated-components)
7. [A07 - Authentication Failures](#a07---identification-and-authentication-failures)
8. [A08 - Software and Data Integrity](#a08---software-and-data-integrity-failures)
9. [A09 - Security Logging Failures](#a09---security-logging-and-monitoring-failures)
10. [A10 - Server-Side Request Forgery](#a10---server-side-request-forgery-ssrf)

---

## A01 - Broken Access Control

Access control enforces policy such that users cannot act outside their intended permissions.

### Common Vulnerabilities

- Bypassing access control checks by modifying URL or API requests
- Insecure Direct Object References (IDOR)
- Missing access control for POST, PUT, DELETE
- Privilege escalation (acting as admin when logged in as user)
- CORS misconfiguration

### Prevention

#### Authorization Middleware

```typescript
// Express.js - Role-based access control
interface User {
  id: string;
  role: 'user' | 'admin' | 'moderator';
  permissions: string[];
}

// Middleware factory
const requireRole = (...roles: string[]) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const user = req.user as User;

    if (!user) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    if (!roles.includes(user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }

    next();
  };
};

// Permission-based
const requirePermission = (permission: string) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const user = req.user as User;

    if (!user?.permissions?.includes(permission)) {
      return res.status(403).json({ error: 'Permission denied' });
    }

    next();
  };
};

// Usage
app.delete('/users/:id',
  requireRole('admin'),
  deleteUserHandler
);

app.post('/posts',
  requirePermission('posts:create'),
  createPostHandler
);
```

#### Prevent IDOR

```typescript
// ❌ Vulnerable - No ownership check
app.get('/documents/:id', async (req, res) => {
  const doc = await Document.findById(req.params.id);
  res.json(doc);
});

// ✅ Secure - Verify ownership
app.get('/documents/:id', async (req, res) => {
  const doc = await Document.findOne({
    _id: req.params.id,
    userId: req.user.id  // Ensure user owns the document
  });

  if (!doc) {
    return res.status(404).json({ error: 'Document not found' });
  }

  res.json(doc);
});
```

#### Spring Security Example

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/user/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .csrf(csrf -> csrf.csrfTokenRepository(
                CookieCsrfTokenRepository.withHttpOnlyFalse()
            ));
        return http.build();
    }
}

// Method-level security
@Service
public class DocumentService {

    @PreAuthorize("hasRole('ADMIN') or @documentSecurity.isOwner(#id)")
    public Document getDocument(Long id) {
        return documentRepository.findById(id)
            .orElseThrow(() -> new NotFoundException("Document not found"));
    }

    @PostAuthorize("returnObject.userId == authentication.principal.id")
    public Document findDocument(Long id) {
        return documentRepository.findById(id).orElse(null);
    }
}

@Component("documentSecurity")
public class DocumentSecurity {

    public boolean isOwner(Long documentId) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        Document doc = documentRepository.findById(documentId).orElse(null);
        return doc != null && doc.getUserId().equals(auth.getName());
    }
}
```

---

## A02 - Cryptographic Failures

Failures related to cryptography which often lead to sensitive data exposure.

### Common Vulnerabilities

- Transmitting data in clear text (HTTP, SMTP, FTP)
- Using weak cryptographic algorithms (MD5, SHA1, DES)
- Using hard-coded or weak encryption keys
- Not enforcing encryption with security headers
- Improper certificate validation

### Prevention

#### Secure Password Hashing

```typescript
// Node.js with bcrypt
import bcrypt from 'bcrypt';

const SALT_ROUNDS = 12;

export async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS);
}

export async function verifyPassword(
  password: string,
  hash: string
): Promise<boolean> {
  return bcrypt.compare(password, hash);
}

// Usage
const hashedPassword = await hashPassword('userPassword123');
const isValid = await verifyPassword('userPassword123', hashedPassword);
```

```python
# Python with argon2
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

ph = PasswordHasher(
    time_cost=3,
    memory_cost=65536,
    parallelism=4
)

def hash_password(password: str) -> str:
    return ph.hash(password)

def verify_password(password: str, hash: str) -> bool:
    try:
        ph.verify(hash, password)
        return True
    except VerifyMismatchError:
        return False
```

#### Encryption at Rest

```typescript
import crypto from 'crypto';

const ALGORITHM = 'aes-256-gcm';
const IV_LENGTH = 16;
const AUTH_TAG_LENGTH = 16;

export function encrypt(text: string, key: Buffer): string {
  const iv = crypto.randomBytes(IV_LENGTH);
  const cipher = crypto.createCipheriv(ALGORITHM, key, iv);

  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');

  const authTag = cipher.getAuthTag();

  // Return IV + AuthTag + Encrypted data
  return iv.toString('hex') + authTag.toString('hex') + encrypted;
}

export function decrypt(encryptedData: string, key: Buffer): string {
  const iv = Buffer.from(encryptedData.slice(0, IV_LENGTH * 2), 'hex');
  const authTag = Buffer.from(
    encryptedData.slice(IV_LENGTH * 2, (IV_LENGTH + AUTH_TAG_LENGTH) * 2),
    'hex'
  );
  const encrypted = encryptedData.slice((IV_LENGTH + AUTH_TAG_LENGTH) * 2);

  const decipher = crypto.createDecipheriv(ALGORITHM, key, iv);
  decipher.setAuthTag(authTag);

  let decrypted = decipher.update(encrypted, 'hex', 'utf8');
  decrypted += decipher.final('utf8');

  return decrypted;
}

// Key derivation from password
export function deriveKey(password: string, salt: Buffer): Buffer {
  return crypto.pbkdf2Sync(password, salt, 100000, 32, 'sha256');
}
```

#### Security Headers

```typescript
// Express.js with Helmet
import helmet from 'helmet';

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'", "https://api.example.com"],
      fontSrc: ["'self'"],
      objectSrc: ["'none'"],
      mediaSrc: ["'self'"],
      frameSrc: ["'none'"],
    },
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  },
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' }
}));

// Force HTTPS
app.use((req, res, next) => {
  if (req.headers['x-forwarded-proto'] !== 'https' && process.env.NODE_ENV === 'production') {
    return res.redirect(301, `https://${req.hostname}${req.url}`);
  }
  next();
});
```

---

## A03 - Injection

Injection flaws occur when untrusted data is sent to an interpreter as part of a command or query.

### Common Vulnerabilities

- SQL Injection
- NoSQL Injection
- Command Injection
- LDAP Injection
- XPath Injection
- Template Injection

### Prevention

#### SQL Injection Prevention

```typescript
// ❌ Vulnerable
const query = `SELECT * FROM users WHERE email = '${email}'`;

// ✅ Parameterized Query (node-postgres)
const result = await pool.query(
  'SELECT * FROM users WHERE email = $1',
  [email]
);

// ✅ Using ORM (Prisma)
const user = await prisma.user.findUnique({
  where: { email }
});

// ✅ Using ORM (TypeORM)
const user = await userRepository.findOne({
  where: { email }
});

// ✅ Query Builder with parameters
const users = await dataSource
  .createQueryBuilder("user")
  .where("user.email = :email", { email })
  .andWhere("user.status = :status", { status: 'active' })
  .getMany();
```

```python
# ❌ Vulnerable
cursor.execute(f"SELECT * FROM users WHERE email = '{email}'")

# ✅ Parameterized Query
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))

# ✅ SQLAlchemy ORM
user = session.query(User).filter(User.email == email).first()

# ✅ SQLAlchemy Core
stmt = select(users).where(users.c.email == email)
result = connection.execute(stmt)
```

```java
// ❌ Vulnerable
String query = "SELECT * FROM users WHERE email = '" + email + "'";

// ✅ PreparedStatement
PreparedStatement stmt = conn.prepareStatement(
    "SELECT * FROM users WHERE email = ?"
);
stmt.setString(1, email);

// ✅ JPA Named Parameters
@Query("SELECT u FROM User u WHERE u.email = :email")
User findByEmail(@Param("email") String email);

// ✅ Criteria API
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<User> cq = cb.createQuery(User.class);
Root<User> root = cq.from(User.class);
cq.where(cb.equal(root.get("email"), email));
```

#### NoSQL Injection Prevention

```typescript
// ❌ Vulnerable (MongoDB)
const user = await db.collection('users').findOne({
  email: req.body.email,
  password: req.body.password  // Could be { $gt: '' }
});

// ✅ Type validation
import { z } from 'zod';

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8)
});

const { email, password } = loginSchema.parse(req.body);

// Now safe to use
const user = await db.collection('users').findOne({
  email,
  password: await hashPassword(password)
});

// ✅ Mongoose with schema
const userSchema = new Schema({
  email: { type: String, required: true },
  password: { type: String, required: true }
});

const user = await User.findOne({ email });
```

#### Command Injection Prevention

```typescript
// ❌ Vulnerable
const { exec } = require('child_process');
exec(`ls ${userInput}`, callback);

// ✅ Use spawn with arguments array
import { spawn } from 'child_process';

const ls = spawn('ls', ['-la', sanitizedPath]);

// ✅ Validate and sanitize input
import path from 'path';

function safeListDirectory(userPath: string): void {
  // Resolve to absolute path and check it's within allowed directory
  const resolvedPath = path.resolve('/allowed/base', userPath);

  if (!resolvedPath.startsWith('/allowed/base')) {
    throw new Error('Path traversal attempt detected');
  }

  const ls = spawn('ls', ['-la', resolvedPath]);
}
```

#### XSS Prevention

```typescript
// React - Automatic escaping
function UserProfile({ user }) {
  return <div>{user.name}</div>; // Safe - React escapes
}

// ❌ Dangerous
function Comment({ html }) {
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}

// ✅ Sanitize if HTML is needed
import DOMPurify from 'dompurify';

function Comment({ html }) {
  const sanitized = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a'],
    ALLOWED_ATTR: ['href']
  });
  return <div dangerouslySetInnerHTML={{ __html: sanitized }} />;
}

// Server-side output encoding
import { encode } from 'html-entities';

app.get('/search', (req, res) => {
  const query = encode(req.query.q);
  res.send(`<h1>Results for: ${query}</h1>`);
});
```

---

## A04 - Insecure Design

Insecure design represents weaknesses in design patterns and architecture that make systems vulnerable.

### Common Vulnerabilities

- Missing rate limiting
- Lack of input validation at design level
- Trust boundaries not defined
- Missing defense in depth
- Insufficient business logic protection

### Prevention

#### Rate Limiting

```typescript
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import Redis from 'ioredis';

const redisClient = new Redis(process.env.REDIS_URL);

// General API rate limit
const apiLimiter = rateLimit({
  store: new RedisStore({
    sendCommand: (...args: string[]) => redisClient.call(...args),
  }),
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,
  message: { error: 'Too many requests, please try again later' },
  standardHeaders: true,
  legacyHeaders: false,
});

// Strict limit for authentication
const authLimiter = rateLimit({
  store: new RedisStore({
    sendCommand: (...args: string[]) => redisClient.call(...args),
  }),
  windowMs: 60 * 60 * 1000, // 1 hour
  max: 5,
  message: { error: 'Too many login attempts' },
  keyGenerator: (req) => req.body.email || req.ip,
});

app.use('/api', apiLimiter);
app.post('/api/auth/login', authLimiter, loginHandler);
```

#### Input Validation Schema

```typescript
import { z } from 'zod';

// Define strict schemas
const createUserSchema = z.object({
  email: z.string().email().max(255),
  password: z.string()
    .min(12, 'Password must be at least 12 characters')
    .regex(/[A-Z]/, 'Must contain uppercase')
    .regex(/[a-z]/, 'Must contain lowercase')
    .regex(/[0-9]/, 'Must contain number')
    .regex(/[^A-Za-z0-9]/, 'Must contain special character'),
  name: z.string().min(2).max(100).regex(/^[\w\s-]+$/),
  role: z.enum(['user', 'moderator']).default('user'),
  age: z.number().int().min(18).max(120).optional(),
});

// Validation middleware
const validate = <T>(schema: z.ZodSchema<T>) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.body);

    if (!result.success) {
      return res.status(400).json({
        error: 'Validation failed',
        details: result.error.issues
      });
    }

    req.body = result.data;
    next();
  };
};

app.post('/users', validate(createUserSchema), createUserHandler);
```

#### Business Logic Protection

```typescript
// Order processing with proper checks
class OrderService {
  async createOrder(userId: string, items: OrderItem[]): Promise<Order> {
    // 1. Validate user exists and is active
    const user = await this.userService.findActive(userId);
    if (!user) throw new ForbiddenError('User not authorized');

    // 2. Validate items exist and are in stock
    for (const item of items) {
      const product = await this.productService.findById(item.productId);
      if (!product) throw new NotFoundError(`Product ${item.productId} not found`);
      if (product.stock < item.quantity) {
        throw new BusinessError(`Insufficient stock for ${product.name}`);
      }
    }

    // 3. Calculate total server-side (never trust client)
    const total = await this.calculateTotal(items);

    // 4. Validate against spending limits
    const dailySpent = await this.getDailySpending(userId);
    if (dailySpent + total > user.dailyLimit) {
      throw new BusinessError('Daily spending limit exceeded');
    }

    // 5. Create order within transaction
    return this.db.transaction(async (tx) => {
      // Reserve stock
      for (const item of items) {
        await tx.product.update({
          where: { id: item.productId },
          data: { stock: { decrement: item.quantity } }
        });
      }

      // Create order
      return tx.order.create({
        data: { userId, items, total, status: 'pending' }
      });
    });
  }
}
```

---

## A05 - Security Misconfiguration

Security misconfiguration is the most common vulnerability, often resulting from default configurations.

### Common Vulnerabilities

- Default credentials not changed
- Unnecessary features enabled
- Error messages revealing sensitive info
- Missing security headers
- Outdated software
- Insecure cloud service permissions

### Prevention

#### Environment Configuration

```typescript
// config/security.ts
export const securityConfig = {
  development: {
    cors: {
      origin: '*',
      credentials: false
    },
    rateLimit: { max: 1000 },
    logLevel: 'debug'
  },
  production: {
    cors: {
      origin: process.env.ALLOWED_ORIGINS?.split(',') || [],
      credentials: true,
      methods: ['GET', 'POST', 'PUT', 'DELETE'],
      allowedHeaders: ['Content-Type', 'Authorization']
    },
    rateLimit: { max: 100 },
    logLevel: 'warn'
  }
};

// Secure error handling
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  // Log full error internally
  logger.error({
    message: err.message,
    stack: err.stack,
    url: req.url,
    method: req.method,
    userId: req.user?.id
  });

  // Return sanitized error to client
  if (process.env.NODE_ENV === 'production') {
    res.status(500).json({
      error: 'Internal server error',
      requestId: req.id
    });
  } else {
    res.status(500).json({
      error: err.message,
      stack: err.stack
    });
  }
});
```

#### Docker Security Configuration

```dockerfile
# Use specific version, not latest
FROM node:20-alpine

# Run as non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Set secure permissions
WORKDIR /app
COPY --chown=nodejs:nodejs package*.json ./
RUN npm ci --only=production

COPY --chown=nodejs:nodejs . .

# Use non-root user
USER nodejs

# Don't expose sensitive ports
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/index.js"]
```

```yaml
# docker-compose.yml security
services:
  api:
    image: myapp:latest
    read_only: true
    tmpfs:
      - /tmp
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M
```

#### Secure Headers Configuration

```nginx
# Nginx security headers
server {
    # SSL Configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers on;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_stapling on;
    ssl_stapling_verify on;

    # Security Headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

    # Hide server version
    server_tokens off;

    # Disable unnecessary methods
    if ($request_method !~ ^(GET|HEAD|POST|PUT|DELETE)$) {
        return 405;
    }
}
```

---

## A06 - Vulnerable and Outdated Components

Using components with known vulnerabilities can undermine application defenses.

### Prevention

#### Dependency Scanning

```yaml
# GitHub Actions workflow
name: Security Scan

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 0 * * *'

jobs:
  npm-audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run npm audit
        run: npm audit --audit-level=high

      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

  trivy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Trivy
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          severity: 'CRITICAL,HIGH'
```

#### Dependency Management

```json
// package.json - Pin versions
{
  "dependencies": {
    "express": "4.18.2",
    "lodash": "4.17.21"
  },
  "overrides": {
    "minimist": "1.2.8"
  }
}
```

```python
# requirements.txt - Pin versions
Django==4.2.7
requests==2.31.0
cryptography==41.0.5

# Or use pip-compile for hash verification
# requirements.txt (generated)
django==4.2.7 \
    --hash=sha256:xxx
```

#### Automated Updates

```yaml
# Renovate configuration
# renovate.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:base",
    ":semanticCommits",
    "group:allNonMajor"
  ],
  "vulnerabilityAlerts": {
    "enabled": true,
    "labels": ["security"]
  },
  "packageRules": [
    {
      "matchUpdateTypes": ["patch", "minor"],
      "automerge": true,
      "automergeType": "pr"
    },
    {
      "matchDepTypes": ["devDependencies"],
      "automerge": true
    }
  ]
}
```

---

## A07 - Identification and Authentication Failures

Confirmation of user identity and session management is critical to protect against authentication attacks.

### Common Vulnerabilities

- Weak password requirements
- Credential stuffing vulnerabilities
- Session fixation
- Missing MFA
- Exposing session IDs in URLs

### Prevention

#### Secure Authentication

```typescript
import { Argon2id } from 'oslo/password';
import { generateId } from 'lucia';

const argon2id = new Argon2id();

export async function signup(email: string, password: string) {
  // Validate password strength
  if (!isStrongPassword(password)) {
    throw new ValidationError('Password does not meet requirements');
  }

  // Check for breached passwords
  const isBreached = await checkHaveIBeenPwned(password);
  if (isBreached) {
    throw new ValidationError('This password has been compromised in a data breach');
  }

  // Hash password with Argon2id
  const hashedPassword = await argon2id.hash(password);

  const userId = generateId(15);

  await db.user.create({
    data: {
      id: userId,
      email: email.toLowerCase(),
      hashedPassword,
      emailVerified: false
    }
  });

  // Send verification email
  await sendVerificationEmail(email, userId);

  return { userId };
}

export async function login(email: string, password: string, ip: string) {
  // Rate limiting check
  const attempts = await getLoginAttempts(email, ip);
  if (attempts > 5) {
    throw new TooManyRequestsError('Too many login attempts');
  }

  const user = await db.user.findUnique({
    where: { email: email.toLowerCase() }
  });

  // Constant-time comparison even if user doesn't exist
  const validPassword = user
    ? await argon2id.verify(user.hashedPassword, password)
    : false;

  if (!user || !validPassword) {
    await recordFailedAttempt(email, ip);
    throw new AuthenticationError('Invalid credentials');
  }

  // Clear failed attempts on success
  await clearLoginAttempts(email, ip);

  // Create session
  const session = await lucia.createSession(user.id, {
    ip,
    userAgent: req.headers['user-agent']
  });

  return session;
}
```

#### Session Management

```typescript
import { Lucia } from 'lucia';
import { PrismaAdapter } from '@lucia-auth/adapter-prisma';

const adapter = new PrismaAdapter(prisma.session, prisma.user);

export const lucia = new Lucia(adapter, {
  sessionExpiresIn: new TimeSpan(30, 'd'),
  sessionCookie: {
    name: '__Host-session',  // Use __Host- prefix for security
    attributes: {
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      httpOnly: true,
      path: '/'
    }
  },
  getUserAttributes: (attributes) => ({
    email: attributes.email,
    role: attributes.role
  })
});

// Session validation middleware
export async function validateSession(req: Request, res: Response, next: NextFunction) {
  const sessionId = lucia.readSessionCookie(req.headers.cookie ?? '');

  if (!sessionId) {
    res.locals.user = null;
    res.locals.session = null;
    return next();
  }

  const { session, user } = await lucia.validateSession(sessionId);

  if (session && session.fresh) {
    res.appendHeader(
      'Set-Cookie',
      lucia.createSessionCookie(session.id).serialize()
    );
  }

  if (!session) {
    res.appendHeader(
      'Set-Cookie',
      lucia.createBlankSessionCookie().serialize()
    );
  }

  res.locals.user = user;
  res.locals.session = session;
  next();
}
```

#### Multi-Factor Authentication

```typescript
import { createTOTPKeyURI, TOTPController } from 'oslo/otp';
import { encodeBase32 } from 'oslo/encoding';
import QRCode from 'qrcode';

export async function setupMFA(userId: string) {
  // Generate secret
  const secret = crypto.getRandomValues(new Uint8Array(20));
  const encodedSecret = encodeBase32(secret);

  // Store encrypted secret
  await db.user.update({
    where: { id: userId },
    data: {
      mfaSecret: encrypt(encodedSecret),
      mfaEnabled: false  // Not enabled until verified
    }
  });

  // Generate QR code URI
  const user = await db.user.findUnique({ where: { id: userId } });
  const uri = createTOTPKeyURI('MyApp', user.email, secret);
  const qrCode = await QRCode.toDataURL(uri);

  return { qrCode, secret: encodedSecret };
}

export async function verifyMFA(userId: string, code: string): Promise<boolean> {
  const user = await db.user.findUnique({ where: { id: userId } });
  const secret = decrypt(user.mfaSecret);

  const totp = new TOTPController();
  const isValid = totp.verify(code, decodeBase32(secret));

  if (isValid && !user.mfaEnabled) {
    await db.user.update({
      where: { id: userId },
      data: { mfaEnabled: true }
    });
  }

  return isValid;
}
```

---

## A08 - Software and Data Integrity Failures

Failures related to code and infrastructure that does not protect against integrity violations.

### Common Vulnerabilities

- Insecure CI/CD pipelines
- Auto-updating without verification
- Insecure deserialization
- Unsigned software updates

### Prevention

#### Secure Deserialization

```typescript
// ❌ Dangerous - Don't deserialize untrusted data
const data = JSON.parse(untrustedInput);
eval(data.code);

// ✅ Use safe parsing with schema validation
import { z } from 'zod';

const configSchema = z.object({
  name: z.string(),
  settings: z.object({
    timeout: z.number().positive(),
    retries: z.number().int().min(0).max(10)
  })
});

function parseConfig(input: string): Config {
  const parsed = JSON.parse(input);
  return configSchema.parse(parsed);
}
```

```python
# ❌ Never use pickle with untrusted data
import pickle
data = pickle.loads(untrusted_input)  # DANGEROUS

# ✅ Use safe alternatives
import json
from pydantic import BaseModel

class Config(BaseModel):
    name: str
    timeout: int

data = Config.model_validate_json(trusted_input)
```

#### Subresource Integrity

```html
<!-- Verify external scripts -->
<script
  src="https://cdn.example.com/lib.js"
  integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC"
  crossorigin="anonymous">
</script>

<link
  rel="stylesheet"
  href="https://cdn.example.com/style.css"
  integrity="sha384-xxx"
  crossorigin="anonymous">
```

#### Signed Commits and Releases

```yaml
# GitHub Actions with signed releases
name: Release

on:
  push:
    tags: ['v*']

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      id-token: write
      attestations: write

    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: npm run build

      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          artifact-name: sbom.spdx.json

      - name: Sign artifact
        uses: sigstore/cosign-installer@v3

      - run: cosign sign-blob --yes dist/app.tar.gz > dist/app.tar.gz.sig

      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          files: |
            dist/app.tar.gz
            dist/app.tar.gz.sig
            sbom.spdx.json
```

---

## A09 - Security Logging and Monitoring Failures

Without logging and monitoring, breaches cannot be detected.

### Prevention

#### Comprehensive Security Logging

```typescript
import winston from 'winston';

const securityLogger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  defaultMeta: { service: 'security' },
  transports: [
    new winston.transports.File({ filename: 'security.log' }),
    new winston.transports.Console()
  ]
});

// Security event types
interface SecurityEvent {
  type: 'AUTH_SUCCESS' | 'AUTH_FAILURE' | 'ACCESS_DENIED' | 'SUSPICIOUS_ACTIVITY';
  userId?: string;
  ip: string;
  userAgent?: string;
  endpoint: string;
  details: Record<string, unknown>;
}

export function logSecurityEvent(event: SecurityEvent): void {
  securityLogger.info({
    ...event,
    timestamp: new Date().toISOString()
  });
}

// Usage in authentication
async function login(email: string, password: string, req: Request) {
  try {
    const user = await authenticate(email, password);

    logSecurityEvent({
      type: 'AUTH_SUCCESS',
      userId: user.id,
      ip: req.ip,
      userAgent: req.headers['user-agent'],
      endpoint: '/api/auth/login',
      details: { email }
    });

    return user;
  } catch (error) {
    logSecurityEvent({
      type: 'AUTH_FAILURE',
      ip: req.ip,
      userAgent: req.headers['user-agent'],
      endpoint: '/api/auth/login',
      details: {
        email,
        reason: error.message
      }
    });

    throw error;
  }
}

// Suspicious activity detection
export function detectSuspiciousActivity(req: Request): void {
  const suspicious = [
    // SQL injection attempts
    /('|"|;|--|\b(union|select|insert|update|delete|drop)\b)/i,
    // XSS attempts
    /(<script|javascript:|on\w+\s*=)/i,
    // Path traversal
    /\.\.\//,
  ];

  const input = JSON.stringify(req.body) + req.url;

  for (const pattern of suspicious) {
    if (pattern.test(input)) {
      logSecurityEvent({
        type: 'SUSPICIOUS_ACTIVITY',
        ip: req.ip,
        userAgent: req.headers['user-agent'],
        endpoint: req.url,
        details: {
          pattern: pattern.toString(),
          body: req.body
        }
      });
      break;
    }
  }
}
```

#### Alerting Configuration

```typescript
// Alert on security events
import { WebClient } from '@slack/web-api';

const slack = new WebClient(process.env.SLACK_TOKEN);

const ALERT_THRESHOLDS = {
  AUTH_FAILURE: { count: 5, windowMs: 60000 },  // 5 failures in 1 minute
  ACCESS_DENIED: { count: 10, windowMs: 60000 },
  SUSPICIOUS_ACTIVITY: { count: 1, windowMs: 60000 }  // Immediate
};

const eventCounts = new Map<string, { count: number; timestamp: number }>();

export async function checkAndAlert(event: SecurityEvent): Promise<void> {
  const threshold = ALERT_THRESHOLDS[event.type];
  if (!threshold) return;

  const key = `${event.type}:${event.ip}`;
  const now = Date.now();

  const current = eventCounts.get(key);

  if (!current || now - current.timestamp > threshold.windowMs) {
    eventCounts.set(key, { count: 1, timestamp: now });
  } else {
    current.count++;

    if (current.count >= threshold.count) {
      await slack.chat.postMessage({
        channel: '#security-alerts',
        text: `:warning: Security Alert: ${event.type}`,
        attachments: [{
          color: 'danger',
          fields: [
            { title: 'IP', value: event.ip, short: true },
            { title: 'Endpoint', value: event.endpoint, short: true },
            { title: 'Count', value: current.count.toString(), short: true },
            { title: 'Details', value: JSON.stringify(event.details) }
          ]
        }]
      });

      // Reset counter after alert
      eventCounts.delete(key);
    }
  }
}
```

---

## A10 - Server-Side Request Forgery (SSRF)

SSRF flaws occur when a web application fetches a remote resource without validating the user-supplied URL.

### Common Vulnerabilities

- Fetching URLs provided by users
- Accessing internal services through the application
- Cloud metadata endpoint access

### Prevention

#### URL Validation

```typescript
import { URL } from 'url';
import dns from 'dns/promises';
import ipaddr from 'ipaddr.js';

const BLOCKED_HOSTS = ['localhost', '127.0.0.1', '0.0.0.0', '169.254.169.254'];
const ALLOWED_PROTOCOLS = ['http:', 'https:'];

async function isPrivateIP(ip: string): Promise<boolean> {
  try {
    const parsed = ipaddr.parse(ip);
    const range = parsed.range();
    return ['private', 'loopback', 'linkLocal', 'uniqueLocal'].includes(range);
  } catch {
    return true; // Treat invalid IPs as private
  }
}

export async function validateUrl(urlString: string): Promise<URL> {
  let url: URL;

  try {
    url = new URL(urlString);
  } catch {
    throw new ValidationError('Invalid URL format');
  }

  // Check protocol
  if (!ALLOWED_PROTOCOLS.includes(url.protocol)) {
    throw new ValidationError('Invalid protocol');
  }

  // Check for blocked hosts
  if (BLOCKED_HOSTS.includes(url.hostname)) {
    throw new ValidationError('Blocked host');
  }

  // Resolve DNS and check for private IPs
  try {
    const addresses = await dns.resolve4(url.hostname);
    for (const ip of addresses) {
      if (await isPrivateIP(ip)) {
        throw new ValidationError('Private IP addresses not allowed');
      }
    }
  } catch (error) {
    if (error.code === 'ENOTFOUND') {
      throw new ValidationError('Host not found');
    }
    throw error;
  }

  return url;
}

// Safe fetch wrapper
export async function safeFetch(urlString: string): Promise<Response> {
  const url = await validateUrl(urlString);

  return fetch(url.toString(), {
    redirect: 'manual',  // Don't follow redirects automatically
    timeout: 10000,
    headers: {
      'User-Agent': 'MyApp/1.0'
    }
  });
}
```

#### Allowlist Approach

```typescript
const ALLOWED_DOMAINS = [
  'api.github.com',
  'api.stripe.com',
  'hooks.slack.com'
];

export function validateExternalUrl(urlString: string): void {
  const url = new URL(urlString);

  if (!ALLOWED_DOMAINS.includes(url.hostname)) {
    throw new ValidationError(`Domain ${url.hostname} is not allowed`);
  }
}

// Webhook validation
export async function processWebhook(payload: WebhookPayload): Promise<void> {
  // Only allow configured callback URLs
  const callback = await db.webhook.findUnique({
    where: { id: payload.webhookId }
  });

  if (!callback) {
    throw new NotFoundError('Webhook not found');
  }

  // URL was validated when webhook was created
  await fetch(callback.url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(payload.data)
  });
}
```

---

## Security Checklist

### Authentication & Authorization
- [ ] Strong password policy enforced
- [ ] Passwords hashed with Argon2id or bcrypt
- [ ] MFA available and encouraged
- [ ] Session tokens are secure (HttpOnly, Secure, SameSite)
- [ ] Rate limiting on authentication endpoints
- [ ] Account lockout after failed attempts
- [ ] RBAC or ABAC implemented

### Data Protection
- [ ] TLS 1.2+ enforced
- [ ] Sensitive data encrypted at rest
- [ ] Proper key management
- [ ] No sensitive data in logs
- [ ] Data classification implemented

### Input Validation
- [ ] All input validated server-side
- [ ] Parameterized queries used
- [ ] Output encoding applied
- [ ] File upload restrictions

### Configuration
- [ ] Security headers configured
- [ ] CORS properly configured
- [ ] Debug mode disabled in production
- [ ] Default credentials changed
- [ ] Unnecessary features disabled

### Monitoring
- [ ] Security events logged
- [ ] Alerting configured
- [ ] Log integrity protected
- [ ] Regular log review

### Dependencies
- [ ] Dependencies pinned
- [ ] Regular security scanning
- [ ] Automated updates configured
- [ ] SBOM maintained
