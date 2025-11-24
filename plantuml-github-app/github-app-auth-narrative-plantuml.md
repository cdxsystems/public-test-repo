<!--example file:

    - may be dleted
    - has broken references to simulator
-->

# GitHub App Authentication Flow: A Complete Guide

## Introduction

GitHub Apps provide a powerful and secure way to integrate with GitHub's platform. Unlike OAuth Apps that act on behalf of users, GitHub Apps have their own identity and can operate independently with fine-grained permissions. This guide walks through the complete authentication flow, from installation to making authenticated API calls.

## Overview

The GitHub App authentication model is built on three core concepts:

1. **JWT (JSON Web Token)**: A short-lived token that authenticates your app itself
2. **Installation Access Token**: A token scoped to a specific installation with limited permissions
3. **Webhook Verification**: Cryptographic proof that events come from GitHub

This multi-step authentication process ensures security while providing the flexibility to act on behalf of different organizations and users who install your app.

## The Complete Flow

![GitHub App Authentication Flow](http://172.17.0.3:8080/proxy?src=https://raw.githubusercontent.com/YOUR_USERNAME/YOUR_REPO/main/github-app-auth-flow.puml)

The diagram above shows the complete authentication and webhook flow. Let's walk through each phase in detail.

## Phase 1: Installation

When a user decides to install your GitHub App, they initiate a flow that establishes the relationship between your app and their repositories.

### Installation Steps

1. **User visits your app's installation page**: This can be from GitHub Marketplace or a direct link to `https://github.com/apps/your-app-name/installations/new`

2. **Repository selection**: GitHub presents the user with options to:
   - Install on all repositories
   - Select specific repositories

3. **Permission review**: The user sees exactly what permissions your app is requesting (contents read/write, pull requests, issues, etc.)

4. **Installation confirmation**: Once confirmed, GitHub creates an installation record

5. **Webhook delivery**: GitHub sends an `installation.created` webhook event to your server with crucial information:
   ```json
   {
     "action": "created",
     "installation": {
       "id": 12345,
       "account": {
         "login": "octocat",
         "type": "Organization"
       }
     },
     "repositories": [
       {"id": 1, "name": "repo1"},
       {"id": 2, "name": "repo2"}
     ]
   }
   ```

Your app server must store the `installation.id` as this is the key to generating installation access tokens later.

## Phase 2: JWT Generation

Every time your app needs to authenticate with GitHub, it starts by generating a JWT. This token proves that the request is coming from your app.

### JWT Structure

The JWT consists of three parts: header, payload, and signature.

**Header**:
```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

**Payload**:
```json
{
  "iat": 1634567890,
  "exp": 1634568490,
  "iss": "123456"
}
```

- `iat` (issued at): Current Unix timestamp
- `exp` (expiration): Timestamp when the JWT expires (max 10 minutes from `iat`)
- `iss` (issuer): Your GitHub App's ID

**Signature**: Created by signing the header and payload with your app's private key using RS256 algorithm.

### Implementation Example

```javascript
const jwt = require('jsonwebtoken');
const fs = require('fs');

function generateJWT(appId, privateKeyPath) {
  const privateKey = fs.readFileSync(privateKeyPath, 'utf8');

  const now = Math.floor(Date.now() / 1000);

  const payload = {
    iat: now,
    exp: now + (10 * 60), // 10 minutes
    iss: appId
  };

  const token = jwt.sign(payload, privateKey, { algorithm: 'RS256' });

  return token;
}
```

### Important Considerations

- **10-minute maximum lifetime**: GitHub enforces this limit for security
- **Private key security**: Never commit your private key to version control
- **Key rotation**: GitHub allows generating new private keys without downtime

## Phase 3: Obtaining Installation Access Token

With a valid JWT, your app can now request an installation access token. This token is scoped to a specific installation and has the permissions granted during installation.

### Token Request

```javascript
async function getInstallationToken(installationId, jwt) {
  const response = await fetch(
    `https://api.github.com/app/installations/${installationId}/access_tokens`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${jwt}`,
        'Accept': 'application/vnd.github+json',
        'X-GitHub-Api-Version': '2022-11-28'
      }
    }
  );

  const data = await response.json();

  return {
    token: data.token,
    expiresAt: data.expires_at,
    permissions: data.permissions,
    repositories: data.repositories
  };
}
```

### Token Response

GitHub returns a token with metadata:

```json
{
  "token": "ghs_16C7e42F292c6912E7710c838347Ae178B4a",
  "expires_at": "2025-10-15T10:00:00Z",
  "permissions": {
    "contents": "write",
    "pull_requests": "write",
    "metadata": "read"
  },
  "repositories": [
    {
      "id": 1296269,
      "name": "Hello-World",
      "full_name": "octocat/Hello-World"
    }
  ]
}
```

### Token Characteristics

- **One-hour lifetime**: Tokens automatically expire after 60 minutes
- **Scoped permissions**: Limited to what was granted during installation
- **Repository scoped**: Only works for repositories included in the installation
- **Independent rate limits**: Each installation gets 5,000 requests/hour

### Token Caching Strategy

Since tokens last one hour, implement caching to avoid unnecessary token generation:

```javascript
class TokenCache {
  constructor() {
    this.cache = new Map();
  }

  get(installationId) {
    const cached = this.cache.get(installationId);

    if (!cached) return null;

    // Check if token expires in next 5 minutes
    const expiresAt = new Date(cached.expiresAt);
    const now = new Date();
    const bufferMs = 5 * 60 * 1000; // 5 minutes

    if (expiresAt - now < bufferMs) {
      this.cache.delete(installationId);
      return null;
    }

    return cached.token;
  }

  set(installationId, token, expiresAt) {
    this.cache.set(installationId, { token, expiresAt });
  }
}
```

## Phase 4: Making API Calls

With an installation access token, your app can now interact with GitHub's API on behalf of the installation.

### API Request Pattern

```javascript
async function createPullRequestComment(owner, repo, pullNumber, body, token) {
  const response = await fetch(
    `https://api.github.com/repos/${owner}/${repo}/issues/${pullNumber}/comments`,
    {
      method: 'POST',
      headers: {
        'Authorization': `token ${token}`,
        'Accept': 'application/vnd.github+json',
        'X-GitHub-Api-Version': '2022-11-28',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ body })
    }
  );

  return response.json();
}
```

### Permission Enforcement

GitHub validates every API call against the token's permissions:

| Permission | API Access |
|------------|-----------|
| `contents: read` | Read repository files, commits, branches |
| `contents: write` | Create/update files, branches, tags |
| `pull_requests: read` | Read PR details, comments, reviews |
| `pull_requests: write` | Create PRs, comments, request reviews |
| `issues: write` | Create/update issues, labels, milestones |
| `checks: write` | Create/update check runs and suites |

If your app attempts an action without proper permissions, GitHub returns a `403 Forbidden` error.

## Phase 5: Webhook Event Handling

Webhooks are how GitHub notifies your app about events in real-time. This is the primary way apps stay synchronized with repository activity.

### Webhook Security

Every webhook includes a signature header that proves authenticity:

```
X-Hub-Signature-256: sha256=abc123def456...
```

This signature is an HMAC-SHA256 hash of the request body using your webhook secret.

### Verification Implementation

```javascript
const crypto = require('crypto');

