# Search Drugs

Search and list available drugs in the VetDrugs database.

## Endpoint

```
GET /api/drugs
```

## Description

Search through the comprehensive database of veterinary drugs. Use this endpoint to find available medications, check spelling, or build drug selection interfaces.

## Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `search` | String | No | Search term for drug name |
| `category` | String | No | Filter by drug category |

## Request Examples

### List All Drugs

```bash
curl -H "X-API-Key: vd_your_api_key" \
     "https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/drugs"
```

### Search for Specific Drug

```bash
curl -H "X-API-Key: vd_your_api_key" \
     "https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/drugs?search=cephalexin"
```

### Filter by Category

```bash
curl -H "X-API-Key: vd_your_api_key" \
     "https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/drugs?category=antibiotic"
```

## Response Format

```json
{
  "drugs": [
    {
      "name": "Cephalexin",
      "drug_name": "Cephalexin",
      "category": "Antibiotic",
      "class": "Beta-lactam",
      "default_dose": 22,
      "dose_unit": "mg",
      "dosage_unit": "mg/kg",
      "dose_range": {
        "min": 15,
        "max": 30,
        "unit": "mg/kg"
      },
      "route": "PO",
      "species": ["dog", "cat"],
      "has_feline_dose": true,
      "has_cri": false,
      "has_fluid_therapy": false,
      "concentration_options": [
        {
          "id": 1,
          "name": "Oral Suspension",
          "value": 250,
          "unit": "mg/ml",
          "selected": true
        },
        {
          "id": 2,
          "name": "Capsules",
          "value": 500,
          "unit": "mg/cap",
          "selected": false
        }
      ],
      "brand_name": "Keflex"
    }
  ],
  "total_count": 1,
  "metadata": {
    "search_term": "cephalexin",
    "category": null,
    "timestamp": "2025-07-11T18:30:00Z",
    "total_in_database": 500,
    "data_source": "kv-cached",
    "api_version": "2.2.0-enhanced"
  }
}
```

## Drug Object Properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | String | Primary drug name |
| `category` | String | Drug classification (Antibiotic, NSAID, etc.) |
| `default_dose` | Number | Standard dose per kg |
| `dose_range` | Object | Min/max recommended doses |
| `route` | String | Administration route (PO, IV, IM, etc.) |
| `species` | Array | Applicable species |
| `has_feline_dose` | Boolean | Has cat-specific dosing |
| `has_cri` | Boolean | Supports CRI calculations |
| `concentration_options` | Array | Available formulations |

## Search Features

### Fuzzy Matching
The search supports partial matches and typos:
```bash
# These all find "Cephalexin"
?search=ceph
?search=cepha
?search=keflex
```

### Multi-word Search
```bash
# Finds "Amoxicillin Clavulanate" 
?search=amoxicillin clav
```

### Category Filtering
Available categories include:
- `antibiotic`
- `nsaid`
- `analgesic`
- `sedative`
- `fluid`
- `cardiac`
- `respiratory`

## JavaScript Example

```javascript
// Search for antibiotics
const searchDrugs = async (searchTerm) => {
  const response = await fetch(
    `https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/drugs?search=${encodeURIComponent(searchTerm)}`,
    {
      headers: {
        'X-API-Key': 'vd_your_api_key'
      }
    }
  );
  
  const data = await response.json();
  return data.drugs;
};

// Usage
const antibiotics = await searchDrugs('antibiotic');
console.log(`Found ${antibiotics.length} antibiotics`);
```

## Error Responses

### Invalid API Key (401)
```json
{
  "error": "invalid_api_key",
  "message": "API key is required or invalid"
}
```

### Service Unavailable (503)
```json
{
  "error": "service_unavailable",
  "message": "Drug data is unavailable from KV store"
}
```

## Use Cases

1. **Drug Selection Interface**: Build searchable drug lists in PMI systems
2. **Spell Checking**: Validate drug names before calculation
3. **Category Browsing**: Show drugs by therapeutic category
4. **Autocomplete**: Implement typeahead drug search
5. **Inventory Management**: Cross-reference with practice formulary