# Changelog

All notable changes to the VetDrugs API are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [2.2.0-enhanced] - 2025-07-11

### 🚀 Added
- **New Drug Information Endpoint**: `GET /api/drug-info/{drug_name}` provides detailed drug information including contraindications, side effects, and drug interactions
- **Enhanced Search Capabilities**: Fuzzy matching and multi-word search support in `/api/drugs`
- **Rate Limit Headers**: All responses now include detailed rate limiting information
- **Burst Rate Limiting**: Short-term burst protection alongside per-minute and per-hour limits
- **Performance Monitoring**: Sub-150ms global response times with enhanced Cloudflare optimization
- **Extended Drug Database**: Added 50+ new drugs including exotic animal medications
- **Species-Specific Dosing**: Enhanced feline dosing calculations for 30+ drugs
- **Drug Categories**: Improved categorization with subcategories for better filtering

### 🔧 Improved
- **Response Format Consistency**: Standardized all API responses with consistent metadata structure
- **Error Messages**: More descriptive error messages with specific validation guidance
- **Documentation**: Complete API documentation with interactive examples
- **Caching Strategy**: Improved KV store caching with 99.9% cache hit rate
- **Search Performance**: 40% faster drug search with optimized indexing
- **Global Distribution**: Enhanced CDN performance for international users

### 🐛 Fixed
- **Weight Conversion**: Fixed edge cases in pounds-to-kilograms conversion
- **Species Validation**: Improved species name normalization and validation
- **Concentration Handling**: Better handling of custom drug concentrations
- **Unicode Support**: Proper handling of special characters in drug names
- **Rate Limit Reset**: Fixed rate limit window calculation edge cases

### 🔄 Changed
- **API Versioning**: Added version information to all response metadata
- **Default Concentrations**: Updated default concentrations for 15+ drugs based on current formulations
- **Request Validation**: Enhanced validation with more specific error messages
- **Response Times**: Improved average response time from 200ms to 120ms

---

## [2.1.3] - 2025-06-15

### 🔧 Improved
- **Database Updates**: Added 25 new veterinary drugs to the calculation database
- **Dose Range Validation**: Enhanced dose range checking with more granular warnings
- **API Response Time**: Reduced average response time by 15% through caching optimizations

### 🐛 Fixed
- **Feline Dosing**: Fixed calculation errors for cat-specific dosing protocols
- **Route Validation**: Improved validation for administration routes (IV, IM, SC, PO, etc.)
- **Concentration Units**: Fixed unit conversion issues for mg/ml vs mg/cap formulations

---

## [2.1.2] - 2025-05-20

### 🚀 Added
- **Health Check Endpoint**: New `/api/health` endpoint for monitoring API status
- **Request Logging**: Enhanced logging for better troubleshooting and analytics
- **Documentation**: Initial API documentation portal

### 🔧 Improved
- **Error Handling**: More consistent error response format across all endpoints
- **Validation**: Better input validation with specific error messages for common mistakes
- **Performance**: Database query optimization reducing response times by 10%

### 🐛 Fixed
- **Species Case Sensitivity**: Fixed issues where species names were case-sensitive
- **Empty Drug Lists**: Better handling of requests with empty drug arrays
- **Timeout Handling**: Improved timeout handling for slow network connections

---

## [2.1.1] - 2025-04-18

### 🔧 Improved
- **Drug Search**: Enhanced search algorithm with partial matching capabilities
- **API Stability**: Reduced API error rate from 0.1% to 0.05%
- **Response Format**: More consistent JSON structure across all endpoints

### 🐛 Fixed
- **Calculation Precision**: Fixed floating-point precision issues in dose calculations
- **API Key Validation**: Improved API key format validation and error messages
- **CORS Headers**: Fixed CORS issues for browser-based integrations

---

## [2.1.0] - 2025-03-22

### 🚀 Added
- **Advanced Drug Search**: New `/api/drugs` endpoint with category filtering and search capabilities
- **Custom Concentrations**: Support for custom drug concentrations in calculations
- **Bulk Calculations**: Ability to calculate multiple drugs in a single API request
- **Enhanced Metadata**: Response metadata including calculation timestamps and API version

