# Security Audit Report: JavaScriptSolidServer

**Audit Date:** 2026-01-02
**Auditor:** Code Review Agent
**Version:** Current main branch
**Scope:** Authentication, Authorization (WAC), Input Validation, Cryptography, Dependencies, Server Configuration

---

## Executive Summary

JavaScriptSolidServer (JSS) demonstrates a generally solid security posture for a development-stage Solid server implementation. The codebase shows evidence of security-conscious design decisions, particularly in:

- Proper use of the `jose` library for JWT/DPoP operations
- Implementation of bcrypt for password hashing
- Basic path traversal protections
- DPoP binding verification for Solid-OIDC

However, this audit identified several areas requiring attention, ranging from a critical DPoP replay vulnerability to informational items about hardcoded defaults. The most significant issues relate to:

1. **Missing DPoP nonce/jti replay protection** (Critical)
2. **Hardcoded token secret for simple auth mode** (High)
3. **Permissive default when no ACL exists** (Medium)
4. **Path traversal protection could be bypassed** (Medium)

**Overall Risk Assessment:** Moderate - Suitable for development/testing; requires hardening before production deployment.

---

## Findings by Severity

### CRITICAL

#### C-01: DPoP Replay Attack Vulnerability

**Location:** `/home/devuser/workspace/jss-contributions/jss-upstream/src/auth/solid-oidc.js` (lines 163-167)

**Description:**
The DPoP proof verification correctly checks for the presence of the `jti` (JWT ID) claim but does not track previously used `jti` values. This allows an attacker who captures a valid DPoP proof to replay it within the 5-minute validity window.

**Current Code:**
```javascript
// jti: Unique identifier (we should track these to prevent replay, but skip for now)
if (!payload.jti) {
  return { thumbprint: null, error: 'DPoP proof missing jti' };
}
```

**Impact:**
- An attacker with network access (MITM, compromised proxy, access to logs) can replay captured DPoP proofs
- Enables unauthorized access to protected resources using stolen credentials
- Violates Solid-OIDC and RFC 9449 (DPoP) security requirements

**Recommendation:**
Implement a time-bounded cache to track used `jti` values. Example approach:

```javascript
// In-memory cache with automatic cleanup (use Redis for distributed deployments)
const usedJtis = new Map();
const JTI_CLEANUP_INTERVAL = 60 * 1000; // 1 minute

// Cleanup expired jti entries periodically
setInterval(() => {
  const cutoff = Date.now() - (DPOP_MAX_AGE * 1000);
  for (const [jti, timestamp] of usedJtis) {
    if (timestamp < cutoff) usedJtis.delete(jti);
  }
}, JTI_CLEANUP_INTERVAL);

// In verifyDpopProof, after jti check:
if (usedJtis.has(payload.jti)) {
  return { thumbprint: null, error: 'DPoP proof already used (replay detected)' };
}
usedJtis.set(payload.jti, Date.now());
```

**Cherry-pick Commit Title:** `fix(auth): implement DPoP jti replay protection`

---

### HIGH

#### H-01: Hardcoded Token Secret in Simple Auth Mode

**Location:** `/home/devuser/workspace/jss-contributions/jss-upstream/src/auth/token.js` (line 15)

**Description:**
The HMAC secret for simple Bearer tokens falls back to a hardcoded value when `TOKEN_SECRET` environment variable is not set:

```javascript
const SECRET = process.env.TOKEN_SECRET || 'dev-secret-change-in-production';
```

**Impact:**
- All deployments using simple auth mode without explicitly setting `TOKEN_SECRET` share the same signing key
- Attackers can forge valid tokens for any WebID
- Tokens generated on one server would be valid on another

**Recommendation:**
1. Log a warning on startup if using the default secret
2. Consider refusing to start in non-development mode without an explicit secret
3. Generate a random secret on first run and persist it

