# C# (.NET) SDK and Examples

Complete C# integration examples for the VetDrugs API, perfect for Windows-based PMI systems.

## Installation

### NuGet Package Manager
```powershell
Install-Package Newtonsoft.Json
Install-Package System.Net.Http
```

### Package References (.csproj)
```xml
<PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
<PackageReference Include="System.Net.Http" Version="4.3.4" />
```

## Core SDK Implementation

### VetDrugsApiClient Class

```csharp
using System;
using System.Collections.Generic;
using System.Net.Http;
using System.Text;
using System.Threading.Tasks;
using Newtonsoft.Json;

namespace VetDrugs.SDK
{
    public class VetDrugsApiClient : IDisposable
    {
        private readonly HttpClient _httpClient;
        private readonly string _apiKey;
        private readonly string _baseUrl;
        private bool _disposed = false;

        public VetDrugsApiClient(string apiKey, string baseUrl = "https://vetdrugs-calculator-api.vetcalculators.workers.dev")
        {
            _apiKey = apiKey ?? throw new ArgumentNullException(nameof(apiKey));
            _baseUrl = baseUrl.TrimEnd('/');
            
            _httpClient = new HttpClient
            {
                Timeout = TimeSpan.FromSeconds(30)
            };
            
            _httpClient.DefaultRequestHeaders.Add("X-API-Key", _apiKey);
            _httpClient.DefaultRequestHeaders.Add("User-Agent", "VetDrugs-CSharp-SDK/1.0");
        }

        /// <summary>
        /// Calculate drug dosages for a patient
        /// </summary>
        public async Task<CalculationResponse> CalculateDrugsAsync(CalculationRequest request)
        {
            ValidateCalculationRequest(request);
            
            var json = JsonConvert.SerializeObject(request, Formatting.None);
            var content = new StringContent(json, Encoding.UTF8, "application/json");

            var response = await _httpClient.PostAsync($"{_baseUrl}/api/calculate", content);
            
            if (!response.IsSuccessStatusCode)
            {
                await HandleErrorResponse(response);
            }

            var responseJson = await response.Content.ReadAsStringAsync();
            return JsonConvert.DeserializeObject<CalculationResponse>(responseJson);
        }

        /// <summary>
        /// Search for drugs in the database
        /// </summary>
        public async Task<DrugSearchResponse> SearchDrugsAsync(string searchTerm = null, string category = null)
        {
            var queryParams = new List<string>();
            
            if (!string.IsNullOrEmpty(searchTerm))
                queryParams.Add($"search={Uri.EscapeDataString(searchTerm)}");
            
            if (!string.IsNullOrEmpty(category))
                queryParams.Add($"category={Uri.EscapeDataString(category)}");
            
            var queryString = queryParams.Count > 0 ? "?" + string.Join("&", queryParams) : "";
            var url = $"{_baseUrl}/api/drugs{queryString}";

            var response = await _httpClient.GetAsync(url);
            
            if (!response.IsSuccessStatusCode)
            {
                await HandleErrorResponse(response);
            }

            var responseJson = await response.Content.ReadAsStringAsync();
            return JsonConvert.DeserializeObject<DrugSearchResponse>(responseJson);
        }

        /// <summary>
        /// Get detailed information about a specific drug
        /// </summary>
        public async Task<DrugInfoResponse> GetDrugInfoAsync(string drugName)
        {
            if (string.IsNullOrEmpty(drugName))
                throw new ArgumentException("Drug name is required", nameof(drugName));

            var url = $"{_baseUrl}/api/drug-info/{Uri.EscapeDataString(drugName)}";
            var response = await _httpClient.GetAsync(url);
            
            if (!response.IsSuccessStatusCode)
            {
                await HandleErrorResponse(response);
            }

            var responseJson = await response.Content.ReadAsStringAsync();
            return JsonConvert.DeserializeObject<DrugInfoResponse>(responseJson);
        }

        /// <summary>
        /// Check API health status
        /// </summary>
        public async Task<HealthResponse> HealthCheckAsync()
        {
            var response = await _httpClient.GetAsync($"{_baseUrl}/api/health");
            var responseJson = await response.Content.ReadAsStringAsync();
            return JsonConvert.DeserializeObject<HealthResponse>(responseJson);
        }

        private void ValidateCalculationRequest(CalculationRequest request)
        {
            var errors = new List<string>();

            if (request?.Patient == null)
            {
                errors.Add("Patient information is required");
            }
            else
            {
                if (request.Patient.WeightKg <= 0)
                    errors.Add("Valid patient weight is required");
                
                if (!IsValidSpecies(request.Patient.Species))
                    errors.Add("Species must be 'dog' or 'cat'");
            }

            if (request?.Drugs == null || request.Drugs.Count == 0)
            {
                errors.Add("At least one drug is required");
            }
            else
            {
                for (int i = 0; i < request.Drugs.Count; i++)
                {
                    if (string.IsNullOrEmpty(request.Drugs[i].DrugName))
                        errors.Add($"Drug #{i + 1}: Drug name is required");
                }
            }

            if (errors.Count > 0)
                throw new VetDrugsValidationException("Request validation failed", errors);
        }

        private bool IsValidSpecies(string species)
        {
            return !string.IsNullOrEmpty(species) && 
                   (species.Equals("dog", StringComparison.OrdinalIgnoreCase) || 
                    species.Equals("cat", StringComparison.OrdinalIgnoreCase));
        }

        private async Task HandleErrorResponse(HttpResponseMessage response)
        {
            var errorContent = await response.Content.ReadAsStringAsync();
            var statusCode = (int)response.StatusCode;

            try
            {
                var errorResponse = JsonConvert.DeserializeObject<ErrorResponse>(errorContent);
                throw new VetDrugsApiException(errorResponse.Message, statusCode, errorResponse);
            }
            catch (JsonException)
            {
                throw new VetDrugsApiException($"HTTP {statusCode}: {errorContent}", statusCode);
            }
        }

        public void Dispose()
        {
            if (!_disposed)
            {
                _httpClient?.Dispose();
                _disposed = true;
            }
        }
    }
}
```

