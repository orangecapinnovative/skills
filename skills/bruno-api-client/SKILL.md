---
name: bruno-api-client
description: "Expert in Bruno API client — writing .bru files, managing collections, scripting with the bru/req/res API, and building Git-first API test workflows."
risk: safe
source: community
date_added: '2026-05-12'
---

## Use this skill when

- Creating or editing `.bru` request files
- Setting up Bruno collections, environments, or folder structures
- Writing pre-request / post-response scripts using the `bru`, `req`, or `res` APIs
- Writing tests with the Bruno/Chai.js test framework
- Chaining requests, managing variables, or handling cookies in Bruno
- Migrating API collections from Postman or Insomnia to Bruno
- Configuring `bruno.json` or environment files
- Running Bruno collections in CI/CD pipelines via the CLI

## Do not use this skill when

- Working with Postman, Insomnia, or other API clients (unless migrating from them)
- The task is general HTTP/API design unrelated to Bruno tooling

## Instructions

- Always use `.bru` plain-text format — never suggest JSON for request definitions.
- Embrace the file-based, one-request-per-file approach.
- Use `{{variableName}}` syntax for environment and runtime variables.
- Place secrets in `vars:secret` blocks so they are never committed.
- Write `tests` blocks for every request: status code, structure, and data validation.
- Use `script:pre-request` for setup/validation and `script:post-response` for chaining.
- Structure collections to be Git-friendly — meaningful folder names, descriptive file names.
- Validate work: confirm the `.bru` syntax is correct and variables are referenced consistently.

---

## Core Philosophy

| Principle | Meaning |
|---|---|
| Offline-First | No cloud sync; everything stored locally |
| Git-Collaborative | Collections version-controlled alongside code |
| Plain Text | Human-readable `.bru` files |
| File-Based | No databases — pure filesystem |

---

## Bru File Format

The fundamental building block. Every request is its own `.bru` file.

```bru
meta {
  name: Create User
  type: http
  seq: 1
  tags: [user-management, post-request]
}

post {
  url: {{baseUrl}}/api/users
  body: json
  auth: bearer
}

headers {
  content-type: application/json
  accept: application/json
  x-api-version: {{apiVersion}}
}

auth:bearer {
  token: {{authToken}}
}

body:json {
  {
    "name": "{{userName}}",
    "email": "{{userEmail}}",
    "role": "user"
  }
}

script:pre-request {
  const timestamp = Date.now();
  bru.setVar("requestId", `req_${timestamp}`);

  if (!bru.getVar("userName")) {
    throw new Error("userName is required");
  }
}

script:post-response {
  if (res.status === 201) {
    bru.setVar("newUserId", res.body.id);
    bru.setVar("userCreated", true);
  }
}

tests {
  test("User created successfully", function() {
    expect(res.status).to.equal(201);
    expect(res.body).to.have.property("id");
    expect(res.body.name).to.equal(bru.getVar("userName"));
  });

  test("Response time is acceptable", function() {
    expect(res.responseTime).to.be.below(2000);
  });
}

vars:pre-request {
  userName: John Doe
  userEmail: john@example.com
}

vars:post-response {
  userId: {{res.body.id}}
  createdAt: {{res.body.created_at}}
}
```

---

## Collection File Structure

```
my-api-collection/
├── bruno.json              # Collection metadata & proxy config
├── collection.bru          # Collection-level scripts/auth
├── environments/
│   ├── dev.bru
│   ├── staging.bru
│   └── prod.bru
├── users/
│   ├── folder.bru          # Folder-level auth/scripts
│   ├── get-user.bru
│   ├── create-user.bru
│   └── update-user.bru
└── auth/
    ├── login.bru
    └── refresh-token.bru
```

### Environment file (`environments/dev.bru`)

```bru
vars {
  baseUrl: https://api.example.com
  apiVersion: v1
  timeout: 5000
}

vars:secret [
  apiKey,
  clientSecret,
  refreshToken
]
```