```javascript
import crypto from 'crypto';

function getOrCreateSecret() {
  if (process.env.TOKEN_SECRET) {
    return process.env.TOKEN_SECRET;
  }

  if (process.env.NODE_ENV === 'production') {
    console.error('FATAL: TOKEN_SECRET must be set in production mode');
    process.exit(1);
  }

  console.warn('WARNING: Using default token secret. Set TOKEN_SECRET for production.');
  return 'dev-secret-change-in-production';
}

const SECRET = getOrCreateSecret();
```

**Cherry-pick Commit Title:** `fix(auth): enforce TOKEN_SECRET in production mode`

---

#### H-02: JWT Token Verification Skipped for Bearer Tokens

**Location:** `/home/devuser/workspace/jss-contributions/jss-upstream/src/auth/token.js` (lines 93-122)

**Description:**
The `verifyJwtToken` function decodes JWT tokens (3-part format) but only validates expiration - it does not verify the signature:

```javascript
function verifyJwtToken(token) {
  try {
    const parts = token.split('.');
    if (parts.length !== 3) {
      return null;
    }
    // Decode the payload (middle part)
    const payload = JSON.parse(Buffer.from(parts[1], 'base64url').toString());
    // Check expiration
    if (payload.exp && payload.exp < Math.floor(Date.now() / 1000)) {
      return null;
    }
    // ... returns payload without signature verification
```

**Impact:**
- Attackers can forge JWT tokens with arbitrary claims
- Any crafted JWT with valid structure and future expiration will be accepted
- Complete authentication bypass for the simple auth path

**Recommendation:**
Either verify the JWT signature using the IdP's JWKS, or reject JWTs in the simple auth path (since DPoP tokens have their own flow):

```javascript
function verifyJwtToken(token) {
  // Simple auth mode should only accept 2-part tokens we issued
  // JWTs from the IdP go through the DPoP flow
  console.warn('JWT token received via Bearer auth - use DPoP for Solid-OIDC tokens');
  return null;
}
```

**Cherry-pick Commit Title:** `fix(auth): reject unsigned JWTs in simple auth mode`

---

### MEDIUM

#### M-01: Permissive Default When No ACL Exists

**Location:** `/home/devuser/workspace/jss-contributions/jss-upstream/src/wac/checker.js` (lines 30-34)

**Description:**
When no ACL file is found in the hierarchy, the server defaults to permissive mode:

```javascript
if (!aclResult) {
  // No ACL found - allow by default (permissive mode)
  // This allows resources without ACLs to be publicly accessible
  return { allowed: true, wacAllow: 'user="read write append control", public="read write append"' };
}
```

**Impact:**
- Resources without ACLs are fully writable by unauthenticated users
- Data can be modified or deleted by anyone
- Violates principle of least privilege

**Recommendation:**
Default to read-only for unauthenticated users, require authentication for writes:

```javascript
if (!aclResult) {
  // No ACL found - default to public read, authenticated write
  if (agentWebId) {
    return { allowed: true, wacAllow: 'user="read write append", public="read"' };
  }
  // Unauthenticated: read-only
  const isReadOperation = requiredMode === AccessMode.READ;
  return {
    allowed: isReadOperation,
    wacAllow: 'user="read write append", public="read"'
  };
}
```

**Cherry-pick Commit Title:** `fix(wac): restrict default permissions when no ACL exists`

---

#### M-02: Incomplete Path Traversal Protection

**Location:** `/home/devuser/workspace/jss-contributions/jss-upstream/src/utils/url.js` (lines 28-29, 46-47)

**Description:**
Path traversal protection uses simple string replacement:

```javascript
// Security: prevent path traversal
normalized = normalized.replace(/\.\./g, '');
```

This approach has bypasses:
- URL-encoded sequences: `%2e%2e` or `%2e.` or `.%2e`
- Unicode normalization attacks
- Double encoding: `%252e%252e`

**Impact:**
- Potential access to files outside DATA_ROOT
- Could read sensitive system files
- Could overwrite critical files

**Recommendation:**
Use path.resolve() and verify the result stays within DATA_ROOT:

