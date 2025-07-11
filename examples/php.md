# PHP SDK and Examples

Complete PHP integration examples for the VetDrugs API.

## Installation

### Using Composer

```bash
composer require guzzlehttp/guzzle
```

### Manual Installation

Download and include the required files:
- cURL support (usually included in PHP)
- JSON support (usually included in PHP)

## Core SDK Implementation

### VetDrugsApiClient Class

```php
<?php

class VetDrugsApiClient
{
    private $apiKey;
    private $baseUrl;
    private $timeout;

    public function __construct($apiKey, $baseUrl = 'https://vetdrugs-calculator-api.vetcalculators.workers.dev')
    {
        if (empty($apiKey)) {
            throw new InvalidArgumentException('API key is required');
        }

        $this->apiKey = $apiKey;
        $this->baseUrl = rtrim($baseUrl, '/');
        $this->timeout = 30;
    }

    /**
     * Calculate drug dosages for a patient
     *
     * @param array $patient Patient information (weight_kg, species)
     * @param array $drugs Array of drugs to calculate
     * @return array Calculation response
     * @throws VetDrugsApiException
     */
    public function calculateDrugs($patient, $drugs)
    {
        $this->validateCalculationRequest($patient, $drugs);

        $requestData = [
            'patient' => $patient,
            'drugs' => $drugs
        ];

        return $this->makeRequest('POST', '/api/calculate', $requestData);
    }

    /**
     * Search for drugs in the database
     *
     * @param string|null $searchTerm Optional search term
     * @param string|null $category Optional category filter
     * @return array Search response
     * @throws VetDrugsApiException
     */
    public function searchDrugs($searchTerm = null, $category = null)
    {
        $params = [];
        if (!empty($searchTerm)) {
            $params['search'] = $searchTerm;
        }
        if (!empty($category)) {
            $params['category'] = $category;
        }

        $endpoint = '/api/drugs';
        if (!empty($params)) {
            $endpoint .= '?' . http_build_query($params);
        }

        return $this->makeRequest('GET', $endpoint);
    }

    /**
     * Get detailed information about a specific drug
     *
     * @param string $drugName Name of the drug
     * @return array Drug information response
     * @throws VetDrugsApiException
     */
    public function getDrugInfo($drugName)
    {
        if (empty($drugName)) {
            throw new InvalidArgumentException('Drug name is required');
        }

        $endpoint = '/api/drug-info/' . urlencode($drugName);
        return $this->makeRequest('GET', $endpoint);
    }

    /**
     * Check API health status
     *
     * @return array Health status response
     * @throws VetDrugsApiException
     */
    public function healthCheck()
    {
        return $this->makeRequest('GET', '/api/health');
    }

    /**
     * Make HTTP request to the API
     *
     * @param string $method HTTP method
     * @param string $endpoint API endpoint
     * @param array|null $data Request data
     * @return array Response data
     * @throws VetDrugsApiException
     */
    private function makeRequest($method, $endpoint, $data = null)
    {
        $url = $this->baseUrl . $endpoint;

        $headers = [
            'X-API-Key: ' . $this->apiKey,
            'Content-Type: application/json',
            'User-Agent: VetDrugs-PHP-SDK/1.0'
        ];

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_HTTPHEADER => $headers,
            CURLOPT_CUSTOMREQUEST => $method
        ]);

        if ($data !== null) {
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($response === false) {
            throw new VetDrugsApiException("cURL error: $error");
        }

        if ($httpCode === 429) {
            throw new VetDrugsRateLimitException('Rate limit exceeded', $httpCode);
        }

        if ($httpCode >= 400) {
            $errorData = json_decode($response, true);
            $message = isset($errorData['message']) ? $errorData['message'] : "HTTP $httpCode error";
            throw new VetDrugsApiException($message, $httpCode);
        }

        $decodedResponse = json_decode($response, true);
        if (json_last_error() !== JSON_ERROR_NONE) {
            throw new VetDrugsApiException('Invalid JSON response: ' . json_last_error_msg());
        }

        return $decodedResponse;
    }

    /**
     * Validate calculation request data
     *
     * @param array $patient Patient data
     * @param array $drugs Drugs array
     * @throws VetDrugsValidationException
     */
    private function validateCalculationRequest($patient, $drugs)
    {
        $errors = [];

        // Validate patient
        if (empty($patient)) {
            $errors[] = 'Patient information is required';
        } else {
            if (!isset($patient['weight_kg']) || $patient['weight_kg'] <= 0) {
                $errors[] = 'Valid patient weight is required';
            }

            $species = strtolower($patient['species'] ?? '');
            if (!in_array($species, ['dog', 'cat'])) {
                $errors[] = 'Species must be dog or cat';
            }
        }

        // Validate drugs
        if (empty($drugs) || !is_array($drugs)) {
            $errors[] = 'At least one drug is required';
        } else {
            foreach ($drugs as $index => $drug) {
                if (empty($drug['drug_name'])) {
                    $errors[] = "Drug #" . ($index + 1) . ": drug_name is required";
                }
            }
        }

        if (!empty($errors)) {
            throw new VetDrugsValidationException('Request validation failed', $errors);
        }
    }
}

/**
 * Base exception for VetDrugs API errors
 */
class VetDrugsApiException extends Exception
{
    protected $statusCode;

    public function __construct($message, $statusCode = 0, Exception $previous = null)
    {
        parent::__construct($message, 0, $previous);
        $this->statusCode = $statusCode;
    }

    public function getStatusCode()
    {
        return $this->statusCode;
    }
}

/**
 * Exception for validation errors
 */
class VetDrugsValidationException extends VetDrugsApiException
{
    protected $errors;

    public function __construct($message, $errors = [])
    {
        parent::__construct($message);
        $this->errors = $errors;
    }

    public function getErrors()
    {
        return $this->errors;
    }
}

/**
 * Exception for rate limit errors
 */
class VetDrugsRateLimitException extends VetDrugsApiException
{
    // Inherits from base exception
}
```

