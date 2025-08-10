# FHIR Data Pipeline with N8N

A robust, automated data pipeline for extracting, transforming, and loading FHIR patient data from healthcare APIs into a PostgreSQL database with comprehensive data quality assessment and audit logging.

## 🎯 Project Goals

- **Automated FHIR Data Extraction**: Continuously sync patient data from SMART on FHIR servers
- **Data Quality Assessment**: Implement comprehensive validation and scoring for healthcare data
- **Scalable Processing**: Handle large volumes of patient records with pagination support
- **Audit Trail**: Complete logging and monitoring of all data operations
- **Error Resilience**: Graceful handling of duplicates, API failures, and data quality issues

## 🏗️ Architecture Overview

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────────┐
│   FHIR Server   │    │   N8N Workflow   │    │   PostgreSQL DB     │
│                 │───▶│                  │───▶│                     │
│ SMART on FHIR   │    │ Data Pipeline    │    │ • Patient Records   │
│ Test Server     │    │                  │    │ • Quality Metrics   │
└─────────────────┘    └──────────────────┘    │ • Audit Logs        │
                                               │ • Sync Status       │
                                               └─────────────────────┘
```

## 🔄 N8N Workflow Components

### Core Pipeline Stages

1. **🕐 Scheduler Trigger**
   - Runs every 30 minutes
   - Initiates new data sync batches

2. **🏁 Batch Initialization**
   - Generates unique batch IDs
   - Sets up processing parameters
   - Creates sync status records

3. **📥 FHIR Data Fetching**
   - Connects to SMART on FHIR test server
   - Handles pagination automatically
   - Implements rate limiting (3-second delays)
   - Fetches 50 records per page

4. **🔍 Data Extraction & Parsing**
   - Validates FHIR Bundle responses
   - Extracts individual patient entries
   - Handles API errors gracefully

5. **🛠️ Data Transformation & Validation**
   - Comprehensive patient data transformation
   - Data quality scoring (0.0 - 1.0 scale)
   - Test data detection and flagging
   - Demographics validation
   - Contact information extraction

6. **💾 Database Storage**
   - Upsert operations to handle duplicates
   - JSONB storage for flexible patient data
   - Relational structure for queries

7. **📊 Quality Assessment**
   - Real-time data quality metrics
   - Batch statistics calculation
   - Quality distribution analysis

8. **🔍 Audit & Monitoring**
   - Complete operation logging
   - Performance metrics tracking
   - Error handling and reporting

### Workflow Node Details

| Node Name | Purpose | Key Features |
|-----------|---------|--------------|
| `Every 30 Minutes1` | Scheduler | Automated trigger every 30 minutes |
| `Initialize Batch` | Setup | Unique batch ID, parameters |
| `Fetch FHIR Data` | API Call | HTTP request with pagination |
| `Extract Bundle Entries1` | Parse | FHIR Bundle processing |
| `Transform & Validate Patient1` | ETL | Data quality scoring, validation |
| `Insert Patients to Supabase1` | Storage | Upsert to prevent duplicates |
| `Audit Successful Records1` | Logging | Operation audit trail |
| `Detect Last Item & Stats1` | Analytics | Batch completion detection |
| `Has Next Page?1` | Control | Pagination logic |
| `Setup Next Page1` | Navigation | Next page preparation |

## 🗄️ Database Schema

### Core Tables

#### `patient_records`
Stores processed FHIR patient data with quality metrics.

```sql
CREATE TABLE patient_records (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fhir_id VARCHAR NOT NULL,
    patient_data JSONB NOT NULL,  -- Complete patient information
    data_quality_score NUMERIC CHECK (data_quality_score BETWEEN 0 AND 1),
    source_system VARCHAR DEFAULT 'smart_on_fhir',
    sync_batch_id VARCHAR REFERENCES sync_status(batch_id),
    processed_at TIMESTAMPTZ DEFAULT NOW(),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Unique constraint to prevent duplicates
ALTER TABLE patient_records ADD CONSTRAINT unique_patient_per_source 
UNIQUE (fhir_id, source_system);
```

#### `sync_status`
Tracks batch processing status and statistics.

```sql
CREATE TABLE sync_status (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    batch_id VARCHAR UNIQUE NOT NULL,
    start_time TIMESTAMPTZ DEFAULT NOW(),
    end_time TIMESTAMPTZ,
    total_records INTEGER DEFAULT 0,
    successful_records INTEGER DEFAULT 0,
    failed_records INTEGER DEFAULT 0,
    status VARCHAR DEFAULT 'running' CHECK (status IN ('running', 'completed', 'failed', 'cancelled')),
    last_processed_page INTEGER DEFAULT 1,
    next_page_url TEXT,
    error_message TEXT
);
```

#### `pipeline_audit_log`
Comprehensive audit trail for all operations.

```sql
CREATE TABLE pipeline_audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operation_type VARCHAR NOT NULL CHECK (operation_type IN 
        ('fetch', 'validate', 'transform', 'store', 'error', 'retry', 'batch_start', 'batch_end')),
    resource_type VARCHAR NOT NULL,
    resource_id VARCHAR,
    status VARCHAR NOT NULL CHECK (status IN ('success', 'error', 'warning', 'info')),
    execution_time_ms INTEGER,
    error_details JSONB,
    metadata JSONB,
    sync_batch_id VARCHAR REFERENCES sync_status(batch_id),
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

#### `data_quality_metrics`
Aggregated quality metrics by time period and source.

```sql
CREATE TABLE data_quality_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    date_period DATE NOT NULL,
    source_system VARCHAR NOT NULL,
    total_records INTEGER DEFAULT 0,
    avg_quality_score NUMERIC,
    high_quality_records INTEGER DEFAULT 0,
    medium_quality_records INTEGER DEFAULT 0,
    low_quality_records INTEGER DEFAULT 0,
    validation_errors INTEGER DEFAULT 0,
    validation_warnings INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

## 📊 Data Quality Framework

### Quality Scoring Algorithm (0.0 - 1.0)

The pipeline implements a comprehensive quality scoring system:

| Component | Weight | Criteria |
|-----------|--------|----------|
| **Core Identity** | 30% | Patient ID, Name presence |
| **Demographics** | 35% | Gender, Birth Date, Address |
| **Contact Info** | 20% | Phone, Email availability |
| **Identifiers** | 10% | Medical record numbers, SSN |
| **Data Flags** | 5% | Test data detection, completeness |

### Quality Tiers

- **Excellent (0.9-1.0)**: Complete demographics, contact info, valid identifiers
- **Good (0.7-0.89)**: Minor missing fields, mostly complete
- **Fair (0.5-0.69)**: Some missing demographics or contact info
- **Poor (<0.5)**: Significant missing data, potential test records

### Test Data Detection

Automatic identification of test/demo records using pattern matching:
- Names starting with "test", "sample", "demo"
- Numeric-only names
- Pattern matching for "patient123" style entries

## 🚀 Recent Performance Results

### Latest Batch Analysis (ID: `1754827024771_dli3nd0om`)

#### ✅ Success Metrics
- **200 patient records** successfully processed
- **100% data integrity** - no orphaned or missing records
- **0 duplicate violations** - unique constraints working
- **7 minutes 12 seconds** total processing time
- **0.46 records/second** average throughput

#### 📊 Quality Distribution
- **117 records (58.5%)** - Excellent quality (≥0.9)
- **24 records (12%)** - Fair quality (0.5-0.69)
- **59 records (29.5%)** - Poor quality (<0.5)
- **Average quality score**: 0.756/1.0

#### 🎯 Data Completeness
- **100% valid FHIR structure**
- **100% have patient names**
- **70.5% have birth dates** (primary quality issue)
- **100% have contact information**
- **Some test data detected** and flagged

## 🔧 Configuration & Setup

### Prerequisites
- N8N workflow automation platform
- PostgreSQL database (Supabase)
- Access to SMART on FHIR test server

### Environment Variables
```env
SUPABASE_URL=your_supabase_url
SUPABASE_KEY=your_supabase_api_key
FHIR_BASE_URL=https://r4.smarthealthit.org
```

### N8N Setup
1. Import the workflow JSON into N8N
2. Configure Supabase credentials
3. Set up scheduler trigger (default: 30 minutes)
4. Test with manual execution

## 📈 Monitoring & Validation

### Key Monitoring Queries

```sql
-- Monitor batch processing status
SELECT batch_id, status, total_records, successful_records, 
       end_time - start_time as duration
FROM sync_status 
ORDER BY start_time DESC LIMIT 10;

-- Check data quality trends
SELECT date_period, source_system, total_records, avg_quality_score
FROM data_quality_metrics 
ORDER BY date_period DESC;

-- Audit recent operations
SELECT operation_type, status, COUNT(*) as count
FROM pipeline_audit_log 
WHERE created_at > NOW() - INTERVAL '24 hours'
GROUP BY operation_type, status;
```

## 🚨 Error Handling

### Common Issues & Solutions

#### Duplicate Key Violations
**Solution**: Workflow uses upsert operations to handle existing records gracefully.

#### API Rate Limiting
**Solution**: Built-in 3-second delays between requests.

#### Pagination Failures
**Solution**: Robust pagination detection with retry logic.

#### Data Quality Issues
**Solution**: Graceful degradation with quality scoring rather than rejection.

## 🔍 Data Validation Framework

The project includes comprehensive validation queries to verify:
- Data completeness and consistency
- Audit trail accuracy
- Batch processing status
- Quality score distributions
- Performance metrics

## 🛠️ Current Known Issues

1. **Sync Status Update**: Some batches may not update status correctly (under investigation)
2. **Audit Log Volume**: Excessive logging entries in some scenarios (optimization needed)
3. **Birth Date Gaps**: ~30% of records missing birth dates from source data

## 🔮 Future Enhancements

- [ ] Real-time alerting for data quality thresholds
- [ ] Machine learning-based quality prediction
- [ ] Advanced deduplication algorithms
- [ ] Multi-source FHIR server support
- [ ] Data lineage tracking
- [ ] Automated data remediation workflows

## 📚 Documentation

For detailed technical documentation:
- Database schema definitions in `/docs/schema.sql`
- Validation queries in `/docs/validation_queries.sql`
- N8N workflow export in `/workflows/fhir_pipeline.json`

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Test thoroughly with validation queries
4. Submit pull request with documentation updates

## 📄 License

This project is licensed under the MIT License - see the LICENSE file for details.

---

*Last updated: August 2025*
*Pipeline Version: 1.0*
*N8N Version: 1.104.2*