```javascript
export function urlToPath(urlPath) {
  // Normalize: remove leading slash, decode URI
  let normalized = urlPath.startsWith('/') ? urlPath.slice(1) : urlPath;

  // Decode URI components (handles %2e etc.)
  try {
    normalized = decodeURIComponent(normalized);
  } catch (e) {
    // Invalid encoding - reject
    throw new Error('Invalid URL encoding');
  }

  const dataRoot = path.resolve(getDataRoot());
  const resolved = path.resolve(dataRoot, normalized);

  // Verify resolved path is within data root
  if (!resolved.startsWith(dataRoot + path.sep) && resolved !== dataRoot) {
    throw new Error('Path traversal detected');
  }

  return resolved;
}
```

**Cherry-pick Commit Title:** `fix(security): strengthen path traversal protection`

---

#### M-03: Git HTTP Backend Command Injection Risk

**Location:** `/home/devuser/workspace/jss-contributions/jss-upstream/src/handlers/git.js` (lines 80-82, 106-122)

**Description:**
The URL path is decoded and used directly in git operations:

```javascript
const urlPath = decodeURIComponent(request.url.split('?')[0]);
// ...
const repoAbs = resolve(dataRoot, repoRelative);
// ...
execSync('git config http.receivepack true', {
  cwd: repoAbs,
  // ...
});
```

While the path is used for `cwd` (not directly in the command), a malicious repository name could still cause issues if the path validation is bypassed.

**Impact:**
- Potential for path traversal to access repositories outside intended scope
- Could enable git operations on unintended directories

**Recommendation:**
Add explicit validation of the repository path:

```javascript
// Validate repoRelative contains only safe characters
if (!/^[\w\-\/\.]+$/.test(repoRelative)) {
  return reply.code(400).send({ error: 'Invalid repository path' });
}

// Ensure repoAbs is within dataRoot
const normalizedDataRoot = resolve(dataRoot);
if (!repoAbs.startsWith(normalizedDataRoot + '/')) {
  return reply.code(403).send({ error: 'Access denied' });
}
```

**Cherry-pick Commit Title:** `fix(git): validate repository paths to prevent traversal`

---

#### M-04: Overly Permissive CORS for Git Handler

**Location:** `/home/devuser/workspace/jss-contributions/jss-upstream/src/handlers/git.js` (lines 189-192)

**Description:**
The git handler sets `Access-Control-Allow-Origin: *` unconditionally:

```javascript
// Add CORS headers for browser git clients
reply.raw.setHeader('Access-Control-Allow-Origin', '*');
```

**Impact:**
- Any website can make authenticated requests to the git endpoint
- Could enable CSRF-like attacks for git push operations
- Credentials could be exposed cross-origin

**Recommendation:**
Use the origin from the request or a configured whitelist:

```javascript
const origin = request.headers.origin;
const allowedOrigins = config.gitCorsOrigins || [];

if (origin && (allowedOrigins.includes(origin) || allowedOrigins.includes('*'))) {
  reply.raw.setHeader('Access-Control-Allow-Origin', origin);
  reply.raw.setHeader('Access-Control-Allow-Credentials', 'true');
} else {
  reply.raw.setHeader('Access-Control-Allow-Origin', request.hostname);
}
```

**Cherry-pick Commit Title:** `fix(git): restrict CORS to configured origins`

---

### LOW

#### L-01: No Rate Limiting on Authentication Endpoints

**Location:** `/home/devuser/workspace/jss-contributions/jss-upstream/src/idp/interactions.js`, `/home/devuser/workspace/jss-contributions/jss-upstream/src/idp/credentials.js`

**Description:**
Login and credential endpoints lack rate limiting, enabling brute-force attacks against user accounts.

**Impact:**
- Account credential enumeration
- Brute-force password attacks
- Potential denial of service

**Recommendation:**
Implement rate limiting using a package like `@fastify/rate-limit`:

