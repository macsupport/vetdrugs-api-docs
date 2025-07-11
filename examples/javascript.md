# JavaScript SDK and Examples

Complete JavaScript/Node.js integration examples for the VetDrugs API.

## Installation

### Browser (CDN)
```html
<script src="https://cdn.jsdelivr.net/npm/vetdrugs-api@latest/dist/vetdrugs.min.js"></script>
```

### Node.js (NPM)
```bash
npm install vetdrugs-api
```

## Basic Usage

### Initialize Client

```javascript
// Node.js
const VetDrugsAPI = require('vetdrugs-api');

// Browser
// VetDrugsAPI is available globally

const client = new VetDrugsAPI({
  apiKey: 'vd_your_api_key_here',
  baseURL: 'https://vetdrugs-calculator-api.vetcalculators.workers.dev'
});
```

### Simple Drug Calculation

```javascript
const calculateDrug = async () => {
  try {
    const result = await client.calculate({
      patient: {
        weight_kg: 25,
        species: 'dog'
      },
      drugs: [
        { drug_name: 'Cephalexin' }
      ]
    });

    console.log('Calculation result:', result);
    return result;
  } catch (error) {
    console.error('Calculation failed:', error.message);
    throw error;
  }
};
```

## Complete API Client Implementation

Here's a full-featured JavaScript client:

```javascript
class VetDrugsAPIClient {
  constructor(options = {}) {
    this.apiKey = options.apiKey;
    this.baseURL = options.baseURL || 'https://vetdrugs-calculator-api.vetcalculators.workers.dev';
    this.timeout = options.timeout || 30000;
    
    if (!this.apiKey) {
      throw new Error('API key is required');
    }
  }

  /**
   * Make authenticated API request
   */
  async request(endpoint, options = {}) {
    const url = `${this.baseURL}${endpoint}`;
    
    const config = {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
        'X-API-Key': this.apiKey,
        ...options.headers
      },
      ...options
    };

    // Add timeout
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), this.timeout);
    config.signal = controller.signal;

    try {
      const response = await fetch(url, config);
      clearTimeout(timeoutId);

      // Handle rate limiting
      if (response.status === 429) {
        const retryAfter = response.headers.get('Retry-After') || 60;
        throw new RateLimitError(`Rate limit exceeded. Retry after ${retryAfter} seconds.`, retryAfter);
      }

      // Handle other errors
      if (!response.ok) {
        const errorData = await response.json().catch(() => ({}));
        throw new APIError(errorData.message || `HTTP ${response.status}`, response.status, errorData);
      }

      return await response.json();
    } catch (error) {
      clearTimeout(timeoutId);
      
      if (error.name === 'AbortError') {
        throw new TimeoutError('Request timeout');
      }
      
      throw error;
    }
  }

  /**
   * Calculate drug dosages
   */
  async calculate(request) {
    this.validateCalculationRequest(request);
    
    return await this.request('/api/calculate', {
      method: 'POST',
      body: JSON.stringify(request)
    });
  }

  /**
   * Search drugs
   */
  async searchDrugs(searchTerm, category = null) {
    const params = new URLSearchParams();
    if (searchTerm) params.append('search', searchTerm);
    if (category) params.append('category', category);
    
    const query = params.toString();
    const endpoint = `/api/drugs${query ? `?${query}` : ''}`;
    
    return await this.request(endpoint);
  }

  /**
   * Get API health status
   */
  async healthCheck() {
    return await this.request('/api/health');
  }

  /**
   * Validate calculation request
   */
  validateCalculationRequest(request) {
    const errors = [];

    if (!request.patient) {
      errors.push('Patient information is required');
    } else {
      if (!request.patient.weight_kg || request.patient.weight_kg <= 0) {
        errors.push('Valid patient weight is required');
      }
      
      if (!['dog', 'cat'].includes(request.patient.species)) {
        errors.push('Species must be dog or cat');
      }
    }

    if (!request.drugs || !Array.isArray(request.drugs) || request.drugs.length === 0) {
      errors.push('At least one drug is required');
    } else {
      request.drugs.forEach((drug, index) => {
        if (!drug.drug_name) {
          errors.push(`Drug #${index + 1}: drug_name is required`);
        }
      });
    }

    if (errors.length > 0) {
      throw new ValidationError('Request validation failed', errors);
    }
  }
}

