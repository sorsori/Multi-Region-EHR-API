# Multi-Region EHR API

A cross-border health record **prototype** designed to respect **GDPR** and keep patient data inside its country of origin. It follows simplified **FHIR R4** structures and uses **SNOMED codes** where relevant. The aim is to show how a **federated model** can work across the UK, Germany, and France without centralising clinical data.

---

## 1. Overview

Sharing healthcare data across countries is complicated. This project takes a practical approach: **keep data in the patient’s home region**, but still allow clinicians to locate and view records when needed. It acts as a lightweight **Master Patient Index (MPI)** that links the same person across regions without exposing raw identifiers.

The system is **serverless**, built fully on **AWS**, with regional **DynamoDB** tables for medical data and a central MPI storing only **pseudonymised pointers**.

---

## 2. Core Architectural Strategy

To comply with **GDPR** and **FHIR** expectations, the design separates patient identity from clinical content:

* **Regional Vaults:** Each country (UK/DE/FR) has its own **DynamoDB** table holding Patient, Encounter, and Observation resources. **No medical data leaves the region at rest.**
* **Master Patient Index:** A **DynamoDB Global Table** storing internal **UUID** links between regions. No clinical data or raw national IDs are stored here.
* **Secure Linking:** A **Lambda** function hashes national IDs using a salt stored in **Secrets Manager**. Only the hash is used for MPI lookups.
* **Network Security:** All internal service calls flow through **VPC Endpoints (PrivateLink)** to keep traffic off the public internet.

---

## 3. Main Technologies

### Authentication & Access Control
* **Amazon Cognito** (User Pools + RBAC)

### Edge Layer
* **Amazon CloudFront** + **AWS WAF** (geo controls)

### API Layer
* **API Gateway** (REST) with Cognito authorisation

### Compute (Lambda Microservices)
* **Search Service:** Handles ID hashing and MPI lookups.
* **CRUD Lambdas:** For Patient, Encounter, and Observation resources.
* **Analysis Lambda:** For extracting clinical insights using **Comprehend Medical**.

### Storage
* **DynamoDB** regional tables (for clinical data)
* **DynamoDB Global Table** (for the MPI)

### Security
* **AWS KMS** (for encryption at rest)
* **AWS Secrets Manager** (for salts and tokens)

### Logging & Audit
* **Amazon S3** (for logs)
* **AWS CloudTrail** (for access auditing)

---

## 4. Data Flow (The Clinic Visit)

1.  A patient arrives and informs the clinician that they have a medical record in another supported country.
2.  The clinician enters the patient’s foreign **National Health ID**.
3.  The **Search Lambda** retrieves the hashing salt, hashes the ID, and queries the **MPI**.
4.  If a match is found, the system fetches relevant clinical data from that country’s regional table.
5.  Records are **merged in memory** and shown to the clinician. **Nothing is written centrally.**
6.  When new notes are added, **DynamoDB Streams** trigger the **Analysis Lambda**, which uses services like **Comprehend Medical** to extract coded insights.
7.  The frontend application aggregates these insights into a **visual clinical timeline**, allowing doctors to spot cross-border patterns (e.g., recurring symptoms) directly in the browser without persisting data centrally.

---

## 5. Project Structure

.
├── terraform/ # IaC definitions
│ ├── regions/
│ │ ├── germany/
│ │ ├── france/
│ │ └── uk/
│ └── mpi/ # Master Patient Index
├── lambda/
│ ├── search_service/ # Hash + MPI lookup
│ ├── patient_crud/
│ ├── encounter_crud/
│ └── observation_crud/
├── diagrams/ # Architecture diagrams
└── README.md

---

## 6. Development & Operations

* **Infrastructure as Code:** **Terraform** manages all infrastructure.
* **CI/CD Pipeline:** **GitHub Actions** handles testing and deployment.
* **Monitoring:** **CloudWatch** for metrics and logs; **CloudTrail** for audit trails.

---

## 7. Production Readiness: Target Multi-Account Strategy

### 7.1 Multi-Account Strategy (Security & Governance)
While this prototype leverages **Multi-Region** isolation within a single AWS account for demonstration purposes, a real-world production deployment would utilize **AWS Organizations** to enforce strict account-level isolation.

The following Organizational Unit (OU) structure ensures **GDPR compliance**, minimize **blast radius**, and enforce **Separation of Duties**:

Root
 ├── Security OU
 │    ├── Security-Tooling (GuardDuty, Security Hub aggregation)
 │    └── Incident-Response (Forensics environment)
 │
 ├── Log/Archive OU
 │    └── Logs-Audit (Immutable CloudTrail & Config archives)
 │
 ├── Workloads OU (Isolated by Sovereignty)
 │    ├── UK-Prod-Account
 │    ├── DE-Prod-Account (Strict Data Residency)
 │    ├── FR-Prod-Account
 │    └── Shared-Network-Services (Transit Gateway / Direct Connect)
 │
 ├── Research OU
 │    └── Research-Lake (De-identified data analysis)
 │
 └── Sandbox OU
      └── Developer-Sandboxes

### 7.2Application Hardening (Reliability & Performance)
To meet enterprise SLAs, the following architectural enhancements are designed for the production release:
* **Resilience & Fault Tolerance:** Implement **Amazon SQS Dead Letter Queues (DLQs)** and automated retry policies for the Analysis service to ensure zero data loss during downstream processing failures.
* **Performance Tuning:** Configure **Provisioned Concurrency** for the Search Lambda to eliminate cold starts, guaranteeing consistent low-latency lookups for clinicians during peak hours.

---

## 8. Future Work

* **Health Analytics:**  **Amazon QuickSight** dashboards visualising condition-specific clinical summaries across regions.

* **Research Data Lake:** A secondary, anonymised data store in **S3** populated via DynamoDB Streams for medical research.

* **Real-time Clinical Alerts:** **EventBridge** alerting system to notify clinicians of critical events (abnormal observation values, etc).

* **Enhanced Identification:** Support for alternative identifiers (e.g. passport numbers) to facilitate patient matching in emergency scenarios.

---

## 9. License

MIT