function verifyWebhookSignature(payload, signature, secret) {
  const hmac = crypto.createHmac('sha256', secret);
  hmac.update(JSON.stringify(payload));
  const calculatedSignature = `sha256=${hmac.digest('hex')}`;

  // Use constant-time comparison to prevent timing attacks
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(calculatedSignature)
  );
}

// Express.js middleware example
function webhookVerifier(req, res, next) {
  const signature = req.headers['x-hub-signature-256'];

  if (!signature) {
    return res.status(401).send('Missing signature');
  }

  const isValid = verifyWebhookSignature(
    req.body,
    signature,
    process.env.WEBHOOK_SECRET
  );

  if (!isValid) {
    return res.status(401).send('Invalid signature');
  }

  next();
}
```

### Common Webhook Events

| Event | Trigger | Use Case |
|-------|---------|----------|
| `pull_request` | PR opened/updated/closed | Run CI checks, enforce policies |
| `push` | Commits pushed | Deploy, run tests |
| `issues` | Issue opened/updated | Auto-label, assign, triage |
| `pull_request_review` | PR reviewed | Track approvals, auto-merge |
| `check_suite` | Check suite requested | Run custom checks |
| `installation` | App installed/uninstalled | Setup/cleanup |
| `installation_repositories` | Repos added/removed | Update permissions |

### Webhook Processing Flow

```javascript
async function handleWebhook(event, payload) {
  const installationId = payload.installation.id;

  // Get or refresh token
  let token = tokenCache.get(installationId);

  if (!token) {
    const jwt = generateJWT(appId, privateKeyPath);
    const tokenData = await getInstallationToken(installationId, jwt);
    token = tokenData.token;
    tokenCache.set(installationId, token, tokenData.expiresAt);
  }

  // Handle specific events
  switch (event) {
    case 'pull_request':
      if (payload.action === 'opened') {
        await handlePullRequestOpened(payload, token);
      }
      break;

    case 'push':
      await handlePush(payload, token);
      break;

    case 'issues':
      if (payload.action === 'opened') {
        await handleIssueOpened(payload, token);
      }
      break;
  }
}