### 🔧 Improved
- **Rate Limiting**: Implemented sophisticated rate limiting with per-endpoint limits
- **Drug Database**: Expanded database to 500+ veterinary drugs
- **Calculation Accuracy**: Enhanced calculation algorithms for improved accuracy
- **API Documentation**: Comprehensive endpoint documentation with examples

### 🐛 Fixed
- **Memory Usage**: Optimized memory usage reducing cold start times by 30%
- **Edge Cases**: Fixed calculation edge cases for very small or large animals
- **Input Sanitization**: Improved input sanitization and validation

### 🔄 Changed
- **Response Structure**: Updated response format for better consistency (backward compatible)
- **Error Codes**: Standardized error codes across all endpoints

---

## [2.0.0] - 2025-02-10

### 🚀 Added
- **RESTful API**: Complete redesign as a RESTful API service
- **Cloud Infrastructure**: Migrated to Cloudflare Workers for global performance
- **API Authentication**: Secure API key-based authentication system
- **Rate Limiting**: Comprehensive rate limiting to ensure fair usage
- **Multiple Species**: Support for both canine and feline calculations
- **Drug Validation**: Real-time drug name validation and suggestions

### 🔧 Improved
- **Performance**: Sub-200ms global response times
- **Reliability**: 99.9% uptime with automatic failover
- **Scalability**: Auto-scaling infrastructure handling 1M+ requests/day
- **Security**: Enhanced security with API key management and request validation

### 🔄 Changed
- **Architecture**: Complete rewrite from mobile app to API service
- **Data Format**: JSON-based request/response format
- **Access Model**: Subscription-based API access replacing one-time app purchase

### ⚠️ Breaking Changes
- **API Format**: New REST API format incompatible with previous mobile app integrations
- **Authentication**: Requires API key for all requests
- **Endpoints**: All new endpoint structure and naming conventions

---

## [1.5.2] - 2024-12-15

### 🐛 Fixed
- **iOS Compatibility**: Fixed compatibility issues with iOS 17
- **Database Updates**: Updated drug database with latest formulations
- **UI Improvements**: Minor user interface enhancements

---

## [1.5.1] - 2024-11-08

### 🔧 Improved
- **Calculation Engine**: Enhanced calculation precision for edge cases
- **User Interface**: Improved user experience with better error messages
- **Performance**: Faster app startup and calculation times

### 🐛 Fixed
- **Weight Input**: Fixed decimal point input issues on some devices
- **Drug Selection**: Improved drug autocomplete functionality
- **Memory Management**: Fixed memory leaks in calculation engine

---

## [1.5.0] - 2024-10-12

### 🚀 Added
- **Feline Support**: Added comprehensive cat-specific drug calculations
- **Drug Interactions**: Basic drug interaction warnings
- **Calculation History**: Local storage of previous calculations
- **Export Feature**: Ability to export calculations as PDF

### 🔧 Improved
- **Drug Database**: Expanded to 400+ drugs with updated dosing information
- **UI/UX**: Modern interface redesign for better usability
- **Offline Support**: Basic offline functionality for previously calculated drugs

---

## [1.4.1] - 2024-08-20

### 🐛 Fixed
- **Critical Bug**: Fixed calculation errors for drugs with multiple formulations
- **Data Sync**: Improved drug database synchronization
- **Crash Issues**: Fixed app crashes on certain device configurations

---

## [1.4.0] - 2024-07-15

### 🚀 Added
- **Advanced Calculations**: Support for CRI (Constant Rate Infusion) calculations
- **Fluid Therapy**: Comprehensive fluid therapy calculations
- **Drug Categories**: Organized drugs by therapeutic categories
- **Search Enhancement**: Improved drug search with filters

### 🔧 Improved
- **Calculation Speed**: 50% faster calculation processing
- **Database Size**: Expanded to 350+ veterinary drugs
- **User Interface**: Enhanced visual design and navigation

---

## [1.3.0] - 2024-05-18

### 🚀 Added
- **Multiple Formulations**: Support for different drug concentrations
- **Route Specification**: Specific calculations for IV, IM, SC, PO routes
- **Dosing Warnings**: Automatic warnings for out-of-range doses
- **Unit Conversion**: Automatic conversion between metric and imperial units

### 🔧 Improved
- **Accuracy**: Enhanced calculation algorithms for better precision
- **Drug Data**: Updated drug information with latest veterinary guidelines
- **Performance**: Optimized app performance and reduced battery usage

