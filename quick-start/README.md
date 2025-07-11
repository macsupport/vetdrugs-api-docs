# Quick Start Guide

Get up and running with the VetDrugs API in 15 minutes.

## 1. Get Your API Key

Contact us at support@vetdrugscalculators.com to get your API key.

## 2. Test Connection

```bash
curl "https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/health"
```

## 3. Make Your First Calculation

```javascript
const response = await fetch('https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/calculate', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-API-Key': 'your_api_key_here'
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
console.log(result);
```

## Expected Response

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
      "volume": 2.2,
      "volume_unit": "ml",
      "concentration": 250,
      "concentration_unit": "mg/ml",
      "instructions": "Give 2.2ml PO BID",
      "dose_range_status": "within_range",
      "warnings": []
    }
  ],
  "metadata": {
    "calculation_timestamp": "2025-07-11T18:30:00Z",
    "api_version": "2.2.0"
  }
}
```

## Next Steps

- [View all API endpoints](../api-reference/)
- [See integration examples](../integrations/)
- [Download SDKs](../examples/)
- [Authentication guide](authentication.md)