async function handlePullRequestOpened(payload, token) {
  const { repository, pull_request } = payload;

  // Create a check run
  await fetch(
    `https://api.github.com/repos/${repository.full_name}/check-runs`,
    {
      method: 'POST',
      headers: {
        'Authorization': `token ${token}`,
        'Accept': 'application/vnd.github+json'
      },
      body: JSON.stringify({
        name: 'Code Quality Check',
        head_sha: pull_request.head.sha,
        status: 'in_progress',
        started_at: new Date().toISOString()
      })
    }
  );

  // Run your checks...
  // Update check run with results...
}
```

## State Management

Understanding the state transitions in GitHub App authentication is crucial for building reliable integrations.

![GitHub App Authentication States](http://172.17.0.3:8080/proxy?src=https://raw.githubusercontent.com/YOUR_USERNAME/YOUR_REPO/main/github-app-auth-flow.puml#state-machine)

### State Descriptions

1. **AppRegistered**: Your app exists on GitHub with an app ID and private key
2. **Installed**: A user/org has installed your app on their repositories
3. **NoToken**: No valid authentication token exists
4. **GeneratingJWT**: Creating a JWT to authenticate as the app
5. **JWTValid**: JWT is valid and can be used to request installation tokens
6. **RequestingToken**: Exchanging JWT for an installation access token
7. **TokenValid**: Installation token is valid and can be used for API calls
8. **MakingAPICall**: Actively using the token to interact with GitHub API

### Transition Triggers

- **JWT Expiration** (10 minutes): Automatically transition from `JWTValid` to `GeneratingJWT`
- **Token Expiration** (1 hour): Automatically transition from `TokenValid` to `GeneratingJWT`
- **API Call Need**: Transition from `NoToken` or expired states to `GeneratingJWT`
- **Uninstallation**: Transition from any state to terminal state

## Security Considerations

![GitHub App Security Flow](http://172.17.0.3:8080/proxy?src=https://raw.githubusercontent.com/YOUR_USERNAME/YOUR_REPO/main/github-app-auth-flow.puml#security-flow)

### Security Best Practices

#### 1. Private Key Protection

Your private key is the most sensitive credential:

```bash
# Generate secure permissions
chmod 600 private-key.pem

# Never commit to version control
echo "*.pem" >> .gitignore

# Use environment variables or secret management
export GITHUB_APP_PRIVATE_KEY_PATH="/secure/path/private-key.pem"

# Or use secret managers
# AWS Secrets Manager, HashiCorp Vault, etc.
```

#### 2. Webhook Secret Validation

Always verify webhook signatures before processing:

```javascript
// GOOD: Verify before processing
app.post('/webhooks', webhookVerifier, async (req, res) => {
  await processWebhook(req.body);
  res.status(200).send('OK');
});

// BAD: Process without verification
app.post('/webhooks', async (req, res) => {
  await processWebhook(req.body); // Dangerous!
  res.status(200).send('OK');
});
```

#### 3. Token Storage

Never log or expose tokens:

```javascript
// GOOD: Secure logging
logger.info('Token obtained', {
  installationId,
  expiresAt,
  permissions: Object.keys(permissions)
});