### Data Models

```csharp
using System;
using System.Collections.Generic;
using Newtonsoft.Json;

namespace VetDrugs.SDK
{
    public class CalculationRequest
    {
        [JsonProperty("patient")]
        public Patient Patient { get; set; }

        [JsonProperty("drugs")]
        public List<DrugRequest> Drugs { get; set; } = new List<DrugRequest>();
    }

    public class Patient
    {
        [JsonProperty("weight_kg")]
        public double WeightKg { get; set; }

        [JsonProperty("species")]
        public string Species { get; set; }

        // Helper properties for convenience
        [JsonIgnore]
        public double WeightLbs
        {
            get => WeightKg * 2.20462;
            set => WeightKg = value / 2.20462;
        }

        [JsonIgnore]
        public bool IsDog => Species?.Equals("dog", StringComparison.OrdinalIgnoreCase) == true;

        [JsonIgnore]
        public bool IsCat => Species?.Equals("cat", StringComparison.OrdinalIgnoreCase) == true;
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

        [JsonProperty("custom_dose")]
        public double? CustomDose { get; set; }
    }

    public class CalculationResponse
    {
        [JsonProperty("patient")]
        public Patient Patient { get; set; }

        [JsonProperty("calculations")]
        public List<DrugCalculation> Calculations { get; set; } = new List<DrugCalculation>();

        [JsonProperty("warnings")]
        public List<string> Warnings { get; set; } = new List<string>();

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
        public List<string> Warnings { get; set; } = new List<string>();

        // Helper properties
        [JsonIgnore]
        public bool IsWithinRange => DoseRangeStatus == "within_range";

        [JsonIgnore]
        public bool HasWarnings => Warnings?.Count > 0;

        [JsonIgnore]
        public string FormattedInstruction => $"{Volume:F2} {VolumeUnit} {Route} {Frequency}";
    }

    public class DrugSearchResponse
    {
        [JsonProperty("drugs")]
        public List<DrugInfo> Drugs { get; set; } = new List<DrugInfo>();

        [JsonProperty("total_count")]
        public int TotalCount { get; set; }

        [JsonProperty("metadata")]
        public SearchMetadata Metadata { get; set; }
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
        public List<string> Species { get; set; } = new List<string>();

        [JsonProperty("has_feline_dose")]
        public bool HasFelineDose { get; set; }

        [JsonProperty("brand_name")]
        public string BrandName { get; set; }
    }

    public class DrugInfoResponse
    {
        [JsonProperty("drug")]
        public DetailedDrugInfo Drug { get; set; }
    }

    public class DetailedDrugInfo : DrugInfo
    {
        [JsonProperty("description")]
        public string Description { get; set; }

        [JsonProperty("contraindications")]
        public List<string> Contraindications { get; set; } = new List<string>();

        [JsonProperty("side_effects")]
        public List<string> SideEffects { get; set; } = new List<string>();

        [JsonProperty("drug_interactions")]
        public List<string> DrugInteractions { get; set; } = new List<string>();
    }

    public class HealthResponse
    {
        [JsonProperty("status")]
        public string Status { get; set; }

        [JsonProperty("version")]
        public string Version { get; set; }

        [JsonProperty("uptime")]
        public long Uptime { get; set; }
    }

    public class ResponseMetadata
    {
        [JsonProperty("calculation_timestamp")]
        public DateTime CalculationTimestamp { get; set; }

        [JsonProperty("api_version")]
        public string ApiVersion { get; set; }

        [JsonProperty("drugs_not_found")]
        public List<string> DrugsNotFound { get; set; } = new List<string>();
    }

    public class SearchMetadata
    {
        [JsonProperty("search_term")]
        public string SearchTerm { get; set; }

        [JsonProperty("category")]
        public string Category { get; set; }

        [JsonProperty("total_in_database")]
        public int TotalInDatabase { get; set; }
    }

    public class ErrorResponse
    {
        [JsonProperty("error")]
        public string Error { get; set; }

        [JsonProperty("message")]
        public string Message { get; set; }

        [JsonProperty("details")]
        public string Details { get; set; }
    }
}
```

