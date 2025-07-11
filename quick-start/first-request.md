# Your First Request

This guide walks you through making your first drug calculation request.

## Basic Drug Calculation

The most common use case is calculating a drug dose for a patient.

### Request Format

```bash
POST https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/calculate
```

### Required Headers

```
Content-Type: application/json
X-API-Key: your_api_key_here
```

### Request Body

```json
{
  "patient": {
    "weight_kg": 25,
    "species": "dog"
  },
  "drugs": [
    {
      "drug_name": "Cephalexin"
    }
  ]
}
```

### Complete Example

```javascript
// JavaScript example
const response = await fetch('https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/calculate', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-API-Key': 'vd_your_api_key_here'
  },
  body: JSON.stringify({
    patient: {
      weight_kg: 25,
      species: 'dog'
    },
    drugs: [
      { drug_name: 'Cephalexin' }
    ]
  })
});

const result = await response.json();
```

### cURL Example

```bash
curl -X POST "https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/calculate" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: vd_your_api_key_here" \
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

## Response Format

```json
{
  "patient": {
    "weight_kg": 25,
    "species": "dog"
  },
  "calculations": [
    {
      "drug_name": "Cephalexin",
      "dose_per_kg": 22,
      "total_dose": 550,
      "dose_unit": "mg",
      "dosage_unit": "mg/kg",
      "volume": 2.2,
      "volume_unit": "ml",
      "concentration": 250,
      "concentration_unit": "mg/ml",
      "route": "PO",
      "frequency": "BID",
      "instructions": "Give 2.2ml PO BID",
      "dose_range_status": "within_range",
      "warnings": [],
      "calculation_details": {
        "formula": "Weight-based: 25kg × 22mg/kg = 550mg",
        "full_calculation": "25kg × 22mg/kg ÷ 250mg/ml = 2.2ml"
      }
    }
  ],
  "metadata": {
    "calculation_timestamp": "2025-07-11T18:30:00Z",
    "api_version": "2.2.0",
    "drugs_processed": 1,
    "drugs_not_found": []
  }
}
```

## Multiple Drugs

You can calculate multiple drugs in one request:

```json
{
  "patient": {
    "weight_kg": 15,
    "species": "cat"
  },
  "drugs": [
    {"drug_name": "Cephalexin"},
    {"drug_name": "Meloxicam"}
  ]
}
```

## Next Steps

- [Learn about all API endpoints](../api-reference/)
- [See integration examples](../integrations/)
- [Handle errors properly](../api-reference/errors.md)
- [Understanding rate limits](../api-reference/rate-limits.md)