// Custom error classes
class APIError extends Error {
  constructor(message, statusCode, details) {
    super(message);
    this.name = 'APIError';
    this.statusCode = statusCode;
    this.details = details;
  }
}

class RateLimitError extends APIError {
  constructor(message, retryAfter) {
    super(message, 429);
    this.name = 'RateLimitError';
    this.retryAfter = retryAfter;
  }
}

class ValidationError extends Error {
  constructor(message, errors) {
    super(message);
    this.name = 'ValidationError';
    this.errors = errors;
  }
}

class TimeoutError extends Error {
  constructor(message) {
    super(message);
    this.name = 'TimeoutError';
  }
}

// Export for Node.js
if (typeof module !== 'undefined' && module.exports) {
  module.exports = VetDrugsAPIClient;
}
```

## Practical Examples

### 1. Multiple Drug Calculation with Error Handling

```javascript
const calculateMultipleDrugs = async (patientWeight, species, drugNames) => {
  const client = new VetDrugsAPIClient({ apiKey: 'vd_your_api_key' });
  
  try {
    const result = await client.calculate({
      patient: {
        weight_kg: patientWeight,
        species: species
      },
      drugs: drugNames.map(name => ({ drug_name: name }))
    });

    // Process successful calculations
    result.calculations.forEach(calc => {
      console.log(`${calc.drug_name}: ${calc.instructions}`);
      
      // Check for warnings
      if (calc.warnings && calc.warnings.length > 0) {
        console.warn(`⚠️ ${calc.drug_name} warnings:`, calc.warnings);
      }
      
      // Check dose range
      if (calc.dose_range_status !== 'within_range') {
        console.warn(`⚠️ ${calc.drug_name} dose is ${calc.dose_range_status}`);
      }
    });

    // Check for drugs not found
    if (result.warnings && result.warnings.length > 0) {
      console.warn('Some drugs were not found:', result.warnings);
    }

    return result;
  } catch (error) {
    handleAPIError(error);
    throw error;
  }
};

const handleAPIError = (error) => {
  if (error instanceof ValidationError) {
    console.error('Validation errors:', error.errors);
  } else if (error instanceof RateLimitError) {
    console.error(`Rate limited. Retry after ${error.retryAfter} seconds`);
  } else if (error instanceof TimeoutError) {
    console.error('Request timed out. Please try again.');
  } else if (error instanceof APIError) {
    console.error(`API Error (${error.statusCode}): ${error.message}`);
  } else {
    console.error('Unexpected error:', error.message);
  }
};
```

### 2. Drug Search with Autocomplete

```javascript
class DrugSearchWidget {
  constructor(inputElement, resultsElement, apiClient) {
    this.input = inputElement;
    this.results = resultsElement;
    this.client = apiClient;
    this.searchTimeout = null;
    
    this.setupEventListeners();
  }

  setupEventListeners() {
    this.input.addEventListener('input', (e) => {
      clearTimeout(this.searchTimeout);
      this.searchTimeout = setTimeout(() => {
        this.performSearch(e.target.value);
      }, 300); // Debounce searches
    });

    this.input.addEventListener('blur', () => {
      // Hide results after a short delay to allow for clicks
      setTimeout(() => this.hideResults(), 150);
    });
  }

  async performSearch(searchTerm) {
    if (searchTerm.length < 3) {
      this.hideResults();
      return;
    }

    try {
      this.showLoading();
      const response = await this.client.searchDrugs(searchTerm);
      this.displayResults(response.drugs);
    } catch (error) {
      console.error('Search failed:', error);
      this.showError('Search failed. Please try again.');
    }
  }

  displayResults(drugs) {
    this.results.innerHTML = '';
    
    if (drugs.length === 0) {
      this.showMessage('No drugs found');
      return;
    }

    drugs.forEach(drug => {
      const item = document.createElement('div');
      item.className = 'search-result-item';
      item.innerHTML = `
        <div class="drug-name">${drug.name}</div>
        <div class="drug-details">${drug.category} - ${drug.route}</div>
      `;
      
      item.addEventListener('click', () => {
        this.selectDrug(drug);
      });
      
      this.results.appendChild(item);
    });

    this.showResults();
  }

  selectDrug(drug) {
    this.input.value = drug.name;
    this.hideResults();
    
    // Trigger custom event
    this.input.dispatchEvent(new CustomEvent('drugSelected', {
      detail: { drug }
    }));
  }

  showLoading() {
    this.results.innerHTML = '<div class="loading">Searching...</div>';
    this.showResults();
  }

