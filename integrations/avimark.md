# Avimark Integration Guide

Complete integration guide for Avimark Practice Management Software.

## Overview

Avimark is a Windows-based veterinary practice management system with SQL Server database backend. This guide shows how to integrate VetDrugs API calculations using direct database queries and custom applications.

## Prerequisites

- Avimark Practice Management Software
- SQL Server access to Avimark database
- VetDrugs API key
- .NET Framework 4.5 or higher
- Database administrator access

## Integration Architecture

```
Avimark SQL Database → Custom .NET Application → VetDrugs API → Drug Calculations
```

## Setup Instructions

### 1. Database Access Configuration

First, ensure you have read access to the Avimark database:

```sql
-- Verify access to key Avimark tables
SELECT TOP 5 * FROM Patients;
SELECT TOP 5 * FROM Invoices;
SELECT TOP 5 * FROM Invoice_Items;
```

### 2. Key Avimark Database Tables

Understanding the Avimark database structure:

| Table | Purpose | Key Fields |
|-------|---------|-------------|
| `Patients` | Patient information | `PatientId`, `Name`, `Weight`, `Species`, `Breed` |
| `Invoices` | Visit records | `InvoiceId`, `PatientId`, `Date`, `DoctorId` |
| `Invoice_Items` | Services and medications | `ItemId`, `InvoiceId`, `Description`, `Quantity` |
| `Doctors` | Veterinarian information | `DoctorId`, `Name`, `Initials` |

### 3. .NET Integration Application

#### Database Connection and Models

```csharp
using System;
using System.Collections.Generic;
using System.Data;
using System.Data.SqlClient;
using System.Configuration;

namespace AvimarkVetDrugs.Models
{
    public class AvimarkPatient
    {
        public int PatientId { get; set; }
        public string Name { get; set; }
        public decimal Weight { get; set; }
        public string WeightUnit { get; set; }
        public string Species { get; set; }
        public string Breed { get; set; }
        public DateTime? LastVisit { get; set; }

        public double WeightKg
        {
            get
            {
                if (WeightUnit?.ToLower() == "lb" || WeightUnit?.ToLower() == "lbs")
                    return (double)(Weight * 0.453592m);
                return (double)Weight;
            }
        }

        public string VetDrugsSpecies
        {
            get
            {
                if (Species?.ToLower().Contains("dog") == true || 
                    Species?.ToLower().Contains("canine") == true)
                    return "dog";
                
                if (Species?.ToLower().Contains("cat") == true || 
                    Species?.ToLower().Contains("feline") == true)
                    return "cat";
                
                return "dog"; // Default fallback
            }
        }
    }

    public class AvimarkInvoice
    {
        public int InvoiceId { get; set; }
        public int PatientId { get; set; }
        public DateTime InvoiceDate { get; set; }
        public int DoctorId { get; set; }
        public string DoctorName { get; set; }
        public List<AvimarkInvoiceItem> Items { get; set; } = new List<AvimarkInvoiceItem>();
    }

    public class AvimarkInvoiceItem
    {
        public int ItemId { get; set; }
        public int InvoiceId { get; set; }
        public string Description { get; set; }
        public decimal Quantity { get; set; }
        public decimal Price { get; set; }
        public string Category { get; set; }
    }
}
```

#### Database Access Layer