---

## [1.2.0] - 2024-03-22

### 🚀 Added
- **Drug Database**: Expanded to 300+ commonly used veterinary drugs
- **Frequency Options**: Multiple dosing frequency options (SID, BID, TID, QID)
- **Custom Doses**: Ability to enter custom dose ranges
- **Results History**: View and compare previous calculations

### 🔧 Improved
- **User Interface**: Streamlined interface for faster drug calculations
- **Data Validation**: Enhanced input validation and error handling
- **Help System**: Comprehensive in-app help and tutorials

---

## [1.1.0] - 2024-01-15

### 🚀 Added
- **Weight Units**: Support for both kilograms and pounds
- **Drug Search**: Searchable drug database with autocomplete
- **Calculation Notes**: Ability to add notes to calculations
- **Settings**: Customizable app settings and preferences

### 🔧 Improved
- **Calculation Engine**: More accurate dosing calculations
- **Database**: Updated drug information and dosing protocols
- **Performance**: Faster app loading and smoother navigation

---

## [1.0.0] - 2023-11-20

### 🚀 Initial Release
- **Core Calculations**: Basic veterinary drug dosage calculations
- **Essential Drugs**: 200+ commonly used veterinary drugs
- **Simple Interface**: Easy-to-use interface for quick calculations
- **Dose Validation**: Basic dose range validation and warnings
- **iOS Support**: Initial release for iOS devices

---

## Upcoming Releases

### [2.3.0] - Planned Q3 2025

**🚀 New Features**:
- **Drug Interaction Checker**: Advanced drug interaction detection and warnings
- **Custom Protocols**: Ability to create and save custom dosing protocols
- **Batch Calculations**: Enhanced bulk calculation capabilities
- **WebSocket Support**: Real-time calculations for integrated systems
- **Mobile SDK**: Native iOS and Android SDKs

**🔧 Improvements**:
- **AI-Powered Suggestions**: Machine learning-based dosing recommendations
- **Enhanced Analytics**: Detailed usage analytics and reporting
- **Multi-language Support**: Initial support for Spanish and French
- **Performance**: Target <100ms global response times

### [2.4.0] - Planned Q4 2025

**🚀 New Features**:
- **Exotic Animal Support**: Dosing calculations for exotic and wildlife species
- **Integration Hub**: Pre-built integrations for major PMI systems
- **Advanced Reporting**: Comprehensive calculation reporting and analytics
- **Audit Trail**: Complete audit logging for regulatory compliance

---

## Migration Guides

### Upgrading from v2.1.x to v2.2.0

**No Breaking Changes**: This is a backward-compatible release.

**New Features Available**:
- Use the new `/api/drug-info/{drug_name}` endpoint for detailed drug information
- Take advantage of improved search capabilities with fuzzy matching
- Monitor your rate limit usage with the new headers

**Recommended Updates**:
```javascript
// Before (v2.1.x)
const response = await fetch('/api/drugs?search=exact_name');

// After (v2.2.0) - fuzzy matching now available
const response = await fetch('/api/drugs?search=partial_name');
// Will find drugs even with slight spelling variations
```

### Upgrading from v2.0.x to v2.1.0

**No Breaking Changes**: All v2.0.x integrations continue to work.

**New Features**:
- New `/api/drugs` endpoint for drug search
- Enhanced metadata in all responses
- Support for custom concentrations

**Migration Steps**:
1. Update your application to handle new metadata fields (optional)
2. Consider using the new search endpoint for better UX
3. Test custom concentration features if applicable

### Upgrading from v1.x to v2.0.0

**⚠️ Breaking Changes**: Complete API redesign.

This is a major version upgrade requiring code changes. See our [Migration Guide](migration-guide.md) for detailed instructions.

---

## Support

For questions about any release or help with migration:
- 📧 Email: [support@vetdrugscalculators.com](mailto:support@vetdrugscalculators.com)
- 📚 Documentation: [docs.vetdrugscalculators.com](https://docs.vetdrugscalculators.com)
- 🐛 Bug Reports: [GitHub Issues](https://github.com/vetdrugs/api-issues)

---

**Stay Updated**: Email [support@vetdrugscalculators.com](mailto:support@vetdrugscalculators.com) to subscribe to release announcements and API updates.
