# Architecture & Implementation Plan
## Remote Patient Monitoring System

---

## Service Mapping Table

| Layer | Service (Cloud) | Role in Solution | Related Assignment/Module |
|-------|----------------|------------------|---------------------------|
| **Storage - Raw Data** | GCP Cloud Storage (Bucket) | Store raw JSON payloads from wearable devices in data lake format; organized by date/patient for audit trail and reprocessing capabilities | Assignment 3: Cloud Storage & Data Lakes |
| **Storage - Reports** | GCP Cloud Storage (Bucket) | Archive generated analytics reports and visualizations for long-term retention and compliance | Assignment 3: Cloud Storage & Data Lakes |
| **Compute - Web Application** | Cloud Run (Container) | Run containerized Flask web application for clinician dashboard; auto-scales based on demand, zero cost when idle | Assignment 5: Containers & Cloud Run |
| **Compute - Data Processing** | Cloud Functions (Serverless) | Event-driven processing triggered when new device data arrives; validates, enriches, and loads data into database | Assignment 6: Serverless Computing |
| **Compute - Alert Generation** | Cloud Functions (Serverless) | Scheduled function that runs every 5 minutes to check new vitals against thresholds and generate alerts | Assignment 6: Serverless Computing |
| **Database - Relational** | Cloud SQL (PostgreSQL) | Store structured patient data, vital sign time-series, alert configurations, and clinician action logs with ACID guarantees | Assignment 4: Cloud Databases & SQL |
| **Analytics - Data Warehouse** | BigQuery | Store aggregated analytics data for complex queries across large datasets; enables trend analysis and reporting at scale | Assignment 7: Big Data & Analytics |
| **Analytics - ML/AI** | Vertex AI Workbench (Notebooks) | Run Python notebooks for predictive analytics, trend detection algorithms, and optional ML models for patient deterioration prediction | Assignment 8: Cloud AI/ML Services |
| **Security** | Cloud IAM & Secret Manager | Manage role-based access control; store database credentials, API keys securely; enforce least-privilege principle | Assignment 9: Cloud Security & Governance |
| **Monitoring** | Cloud Logging & Monitoring | Track application performance, audit data access, set up alerts for system failures or anomalies | Assignment 2: Cloud Management & Monitoring |

---

## Data Flow Narrative

### End-to-End System Flow

**Phase 1: Patient Onboarding**
1. Healthcare administrator uploads patient master data CSV file through Flask web interface
2. Flask application validates CSV format and data completeness
3. Patient records are inserted into Cloud SQL PostgreSQL database with encrypted PHI fields
4. Default alert thresholds are configured based on patient diagnosis codes

**Phase 2: Real-Time Data Ingestion**
1. Patient wearable device (glucose monitor, BP cuff, etc.) collects vital sign reading
2. Device transmits reading via mobile app to API Gateway endpoint
3. API Gateway authenticates device/patient and forwards JSON payload
4. Raw JSON is immediately stored in Cloud Storage bucket (`raw-vitals/YYYY-MM-DD/patient-{id}/`) for audit trail

**Phase 3: Data Processing Pipeline**
1. Cloud Storage write event triggers a Cloud Function (`process-vital-data`)
2. Function retrieves JSON payload, validates schema and data quality
3. Function queries Cloud SQL to enrich data with patient context (name, assigned clinician, baseline ranges)
4. Validated and enriched reading is inserted into Cloud SQL time-series table (`patient_vitals`) with indexed timestamp and patient_id
5. Processing metadata is logged to Cloud Logging

**Phase 4: Alert Generation**
1. Scheduled Cloud Function (`generate-alerts`) runs every 5 minutes via Cloud Scheduler
2. Function queries newly inserted vitals from last 5-minute window
3. Each reading is compared against patient-specific thresholds from `alert_config` table
4. Algorithm checks both absolute values (e.g., glucose > 250 mg/dL) and trends (e.g., 3 consecutive high readings)
5. Generated alerts are inserted into `clinical_alerts` table with severity level (CRITICAL, HIGH, MEDIUM, LOW)
6. Critical alerts trigger notification service (email/SMS to on-call clinician)

**Phase 5: Analytics & Trending**
1. Daily ETL job (Cloud Function) runs at 2 AM to extract previous day's data
2. Aggregated metrics are computed and loaded into BigQuery tables for data warehousing
3. Weekly scheduled Vertex AI notebook executes trend analysis algorithms
4. Notebook identifies patients with declining trends who haven't triggered acute alerts
5. Summary reports are generated as PDF/CSV and stored in Cloud Storage reports bucket
6. Dashboard cache is updated with latest analytics results

**Phase 6: Clinician Dashboard Access**
1. Clinician logs into Flask web application hosted on Cloud Run
2. Identity verified through Cloud IAM authentication
3. Dashboard queries Cloud SQL for:
   - Active alerts filtered by clinician's patient panel
   - Recent vital readings for selected patients
   - Trending graphs from cached analytics results
4. React/JavaScript frontend renders interactive visualizations
5. Clinician can acknowledge alerts, add clinical notes, or adjust thresholds
6. All actions are logged to `audit_log` table with timestamp and user identity

