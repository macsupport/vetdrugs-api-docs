# ezyVet Integration Guide

Complete integration guide for ezyVet Practice Management Software.

## Overview

ezyVet is a cloud-based veterinary practice management system with REST API capabilities. This guide shows how to integrate VetDrugs API calculations directly into your ezyVet workflows.

## Prerequisites

- ezyVet Practice Management Software
- ezyVet API access credentials
- VetDrugs API key
- Developer access to custom integrations

## Integration Architecture

```
ezyVet → Custom Integration → VetDrugs API → Drug Calculations
```

## Setup Instructions

### 1. ezyVet API Access

First, obtain API credentials from ezyVet:

1. **Contact ezyVet Support** to enable API access
2. **Get your API endpoint** (typically `https://api.trial.ezyvet.com` or `https://api.ezyvet.com`)
3. **Obtain API credentials**:
   - Partner ID
   - Client ID
   - Client Secret
   - Username/Password for OAuth

### 2. Authentication with ezyVet

```javascript
// ezyVet OAuth authentication
const getEzyVetToken = async () => {
  const response = await fetch('https://api.ezyvet.com/v1/oauth/access_token', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded'
    },
    body: new URLSearchParams({
      'grant_type': 'password',
      'client_id': 'your_client_id',
      'client_secret': 'your_client_secret',
      'username': 'your_username',
      'password': 'your_password',
      'partner_id': 'your_partner_id'
    })
  });
  
  const data = await response.json();
  return data.access_token;
};
```

### 3. Integration Implementation

#### Complete Integration Class