```csharp
using System;
using System.Collections.Generic;
using System.Data;
using System.Data.SqlClient;
using AvimarkVetDrugs.Models;

namespace AvimarkVetDrugs.Data
{
    public class AvimarkDataAccess
    {
        private readonly string _connectionString;

        public AvimarkDataAccess(string connectionString)
        {
            _connectionString = connectionString ?? throw new ArgumentNullException(nameof(connectionString));
        }

        public AvimarkPatient GetPatient(int patientId)
        {
            using (var connection = new SqlConnection(_connectionString))
            {
                connection.Open();
                
                var query = @"
                    SELECT 
                        p.PatientId,
                        p.Name,
                        p.Weight,
                        p.WeightUnit,
                        p.Species,
                        p.Breed,
                        MAX(i.Date) as LastVisit
                    FROM Patients p
                    LEFT JOIN Invoices i ON p.PatientId = i.PatientId
                    WHERE p.PatientId = @PatientId
                    GROUP BY p.PatientId, p.Name, p.Weight, p.WeightUnit, p.Species, p.Breed";

                using (var command = new SqlCommand(query, connection))
                {
                    command.Parameters.AddWithValue("@PatientId", patientId);
                    
                    using (var reader = command.ExecuteReader())
                    {
                        if (reader.Read())
                        {
                            return new AvimarkPatient
                            {
                                PatientId = reader.GetInt32("PatientId"),
                                Name = reader.GetString("Name"),
                                Weight = reader.GetDecimal("Weight"),
                                WeightUnit = reader.IsDBNull("WeightUnit") ? "lb" : reader.GetString("WeightUnit"),
                                Species = reader.IsDBNull("Species") ? "Dog" : reader.GetString("Species"),
                                Breed = reader.IsDBNull("Breed") ? "" : reader.GetString("Breed"),
                                LastVisit = reader.IsDBNull("LastVisit") ? (DateTime?)null : reader.GetDateTime("LastVisit")
                            };
                        }
                    }
                }
            }
            
            return null;
        }

        public List<AvimarkPatient> SearchPatients(string searchTerm)
        {
            var patients = new List<AvimarkPatient>();
            
            using (var connection = new SqlConnection(_connectionString))
            {
                connection.Open();
                
                var query = @"
                    SELECT 
                        p.PatientId,
                        p.Name,
                        p.Weight,
                        p.WeightUnit,
                        p.Species,
                        p.Breed,
                        MAX(i.Date) as LastVisit
                    FROM Patients p
                    LEFT JOIN Invoices i ON p.PatientId = i.PatientId
                    WHERE p.Name LIKE @SearchTerm 
                       OR CAST(p.PatientId AS VARCHAR) LIKE @SearchTerm
                    GROUP BY p.PatientId, p.Name, p.Weight, p.WeightUnit, p.Species, p.Breed
                    ORDER BY p.Name";

                using (var command = new SqlCommand(query, connection))
                {
                    command.Parameters.AddWithValue("@SearchTerm", $"%{searchTerm}%");
                    
                    using (var reader = command.ExecuteReader())
                    {
                        while (reader.Read())
                        {
                            patients.Add(new AvimarkPatient
                            {
                                PatientId = reader.GetInt32("PatientId"),
                                Name = reader.GetString("Name"),
                                Weight = reader.GetDecimal("Weight"),
                                WeightUnit = reader.IsDBNull("WeightUnit") ? "lb" : reader.GetString("WeightUnit"),
                                Species = reader.IsDBNull("Species") ? "Dog" : reader.GetString("Species"),
                                Breed = reader.IsDBNull("Breed") ? "" : reader.GetString("Breed"),
                                LastVisit = reader.IsDBNull("LastVisit") ? (DateTime?)null : reader.GetDateTime("LastVisit")
                            });
                        }
                    }
                }
            }
            
            return patients;
        }

        public AvimarkInvoice GetCurrentInvoice(int patientId)
        {
            using (var connection = new SqlConnection(_connectionString))
            {
                connection.Open();
                
                // Get the most recent open invoice
                var invoiceQuery = @"
                    SELECT TOP 1 
                        i.InvoiceId,
                        i.PatientId,
                        i.Date as InvoiceDate,
                        i.DoctorId,
                        d.Name as DoctorName
                    FROM Invoices i
                    LEFT JOIN Doctors d ON i.DoctorId = d.DoctorId
                    WHERE i.PatientId = @PatientId 
                      AND i.Status = 'Open'
                    ORDER BY i.Date DESC";

                using (var command = new SqlCommand(invoiceQuery, connection))
                {
                    command.Parameters.AddWithValue("@PatientId", patientId);
                    
                    using (var reader = command.ExecuteReader())
                    {
                        if (reader.Read())
                        {
                            var invoice = new AvimarkInvoice
                            {
                                InvoiceId = reader.GetInt32("InvoiceId"),
                                PatientId = reader.GetInt32("PatientId"),
                                InvoiceDate = reader.GetDateTime("InvoiceDate"),
                                DoctorId = reader.GetInt32("DoctorId"),
                                DoctorName = reader.IsDBNull("DoctorName") ? "" : reader.GetString("DoctorName")
                            };
                            
                            return invoice;
                        }
                    }
                }
            }
            
            return null;
        }

        public void LogDrugCalculation(int patientId, int invoiceId, string drugName, 
                                     string instructions, string calculatedBy)
        {
            using (var connection = new SqlConnection(_connectionString))
            {
                connection.Open();
                
                var query = @"
                    INSERT INTO Invoice_Items (InvoiceId, Description, Quantity, Price, Category, Notes)
                    VALUES (@InvoiceId, @Description, 1, 0, 'Drug Calculation', @Notes)";

                using (var command = new SqlCommand(query, connection))
                {
                    command.Parameters.AddWithValue("@InvoiceId", invoiceId);
                    command.Parameters.AddWithValue("@Description", $"VetDrugs Calculation: {drugName}");
                    command.Parameters.AddWithValue("@Notes", 
                        $"{instructions}\nCalculated by: {calculatedBy}\nTimestamp: {DateTime.Now:yyyy-MM-dd HH:mm:ss}");
                    
                    command.ExecuteNonQuery();
                }
            }
        }
    }
}
```

