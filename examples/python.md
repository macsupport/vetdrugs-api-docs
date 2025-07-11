# Python SDK and Examples

Complete Python integration examples for the VetDrugs API.

## Installation

### Using pip
```bash
pip install requests
pip install vetdrugs-api  # When available
```

### Requirements.txt
```text
requests>=2.25.0
python-dateutil>=2.8.0
```

## Core SDK Implementation

### VetDrugsApiClient Class

```python
import requests
import json
from typing import List, Dict, Optional, Any
from datetime import datetime
import time
from urllib.parse import urlencode

class VetDrugsApiClient:
    """Python client for VetDrugs API"""
    
    def __init__(self, api_key: str, base_url: str = "https://vetdrugs-calculator-api.vetcalculators.workers.dev"):
        """
        Initialize the VetDrugs API client
        
        Args:
            api_key: Your VetDrugs API key
            base_url: API base URL (default: production URL)
        """
        if not api_key:
            raise ValueError("API key is required")
        
        self.api_key = api_key
        self.base_url = base_url.rstrip('/')
        self.timeout = 30
        
        self.session = requests.Session()
        self.session.headers.update({
            'X-API-Key': api_key,
            'Content-Type': 'application/json',
            'User-Agent': 'VetDrugs-Python-SDK/1.0'
        })

    def calculate_drugs(self, patient: Dict[str, Any], drugs: List[Dict[str, Any]]) -> Dict[str, Any]:
        """
        Calculate drug dosages for a patient
        
        Args:
            patient: Patient information (weight_kg, species)
            drugs: List of drugs to calculate
            
        Returns:
            Calculation response with dosages and instructions
        """
        request_data = {
            'patient': patient,
            'drugs': drugs
        }
        
        self._validate_calculation_request(request_data)
        
        try:
            response = self.session.post(
                f"{self.base_url}/api/calculate",
                json=request_data,
                timeout=self.timeout
            )
            
            if response.status_code == 429:
                retry_after = int(response.headers.get('Retry-After', 60))
                raise RateLimitError(f"Rate limit exceeded. Retry after {retry_after} seconds", retry_after)
            
            response.raise_for_status()
            return response.json()
            
        except requests.exceptions.RequestException as e:
            raise VetDrugsApiError(f"Request failed: {str(e)}")

    def search_drugs(self, search_term: Optional[str] = None, category: Optional[str] = None) -> Dict[str, Any]:
        """
        Search for drugs in the database
        
        Args:
            search_term: Optional search term for drug names
            category: Optional category filter
            
        Returns:
            Search response with matching drugs
        """
        params = {}
        if search_term:
            params['search'] = search_term
        if category:
            params['category'] = category
        
        url = f"{self.base_url}/api/drugs"
        if params:
            url += f"?{urlencode(params)}"
        
        try:
            response = self.session.get(url, timeout=self.timeout)
            response.raise_for_status()
            return response.json()
            
        except requests.exceptions.RequestException as e:
            raise VetDrugsApiError(f"Search failed: {str(e)}")

    def get_drug_info(self, drug_name: str) -> Dict[str, Any]:
        """
        Get detailed information about a specific drug
        
        Args:
            drug_name: Name of the drug to lookup
            
        Returns:
            Detailed drug information
        """
        if not drug_name:
            raise ValueError("Drug name is required")
        
        try:
            response = self.session.get(
                f"{self.base_url}/api/drug-info/{requests.utils.quote(drug_name)}",
                timeout=self.timeout
            )
            response.raise_for_status()
            return response.json()
            
        except requests.exceptions.RequestException as e:
            raise VetDrugsApiError(f"Drug info request failed: {str(e)}")

    def health_check(self) -> Dict[str, Any]:
        """
        Check API health status
        
        Returns:
            Health status information
        """
        try:
            response = self.session.get(f"{self.base_url}/api/health", timeout=self.timeout)
            response.raise_for_status()
            return response.json()
            
        except requests.exceptions.RequestException as e:
            raise VetDrugsApiError(f"Health check failed: {str(e)}")

    def _validate_calculation_request(self, request_data: Dict[str, Any]) -> None:
        """Validate calculation request data"""
        errors = []
        
        # Validate patient
        patient = request_data.get('patient')
        if not patient:
            errors.append("Patient information is required")
        else:
            weight = patient.get('weight_kg')
            if not weight or weight <= 0:
                errors.append("Valid patient weight is required")
            
            species = patient.get('species', '').lower()
            if species not in ['dog', 'cat']:
                errors.append("Species must be 'dog' or 'cat'")
        
        # Validate drugs
        drugs = request_data.get('drugs')
        if not drugs or len(drugs) == 0:
            errors.append("At least one drug is required")
        else:
            for i, drug in enumerate(drugs):
                if not drug.get('drug_name'):
                    errors.append(f"Drug #{i + 1}: drug_name is required")
        
        if errors:
            raise VetDrugsValidationError("Request validation failed", errors)

    def close(self):
        """Close the session"""
        self.session.close()

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.close()


# Custom Exceptions
class VetDrugsApiError(Exception):
    """Base exception for VetDrugs API errors"""
    def __init__(self, message: str, status_code: Optional[int] = None):
        super().__init__(message)
        self.status_code = status_code


class VetDrugsValidationError(VetDrugsApiError):
    """Exception for validation errors"""
    def __init__(self, message: str, errors: List[str]):
        super().__init__(message)
        self.errors = errors


class RateLimitError(VetDrugsApiError):
    """Exception for rate limit errors"""
    def __init__(self, message: str, retry_after: int):
        super().__init__(message, 429)
        self.retry_after = retry_after
```

