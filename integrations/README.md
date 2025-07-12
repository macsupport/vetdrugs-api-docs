# Integration Guides

Complete integration guides for major Practice Management Information (PMI) systems.

## Overview

The VetDrugs API is designed to integrate seamlessly with veterinary practice management software. We provide comprehensive integration guides for the most popular PMI systems used in veterinary practices.

## Supported PMI Systems

### Windows-Based Systems

**[Cornerstone](cornerstone.md)**
- **Platform**: Windows/.NET Framework
- **Database**: SQL Server
- **Integration**: Direct API calls with .NET SDK
- **Features**: Real-time calculations, invoice logging, drug search
- **Complexity**: ⭐⭐⭐ Moderate
- **Setup Time**: 2-4 hours

**[Avimark](avimark.md)** 
- **Platform**: Windows with SQL Server backend
- **Database**: Direct SQL Server access
- **Integration**: Custom .NET application + database queries
- **Features**: Patient lookup, calculation logging, custom UI
- **Complexity**: ⭐⭐⭐⭐ Advanced
- **Setup Time**: 4-8 hours

### Cloud-Based Systems

**[ezyVet](ezyvet.md)**
- **Platform**: Cloud-based REST API
- **Database**: ezyVet cloud database
- **Integration**: OAuth + REST API calls
- **Features**: Real-time sync, webhook support, consultation logging
- **Complexity**: ⭐⭐⭐ Moderate
- **Setup Time**: 3-6 hours

## Integration Architecture

```
PMI System → Custom Integration → VetDrugs API → Drug Calculations
     ↓              ↓                   ↓              ↓
 Patient Data → API Formatting → Dose Calculation → Results Display
```

## Common Integration Patterns

### 1. Real-Time Calculations
- User selects patient and drugs in PMI
- Integration calls VetDrugs API in real-time
- Results displayed immediately in PMI interface
- Calculations logged to patient record

### 2. Batch Processing
- Queue multiple calculations
- Process during low-usage periods
- Bulk import results into PMI
- Generate summary reports

### 3. Background Sync
- Continuous sync of drug database
- Pre-calculate common drug combinations
- Cache results for faster response
- Update calculations when formulations change

## Pre-Integration Checklist

### Technical Requirements

**API Access**:
- ✅ VetDrugs API key obtained
- ✅ Network connectivity to API endpoints
- ✅ HTTPS/SSL support enabled
- ✅ JSON parsing capabilities

**PMI System Access**:
- ✅ Administrative access to PMI system
- ✅ Database credentials (if applicable)
- ✅ API credentials (for cloud systems)
- ✅ Development/testing environment

**Development Environment**:
- ✅ Programming language SDK available
- ✅ HTTP client library installed
- ✅ Error handling framework
- ✅ Logging and monitoring tools

### Business Requirements

**Clinical Workflow**:
- ✅ Define when calculations are triggered
- ✅ Identify required drug calculation types
- ✅ Determine where results are displayed
- ✅ Plan for error handling and edge cases

**Data Flow**:
- ✅ Map PMI patient data to VetDrugs format
- ✅ Define calculation result storage
- ✅ Plan audit trail and logging
- ✅ Consider data backup and recovery

## Integration Support

### Development Assistance

**Free Support** (All customers):
- Integration architecture guidance
- API implementation examples
- Troubleshooting common issues
- Documentation and best practices

**Premium Support** (Enterprise customers):
- Dedicated technical account manager
- Custom code review and optimization
- Priority support with 4-hour response
- Direct developer consultation calls

### Professional Services

**Custom Integration Development**:
- Full integration development
- Testing and quality assurance
- Staff training and documentation
- Ongoing maintenance and support

**Pricing**: Contact [enterprise@vetdrugscalculators.com](mailto:enterprise@vetdrugscalculators.com)

## Getting Started

### Step 1: Choose Your PMI System
Select your practice management system from the guides above.

### Step 2: Review Prerequisites
Each guide includes specific technical and business requirements.

### Step 3: Development Environment
Set up your development environment following the guide instructions.

### Step 4: API Testing
Test basic API connectivity and authentication before full integration.

### Step 5: Implementation
Follow the step-by-step integration guide for your PMI system.

### Step 6: Testing & Deployment
Thoroughly test in a development environment before production deployment.

## Additional Resources

**Code Examples**: [View language-specific examples](../examples/)
- [JavaScript](../examples/javascript.md)
- [C# (.NET)](../examples/csharp.md)
- [Python](../examples/python.md)
- [PHP](../examples/php.md)

**API Reference**: [Complete API documentation](../api-reference/)
- [Drug Calculations](../api-reference/calculate.md)
- [Drug Search](../api-reference/search.md)
- [Error Handling](../api-reference/errors.md)
- [Rate Limits](../api-reference/rate-limits.md)

**Support**: [Get help with integration](../support.md)
- Technical support channels
- Common troubleshooting solutions
- Enterprise support options

## Community & Feedback

**Feature Requests**: We're constantly improving our integrations based on user feedback. Contact us with:
- New PMI systems to support
- Additional integration features
- Workflow improvements
- Performance optimizations

**Success Stories**: Share your integration success with the community to help other practices implement similar solutions.

---

**Ready to integrate?** Choose your PMI system above and start with the comprehensive integration guide.

**Need help?** Contact [support@vetdrugscalculators.com](mailto:support@vetdrugscalculators.com) for assistance.