### Helper Classes and Functions

```php
<?php

/**
 * Helper class for common VetDrugs operations
 */
class VetDrugsHelpers
{
    /**
     * Convert pounds to kilograms
     *
     * @param float $pounds Weight in pounds
     * @return float Weight in kilograms
     */
    public static function poundsToKg($pounds)
    {
        return round($pounds * 0.453592, 2);
    }

    /**
     * Convert kilograms to pounds
     *
     * @param float $kg Weight in kilograms
     * @return float Weight in pounds
     */
    public static function kgToPounds($kg)
    {
        return round($kg * 2.20462, 2);
    }

    /**
     * Check if weight is within reasonable range
     *
     * @param float $weightKg Weight in kilograms
     * @return bool True if valid
     */
    public static function isValidWeight($weightKg)
    {
        return $weightKg >= 0.1 && $weightKg <= 1000;
    }

    /**
     * Get user-friendly status message
     *
     * @param string $doseRangeStatus Status from API
     * @return string User-friendly message
     */
    public static function getStatusMessage($doseRangeStatus)
    {
        $statusMessages = [
            'within_range' => '✓ Normal dose',
            'below_range' => '⚠️ Below normal range',
            'above_range' => '⚠️ Above normal range',
            'contraindicated' => '❌ Contraindicated'
        ];

        return $statusMessages[$doseRangeStatus] ?? 'Unknown status';
    }

    /**
     * Format dosage instructions for display
     *
     * @param array $calculation Drug calculation data
     * @return string Formatted instructions
     */
    public static function formatDosageInstructions($calculation)
    {
        return sprintf(
            "Give %.2f %s %s %s",
            $calculation['volume'],
            $calculation['volume_unit'],
            $calculation['route'],
            $calculation['frequency']
        );
    }

    /**
     * Validate species string
     *
     * @param string $species Species to validate
     * @return string Normalized species (dog|cat)
     */
    public static function normalizeSpecies($species)
    {
        $species = strtolower(trim($species));
        
        if (in_array($species, ['dog', 'canine', 'k9'])) {
            return 'dog';
        }
        
        if (in_array($species, ['cat', 'feline'])) {
            return 'cat';
        }
        
        return 'dog'; // Default fallback
    }
}

/**
 * Patient data class
 */
class VetDrugsPatient
{
    public $weightKg;
    public $species;

    public function __construct($weightKg, $species)
    {
        $this->weightKg = (float) $weightKg;
        $this->species = VetDrugsHelpers::normalizeSpecies($species);
    }

    /**
     * Get weight in pounds
     *
     * @return float Weight in pounds
     */
    public function getWeightLbs()
    {
        return VetDrugsHelpers::kgToPounds($this->weightKg);
    }

    /**
     * Set weight from pounds
     *
     * @param float $pounds Weight in pounds
     */
    public function setWeightLbs($pounds)
    {
        $this->weightKg = VetDrugsHelpers::poundsToKg($pounds);
    }

    /**
     * Convert to array for API calls
     *
     * @return array Patient data array
     */
    public function toArray()
    {
        return [
            'weight_kg' => $this->weightKg,
            'species' => $this->species
        ];
    }

    /**
     * Validate patient data
     *
     * @return bool True if valid
     */
    public function isValid()
    {
        return VetDrugsHelpers::isValidWeight($this->weightKg) &&
               in_array($this->species, ['dog', 'cat']);
    }
}

/**
 * Drug request class
 */
class VetDrugsDrugRequest
{
    public $drugName;
    public $concentration;
    public $route;
    public $frequency;
    public $customDose;

    public function __construct($drugName, $concentration = null, $route = null, 
                               $frequency = null, $customDose = null)
    {
        $this->drugName = trim($drugName);
        $this->concentration = $concentration;
        $this->route = $route;
        $this->frequency = $frequency;
        $this->customDose = $customDose;
    }

    /**
     * Convert to array for API calls
     *
     * @return array Drug request data
     */
    public function toArray()
    {
        $data = ['drug_name' => $this->drugName];
        
        if ($this->concentration !== null) {
            $data['concentration'] = $this->concentration;
        }
        if ($this->route !== null) {
            $data['route'] = $this->route;
        }
        if ($this->frequency !== null) {
            $data['frequency'] = $this->frequency;
        }
        if ($this->customDose !== null) {
            $data['custom_dose'] = $this->customDose;
        }
        
        return $data;
    }
}
```