### Data Models and Helpers

```python
from dataclasses import dataclass
from typing import List, Optional, Dict, Any
from datetime import datetime

@dataclass
class Patient:
    """Patient information for drug calculations"""
    weight_kg: float
    species: str
    
    @property
    def weight_lbs(self) -> float:
        """Convert weight to pounds"""
        return round(self.weight_kg * 2.20462, 2)
    
    @weight_lbs.setter
    def weight_lbs(self, pounds: float):
        """Set weight from pounds"""
        self.weight_kg = round(pounds / 2.20462, 2)
    
    @property
    def is_dog(self) -> bool:
        return self.species.lower() == 'dog'
    
    @property
    def is_cat(self) -> bool:
        return self.species.lower() == 'cat'
    
    def to_dict(self) -> Dict[str, Any]:
        return {
            'weight_kg': self.weight_kg,
            'species': self.species
        }

@dataclass
class DrugRequest:
    """Drug request for calculations"""
    drug_name: str
    concentration: Optional[float] = None
    route: Optional[str] = None
    frequency: Optional[str] = None
    custom_dose: Optional[float] = None
    
    def to_dict(self) -> Dict[str, Any]:
        result = {'drug_name': self.drug_name}
        if self.concentration is not None:
            result['concentration'] = self.concentration
        if self.route:
            result['route'] = self.route
        if self.frequency:
            result['frequency'] = self.frequency
        if self.custom_dose is not None:
            result['custom_dose'] = self.custom_dose
        return result

@dataclass
class DrugCalculation:
    """Drug calculation result"""
    drug_name: str
    dose_per_kg: float
    total_dose: float
    dose_unit: str
    volume: float
    volume_unit: str
    concentration: float
    concentration_unit: str
    route: str
    frequency: str
    instructions: str
    dose_range_status: str
    warnings: List[str]
    
    @property
    def is_within_range(self) -> bool:
        return self.dose_range_status == 'within_range'
    
    @property
    def has_warnings(self) -> bool:
        return len(self.warnings) > 0
    
    @property
    def formatted_instruction(self) -> str:
        return f"{self.volume:.2f} {self.volume_unit} {self.route} {self.frequency}"
    
    @classmethod
    def from_dict(cls, data: Dict[str, Any]) -> 'DrugCalculation':
        return cls(
            drug_name=data['drug_name'],
            dose_per_kg=data['dose_per_kg'],
            total_dose=data['total_dose'],
            dose_unit=data['dose_unit'],
            volume=data['volume'],
            volume_unit=data['volume_unit'],
            concentration=data['concentration'],
            concentration_unit=data['concentration_unit'],
            route=data['route'],
            frequency=data['frequency'],
            instructions=data['instructions'],
            dose_range_status=data['dose_range_status'],
            warnings=data.get('warnings', [])
        )

class VetDrugsHelpers:
    """Helper functions for common operations"""
    
    @staticmethod
    def pounds_to_kg(pounds: float) -> float:
        """Convert pounds to kilograms"""
        return round(pounds * 0.453592, 2)
    
    @staticmethod
    def kg_to_pounds(kg: float) -> float:
        """Convert kilograms to pounds"""
        return round(kg * 2.20462, 2)
    
    @staticmethod
    def is_valid_weight(weight_kg: float) -> bool:
        """Check if weight is within reasonable range"""
        return 0.1 <= weight_kg <= 1000
    
    @staticmethod
    def get_status_message(dose_range_status: str) -> str:
        """Get user-friendly status message"""
        status_messages = {
            'within_range': '✓ Normal dose',
            'below_range': '⚠️ Below normal range',
            'above_range': '⚠️ Above normal range',
            'contraindicated': '❌ Contraindicated'
        }
        return status_messages.get(dose_range_status, 'Unknown status')
    
    @staticmethod
    def format_dosage_instructions(calc: DrugCalculation) -> str:
        """Format dosage instructions for display"""
        return f"Give {calc.volume:.2f} {calc.volume_unit} {calc.route} {calc.frequency}"
```