// BAD: Leaking tokens
logger.info('Token obtained', { token }); // Never do this!
```

#### 4. HTTPS Only

GitHub requires HTTPS for webhook URLs:

```javascript
// Production webhook endpoint must use HTTPS
const webhookUrl = 'https://your-app.com/webhooks';

// For local development, use tools like:
// - ngrok: ngrok http 3000
// - smee.io: https://smee.io/
```

#### 5. Rate Limit Handling

Respect GitHub's rate limits:

```javascript
async function makeAPICall(url, token) {
  const response = await fetch(url, {
    headers: { 'Authorization': `token ${token}` }
  });

  // Check rate limit headers
  const remaining = response.headers.get('X-RateLimit-Remaining');
  const reset = response.headers.get('X-RateLimit-Reset');

  if (response.status === 429) {
    const resetDate = new Date(reset * 1000);
    const waitMs = resetDate - Date.now();

    console.log(`Rate limited. Waiting ${waitMs}ms`);
    await sleep(waitMs);

    // Retry the request
    return makeAPICall(url, token);
  }

  return response;
}
```

## Architecture Overview

![GitHub App Architecture](http://172.17.0.3:8080/proxy?src=https://raw.githubusercontent.com/YOUR_USERNAME/YOUR_REPO/main/github-app-auth-flow.puml#architecture)

### Component Responsibilities

#### Webhook Handler
- Receives webhook events from GitHub
- Verifies webhook signatures
- Routes events to appropriate handlers
- Manages response timing (must respond within 10 seconds)

#### JWT Generator
- Loads private key from secure storage
- Creates JWT with proper claims
- Handles JWT expiration and regeneration
- Signs tokens using RS256 algorithm

#### Token Manager
- Requests installation tokens using JWT
- Caches tokens with expiration tracking
- Handles token refresh on expiration
- Provides tokens to API client

#### API Client
- Makes authenticated requests to GitHub API
- Handles rate limiting and retries
- Processes API responses
- Manages error conditions

#### Business Logic
- Implements app-specific functionality
- Responds to webhook events
- Orchestrates API calls
- Maintains application state

### Scalability Considerations

#### Horizontal Scaling

GitHub Apps scale naturally because each installation has independent rate limits:

```
Installation 1 (Microsoft) → 5,000 req/hr
Installation 2 (Google)    → 5,000 req/hr
Installation 3 (Facebook)  → 5,000 req/hr
Total capacity: 15,000 req/hr
```

#### Token Caching Strategies

**Single-Instance (In-Memory)**:
```javascript
const tokenCache = new Map();
```

**Multi-Instance (Redis)**:
```javascript
const redis = require('redis');
const client = redis.createClient();

async function cacheToken(installationId, token, expiresAt) {
  const ttl = Math.floor((new Date(expiresAt) - Date.now()) / 1000);
  await client.setex(
    `gh-token:${installationId}`,
    ttl,
    JSON.stringify({ token, expiresAt })
  );
}
```

**Multi-Region (DynamoDB)**:
```javascript
const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB.DocumentClient();

async function cacheToken(installationId, token, expiresAt) {
  await dynamodb.put({
    TableName: 'github-tokens',
    Item: {
      installation_id: installationId,
      token,
      expires_at: expiresAt,
      ttl: Math.floor(new Date(expiresAt) / 1000)
    }
  }).promise();
}
```

## Permission Model

![GitHub App Permission Model](http://172.17.0.3:8080/proxy?src=https://raw.githubusercontent.com/YOUR_USERNAME/YOUR_REPO/main/github-app-auth-flow.puml#permission-model)

### Permission Scoping

GitHub Apps use a principle of least privilege. Each installation token is restricted by:

1. **Requested Permissions**: What your app asks for during registration
2. **Granted Permissions**: What the user approves during installation
3. **Repository Access**: Which repositories were selected
4. **Time Limit**: Tokens expire after 1 hour

### Available Permissions

| Resource | Levels | API Access |
|----------|--------|-----------|
| **Actions** | read, write | Workflow runs, artifacts, secrets |
| **Checks** | read, write | Check runs, check suites |
| **Contents** | read, write | Repository files, commits, branches, tags |
| **Deployments** | read, write | Deployment statuses, environments |
| **Issues** | read, write | Issues, labels, assignees, milestones |
| **Metadata** | read | Repository metadata (always granted) |
| **Pull requests** | read, write | PRs, reviews, comments |
| **Secrets** | read, write | Dependabot secrets |
| **Security events** | read, write | Code scanning, secret scanning |
| **Statuses** | read, write | Commit statuses |
| **Workflows** | write | Update workflow files |

### Permission Examples

**CI/CD Bot**:
```yaml
permissions:
  checks: write
  contents: read
  pull_requests: write
  statuses: write