  showError(message) {
    this.results.innerHTML = `<div class="error">${message}</div>`;
    this.showResults();
  }

  showMessage(message) {
    this.results.innerHTML = `<div class="message">${message}</div>`;
    this.showResults();
  }

  showResults() {
    this.results.style.display = 'block';
  }

  hideResults() {
    this.results.style.display = 'none';
  }
}

// Usage
const client = new VetDrugsAPIClient({ apiKey: 'vd_your_api_key' });
const searchWidget = new DrugSearchWidget(
  document.getElementById('drug-search'),
  document.getElementById('search-results'),
  client
);
```

### 3. React Hook for VetDrugs API

```javascript
import { useState, useEffect, useCallback } from 'react';

const useVetDrugsAPI = (apiKey) => {
  const [client, setClient] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    if (apiKey) {
      setClient(new VetDrugsAPIClient({ apiKey }));
    }
  }, [apiKey]);

  const calculate = useCallback(async (request) => {
    if (!client) throw new Error('API client not initialized');
    
    setLoading(true);
    setError(null);
    
    try {
      const result = await client.calculate(request);
      return result;
    } catch (err) {
      setError(err);
      throw err;
    } finally {
      setLoading(false);
    }
  }, [client]);

  const searchDrugs = useCallback(async (searchTerm, category) => {
    if (!client) throw new Error('API client not initialized');
    
    setLoading(true);
    setError(null);
    
    try {
      const result = await client.searchDrugs(searchTerm, category);
      return result;
    } catch (err) {
      setError(err);
      throw err;
    } finally {
      setLoading(false);
    }
  }, [client]);

  return {
    client,
    loading,
    error,
    calculate,
    searchDrugs
  };
};