## Practical Examples

### 1. Basic Drug Calculation

```python
def calculate_basic_drugs():
    """Example: Calculate drugs for a 25kg dog"""
    
    # Initialize client
    client = VetDrugsApiClient('vd_your_api_key_here')
    
    try:
        # Create patient
        patient = Patient(weight_kg=25.0, species='dog')
        
        # Create drug requests
        drugs = [
            DrugRequest('Cephalexin'),
            DrugRequest('Carprofen')
        ]
        
        # Calculate
        response = client.calculate_drugs(
            patient=patient.to_dict(),
            drugs=[drug.to_dict() for drug in drugs]
        )
        
        # Process results
        print(f"Calculations for {patient.weight_kg}kg {patient.species}:")
        
        for calc_data in response['calculations']:
            calc = DrugCalculation.from_dict(calc_data)
            
            print(f"\n{calc.drug_name}:")
            print(f"  Instructions: {calc.instructions}")
            print(f"  Dose: {calc.dose_per_kg} {calc.dose_unit}/kg")
            print(f"  Total: {calc.total_dose} {calc.dose_unit}")
            print(f"  Volume: {calc.volume:.2f} {calc.volume_unit}")
            print(f"  Status: {VetDrugsHelpers.get_status_message(calc.dose_range_status)}")
            
            if calc.has_warnings:
                print(f"  ⚠️ Warnings: {', '.join(calc.warnings)}")
        
        # Check for overall warnings
        if response.get('warnings'):
            print(f"\n⚠️ Overall warnings: {', '.join(response['warnings'])}")
            
    except VetDrugsValidationError as e:
        print(f"Validation errors: {', '.join(e.errors)}")
    except VetDrugsApiError as e:
        print(f"API error: {e}")
    finally:
        client.close()

if __name__ == "__main__":
    calculate_basic_drugs()
```

### 2. Drug Search with Filtering