### 4. VetDrugs Integration Service

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using VetDrugs.SDK; // Using the C# SDK we created earlier
using AvimarkVetDrugs.Models;
using AvimarkVetDrugs.Data;

namespace AvimarkVetDrugs.Services
{
    public class AvimarkVetDrugsService
    {
        private readonly VetDrugsApiClient _vetDrugsClient;
        private readonly AvimarkDataAccess _avimarkData;

        public AvimarkVetDrugsService(string vetDrugsApiKey, string avimarkConnectionString)
        {
            _vetDrugsClient = new VetDrugsApiClient(vetDrugsApiKey);
            _avimarkData = new AvimarkDataAccess(avimarkConnectionString);
        }

        public async Task<List<DrugCalculation>> CalculateDrugsForPatient(
            int patientId, 
            List<string> drugNames, 
            string calculatedBy)
        {
            try
            {
                // Get patient from Avimark
                var patient = _avimarkData.GetPatient(patientId);
                if (patient == null)
                {
                    throw new ArgumentException($"Patient {patientId} not found in Avimark");
                }

                // Convert to VetDrugs format
                var vetDrugsPatient = new Patient
                {
                    WeightKg = patient.WeightKg,
                    Species = patient.VetDrugsSpecies
                };

                var drugRequests = drugNames.Select(name => new DrugRequest
                {
                    DrugName = name
                }).ToList();

                var request = new CalculationRequest
                {
                    Patient = vetDrugsPatient,
                    Drugs = drugRequests
                };

                // Calculate using VetDrugs API
                var response = await _vetDrugsClient.CalculateDrugsAsync(request);

                // Log calculations to current invoice
                var currentInvoice = _avimarkData.GetCurrentInvoice(patientId);
                if (currentInvoice != null)
                {
                    foreach (var calc in response.Calculations)
                    {
                        _avimarkData.LogDrugCalculation(
                            patientId, 
                            currentInvoice.InvoiceId, 
                            calc.DrugName, 
                            calc.Instructions, 
                            calculatedBy);
                    }
                }

                return response.Calculations;
            }
            catch (Exception ex)
            {
                throw new Exception($"Drug calculation failed: {ex.Message}", ex);
            }
        }

        public async Task<List<DrugInfo>> SearchDrugs(string searchTerm)
        {
            try
            {
                var response = await _vetDrugsClient.SearchDrugsAsync(searchTerm);
                return response.Drugs;
            }
            catch (Exception ex)
            {
                throw new Exception($"Drug search failed: {ex.Message}", ex);
            }
        }

        public List<AvimarkPatient> SearchPatients(string searchTerm)
        {
            return _avimarkData.SearchPatients(searchTerm);
        }

        public AvimarkPatient GetPatient(int patientId)
        {
            return _avimarkData.GetPatient(patientId);
        }

        public void Dispose()
        {
            _vetDrugsClient?.Dispose();
        }
    }
}
```

### 5. Windows Forms Application

```csharp
using System;
using System.Collections.Generic;
using System.Drawing;
using System.Linq;
using System.Threading.Tasks;
using System.Windows.Forms;
using AvimarkVetDrugs.Services;
using AvimarkVetDrugs.Models;

namespace AvimarkVetDrugs.UI
{
    public partial class DrugCalculatorForm : Form
    {
        private AvimarkVetDrugsService _service;
        private TextBox txtPatientSearch;
        private ListBox lstPatients;
        private Label lblSelectedPatient;
        private CheckedListBox clbDrugs;
        private Button btnCalculate;
        private DataGridView dgvResults;
        private TextBox txtCustomDrug;
        private Button btnAddCustomDrug;
        private ProgressBar progressBar;