```javascript
await fastify.register(import('@fastify/rate-limit'), {
  max: 5,
  timeWindow: '1 minute',
  keyGenerator: (request) => {
    // Rate limit by IP + username combination
    const body = request.body || {};
    return `${request.ip}-${body.username || body.email || 'unknown'}`;
  }
});
```

**Cherry-pick Commit Title:** `feat(security): add rate limiting to authentication endpoints`

---

#### L-02: Information Disclosure in Error Messages

**Location:** Multiple files including `/home/devuser/workspace/jss-contributions/jss-upstream/src/idp/interactions.js` (line 43), `/home/devuser/workspace/jss-contributions/jss-upstream/src/handlers/resource.js`

**Description:**
Error messages sometimes include internal details:

```javascript
return reply.code(500).type('text/html').send(errorPage('Server Error', err.message));
```

**Impact:**
- Stack traces or internal paths could be exposed
- Aids attackers in understanding system internals

**Recommendation:**
Log detailed errors server-side, return generic messages to clients:

```javascript
request.log.error(err, 'Interaction error');
return reply.code(500).type('text/html').send(
  errorPage('Server Error', 'An internal error occurred. Please try again.')
);
```

**Cherry-pick Commit Title:** `fix(security): sanitize error messages in responses`

---

#### L-03: WebSocket Subscription Lacks Authentication Check

**Location:** `/home/devuser/workspace/jss-contributions/jss-upstream/src/notifications/websocket.js` (lines 26-64)

**Description:**
WebSocket connections for Solid Notifications don't verify if the client is authorized to receive updates for the subscribed resources:

```javascript
if (msg.startsWith('sub ')) {
  const url = msg.slice(4).trim();
  if (url) {
    subscribe(socket, url);
    socket.send(`ack ${url}`);
  }
}
```

**Impact:**
- Any client can subscribe to any resource URL
- Enables monitoring of private resource changes
- Information disclosure about activity patterns

**Recommendation:**
Validate authorization before accepting subscriptions:

```javascript
if (msg.startsWith('sub ')) {
  const url = msg.slice(4).trim();
  if (url) {
    // Check if client is authorized to read this resource
    const authorized = await checkSubscriptionAuth(socket, url);
    if (authorized) {
      subscribe(socket, url);
      socket.send(`ack ${url}`);
    } else {
      socket.send(`error unauthorized ${url}`);
    }
  }
}
```

**Cherry-pick Commit Title:** `fix(notifications): add authorization check for subscriptions`

---

#### L-04: Client Document Fetch Lacks Timeout

**Location:** `/home/devuser/workspace/jss-contributions/jss-upstream/src/idp/provider.js` (lines 20-71)

**Description:**
The `fetchClientDocument` function fetches client metadata from URLs provided in the authorization request without a timeout:

```javascript
const response = await fetch(clientId, {
  headers: { 'Accept': 'application/json, application/ld+json' },
});
```

**Impact:**
- Slow or unresponsive client document servers could cause hangs
- Denial of service by specifying slow endpoints
- Resource exhaustion from many pending requests

**Recommendation:**
Add AbortController with timeout:

```javascript
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 10000);

try {
  const response = await fetch(clientId, {
    headers: { 'Accept': 'application/json, application/ld+json' },
    signal: controller.signal,
  });
} finally {
  clearTimeout(timeout);
}
```

**Cherry-pick Commit Title:** `fix(idp): add timeout to client document fetches`

---

### INFORMATIONAL

#### I-01: PKCE Required for All Clients

**Location:** `/home/devuser/workspace/jss-contributions/jss-upstream/src/idp/provider.js` (lines 326-331)

**Observation:**
PKCE is correctly required for all clients:

```javascript
pkceMethods: ['S256'],
pkce: {
  required: () => true,
  methods: ['S256'],
},
```

This is a positive security measure, consistent with OAuth 2.1 best practices.

---

#### I-02: Secure Cookie Configuration

**Location:** `/home/devuser/workspace/jss-contributions/jss-upstream/src/idp/provider.js` (lines 90-106)