```python
def search_drugs_example():
    """Example: Search for antibiotics and display results"""
    
    with VetDrugsApiClient('vd_your_api_key_here') as client:
        try:
            # Search for antibiotics
            response = client.search_drugs(category='antibiotic')
            
            print(f"Found {response['total_count']} antibiotics:")
            
            for drug in response['drugs']:
                print(f"\n{drug['name']} ({drug['brand_name'] if 'brand_name' in drug else 'Generic'})")
                print(f"  Category: {drug['category']}")
                print(f"  Default dose: {drug['default_dose']} {drug['dosage_unit']}")
                print(f"  Route: {drug['route']}")
                print(f"  Species: {', '.join(drug['species'])}")
                
                if drug.get('has_feline_dose'):
                    print("  ✓ Has specific feline dosing")
                    
        except VetDrugsApiError as e:
            print(f"Search failed: {e}")

def search_with_autocomplete():
    """Example: Implement drug name autocomplete"""
    
    def get_drug_suggestions(search_term: str, min_length: int = 3) -> List[str]:
        if len(search_term) < min_length:
            return []
        
        with VetDrugsApiClient('vd_your_api_key_here') as client:
            try:
                response = client.search_drugs(search_term=search_term)
                return [drug['name'] for drug in response['drugs'][:10]]  # Top 10 suggestions
            except VetDrugsApiError:
                return []
    
    # Example usage
    user_input = "ceph"
    suggestions = get_drug_suggestions(user_input)
    print(f"Suggestions for '{user_input}': {suggestions}")

if __name__ == "__main__":
    search_drugs_example()
    search_with_autocomplete()
```

### 3. Flask Web Application Integration

```python
from flask import Flask, request, jsonify
from flask_cors import CORS

app = Flask(__name__)
CORS(app)

# Initialize API client (in production, use proper configuration)
api_client = VetDrugsApiClient('vd_your_api_key_here')

@app.route('/api/calculate', methods=['POST'])
def calculate_drugs():
    """Calculate drug dosages endpoint"""
    try:
        data = request.get_json()
        
        # Validate request
        if not data or 'patient' not in data or 'drugs' not in data:
            return jsonify({'error': 'Invalid request format'}), 400
        
        # Calculate using VetDrugs API
        response = api_client.calculate_drugs(
            patient=data['patient'],
            drugs=data['drugs']
        )
        
        return jsonify(response)
        
    except VetDrugsValidationError as e:
        return jsonify({
            'error': 'validation_failed',
            'message': 'Request validation failed',
            'errors': e.errors
        }), 400
        
    except VetDrugsApiError as e:
        return jsonify({
            'error': 'api_error',
            'message': str(e)
        }), 500

@app.route('/api/drugs/search', methods=['GET'])
def search_drugs():
    """Drug search endpoint"""
    try:
        search_term = request.args.get('q')
        category = request.args.get('category')
        
        response = api_client.search_drugs(
            search_term=search_term,
            category=category
        )
        
        return jsonify(response)
        
    except VetDrugsApiError as e:
        return jsonify({
            'error': 'search_failed',
            'message': str(e)
        }), 500

@app.route('/api/drugs/<drug_name>', methods=['GET'])
def get_drug_info(drug_name):
    """Get drug information endpoint"""
    try:
        response = api_client.get_drug_info(drug_name)
        return jsonify(response)
        
    except VetDrugsApiError as e:
        return jsonify({
            'error': 'drug_not_found',
            'message': str(e)
        }), 404

@app.route('/health', methods=['GET'])
def health_check():
    """Health check endpoint"""
    try:
        api_health = api_client.health_check()
        return jsonify({
            'status': 'healthy',
            'api_status': api_health
        })
    except Exception as e:
        return jsonify({
            'status': 'unhealthy',
            'error': str(e)
        }), 503

if __name__ == '__main__':
    app.run(debug=True)
```

### 4. Django Integration

