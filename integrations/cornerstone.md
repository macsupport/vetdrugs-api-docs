# Cornerstone Integration Guide

Complete integration guide for Cornerstone Practice Management Software.

## Overview

Cornerstone is a Windows-based practice management system built on .NET Framework. This guide shows how to integrate VetDrugs API calculations directly into your Cornerstone workflows.

## Prerequisites

- Cornerstone Practice Management Software
- VetDrugs API key
- .NET Framework 4.5 or higher
- Administrator access to Cornerstone system

## Integration Architecture

```
Cornerstone → .NET API Client → VetDrugs API → Drug Calculations
```

## Setup Instructions

### 1. Install Required NuGet Packages

Add these packages to your Cornerstone project:

```xml
<PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
<PackageReference Include="System.Net.Http" Version="4.3.4" />
```

### 2. Create VetDrugs API Client

Create a new class file `VetDrugsApiClient.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Net.Http;
using System.Text;
using System.Threading.Tasks;
using Newtonsoft.Json;

namespace CornerStone.VetDrugs
{
    public class VetDrugsApiClient
    {
        private readonly HttpClient _httpClient;
        private readonly string _apiKey;
        private readonly string _baseUrl;

        public VetDrugsApiClient(string apiKey, string baseUrl = "https://vetdrugs-calculator-api.vetcalculators.workers.dev")
        {
            _apiKey = apiKey ?? throw new ArgumentNullException(nameof(apiKey));
            _baseUrl = baseUrl;
            
            _httpClient = new HttpClient();
            _httpClient.DefaultRequestHeaders.Add("X-API-Key", _apiKey);
            _httpClient.Timeout = TimeSpan.FromSeconds(30);
        }

        public async Task<CalculationResponse> CalculateDrugsAsync(CalculationRequest request)
        {
            var json = JsonConvert.SerializeObject(request, Formatting.None);
            var content = new StringContent(json, Encoding.UTF8, "application/json");

            var response = await _httpClient.PostAsync($"{_baseUrl}/api/calculate", content);
            
            if (!response.IsSuccessStatusCode)
            {
                var errorContent = await response.Content.ReadAsStringAsync();
                throw new VetDrugsApiException($"API Error ({response.StatusCode}): {errorContent}");
            }

            var responseJson = await response.Content.ReadAsStringAsync();
            return JsonConvert.DeserializeObject<CalculationResponse>(responseJson);
        }

        public async Task<DrugSearchResponse> SearchDrugsAsync(string searchTerm, string category = null)
        {
            var url = $"{_baseUrl}/api/drugs";
            var queryParams = new List<string>();
            
            if (!string.IsNullOrEmpty(searchTerm))
                queryParams.Add($"search={Uri.EscapeDataString(searchTerm)}");
            
            if (!string.IsNullOrEmpty(category))
                queryParams.Add($"category={Uri.EscapeDataString(category)}");
            
            if (queryParams.Count > 0)
                url += "?" + string.Join("&", queryParams);

            var response = await _httpClient.GetAsync(url);
            
            if (!response.IsSuccessStatusCode)
            {
                var errorContent = await response.Content.ReadAsStringAsync();
                throw new VetDrugsApiException($"Search Error ({response.StatusCode}): {errorContent}");
            }

            var responseJson = await response.Content.ReadAsStringAsync();
            return JsonConvert.DeserializeObject<DrugSearchResponse>(responseJson);
        }

        public void Dispose()
        {
            _httpClient?.Dispose();
        }
    }

    public class VetDrugsApiException : Exception
    {
        public VetDrugsApiException(string message) : base(message) { }
        public VetDrugsApiException(string message, Exception innerException) : base(message, innerException) { }
    }
}
```

### 3. Define Data Models

Create `VetDrugsModels.cs`:

