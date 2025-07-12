# API Reference

Complete reference documentation for all VetDrugs API endpoints.

## Overview

The VetDrugs API provides comprehensive veterinary drug dosage calculations through a RESTful interface. All endpoints return JSON responses and require API key authentication.

**Base URL**: `https://vetdrugs-calculator-api.vetcalculators.workers.dev`

## Authentication

All API requests require authentication using an API key:

```http
X-API-Key: vd_your_api_key_here
```

Get your API key by contacting [support@vetdrugscalculators.com](mailto:support@vetdrugscalculators.com).

## Endpoints Overview

### Core Calculation

**[Drug Calculations](calculate.md)** - `POST /api/calculate`
- Calculate drug dosages for patients
- Support for multiple drugs per request
- Species-specific dosing (canine/feline)
- Custom concentrations and routes
- Dose range validation and warnings

### Drug Database

**[Drug Search](search.md)** - `GET /api/drugs`
- Search veterinary drugs by name
- Filter by therapeutic category
- Fuzzy matching for spell tolerance
- 500+ drug database with formulations

**[Drug Information](drug-info.md)** - `GET /api/drug-info/{drug_name}`
- Detailed drug information
- Contraindications and side effects
- Drug interactions and monitoring
- Pharmacokinetics and clinical notes

### System Health

**Health Check** - `GET /api/health`
- API service status
- Version information
- Performance metrics
- Uptime statistics

## Request/Response Format

### Standard Request Headers
```http
Content-Type: application/json
X-API-Key: vd_your_api_key
User-Agent: YourApp/1.0
```

### Standard Response Format
```json
{
  "data": {
    // Endpoint-specific response data
  },
  "metadata": {
    "timestamp": "2025-07-11T18:30:00Z",
    "api_version": "2.2.0-enhanced",
    "response_time_ms": 120
  }
}
```

### Error Response Format
```json
{
  "error": "error_code",
  "message": "Human-readable error description",
  "details": "Additional context when available"
}
```

## Rate Limits

**[Rate Limiting](rate-limits.md)** - Comprehensive rate limiting guide
- Per-endpoint rate limits
- Burst protection
- Rate limit headers
- Handling strategies

**Current Limits**:
- **Drug Calculations**: 100 requests/minute
- **Drug Search**: 200 requests/minute  
- **Drug Information**: 150 requests/minute
- **Combined**: 300 requests/minute

## Error Handling

**[Error Handling](errors.md)** - Complete error reference
- HTTP status codes
- Error response formats
- Common error scenarios
- Debugging strategies

**Common Status Codes**:
- `200` - Success
- `400` - Bad Request (validation error)
- `401` - Unauthorized (invalid API key)
- `429` - Too Many Requests (rate limited)
- `503` - Service Unavailable

## Data Models

### Patient Object
```json
{
  "weight_kg": 25.0,
  "species": "dog"
}
```

### Drug Request Object
```json
{
  "drug_name": "Cephalexin",
  "concentration": 250,
  "route": "PO",
  "frequency": "BID"
}
```

### Drug Calculation Response
```json
{
  "drug_name": "Cephalexin",
  "dose_per_kg": 22,
  "total_dose": 550,
  "dose_unit": "mg",
  "volume": 2.2,
  "volume_unit": "ml",
  "instructions": "Give 2.2ml PO BID",
  "dose_range_status": "within_range",
  "warnings": []
}
```

## API Versioning

**Current Version**: 2.2.0-enhanced

**Version Information**:
- Included in response metadata
- Backward compatibility maintained
- Breaking changes use major version increments
- Feature additions use minor version increments

**Version History**:
- **v2.2.0**: Enhanced drug information, improved search
- **v2.1.0**: Drug search endpoint, bulk calculations
- **v2.0.0**: Initial REST API release

## Performance & Reliability

### Response Times
- **Global Average**: <150ms
- **99th Percentile**: <300ms
- **Calculation Endpoint**: 50-120ms
- **Search Endpoint**: 30-100ms

### Availability
- **Uptime SLA**: 99.9%
- **Global CDN**: Cloudflare network
- **Auto-scaling**: Handles traffic spikes
- **Monitoring**: 24/7 system monitoring

### Caching
- **Drug Database**: Cached for performance
- **Search Results**: Intelligent caching
- **Client-Side**: Recommended for frequently accessed data

## SDKs & Libraries

**Official SDKs**:
- [JavaScript/Node.js](../examples/javascript.md)
- [C# (.NET)](../examples/csharp.md)
- [Python](../examples/python.md)
- [PHP](../examples/php.md)

**Community Libraries**:
- Ruby (community-maintained)
- Go (community-maintained)
- Java (community-maintained)

## Testing & Development

### API Testing Tools
```bash
# Health check
curl -H "X-API-Key: your_key" \
  https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/health

# Test calculation  
curl -H "X-API-Key: your_key" \
     -H "Content-Type: application/json" \
     -d '{"patient":{"weight_kg":25,"species":"dog"},"drugs":[{"drug_name":"Cephalexin"}]}' \
     https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/calculate
```

### Development Environment
- **Sandbox API**: Full-featured testing environment
- **Mock Data**: Realistic test data sets
- **Validation Tools**: Request/response validation
- **Debug Headers**: Enhanced debugging information

## Security

### API Security
- **HTTPS Only**: All requests must use HTTPS
- **API Key Authentication**: Secure key-based access
- **Rate Limiting**: Prevents abuse and ensures fair usage
- **Input Validation**: Comprehensive request validation

### Best Practices
- Store API keys securely (environment variables)
- Use HTTPS for all requests
- Implement proper error handling
- Respect rate limits
- Validate responses before using data

## Migration Guides

### Upgrading API Versions
- **v2.1.x → v2.2.0**: No breaking changes, new features available
- **v2.0.x → v2.1.0**: No breaking changes, enhanced capabilities
- **v1.x → v2.0.0**: Major upgrade, see migration guide

### Breaking Change Policy
- **Major versions**: May include breaking changes
- **Minor versions**: Backward compatible feature additions
- **Patch versions**: Bug fixes and improvements
- **Advance notice**: 30+ days for breaking changes

## Support Resources

### Documentation
- **[Integration Guides](../integrations/)**: PMI system integrations
- **[Code Examples](../examples/)**: Language-specific implementations
- **[Support](../support.md)**: Help and troubleshooting

### Community
- **GitHub Discussions**: Community Q&A
- **Stack Overflow**: Technical questions (`vetdrugs-api` tag)
- **Developer Newsletter**: Updates and best practices

### Professional Support
- **Email**: support@vetdrugscalculators.com
- **Enterprise**: Dedicated technical account management
- **Professional Services**: Custom integration development

---

**Ready to start?** Begin with the [Quick Start guide](../quick-start/) or dive into the [endpoint documentation](calculate.md).

**Need help?** Contact [support@vetdrugscalculators.com](mailto:support@vetdrugscalculators.com) for assistance.