**Observation:**
Cookies are configured with appropriate security flags:
- `httpOnly: true` - Prevents XSS access
- `signed: true` - Prevents tampering
- `sameSite: 'lax'` - CSRF protection

Note: `secure: true` should be added for production HTTPS deployments.

---

#### I-03: Password Hashing Uses bcrypt

**Location:** `/home/devuser/workspace/jss-contributions/jss-upstream/src/idp/accounts.js` (verified via credentials.js reference)

**Observation:**
bcrypt is used for password hashing, which is appropriate. Verify cost factor is set appropriately (recommend minimum of 10, ideally 12+).

---

#### I-04: Filesystem Adapter ID Sanitization

**Location:** `/home/devuser/workspace/jss-contributions/jss-upstream/src/idp/adapter.js` (lines 44-47)

**Observation:**
IDs are sanitized before use in file paths:

```javascript
_path(id) {
  // Sanitize ID to prevent path traversal
  const safeId = id.replace(/[^a-zA-Z0-9_-]/g, '_');
  return path.join(this.dir, `${safeId}.json`);
}
```

This is a positive security measure.

---

#### I-05: Slug Sanitization in Container Handler

**Location:** `/home/devuser/workspace/jss-contributions/jss-upstream/src/storage/filesystem.js` (lines 136-137)

**Observation:**
POST slug headers are sanitized:

```javascript
// Remove any path traversal attempts
name = name.replace(/[/\\]/g, '-');
```

Consider extending to handle URL-encoded path separators.

---

## Dependency Security

**npm audit result:** 0 vulnerabilities

The project has no known vulnerable dependencies as of the audit date. Key security-relevant dependencies:

| Package | Version | Status |
|---------|---------|--------|
| jose | (current) | OK - Well-maintained JWT library |
| oidc-provider | (current) | OK - Certified OpenID Connect implementation |
| bcrypt | (current) | OK - Industry-standard password hashing |
| nostr-tools | (current) | OK - Nostr protocol implementation |

**Recommendation:** Set up automated dependency scanning (e.g., Dependabot, Snyk) to catch future vulnerabilities.

---

## Server Configuration Review

### Positive Configurations

1. **X-Frame-Options: DENY** set for Mashlib pages (prevents clickjacking)
2. **Content-Security-Policy: frame-ancestors 'none'** for data browser pages
3. **Cache-Control: no-store** for dynamic content
4. **Vary headers** properly set for content negotiation

### Areas for Improvement

1. **No HSTS header** - Should add `Strict-Transport-Security` for HTTPS deployments
2. **No Content-Type-Options** - Add `X-Content-Type-Options: nosniff`
3. **No XSS Protection header** - Add `X-XSS-Protection: 1; mode=block`
4. **Default host binding** - Binds to `0.0.0.0` by default, should document security implications

---

## Summary of Recommended Actions

### Immediate (Before Production)

1. Implement DPoP jti replay protection (C-01)
2. Enforce TOKEN_SECRET configuration (H-01)
3. Fix JWT signature verification (H-02)
4. Strengthen path traversal protection (M-02)

### Short-Term

5. Restrict default permissions when no ACL (M-01)
6. Validate git repository paths (M-03)
7. Restrict git CORS configuration (M-04)
8. Add rate limiting (L-01)

### Medium-Term

9. Add WebSocket subscription authorization (L-03)
10. Add timeout to client document fetches (L-04)
11. Sanitize error messages (L-02)
12. Add security headers (HSTS, X-Content-Type-Options, etc.)

---

## Conclusion

JavaScriptSolidServer shows thoughtful security design in many areas, particularly around the core Solid-OIDC implementation. The identified issues are addressable with targeted fixes, and the codebase provides a solid foundation for a secure Solid server implementation.

The most critical fix is implementing DPoP replay protection (C-01), as this directly impacts the security of the authentication system. The high-severity items around token secrets and JWT verification should also be addressed before any production deployment.

This audit was conducted as a constructive contribution to improve the project's security posture.

---

*Report generated by Code Review Agent*
*Methodology: Static code analysis with manual review*