### Collection config (`bruno.json`)

```json
{
  "version": "1",
  "name": "My API Collection",
  "type": "collection",
  "scripts": {
    "moduleWhitelist": ["crypto", "buffer", "form-data"],
    "filesystemAccess": { "allow": true }
  }
}
```

---

## JavaScript API Reference

### `req` — Request Object (pre-request scripts)

| Method | Description |
|---|---|
| `req.getUrl()` / `req.setUrl(url)` | Read or override the request URL |
| `req.getMethod()` / `req.setMethod(m)` | Read or override HTTP method |
| `req.getHeader(name)` / `req.setHeader(name, val)` | Single header access |
| `req.getHeaders()` / `req.setHeaders(obj)` | Bulk header access |
| `req.getBody()` / `req.setBody(body)` | Read or override body |
| `req.setTimeout(ms)` | Set timeout |
| `req.getName()` | Get request name |
| `req.getTags()` | Get tags array |

### `res` — Response Object (post-response scripts & tests)

| Property/Method | Description |
|---|---|
| `res.status` | HTTP status code |
| `res.statusText` | Status text |
| `res.headers` | Headers object |
| `res.body` | Parsed body (JSON auto-parsed) |
| `res.responseTime` | Response time in ms |
| `res.getHeader(name)` | Single header |

### `bru` — Runtime Object

**Variables**
```javascript
bru.setVar("key", value)       // runtime variable (cross-request)
bru.getVar("key")
bru.setEnvVar("key", value)    // environment variable
bru.getEnvVar("key")
bru.getProcessEnv("KEY")       // system env var
```

**Flow control**
```javascript
bru.setNextRequest("Request Name")  // chain to another request
bru.sleep(ms)                       // pause execution
bru.runner.skipRequest()            // skip current request in runner
```

**Utilities**
```javascript
bru.cwd()                           // working directory
bru.interpolate("{{$randomEmail}}") // interpolate dynamic variables
bru.disableParsingResponseJson()    // disable auto JSON parse (call in pre-request)
bru.getTestResults()                // get test results object
```

---

## Cookie Management

```javascript
const jar = bru.cookies.jar();

// Set cookie
jar.setCookie("https://api.example.com", "sessionId", "abc123");

// Set cookie with options
jar.setCookie("https://api.example.com", {
  key: "authToken",
  value: "xyz789",
  domain: "example.com",
  path: "/api",
  secure: true,
  httpOnly: true,
  maxAge: 3600
});

const cookie = await jar.getCookie("https://api.example.com", "sessionId");
const all    = await jar.getCookies("https://api.example.com");
jar.deleteCookie("https://api.example.com", "sessionId");
jar.deleteCookies("https://api.example.com");
jar.clear();
```

---

## Dynamic Variables

Use directly in `.bru` files or interpolate in scripts.

| Variable | Output |
|---|---|
| `{{$guid}}` / `{{$randomUUID}}` | Random UUID |
| `{{$randomEmail}}` | Random email |
| `{{$randomFirstName}}` / `{{$randomLastName}}` / `{{$randomFullName}}` | Names |
| `{{$randomPhoneNumber}}` | Phone number |
| `{{$randomCity}}` / `{{$randomCountry}}` / `{{$randomStreetAddress}}` | Location |
| `{{$randomInt}}` | Random integer |
| `{{$timestamp}}` | Unix timestamp |
| `{{$isoTimestamp}}` | ISO 8601 timestamp |
| `{{$randomJobTitle}}` / `{{$randomCompanyName}}` | Job / company |

```javascript
// In script
const email = bru.interpolate('{{$randomEmail}}');
bru.setVar("userEmail", email);
```

---

## Authentication Types

```bru
auth:bearer  { token: {{token}} }
auth:basic   { username: user, password: {{pass}} }
auth:apikey  { key: x-api-key, value: {{apiKey}} }
```