        public DrugCalculatorForm()
        {
            InitializeComponent();
            InitializeService();
            LoadCommonDrugs();
        }

        private void InitializeService()
        {
            try
            {
                var vetDrugsApiKey = ConfigurationManager.AppSettings["VetDrugsApiKey"];
                var avimarkConnectionString = ConfigurationManager.ConnectionStrings["AvimarkDB"].ConnectionString;
                
                _service = new AvimarkVetDrugsService(vetDrugsApiKey, avimarkConnectionString);
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Failed to initialize service: {ex.Message}", "Configuration Error");
                this.Close();
            }
        }

        private void InitializeComponent()
        {
            this.Size = new Size(1000, 700);
            this.Text = "Avimark VetDrugs Calculator";
            this.StartPosition = FormStartPosition.CenterScreen;

            // Patient search section
            var lblPatientSearch = new Label 
            { 
                Text = "Search Patient:", 
                Location = new Point(10, 15), 
                Size = new Size(100, 20) 
            };

            txtPatientSearch = new TextBox 
            { 
                Location = new Point(120, 12), 
                Size = new Size(200, 23) 
            };
            txtPatientSearch.TextChanged += TxtPatientSearch_TextChanged;

            lstPatients = new ListBox 
            { 
                Location = new Point(120, 40), 
                Size = new Size(200, 100),
                DisplayMember = "Name"
            };
            lstPatients.SelectedIndexChanged += LstPatients_SelectedIndexChanged;

            lblSelectedPatient = new Label 
            { 
                Location = new Point(10, 150), 
                Size = new Size(400, 40),
                Text = "No patient selected",
                ForeColor = Color.Blue
            };

            // Drug selection section
            var lblDrugs = new Label 
            { 
                Text = "Select Drugs:", 
                Location = new Point(350, 15), 
                Size = new Size(100, 20) 
            };

            clbDrugs = new CheckedListBox 
            { 
                Location = new Point(350, 40), 
                Size = new Size(200, 150),
                CheckOnClick = true
            };

            // Custom drug input
            txtCustomDrug = new TextBox 
            { 
                Location = new Point(570, 40), 
                Size = new Size(150, 23),
                PlaceholderText = "Enter custom drug..."
            };

            btnAddCustomDrug = new Button 
            { 
                Text = "Add Drug", 
                Location = new Point(730, 38), 
                Size = new Size(80, 27) 
            };
            btnAddCustomDrug.Click += BtnAddCustomDrug_Click;

            // Calculate button
            btnCalculate = new Button 
            { 
                Text = "Calculate Dosages", 
                Location = new Point(350, 200), 
                Size = new Size(150, 35),
                BackColor = Color.LightBlue
            };
            btnCalculate.Click += BtnCalculate_Click;

            // Progress bar
            progressBar = new ProgressBar 
            { 
                Location = new Point(510, 205), 
                Size = new Size(200, 25),
                Visible = false
            };

            // Results grid
            dgvResults = new DataGridView 
            { 
                Location = new Point(10, 250), 
                Size = new Size(960, 400),
                AutoSizeColumnsMode = DataGridViewAutoSizeColumnsMode.Fill,
                ReadOnly = true,
                AllowUserToAddRows = false
            };
            SetupResultsGrid();

            // Add all controls
            this.Controls.AddRange(new Control[] 
            { 
                lblPatientSearch, txtPatientSearch, lstPatients, lblSelectedPatient,
                lblDrugs, clbDrugs, txtCustomDrug, btnAddCustomDrug,
                btnCalculate, progressBar, dgvResults 
            });
        }

        private void SetupResultsGrid()
        {
            dgvResults.Columns.Add("DrugName", "Drug");
            dgvResults.Columns.Add("Instructions", "Instructions");
            dgvResults.Columns.Add("Dose", "Dose (mg/kg)");
            dgvResults.Columns.Add("TotalDose", "Total Dose");
            dgvResults.Columns.Add("Volume", "Volume");
            dgvResults.Columns.Add("Route", "Route");
            dgvResults.Columns.Add("Frequency", "Frequency");
            dgvResults.Columns.Add("Status", "Status");
            dgvResults.Columns.Add("Warnings", "Warnings");
        }