### Custom Exceptions

```csharp
using System;
using System.Collections.Generic;

namespace VetDrugs.SDK
{
    public class VetDrugsApiException : Exception
    {
        public int StatusCode { get; }
        public ErrorResponse ErrorResponse { get; }

        public VetDrugsApiException(string message, int statusCode, ErrorResponse errorResponse = null) 
            : base(message)
        {
            StatusCode = statusCode;
            ErrorResponse = errorResponse;
        }

        public VetDrugsApiException(string message, int statusCode, Exception innerException) 
            : base(message, innerException)
        {
            StatusCode = statusCode;
        }
    }

    public class VetDrugsValidationException : Exception
    {
        public List<string> ValidationErrors { get; }

        public VetDrugsValidationException(string message, List<string> validationErrors) 
            : base(message)
        {
            ValidationErrors = validationErrors ?? new List<string>();
        }
    }

    public class VetDrugsTimeoutException : Exception
    {
        public VetDrugsTimeoutException(string message) : base(message) { }
        public VetDrugsTimeoutException(string message, Exception innerException) : base(message, innerException) { }
    }
}
```

## Practical Examples

### 1. Basic Drug Calculation

```csharp
using VetDrugs.SDK;

class Program
{
    static async Task Main(string[] args)
    {
        var client = new VetDrugsApiClient("vd_your_api_key_here");

        try
        {
            var request = new CalculationRequest
            {
                Patient = new Patient
                {
                    WeightKg = 25.0,
                    Species = "dog"
                },
                Drugs = new List<DrugRequest>
                {
                    new DrugRequest { DrugName = "Cephalexin" },
                    new DrugRequest { DrugName = "Carprofen" }
                }
            };

            var response = await client.CalculateDrugsAsync(request);

            Console.WriteLine($"Calculations for {response.Patient.WeightKg}kg {response.Patient.Species}:");
            
            foreach (var calc in response.Calculations)
            {
                Console.WriteLine($"\n{calc.DrugName}:");
                Console.WriteLine($"  Instructions: {calc.Instructions}");
                Console.WriteLine($"  Dose: {calc.DosePerKg} {calc.DoseUnit}/kg");
                Console.WriteLine($"  Total: {calc.TotalDose} {calc.DoseUnit}");
                Console.WriteLine($"  Volume: {calc.Volume} {calc.VolumeUnit}");
                Console.WriteLine($"  Status: {calc.DoseRangeStatus}");

                if (calc.HasWarnings)
                {
                    Console.WriteLine($"  ⚠️ Warnings: {string.Join(", ", calc.Warnings)}");
                }
            }
        }
        catch (VetDrugsApiException ex)
        {
            Console.WriteLine($"API Error ({ex.StatusCode}): {ex.Message}");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error: {ex.Message}");
        }
        finally
        {
            client.Dispose();
        }
    }
}
```

