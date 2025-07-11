# Drug Information API

Get detailed information about specific drugs in the VetDrugs database.

## Endpoint

```
GET /api/drug-info/{drug_name}
```

## Description

Retrieve comprehensive information about a specific drug, including contraindications, side effects, drug interactions, and detailed dosing guidelines.

## Path Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `drug_name` | String | Yes | Exact drug name to lookup |

## Request Examples

### Get Cephalexin Information

```bash
curl -H "X-API-Key: vd_your_api_key" \
     "https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/drug-info/Cephalexin"
```

### Get NSAID Information

```bash
curl -H "X-API-Key: vd_your_api_key" \
     "https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/drug-info/Carprofen"
```

## Response Format

```json
{
  "drug": {
    "name": "Cephalexin",
    "drug_name": "Cephalexin",
    "category": "Antibiotic",
    "class": "Beta-lactam",
    "description": "First-generation cephalosporin antibiotic with broad-spectrum activity against gram-positive bacteria and limited gram-negative coverage.",
    "default_dose": 22,
    "dose_unit": "mg",
    "dosage_unit": "mg/kg",
    "dose_range": {
      "min": 15,
      "max": 30,
      "unit": "mg/kg"
    },
    "route": "PO",
    "frequency": "BID",
    "species": ["dog", "cat"],
    "has_feline_dose": true,
    "feline_dose": {
      "default": 25,
      "range": {
        "min": 20,
        "max": 30
      },
      "frequency": "BID"
    },
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
        "name": "Capsules 250mg",
        "value": 250,
        "unit": "mg/cap",
        "selected": false
      },
      {
        "id": 3,
        "name": "Capsules 500mg",
        "value": 500,
        "unit": "mg/cap",
        "selected": false
      }
    ],
    "contraindications": [
      "Known hypersensitivity to cephalexin or other cephalosporin antibiotics",
      "History of severe penicillin allergy (cross-reactivity possible)",
      "Severe renal impairment without dose adjustment"
    ],
    "side_effects": [
      "Gastrointestinal upset (vomiting, diarrhea)",
      "Loss of appetite",
      "Allergic reactions (rare)",
      "Superinfections with prolonged use"
    ],
    "drug_interactions": [
      "Probenecid may increase cephalexin levels",
      "May potentiate anticoagulant effects of warfarin",
      "Concurrent aminoglycosides may increase nephrotoxicity risk"
    ],
    "monitoring": [
      "Clinical response to therapy",
      "Signs of gastrointestinal upset",
      "Allergic reactions",
      "Kidney function with prolonged use"
    ],
    "storage": "Store at room temperature (15-30°C). Refrigerate oral suspension after reconstitution.",
    "pregnancy_safety": "Category A - Generally safe for use in pregnant animals",
    "brand_names": ["Keflex", "Rilexine", "Ceporex"],
    "mechanism_of_action": "Inhibits bacterial cell wall synthesis by binding to penicillin-binding proteins",
    "pharmacokinetics": {
      "absorption": "Well absorbed orally in dogs and cats",
      "protein_binding": "10-15%",
      "metabolism": "Minimal hepatic metabolism",
      "elimination": "Primarily renal excretion",
      "half_life": {
        "dog": "1.5-2 hours",
        "cat": "2-3 hours"
      }
    },
    "clinical_notes": [
      "Give with food to reduce gastrointestinal upset",
      "Complete full course even if symptoms improve",
      "Not effective against methicillin-resistant staphylococci",
      "Good choice for skin and soft tissue infections"
    ]
  },
  "metadata": {
    "last_updated": "2025-07-11T18:30:00Z",
    "data_source": "kv-cached",
    "api_version": "2.2.0-enhanced"
  }
}
```

## Drug Properties Explained

### Basic Information
| Property | Description |
|----------|-------------|
| `name` | Primary drug name |
| `category` | Therapeutic classification |
| `class` | Pharmacological class |
| `description` | Clinical description and uses |

### Dosing Information
| Property | Description |
|----------|-------------|
| `default_dose` | Standard dose per kg |
| `dose_range` | Minimum and maximum recommended doses |
| `route` | Administration route (PO, IV, IM, SC, etc.) |
| `frequency` | Dosing frequency (SID, BID, TID, QID) |

### Species-Specific Information
| Property | Description |
|----------|-------------|
| `species` | Applicable species array |
| `has_feline_dose` | Whether cat-specific dosing exists |
| `feline_dose` | Cat-specific dosing when different from dogs |

### Safety Information
| Property | Description |
|----------|-------------|
| `contraindications` | When not to use the drug |
| `side_effects` | Potential adverse effects |
| `drug_interactions` | Drugs that interact |
| `monitoring` | What to monitor during therapy |

### Clinical Information
| Property | Description |
|----------|-------------|
| `mechanism_of_action` | How the drug works |
| `pharmacokinetics` | Absorption, distribution, metabolism, excretion |
| `clinical_notes` | Practical usage tips |