```javascript
class EzyVetVetDrugsIntegration {
  constructor(ezyVetConfig, vetDrugsApiKey) {
    this.ezyVetConfig = ezyVetConfig;
    this.vetDrugsApiKey = vetDrugsApiKey;
    this.ezyVetToken = null;
    this.tokenExpiry = null;
  }

  async authenticate() {
    if (this.ezyVetToken && this.tokenExpiry > Date.now()) {
      return this.ezyVetToken;
    }

    const response = await fetch(`${this.ezyVetConfig.apiUrl}/v1/oauth/access_token`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded'
      },
      body: new URLSearchParams({
        'grant_type': 'password',
        'client_id': this.ezyVetConfig.clientId,
        'client_secret': this.ezyVetConfig.clientSecret,
        'username': this.ezyVetConfig.username,
        'password': this.ezyVetConfig.password,
        'partner_id': this.ezyVetConfig.partnerId
      })
    });

    if (!response.ok) {
      throw new Error('ezyVet authentication failed');
    }

    const data = await response.json();
    this.ezyVetToken = data.access_token;
    this.tokenExpiry = Date.now() + (data.expires_in * 1000);
    
    return this.ezyVetToken;
  }

  async getPatientById(patientId) {
    const token = await this.authenticate();
    
    const response = await fetch(`${this.ezyVetConfig.apiUrl}/v1/animal/${patientId}`, {
      headers: {
        'Authorization': `Bearer ${token}`
      }
    });

    if (!response.ok) {
      throw new Error(`Failed to fetch patient ${patientId}`);
    }

    const data = await response.json();
    return data.animal;
  }

  async calculateDrugsForPatient(patientId, drugNames) {
    try {
      // Get patient data from ezyVet
      const patient = await this.getPatientById(patientId);
      
      // Convert ezyVet data to VetDrugs format
      const vetDrugsPatient = this.convertEzyVetPatient(patient);
      
      // Prepare drug requests
      const drugs = drugNames.map(name => ({ drug_name: name }));
      
      // Calculate using VetDrugs API
      const calculationResponse = await this.calculateWithVetDrugs(vetDrugsPatient, drugs);
      
      // Log results back to ezyVet
      await this.logCalculationsToEzyVet(patientId, calculationResponse);
      
      return calculationResponse;
      
    } catch (error) {
      console.error('Drug calculation failed:', error);
      throw error;
    }
  }

  convertEzyVetPatient(ezyVetPatient) {
    // Convert ezyVet weight to kg if needed
    let weightKg = parseFloat(ezyVetPatient.weight);
    
    // ezyVet stores weight in practice's preferred unit
    if (ezyVetPatient.weight_unit === 'lb') {
      weightKg = weightKg * 0.453592;
    }

    // Map ezyVet species to VetDrugs format
    const speciesMap = {
      'Canine': 'dog',
      'Feline': 'cat',
      'Dog': 'dog',
      'Cat': 'cat'
    };

    const species = speciesMap[ezyVetPatient.species_name] || 'dog';

    return {
      weight_kg: Math.round(weightKg * 100) / 100,
      species: species
    };
  }

  async calculateWithVetDrugs(patient, drugs) {
    const response = await fetch('https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/calculate', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-API-Key': this.vetDrugsApiKey
      },
      body: JSON.stringify({
        patient: patient,
        drugs: drugs
      })
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(`VetDrugs API error: ${error.message}`);
    }

    return await response.json();
  }

  async logCalculationsToEzyVet(patientId, calculations) {
    const token = await this.authenticate();
    
    // Create consultation note with drug calculations
    const noteContent = this.formatCalculationsForEzyVet(calculations);
    
    const response = await fetch(`${this.ezyVetConfig.apiUrl}/v1/consult`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        animal_id: patientId,
        consult_type_id: 1, // Adjust based on your setup
        status_id: 1,
        details: noteContent,
        created_at: new Date().toISOString()
      })
    });

    if (!response.ok) {
      console.warn('Failed to log calculations to ezyVet');
    }
  }

  formatCalculationsForEzyVet(response) {
    let content = `=== VetDrugs Calculations ===\n`;
    content += `Patient: ${response.patient.weight_kg}kg ${response.patient.species}\n`;
    content += `Calculated: ${new Date().toLocaleString()}\n\n`;

    response.calculations.forEach(calc => {
      content += `${calc.drug_name}:\n`;
      content += `  Instructions: ${calc.instructions}\n`;
      content += `  Dose: ${calc.dose_per_kg} ${calc.dosage_unit}\n`;
      content += `  Total dose: ${calc.total_dose} ${calc.dose_unit}\n`;
      content += `  Volume: ${calc.volume} ${calc.volume_unit}\n`;
      content += `  Route: ${calc.route}, Frequency: ${calc.frequency}\n`;
      
      if (calc.warnings && calc.warnings.length > 0) {
        content += `  ⚠️ Warnings: ${calc.warnings.join(', ')}\n`;
      }
      
      content += `\n`;
    });

    if (response.warnings && response.warnings.length > 0) {
      content += `⚠️ General Warnings: ${response.warnings.join(', ')}\n`;
    }

    return content;
  }

  async searchDrugsInVetDrugs(searchTerm) {
    const response = await fetch(
      `https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/drugs?search=${encodeURIComponent(searchTerm)}`,
      {
        headers: {
          'X-API-Key': this.vetDrugsApiKey
        }
      }
    );

    if (!response.ok) {
      throw new Error('Drug search failed');
    }

    const data = await response.json();
    return data.drugs;
  }
}
```

### 4. ezyVet Custom Button Integration

Create a custom button in ezyVet to trigger drug calculations:

```javascript
// This would be implemented as an ezyVet custom integration
class EzyVetCustomButton {
  constructor() {
    this.integration = new EzyVetVetDrugsIntegration(ezyVetConfig, vetDrugsApiKey);
  }

  async handleCalculateButtonClick(animalId) {
    try {
      // Show loading state
      this.showLoading('Calculating drug dosages...');
      
      // Get selected drugs from UI
      const selectedDrugs = this.getSelectedDrugs();
      
      if (selectedDrugs.length === 0) {
        this.showMessage('Please select at least one drug to calculate.');
        return;
      }

      // Calculate drugs
      const results = await this.integration.calculateDrugsForPatient(animalId, selectedDrugs);
      
      // Display results
      this.displayResults(results);
      
    } catch (error) {
      this.showError(`Calculation failed: ${error.message}`);
    } finally {
      this.hideLoading();
    }
  }

  getSelectedDrugs() {
    // Implementation depends on your ezyVet UI setup
    // This could be checkboxes, dropdown selections, etc.
    const checkboxes = document.querySelectorAll('.drug-checkbox:checked');
    return Array.from(checkboxes).map(cb => cb.value);
  }

  displayResults(results) {
    const modal = this.createResultsModal();
    
    let html = `
      <div class="vetdrugs-results">
        <h3>Drug Calculations for ${results.patient.weight_kg}kg ${results.patient.species}</h3>
        <div class="calculations">
    `;

    results.calculations.forEach(calc => {
      const statusColor = this.getStatusColor(calc.dose_range_status);
      
      html += `
        <div class="calculation-card" style="border-left: 4px solid ${statusColor}">
          <h4>${calc.drug_name}</h4>
          <p class="instructions">${calc.instructions}</p>
          <div class="details">
            <span>Dose: ${calc.dose_per_kg} ${calc.dosage_unit}</span>
            <span>Volume: ${calc.volume} ${calc.volume_unit}</span>
            <span>Status: ${this.getStatusText(calc.dose_range_status)}</span>
          </div>
          ${calc.warnings.length > 0 ? 
            `<div class="warnings">⚠️ ${calc.warnings.join(', ')}</div>` : ''}
        </div>
      `;
    });

    html += `
        </div>
        <div class="actions">
          <button onclick="this.addToTreatmentPlan()">Add to Treatment Plan</button>
          <button onclick="this.printInstructions()">Print Instructions</button>
          <button onclick="this.closeModal()">Close</button>
        </div>
      </div>
    `;

    modal.innerHTML = html;
    document.body.appendChild(modal);
  }

  getStatusColor(status) {
    const colors = {
      'within_range': '#4CAF50',
      'below_range': '#FF9800',
      'above_range': '#F44336',
      'contraindicated': '#9C27B0'
    };
    return colors[status] || '#757575';
  }

  getStatusText(status) {
    const texts = {
      'within_range': 'Normal dose',
      'below_range': 'Below range',
      'above_range': 'Above range',
      'contraindicated': 'Contraindicated'
    };
    return texts[status] || 'Unknown';
  }

  createResultsModal() {
    const modal = document.createElement('div');
    modal.className = 'vetdrugs-modal';
    modal.style.cssText = `
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      background: rgba(0,0,0,0.5);
      display: flex;
      align-items: center;
      justify-content: center;
      z-index: 10000;
    `;
    return modal;
  }
}
```

### 5. PHP Backend Integration

For server-side integration with ezyVet:

```php
<?php

class EzyVetVetDrugsIntegration {
    private $ezyVetConfig;
    private $vetDrugsApiKey;
    private $ezyVetToken;
    private $tokenExpiry;

    public function __construct($ezyVetConfig, $vetDrugsApiKey) {
        $this->ezyVetConfig = $ezyVetConfig;
        $this->vetDrugsApiKey = $vetDrugsApiKey;
    }

    public function authenticate() {
        if ($this->ezyVetToken && $this->tokenExpiry > time()) {
            return $this->ezyVetToken;
        }

        $data = array(
            'grant_type' => 'password',
            'client_id' => $this->ezyVetConfig['client_id'],
            'client_secret' => $this->ezyVetConfig['client_secret'],
            'username' => $this->ezyVetConfig['username'],
            'password' => $this->ezyVetConfig['password'],
            'partner_id' => $this->ezyVetConfig['partner_id']
        );

        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $this->ezyVetConfig['api_url'] . '/v1/oauth/access_token');
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($data));
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_HTTPHEADER, array('Content-Type: application/x-www-form-urlencoded'));

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($httpCode !== 200) {
            throw new Exception('ezyVet authentication failed');
        }

        $tokenData = json_decode($response, true);
        $this->ezyVetToken = $tokenData['access_token'];
        $this->tokenExpiry = time() + $tokenData['expires_in'];

        return $this->ezyVetToken;
    }

    public function getPatient($patientId) {
        $token = $this->authenticate();

        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $this->ezyVetConfig['api_url'] . '/v1/animal/' . $patientId);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_HTTPHEADER, array(
            'Authorization: Bearer ' . $token
        ));

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($httpCode !== 200) {
            throw new Exception('Failed to fetch patient from ezyVet');
        }

        $data = json_decode($response, true);
        return $data['animal'];
    }

    public function calculateDrugs($patientId, $drugNames) {
        // Get patient from ezyVet
        $ezyVetPatient = $this->getPatient($patientId);
        
        // Convert to VetDrugs format
        $patient = $this->convertPatient($ezyVetPatient);
        
        // Prepare drugs array
        $drugs = array();
        foreach ($drugNames as $drugName) {
            $drugs[] = array('drug_name' => $drugName);
        }

        // Call VetDrugs API
        $calculationData = array(
            'patient' => $patient,
            'drugs' => $drugs
        );

        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, 'https://vetdrugs-calculator-api.vetcalculators.workers.dev/api/calculate');
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($calculationData));
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_HTTPHEADER, array(
            'Content-Type: application/json',
            'X-API-Key: ' . $this->vetDrugsApiKey
        ));

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($httpCode !== 200) {
            throw new Exception('VetDrugs calculation failed');
        }

        $result = json_decode($response, true);
        
        // Log to ezyVet
        $this->logToEzyVet($patientId, $result);
        
        return $result;
    }

    private function convertPatient($ezyVetPatient) {
        // Convert weight to kg
        $weightKg = floatval($ezyVetPatient['weight']);
        if ($ezyVetPatient['weight_unit'] === 'lb') {
            $weightKg = $weightKg * 0.453592;
        }

        // Map species
        $speciesMap = array(
            'Canine' => 'dog',
            'Feline' => 'cat',
            'Dog' => 'dog',
            'Cat' => 'cat'
        );

        $species = isset($speciesMap[$ezyVetPatient['species_name']]) 
                  ? $speciesMap[$ezyVetPatient['species_name']] 
                  : 'dog';

        return array(
            'weight_kg' => round($weightKg, 2),
            'species' => $species
        );
    }

    private function logToEzyVet($patientId, $calculations) {
        $token = $this->authenticate();
        
        $noteContent = $this->formatCalculations($calculations);
        
        $consultData = array(
            'animal_id' => $patientId,
            'consult_type_id' => 1,
            'status_id' => 1,
            'details' => $noteContent,
            'created_at' => date('c')
        );

        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $this->ezyVetConfig['api_url'] . '/v1/consult');
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($consultData));
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_HTTPHEADER, array(
            'Authorization: Bearer ' . $token,
            'Content-Type: application/json'
        ));

        curl_exec($ch);
        curl_close($ch);
    }

    private function formatCalculations($response) {
        $content = "=== VetDrugs Calculations ===\n";
        $content .= "Patient: " . $response['patient']['weight_kg'] . "kg " . $response['patient']['species'] . "\n";
        $content .= "Calculated: " . date('Y-m-d H:i:s') . "\n\n";

        foreach ($response['calculations'] as $calc) {
            $content .= $calc['drug_name'] . ":\n";
            $content .= "  Instructions: " . $calc['instructions'] . "\n";
            $content .= "  Dose: " . $calc['dose_per_kg'] . " " . $calc['dosage_unit'] . "\n";
            $content .= "  Total dose: " . $calc['total_dose'] . " " . $calc['dose_unit'] . "\n";
            $content .= "  Volume: " . $calc['volume'] . " " . $calc['volume_unit'] . "\n";
            
            if (!empty($calc['warnings'])) {
                $content .= "  ⚠️ Warnings: " . implode(', ', $calc['warnings']) . "\n";
            }
            
            $content .= "\n";
        }

        return $content;
    }
}

// Usage example
$ezyVetConfig = array(
    'api_url' => 'https://api.ezyvet.com',
    'client_id' => 'your_client_id',
    'client_secret' => 'your_client_secret',
    'username' => 'your_username',
    'password' => 'your_password',
    'partner_id' => 'your_partner_id'
);

$integration = new EzyVetVetDrugsIntegration($ezyVetConfig, 'vd_your_api_key');

try {
    $result = $integration->calculateDrugs(12345, ['Cephalexin', 'Carprofen']);
    echo json_encode($result);
} catch (Exception $e) {
    echo "Error: " . $e->getMessage();
}
?>
```

## ezyVet Webhook Integration

Set up webhooks to automatically trigger calculations:

```javascript
// Express.js webhook endpoint
app.post('/ezyvet-webhook', express.json(), async (req, res) => {
  try {
    const { event_type, animal_id, consult_id } = req.body;
    
    if (event_type === 'consult.created') {
      // Auto-calculate common drugs for new consultations
      const commonDrugs = ['Cephalexin', 'Carprofen']; // Customize as needed
      
      const integration = new EzyVetVetDrugsIntegration(ezyVetConfig, vetDrugsApiKey);
      await integration.calculateDrugsForPatient(animal_id, commonDrugs);
      
      res.status(200).json({ status: 'processed' });
    } else {
      res.status(200).json({ status: 'ignored' });
    }
  } catch (error) {
    console.error('Webhook processing failed:', error);
    res.status(500).json({ error: 'Processing failed' });
  }
});
```

## Configuration and Deployment

### Environment Variables

```bash
# ezyVet Configuration
EZYVET_API_URL=https://api.ezyvet.com
EZYVET_CLIENT_ID=your_client_id
EZYVET_CLIENT_SECRET=your_client_secret
EZYVET_USERNAME=your_username
EZYVET_PASSWORD=your_password
EZYVET_PARTNER_ID=your_partner_id

# VetDrugs Configuration
VETDRUGS_API_KEY=vd_your_api_key
```

### Error Handling Best Practices

```javascript
class ErrorHandler {
  static handleEzyVetError(error) {
    if (error.message.includes('authentication')) {
      return 'ezyVet authentication failed. Please check your credentials.';
    } else if (error.message.includes('patient')) {
      return 'Patient not found in ezyVet. Please verify the patient ID.';
    } else {
      return `ezyVet error: ${error.message}`;
    }
  }

  static handleVetDrugsError(error) {
    if (error.message.includes('API key')) {
      return 'VetDrugs API key is invalid. Please check your configuration.';
    } else if (error.message.includes('validation')) {
      return 'Invalid patient or drug data. Please check the inputs.';
    } else {
      return `VetDrugs error: ${error.message}`;
    }
  }
}
```

## Testing

### Unit Tests

```javascript
// Jest test example
describe('EzyVetVetDrugsIntegration', () => {
  let integration;

  beforeEach(() => {
    integration = new EzyVetVetDrugsIntegration(mockEzyVetConfig, 'test_api_key');
  });

  test('should convert ezyVet patient correctly', () => {
    const ezyVetPatient = {
      weight: 55.1,
      weight_unit: 'lb',
      species_name: 'Canine'
    };

    const result = integration.convertEzyVetPatient(ezyVetPatient);
    
    expect(result.weight_kg).toBeCloseTo(25, 1);
    expect(result.species).toBe('dog');
  });

  test('should handle authentication errors', async () => {
    // Mock failed authentication
    fetch.mockResolvedValueOnce({
      ok: false,
      status: 401
    });

    await expect(integration.authenticate()).rejects.toThrow('ezyVet authentication failed');
  });
});
```

## Support and Troubleshooting

### Common Issues

1. **Authentication Failures**
   - Verify ezyVet API credentials
   - Check if API access is enabled for your account
   - Ensure correct API endpoint URL

2. **Patient Data Conversion**
   - Weight unit conversion (lb to kg)
   - Species name mapping
   - Missing required fields

3. **Rate Limiting**
   - Both ezyVet and VetDrugs APIs have rate limits
   - Implement appropriate retry logic
   - Cache frequently accessed data

### Monitoring and Logging

```javascript
class IntegrationLogger {
  static logCalculation(patientId, drugNames, success, error = null) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      patient_id: patientId,
      drugs: drugNames,
      success: success,
      error: error
    };

    // Log to your preferred logging service
    console.log('VetDrugs Calculation:', JSON.stringify(logEntry));
  }

  static logApiCall(service, endpoint, responseTime, success) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      service: service,
      endpoint: endpoint,
      response_time_ms: responseTime,
      success: success
    };

    console.log('API Call:', JSON.stringify(logEntry));
  }
}
```

## Next Steps

- [View Avimark integration](avimark.md)
- [Explore other PMI integrations](../integrations/)
- [Check JavaScript examples](../examples/javascript.md)
- [API Reference](../api-reference/)