```python
# models.py
from django.db import models
from django.core.validators import MinValueValidator, MaxValueValidator

class Patient(models.Model):
    SPECIES_CHOICES = [
        ('dog', 'Dog'),
        ('cat', 'Cat'),
    ]
    
    name = models.CharField(max_length=100)
    species = models.CharField(max_length=10, choices=SPECIES_CHOICES)
    weight_kg = models.FloatField(
        validators=[MinValueValidator(0.1), MaxValueValidator(1000)]
    )
    owner = models.CharField(max_length=100)
    created_at = models.DateTimeField(auto_now_add=True)
    
    def __str__(self):
        return f"{self.name} ({self.species}, {self.weight_kg}kg)"

class DrugCalculationHistory(models.Model):
    patient = models.ForeignKey(Patient, on_delete=models.CASCADE)
    drug_name = models.CharField(max_length=100)
    dose_calculated = models.FloatField()
    volume = models.FloatField()
    instructions = models.TextField()
    warnings = models.TextField(blank=True)
    calculated_at = models.DateTimeField(auto_now_add=True)
    calculated_by = models.CharField(max_length=100)

# services.py
from .models import Patient, DrugCalculationHistory

class VetDrugsService:
    def __init__(self):
        self.client = VetDrugsApiClient(settings.VETDRUGS_API_KEY)
    
    def calculate_for_patient(self, patient_id: int, drug_names: List[str], calculated_by: str) -> Dict:
        """Calculate drugs for a patient and save to history"""
        try:
            patient = Patient.objects.get(id=patient_id)
            
            # Prepare API request
            patient_data = {
                'weight_kg': float(patient.weight_kg),
                'species': patient.species
            }
            
            drugs = [{'drug_name': name} for name in drug_names]
            
            # Calculate via API
            response = self.client.calculate_drugs(patient_data, drugs)
            
            # Save to history
            for calc in response['calculations']:
                DrugCalculationHistory.objects.create(
                    patient=patient,
                    drug_name=calc['drug_name'],
                    dose_calculated=calc['total_dose'],
                    volume=calc['volume'],
                    instructions=calc['instructions'],
                    warnings='; '.join(calc.get('warnings', [])),
                    calculated_by=calculated_by
                )
            
            return response
            
        except Patient.DoesNotExist:
            raise ValueError("Patient not found")
        except VetDrugsApiError as e:
            raise Exception(f"Drug calculation failed: {e}")

# views.py
from django.shortcuts import render, get_object_or_404
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
import json

@csrf_exempt
def calculate_drugs_view(request):
    if request.method != 'POST':
        return JsonResponse({'error': 'Method not allowed'}, status=405)
    
    try:
        data = json.loads(request.body)
        patient_id = data.get('patient_id')
        drug_names = data.get('drugs', [])
        
        service = VetDrugsService()
        result = service.calculate_for_patient(
            patient_id=patient_id,
            drug_names=drug_names,
            calculated_by=request.user.username
        )
        
        return JsonResponse(result)
        
    except Exception as e:
        return JsonResponse({'error': str(e)}, status=400)

def patient_history_view(request, patient_id):
    patient = get_object_or_404(Patient, id=patient_id)
    history = DrugCalculationHistory.objects.filter(patient=patient).order_by('-calculated_at')
    
    context = {
        'patient': patient,
        'history': history
    }
    return render(request, 'patient_history.html', context)
```

### 5. Async Support with asyncio

```python
import asyncio
import aiohttp
from typing import List, Dict, Any

class AsyncVetDrugsApiClient:
    """Async version of VetDrugs API client"""
    
    def __init__(self, api_key: str, base_url: str = "https://vetdrugs-calculator-api.vetcalculators.workers.dev"):
        self.api_key = api_key
        self.base_url = base_url.rstrip('/')
        self.headers = {
            'X-API-Key': api_key,
            'Content-Type': 'application/json',
            'User-Agent': 'VetDrugs-Python-Async-SDK/1.0'
        }
    
    async def calculate_drugs(self, patient: Dict[str, Any], drugs: List[Dict[str, Any]]) -> Dict[str, Any]:
        """Async drug calculation"""
        async with aiohttp.ClientSession(headers=self.headers) as session:
            async with session.post(
                f"{self.base_url}/api/calculate",
                json={'patient': patient, 'drugs': drugs}
            ) as response:
                response.raise_for_status()
                return await response.json()
    
    async def search_drugs(self, search_term: str = None, category: str = None) -> Dict[str, Any]:
        """Async drug search"""
        params = {}
        if search_term:
            params['search'] = search_term
        if category:
            params['category'] = category
        
        async with aiohttp.ClientSession(headers=self.headers) as session:
            async with session.get(f"{self.base_url}/api/drugs", params=params) as response:
                response.raise_for_status()
                return await response.json()

async def batch_calculations():
    """Example: Calculate drugs for multiple patients concurrently"""
    client = AsyncVetDrugsApiClient('vd_your_api_key_here')
    
    patients = [
        {'weight_kg': 25, 'species': 'dog'},
        {'weight_kg': 4.5, 'species': 'cat'},
        {'weight_kg': 30, 'species': 'dog'}
    ]
    
    drugs = [{'drug_name': 'Cephalexin'}]
    
    # Calculate for all patients concurrently
    tasks = [
        client.calculate_drugs(patient, drugs)
        for patient in patients
    ]
    
    results = await asyncio.gather(*tasks, return_exceptions=True)
    
    for i, result in enumerate(results):
        if isinstance(result, Exception):
            print(f"Patient {i+1} calculation failed: {result}")
        else:
            print(f"Patient {i+1} calculation successful")
            print(f"  Instructions: {result['calculations'][0]['instructions']}")

# Run async example
if __name__ == "__main__":
    asyncio.run(batch_calculations())
```