## Practical Examples

### 1. Basic Drug Calculation

```php
<?php
require_once 'VetDrugsApiClient.php';

function calculateBasicDrugs()
{
    // Initialize client
    $client = new VetDrugsApiClient('vd_your_api_key_here');
    
    try {
        // Create patient (25kg dog)
        $patient = new VetDrugsPatient(25.0, 'dog');
        
        // Create drug requests
        $drugs = [
            new VetDrugsDrugRequest('Cephalexin'),
            new VetDrugsDrugRequest('Carprofen')
        ];
        
        // Calculate
        $response = $client->calculateDrugs(
            $patient->toArray(),
            array_map(function($drug) { return $drug->toArray(); }, $drugs)
        );
        
        // Display results
        echo "Calculations for {$patient->weightKg}kg {$patient->species}:\n\n";
        
        foreach ($response['calculations'] as $calc) {
            echo "{$calc['drug_name']}:\n";
            echo "  Instructions: {$calc['instructions']}\n";
            echo "  Dose: {$calc['dose_per_kg']} {$calc['dosage_unit']}\n";
            echo "  Total: {$calc['total_dose']} {$calc['dose_unit']}\n";
            echo "  Volume: " . number_format($calc['volume'], 2) . " {$calc['volume_unit']}\n";
            echo "  Status: " . VetDrugsHelpers::getStatusMessage($calc['dose_range_status']) . "\n";
            
            if (!empty($calc['warnings'])) {
                echo "  ⚠️ Warnings: " . implode(', ', $calc['warnings']) . "\n";
            }
            echo "\n";
        }
        
        // Check for overall warnings
        if (!empty($response['warnings'])) {
            echo "⚠️ Overall warnings: " . implode(', ', $response['warnings']) . "\n";
        }
        
    } catch (VetDrugsValidationException $e) {
        echo "Validation errors:\n";
        foreach ($e->getErrors() as $error) {
            echo "  - $error\n";
        }
    } catch (VetDrugsApiException $e) {
        echo "API error: " . $e->getMessage() . "\n";
    }
}

// Run the example
calculateBasicDrugs();
```

