# Support

Get help with the VetDrugs API integration and troubleshooting.

## Getting Help

### 1. Documentation First

Before reaching out for support, please check:

- **[Quick Start Guide](quick-start/)** - Basic setup and first request
- **[API Reference](api-reference/)** - Complete endpoint documentation  
- **[Integration Guides](integrations/)** - PMI-specific implementations
- **[Code Examples](examples/)** - Language-specific examples
- **[Error Handling](api-reference/errors.md)** - Common errors and solutions

### 2. Community Support

**GitHub Discussions** (Coming Soon)
- Ask questions and share solutions
- Community-driven support
- Feature requests and feedback

**Stack Overflow**
- Tag your questions with `vetdrugs-api`
- Search existing questions first

### 3. Direct Support

**Email Support**: [support@vetdrugscalculators.com](mailto:support@vetdrugscalculators.com)

**Response Times**:
- Standard Support: 24-48 hours
- Priority Support: 4-8 hours (Enterprise customers)
- Critical Issues: 2-4 hours (Enterprise customers)

**Phone Support**: Available for Enterprise customers
- Schedule a call: [calendly.com/vetdrugs-support](https://calendly.com/vetdrugs-support)

## What to Include When Contacting Support

### For API Issues

**Required Information**:
1. **API Key** (last 4 characters only): `...abc123`
2. **Request Details**:
   - HTTP method and endpoint
   - Request headers (excluding API key)
   - Request body (if applicable)
3. **Response Received**:
   - HTTP status code
   - Response headers
   - Response body
4. **Expected vs Actual Behavior**
5. **Timestamp** of the issue (including timezone)

**Example Support Request**:
```
Subject: Drug calculation returning unexpected results

API Key: ...abc123
Endpoint: POST /api/calculate
Timestamp: 2025-07-11 14:30:00 PST

Request:
{
  "patient": {"weight_kg": 25, "species": "dog"},
  "drugs": [{"drug_name": "Cephalexin"}]
}

Response (Status 200):
{
  "calculations": [...]
}

Issue: The calculated volume seems incorrect for a 25kg dog.
Expected: ~2ml, Received: 8ml

Additional context: This worked correctly yesterday.
```

### For Integration Issues

**Required Information**:
1. **Platform/Language**: (e.g., "Cornerstone with C#", "WordPress with PHP")
2. **Code Sample**: Minimal reproducible example
3. **Error Messages**: Complete error text and stack traces
4. **Environment**: Development/staging/production
5. **Version Information**: Framework/library versions

### For Account Issues

**Required Information**:
1. **Account Email**: The email associated with your API key
2. **Issue Type**: Billing, access, rate limits, etc.
3. **Urgency Level**: How critical is this to your operations?

## Common Issues and Solutions

### 1. Authentication Problems

#### "Invalid API Key" Error

**Symptoms**:
```json
{
  "error": "invalid_api_key",
  "message": "API key is required or invalid"
}
```

**Solutions**:
- ✅ Verify API key is correct (copy from dashboard)
- ✅ Check header format: `X-API-Key: vd_your_api_key`
- ✅ Ensure no extra spaces or characters
- ✅ Confirm API key is active (not revoked)

#### "Missing API Key" Error

**Symptoms**:
```json
{
  "error": "missing_api_key",
  "message": "Include API key in X-API-Key header"
}
```

**Solutions**:
- ✅ Add `X-API-Key` header to all requests
- ✅ Don't use Authorization header (use X-API-Key instead)
- ✅ Check your HTTP client configuration

### 2. Calculation Issues

#### "Drug not found" in Warnings

**Symptoms**:
```json
{
  "warnings": ["The following drugs were not found: DrugName"],
  "metadata": {
    "drugs_not_found": ["DrugName"]
  }
}
```

**Solutions**:
- ✅ Check drug spelling (case-insensitive but spelling matters)
- ✅ Search for the drug: `GET /api/drugs?search=drugname`
- ✅ Use exact names from search results
- ✅ Try brand names or generic names

#### Unexpected Dose Calculations

**Symptoms**: Doses seem too high or too low

**Solutions**:
- ✅ Verify patient weight is in kilograms
- ✅ Check species is correct ("dog" or "cat")
- ✅ Review drug concentration if specified
- ✅ Compare with manual calculations
- ✅ Check for species-specific dosing differences

### 3. Rate Limiting Issues

#### "Rate limit exceeded" Error

**Symptoms**:
```json
{
  "error": "rate_limit_exceeded",
  "message": "Rate limit of 100 requests per minute exceeded"
}
```

**Solutions**:
- ✅ Implement exponential backoff retry logic
- ✅ Add delays between batch requests
- ✅ Cache frequently accessed data
- ✅ Consider upgrading to higher rate limits
- ✅ Use the `Retry-After` header value

### 4. Integration Issues

#### Request Timeout

**Symptoms**: Requests taking longer than 30 seconds

**Solutions**:
- ✅ Check network connectivity
- ✅ Verify API endpoint URL
- ✅ Implement retry logic for transient failures
- ✅ Consider reducing request complexity

#### SSL/TLS Certificate Issues

**Symptoms**: SSL handshake failures

**Solutions**:
- ✅ Ensure your client supports TLS 1.2+
- ✅ Update certificate store if needed
- ✅ Check firewall/proxy settings
- ✅ Test with curl: `curl -I https://vetdrugs-calculator-api.vetcalculators.workers.dev`

## API Status and Monitoring

### Service Status

**Status Page**: [status.vetdrugscalculators.com](https://status.vetdrugscalculators.com)

**Real-time Status**:
- API availability
- Response time metrics
- Incident reports
- Scheduled maintenance

**Health Check Endpoint**:
```bash
curl https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/health
```

Expected Response:
```json
{
  "status": "healthy",
  "version": "2.2.0-enhanced",
  "uptime": 1234567,
  "timestamp": "2025-07-11T18:30:00Z"
}
```

### Performance Metrics

**Typical Response Times**:
- Drug calculations: 50-150ms
- Drug search: 30-100ms
- Drug information: 40-120ms
- Global 99th percentile: <300ms

**Availability SLA**:
- Uptime: 99.9% (excluding scheduled maintenance)
- Scheduled maintenance: <4 hours/month
- Advance notice: 48+ hours for planned maintenance

## Troubleshooting Tools

### 1. API Test Tool

Test your API integration:

```bash
# Test authentication
curl -H "X-API-Key: vd_your_api_key" \
     https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/health

# Test calculation
curl -H "X-API-Key: vd_your_api_key" \
     -H "Content-Type: application/json" \
     -d '{"patient":{"weight_kg":25,"species":"dog"},"drugs":[{"drug_name":"Cephalexin"}]}' \
     https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/calculate
```

### 2. Debug Headers

Add debug information to your requests:

```bash
curl -H "X-API-Key: vd_your_api_key" \
     -H "X-Debug-Mode: true" \
     -H "X-Request-ID: your-unique-id" \
     https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/calculate
```

Debug headers help our support team trace your specific requests.

### 3. Validation Tool

Use our request validation:

```javascript
function validateRequest(patient, drugs) {
  const errors = [];
  
  // Validate patient
  if (!patient.weight_kg || patient.weight_kg <= 0) {
    errors.push('Valid patient weight required');
  }
  
  if (!['dog', 'cat'].includes(patient.species?.toLowerCase())) {
    errors.push('Species must be dog or cat');
  }
  
  // Validate drugs
  if (!drugs || drugs.length === 0) {
    errors.push('At least one drug required');
  }
  
  drugs.forEach((drug, index) => {
    if (!drug.drug_name) {
      errors.push(`Drug ${index + 1}: name required`);
    }
  });
  
  return errors;
}
```

### 4. Network Diagnostics

Check connectivity to our API:

```bash
# Test DNS resolution
nslookup vetdrugs-calculator-api.vetcalculators.workers.dev

# Test connectivity
ping vetdrugs-calculator-api.vetcalculators.workers.dev

# Test HTTPS
openssl s_client -connect vetdrugs-calculator-api.vetcalculators.workers.dev:443

# Test with verbose output
curl -v https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/health
```

## Feature Requests

### How to Submit

1. **Check existing requests** in our GitHub discussions
2. **Provide detailed description**:
   - Use case and business justification
   - Proposed solution or approach
   - Priority level for your organization
3. **Include implementation details** if you have preferences

### Current Roadmap

**Q3 2025**:
- Additional drug categories (exotic animals)
- Advanced drug interaction checking
- Bulk calculation endpoints
- WebSocket real-time updates

**Q4 2025**:
- Mobile SDK (iOS/Android)
- Advanced analytics dashboard
- Custom dosing protocols
- Multi-language support

**2026**:
- AI-powered dosing recommendations
- Integration with more PMI systems
- Advanced reporting features

## Enterprise Support

### Premium Support Features

**Dedicated Support**:
- Named technical account manager
- Priority email and phone support
- Custom integration assistance
- Monthly check-in calls

**Enhanced SLA**:
- 99.95% uptime guarantee
- 2-hour response time for critical issues
- 24/7 emergency support line
- Advanced monitoring and alerting

**Development Support**:
- Code review assistance
- Architecture consultation
- Custom feature development
- Priority feature requests

### Contact Enterprise Sales

**Email**: [enterprise@vetdrugscalculators.com](mailto:enterprise@vetdrugscalculators.com)
**Phone**: 1-800-VET-DRUG (1-800-838-3784)
**Schedule Demo**: [calendly.com/vetdrugs-demo](https://calendly.com/vetdrugs-demo)

## Contributing

### Bug Reports

Help us improve by reporting bugs:

1. **Search existing issues** first
2. **Use our bug report template**
3. **Include minimal reproduction case**
4. **Specify environment details**

### Documentation Improvements

Found unclear documentation?

1. **Open an issue** describing the problem
2. **Suggest specific improvements**
3. **Submit pull requests** for fixes
4. **Help translate** to other languages

### API Enhancement Ideas

Share your ideas for API improvements:

1. **New endpoints or features**
2. **Performance optimizations**
3. **Developer experience improvements**
4. **Integration suggestions**

## Security

### Reporting Security Issues

**DO NOT** report security vulnerabilities through public GitHub issues.

**Security Email**: [security@vetdrugscalculators.com](mailto:security@vetdrugscalculators.com)

**Include**:
- Description of the vulnerability
- Steps to reproduce
- Potential impact assessment
- Suggested fix (if any)

**Response Process**:
1. **Acknowledgment**: Within 24 hours
2. **Initial Assessment**: Within 72 hours  
3. **Fix Timeline**: Based on severity
4. **Public Disclosure**: After fix deployment

### Security Best Practices

**API Key Security**:
- ✅ Never expose API keys in client-side code
- ✅ Use environment variables for API keys
- ✅ Rotate API keys regularly
- ✅ Use separate keys for dev/staging/production
- ✅ Revoke unused API keys immediately

**Network Security**:
- ✅ Always use HTTPS endpoints
- ✅ Validate SSL certificates
- ✅ Implement request signing (Enterprise)
- ✅ Use IP whitelisting when possible
- ✅ Monitor for unusual API usage patterns

## Contact Information

### Support Channels

| Channel | Best For | Response Time |
|---------|----------|---------------|
| **Email** | Technical issues, bug reports | 24-48 hours |
| **GitHub** | Feature requests, documentation | Community-driven |
| **Phone** | Urgent issues (Enterprise) | During business hours |
| **Chat** | Quick questions (Enterprise) | Business hours |

### Business Hours

**Support Hours**: Monday-Friday, 8 AM - 6 PM PST
**Emergency Support**: 24/7 (Enterprise customers only)
**Holidays**: US federal holidays (reduced support)

### Regional Support

**North America**: Primary support region
**Europe**: Extended hours support (Enterprise)
**Asia-Pacific**: Email support with 12-hour SLA

### Mailing Address

VetDrugs Calculator API  
123 Innovation Drive  
San Francisco, CA 94105  
United States

---

**Need immediate help?** Email [support@vetdrugscalculators.com](mailto:support@vetdrugscalculators.com) with "URGENT" in the subject line for priority handling.