```

**Dependency Update Bot**:
```yaml
permissions:
  contents: write
  pull_requests: write
  metadata: read
```

**Security Scanner**:
```yaml
permissions:
  contents: read
  security_events: write
  pull_requests: write
```

## Rate Limiting

GitHub Apps enjoy generous rate limits compared to OAuth Apps:

### Rate Limit Comparison

| Authentication | Rate Limit | Scope |
|----------------|-----------|-------|
| **Unauthenticated** | 60/hour | Per IP |
| **OAuth User Token** | 5,000/hour | Per user |
| **GitHub App JWT** | 5,000/hour | Per app |
| **Installation Token** | 5,000/hour | **Per installation** |

### Why Installation-Based Limits Matter

If your app is installed on 100 organizations:
- Total capacity: 500,000 requests/hour
- Each organization gets dedicated 5,000/hour
- One organization's usage doesn't affect others

### Monitoring Rate Limits

Every API response includes rate limit headers:

```javascript
const response = await fetch(url, {
  headers: { 'Authorization': `token ${token}` }
});

const rateLimit = {
  limit: response.headers.get('X-RateLimit-Limit'),
  remaining: response.headers.get('X-RateLimit-Remaining'),
  reset: response.headers.get('X-RateLimit-Reset'),
  used: response.headers.get('X-RateLimit-Used'),
  resource: response.headers.get('X-RateLimit-Resource')
};

console.log(`Rate limit: ${rateLimit.remaining}/${rateLimit.limit}`);
console.log(`Resets at: ${new Date(rateLimit.reset * 1000)}`);
```

### Handling Rate Limit Exhaustion

```javascript
async function makeAPICallWithRetry(url, token, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    const response = await fetch(url, {
      headers: { 'Authorization': `token ${token}` }
    });

    if (response.status === 429) {
      const resetTime = response.headers.get('X-RateLimit-Reset');
      const waitMs = (resetTime * 1000) - Date.now();

      console.log(`Rate limited on attempt ${attempt}/${maxRetries}`);
      console.log(`Waiting ${Math.ceil(waitMs / 1000)} seconds`);

      if (attempt < maxRetries) {
        await sleep(waitMs + 1000); // Extra second buffer
        continue;
      }

      throw new Error('Rate limit exceeded after max retries');
    }

    if (response.ok) {
      return response.json();
    }

    throw new Error(`API call failed: ${response.status}`);
  }
}
```

## Error Handling

### Common Error Scenarios

#### 1. Invalid JWT

```json
{
  "message": "A JSON web token could not be decoded",
  "documentation_url": "https://docs.github.com/rest/apps/apps#create-an-installation-access-token-for-an-app"
}
```

**Causes**:
- Expired JWT (over 10 minutes old)
- Invalid signature (wrong private key)
- Malformed JWT structure
- Incorrect app_id in `iss` claim

**Solution**:
```javascript
try {
  const jwt = generateJWT(appId, privateKeyPath);
} catch (error) {
  console.error('JWT generation failed:', error);
  // Check private key file exists and is valid
  // Verify app_id is correct
  // Ensure system clock is synchronized
}
```

#### 2. Installation Not Found

```json
{
  "message": "Not Found",
  "documentation_url": "https://docs.github.com/rest/apps/apps#create-an-installation-access-token-for-an-app"
}
```

**Causes**:
- App was uninstalled
- Wrong installation_id
- JWT doesn't match the app that owns the installation

**Solution**:
```javascript
async function getTokenSafe(installationId) {
  try {
    return await getInstallationToken(installationId, jwt);
  } catch (error) {
    if (error.status === 404) {
      // Remove installation from database
      await db.installations.delete(installationId);
      console.log(`Installation ${installationId} no longer exists`);
    }
    throw error;
  }
}
```

#### 3. Insufficient Permissions

```json
{
  "message": "Resource not accessible by integration",
  "documentation_url": "https://docs.github.com/rest/reference/repos#get-a-repository"
}
```

**Causes**:
- Attempting action without required permission
- Repository not included in installation
- Permission level too low (have `read`, need `write`)

**Solution**:
```javascript
async function createFileWithPermissionCheck(repo, path, content, token) {
  try {
    await createFile(repo, path, content, token);
  } catch (error) {
    if (error.status === 403) {
      console.error('Missing required permission: contents:write');
      // Guide user to reinstall with correct permissions
      // Or request permission upgrade
    }
    throw error;
  }
}
```

#### 4. Invalid Webhook Signature

**Causes**:
- Wrong webhook secret
- Request body modified in transit
- Signature header missing
- Replay attack

**Solution**:
```javascript
function handleWebhookSecurely(req, res) {
  const signature = req.headers['x-hub-signature-256'];
  const delivery = req.headers['x-github-delivery'];

  if (!signature) {
    console.error('Missing signature header');
    return res.status(401).send('Unauthorized');
  }

  if (!verifyWebhookSignature(req.body, signature, webhookSecret)) {
    console.error('Invalid signature', { delivery });
    return res.status(401).send('Invalid signature');
  }

  // Process webhook
}
```

## Testing and Development

### Local Development Setup

#### 1. Use a Tunneling Service

GitHub needs to send webhooks to your local server:

```bash
# Using ngrok
ngrok http 3000