Also supported: OAuth2 (Authorization Code, Client Credentials, Password), AWS Signature, Digest, NTLM.

---

## Testing Framework (Chai.js)

```javascript
tests {
  test("Status is 200", function() {
    expect(res.status).to.equal(200);
  });

  test("Response structure", function() {
    expect(res.body).to.be.an("object");
    expect(res.body).to.have.property("data");
    expect(res.body.data).to.be.an("array");
  });

  test("Response time", function() {
    expect(res.responseTime).to.be.below(2000);
  });

  test("Content-Type header", function() {
    expect(res.headers["content-type"]).to.include("application/json");
  });

  test("Email format", function() {
    expect(res.body.user.email).to.match(/^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$/);
    expect(res.body.user.age).to.be.at.least(18);
  });
}
```

---

## Common Workflow Patterns

### Request Chaining

```javascript
// post-response of login.bru
if (res.status === 200) {
  bru.setVar("authToken", res.body.token);
  bru.setNextRequest("Get User Profile");
}
```

### Environment-Based Skip

```javascript
// pre-request
const env = bru.getEnvVar("environment");
if (env === "production") {
  bru.runner.skipRequest();
}
```

### Data-Driven Testing

```javascript
// script:pre-request
const testUser = {
  email: bru.interpolate('{{$randomEmail}}'),
  name:  bru.interpolate('{{$randomFullName}}'),
  phone: bru.interpolate('{{$randomPhoneNumber}}')
};
req.setBody(JSON.stringify(testUser));
```

### Tag-Based Conditional Execution

```javascript
const tags = req.getTags();
if (!tags.includes("integration-test")) {
  bru.runner.skipRequest();
}
```

---

## Variable Scoping (priority order)

1. **Request variables** — `vars:pre-request` / `vars:post-response` blocks
2. **Folder variables** — `folder.bru`
3. **Collection variables** — `collection.bru`
4. **Environment variables** — active environment file
5. **Runtime variables** — `bru.setVar()` / `bru.getVar()`
6. **Secret variables** — `vars:secret` block (never committed)
7. **Process variables** — `bru.getProcessEnv()`

---

## Best Practices

1. One request per `.bru` file; use descriptive names (`create-user.bru`, not `request1.bru`)
2. All environment-specific values go in `{{variables}}` — never hardcode URLs or tokens
3. Secrets in `vars:secret` blocks only
4. Every request should have a `tests` block covering status, structure, and timing
5. Use `script:pre-request` to validate required variables before sending
6. Use `bru.setNextRequest()` for dependent request chains
7. Organize by domain (auth/, users/, orders/) not by HTTP method
8. Commit `bruno.json` and `.bru` files; never commit `.env` or secret values
9. Test against all environments: dev/staging/prod environment files
10. Use dynamic variables (`{{$randomEmail}}`) for realistic, non-colliding test data

---

## Supported Request Types

- **HTTP/REST** — GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS
- **GraphQL** — queries, mutations, subscriptions with variables
- **gRPC** — protocol buffer requests with streaming
- **WebSocket** — real-time bidirectional communication
- **SOAP** — XML-based web services

---

## Example Interactions

- "Create a Bruno collection for a REST user management API with dev and prod environments"
- "Write pre-request validation that checks required variables before sending"
- "Chain a login request to a profile fetch using `bru.setNextRequest()`"
- "Add comprehensive tests to this `.bru` file"
- "Convert this Postman collection to Bruno format"
- "Set up cookie-based session management across multiple requests"
- "Generate random test data using Bruno dynamic variables"
- "Configure OAuth2 bearer auth at the collection level"

## Limitations

- Use this skill only for Bruno-specific work described above.
- Do not treat generated `.bru` files as production-ready without running them against your actual API.
- Stop and ask for clarification if the target API, environment structure, or authentication method is ambiguous.
