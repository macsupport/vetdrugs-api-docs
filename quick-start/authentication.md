# Authentication

The VetDrugs API uses API keys for authentication. All requests must include a valid API key.

## Getting Your API Key

Contact support@vetdrugscalculators.com to request your API key. Include:
- Practice name
- Contact information
- Intended use case

## Using Your API Key

Include your API key in the request headers:

### Option 1: X-API-Key Header (Recommended)

```bash
curl -H "X-API-Key: vd_your_api_key_here" \
     "https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/calculate"
```

### Option 2: Authorization Bearer Token

```bash
curl -H "Authorization: Bearer vd_your_api_key_here" \
     "https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/calculate"
```

## API Key Format

API keys follow this format:
```
vd_[32_character_random_string]
```

Example: `vd_1a2b3c4d5e6f7g8h9i0j1k2l3m4n5o6p`

## Security Best Practices

- ✅ **Never expose API keys in client-side code**
- ✅ **Store API keys securely in environment variables**
- ✅ **Use HTTPS for all requests**
- ✅ **Rotate keys periodically**
- ✅ **Monitor API usage for anomalies**

## Rate Limits

Each API key has usage limits:
- **Trial**: 100 requests/hour
- **Professional**: 1,000 requests/hour
- **Enterprise**: 10,000 requests/hour

## Authentication Errors

```json
{
  "error": "invalid_api_key",
  "message": "API key is required or invalid"
}
```

## Testing Authentication

```bash
# Test your API key
curl -H "X-API-Key: your_api_key" \
     "https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/health"
```

Valid response:
```json
{
  "status": "healthy",
  "authenticated": true,
  "api_key_valid": true
}
```