# Using smee.io
npm install -g smee-client
smee -u https://smee.io/your-channel -t http://localhost:3000/webhooks
```

#### 2. Create a Test GitHub App

Create a separate GitHub App for development:

```yaml
Development App:
  Name: MyApp Dev
  Webhook URL: https://abc123.ngrok.io/webhooks
  Permissions: (same as production)

Production App:
  Name: MyApp
  Webhook URL: https://api.myapp.com/webhooks
  Permissions: (same as development)
```

#### 3. Environment Configuration

```bash
# .env.development
GITHUB_APP_ID=12345
GITHUB_APP_PRIVATE_KEY_PATH=./dev-private-key.pem
GITHUB_WEBHOOK_SECRET=dev_secret_123
WEBHOOK_URL=https://abc123.ngrok.io/webhooks

# .env.production
GITHUB_APP_ID=67890
GITHUB_APP_PRIVATE_KEY_PATH=/secure/prod-private-key.pem
GITHUB_WEBHOOK_SECRET=prod_secret_456
WEBHOOK_URL=https://api.myapp.com/webhooks
```

### Testing Strategies

#### Unit Tests

```javascript
const { generateJWT, verifyWebhookSignature } = require('./auth');

describe('JWT Generation', () => {
  it('should generate valid JWT', () => {
    const jwt = generateJWT('12345', './test-private-key.pem');
    const decoded = decodeJWT(jwt);

    expect(decoded.iss).toBe('12345');
    expect(decoded.exp - decoded.iat).toBeLessThanOrEqual(600);
  });

  it('should throw on invalid private key', () => {
    expect(() => {
      generateJWT('12345', './nonexistent.pem');
    }).toThrow();
  });
});

describe('Webhook Verification', () => {
  it('should verify valid signature', () => {
    const payload = { action: 'opened' };
    const secret = 'test_secret';
    const signature = generateSignature(payload, secret);

    expect(verifyWebhookSignature(payload, signature, secret)).toBe(true);
  });

  it('should reject invalid signature', () => {
    const payload = { action: 'opened' };
    const signature = 'sha256=invalid';

    expect(verifyWebhookSignature(payload, signature, 'secret')).toBe(false);
  });
});
```

#### Integration Tests

```javascript
const nock = require('nock');

