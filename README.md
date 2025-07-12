# VetDrugs API Documentation

> Professional veterinary drug dosing calculations for Practice Management Information (PMI) systems

## Overview

The VetDrugs API provides accurate, real-time drug dosage calculations for veterinary professionals. Trusted by veterinary practices across North America for precise medication dosing.

### Key Features

- ✅ **500+ Veterinary Drugs** - Comprehensive database of veterinary medications
- ✅ **Species-Specific Dosing** - Accurate calculations for dogs, cats, and other species
- ✅ **CRI Calculations** - Constant rate infusion protocols
- ✅ **Fluid Therapy** - Maintenance and deficit fluid calculations
- ✅ **Safety Warnings** - Species contraindications and dose range validation
- ✅ **Enterprise Ready** - 99.9% uptime, rate limiting, API key management

### Quick Start

Get your first drug calculation in under 5 minutes:

```bash
curl -X POST "https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/calculate" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your_api_key" \
  -d '{
    "patient": {
      "weight_kg": 25,
      "species": "dog"
    },
    "drugs": [
      {"drug_name": "Cephalexin"}
    ]
  }'
```

### API Endpoint

```
https://vetdrugs-calculator-api.vetcalculators.workers.dev/
```

### Getting Started

1. [Get API Key](quick-start/authentication.md)
2. [Make Your First Request](quick-start/first-request.md)
3. [Integration Examples](integrations/)

### Support

- 📧 Email: support@vetdrugscalculators.com
- 📖 Documentation: docs.vetdrugscalculators.com
- 🔍 API Health: `/api/health` endpoint

---

**Ready to integrate? Check out our [Quick Start Guide](quick-start/) →**# Test GitHub Pages Deploy

This is a test commit to trigger GitHub Pages deployment.

## Documentation Structure ✅

- API Reference: Complete
- Integration Guides: Complete  
- Code Examples: Complete
- Support Documentation: Complete

**Total**: 14 files, 9,120+ lines of professional documentation

---

**Deployment Status**: Testing GitHub Pages setup

**Next**: Verify all links work and search functionality is enabled.