### 2. WordPress Plugin Integration

```php
<?php
/**
 * Plugin Name: VetDrugs Calculator
 * Description: Veterinary drug dosage calculator integration
 * Version: 1.0
 */

// Prevent direct access
if (!defined('ABSPATH')) {
    exit;
}

class VetDrugsWordPressPlugin
{
    private $client;

    public function __construct()
    {
        add_action('init', [$this, 'init']);
        add_action('wp_ajax_calculate_drugs', [$this, 'ajaxCalculateDrugs']);
        add_action('wp_ajax_nopriv_calculate_drugs', [$this, 'ajaxCalculateDrugs']);
        add_action('wp_enqueue_scripts', [$this, 'enqueueScripts']);
        add_shortcode('vetdrugs_calculator', [$this, 'renderCalculator']);
    }

    public function init()
    {
        require_once plugin_dir_path(__FILE__) . 'includes/VetDrugsApiClient.php';
        
        $apiKey = get_option('vetdrugs_api_key');
        if ($apiKey) {
            $this->client = new VetDrugsApiClient($apiKey);
        }
    }

    public function enqueueScripts()
    {
        wp_enqueue_script('jquery');
        wp_enqueue_script(
            'vetdrugs-calculator',
            plugin_dir_url(__FILE__) . 'js/calculator.js',
            ['jquery'],
            '1.0',
            true
        );
        
        wp_localize_script('vetdrugs-calculator', 'vetdrugs_ajax', [
            'ajax_url' => admin_url('admin-ajax.php'),
            'nonce' => wp_create_nonce('vetdrugs_nonce')
        ]);
    }

    public function renderCalculator($atts)
    {
        $atts = shortcode_atts([
            'show_search' => 'true',
            'default_drugs' => 'Cephalexin,Carprofen'
        ], $atts);

        ob_start();
        ?>
        <div id="vetdrugs-calculator">
            <h3>Veterinary Drug Calculator</h3>
            
            <form id="drug-calculator-form">
                <div class="patient-info">
                    <h4>Patient Information</h4>
                    <label>
                        Weight:
                        <input type="number" name="weight" step="0.1" required>
                        <select name="weight_unit">
                            <option value="kg">kg</option>
                            <option value="lb">lb</option>
                        </select>
                    </label>
                    
                    <label>
                        Species:
                        <select name="species" required>
                            <option value="">Select species</option>
                            <option value="dog">Dog</option>
                            <option value="cat">Cat</option>
                        </select>
                    </label>
                </div>
                
                <div class="drug-selection">
                    <h4>Select Drugs</h4>
                    <?php foreach (explode(',', $atts['default_drugs']) as $drug): ?>
                        <label>
                            <input type="checkbox" name="drugs[]" value="<?php echo esc_attr(trim($drug)); ?>">
                            <?php echo esc_html(trim($drug)); ?>
                        </label><br>
                    <?php endforeach; ?>
                    
                    <?php if ($atts['show_search'] === 'true'): ?>
                        <div class="custom-drug">
                            <input type="text" placeholder="Search for drugs..." id="drug-search">
                            <div id="drug-suggestions"></div>
                        </div>
                    <?php endif; ?>
                </div>
                
                <button type="submit">Calculate Dosages</button>
            </form>
            
            <div id="calculation-results" style="display: none;">
                <h4>Calculation Results</h4>
                <div id="results-content"></div>
            </div>
        </div>
        <?php
        return ob_get_clean();
    }

    public function ajaxCalculateDrugs()
    {
        // Verify nonce
        if (!wp_verify_nonce($_POST['nonce'], 'vetdrugs_nonce')) {
            wp_die('Invalid nonce');
        }

        if (!$this->client) {
            wp_send_json_error('API client not configured');
            return;
        }

        try {
            $weight = (float) $_POST['weight'];
            $weightUnit = $_POST['weight_unit'];
            $species = $_POST['species'];
            $drugs = $_POST['drugs'];

            // Convert weight to kg if needed
            if ($weightUnit === 'lb') {
                $weight = VetDrugsHelpers::poundsToKg($weight);
            }

            $patient = ['weight_kg' => $weight, 'species' => $species];
            $drugRequests = array_map(function($drugName) {
                return ['drug_name' => $drugName];
            }, $drugs);

            $response = $this->client->calculateDrugs($patient, $drugRequests);
            
            wp_send_json_success($response);
            
        } catch (Exception $e) {
            wp_send_json_error($e->getMessage());
        }
    }
}

// Initialize plugin
new VetDrugsWordPressPlugin();

// Admin settings page
add_action('admin_menu', function() {
    add_options_page(
        'VetDrugs Settings',
        'VetDrugs',
        'manage_options',
        'vetdrugs-settings',
        'vetdrugs_settings_page'
    );
});

function vetdrugs_settings_page()
{
    if (isset($_POST['submit'])) {
        update_option('vetdrugs_api_key', sanitize_text_field($_POST['api_key']));
        echo '<div class="notice notice-success"><p>Settings saved!</p></div>';
    }
    
    $apiKey = get_option('vetdrugs_api_key');
    ?>
    <div class="wrap">
        <h1>VetDrugs Settings</h1>
        <form method="post">
            <table class="form-table">
                <tr>
                    <th scope="row">API Key</th>
                    <td>
                        <input type="text" name="api_key" value="<?php echo esc_attr($apiKey); ?>" 
                               class="regular-text" placeholder="vd_your_api_key">
                        <p class="description">Enter your VetDrugs API key</p>
                    </td>
                </tr>
            </table>
            <?php submit_button(); ?>
        </form>
    </div>
    <?php
}
```

