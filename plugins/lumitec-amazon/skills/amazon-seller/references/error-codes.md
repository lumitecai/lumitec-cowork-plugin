# SP-API Error Handling

## HTTP Status Codes

### 400 Bad Request
- **Always surface** `errors[].message` from the response body — don't swallow it
- Common causes: missing required params, invalid date format, malformed JSON
- Fix: check param names (SP-API uses PascalCase for some query params)

### 401 Unauthorized
- Access token expired or invalid
- Action: `getAccessToken` auto-refreshes. Retry the call.
- If persistent: the user's refresh token is likely expired or revoked — re-run the `setup-amazon` skill to re-provision

### 403 Forbidden
- **NOT a credentials problem** — usually missing role/scope on the LWA app
- Action: check app registration in Seller Central Developer Console
- Verify the app has the required SP-API roles for the endpoint
- Common trap: new endpoints need new role grants

### 404 Not Found
- Invalid ASIN, SKU, orderId, or reportId
- Action: verify the identifier exists, try a search instead
- For reports: the report may have been garbage-collected (>30 days old)

### 429 Too Many Requests
- Token bucket is empty for this endpoint
- Read `x-amzn-RateLimit-Limit` header for current rate
- Action: exponential backoff with jitter, don't just wait-and-retry
- See `rate-limits.md` for per-endpoint rates

### 500 Internal Server Error
- Amazon transient error — **common**, especially during peak hours
- Action: retry with exponential backoff (start at 1s, cap at 30s)
- If persistent (>5 retries): likely an Amazon-side issue, try later

### 503 Service Unavailable
- Amazon service overloaded
- Same handling as 500: retry with backoff

## SP-API Error Response Format

```json
{
  "errors": [
    {
      "code": "InvalidInput",
      "message": "The marketplace ID is invalid",
      "details": "..."
    }
  ]
}
```

Always extract and display `errors[0].message` — it's usually descriptive enough for the user.

## Common Error Patterns

| Error message | Root cause | Fix |
|---|---|---|
| "Access to requested resource is denied" | LWA app missing role | Add role in Developer Console |
| "The marketplace ID is invalid" | Wrong marketplace for region | Check `marketplace-ids.md` |
| "Request is throttled" | Rate limit hit | Back off, check `rate-limits.md` |
| "Invalid value for parameter" | Wrong param format | Check API docs for expected format |
| "Report not found" | Report expired or wrong ID | Re-create the report |
