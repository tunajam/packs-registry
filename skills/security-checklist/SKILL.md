# Skill: Security Checklist

Check for common vulnerabilities before shipping. This isn't exhaustive security review—it's the minimum bar.

## When to Use
- Before deploying any user-facing feature
- When handling user input
- When dealing with authentication/authorization
- When adding API endpoints
- When reviewing PRs

## The Checklist

### 🔐 Authentication

- [ ] **Passwords hashed** — Using bcrypt/argon2, not MD5/SHA1
- [ ] **No credentials in code** — Secrets in env vars, not committed
- [ ] **Session tokens are secure** — HttpOnly, Secure, SameSite flags
- [ ] **Logout actually works** — Token invalidated server-side
- [ ] **Password reset is safe** — Tokens expire, single-use, unpredictable

### 🚪 Authorization

- [ ] **Every endpoint checks permissions** — Not just the frontend
- [ ] **Can't access other users' data** — `user_id` comes from session, not request
- [ ] **Role checks on sensitive actions** — Admin functions actually check admin
- [ ] **No IDOR vulnerabilities** — Can't just change `?id=123` to see someone else's data

### 📥 Input Validation

- [ ] **All input validated server-side** — Never trust the client
- [ ] **SQL injection prevented** — Parameterized queries, not string concat
- [ ] **XSS prevented** — Output encoded, dangerous HTML escaped
- [ ] **File uploads validated** — Type checked, size limited, not executable
- [ ] **No path traversal** — Can't use `../` to access other files

### 🌐 API Security

- [ ] **Rate limiting** — Can't spam endpoints
- [ ] **CORS configured correctly** — Not `*` in production
- [ ] **No sensitive data in URLs** — Tokens/passwords in headers/body
- [ ] **Errors don't leak info** — No stack traces, DB errors to users
- [ ] **HTTPS everywhere** — No mixed content

### 💾 Data Protection

- [ ] **Sensitive data encrypted at rest** — PII, financial data
- [ ] **Backups encrypted** — And tested for restore
- [ ] **Logs don't contain secrets** — No passwords, tokens, PII in logs
- [ ] **Data deleted when requested** — GDPR compliance

### 🔑 Secrets Management

- [ ] **No hardcoded secrets** — Not in code, config files, or Docker images
- [ ] **API keys rotatable** — Can change without deployment
- [ ] **Least privilege** — Keys only have permissions they need
- [ ] **Secrets not in git history** — Check old commits too

## Quick Checks by Feature Type

### New API Endpoint
```
□ Auth required?
□ Permission check?
□ Input validated?
□ Rate limited?
□ Errors safe?
```

### User Input Form
```
□ Server validation?
□ XSS escaped?
□ CSRF protected?
□ Size limits?
```

### File Upload
```
□ Type whitelist?
□ Size limit?
□ Filename sanitized?
□ Stored outside webroot?
□ Virus scan?
```

### Payment/Financial
```
□ Use Stripe/established provider?
□ No card numbers in your DB?
□ Webhook signatures verified?
□ Amounts validated server-side?
```

## Red Flags in Code Review

```javascript
// 🚨 SQL injection
`SELECT * FROM users WHERE id = ${userId}`

// 🚨 XSS
element.innerHTML = userInput

// 🚨 Hardcoded secret
const API_KEY = "sk_live_abc123"

// 🚨 Missing auth check
app.get('/admin/users', (req, res) => { ... })

// 🚨 Trusting client-side ID
const userId = req.body.userId  // Should be req.session.userId

// 🚨 Path traversal
const file = `./uploads/${req.params.filename}`
```

## The One Question

Before shipping, ask:

> "If a malicious user tried to break this, what would they try?"

Then verify you've blocked those vectors.

## Output

Run through relevant sections of this checklist. Document any risks accepted and why.