### 3. Laravel Integration

```php
<?php

namespace App\Services;

use App\Exceptions\VetDrugsException;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Log;

class VetDrugsService
{
    private $apiKey;
    private $baseUrl;

    public function __construct()
    {
        $this->apiKey = config('vetdrugs.api_key');
        $this->baseUrl = config('vetdrugs.api_url', 'https://vetdrugs-calculator-api.vetcalculators.workers.dev');
    }

    /**
     * Calculate drug dosages for a patient
     */
    public function calculateDrugs(array $patient, array $drugs): array
    {
        $this->validateCalculationRequest($patient, $drugs);

        try {
            $response = Http::withHeaders([
                'X-API-Key' => $this->apiKey,
                'Content-Type' => 'application/json'
            ])->timeout(30)->post($this->baseUrl . '/api/calculate', [
                'patient' => $patient,
                'drugs' => $drugs
            ]);

            if ($response->status() === 429) {
                throw new VetDrugsException('Rate limit exceeded', 429);
            }

            $response->throw();

            $data = $response->json();
            
            Log::info('VetDrugs calculation successful', [
                'patient_weight' => $patient['weight_kg'],
                'species' => $patient['species'],
                'drug_count' => count($drugs)
            ]);

            return $data;

        } catch (\Exception $e) {
            Log::error('VetDrugs calculation failed', [
                'error' => $e->getMessage(),
                'patient' => $patient,
                'drugs' => $drugs
            ]);

            throw new VetDrugsException('Drug calculation failed: ' . $e->getMessage());
        }
    }

    /**
     * Search for drugs
     */
    public function searchDrugs(?string $searchTerm = null, ?string $category = null): array
    {
        $params = array_filter([
            'search' => $searchTerm,
            'category' => $category
        ]);

        try {
            $response = Http::withHeaders([
                'X-API-Key' => $this->apiKey
            ])->timeout(30)->get($this->baseUrl . '/api/drugs', $params);

            $response->throw();
            return $response->json();

        } catch (\Exception $e) {
            Log::error('VetDrugs search failed', [
                'error' => $e->getMessage(),
                'search_term' => $searchTerm,
                'category' => $category
            ]);

            throw new VetDrugsException('Drug search failed: ' . $e->getMessage());
        }
    }

    /**
     * Get drug information
     */
    public function getDrugInfo(string $drugName): array
    {
        try {
            $response = Http::withHeaders([
                'X-API-Key' => $this->apiKey
            ])->timeout(30)->get($this->baseUrl . '/api/drug-info/' . urlencode($drugName));

            $response->throw();
            return $response->json();

        } catch (\Exception $e) {
            Log::error('VetDrugs drug info failed', [
                'error' => $e->getMessage(),
                'drug_name' => $drugName
            ]);

            throw new VetDrugsException('Drug info lookup failed: ' . $e->getMessage());
        }
    }

    private function validateCalculationRequest(array $patient, array $drugs): void
    {
        $errors = [];

        if (empty($patient['weight_kg']) || $patient['weight_kg'] <= 0) {
            $errors[] = 'Valid patient weight is required';
        }

        if (!in_array(strtolower($patient['species'] ?? ''), ['dog', 'cat'])) {
            $errors[] = 'Species must be dog or cat';
        }

        if (empty($drugs)) {
            $errors[] = 'At least one drug is required';
        }

        foreach ($drugs as $index => $drug) {
            if (empty($drug['drug_name'])) {
                $errors[] = "Drug #" . ($index + 1) . ": drug_name is required";
            }
        }

        if (!empty($errors)) {
            throw new VetDrugsException('Validation failed: ' . implode(', ', $errors));
        }
    }
}

// Laravel Controller
namespace App\Http\Controllers;

use App\Services\VetDrugsService;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class VetDrugsController extends Controller
{
    private $vetDrugsService;

    public function __construct(VetDrugsService $vetDrugsService)
    {
        $this->vetDrugsService = $vetDrugsService;
    }

    public function calculate(Request $request): JsonResponse
    {
        $request->validate([
            'patient.weight_kg' => 'required|numeric|min:0.1|max:1000',
            'patient.species' => 'required|in:dog,cat',
            'drugs' => 'required|array|min:1',
            'drugs.*.drug_name' => 'required|string'
        ]);

        try {
            $result = $this->vetDrugsService->calculateDrugs(
                $request->input('patient'),
                $request->input('drugs')
            );

            return response()->json($result);

        } catch (\Exception $e) {
            return response()->json([
                'error' => $e->getMessage()
            ], 400);
        }
    }

    public function searchDrugs(Request $request): JsonResponse
    {
        try {
            $result = $this->vetDrugsService->searchDrugs(
                $request->input('search'),
                $request->input('category')
            );

            return response()->json($result);

        } catch (\Exception $e) {
            return response()->json([
                'error' => $e->getMessage()
            ], 400);
        }
    }
}

// Config file: config/vetdrugs.php
return [
    'api_key' => env('VETDRUGS_API_KEY'),
    'api_url' => env('VETDRUGS_API_URL', 'https://vetdrugs-calculator-api.vetcalculators.workers.dev'),
];
```

