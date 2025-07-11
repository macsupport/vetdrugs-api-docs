# Error Handling

Understanding and handling VetDrugs API errors effectively.

## Error Response Format

All API errors follow a consistent JSON format:

```json
{
  "error": "error_code",
  "message": "Human-readable error description",
  "details": "Additional error context (when available)"
}
```

## HTTP Status Codes

| Status Code | Meaning | Description |
|-------------|---------|-------------|
| `200` | Success | Request completed successfully |
| `400` | Bad Request | Invalid request format or parameters |
| `401` | Unauthorized | Invalid or missing API key |
| `403` | Forbidden | API key valid but lacks permission |
| `404` | Not Found | Endpoint or resource not found |
| `429` | Too Many Requests | Rate limit exceeded |
| `500` | Internal Server Error | Server-side error |
| `503` | Service Unavailable | Temporary service disruption |

## Common Error Codes

### Authentication Errors (401)

#### Invalid API Key
```json
{
  "error": "invalid_api_key",
  "message": "API key is required or invalid"
}
```

#### Missing API Key
```json
{
  "error": "missing_api_key", 
  "message": "Include API key in Authorization header as 'Bearer vd_xxx' or X-API-Key header"
}
```

### Validation Errors (400)

#### Missing Required Fields
```json
{
  "error": "validation_failed",
  "messages": [
    "Valid patient weight is required (use 'weight_kg' or 'weight' field)",
    "Patient species must be 'dog' or 'cat'"
  ]
}
```

#### Invalid Patient Data
```json
{
  "error": "validation_failed",
  "messages": [
    "Patient weight seems unrealistic (0.1kg - 1000kg range)",
    "At least one drug must be specified"
  ]
}
```

### Rate Limiting Errors (429)

```json
{
  "error": "rate_limit_exceeded",
  "message": "Rate limit of 1000 requests per minute exceeded",
  "limit": 1000,
  "remaining": 0,
  "resetTime": "2025-07-11T19:00:00Z"
}
```

### Service Errors (503)

#### Database Unavailable
```json
{
  "error": "service_unavailable",
  "message": "Drug data is unavailable from KV store",
  "details": "Temporary service disruption"
}
```

## Calculation Warnings

Some responses include warnings rather than errors:

```json
{
  "patient": { ... },
  "calculations": [ ... ],
  "warnings": [
    "The following drugs were not found: InvalidDrugName"
  ],
  "metadata": {
    "drugs_not_found": ["InvalidDrugName"]
  }
}
```

## Error Handling Best Practices

### 1. Check HTTP Status Code First

```javascript
const response = await fetch('/api/calculate', { ... });

if (!response.ok) {
  const error = await response.json();
  console.error(`API Error (${response.status}):`, error.message);
  return;
}

const data = await response.json();
```

### 2. Handle Specific Error Types

```javascript
const handleApiError = (error, statusCode) => {
  switch (statusCode) {
    case 401:
      // Redirect to API key setup
      showApiKeyError();
      break;
    case 429:
      // Implement exponential backoff
      scheduleRetry(error.resetTime);
      break;
    case 503:
      // Show service unavailable message
      showMaintenanceMessage();
      break;
    default:
      // Log unexpected errors
      console.error('Unexpected API error:', error);
  }
};
```

### 3. Implement Retry Logic

```javascript
const makeApiRequest = async (url, options, maxRetries = 3) => {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url, options);
      
      if (response.ok) {
        return await response.json();
      }
      
      if (response.status === 429) {
        // Rate limited - wait before retry
        const retryAfter = response.headers.get('Retry-After') || 60;
        await new Promise(resolve => setTimeout(resolve, retryAfter * 1000));
        continue;
      }
      
      if (response.status >= 500 && attempt < maxRetries) {
        // Server error - exponential backoff
        const delay = Math.pow(2, attempt) * 1000;
        await new Promise(resolve => setTimeout(resolve, delay));
        continue;
      }
      
      // Don't retry client errors (4xx)
      throw new Error(`API Error: ${response.status}`);
      
    } catch (error) {
      if (attempt === maxRetries) throw error;
    }
  }
};
```

### 4. Validate Input Before Sending

```javascript
const validateRequest = (patient, drugs) => {
  const errors = [];
  
  if (!patient.weight_kg || patient.weight_kg <= 0) {
    errors.push('Valid patient weight required');
  }
  
  if (!['dog', 'cat'].includes(patient.species)) {
    errors.push('Species must be dog or cat');
  }
  
  if (!drugs || drugs.length === 0) {
    errors.push('At least one drug required');
  }
  
  return errors;
};
```

## Debugging Tips

### 1. Check Response Headers

Rate limiting information is in response headers:
```javascript
console.log('Rate limit:', response.headers.get('X-RateLimit-Limit'));
console.log('Remaining:', response.headers.get('X-RateLimit-Remaining'));
console.log('Reset time:', response.headers.get('X-RateLimit-Reset'));
```

### 2. Log Request Details

```javascript
console.log('Request URL:', url);
console.log('Request headers:', headers);
console.log('Request body:', JSON.stringify(body, null, 2));
```

### 3. Enable Detailed Error Logging

```javascript
fetch('/api/calculate', options)
  .then(response => {
    console.log('Response status:', response.status);
    console.log('Response headers:', [...response.headers.entries()]);
    return response.json();
  })
  .then(data => console.log('Response data:', data))
  .catch(error => console.error('Request failed:', error));
```

## Support

If you encounter persistent errors:

1. **Check Status Page**: [status.vetdrugscalculators.com](https://status.vetdrugscalculators.com)
2. **Review Documentation**: Ensure correct API usage
3. **Contact Support**: support@vetdrugscalculators.com with:
   - Request details (URL, headers, body)
   - Response received
   - Timestamp of the error
   - Your API key (last 4 characters only)