describe('Installation Token Flow', () => {
  beforeEach(() => {
    nock('https://api.github.com')
      .post('/app/installations/12345/access_tokens')
      .reply(200, {
        token: 'ghs_test_token',
        expires_at: new Date(Date.now() + 3600000).toISOString(),
        permissions: { contents: 'read' }
      });
  });

  it('should obtain installation token', async () => {
    const jwt = generateJWT(appId, privateKeyPath);
    const tokenData = await getInstallationToken(12345, jwt);

    expect(tokenData.token).toBe('ghs_test_token');
    expect(tokenData.permissions.contents).toBe('read');
  });
});
```

#### End-to-End Tests

```javascript
describe('Webhook to API Flow', () => {
  it('should handle pull request webhook', async () => {
    const payload = {
      action: 'opened',
      installation: { id: 12345 },
      pull_request: {
        number: 42,
        head: { sha: 'abc123' }
      },
      repository: { full_name: 'owner/repo' }
    };

    // Mock GitHub API responses
    nock('https://api.github.com')
      .post('/app/installations/12345/access_tokens')
      .reply(200, { token: 'ghs_token' })
      .post('/repos/owner/repo/check-runs')
      .reply(201, { id: 1 });

    // Send webhook
    const response = await request(app)
      .post('/webhooks')
      .set('X-Hub-Signature-256', generateSignature(payload))
      .send(payload);

    expect(response.status).toBe(200);
  });
});
```

## Production Deployment

### Infrastructure Checklist

- [ ] HTTPS enabled for webhook endpoint
- [ ] Private key stored securely (secrets manager)
- [ ] Webhook secret stored securely
- [ ] Token caching implemented (Redis/DynamoDB)
- [ ] Rate limit monitoring
- [ ] Error tracking (Sentry, Rollbar)
- [ ] Logging infrastructure
- [ ] High availability (multiple instances)
- [ ] Auto-scaling configured
- [ ] Health check endpoints
- [ ] Graceful shutdown handling

### Monitoring

```javascript
// Key metrics to track
const metrics = {
  webhook_delivery_success_rate: gauge(),
  webhook_processing_duration: histogram(),
  jwt_generation_errors: counter(),
  token_refresh_count: counter(),
  api_call_success_rate: gauge(),
  rate_limit_remaining: gauge(),
  installation_count: gauge()
};

// Example: Track webhook processing
async function processWebhookWithMetrics(payload) {
  const start = Date.now();

  try {
    await processWebhook(payload);
    metrics.webhook_delivery_success_rate.set(1);
  } catch (error) {
    metrics.webhook_delivery_success_rate.set(0);
    throw error;
  } finally {
    const duration = Date.now() - start;
    metrics.webhook_processing_duration.observe(duration);
  }
}
```

### Alerting

Set up alerts for critical issues:

```yaml
alerts:
  - name: High Webhook Failure Rate
    condition: webhook_success_rate < 0.95
    severity: critical

  - name: JWT Generation Failing
    condition: jwt_generation_errors > 10
    severity: critical

  - name: Rate Limit Approaching
    condition: rate_limit_remaining < 500
    severity: warning

  - name: Token Cache Miss Rate High
    condition: token_cache_miss_rate > 0.5
    severity: warning
```

## Conclusion

GitHub Apps provide a robust, scalable, and secure way to integrate with GitHub. The authentication flow, while multi-step, ensures that:

1. **Your app is authenticated** via JWT signed with your private key
2. **Installation-specific access** is granted through installation tokens
3. **Webhooks are verified** using HMAC signatures
4. **Permissions are scoped** to exactly what users granted
5. **Rate limits scale** with the number of installations

By understanding each phase of the authentication flow, you can build reliable integrations that respect security best practices and provide value to users across thousands of installations.

### Key Takeaways

- **JWT is for app authentication** (max 10 minutes)
- **Installation tokens are for API calls** (1 hour lifetime)
- **Always verify webhook signatures** (HMAC-SHA256)
- **Cache tokens to reduce API calls** (but respect expiration)
- **Each installation gets independent rate limits** (5,000/hour)
- **Use least-privilege permissions** (request only what you need)
- **Implement proper error handling** (installations can be removed)
- **Monitor and alert on critical metrics** (webhook success, rate limits)

### Additional Resources

- [GitHub Apps Documentation](https://docs.github.com/en/apps)
- [REST API Reference](https://docs.github.com/en/rest)
- [Webhook Events](https://docs.github.com/en/webhooks/webhook-events-and-payloads)
- [Permissions Reference](https://docs.github.com/en/rest/overview/permissions-required-for-github-apps)
- [Rate Limiting](https://docs.github.com/en/rest/overview/rate-limits-for-the-rest-api)