### 2. WinForms Integration

```csharp
using System;
using System.Drawing;
using System.Linq;
using System.Threading.Tasks;
using System.Windows.Forms;
using VetDrugs.SDK;

public partial class DrugCalculatorForm : Form
{
    private VetDrugsApiClient _apiClient;
    private ComboBox cmbSpecies;
    private NumericUpDown nudWeight;
    private TextBox txtDrugName;
    private Button btnCalculate;
    private DataGridView dgvResults;
    private Label lblStatus;

    public DrugCalculatorForm()
    {
        InitializeComponent();
        _apiClient = new VetDrugsApiClient("vd_your_api_key_here");
    }

    private void InitializeComponent()
    {
        this.Size = new Size(800, 600);
        this.Text = "VetDrugs Calculator";

        // Species selection
        var lblSpecies = new Label { Text = "Species:", Location = new Point(10, 15), Size = new Size(60, 23) };
        cmbSpecies = new ComboBox
        {
            Location = new Point(80, 12),
            Size = new Size(100, 23),
            DropDownStyle = ComboBoxStyle.DropDownList
        };
        cmbSpecies.Items.AddRange(new[] { "dog", "cat" });
        cmbSpecies.SelectedIndex = 0;

        // Weight input
        var lblWeight = new Label { Text = "Weight (kg):", Location = new Point(200, 15), Size = new Size(80, 23) };
        nudWeight = new NumericUpDown
        {
            Location = new Point(290, 12),
            Size = new Size(80, 23),
            DecimalPlaces = 2,
            Minimum = 0.1m,
            Maximum = 1000m,
            Value = 25m
        };

        // Drug name input
        var lblDrug = new Label { Text = "Drug:", Location = new Point(390, 15), Size = new Size(40, 23) };
        txtDrugName = new TextBox
        {
            Location = new Point(440, 12),
            Size = new Size(150, 23),
            Text = "Cephalexin"
        };

        // Calculate button
        btnCalculate = new Button
        {
            Text = "Calculate",
            Location = new Point(610, 10),
            Size = new Size(80, 27)
        };
        btnCalculate.Click += BtnCalculate_Click;

        // Results grid
        dgvResults = new DataGridView
        {
            Location = new Point(10, 50),
            Size = new Size(760, 400),
            AutoSizeColumnsMode = DataGridViewAutoSizeColumnsMode.Fill,
            ReadOnly = true,
            AllowUserToAddRows = false
        };
        
        SetupResultsGrid();

        // Status label
        lblStatus = new Label
        {
            Location = new Point(10, 460),
            Size = new Size(760, 23),
            Text = "Ready to calculate drug dosages."
        };

        // Add controls to form
        this.Controls.AddRange(new Control[] 
        { 
            lblSpecies, cmbSpecies, lblWeight, nudWeight, 
            lblDrug, txtDrugName, btnCalculate, dgvResults, lblStatus 
        });
    }

    private void SetupResultsGrid()
    {
        dgvResults.Columns.Add("DrugName", "Drug");
        dgvResults.Columns.Add("Instructions", "Instructions");
        dgvResults.Columns.Add("Volume", "Volume");
        dgvResults.Columns.Add("Status", "Status");
        dgvResults.Columns.Add("Warnings", "Warnings");
    }

    private async void BtnCalculate_Click(object sender, EventArgs e)
    {
        if (string.IsNullOrWhiteSpace(txtDrugName.Text))
        {
            MessageBox.Show("Please enter a drug name.", "Validation Error");
            return;
        }

        btnCalculate.Enabled = false;
        lblStatus.Text = "Calculating...";
        dgvResults.Rows.Clear();

        try
        {
            var request = new CalculationRequest
            {
                Patient = new Patient
                {
                    WeightKg = (double)nudWeight.Value,
                    Species = cmbSpecies.SelectedItem.ToString()
                },
                Drugs = new List<DrugRequest>
                {
                    new DrugRequest { DrugName = txtDrugName.Text.Trim() }
                }
            };

            var response = await _apiClient.CalculateDrugsAsync(request);
            DisplayResults(response);
            lblStatus.Text = $"Calculation completed for {response.Calculations.Count} drug(s).";
        }
        catch (VetDrugsValidationException ex)
        {
            MessageBox.Show($"Validation Error:\n{string.Join("\n", ex.ValidationErrors)}", 
                          "Validation Error", MessageBoxButtons.OK, MessageBoxIcon.Warning);
            lblStatus.Text = "Validation failed.";
        }
        catch (VetDrugsApiException ex)
        {
            MessageBox.Show($"API Error ({ex.StatusCode}): {ex.Message}", 
                          "API Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
            lblStatus.Text = "API error occurred.";
        }
        catch (Exception ex)
        {
            MessageBox.Show($"Unexpected error: {ex.Message}", 
                          "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
            lblStatus.Text = "Error occurred.";
        }
        finally
        {
            btnCalculate.Enabled = true;
        }
    }

    private void DisplayResults(CalculationResponse response)
    {
        foreach (var calc in response.Calculations)
        {
            var row = new DataGridViewRow();
            row.CreateCells(dgvResults);

            row.Cells[0].Value = calc.DrugName;
            row.Cells[1].Value = calc.Instructions;
            row.Cells[2].Value = $"{calc.Volume:F2} {calc.VolumeUnit}";
            row.Cells[3].Value = calc.DoseRangeStatus;
            row.Cells[4].Value = calc.HasWarnings ? string.Join(", ", calc.Warnings) : "None";

            // Color coding based on dose status
            switch (calc.DoseRangeStatus)
            {
                case "above_range":
                case "contraindicated":
                    row.DefaultCellStyle.BackColor = Color.LightCoral;
                    break;
                case "below_range":
                    row.DefaultCellStyle.BackColor = Color.LightYellow;
                    break;
                case "within_range":
                    row.DefaultCellStyle.BackColor = Color.LightGreen;
                    break;
            }

            dgvResults.Rows.Add(row);
        }
    }

    protected override void OnFormClosed(FormClosedEventArgs e)
    {
        _apiClient?.Dispose();
        base.OnFormClosed(e);
    }
}
```