### 6. Caching and Performance Optimization

```python
import time
from functools import wraps
from typing import Any, Callable

class CachedVetDrugsClient(VetDrugsApiClient):
    """VetDrugs client with built-in caching"""
    
    def __init__(self, api_key: str, cache_ttl: int = 300):  # 5 minute default cache
        super().__init__(api_key)
        self.cache = {}
        self.cache_ttl = cache_ttl
    
    def _cache_key(self, *args, **kwargs) -> str:
        """Generate cache key from arguments"""
        import hashlib
        key_data = f"{args}{kwargs}"
        return hashlib.md5(key_data.encode()).hexdigest()
    
    def _get_cached(self, key: str) -> Any:
        """Get cached value if not expired"""
        if key in self.cache:
            value, timestamp = self.cache[key]
            if time.time() - timestamp < self.cache_ttl:
                return value
            else:
                del self.cache[key]
        return None
    
    def _set_cached(self, key: str, value: Any) -> None:
        """Set cached value with timestamp"""
        self.cache[key] = (value, time.time())
    
    def search_drugs(self, search_term: str = None, category: str = None) -> Dict[str, Any]:
        """Cached drug search"""
        cache_key = self._cache_key('search', search_term, category)
        cached = self._get_cached(cache_key)
        
        if cached is not None:
            return cached
        
        result = super().search_drugs(search_term, category)
        self._set_cached(cache_key, result)
        return result
    
    def get_drug_info(self, drug_name: str) -> Dict[str, Any]:
        """Cached drug info lookup"""
        cache_key = self._cache_key('drug_info', drug_name)
        cached = self._get_cached(cache_key)
        
        if cached is not None:
            return cached
        
        result = super().get_drug_info(drug_name)
        self._set_cached(cache_key, result)
        return result

def retry_on_failure(max_retries: int = 3, delay: float = 1.0):
    """Decorator to retry API calls on failure"""
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except (VetDrugsApiError, requests.exceptions.RequestException) as e:
                    last_exception = e
                    if attempt < max_retries - 1:  # Don't sleep on last attempt
                        time.sleep(delay * (2 ** attempt))  # Exponential backoff
            
            raise last_exception
        return wrapper
    return decorator

class RobustVetDrugsClient(VetDrugsApiClient):
    """VetDrugs client with retry logic"""
    
    @retry_on_failure(max_retries=3, delay=1.0)
    def calculate_drugs(self, patient: Dict[str, Any], drugs: List[Dict[str, Any]]) -> Dict[str, Any]:
        return super().calculate_drugs(patient, drugs)
    
    @retry_on_failure(max_retries=2, delay=0.5)
    def search_drugs(self, search_term: str = None, category: str = None) -> Dict[str, Any]:
        return super().search_drugs(search_term, category)
```

## Configuration and Best Practices

### Environment Configuration