### 4. CodeIgniter Integration

```php
<?php
// application/libraries/VetDrugs.php

class VetDrugs
{
    private $CI;
    private $client;

    public function __construct()
    {
        $this->CI =& get_instance();
        $this->CI->load->config('vetdrugs');
        
        require_once APPPATH . 'third_party/VetDrugsApiClient.php';
        
        $apiKey = $this->CI->config->item('vetdrugs_api_key');
        $this->client = new VetDrugsApiClient($apiKey);
    }

    public function calculate($patient, $drugs)
    {
        try {
            return $this->client->calculateDrugs($patient, $drugs);
        } catch (Exception $e) {
            log_message('error', 'VetDrugs calculation failed: ' . $e->getMessage());
            return false;
        }
    }

    public function search($searchTerm, $category = null)
    {
        try {
            return $this->client->searchDrugs($searchTerm, $category);
        } catch (Exception $e) {
            log_message('error', 'VetDrugs search failed: ' . $e->getMessage());
            return false;
        }
    }
}

// application/controllers/DrugCalculator.php
class DrugCalculator extends CI_Controller
{
    public function __construct()
    {
        parent::__construct();
        $this->load->library('vetdrugs');
        $this->load->helper(['form', 'url']);
        $this->load->library('form_validation');
    }

    public function index()
    {
        $this->load->view('drug_calculator');
    }

    public function calculate()
    {
        $this->form_validation->set_rules('weight', 'Weight', 'required|numeric');
        $this->form_validation->set_rules('species', 'Species', 'required|in_list[dog,cat]');
        $this->form_validation->set_rules('drugs[]', 'Drugs', 'required');

        if ($this->form_validation->run() === FALSE) {
            $this->load->view('drug_calculator');
            return;
        }

        $weight = $this->input->post('weight');
        $weightUnit = $this->input->post('weight_unit');
        $species = $this->input->post('species');
        $drugs = $this->input->post('drugs');

        // Convert to kg if needed
        if ($weightUnit === 'lb') {
            $weight = $weight * 0.453592;
        }

        $patient = [
            'weight_kg' => $weight,
            'species' => $species
        ];

        $drugRequests = array_map(function($drugName) {
            return ['drug_name' => $drugName];
        }, $drugs);

        $result = $this->vetdrugs->calculate($patient, $drugRequests);

        if ($result === false) {
            $data['error'] = 'Calculation failed. Please try again.';
        } else {
            $data['result'] = $result;
        }

        $this->load->view('drug_calculator_results', $data);
    }
}
```