### 3. Drug Search with Autocomplete

```csharp
public class DrugSearchTextBox : TextBox
{
    private ListBox _suggestionBox;
    private VetDrugsApiClient _apiClient;
    private Timer _searchTimer;

    public event EventHandler<DrugInfo> DrugSelected;

    public DrugSearchTextBox(VetDrugsApiClient apiClient)
    {
        _apiClient = apiClient;
        
        _suggestionBox = new ListBox
        {
            Visible = false,
            DisplayMember = "Name"
        };
        
        _searchTimer = new Timer { Interval = 300 };
        _searchTimer.Tick += SearchTimer_Tick;

        this.TextChanged += OnTextChanged;
        this.KeyDown += OnKeyDown;
        this.Leave += OnLeave;
        
        _suggestionBox.Click += OnSuggestionClick;
        
        // Add suggestion box to parent
        this.ParentChanged += (s, e) => 
        {
            if (this.Parent != null)
                this.Parent.Controls.Add(_suggestionBox);
        };
    }

    private void OnTextChanged(object sender, EventArgs e)
    {
        _searchTimer.Stop();
        
        if (this.Text.Length >= 3)
        {
            _searchTimer.Start();
        }
        else
        {
            HideSuggestions();
        }
    }

    private async void SearchTimer_Tick(object sender, EventArgs e)
    {
        _searchTimer.Stop();
        
        try
        {
            var response = await _apiClient.SearchDrugsAsync(this.Text);
            ShowSuggestions(response.Drugs.Take(10).ToList());
        }
        catch
        {
            HideSuggestions();
        }
    }

    private void ShowSuggestions(List<DrugInfo> drugs)
    {
        _suggestionBox.Items.Clear();
        
        foreach (var drug in drugs)
        {
            _suggestionBox.Items.Add(drug);
        }

        if (_suggestionBox.Items.Count > 0)
        {
            _suggestionBox.Location = new Point(this.Left, this.Bottom);
            _suggestionBox.Width = this.Width;
            _suggestionBox.Height = Math.Min(drugs.Count * 20, 200);
            _suggestionBox.Visible = true;
            _suggestionBox.BringToFront();
        }
        else
        {
            HideSuggestions();
        }
    }

    private void HideSuggestions()
    {
        _suggestionBox.Visible = false;
    }

    private void OnKeyDown(object sender, KeyEventArgs e)
    {
        if (_suggestionBox.Visible)
        {
            switch (e.KeyCode)
            {
                case Keys.Down:
                    if (_suggestionBox.SelectedIndex < _suggestionBox.Items.Count - 1)
                        _suggestionBox.SelectedIndex++;
                    e.Handled = true;
                    break;

                case Keys.Up:
                    if (_suggestionBox.SelectedIndex > 0)
                        _suggestionBox.SelectedIndex--;
                    e.Handled = true;
                    break;

                case Keys.Enter:
                    if (_suggestionBox.SelectedItem is DrugInfo drug)
                        SelectDrug(drug);
                    e.Handled = true;
                    break;

                case Keys.Escape:
                    HideSuggestions();
                    e.Handled = true;
                    break;
            }
        }
    }

    private void OnLeave(object sender, EventArgs e)
    {
        // Hide after a delay to allow for clicks
        Task.Delay(150).ContinueWith(_ => 
        {
            if (this.InvokeRequired)
                this.Invoke(new Action(HideSuggestions));
            else
                HideSuggestions();
        });
    }

    private void OnSuggestionClick(object sender, EventArgs e)
    {
        if (_suggestionBox.SelectedItem is DrugInfo drug)
            SelectDrug(drug);
    }

    private void SelectDrug(DrugInfo drug)
    {
        this.Text = drug.Name;
        HideSuggestions();
        DrugSelected?.Invoke(this, drug);
    }
}
```

