# Mobile App Architecture

**Stack:** React Native (Expo) · Node.js/Express API · PostgreSQL · JWT Auth

---

## 1. Authentication System

### Strategy: JWT + Refresh Token Rotation

- **Access token** — short-lived (15 min), stored in memory (not AsyncStorage)
- **Refresh token** — long-lived (30 days), stored in `expo-secure-store` (Keychain/Keystore)
- **Rotation** — each refresh issues a new refresh token and invalidates the old one
- **Revocation** — refresh tokens are stored in the DB so they can be invalidated on logout or suspicious activity

### Auth Flow

```
[App Launch]
    │
    ├─ No refresh token → Login / Register screen
    │
    └─ Has refresh token
           │
           ├─ POST /auth/refresh → new access token + new refresh token
           │
           ├─ 401 (expired/revoked) → force logout, clear secure store
           │
           └─ Success → proceed to app
```

### Password Security

- **Hashing:** `bcrypt` with cost factor 12
- **Reset flow:** time-limited signed token (SHA-256, 1 hr TTL) emailed to user
- **Rate limiting:** 5 failed attempts per 15 min per IP + per account (separate counters)

### API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/auth/register` | Create account, return tokens |
| POST | `/auth/login` | Verify credentials, return tokens |
| POST | `/auth/refresh` | Rotate refresh token, return new access token |
| POST | `/auth/logout` | Revoke refresh token |
| POST | `/auth/forgot-password` | Send reset email |
| POST | `/auth/reset-password` | Consume reset token, set new password |
| GET  | `/auth/me` | Return current user (requires access token) |

---

## 2. Database Schema (PostgreSQL)

### `users`

```sql
CREATE TABLE users (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email         TEXT UNIQUE NOT NULL,
  email_verified BOOLEAN NOT NULL DEFAULT false,
  password_hash TEXT NOT NULL,
  display_name  TEXT,
  avatar_url    TEXT,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_email ON users(email);
```

### `refresh_tokens`

```sql
CREATE TABLE refresh_tokens (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash  TEXT UNIQUE NOT NULL,   -- SHA-256 of the raw token
  device_name TEXT,                   -- "iPhone 15", "Pixel 8", etc.
  ip_address  INET,
  expires_at  TIMESTAMPTZ NOT NULL,
  revoked_at  TIMESTAMPTZ,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_refresh_tokens_user ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_hash ON refresh_tokens(token_hash);
```

### `password_reset_tokens`

```sql
CREATE TABLE password_reset_tokens (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash  TEXT UNIQUE NOT NULL,
  expires_at  TIMESTAMPTZ NOT NULL,
  used_at     TIMESTAMPTZ,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### `email_verifications`

```sql
CREATE TABLE email_verifications (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash  TEXT UNIQUE NOT NULL,
  expires_at  TIMESTAMPTZ NOT NULL,
  verified_at TIMESTAMPTZ,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### `login_attempts` (rate limiting)

```sql
CREATE TABLE login_attempts (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  identifier  TEXT NOT NULL,   -- email or IP
  kind        TEXT NOT NULL,   -- 'email' | 'ip'
  succeeded   BOOLEAN NOT NULL,
  attempted_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_login_attempts_identifier ON login_attempts(identifier, attempted_at DESC);

-- Auto-purge old rows (optional cron or pg_partman)
```

### Trigger: auto-update `updated_at`

```sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

---

## 3. App Architecture

### Directory Structure

```
/
├── apps/
│   ├── mobile/                  # React Native (Expo)
│   │   ├── app/                 # Expo Router file-based routing
│   │   │   ├── (auth)/
│   │   │   │   ├── login.tsx
│   │   │   │   ├── register.tsx
│   │   │   │   └── forgot-password.tsx
│   │   │   ├── (app)/           # Protected routes
│   │   │   │   ├── _layout.tsx  # Auth guard here
│   │   │   │   └── index.tsx
│   │   │   └── _layout.tsx      # Root layout, token bootstrap
│   │   ├── src/
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   │   └── useAuth.ts
│   │   │   ├── stores/
│   │   │   │   └── authStore.ts  # Zustand, holds access token in memory
│   │   │   ├── services/
│   │   │   │   └── api.ts        # Axios instance with interceptors
│   │   │   └── utils/
│   │   └── package.json
│   │
│   └── api/                     # Node.js / Express
│       ├── src/
│       │   ├── routes/
│       │   │   └── auth.ts
│       │   ├── middleware/
│       │   │   ├── authenticate.ts   # Verify access token
│       │   │   └── rateLimit.ts
│       │   ├── services/
│       │   │   ├── authService.ts
│       │   │   └── tokenService.ts
│       │   ├── db/
│       │   │   ├── client.ts         # pg pool
│       │   │   └── migrations/
│       │   └── app.ts
│       └── package.json
│
├── packages/
│   └── shared-types/            # Shared TypeScript types (monorepo)
│
└── package.json                 # Turborepo / pnpm workspaces
```

### Key Mobile Patterns

**Token storage** (never put JWTs in AsyncStorage):
```ts
// Refresh token → expo-secure-store
await SecureStore.setItemAsync('refresh_token', token);

// Access token → Zustand in-memory store only
authStore.setState({ accessToken: token });
```

**Axios interceptor** (silent token refresh):
```ts
api.interceptors.response.use(
  res => res,
  async err => {
    if (err.response?.status === 401 && !err.config._retry) {
      err.config._retry = true;
      const newToken = await tokenService.refresh();
      err.config.headers['Authorization'] = `Bearer ${newToken}`;
      return api(err.config);
    }
    return Promise.reject(err);
  }
);
```

**Auth guard** (Expo Router `_layout.tsx`):
```ts
export default function AppLayout() {
  const { accessToken } = useAuthStore();
  if (!accessToken) return <Redirect href="/login" />;
  return <Slot />;
}
```

### API Middleware Stack

```
Request
  → helmet (security headers)
  → cors
  → express.json()
  → rateLimit (global)
  → router
      → /auth/* (public)
      → /api/* → authenticate middleware → controllers
```

---

## 4. Environment Variables

### API (`apps/api/.env`)
```
DATABASE_URL=postgresql://user:pass@localhost:5432/appdb
JWT_ACCESS_SECRET=<32-byte random>
JWT_REFRESH_SECRET=<32-byte random>
JWT_ACCESS_TTL=900           # 15 minutes in seconds
JWT_REFRESH_TTL=2592000      # 30 days in seconds
SMTP_HOST=
SMTP_PORT=
SMTP_USER=
SMTP_PASS=
APP_NAME=YourApp
FRONTEND_URL=yourapp://
```

### Mobile (`apps/mobile/.env`)
```
EXPO_PUBLIC_API_URL=https://api.yourdomain.com
```

---

## 5. Security Checklist

- [ ] bcrypt cost ≥ 12
- [ ] Refresh tokens hashed in DB (never stored raw)
- [ ] Access tokens expire in ≤ 15 min
- [ ] Refresh token rotation on every use
- [ ] Rate limit on `/auth/login` and `/auth/register`
- [ ] Password reset tokens expire in 1 hr and are single-use
- [ ] HTTPS enforced in production
- [ ] `expo-secure-store` for all sensitive on-device storage
- [ ] No sensitive data in Expo's `AsyncStorage`
- [ ] DB user has least-privilege (no `DROP`, `CREATE` in production)