### 5. Caching Implementation

```php
<?php

/**
 * Cached VetDrugs client with Redis/Memcached support
 */
class CachedVetDrugsClient extends VetDrugsApiClient
{
    private $cache;
    private $cacheTtl;

    public function __construct($apiKey, $cache = null, $cacheTtl = 300)
    {
        parent::__construct($apiKey);
        $this->cache = $cache; // Redis/Memcached instance
        $this->cacheTtl = $cacheTtl; // 5 minutes default
    }

    public function searchDrugs($searchTerm = null, $category = null)
    {
        if (!$this->cache) {
            return parent::searchDrugs($searchTerm, $category);
        }

        $cacheKey = $this->generateCacheKey('search', $searchTerm, $category);
        $cached = $this->cache->get($cacheKey);

        if ($cached !== false) {
            return json_decode($cached, true);
        }

        $result = parent::searchDrugs($searchTerm, $category);
        $this->cache->setex($cacheKey, $this->cacheTtl, json_encode($result));

        return $result;
    }

    public function getDrugInfo($drugName)
    {
        if (!$this->cache) {
            return parent::getDrugInfo($drugName);
        }

        $cacheKey = $this->generateCacheKey('drug_info', $drugName);
        $cached = $this->cache->get($cacheKey);

        if ($cached !== false) {
            return json_decode($cached, true);
        }

        $result = parent::getDrugInfo($drugName);
        $this->cache->setex($cacheKey, $this->cacheTtl * 4, json_encode($result)); // Cache longer

        return $result;
    }

    private function generateCacheKey(...$parts)
    {
        return 'vetdrugs:' . md5(implode(':', array_filter($parts)));
    }
}

// Usage with Redis
$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

$client = new CachedVetDrugsClient('vd_your_api_key', $redis, 300);
```

### 6. Error Handling and Logging

```php
<?php

/**
 * Enhanced error handling for VetDrugs integration
 */
class VetDrugsErrorHandler
{
    public static function handleException(Exception $e, $context = [])
    {
        $errorData = [
            'message' => $e->getMessage(),
            'code' => $e->getCode(),
            'file' => $e->getFile(),
            'line' => $e->getLine(),
            'context' => $context,
            'timestamp' => date('Y-m-d H:i:s')
        ];

        // Log error
        error_log('VetDrugs Error: ' . json_encode($errorData));

        // Return user-friendly message
        if ($e instanceof VetDrugsValidationException) {
            return [
                'success' => false,
                'message' => 'Validation failed',
                'errors' => $e->getErrors()
            ];
        }

        if ($e instanceof VetDrugsRateLimitException) {
            return [
                'success' => false,
                'message' => 'Too many requests. Please wait a moment and try again.',
                'retry_after' => 60
            ];
        }

        if ($e instanceof VetDrugsApiException) {
            $statusCode = $e->getStatusCode();
            
            switch ($statusCode) {
                case 401:
                    $message = 'Invalid API key. Please check your configuration.';
                    break;
                case 404:
                    $message = 'Drug not found in database.';
                    break;
                case 503:
                    $message = 'Service temporarily unavailable. Please try again later.';
                    break;
                default:
                    $message = 'API error occurred. Please try again.';
            }

            return [
                'success' => false,
                'message' => $message,
                'status_code' => $statusCode
            ];
        }

        return [
            'success' => false,
            'message' => 'An unexpected error occurred. Please try again.'
        ];
    }
}

// Usage in controllers
try {
    $result = $vetDrugsClient->calculateDrugs($patient, $drugs);
    echo json_encode(['success' => true, 'data' => $result]);
} catch (Exception $e) {
    $errorResponse = VetDrugsErrorHandler::handleException($e, [
        'patient' => $patient,
        'drugs' => $drugs
    ]);
    echo json_encode($errorResponse);
}
```