### 4. Async Helper Methods

```csharp
public static class VetDrugsHelpers
{
    /// <summary>
    /// Convert pounds to kilograms
    /// </summary>
    public static double PoundsToKg(double pounds)
    {
        return Math.Round(pounds * 0.453592, 2);
    }

    /// <summary>
    /// Convert kilograms to pounds
    /// </summary>
    public static double KgToPounds(double kg)
    {
        return Math.Round(kg * 2.20462, 2);
    }

    /// <summary>
    /// Validate weight is within reasonable range
    /// </summary>
    public static bool IsValidWeight(double weightKg)
    {
        return weightKg >= 0.1 && weightKg <= 1000;
    }

    /// <summary>
    /// Format dosage instructions for display
    /// </summary>
    public static string FormatDosageInstructions(DrugCalculation calc)
    {
        return $"Give {calc.Volume:F2} {calc.VolumeUnit} {calc.Route} {calc.Frequency}";
    }

    /// <summary>
    /// Get user-friendly status message
    /// </summary>
    public static string GetStatusMessage(string doseRangeStatus)
    {
        return doseRangeStatus switch
        {
            "within_range" => "✓ Normal dose",
            "below_range" => "⚠️ Below normal range",
            "above_range" => "⚠️ Above normal range",
            "contraindicated" => "❌ Contraindicated",
            _ => "Unknown status"
        };
    }

    /// <summary>
    /// Get status color for UI
    /// </summary>
    public static Color GetStatusColor(string doseRangeStatus)
    {
        return doseRangeStatus switch
        {
            "within_range" => Color.Green,
            "below_range" => Color.Orange,
            "above_range" => Color.Red,
            "contraindicated" => Color.DarkRed,
            _ => Color.Gray
        };
    }
}
```