## Searching by Category

Get all drugs in a specific category:

```bash
curl -H "X-API-Key: vd_your_api_key" \
     "https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/drugs?category=antibiotic"
```

## JavaScript Example

```javascript
const getDrugInfo = async (drugName) => {
  try {
    const response = await fetch(
      `https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/drug-info/${encodeURIComponent(drugName)}`,
      {
        headers: {
          'X-API-Key': 'vd_your_api_key'
        }
      }
    );

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    const data = await response.json();
    return data.drug;
  } catch (error) {
    console.error('Failed to fetch drug info:', error);
    throw error;
  }
};

// Usage
const drugInfo = await getDrugInfo('Cephalexin');
console.log(`${drugInfo.name}: ${drugInfo.description}`);
console.log(`Dose: ${drugInfo.default_dose} ${drugInfo.dosage_unit}`);
console.log(`Contraindications: ${drugInfo.contraindications.join(', ')}`);
```

## C# Example

```csharp
public class DrugInfoService
{
    private readonly VetDrugsApiClient _client;

    public DrugInfoService(string apiKey)
    {
        _client = new VetDrugsApiClient(apiKey);
    }

    public async Task<DetailedDrugInfo> GetDrugInfoAsync(string drugName)
    {
        var response = await _client.GetDrugInfoAsync(drugName);
        return response.Drug;
    }

    public async Task<List<string>> GetContraindicationsAsync(string drugName)
    {
        var drugInfo = await GetDrugInfoAsync(drugName);
        return drugInfo.Contraindications;
    }

    public async Task<string> GetDosageInstructionsAsync(string drugName, string species)
    {
        var drugInfo = await GetDrugInfoAsync(drugName);
        
        if (species.ToLower() == "cat" && drugInfo.HasFelineDose)
        {
            return $"{drugInfo.FelineDose.Default} mg/kg {drugInfo.FelineDose.Frequency}";
        }
        
        return $"{drugInfo.DefaultDose} mg/kg {drugInfo.Frequency}";
    }
}
```

## Common Drug Categories

| Category | Description | Example Drugs |
|----------|-------------|---------------|
| `antibiotic` | Antimicrobial agents | Cephalexin, Amoxicillin, Enrofloxacin |
| `nsaid` | Non-steroidal anti-inflammatory | Carprofen, Meloxicam, Firocoxib |
| `analgesic` | Pain relief medications | Tramadol, Gabapentin, Buprenorphine |
| `sedative` | Sedation and anxiety | Acepromazine, Trazodone, Gabapentin |
| `cardiac` | Heart medications | Enalapril, Pimobendan, Digoxin |
| `respiratory` | Respiratory treatments | Theophylline, Terbutaline |
| `fluid` | Fluid therapy calculations | LRS, Saline, Dextrose |

## Error Responses

### Drug Not Found (404)
```json
{
  "error": "drug_not_found",
  "message": "Drug 'InvalidDrugName' was not found in the database"
}
```

### Invalid Drug Name (400)
```json
{
  "error": "invalid_parameter",
  "message": "Drug name parameter is required and cannot be empty"
}
```

## Use Cases

1. **Clinical Decision Support**: Show contraindications and warnings
2. **Client Education**: Generate drug information handouts
3. **Inventory Management**: Cross-reference with available formulations
4. **Drug Interaction Checking**: Warn about potential interactions
5. **Dosing Guidelines**: Provide species-specific dosing recommendations

## Integration Tips

### Building Drug Information Cards

```javascript
const createDrugInfoCard = (drugInfo) => {
  return {
    title: drugInfo.name,
    subtitle: `${drugInfo.category} • ${drugInfo.class}`,
    description: drugInfo.description,
    dosing: `${drugInfo.default_dose} ${drugInfo.dosage_unit} ${drugInfo.frequency}`,
    warnings: drugInfo.contraindications.length,
    interactions: drugInfo.drug_interactions.length,
    species: drugInfo.species.join(', ')
  };
};
```

### PMI System Integration

For practice management systems, consider caching drug information locally and updating periodically:

```csharp
public class CachedDrugInfoService
{
    private readonly Dictionary<string, DetailedDrugInfo> _cache = new();
    private readonly TimeSpan _cacheExpiry = TimeSpan.FromHours(24);
    private readonly VetDrugsApiClient _client;

    public async Task<DetailedDrugInfo> GetDrugInfoAsync(string drugName)
    {
        if (_cache.TryGetValue(drugName, out var cached))
        {
            return cached;
        }

        var drugInfo = await _client.GetDrugInfoAsync(drugName);
        _cache[drugName] = drugInfo;
        
        return drugInfo;
    }
}
```

## Next Steps

- [View search endpoint](search.md)
- [Explore calculation API](calculate.md)
- [Check error handling](errors.md)
