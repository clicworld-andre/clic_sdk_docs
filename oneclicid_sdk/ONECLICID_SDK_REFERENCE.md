# OneClicID SDK Reference

**Version:** 1.0.0
**Last Updated:** January 2026
**Server Port:** 3002
**Base URL:** `http://localhost:3002` (development) | `https://id.clic.world` (production)

---

## Table of Contents

1. [Overview](#overview)
2. [Quick Start](#quick-start)
3. [Authentication](#authentication)
4. [API Endpoints](#api-endpoints)
5. [OIDC Integration](#oidc-integration)
6. [Device Identity](#device-identity)
7. [Wallet Integration](#wallet-integration)
8. [Matrix Integration](#matrix-integration)
9. [Admin API](#admin-api)
10. [Data Models](#data-models)
11. [Error Handling](#error-handling)
12. [Security Best Practices](#security-best-practices)

---

## Overview

OneClicID is an OpenID Connect (OIDC) identity provider server for the Clic ecosystem. It provides:

- **User Authentication**: Standard login/register with optional 2FA (TOTP)
- **Matrix Integration**: Federated identity with Matrix homeservers
- **OIDC Provider**: Full OpenID Connect compliance for SSO
- **Device Identity**: Fingerprint-based device trust management
- **Wallet Integration**: Stellar-based identity and money wallets
- **KYC/MiFIR Compliance**: Identity verification fields for regulatory compliance

### Technology Stack

| Component | Technology |
|-----------|------------|
| Framework | Express.js 4.21 |
| Language | TypeScript 5.6 |
| Database | PostgreSQL (Prisma ORM 5.22) |
| OIDC | oidc-provider 9.6 |
| Password Hashing | Argon2 |
| 2FA | TOTP (otplib) |
| Wallet | Stellar SDK 14.4 |
| Cache | Redis (ioredis) |

---

## Quick Start

### 1. Register a User

```bash
curl -X POST http://localhost:3002/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "johndoe",
    "email": "john@example.com",
    "password": "SecurePass123!",
    "displayName": "John Doe"
  }'
```

### 2. Login

```bash
curl -X POST http://localhost:3002/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "usernameOrEmail": "johndoe",
    "password": "SecurePass123!"
  }'
```

**Response:**
```json
{
  "sessionToken": "eyJ...",
  "refreshToken": "abc123...",
  "expiresIn": 86400,
  "user": {
    "id": "uuid",
    "username": "johndoe",
    "email": "john@example.com"
  }
}
```

### 3. Authenticated Request

```bash
curl http://localhost:3002/auth/me \
  -H "Authorization: Bearer eyJ..."
```

---

## Authentication

### Token Lifecycle

| Token Type | Lifetime | Purpose |
|------------|----------|---------|
| Session Token | 24 hours | API authentication |
| Refresh Token | 14 days | Obtain new session tokens |
| OIDC Access Token | 1 hour | Resource access |
| OIDC ID Token | 1 hour | Identity claims |

### Authentication Header

All authenticated endpoints require:
```
Authorization: Bearer <sessionToken>
```

### Refresh Token Rotation

Refresh tokens use family-based rotation for security:
- Each refresh generates a new token
- Token reuse (replay attack) revokes the entire family
- Prevents concurrent session hijacking

---

## API Endpoints

### Authentication Routes (`/auth`)

#### POST /auth/register
Register a new user account.

**Request Body:**
```json
{
  "username": "string (required, 3-50 chars)",
  "email": "string (required, valid email)",
  "password": "string (required, min 8 chars)",
  "displayName": "string (optional)",
  "phone": "string (optional)"
}
```

**Response:** `201 Created`
```json
{
  "id": "uuid",
  "username": "string",
  "email": "string",
  "displayName": "string",
  "createdAt": "ISO 8601"
}
```

---

#### POST /auth/login
Authenticate with credentials.

**Request Body:**
```json
{
  "usernameOrEmail": "string (required)",
  "password": "string (required)",
  "deviceFingerprint": "object (optional)"
}
```

**Response:** `200 OK`
```json
{
  "sessionToken": "string",
  "refreshToken": "string",
  "expiresIn": 86400,
  "user": { "id", "username", "email", "displayName" },
  "device": { "id", "trustLevel", "isNew" }
}
```

**Response (MFA Required):** `202 Accepted`
```json
{
  "requiresMFA": true,
  "userId": "uuid",
  "mfaType": "totp"
}
```

---

#### POST /auth/login/totp
Complete login with TOTP code.

**Request Body:**
```json
{
  "userId": "uuid (required)",
  "token": "string (required, 6 digits)",
  "deviceFingerprint": "object (optional)"
}
```

**Response:** `200 OK` (same as /auth/login)

---

#### POST /auth/logout
Invalidate current session.

**Headers:** `Authorization: Bearer <token>`

**Response:** `200 OK`
```json
{
  "message": "Logged out successfully"
}
```

---

#### GET /auth/me
Get current user profile.

**Headers:** `Authorization: Bearer <token>`

**Response:** `200 OK`
```json
{
  "id": "uuid",
  "username": "string",
  "email": "string",
  "displayName": "string",
  "phone": "string | null",
  "isEmailVerified": "boolean",
  "isPhoneVerified": "boolean",
  "mfaEnabled": "boolean",
  "kycStatus": "NONE | PENDING | IN_REVIEW | VERIFIED | REJECTED | EXPIRED",
  "role": "USER | DEVELOPER | ADMIN | SUPER_ADMIN",
  "createdAt": "ISO 8601",
  "lastLoginAt": "ISO 8601"
}
```

---

#### POST /auth/refresh
Refresh session token.

**Request Body:**
```json
{
  "refreshToken": "string (required)"
}
```

**Response:** `200 OK`
```json
{
  "sessionToken": "string",
  "refreshToken": "string (new)",
  "expiresIn": 86400
}
```

---

#### GET /auth/sessions
List user's active sessions.

**Headers:** `Authorization: Bearer <token>`

**Response:** `200 OK`
```json
{
  "sessions": [
    {
      "id": "uuid",
      "deviceType": "DESKTOP | MOBILE | TABLET",
      "deviceName": "string",
      "ipAddress": "string",
      "location": "string",
      "lastUsedAt": "ISO 8601",
      "createdAt": "ISO 8601",
      "isCurrent": "boolean"
    }
  ]
}
```

---

#### DELETE /auth/sessions/:sessionId
Revoke a specific session.

**Headers:** `Authorization: Bearer <token>`

**Response:** `200 OK`

---

### Password Management

#### POST /auth/password/change
Change password (authenticated).

**Headers:** `Authorization: Bearer <token>`

**Request Body:**
```json
{
  "currentPassword": "string (required)",
  "newPassword": "string (required, min 8 chars)"
}
```

**Response:** `200 OK`

---

#### POST /auth/password/reset/request
Request password reset email.

**Request Body:**
```json
{
  "email": "string (required)"
}
```

**Response:** `200 OK` (always succeeds to prevent enumeration)

---

#### POST /auth/password/reset
Reset password with token.

**Request Body:**
```json
{
  "token": "string (required)",
  "newPassword": "string (required)"
}
```

**Response:** `200 OK`

---

### Two-Factor Authentication (TOTP)

#### POST /auth/totp/setup
Initialize TOTP setup.

**Headers:** `Authorization: Bearer <token>`

**Response:** `200 OK`
```json
{
  "secret": "string (base32)",
  "qrCode": "data:image/png;base64,...",
  "uri": "otpauth://totp/OneClicID:username?secret=..."
}
```

---

#### POST /auth/totp/verify
Enable TOTP 2FA.

**Headers:** `Authorization: Bearer <token>`

**Request Body:**
```json
{
  "secret": "string (from setup)",
  "token": "string (6 digits)"
}
```

**Response:** `200 OK`
```json
{
  "enabled": true,
  "backupCodes": ["string", "string", "..."]
}
```

---

#### POST /auth/totp/disable
Disable TOTP 2FA.

**Headers:** `Authorization: Bearer <token>`

**Request Body:**
```json
{
  "token": "string (6 digits)"
}
```

**Response:** `200 OK`

---

## OIDC Integration

OneClicID implements OpenID Connect 1.0 with certified oidc-provider.

### Discovery Endpoint

```
GET /.well-known/openid-configuration
```

Returns OIDC provider metadata including supported endpoints, scopes, and features.

### JWKS Endpoint

```
GET /.well-known/jwks.json
```

Returns JSON Web Key Set for token verification.

### Supported Flows

| Flow | Grant Type | Use Case |
|------|------------|----------|
| Authorization Code | `authorization_code` | Web apps (recommended) |
| Authorization Code + PKCE | `authorization_code` | SPAs, mobile apps |
| Implicit | `implicit` | Legacy SPAs |
| Client Credentials | `client_credentials` | Server-to-server |
| Device Code | `urn:ietf:params:oauth:grant-type:device_code` | CLI, IoT, TV apps |
| Refresh Token | `refresh_token` | Token renewal |

### Supported Scopes

| Scope | Description | Claims |
|-------|-------------|--------|
| `openid` | Required for OIDC | `sub` |
| `profile` | Basic profile | `name`, `picture`, `locale`, etc. |
| `email` | Email access | `email`, `email_verified` |
| `phone` | Phone access | `phone_number`, `phone_number_verified` |
| `kyc` | KYC status | `kyc_status`, `kyc_level`, `kyc_verified_at` |
| `wallet` | Wallet info | `primary_public_key`, `federation_address` |
| `matrix` | Matrix info | `matrix_user_id`, `matrix_homeserver` |
| `trading` | Trading profile | `business_type`, `trading_regions` |

### Authorization Request

```
GET /oidc/auth?
  response_type=code&
  client_id=YOUR_CLIENT_ID&
  redirect_uri=https://yourapp.com/callback&
  scope=openid profile email&
  state=random_state&
  code_challenge=BASE64URL(SHA256(code_verifier))&
  code_challenge_method=S256
```

### Token Request

```bash
curl -X POST http://localhost:3002/oidc/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=authorization_code" \
  -d "code=AUTHORIZATION_CODE" \
  -d "redirect_uri=https://yourapp.com/callback" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "code_verifier=YOUR_CODE_VERIFIER"
```

### Token Response

```json
{
  "access_token": "eyJ...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "id_token": "eyJ...",
  "refresh_token": "abc...",
  "scope": "openid profile email"
}
```

### Userinfo Endpoint

```bash
curl http://localhost:3002/oidc/me \
  -H "Authorization: Bearer ACCESS_TOKEN"
```

### Device Authorization Flow

For CLI tools, smart TVs, and IoT devices:

```bash
# 1. Request device code
curl -X POST http://localhost:3002/oidc/device/auth \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "scope=openid profile"

# Response:
{
  "device_code": "...",
  "user_code": "ABCD-1234",
  "verification_uri": "http://localhost:3002/device",
  "expires_in": 600,
  "interval": 5
}

# 2. Poll for token (user completes auth in browser)
curl -X POST http://localhost:3002/oidc/token \
  -d "grant_type=urn:ietf:params:oauth:grant-type:device_code" \
  -d "device_code=DEVICE_CODE" \
  -d "client_id=YOUR_CLIENT_ID"
```

### Token TTLs

| Token Type | Lifetime |
|------------|----------|
| Access Token | 1 hour |
| ID Token | 1 hour |
| Authorization Code | 10 minutes |
| Refresh Token | 14 days |
| Device Code | 10 minutes |
| Session | 14 days |

---

## Device Identity

### Device Fingerprinting

OneClicID tracks devices for security and trust management.

#### POST /device-identity/register
Register a device with fingerprint.

**Headers:** `Authorization: Bearer <token>`

**Request Body:**
```json
{
  "deviceFingerprint": {
    "userAgent": "string",
    "platform": "string",
    "timezone": "string",
    "language": "string",
    "screenResolution": "1920x1080",
    "colorDepth": 24,
    "canvasHash": "string (optional)",
    "webglHash": "string (optional)",
    "audioHash": "string (optional)",
    "fonts": ["Arial", "Times New Roman"],
    "plugins": ["PDF Viewer"],
    "hardwareConcurrency": 8,
    "deviceMemory": 8,
    "touchSupport": false
  }
}
```

**Response:** `200 OK`
```json
{
  "device": {
    "id": "uuid",
    "stableHash": "string",
    "trustLevel": "UNKNOWN | LOW | MEDIUM | HIGH | TRUSTED",
    "deviceType": "DESKTOP | MOBILE | TABLET",
    "friendlyName": "string",
    "firstSeen": "ISO 8601",
    "lastSeen": "ISO 8601"
  },
  "isNewDevice": true,
  "requiresMFA": false
}
```

---

#### POST /device-identity/verify
Verify a device fingerprint.

**Headers:** `Authorization: Bearer <token>`

**Request Body:**
```json
{
  "fingerprint": { "...fingerprint object..." }
}
```

**Response:** `200 OK`
```json
{
  "verified": true,
  "matchType": "exact | partial | similarity",
  "device": { "...device object..." },
  "recommendation": "trust | verify | block"
}
```

---

#### GET /device-identity/devices
List user's registered devices.

**Headers:** `Authorization: Bearer <token>`

**Response:** `200 OK`
```json
{
  "devices": [
    {
      "id": "uuid",
      "friendlyName": "MacBook Pro",
      "deviceType": "DESKTOP",
      "trustLevel": "HIGH",
      "browser": "Chrome 120",
      "os": "macOS 14",
      "firstSeen": "ISO 8601",
      "lastSeen": "ISO 8601",
      "loginCount": 42
    }
  ]
}
```

---

#### DELETE /device-identity/devices/:deviceId
Remove a device.

**Headers:** `Authorization: Bearer <token>`

**Response:** `200 OK`

---

#### POST /device-identity/devices/:deviceId/trust
Explicitly trust a device.

**Headers:** `Authorization: Bearer <token>`

**Request Body:**
```json
{
  "mfaVerified": true
}
```

**Response:** `200 OK`

---

### Trust Levels

| Level | Description | Requirements |
|-------|-------------|--------------|
| `UNKNOWN` | New device | Initial state |
| `LOW` | Basic trust | MFA verified once |
| `MEDIUM` | Established | 7+ days, 5+ logins, no incidents |
| `HIGH` | Trusted | 30+ days, 20+ logins, no incidents |
| `TRUSTED` | Fully trusted | Explicit user action + MFA |

### Trust Escalation

Trust automatically escalates based on:
- Time since first seen
- Number of successful logins
- MFA completions
- Absence of security incidents

---

## Wallet Integration

OneClicID integrates with Stellar for identity and payment wallets.

### Wallet Types

| Type | Purpose | Can Transact |
|------|---------|--------------|
| `IDENTITY` | Attestations, primary ID | No |
| `MONEY` | Payments, transactions | Yes |
| `TRADING` | Exchange operations | Yes |
| `MERCHANT` | Receiving payments | Yes |
| `SAVINGS` | Long-term storage | Limited |

---

#### GET /wallet/list
List all user wallets.

**Headers:** `Authorization: Bearer <token>`

**Response:** `200 OK`
```json
{
  "wallets": [
    {
      "id": "uuid",
      "publicKey": "G...",
      "walletType": "IDENTITY",
      "name": "Identity Wallet",
      "isPrimary": true,
      "isVerified": true,
      "federationAddress": "username*clic.world",
      "network": "Stellar"
    }
  ]
}
```

---

#### GET /wallet/identity
Get the user's identity wallet.

**Headers:** `Authorization: Bearer <token>`

**Response:** `200 OK`

---

#### POST /wallet/link
Link an existing Stellar wallet.

**Headers:** `Authorization: Bearer <token>`

**Request Body:**
```json
{
  "publicKey": "G... (56 chars, Stellar format)",
  "walletType": "MONEY | TRADING | MERCHANT | SAVINGS",
  "name": "string (optional)"
}
```

**Response:** `201 Created`

---

#### GET /wallet/verify/challenge/:walletId
Get signature challenge for wallet verification.

**Headers:** `Authorization: Bearer <token>`

**Response:** `200 OK`
```json
{
  "challenge": "random_string",
  "expiresAt": "ISO 8601",
  "publicKey": "G..."
}
```

---

#### POST /wallet/verify
Verify wallet ownership with Stellar signature.

**Headers:** `Authorization: Bearer <token>`

**Request Body:**
```json
{
  "walletId": "uuid",
  "challenge": "string",
  "signature": "Stellar signature"
}
```

**Response:** `200 OK`

---

#### DELETE /wallet/:walletId
Remove a wallet (cannot remove identity wallet).

**Headers:** `Authorization: Bearer <token>`

**Response:** `200 OK`

---

## Matrix Integration

OneClicID can use Matrix homeservers for federated authentication.

### Matrix Registration

#### POST /auth/register/matrix
Register with Matrix homeserver integration.

**Request Body:**
```json
{
  "username": "string (required)",
  "email": "string (required)",
  "password": "string (required)",
  "displayName": "string (optional)",
  "homeserver": "https://matrix.clic.world (required)",

  "legalFirstName": "string (optional, MiFIR)",
  "legalSurname": "string (optional, MiFIR)",
  "dateOfBirth": "YYYY-MM-DD (optional)",
  "gender": "MALE | FEMALE | OTHER | PREFER_NOT_TO_SAY (optional)",
  "nationality": "ISO 3166-1 alpha-2 (optional)",
  "countryOfResidence": "ISO 3166-1 alpha-2 (optional)",

  "company": "string (optional)",
  "businessType": "string (optional)",
  "tradingRegions": ["EU", "US"] (optional),
  "preferredCurrencies": ["USD", "EUR"] (optional)
}
```

**Response:** `201 Created`
```json
{
  "id": "uuid",
  "username": "string",
  "email": "string",
  "matrixUserId": "@username:matrix.clic.world",
  "matrixAccessToken": "string",
  "matrixDeviceId": "string",
  "federationAddress": "username*clic.world",
  "primaryPublicKey": "G...",
  "wallets": [
    { "type": "IDENTITY", "publicKey": "G..." },
    { "type": "MONEY", "publicKey": "G..." }
  ]
}
```

---

### Matrix Login

#### POST /auth/login/matrix
Login with Matrix credentials.

**Request Body:**
```json
{
  "username": "string (required)",
  "password": "string (required)",
  "homeserver": "https://matrix.clic.world (required)"
}
```

**Response:** `200 OK`
```json
{
  "sessionToken": "string",
  "refreshToken": "string",
  "expiresIn": 86400,
  "matrixAccessToken": "string",
  "matrixDeviceId": "string",
  "user": { "...user object..." }
}
```

---

### Matrix Identity Server Routes

OneClicID implements Matrix Identity Server v2 API.

#### POST /_matrix/identity/v2/validate/email/requestToken
Request email verification token.

**Request Body:**
```json
{
  "client_secret": "string",
  "email": "string",
  "send_attempt": 1,
  "next_link": "string (optional)"
}
```

**Response:**
```json
{
  "sid": "session_id"
}
```

---

#### POST /_matrix/identity/v2/validate/email/submitToken
Submit verification token.

**Request Body:**
```json
{
  "sid": "string",
  "client_secret": "string",
  "token": "string"
}
```

**Response:**
```json
{
  "success": true
}
```

---

#### POST /_matrix/identity/v2/3pid/bind
Bind 3PID to Matrix ID.

**Request Body:**
```json
{
  "sid": "string",
  "client_secret": "string",
  "mxid": "@user:server.com"
}
```

**Response:**
```json
{
  "address": "email@example.com",
  "medium": "email",
  "mxid": "@user:server.com",
  "ts": 1234567890
}
```

---

#### POST /_matrix/identity/v2/lookup
Privacy-preserving identifier lookup.

**Request Body:**
```json
{
  "algorithm": "sha256",
  "pepper": "string (from hash_details)",
  "addresses": ["hashed_address_1", "hashed_address_2"]
}
```

**Response:**
```json
{
  "mappings": {
    "hashed_address_1": "@user:server.com"
  }
}
```

---

## Admin API

All admin routes require `ADMIN` or `SUPER_ADMIN` role.

### Dashboard

#### GET /admin/dashboard/stats
Get system statistics.

**Headers:** `Authorization: Bearer <admin_token>`

**Response:** `200 OK`
```json
{
  "users": {
    "total": 1000,
    "active": 950,
    "suspended": 10,
    "newToday": 5,
    "newThisWeek": 25,
    "newThisMonth": 100
  },
  "usersByRole": {
    "USER": 980,
    "DEVELOPER": 15,
    "ADMIN": 4,
    "SUPER_ADMIN": 1
  },
  "usersByKycStatus": {
    "NONE": 500,
    "PENDING": 100,
    "VERIFIED": 350,
    "REJECTED": 50
  },
  "oauthClients": {
    "total": 20,
    "active": 18
  },
  "recentLogins": 150,
  "recentFailedLogins": 10
}
```

---

### User Management

#### GET /admin/users
List users with pagination.

**Query Parameters:**
- `page` (default: 1)
- `limit` (default: 20, max: 100)
- `search` (username, email, displayName)
- `sortBy` (createdAt, lastLoginAt, username, email)
- `sortOrder` (asc, desc)

**Response:** `200 OK`
```json
{
  "users": [...],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 1000,
    "totalPages": 50
  }
}
```

---

#### PUT /admin/users/:userId/role
Update user role.

**Request Body:**
```json
{
  "role": "USER | DEVELOPER | ADMIN | SUPER_ADMIN",
  "adminScope": "string (optional, for ADMIN role)"
}
```

---

#### POST /admin/users/:userId/suspend
Suspend user account.

**Request Body:**
```json
{
  "reason": "string (required)",
  "duration": "number (optional, days)"
}
```

---

#### POST /admin/users/:userId/unsuspend
Unsuspend user account.

---

#### DELETE /admin/users/:userId
Delete user account (cascades to related data).

---

### OAuth Client Management

#### POST /admin/oauth-clients
Create OAuth client.

**Request Body:**
```json
{
  "name": "string (required)",
  "redirectUris": ["https://app.com/callback"],
  "grantTypes": ["authorization_code", "refresh_token"],
  "scopes": ["openid", "profile", "email"],
  "isFirstParty": false,
  "requireConsent": true,
  "requirePkce": true
}
```

**Response:** `201 Created`
```json
{
  "clientId": "string",
  "clientSecret": "string (ONLY SHOWN ONCE)",
  "name": "string",
  "redirectUris": [...],
  "...other fields..."
}
```

---

#### POST /admin/oauth-clients/:clientId/rotate-secret
Rotate client secret.

**Response:** `200 OK`
```json
{
  "clientSecret": "new_secret (ONLY SHOWN ONCE)"
}
```

---

### Audit Logs

#### GET /admin/audit-logs
Get audit logs.

**Query Parameters:**
- `page`, `limit`
- `userId` - Filter by user
- `action` - Filter by action type
- `startDate`, `endDate` - Date range

**Response:** `200 OK`
```json
{
  "logs": [
    {
      "id": "uuid",
      "userId": "uuid",
      "action": "USER_LOGIN",
      "resource": "session",
      "resourceId": "uuid",
      "ipAddress": "192.168.1.1",
      "userAgent": "...",
      "success": true,
      "metadata": {},
      "createdAt": "ISO 8601"
    }
  ],
  "pagination": {...}
}
```

---

### Device Administration

#### POST /device-identity/admin/block
Block a device globally.

**Request Body:**
```json
{
  "stableHash": "string (device stable hash)",
  "reason": "string",
  "expiresAt": "ISO 8601 (optional)"
}
```

---

#### POST /device-identity/admin/unblock
Unblock a device.

**Request Body:**
```json
{
  "stableHash": "string"
}
```

---

#### GET /device-identity/admin/events
Get device security events.

**Query Parameters:**
- `deviceId`, `userId`
- `eventType` (REGISTERED, VERIFIED, VERIFICATION_FAILED, etc.)
- `severity` (DEBUG, INFO, WARNING, ERROR, CRITICAL)
- `startDate`, `endDate`
- `limit`, `offset`

---

## Data Models

### User

```typescript
interface User {
  id: string;
  createdAt: Date;
  updatedAt: Date;

  // Identity
  username: string;
  email: string;
  displayName?: string;
  phone?: string;
  locale: string;
  timezone: string;

  // Status
  isActive: boolean;
  isEmailVerified: boolean;
  isPhoneVerified: boolean;
  isSuspended: boolean;
  suspendedAt?: Date;
  suspendedReason?: string;

  // Authentication
  mfaEnabled: boolean;
  lastLoginAt?: Date;
  lastLoginIp?: string;
  loginCount: number;
  failedLoginCount: number;
  lockedUntil?: Date;

  // KYC
  kycStatus: 'NONE' | 'PENDING' | 'IN_REVIEW' | 'VERIFIED' | 'REJECTED' | 'EXPIRED';
  kycLevel?: number;
  kycVerifiedAt?: Date;
  kycExpiresAt?: Date;

  // Role
  role: 'USER' | 'DEVELOPER' | 'ADMIN' | 'SUPER_ADMIN';
  adminScope?: string;

  // MiFIR Compliance
  legalFirstName?: string;
  legalSurname?: string;
  dateOfBirth?: Date;
  gender?: 'MALE' | 'FEMALE' | 'OTHER' | 'PREFER_NOT_TO_SAY';
  nationality?: string;
  countryOfResidence?: string;
  nationalId?: string;
  nationalIdType?: 'PASSPORT' | 'NATIONAL_ID' | 'TAX_ID' | 'DRIVERS_LICENSE' | 'RESIDENCE_PERMIT';
  nationalIdCountry?: string;

  // Matrix Integration
  matrixUserId?: string;
  matrixHomeServer?: string;

  // Stellar Integration
  primaryPublicKey?: string;
  federationAddress?: string;

  // Business Profile
  company?: string;
  businessType?: string;
  tradingRegions?: string[];
  preferredCurrencies?: string[];
}
```

### Wallet

```typescript
interface Wallet {
  id: string;
  userId: string;
  network: string;
  publicKey: string;
  name?: string;
  walletType: 'IDENTITY' | 'MONEY' | 'TRADING' | 'MERCHANT' | 'SAVINGS';
  isPrimary: boolean;
  isVerified: boolean;
  verifiedAt?: Date;
  federationAddress?: string;
}
```

### Device

```typescript
interface Device {
  id: string;
  userId: string;
  stableHash: string;
  fullHash: string;
  trustLevel: 'UNKNOWN' | 'LOW' | 'MEDIUM' | 'HIGH' | 'TRUSTED';
  deviceType: 'DESKTOP' | 'MOBILE' | 'TABLET' | 'UNKNOWN';
  friendlyName?: string;
  browser?: string;
  browserVersion?: string;
  os?: string;
  osVersion?: string;
  firstSeen: Date;
  lastSeen: Date;
  loginCount: number;
  mfaCompletions: number;
}
```

### OAuth Client

```typescript
interface OAuthClient {
  id: string;
  clientId: string;
  name: string;
  description?: string;
  logoUri?: string;
  redirectUris: string[];
  postLogoutRedirectUris?: string[];
  grantTypes: string[];
  responseTypes: string[];
  scopes: string[];
  tokenEndpointAuthMethod: string;
  isActive: boolean;
  isFirstParty: boolean;
  requirePkce: boolean;
  requireConsent: boolean;
}
```

---

## Error Handling

### Error Response Format

```json
{
  "error": "error_code",
  "message": "Human-readable description",
  "details": {}
}
```

### Common Error Codes

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | `invalid_request` | Malformed request |
| 400 | `validation_error` | Input validation failed |
| 401 | `unauthorized` | Missing/invalid authentication |
| 401 | `invalid_credentials` | Wrong username/password |
| 401 | `session_expired` | Session has expired |
| 401 | `mfa_required` | 2FA verification needed |
| 403 | `forbidden` | Insufficient permissions |
| 403 | `account_suspended` | Account is suspended |
| 403 | `account_locked` | Account temporarily locked |
| 404 | `not_found` | Resource not found |
| 409 | `conflict` | Resource already exists |
| 429 | `rate_limited` | Too many requests |
| 500 | `internal_error` | Server error |

### OIDC Errors

| Error | Description |
|-------|-------------|
| `invalid_client` | Unknown client_id or wrong secret |
| `invalid_grant` | Invalid authorization code or refresh token |
| `invalid_scope` | Requested scope not allowed |
| `access_denied` | User denied consent |
| `authorization_pending` | Device flow: user hasn't completed auth |
| `slow_down` | Device flow: polling too fast |
| `expired_token` | Device code expired |

---

## Security Best Practices

### For SDK Implementers

1. **Always use HTTPS** in production
2. **Store tokens securely** (encrypted storage, not localStorage for sensitive apps)
3. **Implement PKCE** for public clients (SPAs, mobile apps)
4. **Validate state parameter** to prevent CSRF
5. **Use short-lived access tokens** with refresh token rotation
6. **Implement device fingerprinting** for additional security
7. **Enable 2FA** for sensitive operations

### Token Storage Recommendations

| Platform | Recommendation |
|----------|----------------|
| Web (SPA) | httpOnly cookies or in-memory |
| Mobile | Secure Keychain/Keystore |
| Desktop | OS credential manager |
| Server | Encrypted environment variables |

### Rate Limits

| Endpoint | Limit |
|----------|-------|
| `/auth/login` | 5 requests/minute/IP |
| `/auth/register` | 3 requests/minute/IP |
| `/auth/password/reset/request` | 3 requests/hour/email |
| General API | 100 requests/minute/user |

### Security Headers

OneClicID sets these security headers via Helmet:
- `Strict-Transport-Security`
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `X-XSS-Protection: 1; mode=block`
- `Content-Security-Policy`

---

## Appendix

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `PORT` | Server port | 3002 |
| `DATABASE_URL` | PostgreSQL connection string | - |
| `REDIS_URL` | Redis connection string | - |
| `JWT_SECRET` | JWT signing secret | - |
| `OIDC_ISSUER` | OIDC issuer URL | http://localhost:3002 |
| `STELLAR_NETWORK` | Stellar network (testnet/public) | testnet |
| `MATRIX_HOMESERVER` | Default Matrix homeserver | - |

### Health Check

```
GET /health
```

**Response:** `200 OK`
```json
{
  "status": "healthy",
  "timestamp": "ISO 8601",
  "version": "1.0.0"
}
```

---

## Changelog

### v1.0.0 (January 2026)
- Initial SDK documentation
- Complete API reference
- OIDC integration guide
- Device identity documentation
- Matrix integration documentation

---

*Generated for the Clic Ecosystem*
