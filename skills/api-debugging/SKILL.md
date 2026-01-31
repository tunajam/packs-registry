# API Debugging

Procedural guide for debugging API issues, HTTP troubleshooting, and performance analysis.

## When to Use

- API returning unexpected status codes
- Requests timing out or failing
- Performance issues (slow responses)
- Authentication/authorization problems
- Data not appearing as expected
- Intermittent failures

## Debugging Workflow

### Step 1: Reproduce the Issue

```bash
# Capture the exact request
curl -v -X POST "https://api.example.com/users" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"email": "test@example.com"}'

# Save response for analysis
curl -s "https://api.example.com/users" | jq . > response.json
```

### Step 2: Check Status Code

| Code | Meaning | Common Causes |
|------|---------|---------------|
| 400 | Bad Request | Invalid JSON, missing fields, validation errors |
| 401 | Unauthorized | Missing/expired token, invalid credentials |
| 403 | Forbidden | Valid auth but insufficient permissions |
| 404 | Not Found | Wrong URL, resource doesn't exist |
| 405 | Method Not Allowed | Using GET instead of POST, etc. |
| 408 | Request Timeout | Server took too long |
| 413 | Payload Too Large | Request body exceeds limit |
| 422 | Unprocessable Entity | Valid JSON but failed validation |
| 429 | Too Many Requests | Rate limited |
| 500 | Internal Server Error | Server-side bug |
| 502 | Bad Gateway | Upstream server error |
| 503 | Service Unavailable | Server overloaded or maintenance |
| 504 | Gateway Timeout | Upstream timeout |

### Step 3: Examine Request Details

```bash
# Full request/response with headers
curl -v "https://api.example.com/users" 2>&1

# Check what you're actually sending
curl -v --trace-ascii /dev/stdout "https://api.example.com/users"

# Time breakdown
curl -w "@curl-format.txt" -o /dev/null -s "https://api.example.com/users"
```

**curl-format.txt:**
```
    time_namelookup:  %{time_namelookup}s\n
       time_connect:  %{time_connect}s\n
    time_appconnect:  %{time_appconnect}s\n
   time_pretransfer:  %{time_pretransfer}s\n
      time_redirect:  %{time_redirect}s\n
 time_starttransfer:  %{time_starttransfer}s\n
                    ----------\n
         time_total:  %{time_total}s\n
```

### Step 4: Check Headers

**Common header issues:**

```bash
# Missing Content-Type
curl -X POST "https://api.example.com/users" \
  -H "Content-Type: application/json" \  # Required!
  -d '{"email": "test@example.com"}'

# Wrong Accept header
curl "https://api.example.com/users" \
  -H "Accept: application/json"  # Server might default to XML

# Missing Authorization
curl "https://api.example.com/users" \
  -H "Authorization: Bearer $TOKEN"

# CORS preflight (browser)
curl -X OPTIONS "https://api.example.com/users" \
  -H "Origin: https://myapp.com" \
  -H "Access-Control-Request-Method: POST" \
  -v
```

## Common Issues & Fixes

### Authentication Failures (401)

**Check:**
1. Token format: `Bearer <token>` not `Bearer: <token>`
2. Token expiration: Decode JWT at jwt.io
3. Token scope: Does token have required permissions?
4. Header name: `Authorization` not `Authorisation`

```bash
# Decode JWT payload
echo "$TOKEN" | cut -d'.' -f2 | base64 -d 2>/dev/null | jq .

# Check expiration
echo "$TOKEN" | cut -d'.' -f2 | base64 -d 2>/dev/null | jq '.exp | todate'
```

### Validation Errors (400/422)

**Check:**
1. Required fields missing
2. Field types (string vs number)
3. Field formats (email, date, UUID)
4. Nested object structure

```bash
# Pretty print error response
curl -s "https://api.example.com/users" \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"invalid": "data"}' | jq .

# Validate JSON locally
echo '{"email": "test"}' | jq .
```

### CORS Issues (Browser Only)

**Symptoms:**
- Works in Postman/curl but not browser
- "No 'Access-Control-Allow-Origin' header"

**Check server response headers:**
```bash
curl -I "https://api.example.com/users" \
  -H "Origin: https://myapp.com"

# Look for:
# Access-Control-Allow-Origin: https://myapp.com
# Access-Control-Allow-Methods: GET, POST, OPTIONS
# Access-Control-Allow-Headers: Content-Type, Authorization
```

