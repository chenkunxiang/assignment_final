## Final Project: Remote Patient Monitoring System

---

## Project Overview

This project presents a comprehensive cloud architecture design for a **Remote Patient Monitoring System** aimed at improving chronic disease management through continuous vital sign monitoring and proactive clinical intervention.

### Healthcare Problem Addressed
Patients with chronic conditions (diabetes, hypertension, cardiovascular disease) often experience health deteriorations between clinical visits that go undetected until they require emergency intervention. This system enables continuous remote monitoring with automated alerting, allowing clinicians to intervene early and prevent acute episodes.

---

## Deliverables Summary

1. **Use Case Description** (`use_case.md`)
   - Problem statement for remote patient monitoring
   - Data sources from wearable devices and EHR systems
   - Step-by-step workflow from data ingestion to clinician action

2. **Architecture Diagram** (`architecture_diagram.png`)
   - Visual representation using Mermaid diagram
   - Shows frontend, compute, data, and analytics layers
   - Includes security and monitoring components

3. **Architecture & Implementation Plan** (`architecture_plan.md`)
   - Service mapping table with 10+ cloud services
   - Detailed data flow narrative (7 phases)
   - Security, identity, and governance strategy
   - Cost analysis and optimization for student budget

4. **Reflection** (`reflection.md`)
   - Confidence assessment of design decisions
   - Alternative architecture comparison (Cloud Composer vs. serverless)
   - Future enhancements with unlimited resources

---

## Architecture Highlights

### Cloud Services Integrated (GCP)

| Category | Services Used |
|----------|---------------|
| **Storage** | Cloud Storage (data lake, reports) |
| **Compute** | Cloud Run (Flask app), Cloud Functions (processing, alerts) |
| **Database** | Cloud SQL PostgreSQL (patient data, time-series vitals) |
| **Analytics** | BigQuery (data warehouse), Vertex AI (notebooks, ML) |
| **Security** | Cloud IAM, Secret Manager |
| **Monitoring** | Cloud Logging, Cloud Monitoring |

### Key Design Principles

- **Event-Driven Architecture**: Serverless functions triggered by data arrival
- **Cost-Optimized**: Extensive use of auto-scaling and free-tier services
- **Security-First**: HIPAA/GDPR compliance with encryption and RBAC
- **Scalable**: Handles 100 patients in prototype, scales to 10,000+ in production
- **Real-Time Capable**: 5-minute alert generation cycle, sub-second dashboard queries

---

## Estimated Costs

### Prototype (≤100 patients)
- **Monthly**: €15-25
- **Primary cost**: Cloud SQL db-f1-micro instance
- **Within student budget**:  Yes

### Production (1,000+ patients)
- **Monthly**: €200-400
- **Scales cost-effectively** with managed services

---

## Security & Compliance

-  **PHI Protection**: Encryption at rest and in transit
-  **Access Control**: Role-based IAM policies
-  **Audit Trail**: Complete logging of data access
-  **GDPR Compliant**: EU region data residency
-  **HIPAA Ready**: De-identified test environments

---

## Next Steps (If Implementing)

If this design were to be implemented as Part 2 (optional prototype):

1. Set up GCP project with billing alerts
2. Create Cloud SQL PostgreSQL instance
3. Configure Cloud Storage buckets
4. Build Flask application with basic dashboard
5. Deploy to Cloud Run with CI/CD
6. Implement first Cloud Function for data processing
7. Load synthetic patient data for testing
