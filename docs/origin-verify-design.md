# Origin Verify — Preventing Direct Access Bypass

## Problem

When a customer connects their app to ShieldAI, two access paths exist:

```
Path A (protected):   user → CloudFront → ShieldAI Proxy → customer app
Path B (unprotected): user → customer app (direct)
```

Path B bypasses all security middleware — WAF, rate limiting, SSRF protection, audit logging, etc. An attacker who discovers the origin URL can skip ShieldAI entirely.

This is especially common with hosted platforms (Loveable, Vercel, Netlify, Heroku, Railway) where the customer cannot restrict access by IP.

## Solution

The proxy injects a **per-customer secret header** on every request it forwards. The customer adds a one-line check to their app: reject requests missing the header.

```
Path A: user → CloudFront → ShieldAI Proxy [injects header] → customer app [verifies] ✓
Path B: user → customer app [no header] → rejected ✗
```

---

## Design

### Secret Lifecycle

| Event | What Happens |
|-------|-------------|
| App created | 256-bit secret auto-generated, stored in `apps.origin_verify_secret` |
| App config viewed | Secret returned **once** at creation, then masked in API responses |
| Secret rotated | New secret generated via `POST /customers/{cid}/apps/{aid}/rotate-origin-secret` |
| Grace period | Both old and new secret accepted for 5 minutes after rotation |
| App deleted | Secret deleted with app record |

### Data Model

Add column to `apps` table:

```sql
ALTER TABLE apps ADD COLUMN origin_verify_secret VARCHAR(64);
```

- 64-character hex string (256 bits of entropy via `secrets.token_hex(32)`)
- Stored encrypted at rest (RDS encryption / secrets provider)
- NOT returned in standard GET responses — only at creation and explicit reveal endpoint

### Header Injection

In `proxy/main.py`, after building upstream headers:

```python
# Inject origin verify secret if configured
origin_secret = context.customer_config.get("origin_verify_secret")
if origin_secret:
    headers["X-ShieldAI-Origin-Verify"] = origin_secret
```

### API Changes

**App creation response** (one-time reveal):
```json
{
  "id": "uuid",
  "name": "My App",
  "origin_url": "https://app-xyz.loveable.app",
  "origin_verify_secret": "a1b2c3...64chars...",
  "origin_verify_setup": {
    "header_name": "X-ShieldAI-Origin-Verify",
    "instructions_url": "/docs/origin-verify"
  }
}
```

**Standard GET response** (masked):
```json
{
  "id": "uuid",
  "name": "My App",
  "origin_verify_enabled": true
}
```

**Rotate secret:**
```
POST /api/config/customers/{cid}/apps/{aid}/rotate-origin-secret

Response:
{
  "origin_verify_secret": "d4e5f6...new64chars...",
  "previous_secret_valid_until": "2026-02-15T12:05:00Z"
}
```

**Reveal secret** (requires re-authentication or confirmation):
```
POST /api/config/customers/{cid}/apps/{aid}/reveal-origin-secret

Response:
{
  "origin_verify_secret": "a1b2c3...64chars..."
}
```

### Config Service

`CustomerConfigService` already loads app configs into memory. Add `origin_verify_secret` to the cached config dict:

```python
# In customer_config.py load_all()
new_cache[domain] = {
    "app_id": str(app["id"]),
    "customer_id": str(app["customer_id"]),
    "origin_url": app["origin_url"],
    "origin_verify_secret": app.get("origin_verify_secret"),  # NEW
    "enabled_features": features,
    "settings": app.get("settings", {}),
}
```

### Rotation Grace Period

During rotation, both secrets are valid for 5 minutes:

```python
# In main.py — inject current secret (customer checks against both during rotation)
headers["X-ShieldAI-Origin-Verify"] = origin_secret
```

The rotation endpoint returns `previous_secret_valid_until`. The customer's middleware should accept both values during the grace window. Our SDK snippets handle this automatically via an array of valid secrets.

---

## Customer Integration

### How It Works

1. Customer registers their app with ShieldAI
2. ShieldAI returns a secret and the header name
3. Customer adds a middleware check (one of the snippets below)
4. All direct-access attempts are now rejected

### Express / Node.js

```javascript
// middleware/shieldai-verify.js
const SHIELDAI_SECRET = process.env.SHIELDAI_ORIGIN_SECRET;

function shieldaiVerify(req, res, next) {
  if (req.headers['x-shieldai-origin-verify'] === SHIELDAI_SECRET) {
    return next();
  }
  res.status(403).json({ error: 'Direct access not allowed' });
}

module.exports = shieldaiVerify;

// In app.js:
app.use(shieldaiVerify);
```

### Next.js (middleware.ts)

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const secret = request.headers.get('x-shieldai-origin-verify');
  if (secret !== process.env.SHIELDAI_ORIGIN_SECRET) {
    return NextResponse.json(
      { error: 'Direct access not allowed' },
      { status: 403 }
    );
  }
}

// Optionally limit to API routes only:
export const config = { matcher: ['/api/:path*', '/((?!_next|favicon.ico).*)'] };
```

### Django

```python
# middleware/shieldai.py
import os
from django.http import JsonResponse

SHIELDAI_SECRET = os.environ.get('SHIELDAI_ORIGIN_SECRET', '')

class ShieldAIVerifyMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        if request.META.get('HTTP_X_SHIELDAI_ORIGIN_VERIFY') != SHIELDAI_SECRET:
            return JsonResponse({'error': 'Direct access not allowed'}, status=403)
        return self.get_response(request)

# In settings.py:
MIDDLEWARE = [
    'middleware.shieldai.ShieldAIVerifyMiddleware',
    # ... other middleware
]
```

### Flask

```python
# Before any routes:
import os
from flask import request, jsonify

SHIELDAI_SECRET = os.environ.get('SHIELDAI_ORIGIN_SECRET', '')

@app.before_request
def verify_shieldai():
    if request.headers.get('X-ShieldAI-Origin-Verify') != SHIELDAI_SECRET:
        return jsonify(error='Direct access not allowed'), 403
```

### FastAPI

```python
# middleware.py
import os
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import JSONResponse

SHIELDAI_SECRET = os.environ.get('SHIELDAI_ORIGIN_SECRET', '')

class ShieldAIVerifyMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        if request.headers.get('x-shieldai-origin-verify') != SHIELDAI_SECRET:
            return JSONResponse({'error': 'Direct access not allowed'}, status_code=403)
        return await call_next(request)

# In main.py:
app.add_middleware(ShieldAIVerifyMiddleware)
```

### Rails

```ruby
# app/middleware/shieldai_verify.rb
class ShieldaiVerify
  def initialize(app)
    @app = app
    @secret = ENV['SHIELDAI_ORIGIN_SECRET']
  end

  def call(env)
    if env['HTTP_X_SHIELDAI_ORIGIN_VERIFY'] == @secret
      @app.call(env)
    else
      [403, { 'Content-Type' => 'application/json' }, ['{"error":"Direct access not allowed"}']]
    end
  end
end

# In config/application.rb:
config.middleware.use ShieldaiVerify
```

### Go (net/http)

```go
func shieldaiVerify(next http.Handler) http.Handler {
    secret := os.Getenv("SHIELDAI_ORIGIN_SECRET")
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Header.Get("X-ShieldAI-Origin-Verify") != secret {
            http.Error(w, `{"error":"Direct access not allowed"}`, http.StatusForbidden)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

### Loveable / Vite (via Vite plugin or server middleware)

For Loveable apps that use Vite's dev server or an Express backend:

```javascript
// server.js or vite.config.ts server middleware
const SHIELDAI_SECRET = process.env.SHIELDAI_ORIGIN_SECRET;

export default {
  server: {
    proxy: {}, // existing config
  },
  plugins: [{
    name: 'shieldai-verify',
    configureServer(server) {
      server.middlewares.use((req, res, next) => {
        if (req.headers['x-shieldai-origin-verify'] === SHIELDAI_SECRET) {
          return next();
        }
        res.writeHead(403, { 'Content-Type': 'application/json' });
        res.end('{"error":"Direct access not allowed"}');
      });
    },
  }],
};
```

> **Note:** For static-only Loveable apps with no server component, origin verify requires a serverless function or edge middleware (e.g., Vercel Edge Functions, Cloudflare Workers) to check the header before serving content.

---

## Security Considerations

| Concern | Mitigation |
|---------|-----------|
| Secret leaked in logs | Header value never logged by ShieldAI proxy (stripped from structured logs) |
| Secret in transit | TLS between proxy and origin (HTTPS `origin_url` enforced by SSRF validator) |
| Brute force | 256-bit entropy = 2^256 guesses required |
| Timing attack on comparison | Customer snippets use `===` / `==` (constant-time at language level for fixed-length strings). For paranoid setups, use `hmac.compare_digest` / `crypto.timingSafeEqual` |
| Secret stored in env var | Standard practice; same security posture as database passwords. Customer can use their own secrets manager |
| Rotation window | 5-minute grace period is short enough to limit exposure, long enough for zero-downtime deploy |
| Header stripped by CDN | `X-ShieldAI-*` is non-standard and not stripped by any major CDN. If customer uses a CDN in front of their origin, they must allowlist this header |

---

## Files to Modify

| File | Change |
|------|--------|
| `proxy/models/schema.sql` | Add `origin_verify_secret` column to `apps` |
| `proxy/store/postgres.py` | Include column in CRUD queries, mask in reads |
| `proxy/config/customer_config.py` | Load secret into cached config |
| `proxy/main.py` | Inject header when forwarding to upstream |
| `proxy/api/config_routes.py` | Return secret at creation, add rotate/reveal endpoints |
| `proxy/models/customer.py` | Add field to Pydantic models |
| `tests/test_origin_verify.py` | Unit + attack simulation tests |

---

## Verification Checklist

- [ ] Secret auto-generated on app creation (256-bit)
- [ ] Secret injected as `X-ShieldAI-Origin-Verify` header on every proxied request
- [ ] Secret NOT returned in standard GET app responses
- [ ] Secret returned once at creation and via explicit reveal endpoint
- [ ] Rotate endpoint generates new secret, returns grace period expiry
- [ ] Secret NOT logged in structured logs (stripped from log context)
- [ ] HTTPS enforced for origin_url (existing SSRF validator)
- [ ] Customer middleware snippets tested for each framework
- [ ] Onboarding flow updated to display secret + setup instructions