        private void LoadCommonDrugs()
        {
            var commonDrugs = new[]
            {
                "Cephalexin", "Amoxicillin", "Enrofloxacin", "Doxycycline",
                "Carprofen", "Meloxicam", "Tramadol", "Gabapentin",
                "Acepromazine", "Trazodone", "Alprazolam"
            };

            clbDrugs.Items.AddRange(commonDrugs);
        }

        private async void TxtPatientSearch_TextChanged(object sender, EventArgs e)
        {
            if (txtPatientSearch.Text.Length >= 3)
            {
                try
                {
                    var patients = _service.SearchPatients(txtPatientSearch.Text);
                    lstPatients.DataSource = patients;
                }
                catch (Exception ex)
                {
                    MessageBox.Show($"Patient search failed: {ex.Message}");
                }
            }
        }

        private void LstPatients_SelectedIndexChanged(object sender, EventArgs e)
        {
            if (lstPatients.SelectedItem is AvimarkPatient patient)
            {
                lblSelectedPatient.Text = 
                    $"Selected: {patient.Name} - {patient.WeightKg:F1}kg {patient.VetDrugsSpecies} (ID: {patient.PatientId})";
                btnCalculate.Enabled = true;
            }
        }

        private void BtnAddCustomDrug_Click(object sender, EventArgs e)
        {
            if (!string.IsNullOrWhiteSpace(txtCustomDrug.Text))
            {
                clbDrugs.Items.Add(txtCustomDrug.Text.Trim(), true);
                txtCustomDrug.Clear();
            }
        }