**Common fixes:**
- Add CORS headers to API responses
- Use same-origin requests (proxy through your backend)
- Check preflight (OPTIONS) handling

### Rate Limiting (429)

**Check headers:**
```bash
curl -I "https://api.example.com/users"

# Look for:
# X-RateLimit-Limit: 100
# X-RateLimit-Remaining: 0
# X-RateLimit-Reset: 1640000000
# Retry-After: 60
```

**Solutions:**
- Implement exponential backoff
- Cache responses where possible
- Request rate limit increase

### Timeout Issues (408/504)

**Diagnose:**
```bash
# Test with different timeouts
curl --connect-timeout 5 --max-time 30 "https://api.example.com/slow-endpoint"

# Check DNS resolution time
curl -w "DNS: %{time_namelookup}s\nConnect: %{time_connect}s\n" \
  -o /dev/null -s "https://api.example.com"
```

**Common causes:**
- Large payload processing
- Database queries
- External API calls
- Network issues

### SSL/TLS Issues

```bash
# Check certificate
curl -vI "https://api.example.com" 2>&1 | grep -A 10 "Server certificate"

# Ignore cert errors (debugging only!)
curl -k "https://api.example.com"

# Check with openssl
openssl s_client -connect api.example.com:443 -servername api.example.com
```

## Performance Analysis

### Measure Response Times

```bash
# Multiple requests with statistics
for i in {1..10}; do
  curl -o /dev/null -s -w "%{time_total}\n" "https://api.example.com/users"
done | awk '{sum+=$1} END {print "Avg:", sum/NR, "s"}'

# Using Apache Bench
ab -n 100 -c 10 "https://api.example.com/users"

# Using hey
hey -n 200 -c 50 "https://api.example.com/users"
```

### Identify Bottlenecks

```bash
curl -w "\
DNS Lookup:     %{time_namelookup}s\n\
TCP Connect:    %{time_connect}s\n\
TLS Handshake:  %{time_appconnect}s\n\
First Byte:     %{time_starttransfer}s\n\
Total:          %{time_total}s\n" \
  -o /dev/null -s "https://api.example.com/users"
```

**Interpretation:**
- High `time_namelookup`: DNS issues
- High `time_connect`: Network latency
- High `time_appconnect`: TLS negotiation
- High `time_starttransfer` - `time_appconnect`: Server processing

## API Testing Tools

### HTTPie (curl alternative)
```bash
# GET request
http GET https://api.example.com/users

# POST with JSON
http POST https://api.example.com/users email=test@example.com

# With auth
http GET https://api.example.com/users "Authorization: Bearer $TOKEN"
```

### jq (JSON processing)
```bash
# Pretty print
curl -s "https://api.example.com/users" | jq .

# Extract field
curl -s "https://api.example.com/users" | jq '.data[0].email'

# Filter results
curl -s "https://api.example.com/users" | jq '.data[] | select(.active == true)'
```

## Logging & Monitoring

### Enable verbose logging
```bash
# Most frameworks have debug modes
DEBUG=* node app.js
RUST_LOG=debug cargo run
```

### Check server logs
```bash
# Recent errors
tail -f /var/log/app/error.log

# Search for specific request
grep "request-id-123" /var/log/app/access.log

# Filter by status code
awk '$9 >= 500' /var/log/nginx/access.log
```

## Debugging Checklist

- [ ] Can I reproduce the issue consistently?
- [ ] What's the exact status code?
- [ ] Are request headers correct (Content-Type, Authorization)?
- [ ] Is the request body valid JSON/format?
- [ ] Does it work with curl but not the app?
- [ ] Does it work in staging but not production?
- [ ] Is it a timing/race condition?
- [ ] Are there any error details in the response?
- [ ] What do the server logs say?
- [ ] Is there a pattern (time of day, specific users)?

## Quick Reference

### HTTP Methods
| Method | Idempotent | Body | Use Case |
|--------|------------|------|----------|
| GET | Yes | No | Retrieve resource |
| POST | No | Yes | Create resource |
| PUT | Yes | Yes | Replace resource |
| PATCH | No | Yes | Partial update |
| DELETE | Yes | Optional | Delete resource |

### Content Types
- `application/json` - JSON data
- `application/x-www-form-urlencoded` - Form data
- `multipart/form-data` - File uploads
- `text/plain` - Plain text

## References

- [HTTP Status Codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
- [HTTP Headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers)
- [curl Manual](https://curl.se/docs/manual.html)
- [CORS Explained](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