// React component example
const DrugCalculator = ({ apiKey }) => {
  const { calculate, loading, error } = useVetDrugsAPI(apiKey);
  const [result, setResult] = useState(null);

  const handleCalculate = async () => {
    try {
      const calculation = await calculate({
        patient: { weight_kg: 25, species: 'dog' },
        drugs: [{ drug_name: 'Cephalexin' }]
      });
      setResult(calculation);
    } catch (err) {
      console.error('Calculation failed:', err);
    }
  };

  return (
    <div>
      <button onClick={handleCalculate} disabled={loading}>
        {loading ? 'Calculating...' : 'Calculate'}
      </button>
      
      {error && <div className="error">Error: {error.message}</div>}
      
      {result && (
        <div className="results">
          {result.calculations.map(calc => (
            <div key={calc.drug_name}>
              <h3>{calc.drug_name}</h3>
              <p>{calc.instructions}</p>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};
```

### 4. Caching Implementation

```javascript
class CachedVetDrugsClient extends VetDrugsAPIClient {
  constructor(options = {}) {
    super(options);
    this.cache = new Map();
    this.cacheTTL = options.cacheTTL || 300000; // 5 minutes default
  }

  generateCacheKey(request) {
    return JSON.stringify(request, Object.keys(request).sort());
  }

  getCached(key) {
    const cached = this.cache.get(key);
    if (cached && Date.now() - cached.timestamp < this.cacheTTL) {
      return cached.data;
    }
    this.cache.delete(key);
    return null;
  }

  setCached(key, data) {
    this.cache.set(key, {
      data,
      timestamp: Date.now()
    });
  }

  async calculate(request) {
    const cacheKey = this.generateCacheKey(request);
    const cached = this.getCached(cacheKey);
    
    if (cached) {
      console.log('Cache hit for calculation');
      return cached;
    }

    const result = await super.calculate(request);
    this.setCached(cacheKey, result);
    return result;
  }

  clearCache() {
    this.cache.clear();
  }
}
```

### 5. Retry Logic with Exponential Backoff

```javascript
class RobustVetDrugsClient extends VetDrugsAPIClient {
  async request(endpoint, options = {}, retryCount = 0) {
    const maxRetries = options.maxRetries || 3;
    
    try {
      return await super.request(endpoint, options);
    } catch (error) {
      if (retryCount < maxRetries) {
        if (error instanceof RateLimitError) {
          // Wait for rate limit reset
          const waitTime = (error.retryAfter || 60) * 1000;
          console.log(`Rate limited. Waiting ${waitTime/1000} seconds...`);
          await this.delay(waitTime);
        } else if (error instanceof APIError && error.statusCode >= 500) {
          // Server error - exponential backoff
          const waitTime = Math.pow(2, retryCount) * 1000;
          console.log(`Server error. Retrying in ${waitTime/1000} seconds...`);
          await this.delay(waitTime);
        } else {
          // Don't retry client errors
          throw error;
        }
        
        return this.request(endpoint, options, retryCount + 1);
      }
      
      throw error;
    }
  }

  delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

## Testing

### Unit Tests (Jest)

```javascript
const VetDrugsAPIClient = require('./vetdrugs-api-client');

describe('VetDrugsAPIClient', () => {
  let client;

  beforeEach(() => {
    client = new VetDrugsAPIClient({ apiKey: 'test_key' });
  });

  test('should validate calculation request', () => {
    expect(() => {
      client.validateCalculationRequest({});
    }).toThrow(ValidationError);
  });

  test('should calculate drug dosage', async () => {
    // Mock fetch
    global.fetch = jest.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve({
        calculations: [{ drug_name: 'Cephalexin', instructions: 'Give 2ml PO BID' }]
      })
    });

    const result = await client.calculate({
      patient: { weight_kg: 25, species: 'dog' },
      drugs: [{ drug_name: 'Cephalexin' }]
    });

    expect(result.calculations).toHaveLength(1);
    expect(result.calculations[0].drug_name).toBe('Cephalexin');
  });
});
```

## Browser Implementation

For browser-based PMI systems:

```html
<!DOCTYPE html>
<html>
<head>
    <title>VetDrugs Calculator</title>
    <style>
        .calculator { max-width: 600px; margin: 0 auto; padding: 20px; }
        .form-group { margin-bottom: 15px; }
        .result { background: #f0f0f0; padding: 15px; margin-top: 20px; }
        .warning { color: #ff6600; }
        .error { color: #cc0000; }
    </style>
</head>
<body>
    <div class="calculator">
        <h1>VetDrugs Calculator</h1>
        
        <form id="calculatorForm">
            <div class="form-group">
                <label>Patient Weight (kg):</label>
                <input type="number" id="weight" step="0.1" required>
            </div>
            
            <div class="form-group">
                <label>Species:</label>
                <select id="species" required>
                    <option value="">Select species</option>
                    <option value="dog">Dog</option>
                    <option value="cat">Cat</option>
                </select>
            </div>
            
            <div class="form-group">
                <label>Drug Name:</label>
                <input type="text" id="drugName" required>
            </div>
            
            <button type="submit">Calculate</button>
        </form>
        
        <div id="results"></div>
    </div>

    <script>
        // Include the VetDrugsAPIClient here
        // ... (client code from above)

        const client = new VetDrugsAPIClient({ 
            apiKey: 'vd_your_api_key_here' 
        });

        document.getElementById('calculatorForm').addEventListener('submit', async (e) => {
            e.preventDefault();
            
            const weight = parseFloat(document.getElementById('weight').value);
            const species = document.getElementById('species').value;
            const drugName = document.getElementById('drugName').value;
            
            try {
                const result = await client.calculate({
                    patient: { weight_kg: weight, species },
                    drugs: [{ drug_name: drugName }]
                });
                
                displayResults(result);
            } catch (error) {
                displayError(error);
            }
        });

        function displayResults(result) {
            const resultsDiv = document.getElementById('results');
            
            if (result.calculations.length === 0) {
                resultsDiv.innerHTML = '<div class="error">No calculations returned</div>';
                return;
            }
            
            const calc = result.calculations[0];
            resultsDiv.innerHTML = `
                <div class="result">
                    <h3>${calc.drug_name}</h3>
                    <p><strong>Instructions:</strong> ${calc.instructions}</p>
                    <p><strong>Dose:</strong> ${calc.dose_per_kg} ${calc.dosage_unit}</p>
                    <p><strong>Total dose:</strong> ${calc.total_dose} ${calc.dose_unit}</p>
                    ${calc.warnings.length > 0 ? 
                      `<div class="warning">⚠️ ${calc.warnings.join(', ')}</div>` : ''}
                </div>
            `;
        }

        function displayError(error) {
            const resultsDiv = document.getElementById('results');
            resultsDiv.innerHTML = `<div class="error">Error: ${error.message}</div>`;
        }
    </script>
</body>
</html>
```

## Next Steps

- [View C# examples](csharp.md)
- [Explore PMI integrations](../integrations/)
- [API Reference](../api-reference/)