```csharp
using System;
using System.Collections.Generic;
using Newtonsoft.Json;

namespace CornerStone.VetDrugs
{
    public class CalculationRequest
    {
        [JsonProperty("patient")]
        public Patient Patient { get; set; }

        [JsonProperty("drugs")]
        public List<DrugRequest> Drugs { get; set; }
    }

    public class Patient
    {
        [JsonProperty("weight_kg")]
        public double WeightKg { get; set; }

        [JsonProperty("species")]
        public string Species { get; set; }
    }

    public class DrugRequest
    {
        [JsonProperty("drug_name")]
        public string DrugName { get; set; }

        [JsonProperty("concentration")]
        public double? Concentration { get; set; }

        [JsonProperty("route")]
        public string Route { get; set; }

        [JsonProperty("frequency")]
        public string Frequency { get; set; }
    }

    public class CalculationResponse
    {
        [JsonProperty("patient")]
        public Patient Patient { get; set; }

        [JsonProperty("calculations")]
        public List<DrugCalculation> Calculations { get; set; }

        [JsonProperty("warnings")]
        public List<string> Warnings { get; set; }

        [JsonProperty("metadata")]
        public ResponseMetadata Metadata { get; set; }
    }

    public class DrugCalculation
    {
        [JsonProperty("drug_name")]
        public string DrugName { get; set; }

        [JsonProperty("dose_per_kg")]
        public double DosePerKg { get; set; }

        [JsonProperty("total_dose")]
        public double TotalDose { get; set; }

        [JsonProperty("dose_unit")]
        public string DoseUnit { get; set; }

        [JsonProperty("volume")]
        public double Volume { get; set; }

        [JsonProperty("volume_unit")]
        public string VolumeUnit { get; set; }

        [JsonProperty("concentration")]
        public double Concentration { get; set; }

        [JsonProperty("concentration_unit")]
        public string ConcentrationUnit { get; set; }

        [JsonProperty("route")]
        public string Route { get; set; }

        [JsonProperty("frequency")]
        public string Frequency { get; set; }

        [JsonProperty("instructions")]
        public string Instructions { get; set; }

        [JsonProperty("dose_range_status")]
        public string DoseRangeStatus { get; set; }

        [JsonProperty("warnings")]
        public List<string> Warnings { get; set; }
    }

    public class ResponseMetadata
    {
        [JsonProperty("calculation_timestamp")]
        public DateTime CalculationTimestamp { get; set; }

        [JsonProperty("api_version")]
        public string ApiVersion { get; set; }

        [JsonProperty("drugs_not_found")]
        public List<string> DrugsNotFound { get; set; }
    }

    public class DrugSearchResponse
    {
        [JsonProperty("drugs")]
        public List<DrugInfo> Drugs { get; set; }

        [JsonProperty("total_count")]
        public int TotalCount { get; set; }
    }

    public class DrugInfo
    {
        [JsonProperty("name")]
        public string Name { get; set; }

        [JsonProperty("category")]
        public string Category { get; set; }

        [JsonProperty("default_dose")]
        public double DefaultDose { get; set; }

        [JsonProperty("dose_unit")]
        public string DoseUnit { get; set; }

        [JsonProperty("route")]
        public string Route { get; set; }

        [JsonProperty("species")]
        public List<string> Species { get; set; }
    }
}
```

### 4. Configuration Setup

Add to your Cornerstone configuration:

```csharp
// In your configuration or app.config
public class VetDrugsConfig
{
    public static string ApiKey => ConfigurationManager.AppSettings["VetDrugsApiKey"];
    public static string ApiUrl => ConfigurationManager.AppSettings["VetDrugsApiUrl"] ?? 
                                   "https://vetdrugs-calculator-api.vetcalculators.workers.dev";
}
```

App.config entry:
```xml
<appSettings>
    <add key="VetDrugsApiKey" value="vd_your_api_key_here" />
    <add key="VetDrugsApiUrl" value="https://vetdrugs-calculator-api.vetcalculators.workers.dev" />
</appSettings>
```

## Integration Points

### 1. Treatment Plan Integration

Integrate into Cornerstone's treatment planning:

```csharp
public class TreatmentPlanIntegration
{
    private readonly VetDrugsApiClient _apiClient;

    public TreatmentPlanIntegration()
    {
        _apiClient = new VetDrugsApiClient(VetDrugsConfig.ApiKey);
    }

    public async Task<List<DrugCalculation>> CalculateTreatmentDrugs(
        int patientId, 
        List<string> drugNames)
    {
        // Get patient data from Cornerstone
        var patient = GetPatientFromCornerstone(patientId);
        
        // Convert to API format
        var apiPatient = new Patient
        {
            WeightKg = ConvertPoundsToKg(patient.WeightLbs),
            Species = patient.Species.ToLower()
        };

        var drugRequests = drugNames.Select(name => new DrugRequest
        {
            DrugName = name
        }).ToList();

        var request = new CalculationRequest
        {
            Patient = apiPatient,
            Drugs = drugRequests
        };

        try
        {
            var response = await _apiClient.CalculateDrugsAsync(request);
            
            // Log calculations in Cornerstone
            LogCalculationsToCornerstone(patientId, response.Calculations);
            
            return response.Calculations;
        }
        catch (VetDrugsApiException ex)
        {
            // Handle API errors
            LogError($"VetDrugs API Error: {ex.Message}");
            throw new Exception("Unable to calculate drug doses. Please try again.");
        }
    }

    private double ConvertPoundsToKg(double pounds)
    {
        return Math.Round(pounds * 0.453592, 2);
    }

    private CornerstonePatient GetPatientFromCornerstone(int patientId)
    {
        // Your Cornerstone patient retrieval logic
        // Return patient object with weight and species
        throw new NotImplementedException("Implement patient retrieval");
    }

    private void LogCalculationsToCornerstone(int patientId, List<DrugCalculation> calculations)
    {
        // Log calculations to Cornerstone treatment notes
        // Your implementation here
    }

    private void LogError(string message)
    {
        // Your error logging implementation
    }
}
```

### 2. Drug Search Integration

Add drug search to prescription interface:

```csharp
public class DrugSearchIntegration
{
    private readonly VetDrugsApiClient _apiClient;

    public DrugSearchIntegration()
    {
        _apiClient = new VetDrugsApiClient(VetDrugsConfig.ApiKey);
    }

    public async Task<List<string>> SearchDrugsForAutoComplete(string searchTerm)
    {
        try
        {
            var response = await _apiClient.SearchDrugsAsync(searchTerm);
            return response.Drugs.Select(d => d.Name).ToList();
        }
        catch (Exception ex)
        {
            LogError($"Drug search failed: {ex.Message}");
            return new List<string>(); // Return empty list on error
        }
    }

    public async Task<List<DrugInfo>> GetDrugsByCategory(string category)
    {
        try
        {
            var response = await _apiClient.SearchDrugsAsync(null, category);
            return response.Drugs;
        }
        catch (Exception ex)
        {
            LogError($"Category search failed: {ex.Message}");
            return new List<DrugInfo>();
        }
    }

    private void LogError(string message)
    {
        // Your error logging implementation
    }
}
```

### 3. User Interface Integration

Example Windows Forms integration:

```csharp
public partial class DrugCalculationForm : Form
{
    private readonly TreatmentPlanIntegration _treatmentIntegration;
    private readonly DrugSearchIntegration _drugSearch;

    public DrugCalculationForm()
    {
        InitializeComponent();
        _treatmentIntegration = new TreatmentPlanIntegration();
        _drugSearch = new DrugSearchIntegration();
    }

    private async void CalculateButton_Click(object sender, EventArgs e)
    {
        try
        {
            // Disable UI during calculation
            CalculateButton.Enabled = false;
            Cursor = Cursors.WaitCursor;

            // Get selected drugs
            var selectedDrugs = DrugListBox.CheckedItems.Cast<string>().ToList();
            
            if (!selectedDrugs.Any())
            {
                MessageBox.Show("Please select at least one drug.");
                return;
            }

            // Calculate drugs
            var calculations = await _treatmentIntegration.CalculateTreatmentDrugs(
                PatientId, selectedDrugs);

            // Display results
            DisplayCalculations(calculations);
        }
        catch (Exception ex)
        {
            MessageBox.Show($"Error calculating drugs: {ex.Message}", 
                          "Calculation Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
        }
        finally
        {
            // Re-enable UI
            CalculateButton.Enabled = true;
            Cursor = Cursors.Default;
        }
    }

    private void DisplayCalculations(List<DrugCalculation> calculations)
    {
        ResultsDataGridView.Rows.Clear();
        
        foreach (var calc in calculations)
        {
            var row = new DataGridViewRow();
            row.CreateCells(ResultsDataGridView);
            
            row.Cells[0].Value = calc.DrugName;
            row.Cells[1].Value = $"{calc.Volume} {calc.VolumeUnit}";
            row.Cells[2].Value = calc.Instructions;
            row.Cells[3].Value = calc.DoseRangeStatus;
            
            // Color code based on dose status
            if (calc.DoseRangeStatus == "above_range" || calc.DoseRangeStatus == "contraindicated")
                row.DefaultCellStyle.BackColor = Color.LightCoral;
            else if (calc.DoseRangeStatus == "below_range")
                row.DefaultCellStyle.BackColor = Color.LightYellow;
            
            ResultsDataGridView.Rows.Add(row);
        }
    }

    private async void DrugSearchTextBox_TextChanged(object sender, EventArgs e)
    {
        if (DrugSearchTextBox.Text.Length >= 3)
        {
            var suggestions = await _drugSearch.SearchDrugsForAutoComplete(DrugSearchTextBox.Text);
            // Update autocomplete suggestions
            UpdateAutoComplete(suggestions);
        }
    }
}
```

## Error Handling

Implement comprehensive error handling:

```csharp
public class VetDrugsErrorHandler
{
    public static void HandleApiError(Exception ex, Control parentControl)
    {
        string message;
        string title = "VetDrugs API Error";

        if (ex is VetDrugsApiException apiEx)
        {
            if (apiEx.Message.Contains("401"))
            {
                message = "Invalid API key. Please check your VetDrugs configuration.";
                title = "Authentication Error";
            }
            else if (apiEx.Message.Contains("429"))
            {
                message = "API rate limit exceeded. Please wait a moment and try again.";
                title = "Rate Limit";
            }
            else if (apiEx.Message.Contains("503"))
            {
                message = "VetDrugs service is temporarily unavailable. Please try again later.";
                title = "Service Unavailable";
            }
            else
            {
                message = $"API Error: {apiEx.Message}";
            }
        }
        else if (ex is HttpRequestException)
        {
            message = "Unable to connect to VetDrugs API. Please check your internet connection.";
            title = "Connection Error";
        }
        else
        {
            message = $"Unexpected error: {ex.Message}";
        }

        MessageBox.Show(message, title, MessageBoxButtons.OK, MessageBoxIcon.Warning);
    }
}
```

## Testing

Create unit tests for your integration:

```csharp
[TestMethod]
public async Task CalculateDrugs_ValidRequest_ReturnsCalculations()
{
    // Arrange
    var client = new VetDrugsApiClient("test_api_key");
    var request = new CalculationRequest
    {
        Patient = new Patient { WeightKg = 25, Species = "dog" },
        Drugs = new List<DrugRequest> { new DrugRequest { DrugName = "Cephalexin" } }
    };

    // Act
    var response = await client.CalculateDrugsAsync(request);

    // Assert
    Assert.IsNotNull(response);
    Assert.IsTrue(response.Calculations.Count > 0);
    Assert.AreEqual("Cephalexin", response.Calculations[0].DrugName);
}
```

## Deployment

1. **Build and test** your integration thoroughly
2. **Configure API key** in production environment
3. **Deploy to Cornerstone servers**
4. **Train staff** on new drug calculation features
5. **Monitor usage** and error logs

## Support

For integration support:
- 📧 Email: support@vetdrugscalculators.com
- 📞 Phone: Available for Enterprise customers
- 📚 Documentation updates: [docs.vetdrugscalculators.com](https://docs.vetdrugscalculators.com)

## Next Steps

- [View JavaScript examples](../examples/javascript.md)
- [Explore other PMI integrations](../integrations/)
- [API Reference documentation](../api-reference/)