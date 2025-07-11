# Calculate Drugs

Calculate drug dosages for veterinary patients.

## Endpoint

```
POST /api/calculate
```

## Description

Calculate accurate drug dosages based on patient weight, species, and selected medications. Supports both individual drugs and multiple drug calculations in a single request.

## Request Format

### Headers

| Header | Required | Description |
|--------|----------|-------------|
| `Content-Type` | Yes | `application/json` |
| `X-API-Key` | Yes | Your VetDrugs API key |

### Body Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `patient` | Object | Yes | Patient information |
| `patient.weight_kg` | Number | Yes | Patient weight in kilograms |
| `patient.species` | String | Yes | Patient species: `dog`, `cat` |
| `drugs` | Array | Yes | Array of drugs to calculate |
| `drugs[].drug_name` | String | Yes | Name of the drug |
| `drugs[].concentration` | Number | No | Specific concentration if drug has multiple options |
| `drugs[].route` | String | No | Override default route |
| `drugs[].frequency` | String | No | Override default frequency |

### Example Request

```json
{
  "patient": {
    "weight_kg": 25,
    "species": "dog"
  },
  "drugs": [
    {
      "drug_name": "Cephalexin"
    },
    {
      "drug_name": "Carprofen",
      "concentration": 50
    }
  ]
}
```

## Response Format

### Success Response (200 OK)

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
        "full_calculation": "25kg × 22mg/kg ÷ 250mg/ml = 2.2ml",
        "calculation_type": "Weight-based",
        "species_specific": "standard dose used"
      }
    }
  ],
  "metadata": {
    "calculation_timestamp": "2025-07-11T18:30:00Z",
    "api_version": "2.2.0",
    "drugs_processed": 1,
    "drugs_not_found": [],
    "total_drugs_in_database": 500
  }
}
```

## Dose Range Status

| Status | Description |
|--------|-------------|
| `within_range` | Dose is within recommended range |
| `below_range` | Dose is below minimum recommended |
| `above_range` | Dose is above maximum recommended |
| `contraindicated` | Drug is contraindicated for this species |

## Species-Specific Dosing

The API automatically uses species-specific doses when available:
- **Dogs**: Standard dosing protocols
- **Cats**: Feline-specific doses where different from dogs

## Error Responses

### Drug Not Found (200 OK with warnings)

```json
{
  "patient": { ... },
  "calculations": [],
  "warnings": [
    "The following drugs were not found: InvalidDrugName"
  ],
  "metadata": {
    "drugs_not_found": ["InvalidDrugName"]
  }
}
```

### Validation Error (400 Bad Request)

```json
{
  "error": "validation_failed",
  "messages": [
    "Valid patient weight is required",
    "Patient species must be 'dog' or 'cat'"
  ]
}
```

### Authentication Error (401 Unauthorized)

```json
{
  "error": "invalid_api_key",
  "message": "API key is required or invalid"
}
```

## Rate Limits

- Trial: 100 requests/hour
- Professional: 1,000 requests/hour  
- Enterprise: 10,000 requests/hour

Rate limit headers are included in responses:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1625097600
```

## Examples

### Single Drug Calculation

```bash
curl -X POST "https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/calculate" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: vd_your_api_key" \
  -d '{
    "patient": {"weight_kg": 25, "species": "dog"},
    "drugs": [{"drug_name": "Cephalexin"}]
  }'
```

### Cat with Multiple Drugs

```bash
curl -X POST "https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/calculate" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: vd_your_api_key" \
  -d '{
    "patient": {"weight_kg": 4.5, "species": "cat"},
    "drugs": [
      {"drug_name": "Cephalexin"},
      {"drug_name": "Meloxicam"}
    ]
  }'
```