**Phase 7: Continuous Improvement**
1. Cloud Monitoring tracks system performance metrics (latency, error rates, throughput)
2. Vertex AI notebooks periodically retrain ML models with accumulated data
3. Clinical outcomes data is correlated with alert response times for quality improvement

---

## Security, Identity, and Governance

### Credential Management
All sensitive credentials are managed through **Cloud Secret Manager**, never hardcoded in application code or stored in version control. The Flask application and Cloud Functions access secrets at runtime using service account authentication. Environment variables reference secret paths rather than containing actual values. Database connection strings, API keys for external services, and encryption keys are rotated quarterly following security best practices.

### Access Controls and RBAC
We implement strict role-based access control using **Cloud IAM**:
- **Clinicians**: Read access to patient data only for their assigned panel; write access to clinical notes and alert acknowledgments
- **Administrators**: Full access to patient master data and system configuration; cannot access clinical notes
- **Data Analysts**: Read-only access to de-identified aggregated data in BigQuery; no access to raw PHI
- **Service Accounts**: Each Cloud Function and Cloud Run service uses dedicated service accounts with minimal permissions scoped to specific resources

All database access is authenticated through Cloud SQL Auth Proxy, eliminating the need for database passwords in application configuration.

### PHI Protection Strategy
To avoid exposing real Protected Health Information in public or development environments:
1. **Data Minimization**: Only collect essential patient data required for clinical decision-making
2. **Encryption**: All PHI is encrypted at rest using Cloud SQL encryption and in transit using TLS 1.3
3. **De-identification**: Development and testing environments use synthetic patient data generated with realistic statistical properties but no actual PHI
4. **Access Logging**: Every access to patient data is logged with user identity, timestamp, and purpose for HIPAA audit compliance
5. **Network Isolation**: Cloud SQL instances are not publicly accessible; access only through private VPC and Cloud SQL Proxy
6. **Data Residency**: All data stored in EU region to comply with GDPR requirements for the Bordeaux healthcare system

Regular security audits are conducted to validate compliance with HIPAA, GDPR, and local French healthcare data protection regulations.

---

## Cost and Operational Considerations

### Cost Analysis by Component

**Most Expensive Components:**
1. **Cloud SQL (PostgreSQL)**: Running a managed database instance 24/7 is the largest cost driver, estimated at €50-100/month for a small instance with high availability. The database must remain always-on to handle real-time vital sign ingestion.
2. **BigQuery**: While storage is relatively cheap, query costs can accumulate if analytics run frequently on large datasets. Estimated €20-40/month for weekly analytics workloads.
3. **Data Egress**: If clinicians access the dashboard from external networks, data transfer costs can add €10-20/month depending on usage patterns.

**Lower Cost Components:**
1. **Cloud Storage**: Very cost-effective for raw data archival at ~€0.02 per GB/month
2. **Cloud Functions**: Event-driven and billed per invocation with generous free tier (2 million invocations/month free)
3. **Cloud Run**: Auto-scales to zero when not in use; only pay for active request processing time

### Serverless vs. Always-On Trade-offs

**Use Serverless Functions for:**
- **Data processing pipeline**: Processing occurs only when new data arrives; no need for always-on compute
- **Alert generation**: Scheduled checks every 5 minutes means compute is idle 95% of the time
- **Report generation**: Weekly/monthly reports run briefly then terminate

**Use Always-On Database for:**
- **Cloud SQL**: Must remain available for real-time writes from wearable devices and dashboard queries; even brief downtime could miss critical vital signs

**Use Auto-Scaling Containers for:**
- **Flask dashboard**: Cloud Run scales to zero during night hours when no clinicians are active, but can instantly scale up during business hours; optimal balance of availability and cost

### Student Budget Optimization Strategies

To keep this project within free tier or minimal cost:
1. **Use Cloud SQL smallest instance type**: db-f1-micro (0.6 GB RAM) is sufficient for prototype with <100 patients
2. **Limit BigQuery queries**: Schedule analytics to run weekly instead of daily; use cached results
3. **Implement request caching**: Cache dashboard queries for 5-10 minutes to reduce database load
4. **Use Cloud Storage Standard tier**: Cheaper than Nearline for frequently accessed raw data
5. **Set up budget alerts**: Configure Cloud Billing alerts at €10, €25, €50 thresholds
6. **Leverage free tiers**: 
   - Cloud Functions: 2M invocations/month free
   - Cloud Run: 2M requests/month free
   - Cloud Storage: 5 GB/month free
   - BigQuery: 10 GB storage + 1 TB queries/month free
7. **Regional selection**: Use european-west9 (Paris) region for lower costs and GDPR compliance
8. **Auto-shutdown development environments**: Vertex AI notebooks should be stopped when not actively developing

**Estimated Monthly Cost for Prototype:**
- Cloud SQL (db-f1-micro): €10-15
- Cloud Storage: €1-2
- Cloud Run + Functions: €0-5 (mostly free tier)
- BigQuery: €0-3 (within free tier)
- **Total: €15-25/month** well within student budget

For production deployment with 1,000+ patients, costs would scale to €200-400/month but remain significantly cheaper than on-premises infrastructure.