## Configuration Examples

### Environment Variables (.env)

```bash
VETDRUGS_API_KEY=vd_your_api_key_here
VETDRUGS_API_URL=https://vetdrugs-calculator-api.vetcalculators.workers.dev
VETDRUGS_TIMEOUT=30
VETDRUGS_CACHE_TTL=300
```

### Configuration Class

```php
<?php

class VetDrugsConfig
{
    public static function getApiKey()
    {
        return $_ENV['VETDRUGS_API_KEY'] ?? 
               getenv('VETDRUGS_API_KEY') ?? 
               null;
    }

    public static function getApiUrl()
    {
        return $_ENV['VETDRUGS_API_URL'] ?? 
               getenv('VETDRUGS_API_URL') ?? 
               'https://vetdrugs-calculator-api.vetcalculators.workers.dev';
    }

    public static function getTimeout()
    {
        return (int) ($_ENV['VETDRUGS_TIMEOUT'] ?? 
                     getenv('VETDRUGS_TIMEOUT') ?? 
                     30);
    }

    public static function getCacheTtl()
    {
        return (int) ($_ENV['VETDRUGS_CACHE_TTL'] ?? 
                     getenv('VETDRUGS_CACHE_TTL') ?? 
                     300);
    }

    public static function validate()
    {
        $apiKey = self::getApiKey();
        if (empty($apiKey)) {
            throw new Exception('VETDRUGS_API_KEY environment variable is required');
        }

        if (!filter_var(self::getApiUrl(), FILTER_VALIDATE_URL)) {
            throw new Exception('Invalid VETDRUGS_API_URL');
        }

        return true;
    }
}
```

## Testing

### PHPUnit Tests

```php
<?php

use PHPUnit\Framework\TestCase;

class VetDrugsApiClientTest extends TestCase
{
    private $client;

    protected function setUp(): void
    {
        $this->client = new VetDrugsApiClient('test_api_key');
    }

    public function testValidateCalculationRequestWithValidData()
    {
        $patient = ['weight_kg' => 25, 'species' => 'dog'];
        $drugs = [['drug_name' => 'Cephalexin']];

        // Should not throw exception
        $reflection = new ReflectionClass($this->client);
        $method = $reflection->getMethod('validateCalculationRequest');
        $method->setAccessible(true);
        
        $this->assertNull($method->invoke($this->client, $patient, $drugs));
    }

    public function testValidateCalculationRequestWithInvalidWeight()
    {
        $this->expectException(VetDrugsValidationException::class);
        
        $patient = ['weight_kg' => 0, 'species' => 'dog'];
        $drugs = [['drug_name' => 'Cephalexin']];

        $reflection = new ReflectionClass($this->client);
        $method = $reflection->getMethod('validateCalculationRequest');
        $method->setAccessible(true);
        
        $method->invoke($this->client, $patient, $drugs);
    }

    public function testPoundsToKgConversion()
    {
        $pounds = 55.1;
        $kg = VetDrugsHelpers::poundsToKg($pounds);
        
        $this->assertEquals(25.0, $kg, '', 0.1);
    }

    public function testSpeciesNormalization()
    {
        $this->assertEquals('dog', VetDrugsHelpers::normalizeSpecies('Canine'));
        $this->assertEquals('cat', VetDrugsHelpers::normalizeSpecies('Feline'));
        $this->assertEquals('dog', VetDrugsHelpers::normalizeSpecies('Unknown'));
    }
}
```

## Next Steps

- [View Python examples](python.md)
- [Explore PMI integrations](../integrations/)
- [API Reference](../api-reference/)
- [Support documentation](../support.md)