## Configuration

### App.config Setup

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <appSettings>
    <add key="VetDrugsApiKey" value="vd_your_api_key_here" />
    <add key="VetDrugsApiUrl" value="https://vetdrugs-calculator-api.vetcalculators.workers.dev" />
    <add key="RequestTimeoutSeconds" value="30" />
  </appSettings>
</configuration>
```

### Configuration Helper

```csharp
public static class VetDrugsConfig
{
    public static string ApiKey => ConfigurationManager.AppSettings["VetDrugsApiKey"];
    public static string ApiUrl => ConfigurationManager.AppSettings["VetDrugsApiUrl"];
    public static int TimeoutSeconds => int.Parse(ConfigurationManager.AppSettings["RequestTimeoutSeconds"] ?? "30");

    public static void ValidateConfiguration()
    {
        if (string.IsNullOrEmpty(ApiKey))
            throw new ConfigurationErrorsException("VetDrugsApiKey is not configured");
        
        if (string.IsNullOrEmpty(ApiUrl))
            throw new ConfigurationErrorsException("VetDrugsApiUrl is not configured");
    }
}
```

## Unit Testing

### NUnit Test Example

```csharp
using NUnit.Framework;
using VetDrugs.SDK;
using System.Threading.Tasks;

[TestFixture]
public class VetDrugsApiClientTests
{
    private VetDrugsApiClient _client;

    [SetUp]
    public void Setup()
    {
        _client = new VetDrugsApiClient("test_api_key");
    }

    [TearDown]
    public void TearDown()
    {
        _client?.Dispose();
    }

    [Test]
    public void ValidateCalculationRequest_ValidRequest_DoesNotThrow()
    {
        var request = new CalculationRequest
        {
            Patient = new Patient { WeightKg = 25, Species = "dog" },
            Drugs = new List<DrugRequest> { new DrugRequest { DrugName = "Cephalexin" } }
        };

        Assert.DoesNotThrow(() => _client.ValidateCalculationRequest(request));
    }

    [Test]
    public void ValidateCalculationRequest_InvalidWeight_ThrowsValidationException()
    {
        var request = new CalculationRequest
        {
            Patient = new Patient { WeightKg = 0, Species = "dog" },
            Drugs = new List<DrugRequest> { new DrugRequest { DrugName = "Cephalexin" } }
        };

        Assert.Throws<VetDrugsValidationException>(() => _client.ValidateCalculationRequest(request));
    }

    [Test]
    public void WeightConversion_PoundsToKg_CalculatesCorrectly()
    {
        var patient = new Patient { WeightLbs = 55.1 }; // Should be 25kg
        Assert.AreEqual(25, patient.WeightKg, 0.1);
    }
}
```

## Next Steps

- [View Python examples](python.md)
- [Explore PMI integrations](../integrations/)
- [API Reference](../api-reference/)