```python
import os
from dataclasses import dataclass

@dataclass
class VetDrugsConfig:
    api_key: str
    base_url: str = "https://vetdrugs-calculator-api.vetcalculators.workers.dev"
    timeout: int = 30
    max_retries: int = 3
    cache_ttl: int = 300
    
    @classmethod
    def from_env(cls) -> 'VetDrugsConfig':
        """Load configuration from environment variables"""
        api_key = os.getenv('VETDRUGS_API_KEY')
        if not api_key:
            raise ValueError("VETDRUGS_API_KEY environment variable is required")
        
        return cls(
            api_key=api_key,
            base_url=os.getenv('VETDRUGS_API_URL', cls.__dataclass_fields__['base_url'].default),
            timeout=int(os.getenv('VETDRUGS_TIMEOUT', cls.__dataclass_fields__['timeout'].default)),
            max_retries=int(os.getenv('VETDRUGS_MAX_RETRIES', cls.__dataclass_fields__['max_retries'].default)),
            cache_ttl=int(os.getenv('VETDRUGS_CACHE_TTL', cls.__dataclass_fields__['cache_ttl'].default))
        )

# Usage
config = VetDrugsConfig.from_env()
client = VetDrugsApiClient(config.api_key, config.base_url)
```

### Logging Integration

```python
import logging

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class LoggingVetDrugsClient(VetDrugsApiClient):
    """VetDrugs client with comprehensive logging"""
    
    def calculate_drugs(self, patient: Dict[str, Any], drugs: List[Dict[str, Any]]) -> Dict[str, Any]:
        drug_names = [drug['drug_name'] for drug in drugs]
        logger.info(f"Calculating drugs for {patient['species']} ({patient['weight_kg']}kg): {drug_names}")
        
        try:
            result = super().calculate_drugs(patient, drugs)
            logger.info(f"Successfully calculated {len(result['calculations'])} drugs")
            return result
        except Exception as e:
            logger.error(f"Drug calculation failed: {e}")
            raise
    
    def search_drugs(self, search_term: str = None, category: str = None) -> Dict[str, Any]:
        logger.info(f"Searching drugs: term='{search_term}', category='{category}'")
        
        try:
            result = super().search_drugs(search_term, category)
            logger.info(f"Found {result['total_count']} drugs")
            return result
        except Exception as e:
            logger.error(f"Drug search failed: {e}")
            raise
```

### Testing

```python
import unittest
from unittest.mock import Mock, patch

class TestVetDrugsApiClient(unittest.TestCase):
    def setUp(self):
        self.client = VetDrugsApiClient('test_api_key')
    
    def test_validate_calculation_request_valid(self):
        """Test valid request passes validation"""
        request = {
            'patient': {'weight_kg': 25, 'species': 'dog'},
            'drugs': [{'drug_name': 'Cephalexin'}]
        }
        
        # Should not raise an exception
        self.client._validate_calculation_request(request)
    
    def test_validate_calculation_request_invalid_weight(self):
        """Test invalid weight raises validation error"""
        request = {
            'patient': {'weight_kg': 0, 'species': 'dog'},
            'drugs': [{'drug_name': 'Cephalexin'}]
        }
        
        with self.assertRaises(VetDrugsValidationError):
            self.client._validate_calculation_request(request)
    
    @patch('requests.Session.post')
    def test_calculate_drugs_success(self, mock_post):
        """Test successful drug calculation"""
        # Mock response
        mock_response = Mock()
        mock_response.status_code = 200
        mock_response.json.return_value = {
            'calculations': [
                {
                    'drug_name': 'Cephalexin',
                    'instructions': 'Give 2ml PO BID'
                }
            ]
        }
        mock_post.return_value = mock_response
        
        result = self.client.calculate_drugs(
            patient={'weight_kg': 25, 'species': 'dog'},
            drugs=[{'drug_name': 'Cephalexin'}]
        )
        
        self.assertEqual(len(result['calculations']), 1)
        self.assertEqual(result['calculations'][0]['drug_name'], 'Cephalexin')

if __name__ == '__main__':
    unittest.main()
```

## Next Steps

- [View PHP examples](php.md)
- [Explore PMI integrations](../integrations/)
- [API Reference](../api-reference/)