        private async void BtnCalculate_Click(object sender, EventArgs e)
        {
            if (!(lstPatients.SelectedItem is AvimarkPatient selectedPatient))
            {
                MessageBox.Show("Please select a patient first.");
                return;
            }

            var selectedDrugs = clbDrugs.CheckedItems.Cast<string>().ToList();
            if (selectedDrugs.Count == 0)
            {
                MessageBox.Show("Please select at least one drug.");
                return;
            }

            btnCalculate.Enabled = false;
            progressBar.Visible = true;
            progressBar.Style = ProgressBarStyle.Marquee;
            dgvResults.Rows.Clear();

            try
            {
                var calculations = await _service.CalculateDrugsForPatient(
                    selectedPatient.PatientId, 
                    selectedDrugs, 
                    Environment.UserName);

                DisplayResults(calculations);
                
                MessageBox.Show($"Successfully calculated dosages for {calculations.Count} drugs.\n" +
                               "Results have been logged to the current invoice.", 
                               "Calculation Complete", MessageBoxButtons.OK, MessageBoxIcon.Information);
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Calculation failed: {ex.Message}", 
                              "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
            finally
            {
                btnCalculate.Enabled = true;
                progressBar.Visible = false;
            }
        }

        private void DisplayResults(List<DrugCalculation> calculations)
        {
            foreach (var calc in calculations)
            {
                var row = new DataGridViewRow();
                row.CreateCells(dgvResults);

                row.Cells[0].Value = calc.DrugName;
                row.Cells[1].Value = calc.Instructions;
                row.Cells[2].Value = $"{calc.DosePerKg:F2} {calc.DoseUnit}/kg";
                row.Cells[3].Value = $"{calc.TotalDose:F2} {calc.DoseUnit}";
                row.Cells[4].Value = $"{calc.Volume:F2} {calc.VolumeUnit}";
                row.Cells[5].Value = calc.Route;
                row.Cells[6].Value = calc.Frequency;
                row.Cells[7].Value = GetStatusDisplay(calc.DoseRangeStatus);
                row.Cells[8].Value = calc.HasWarnings ? string.Join(", ", calc.Warnings) : "None";

                // Color coding
                switch (calc.DoseRangeStatus)
                {
                    case "within_range":
                        row.DefaultCellStyle.BackColor = Color.LightGreen;
                        break;
                    case "below_range":
                        row.DefaultCellStyle.BackColor = Color.LightYellow;
                        break;
                    case "above_range":
                    case "contraindicated":
                        row.DefaultCellStyle.BackColor = Color.LightCoral;
                        break;
                }

                dgvResults.Rows.Add(row);
            }
        }

        private string GetStatusDisplay(string status)
        {
            return status switch
            {
                "within_range" => "✓ Normal",
                "below_range" => "⚠ Below Range",
                "above_range" => "⚠ Above Range",
                "contraindicated" => "❌ Contraindicated",
                _ => status
            };
        }

        protected override void OnFormClosed(FormClosedEventArgs e)
        {
            _service?.Dispose();
            base.OnFormClosed(e);
        }
    }
}
```

### 6. Configuration Files

#### App.config

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <appSettings>
    <add key="VetDrugsApiKey" value="vd_your_api_key_here" />
  </appSettings>
  
  <connectionStrings>
    <add name="AvimarkDB" 
         connectionString="Server=AVIMARK-SERVER;Database=AvimarkData;Integrated Security=true;" 
         providerName="System.Data.SqlClient" />
  </connectionStrings>
  
  <startup>
    <supportedRuntime version="v4.0" sku=".NETFramework,Version=v4.7.2" />
  </startup>
</configuration>
```

## SQL Server Stored Procedures

Create helpful stored procedures for common operations:

```sql
-- Get patient with recent medications
CREATE PROCEDURE sp_GetPatientWithMedications
    @PatientId INT
AS
BEGIN
    SELECT 
        p.PatientId,
        p.Name,
        p.Weight,
        p.WeightUnit,
        p.Species,
        p.Breed
    FROM Patients p
    WHERE p.PatientId = @PatientId;
    
    -- Recent medications
    SELECT TOP 10
        ii.Description,
        ii.Quantity,
        i.Date
    FROM Invoice_Items ii
    INNER JOIN Invoices i ON ii.InvoiceId = i.InvoiceId
    WHERE i.PatientId = @PatientId
      AND ii.Category LIKE '%medication%'
    ORDER BY i.Date DESC;
END

-- Log VetDrugs calculation
CREATE PROCEDURE sp_LogVetDrugsCalculation
    @PatientId INT,
    @InvoiceId INT,
    @DrugName NVARCHAR(100),
    @Instructions NVARCHAR(500),
    @CalculatedBy NVARCHAR(100)
AS
BEGIN
    INSERT INTO Invoice_Items (
        InvoiceId, 
        Description, 
        Quantity, 
        Price, 
        Category, 
        Notes,
        CreatedDate,
        CreatedBy
    )
    VALUES (
        @InvoiceId,
        'VetDrugs Calculation: ' + @DrugName,
        1,
        0,
        'Drug Calculation',
        @Instructions,
        GETDATE(),
        @CalculatedBy
    );
END
```

## Report Generation

Create reports for drug calculation usage:

```sql
-- VetDrugs usage report
SELECT 
    d.Name as Doctor,
    COUNT(*) as CalculationsPerformed,
    COUNT(DISTINCT i.PatientId) as UniquePatients,
    MIN(i.Date) as FirstCalculation,
    MAX(i.Date) as LastCalculation
FROM Invoice_Items ii
INNER JOIN Invoices i ON ii.InvoiceId = i.InvoiceId
INNER JOIN Doctors d ON i.DoctorId = d.DoctorId
WHERE ii.Description LIKE 'VetDrugs Calculation:%'
  AND i.Date >= DATEADD(month, -3, GETDATE())
GROUP BY d.Name
ORDER BY CalculationsPerformed DESC;

-- Most calculated drugs
SELECT 
    SUBSTRING(ii.Description, 23, 100) as DrugName,
    COUNT(*) as TimesCalculated,
    COUNT(DISTINCT i.PatientId) as UniquePatients
FROM Invoice_Items ii
INNER JOIN Invoices i ON ii.InvoiceId = i.InvoiceId
WHERE ii.Description LIKE 'VetDrugs Calculation:%'
  AND i.Date >= DATEADD(month, -1, GETDATE())
GROUP BY SUBSTRING(ii.Description, 23, 100)
ORDER BY TimesCalculated DESC;
```

## Deployment and Installation

### 1. Prerequisites Installation

```powershell
# Install .NET Framework 4.7.2 or higher
# Download from Microsoft website

# Verify SQL Server connectivity
sqlcmd -S AVIMARK-SERVER -E -Q "SELECT @@VERSION"
```

### 2. Application Deployment

```batch
REM Create application directory
mkdir "C:\Program Files\AvimarkVetDrugs"

REM Copy application files
copy AvimarkVetDrugs.exe "C:\Program Files\AvimarkVetDrugs\"
copy AvimarkVetDrugs.exe.config "C:\Program Files\AvimarkVetDrugs\"
copy *.dll "C:\Program Files\AvimarkVetDrugs\"

REM Create desktop shortcut
echo [InternetShortcut] > "%USERPROFILE%\Desktop\Avimark VetDrugs.lnk"
echo URL=file:///C:/Program Files/AvimarkVetDrugs/AvimarkVetDrugs.exe >> "%USERPROFILE%\Desktop\Avimark VetDrugs.lnk"
```

### 3. Database Setup

```sql
-- Grant read access to application user
GRANT SELECT ON Patients TO [DOMAIN\AvimarkVetDrugsUser];
GRANT SELECT ON Invoices TO [DOMAIN\AvimarkVetDrugsUser];
GRANT SELECT ON Invoice_Items TO [DOMAIN\AvimarkVetDrugsUser];
GRANT SELECT ON Doctors TO [DOMAIN\AvimarkVetDrugsUser];

-- Grant insert access for logging
GRANT INSERT ON Invoice_Items TO [DOMAIN\AvimarkVetDrugsUser];
```

## Troubleshooting

### Common Issues

1. **Database Connection Problems**
   ```csharp
   // Test database connectivity
   try
   {
       using (var connection = new SqlConnection(connectionString))
       {
           connection.Open();
           Console.WriteLine("Database connection successful");
       }
   }
   catch (Exception ex)
   {
       Console.WriteLine($"Database connection failed: {ex.Message}");
   }
   ```

2. **Weight Unit Conversion**
   ```csharp
   // Handle various weight units from Avimark
   public double ConvertToKg(decimal weight, string unit)
   {
       return unit?.ToLower() switch
       {
           "lb" or "lbs" or "pounds" => (double)(weight * 0.453592m),
           "g" or "grams" => (double)(weight / 1000m),
           "oz" or "ounces" => (double)(weight * 0.0283495m),
           _ => (double)weight // Assume kg
       };
   }
   ```

3. **Species Mapping**
   ```csharp
   public string MapSpeciesToVetDrugs(string avimarkSpecies)
   {
       var species = avimarkSpecies?.ToLower() ?? "";
       
       if (species.Contains("dog") || species.Contains("canine") || 
           species.Contains("pup") || species.Contains("retriever") ||
           species.Contains("shepherd") || species.Contains("terrier"))
           return "dog";
           
       if (species.Contains("cat") || species.Contains("feline") || 
           species.Contains("kitten"))
           return "cat";
           
       return "dog"; // Default fallback
   }
   ```

## Support and Maintenance

### Logging Implementation

```csharp
public class Logger
{
    private static readonly string LogPath = Path.Combine(
        Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData),
        "AvimarkVetDrugs", "logs");

    static Logger()
    {
        Directory.CreateDirectory(LogPath);
    }

    public static void LogInfo(string message)
    {
        WriteLog("INFO", message);
    }

    public static void LogError(string message, Exception ex = null)
    {
        var errorMessage = ex != null ? $"{message}: {ex}" : message;
        WriteLog("ERROR", errorMessage);
    }

    private static void WriteLog(string level, string message)
    {
        var logFile = Path.Combine(LogPath, $"avimark-vetdrugs-{DateTime.Now:yyyy-MM-dd}.log");
        var logEntry = $"{DateTime.Now:yyyy-MM-dd HH:mm:ss} [{level}] {message}";
        
        File.AppendAllText(logFile, logEntry + Environment.NewLine);
    }
}
```

### Performance Monitoring

```csharp
public class PerformanceMonitor
{
    public static async Task<T> MonitorAsync<T>(string operation, Func<Task<T>> func)
    {
        var stopwatch = Stopwatch.StartNew();
        try
        {
            var result = await func();
            stopwatch.Stop();
            Logger.LogInfo($"{operation} completed in {stopwatch.ElapsedMilliseconds}ms");
            return result;
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            Logger.LogError($"{operation} failed after {stopwatch.ElapsedMilliseconds}ms", ex);
            throw;
        }
    }
}
```

## Next Steps

- [View other PMI integrations](../integrations/)
- [Explore C# examples](../examples/csharp.md)
- [Check API reference](../api-reference/)
- [Review support documentation](../support.md)
