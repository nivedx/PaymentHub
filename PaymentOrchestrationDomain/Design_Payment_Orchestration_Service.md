# Payment Orchestration Service
## Unified End-to-End Design Document

---

### Document Control

| Version | Date | Author | Description |
|---------|------|--------|-------------|
| 2.0 | February 23, 2026 | Payment Hub Team | Consolidated Unified Service Design |

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Service Architecture Overview](#2-service-architecture-overview)
3. [Module Specifications](#3-module-specifications)
   - 3.1 [Payment Gateway Core](#31-payment-gateway-core)
   - 3.2 [TransactionContext Management](#32-transactioncontext-management)
   - 3.3 [Validation Pipeline](#33-validation-pipeline)
   - 3.4 [Limit Control Engine](#34-limit-control-engine)
   - 3.5 [Queue Management](#35-queue-management)
   - 3.6 [Status & Lifecycle Management](#36-status--lifecycle-management)
   - 3.7 [Embedded Routing Engine](#37-embedded-routing-engine)
4. [Domain Model](#4-domain-model)
5. [Data Architecture](#5-data-architecture)
6. [API Design](#6-api-design)
7. [Event-Driven Architecture](#7-event-driven-architecture)
8. [Processing Flow & Sequence Diagrams](#8-processing-flow--sequence-diagrams)
9. [Integration Points](#9-integration-points)
10. [Security Design](#10-security-design)
11. [Non-Functional Requirements](#11-non-functional-requirements)
12. [Deployment Architecture](#12-deployment-architecture)
13. [Monitoring & Alerting](#13-monitoring--alerting)
14. [Test Strategy](#14-test-strategy)
15. [Implementation Roadmap](#15-implementation-roadmap)

---

## 1. Executive Summary

The **Payment Orchestration Service** is the unified core service of the Payment Hub system, consolidating all payment orchestration, validation, limit control, queuing, status management, and routing capabilities into a single high-performance service. Following **ADR-005a**, this architecture eliminates inter-service network latency and enables seamless data sharing via a centralized **TransactionContext** object with write-through caching and disaster recovery support.

### Service Identity

| Attribute | Value |
|-----------|-------|
| **Service Name** | Payment Orchestration Service |
| **Bounded Context** | Payment Orchestration Domain |
| **Service Port** | 8001 |
| **API Version** | v1 |
| **Repository** | payment-orchestration-service |

### Key Capabilities

| Module | Capabilities |
|--------|--------------|
| **Payment Gateway Core** | Multi-channel reception, protocol translation, transaction initialization, request enrichment, orchestration |
| **TransactionContext** | Centralized state management, progressive enrichment, audit buffering, snapshot/recovery, disaster recovery |
| **Validation Pipeline** | 4-stage sequential validation (schema → business rules → reference data → duplicate detection) |
| **Limit Control Engine** | 3-step limit control (lookup → calculation → decision), multi-level limits, usage tracking |
| **Queue Management** | Priority assignment, queue placement, scheduling, backpressure management, SLA enforcement |
| **Status & Lifecycle** | State machine management, event publishing, retry handling, SLA monitoring, inline routing trigger |
| **Embedded Routing Engine** | Rule evaluation, multi-criteria scoring, rail selection, fallback handling, rail execution |

### Performance Metrics (ADR-005a)

| Metric | Before (Separate Services) | After (Unified Service) | Improvement |
|--------|---------------------------|------------------------|-------------|
| **End-to-End Latency (p99)** | 200ms | 150ms | 25% faster |
| **Throughput** | 5,000 TPS | 7,500 TPS | 50% increase |
| **Network Hops** | 3 network calls | 0 network calls | 100% reduction |
| **Infrastructure Cost** | 2 services | 1 service | ~30% reduction |
| **Context Serialization** | Required per hop | Zero serialization | 100% eliminated |
| **Routing Latency** | 30-50ms | <5ms | 85-90% faster |

### Business Value

- **Unified Entry Point**: Single gateway for all payment channels ensures consistency
- **High Performance**: Zero-serialization context sharing achieves 7,500+ TPS
- **Optimal Routing**: Multi-criteria scoring ensures best rail selection with 15-25% cost optimization
- **Resilience**: Automatic fallback handling and disaster recovery support
- **Compliance**: Complete audit trail for all processing stages
- **Operational Visibility**: Real-time status tracking and SLA monitoring

---

## 2. Service Architecture Overview

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           PAYMENT ORCHESTRATION SERVICE (Port: 8001)                     │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐│
│  │                              ENTRY LAYER                                            ││
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  ┌───────────────┐ ││
│  │  │ REST Controller │  │ GraphQL Resolver│  │   MQ Listener   │  │ File Processor│ ││
│  │  │ POST /payments  │  │ mutation        │  │ payment.request │  │ *.csv, *.xml  │ ││
│  │  │ POST /transfers │  │ createPayment   │  │ payment.bulk    │  │ ISO20022      │ ││
│  │  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘  └───────┬───────┘ ││
│  │           └───────────────────┬┴───────────────────┬┴───────────────────┘         ││
│  └───────────────────────────────┼─────────────────────────────────────────────────────┘│
│                                  ▼                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐│
│  │                         PAYMENT GATEWAY CORE                                        ││
│  │  Protocol Translation → Transaction Initialization → Request Enrichment →          ││
│  │  Idempotency Check → Orchestration Engine                                          ││
│  └────────────────────────────────┬───────────────────────────────────────────────────┘│
│                                   │ Creates & Manages                                   │
│  ┌────────────────────────────────▼───────────────────────────────────────────────────┐│
│  │                    TRANSACTION CONTEXT (Persistent with Write-Through Cache)        ││
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌─────────────────┐  ││
│  │  │ Core Data  │ │ Enrichment │ │Stage Results│ │ Status     │ │ Audit Buffer   │  ││
│  │  │ Payment    │ │ Customer   │ │ Validation │ │ History    │ │ Compliance Recs│  ││
│  │  │ Details    │ │ Metadata   │ │ Limit,Queue│ │ Timeline   │ │ Stage Events   │  ││
│  │  └────────────┘ └────────────┘ └────────────┘ └────────────┘ └─────────────────┘  ││
│  └────────────────────────────────┬───────────────────────────────────────────────────┘│
│                                   │ Shared Reference (Zero-Copy)                        │
│                                   ▼                                                     │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐│
│  │                          PROCESSING PIPELINE                                        ││
│  │                                                                                     ││
│  │  ┌─────────────────────────────────────────────────────────────────────────────┐  ││
│  │  │ VALIDATION PIPELINE (4 Stages - Fail-Fast)                                   │  ││
│  │  │ [Schema] → [Business Rules] → [Reference Data] → [Duplicate Detection]       │  ││
│  │  └──────────────────────────────────┬──────────────────────────────────────────┘  ││
│  │                                     ▼                                              ││
│  │  ┌─────────────────────────────────────────────────────────────────────────────┐  ││
│  │  │ LIMIT CONTROL ENGINE (3 Steps)                                               │  ││
│  │  │ [Limit Lookup] → [Limit Calculation] → [Limit Decision]                      │  ││
│  │  │  ↓ Cache         ↓ Usage Counter       ↓ Allow/Block/Warn                    │  ││
│  │  └──────────────────────────────────┬──────────────────────────────────────────┘  ││
│  │                                     ▼                                              ││
│  │  ┌─────────────────────────────────────────────────────────────────────────────┐  ││
│  │  │ QUEUE MANAGEMENT (3 Phases)                                                  │  ││
│  │  │ [Priority Assignment] → [Queue Placement] → [Scheduling]                     │  ││
│  │  │  ↓ Urgency/Segment      ↓ Priority/Std/Batch  ↓ Immediate/Deferred          │  ││
│  │  └──────────────────────────────────┬──────────────────────────────────────────┘  ││
│  │                                     ▼                                              ││
│  │  ┌─────────────────────────────────────────────────────────────────────────────┐  ││
│  │  │ STATUS & LIFECYCLE MANAGEMENT                                                │  ││
│  │  │ [State Machine] → [Event Publisher] → [Retry Handler] → [SLA Monitor]       │  ││
│  │  │                                              ↓ Ready for Routing             │  ││
│  │  └──────────────────────────────────┬──────────────────────────────────────────┘  ││
│  │                                     ▼ Direct Method Call (Zero Network Hop)       ││
│  │  ┌─────────────────────────────────────────────────────────────────────────────┐  ││
│  │  │ EMBEDDED ROUTING ENGINE                                                      │  ││
│  │  │ [Rule Engine] → [Smart Router] → [Decision Maker] → [Rail Executor]         │  ││
│  │  │  ↓ Rules Cache   ↓ Multi-Criteria   ↓ Primary+Fallback  ↓ IPP/IPI/FTS/SWIFT │  ││
│  │  └─────────────────────────────────────────────────────────────────────────────┘  ││
│  └────────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                         │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐│
│  │                          CROSS-CUTTING CONCERNS                                     ││
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ ││
│  │  │ Health &    │  │ Metrics     │  │ Audit &     │  │ Security    │  │ Config   │ ││
│  │  │ Readiness   │  │ Publisher   │  │ Compliance  │  │ Context     │  │ Refresh  │ ││
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  └──────────┘ ││
│  └────────────────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                           │
                    ┌──────────────────────┼──────────────────────┐
                    ▼                      ▼                      ▼
          ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
          │ Configuration    │   │ Risk & Compliance│   │  Payment Rails   │
          │ Domain           │   │ Domain           │   │  IPP/IPI/FTS/SWIFT│
          └──────────────────┘   └──────────────────┘   └──────────────────┘
```

### 2.2 Processing Flow Summary

```
Payment Request
       │
       ▼
┌──────────────────┐
│ 1. Gateway Core  │ → Protocol Translation → Initialize Transaction → Enrich → Create Context
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 2. Validation    │ → Schema → Business Rules → Reference Data → Duplicate Detection
│    Pipeline      │   (Fail-Fast: Any failure = Immediate Rejection)
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 3. Limit Control │ → Lookup Limits → Calculate Usage → Make Decision (Allow/Block/Warn)
│    Engine        │   (Breach = Block or Route to Approval)
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 4. Queue Mgmt    │ → Calculate Priority → Assign Queue → Schedule Execution
│                  │
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 5. Status &      │ → Update State → Publish Events → Check SLA → Trigger Routing
│    Lifecycle     │
└────────┬─────────┘
         ▼ (Direct Method Call)
┌──────────────────┐
│ 6. Routing       │ → Evaluate Rules → Score Rails → Select Primary+Fallback → Execute
│    Engine        │
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 7. Rail Executor │ → Execute on Rail → Handle Response → Fallback if Needed → Finalize
│                  │
└────────┬─────────┘
         ▼
   Payment Complete (Success/Failure)
```

### 2.3 Module Interactions

| Source Module | Target Module | Interaction Type | Data Exchanged |
|---------------|---------------|------------------|----------------|
| Gateway Core | TransactionContext | Create/Write | Initial context with payment details |
| Gateway Core | Validation Pipeline | Method Call | TransactionContext reference |
| Validation Pipeline | TransactionContext | Read/Write | Validation results, enrichment data |
| Validation Pipeline | Limit Control | Method Call | TransactionContext reference |
| Limit Control | TransactionContext | Read/Write | Limit check results |
| Limit Control | Queue Management | Method Call | TransactionContext reference |
| Queue Management | TransactionContext | Read/Write | Queue assignment, priority |
| Queue Management | Status & Lifecycle | Method Call | TransactionContext reference |
| Status & Lifecycle | TransactionContext | Read/Write | Status updates, history |
| Status & Lifecycle | Routing Engine | Direct Method Call | TransactionContext reference |
| Routing Engine | TransactionContext | Read/Write | Routing decision, execution result |
| Routing Engine | External Rails | Adapter Call | Rail-specific payloads |

---

## 3. Module Specifications

### 3.1 Payment Gateway Core

The **Payment Gateway Core** is the entry point for all payment requests, handling multi-channel reception, protocol translation, and orchestration initiation.

#### 3.1.1 Component Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       PAYMENT GATEWAY CORE MODULE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │  Channel Receivers  │  │ Protocol Translators│  │ Request Validators │ │
│  │                     │  │                     │  │                     │ │
│  │ • REST Controller   │  │ • JSON Translator   │  │ • Schema Validator  │ │
│  │ • GraphQL Resolver  │  │ • XML/ISO20022 Trans│  │ • Format Validator  │ │
│  │ • MQ Listener       │  │ • CSV Translator    │  │ • Mandatory Fields  │ │
│  │ • File Processor    │  │ • CAMT/PAIN Trans   │  │ • Channel Rules     │ │
│  │ • Webhook Receiver  │  │ • Custom Mappers    │  │ • Rate Limiting     │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │  Transaction        │  │   Request           │  │   Idempotency       │ │
│  │  Initializer        │  │   Enrichment        │  │   Manager           │ │
│  │                     │  │                     │  │                     │ │
│  │ • ID Generation     │  │ • Customer Lookup   │  │ • Key Generation    │ │
│  │ • Timestamp Mgmt    │  │ • Channel Metadata  │  │ • Duplicate Check   │ │
│  │ • Correlation IDs   │  │ • Compliance Flags  │  │ • Response Cache    │ │
│  │ • Initial State     │  │ • Risk Indicators   │  │ • TTL Management    │ │
│  │ • Context Creation  │  │ • Trace Context     │  │ • Cleanup Jobs      │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │   Orchestration     │  │   Batch Processing  │  │   Response          │ │
│  │   Engine            │  │   Engine            │  │   Builder           │ │
│  │                     │  │                     │  │                     │ │
│  │ • Flow Coordinator  │  │ • File Parsing      │  │ • Sync Response     │ │
│  │ • Stage Dispatcher  │  │ • Chunk Processing  │  │ • Async Ack         │ │
│  │ • Error Handler     │  │ • Progress Tracking │  │ • Batch Response    │ │
│  │ • Timeout Manager   │  │ • Error Aggregation │  │ • Error Mapping     │ │
│  │ • Circuit Breaker   │  │ • Result Generation │  │ • Status Formatting │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.1.2 Functional Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| FR-PGC-001 | Accept payment requests via REST API, GraphQL, MQ, File/SFTP, Webhooks | High |
| FR-PGC-002 | Translate JSON, XML/ISO20022, CSV to canonical PaymentRequest model | High |
| FR-PGC-003 | Generate UUID v7 transaction ID, correlation ID, timestamps | High |
| FR-PGC-004 | Enrich requests with customer profile, channel metadata, compliance flags | High |
| FR-PGC-005 | Manage idempotency keys with 24h TTL, return cached responses for duplicates | High |
| FR-PGC-006 | Orchestrate sequential flow with timeout and circuit breaker support | High |
| FR-PGC-007 | Process batch files with chunking (1000 records), progress tracking, error aggregation | High |
| FR-PGC-008 | Build sync/async/batch responses with correlation IDs | High |
| FR-PGC-009 | Create and share TransactionContext across all stages | High |
| FR-PGC-010 | Log all requests/responses with PII masking for compliance | High |

#### 3.1.3 Performance Targets

| Metric | Target |
|--------|--------|
| Request Reception Latency (p99) | < 5ms |
| Protocol Translation (p99) | < 3ms |
| Context Creation (p99) | < 15ms |
| Idempotency Check (p99) | < 2ms |
| Batch Processing Rate | 500+ records/sec |

---

### 3.2 TransactionContext Management

The **TransactionContext** is the central state management component providing a shared persistent context object with write-through caching that flows through all processing stages.

#### 3.2.1 Component Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TRANSACTION CONTEXT MODULE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │  Context Factory    │  │  Context Registry   │  │  Context Lifecycle  │ │
│  │                     │  │                     │  │  Manager            │ │
│  │ • Context Creation  │  │ • Active Contexts   │  │ • State Transitions │ │
│  │ • ID Generation     │  │ • Lookup by TxnId   │  │ • TTL Management    │ │
│  │ • Initial State     │  │ • Memory Tracking   │  │ • Cleanup Triggers  │ │
│  │ • Template Apply    │  │ • Eviction Policy   │  │ • Recovery Handling │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │  State Holder       │  │  Enrichment         │  │  Stage Result       │ │
│  │  (Core Data)        │  │  Container          │  │  Aggregator         │ │
│  │                     │  │                     │  │                     │ │
│  │ • Transaction ID    │  │ • Customer Profile  │  │ • Validation Result │ │
│  │ • Payment Details   │  │ • Channel Metadata  │  │ • Limit Result      │ │
│  │ • Account Info      │  │ • Risk Indicators   │  │ • Queue Assignment  │ │
│  │ • Timestamps        │  │ • Beneficiary Info  │  │ • Routing Decision  │ │
│  │ • Correlation IDs   │  │ • Compliance Flags  │  │ • Execution Result  │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │  Audit Buffer       │  │  Snapshot Manager   │  │  Concurrency        │ │
│  │                     │  │                     │  │  Controller         │ │
│  │ • Entry Buffering   │  │ • Point-in-Time     │  │ • Read/Write Locks  │ │
│  │ • Batch Flush       │  │ • Delta Tracking    │  │ • Optimistic Locking│ │
│  │ • Compliance Records│  │ • Restoration       │  │ • Version Tracking  │ │
│  │ • Async Persistence │  │ • Serialization     │  │ • Deadlock Prevention│ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.2.2 Context Structure

```
TransactionContext (Central State Object)
├── Meta Information
│   ├── contextId: UUID
│   ├── version: Long (optimistic locking)
│   ├── stage: ContextStage
│   ├── createdAt/lastModifiedAt: Timestamp
│   └── ttlExpiresAt: Timestamp
│
├── Core Transaction Data
│   ├── transactionId: UUID (time-ordered)
│   ├── correlationId: String
│   ├── idempotencyKey: String
│   ├── channelType: ChannelType
│   ├── paymentType: PaymentType
│   └── receivedAt: Timestamp
│
├── Payment Details
│   ├── amount: BigDecimal
│   ├── currency: String (ISO 4217)
│   ├── sourceAccount: AccountInfo
│   ├── destinationAccount: AccountInfo
│   ├── valueDate: LocalDate
│   └── remittanceInfo: String
│
├── Enrichment Container
│   ├── customerProfile: CustomerProfile
│   ├── channelMetadata: ChannelMetadata
│   ├── riskIndicators: RiskIndicators
│   └── complianceFlags: List<String>
│
├── Stage Results
│   ├── validationResult: ValidationResult
│   ├── limitResult: LimitResult
│   ├── queueAssignment: QueueAssignment
│   ├── routingDecision: RoutingDecision
│   └── executionResult: ExecutionResult
│
├── Status Tracking
│   ├── currentStatus: TransactionStatus
│   ├── statusHistory: List<StatusChange>
│   └── slaTarget: Timestamp
│
├── Retry Tracking
│   ├── retryCount: Integer
│   ├── nextRetryAt: Timestamp
│   └── retryHistory: List<RetryAttempt>
│
├── Audit Buffer
│   ├── auditEntries: List<AuditEntry>
│   └── complianceRecords: List<ComplianceRecord>
│
└── Extension Data
    └── attributes: Map<String, Object>
```

#### 3.2.3 Context Ownership Model

| Context Section | Writer | Readers | Access Pattern |
|-----------------|--------|---------|----------------|
| Core Transaction Data | Gateway Core | ALL | Write-Once |
| Payment Details | Gateway Core | ALL | Write-Once |
| Customer Profile | Validation Pipeline | Limit, Routing, Execution | Write-Once |
| Validation Result | Validation Pipeline | Limit, Queue, Status | Write-Once |
| Limit Result | Limit Control Engine | Queue, Status, Routing | Write-Once |
| Queue Assignment | Queue Management | Status, Routing | Write-Once |
| Current Status | Status & Lifecycle | ALL | Multi-Write |
| Routing Decision | Routing Engine | Execution, Status | Write-Once |
| Execution Result | Rail Executor | Status | Multi-Write (retries) |
| Audit Buffer | ALL Stages | Audit System | Append-Only |

#### 3.2.4 Performance Targets

| Metric | Target |
|--------|--------|
| Context Access (cached) | < 3ms |
| Inter-Stage Data Transfer | < 2ms |
| Context Creation | < 15ms |
| Snapshot Creation | < 10ms |
| Throughput | 7,500+ TPS |

---

### 3.3 Validation Pipeline

The **Validation Pipeline** implements a 4-stage sequential validation process with fail-fast behavior.

#### 3.3.1 Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       VALIDATION PIPELINE MODULE                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    PIPELINE ORCHESTRATOR                             │   │
│  │  • Stage Sequencing  • Fail-Fast Control  • Result Aggregation      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│         ┌──────────────────────────┼──────────────────────────┐            │
│         ▼                          ▼                          ▼            │
│  ┌─────────────────┐  ┌────────────────────┐  ┌─────────────────────────┐ │
│  │  STAGE 1        │  │  STAGE 2           │  │  STAGE 3                │ │
│  │  Schema         │  │  Business Rules    │  │  Reference Data         │ │
│  │  Validation     │  │  Validation        │  │  Validation             │ │
│  │                 │  │                    │  │                         │ │
│  │ • JSON Schema   │  │ • Payment Type     │  │ • Holiday Calendar      │ │
│  │ • XML Schema    │  │ • Channel Rules    │  │ • Cut-off Times         │ │
│  │ • Required Flds │  │ • Amount Threshold │  │ • Participant Check     │ │
│  │ • Data Types    │  │ • Time Windows     │  │ • BIC/IBAN Validation   │ │
│  │ • Format Check  │  │ • Custom Rules     │  │ • Currency Support      │ │
│  │                 │  │                    │  │                         │ │
│  │ Uses: Cache     │  │ Uses: Rules Repo   │  │ Uses: Config Domain     │ │
│  └─────────────────┘  └────────────────────┘  └─────────────────────────┘ │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  STAGE 4: DUPLICATE DETECTION                                       │   │
│  │                                                                      │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │   │
│  │  │ Transaction ID  │  │ Idempotency Key │  │ Content-Based       │  │   │
│  │  │ Uniqueness      │  │ Validation      │  │ Duplicate Check     │  │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────┘  │   │
│  │  Triggers: Fraud Detection Service                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.3.2 Stage Details

| Stage | Purpose | Checks Performed | Integration |
|-------|---------|------------------|-------------|
| **Stage 1: Schema** | Structural integrity | JSON/XML schema, required fields, data types, format | Cache Service |
| **Stage 2: Business Rules** | Business logic | Payment type rules, channel rules, amount limits, time windows | Rules Repository |
| **Stage 3: Reference Data** | External validation | Holiday calendar, cut-off times, participant availability, BIC/IBAN | Configuration Domain |
| **Stage 4: Duplicate Detection** | Uniqueness | Transaction ID, idempotency key, content-based matching | Fraud Detection |

#### 3.3.3 Performance Targets

| Metric | Target |
|--------|--------|
| Stage 1 Latency (p99) | < 5ms |
| Stage 2 Latency (p99) | < 10ms |
| Stage 3 Latency (p99) | < 15ms |
| Stage 4 Latency (p99) | < 8ms |
| **Total Pipeline (p99)** | **< 35ms** |
| Schema Cache Hit Rate | > 99% |
| Validation Pass Rate | > 95% |

---

### 3.4 Limit Control Engine

The **Limit Control Engine** implements a 3-step limit control process enforcing transaction limits across multiple dimensions.

#### 3.4.1 Engine Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      LIMIT CONTROL ENGINE MODULE                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    ENGINE ORCHESTRATOR                               │   │
│  │  • Step Sequencing   • Decision Coordination  • Result Aggregation  │   │
│  │  • Hold Management   • Alert Triggering       • Context Enrichment  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│         ┌──────────────────────────┼──────────────────────────┐            │
│         ▼                          ▼                          ▼            │
│  ┌─────────────────┐  ┌─────────────────────┐  ┌─────────────────────────┐ │
│  │  STEP 1         │  │  STEP 2             │  │  STEP 3                 │ │
│  │  Limit Lookup   │  │  Limit Calculation  │  │  Limit Decision         │ │
│  │                 │  │                     │  │                         │ │
│  │ • User Limits   │  │ • Usage Aggregation │  │ • Allow/Block/Warn      │ │
│  │ • Account Limits│  │ • Available Balance │  │ • Temporary Holds       │ │
│  │ • Channel Limits│  │ • Utilization %     │  │ • Breach Alerts         │ │
│  │ • Product Limits│  │ • Time-Based Reset  │  │ • Approval Routing      │ │
│  │ • Corp Hierarchy│  │ • Multi-Currency    │  │ • AML Trigger           │ │
│  │                 │  │                     │  │                         │ │
│  │ Uses:           │  │ Uses:               │  │ Triggers:               │ │
│  │ • Limit Defs    │  │ • Pricing Engine    │  │ • AML Screening         │ │
│  │ • Cache Service │  │ • Usage Counters    │  │ • Approval Workflow     │ │
│  └─────────────────┘  └─────────────────────┘  └─────────────────────────┘ │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    LIMIT TYPE HIERARCHY                              │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │   │
│  │  │ User-Level  │  │ Account-    │  │ Channel-    │  │ Product-    │ │   │
│  │  │ Daily/Week/ │  │ Level       │  │ Level       │  │ Level       │ │   │
│  │  │ Monthly/Txn │  │ Balance/Cnt │  │ Web/Mobile  │  │ P2P/B2B     │ │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘ │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │   │
│  │  │ Corporate   │  │ Beneficiary │  │ Geographic  │  │ Currency    │ │   │
│  │  │ Hierarchy   │  │ Limits      │  │ Limits      │  │ Limits      │ │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.4.2 Limit Decision Outcomes

| Decision | Action | Next Step |
|----------|--------|-----------|
| **ALLOW** | Proceed to queue management | Continue pipeline |
| **BLOCK** | Reject transaction | Return error, publish event |
| **WARN** | Proceed with notification | Continue pipeline, alert ops |
| **APPROVAL_REQUIRED** | Route to approval queue | Pause pipeline, notify approver |

#### 3.4.3 Performance Targets

| Metric | Target |
|--------|--------|
| Step 1 Latency (p99) | < 3ms |
| Step 2 Latency (p99) | < 5ms |
| Step 3 Latency (p99) | < 2ms |
| **Total Engine (p99)** | **< 10ms** |
| Limit Cache Hit Rate | > 99% |
| Counter Update | < 1ms |
| Throughput | > 10,000 TPS |

---

### 3.5 Queue Management

The **Queue Management** module handles intelligent queuing, prioritization, and scheduling of payment transactions.

#### 3.5.1 Queue Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       QUEUE MANAGEMENT MODULE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │  Priority           │  │   Queue             │  │   Scheduling        │ │
│  │  Calculator         │  │   Router            │  │   Engine            │ │
│  │                     │  │                     │  │                     │ │
│  │ • Urgency Scorer    │  │ • Queue Selector    │  │ • Immediate Handler │ │
│  │ • Segment Weigher   │  │ • Priority Queue    │  │ • Deferred Scheduler│ │
│  │ • Amount Scorer     │  │ • Standard Queue    │  │ • Future Date Mgr   │ │
│  │ • SLA Calculator    │  │ • Batch Queue       │  │ • Retry Scheduler   │ │
│  │ • Priority Combiner │  │ • DLQ Handler       │  │ • Cron Processor    │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │  Queue Consumer     │  │   Backpressure      │  │   SLA Monitor       │ │
│  │                     │  │   Controller        │  │                     │ │
│  │ • Dequeue Manager   │  │ • Rate Limiter      │  │ • SLA Tracker       │ │
│  │ • Batch Poller      │  │ • Load Shedder      │  │ • Breach Predictor  │ │
│  │ • Priority Poller   │  │ • Circuit Breaker   │  │ • Escalation Engine │ │
│  │ • Consumer Group Mgr│  │ • Throttle Manager  │  │ • Alert Generator   │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.5.2 Queue Types

| Queue | Priority | Use Case | Target Latency |
|-------|----------|----------|----------------|
| **Priority Queue** | Highest | Urgent, high-value, VIP transactions | < 50ms |
| **Standard Queue** | Normal | Regular payment processing | < 200ms |
| **Batch Queue** | Lower | Scheduled, bulk, low-priority | < 5 min |
| **Dead Letter Queue** | N/A | Failed transactions requiring intervention | < 1 hour |

#### 3.5.3 Priority Calculation

```
Priority Score = (Urgency × 0.30) + (Segment × 0.25) + (Amount × 0.20) + (SLA × 0.25)

Where:
- Urgency: Time-sensitivity score (0-100)
- Segment: Customer segment weight (Retail=40, SME=60, Corporate=80, VIP=100)
- Amount: Amount-based score (scaled logarithmically)
- SLA: SLA urgency score based on deadline proximity
```

#### 3.5.4 Performance Targets

| Metric | Target |
|--------|--------|
| Queue Assignment Time | < 5ms |
| Priority Queue Latency | < 50ms |
| Standard Queue Latency | < 200ms |
| Queue Throughput | 10,000+ TPS |
| SLA Breach Rate | < 0.1% |

---

### 3.6 Status & Lifecycle Management

The **Status & Lifecycle Management** module manages the complete transaction lifecycle with state tracking and inline routing trigger.

#### 3.6.1 Module Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│               STATUS & LIFECYCLE MANAGEMENT MODULE                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │   Status Manager    │  │   State Machine     │  │  Event Publisher    │ │
│  │                     │  │                     │  │                     │ │
│  │ • Status CRUD       │  │ • State Definitions │  │ • Event Generation  │ │
│  │ • Current State     │  │ • Transition Rules  │  │ • Topic Routing     │ │
│  │ • State Validation  │  │ • Guard Conditions  │  │ • Webhook Dispatch  │ │
│  │ • Inline Routing    │  │ • Action Triggers   │  │ • Delivery Guarantee│ │
│  │ • Context Updates   │  │ • Error States      │  │ • Dead Letter Pub   │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │   Retry Handler     │  │   SLA Monitor       │  │  Notification       │ │
│  │                     │  │                     │  │  Engine             │ │
│  │ • Retry Policy      │  │ • SLA Definitions   │  │ • Alert Triggers    │ │
│  │ • Backoff Strategy  │  │ • Threshold Check   │  │ • Channel Routing   │ │
│  │ • Max Attempts      │  │ • Breach Alerting   │  │ • Template Engine   │ │
│  │ • Circuit Breaker   │  │ • Escalation Rules  │  │ • Preference Mgmt   │ │
│  │ • DLQ Routing       │  │ • Dashboard Metrics │  │ • Rate Limiting     │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.6.2 Transaction State Machine

```
                              ┌─────────────┐
                              │  RECEIVED   │
                              └──────┬──────┘
                                     │ validate()
                                     ▼
                              ┌─────────────┐
                         ┌────│  VALIDATING │────┐
                   reject()   └──────┬──────┘    │ error()
                         │           │ pass()    │
                         ▼           ▼           ▼
                  ┌──────────┐ ┌──────────┐ ┌──────────────┐
                  │ REJECTED │ │ VALIDATED│ │VALIDATION_ERR│
                  └──────────┘ └────┬─────┘ └──────────────┘
                                    │ checkLimits()
                                    ▼
                              ┌─────────────┐
                         ┌────│LIMIT_CHECK  │────┬────────┐
                   block()    └──────┬──────┘    │        │ approvalRequired()
                         │           │           │ error()│
                         ▼           │ allow()   ▼        ▼
                  ┌──────────┐       │    ┌──────────┐ ┌────────────┐
                  │ BLOCKED  │       │    │LIMIT_ERR │ │ PENDING_   │
                  └──────────┘       │    └──────────┘ │ APPROVAL   │
                                     ▼                  └────────────┘
                              ┌─────────────┐
                              │   QUEUED    │
                              └──────┬──────┘
                                     │ dequeue()
                                     ▼
                              ┌─────────────┐
                              │ READY_FOR_  │
                              │ ROUTING     │
                              └──────┬──────┘
                                     │ route() [Direct Method Call]
                                     ▼
                              ┌─────────────┐
                         ┌────│  ROUTING    │────┐
                  noRail()    └──────┬──────┘    │ routingError()
                         │           │ routed()  │
                         ▼           ▼           ▼
                  ┌──────────┐ ┌──────────┐ ┌──────────────┐
                  │ NO_RAIL  │ │  ROUTED  │ │ ROUTING_ERR  │
                  └──────────┘ └────┬─────┘ └──────────────┘
                                    │ execute()
                                    ▼
                              ┌─────────────┐
                         ┌────│ EXECUTING   │────┬────────┐
                   fail()     └──────┬──────┘    │        │ pending()
                         │           │ success() │ error()│
                         ▼           ▼           ▼        ▼
                  ┌──────────┐ ┌──────────┐ ┌────────┐ ┌─────────────┐
                  │ EXEC_    │ │COMPLETED │ │ EXEC_  │ │ PENDING_    │
                  │ FAILED   │ │          │ │ ERROR  │ │ SETTLEMENT  │
                  └────┬─────┘ └──────────┘ └────────┘ └─────────────┘
                       │
                       │ retry()
                       ▼
                  ┌──────────┐
                  │ RETRYING │ ──→ (back to ROUTING or EXECUTING)
                  └──────────┘
```

#### 3.6.3 Inline Routing Trigger

When a transaction reaches `READY_FOR_ROUTING` status, the Status Manager directly invokes the Embedded Routing Engine via method call (zero network hop):

```java
// Pseudocode for inline routing
if (newStatus == READY_FOR_ROUTING) {
    routingEngine.routeAndExecute(transactionContext);  // Direct method call
}
```

---

### 3.7 Embedded Routing Engine

The **Embedded Routing Engine** determines the optimal payment rail and executes transactions through rail-specific adapters.

#### 3.7.1 Engine Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    EMBEDDED ROUTING ENGINE MODULE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │   Rule Engine       │  │   Smart Router      │  │   Decision Maker    │ │
│  │                     │  │                     │  │                     │ │
│  │ • Rule Loader       │  │ • Score Calculator  │  │ • Primary Selector  │ │
│  │ • Rule Evaluator    │  │ • Cost Analyzer     │  │ • Fallback Handler  │ │
│  │ • Constraint Checker│  │ • Speed Evaluator   │  │ • Cost-Benefit Calc │ │
│  │ • Score Aggregator  │  │ • Health Integrator │  │ • Decision Auditor  │ │
│  │ • Time-Based Router │  │ • Weight Applier    │  │ • Context Updater   │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │   Rail Executor     │  │   Health Checker    │  │   Fallback Manager  │ │
│  │                     │  │                     │  │                     │ │
│  │ • Adapter Factory   │  │ • Health Monitor    │  │ • Fallback Strategy │ │
│  │ • IPP Adapter       │  │ • Availability Check│  │ • Retry Coordinator │ │
│  │ • IPI Adapter       │  │ • Performance Stats │  │ • Circuit Breaker   │ │
│  │ • FTS-NG Adapter    │  │ • Degradation Alert │  │ • Recovery Handler  │ │
│  │ • SWIFT Adapter     │  │ • Health Cache      │  │ • Escalation Logic  │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │   Load Balancer     │  │   Cost Optimizer    │  │   Routing Cache     │ │
│  │                     │  │                     │  │                     │ │
│  │ • Round Robin       │  │ • Fee Calculator    │  │ • Rule Cache        │ │
│  │ • Weighted Distrib  │  │ • FX Cost Analyzer  │  │ • Score Cache       │ │
│  │ • Least Connections │  │ • Threshold Checker │  │ • Health Cache      │ │
│  │ • Capacity Manager  │  │ • Optimal Path Calc │  │ • TTL Management    │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.7.2 Supported Payment Rails

| Rail Code | Rail Name | Type | Speed | Use Cases | Cut-off |
|-----------|-----------|------|-------|-----------|---------|
| **IPP** | Instant Payment Platform | Real-time | Seconds | P2P, Urgent, Instant | 24/7 |
| **IPI** | Interbank Payment Interface | Near real-time | Minutes | Interbank, Domestic | 23:59 |
| **FTS-NG** | Funds Transfer System NG | Batch/SFTP | 48hr TAT | Financial (MT103/102/202/203/2C2), Non-Financial (CB182, Query/Answer, Free Format, Charge Claim, CB002/CB003, Statement Request, CAD, CB101/CB103, Return 202), SFTP Infrastructure | 18:30 |
| **SWIFT** | SWIFT Network | International | 1-5 days | Cross-border | Varies |
| **RTGS** | Real-Time Gross Settlement | Real-time | Seconds | High-value | 17:00 |
| **ACH** | Automated Clearing House | Batch | 1-2 days | Payroll, Government | 15:00 |

> **FTS-NG Technical Architecture:** All FTS messages are file-based and uploaded to / downloaded from the Central Bank (CB) SFTP server. A 48-hour Turnaround Time (TAT) applies for responses. Both outward (sending) and inward (receiving) flows are required for all 14 FTS message types:
> - **Financial Messages (5 types):** MT103 Single Customer Transfer, MT102 Bulk Customer Transfer, MT202 Single Institution Transfer, MT203 Bulk Institution Transfer, MT2C2 Cover Transfer
> - **Non-Financial Messages (9 types):** CB182 Refund, Message Query/Answer, Free Format, Charge Claim, CB002/CB003 Exchange House, Statement Request, CAD Daily Report, CB101/CB103, Return 202
> - **Shared Infrastructure:** SFTP Integration Layer (file upload/download, retry logic, async response correlation) and Message Parser/Generator (file format processing for all message types)

#### 3.7.3 Multi-Criteria Scoring Algorithm

```
Final Score = (Cost × 0.30) + (Speed × 0.35) + (Health × 0.35)

Cost Score:
- Base fee normalized (0-100)
- FX cost for cross-currency (0-100)
- Customer-specific rate adjustments

Speed Score:
- Processing time vs. urgency requirement (0-100)
- Settlement time factor (0-100)

Health Score:
- Current availability (0-100)
- Recent success rate (0-100)
- Latency performance (0-100)
```

#### 3.7.4 Internal Flow

```
TransactionContext (Ready for Routing)
        │
        ▼
┌───────────────────────────┐
│      RULE ENGINE          │
│ • Load routing rules      │
│ • Evaluate constraints    │
│ • Calculate rule scores   │
│ • Apply time-based rules  │
└───────────┬───────────────┘
            │ List<RuleEvaluation>
            ▼
┌───────────────────────────┐
│      SMART ROUTER         │
│ • Get health scores       │
│ • Calculate cost scores   │
│ • Calculate speed scores  │
│ • Apply load balancing    │
│ • Compute final scores    │
└───────────┬───────────────┘
            │ List<RailScore>
            ▼
┌───────────────────────────┐
│     DECISION MAKER        │
│ • Select primary rail     │
│ • Select fallback rail    │
│ • Validate selections     │
│ • Update context          │
└───────────┬───────────────┘
            │ RoutingDecision
            ▼
┌───────────────────────────┐
│      RAIL EXECUTOR        │
│ • Get adapter for rail    │
│ • Execute transaction     │
│ • Handle response         │
│ • Fallback if needed      │
│ • Update execution result │
└───────────┬───────────────┘
            │ ExecutionResult
            ▼
   TransactionContext (Updated)
```

#### 3.7.5 Performance Targets

| Metric | Target |
|--------|--------|
| Rule Evaluation | < 2ms |
| Score Calculation | < 3ms |
| Decision Making | < 1ms |
| **Total Routing (p99)** | **< 5ms** |
| Execution (rail-dependent) | Rail SLA |
| Fallback Latency | < 50ms |

---

## 4. Domain Model

### 4.1 Core Entity Relationships

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PAYMENT ORCHESTRATION SERVICE DOMAIN MODEL                │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────┐      1       1   ┌─────────────────────────┐
│    PaymentRequest       │◆─────────────────│   TransactionContext    │
├─────────────────────────┤                  ├─────────────────────────┤
│ - requestId: UUID       │                  │ - contextId: UUID       │
│ - transactionId: UUID   │                  │ - transactionId: UUID   │
│ - idempotencyKey: String│                  │ - version: Long         │
│ - channelType: Enum     │                  │ - stage: ContextStage   │
│ - paymentType: Enum     │                  │ - status: ContextStatus │
│ - amount: BigDecimal    │                  │ - createdAt: Timestamp  │
│ - currency: String      │                  └─────────────────────────┘
│ - sourceAccount: String │                            │
│ - destAccount: String   │                            │ 1
│ - valueDate: Date       │                            ▼
│ - priority: Enum        │      ┌─────────────────────────────────────┐
│ - receivedAt: Timestamp │      │      Composed Context Sections      │
└─────────────────────────┘      │                                     │
                                 │  ┌───────────────────────────────┐  │
                                 │  │ CoreTransactionData           │  │
                                 │  │ PaymentDetails                │  │
                                 │  │ EnrichmentContainer           │  │
                                 │  │ StageResultAggregator         │  │
                                 │  │ StatusTracker                 │  │
                                 │  │ RetryTracker                  │  │
                                 │  │ AuditBuffer                   │  │
                                 │  │ ExtensionData                 │  │
                                 │  └───────────────────────────────┘  │
                                 └─────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    STAGE RESULT ENTITIES                                     │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────┐  ┌─────────────────────────┐  ┌───────────────────┐
│   ValidationResult      │  │   LimitResult           │  │ QueueAssignment   │
├─────────────────────────┤  ├─────────────────────────┤  ├───────────────────┤
│ - isValid: Boolean      │  │ - decision: Enum        │  │ - queueName: Str  │
│ - validationStage: Int  │  │ - limitsChecked: List   │  │ - priority: Int   │
│ - errors: List<Error>   │  │ - utilizationPct: Dec   │  │ - scheduledTime   │
│ - warningFlags: List    │  │ - approvalRequired: Bool│  │ - processingMode  │
│ - validatedAt: Timestamp│  │ - evaluatedAt: Timestamp│  │ - assignedAt: TS  │
└─────────────────────────┘  └─────────────────────────┘  └───────────────────┘

┌─────────────────────────┐  ┌─────────────────────────┐
│   RoutingDecision       │  │   ExecutionResult       │
├─────────────────────────┤  ├─────────────────────────┤
│ - decisionId: UUID      │  │ - executionId: UUID     │
│ - primaryRail: Rail     │  │ - status: Enum          │
│ - primaryScore: Decimal │  │ - railTransactionId: Str│
│ - fallbackRail: Rail    │  │ - executedAt: Timestamp │
│ - scoreBreakdown: JSON  │  │ - settlementDate: Date  │
│ - decisionTimestamp: TS │  │ - durationMs: Long      │
└─────────────────────────┘  └─────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    ROUTING & RAIL ENTITIES                                   │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────┐      1       *   ┌─────────────────────────┐
│    PaymentRail          │◆─────────────────│   RailCapability        │
├─────────────────────────┤                  ├─────────────────────────┤
│ - railCode: String      │                  │ - currency: String      │
│ - railName: String      │                  │ - minAmount: BigDecimal │
│ - railType: Enum        │                  │ - maxAmount: BigDecimal │
│ - cutoffTime: LocalTime │                  │ - supportedCountries: []│
│ - baseFee: BigDecimal   │                  │ - processingSpeed: Enum │
│ - healthStatus: Enum    │                  │ - availableHours: String│
└─────────────────────────┘                  └─────────────────────────┘

┌─────────────────────────┐      1       *   ┌─────────────────────────┐
│    RoutingRule          │◆─────────────────│   RuleCondition         │
├─────────────────────────┤                  ├─────────────────────────┤
│ - ruleId: UUID          │                  │ - fieldName: String     │
│ - ruleName: String      │                  │ - operator: Enum        │
│ - priority: Integer     │                  │ - expectedValue: String │
│ - targetRail: String    │                  │ - dataType: Enum        │
│ - baseScore: BigDecimal │                  │ - isRequired: Boolean   │
│ - isActive: Boolean     │                  └─────────────────────────┘
└─────────────────────────┘
```

### 4.2 Enumerations

```java
// Channel Types
enum ChannelType { API, GRAPHQL, MESSAGE_QUEUE, FILE, WEBHOOK }

// Payment Types
enum PaymentType { P2P, B2B, PAYROLL, BILL_PAY, INTERNATIONAL, INSTANT }

// Transaction Status
enum TransactionStatus {
    RECEIVED, VALIDATING, VALIDATED, VALIDATION_ERROR, REJECTED,
    LIMIT_CHECK, LIMIT_ERROR, BLOCKED, PENDING_APPROVAL,
    QUEUED, READY_FOR_ROUTING, ROUTING, ROUTED, ROUTING_ERROR, NO_RAIL,
    EXECUTING, COMPLETED, EXEC_FAILED, EXEC_ERROR, PENDING_SETTLEMENT,
    RETRYING, CANCELLED, EXPIRED
}

// Limit Decision
enum LimitDecision { ALLOW, BLOCK, WARN, APPROVAL_REQUIRED }

// Queue Types
enum QueueType { PRIORITY, STANDARD, BATCH, DEAD_LETTER }

// Processing Mode
enum ProcessingMode { IMMEDIATE, DEFERRED, SCHEDULED, RETRY }

// Rail Types
enum RailType { REAL_TIME, NEAR_REAL_TIME, BATCH, INTERNATIONAL }

// Execution Status
enum ExecutionStatus { SUCCESS, FAILED, PENDING, TIMEOUT, REJECTED_BY_RAIL }
```

---

## 5. Data Architecture

### 5.1 Database Design

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DATA ARCHITECTURE                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     PRIMARY DATABASE (PostgreSQL)                    │   │
│  │                                                                      │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │   │
│  │  │ transactions    │  │ transaction_    │  │ idempotency_       │  │   │
│  │  │                 │  │ status_history  │  │ records            │  │   │
│  │  │ • id (PK)       │  │                 │  │                    │  │   │
│  │  │ • context_data  │  │ • txn_id (FK)   │  │ • key (PK)         │  │   │
│  │  │ • status        │  │ • status        │  │ • txn_id           │  │   │
│  │  │ • created_at    │  │ • changed_at    │  │ • response_cache   │  │   │
│  │  │ • updated_at    │  │ • reason        │  │ • expires_at       │  │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────┘  │   │
│  │                                                                      │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │   │
│  │  │ routing_        │  │ rail_           │  │ batch_jobs         │  │   │
│  │  │ decisions       │  │ executions      │  │                    │  │   │
│  │  │                 │  │                 │  │ • batch_id (PK)    │  │   │
│  │  │ • txn_id (FK)   │  │ • exec_id (PK)  │  │ • status           │  │   │
│  │  │ • primary_rail  │  │ • txn_id (FK)   │  │ • total_count      │  │   │
│  │  │ • fallback_rail │  │ • rail_code     │  │ • processed_count  │  │   │
│  │  │ • scores        │  │ • response      │  │ • error_summary    │  │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                       REDIS CACHE CLUSTER                            │   │
│  │                                                                      │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │   │
│  │  │ context_cache   │  │ limit_cache     │  │ routing_cache       │  │   │
│  │  │                 │  │                 │  │                     │  │   │
│  │  │ • Active txn    │  │ • Limit defs    │  │ • Routing rules     │  │   │
│  │  │   contexts      │  │ • Usage counters│  │ • Rail health       │  │   │
│  │  │ • Write-through │  │ • Lock holds    │  │ • Score cache       │  │   │
│  │  │   caching       │  │                 │  │ • Participant data  │  │   │
│  │  │ TTL: 30 min     │  │ TTL: Varies     │  │ TTL: 5 min          │  │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    KAFKA EVENT STREAMS                               │   │
│  │                                                                      │   │
│  │  payment.received  │  payment.validated  │  payment.routed          │   │
│  │  payment.executed  │  payment.completed  │  payment.failed          │   │
│  │  payment.status.changed  │  audit.events  │  dlq.payments           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Key Tables Schema

```sql
-- Transactions Table
CREATE TABLE transactions (
    id UUID PRIMARY KEY,
    correlation_id VARCHAR(64) NOT NULL,
    idempotency_key VARCHAR(128) UNIQUE,
    channel_type VARCHAR(20) NOT NULL,
    payment_type VARCHAR(20) NOT NULL,
    amount DECIMAL(18,2) NOT NULL,
    currency CHAR(3) NOT NULL,
    source_account JSONB NOT NULL,
    destination_account JSONB NOT NULL,
    value_date DATE NOT NULL,
    status VARCHAR(30) NOT NULL,
    context_data JSONB NOT NULL,
    stage_results JSONB,
    routing_decision JSONB,
    execution_result JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    version BIGINT DEFAULT 0
);

-- Indexes for performance
CREATE INDEX idx_txn_status ON transactions(status);
CREATE INDEX idx_txn_created ON transactions(created_at);
CREATE INDEX idx_txn_correlation ON transactions(correlation_id);
CREATE INDEX idx_txn_idempotency ON transactions(idempotency_key);

-- Transaction Status History
CREATE TABLE transaction_status_history (
    id UUID PRIMARY KEY,
    transaction_id UUID REFERENCES transactions(id),
    status VARCHAR(30) NOT NULL,
    previous_status VARCHAR(30),
    reason VARCHAR(255),
    changed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    changed_by VARCHAR(64)
);

-- Audit Log (append-only)
CREATE TABLE audit_log (
    id UUID PRIMARY KEY,
    transaction_id UUID,
    correlation_id VARCHAR(64),
    event_type VARCHAR(50) NOT NULL,
    stage VARCHAR(30) NOT NULL,
    action VARCHAR(50) NOT NULL,
    actor VARCHAR(64),
    details JSONB,
    timestamp TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

## 6. API Design

### 6.1 REST API Endpoints

```yaml
openapi: 3.0.3
info:
  title: Payment Orchestration Service API
  version: 1.0.0
  description: Unified API for payment processing

servers:
  - url: https://api.paymenthub.com/orchestration/v1
    description: Production

paths:
  /payments:
    post:
      summary: Create a new payment
      operationId: createPayment
      tags: [Payments]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/PaymentRequest'
      responses:
        '202':
          description: Payment accepted for processing
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PaymentResponse'
        '400':
          description: Validation error
        '409':
          description: Duplicate payment (idempotency key exists)

  /payments/{transactionId}:
    get:
      summary: Get payment status
      operationId: getPayment
      tags: [Payments]
      parameters:
        - name: transactionId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Payment details
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PaymentDetails'

  /payments/{transactionId}/status:
    get:
      summary: Get payment status history
      operationId: getPaymentStatus
      tags: [Payments]
      parameters:
        - name: transactionId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Status history
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/StatusHistory'

  /payments/batch:
    post:
      summary: Submit batch payment file
      operationId: submitBatch
      tags: [Batch]
      requestBody:
        required: true
        content:
          multipart/form-data:
            schema:
              type: object
              properties:
                file:
                  type: string
                  format: binary
                format:
                  type: string
                  enum: [CSV, XML, ISO20022]
      responses:
        '202':
          description: Batch accepted
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/BatchResponse'

  /batches/{batchId}:
    get:
      summary: Get batch status
      operationId: getBatchStatus
      tags: [Batch]
      parameters:
        - name: batchId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Batch status
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/BatchStatus'

  /health:
    get:
      summary: Service health check
      operationId: healthCheck
      tags: [Health]
      responses:
        '200':
          description: Service healthy

  /metrics:
    get:
      summary: Prometheus metrics
      operationId: getMetrics
      tags: [Monitoring]
      responses:
        '200':
          description: Metrics in Prometheus format

components:
  schemas:
    PaymentRequest:
      type: object
      required:
        - amount
        - currency
        - sourceAccount
        - destinationAccount
      properties:
        idempotencyKey:
          type: string
          maxLength: 128
        paymentType:
          type: string
          enum: [P2P, B2B, PAYROLL, BILL_PAY, INTERNATIONAL, INSTANT]
        amount:
          type: number
          format: decimal
        currency:
          type: string
          pattern: '^[A-Z]{3}$'
        sourceAccount:
          $ref: '#/components/schemas/AccountInfo'
        destinationAccount:
          $ref: '#/components/schemas/AccountInfo'
        valueDate:
          type: string
          format: date
        remittanceInfo:
          type: string
          maxLength: 140
        priority:
          type: string
          enum: [HIGH, NORMAL, LOW]

    PaymentResponse:
      type: object
      properties:
        transactionId:
          type: string
          format: uuid
        correlationId:
          type: string
        status:
          type: string
        message:
          type: string
        receivedAt:
          type: string
          format: date-time

    AccountInfo:
      type: object
      properties:
        accountNumber:
          type: string
        accountType:
          type: string
        bankCode:
          type: string
        accountName:
          type: string
        countryCode:
          type: string
```

### 6.2 GraphQL Schema

```graphql
type Query {
  payment(transactionId: ID!): Payment
  paymentByCorrelation(correlationId: String!): Payment
  payments(filter: PaymentFilter, pagination: Pagination): PaymentConnection
  batchStatus(batchId: ID!): BatchStatus
}

type Mutation {
  createPayment(input: PaymentInput!): PaymentResult!
  cancelPayment(transactionId: ID!, reason: String!): CancelResult!
}

type Subscription {
  paymentStatusChanged(transactionId: ID!): StatusChangeEvent!
}

type Payment {
  transactionId: ID!
  correlationId: String!
  status: TransactionStatus!
  amount: Decimal!
  currency: String!
  sourceAccount: AccountInfo!
  destinationAccount: AccountInfo!
  createdAt: DateTime!
  statusHistory: [StatusChange!]!
  routingDecision: RoutingDecision
  executionResult: ExecutionResult
}

input PaymentInput {
  idempotencyKey: String
  paymentType: PaymentType!
  amount: Decimal!
  currency: String!
  sourceAccount: AccountInfoInput!
  destinationAccount: AccountInfoInput!
  valueDate: Date
  remittanceInfo: String
  priority: Priority
}

enum TransactionStatus {
  RECEIVED
  VALIDATING
  VALIDATED
  QUEUED
  ROUTING
  EXECUTING
  COMPLETED
  FAILED
}
```

---

## 7. Event-Driven Architecture

### 7.1 Event Topics

| Topic | Publisher | Consumers | Purpose |
|-------|-----------|-----------|---------|
| `payment.received` | Gateway Core | Audit, Monitoring | New payment received |
| `payment.validated` | Validation Pipeline | Audit, Analytics | Validation completed |
| `payment.limit.checked` | Limit Control | Audit, Alerts | Limit check completed |
| `payment.queued` | Queue Management | Monitoring | Payment queued |
| `payment.status.changed` | Status Manager | Notification, Webhook | Status change event |
| `payment.routed` | Routing Engine | Audit, Analytics | Routing decision made |
| `payment.executed` | Rail Executor | Audit, Reconciliation | Rail execution result |
| `payment.completed` | Status Manager | Notification, Settlement | Payment finalized |
| `payment.failed` | Status Manager | Notification, DLQ, Ops | Payment failed |
| `audit.events` | All Modules | Audit Service | Compliance audit trail |
| `dlq.payments` | Queue/Status | DLQ Processor | Dead letter events |

### 7.2 Event Schema

```json
{
  "eventId": "uuid",
  "eventType": "payment.status.changed",
  "eventTime": "2026-02-23T10:30:00Z",
  "correlationId": "correlation-uuid",
  "transactionId": "transaction-uuid",
  "payload": {
    "previousStatus": "ROUTING",
    "newStatus": "EXECUTING",
    "reason": "Routed to IPP rail",
    "metadata": {
      "rail": "IPP",
      "estimatedCompletion": "2026-02-23T10:30:05Z"
    }
  },
  "source": "payment-orchestration-service",
  "version": "1.0"
}
```

---

## 8. Processing Flow & Sequence Diagrams

### 8.1 Happy Path - Real-Time Payment

```
┌────────┐  ┌─────────┐  ┌──────────┐  ┌───────┐  ┌───────┐  ┌────────┐  ┌────────┐  ┌─────────┐
│ Client │  │ Gateway │  │Validation│  │ Limit │  │ Queue │  │ Status │  │Routing │  │  Rail   │
│        │  │  Core   │  │ Pipeline │  │Control│  │  Mgmt │  │  Mgr   │  │ Engine │  │Executor │
└───┬────┘  └────┬────┘  └────┬─────┘  └───┬───┘  └───┬───┘  └───┬────┘  └───┬────┘  └────┬────┘
    │            │            │            │          │          │           │            │
    │ POST /payments          │            │          │          │           │            │
    │───────────>│            │            │          │          │           │            │
    │            │ Create Context          │          │          │           │            │
    │            │──────┐     │            │          │          │           │            │
    │            │<─────┘     │            │          │          │           │            │
    │            │ Validate(ctx)           │          │          │           │            │
    │            │───────────>│            │          │          │           │            │
    │            │            │ Stage 1-4  │          │          │           │            │
    │            │            │──────┐     │          │          │           │            │
    │            │            │<─────┘     │          │          │           │            │
    │            │            │ CheckLimits(ctx)      │          │           │            │
    │            │            │───────────>│          │          │           │            │
    │            │            │            │ Step 1-3 │          │           │            │
    │            │            │            │──────┐   │          │           │            │
    │            │            │            │<─────┘   │          │           │            │
    │            │            │            │ Queue(ctx)          │           │            │
    │            │            │            │─────────>│          │           │            │
    │            │            │            │          │ Priority │           │            │
    │            │            │            │          │──────┐   │           │            │
    │            │            │            │          │<─────┘   │           │            │
    │            │            │            │          │UpdateStatus(ctx)     │            │
    │            │            │            │          │─────────>│           │            │
    │            │            │            │          │          │ Route(ctx)│            │
    │            │            │            │          │          │──────────>│            │
    │            │            │            │          │          │           │ Score Rails│
    │            │            │            │          │          │           │──────┐     │
    │            │            │            │          │          │           │<─────┘     │
    │            │            │            │          │          │           │ Execute(ctx)
    │            │            │            │          │          │           │───────────>│
    │            │            │            │          │          │           │            │ Call Rail
    │            │            │            │          │          │           │            │──────┐
    │            │            │            │          │          │           │            │<─────┘
    │            │            │            │          │          │           │<───────────│
    │            │            │            │          │          │<──────────│            │
    │<───────────│            │            │          │          │           │            │
    │ 202 Accepted            │            │          │          │           │            │
    │ (transactionId)         │            │          │          │           │            │
```

### 8.2 Fallback Scenario

```
┌─────────┐  ┌────────┐  ┌──────────┐  ┌────────┐
│ Status  │  │Routing │  │ Primary  │  │Fallback│
│ Manager │  │ Engine │  │  Rail    │  │  Rail  │
└────┬────┘  └───┬────┘  └────┬─────┘  └───┬────┘
     │           │            │            │
     │ Ready for Routing      │            │
     │──────────>│            │            │
     │           │ Execute Primary         │
     │           │───────────>│            │
     │           │            │ TIMEOUT/ERR│
     │           │<───────────│            │
     │           │ Execute Fallback        │
     │           │────────────────────────>│
     │           │            │            │ SUCCESS
     │           │<────────────────────────│
     │<──────────│            │            │
     │ Update Status (COMPLETED)          │
```

---

## 9. Integration Points

### 9.1 External Integrations

| Integration | Protocol | Direction | Purpose |
|-------------|----------|-----------|---------|
| **Configuration Domain** | gRPC/REST | Outbound | Limit definitions, rules, holidays, cut-offs |
| **Risk & Compliance Domain** | gRPC | Outbound | AML screening, fraud detection, sanctions check |
| **IPP Rail** | REST/WebSocket | Outbound | Instant payment execution |
| **IPI Rail** | ISO20022/MQ | Outbound | Interbank transfer execution |
| **FTS-NG Rail** | SFTP/MQ | Outbound | Batch payment execution |
| **SWIFT Rail** | SWIFT MX | Outbound | Cross-border payment execution |
| **Notification Service** | Kafka | Outbound | User notifications |
| **Audit Service** | Kafka | Outbound | Compliance audit logging |
| **Monitoring (Prometheus)** | HTTP | Inbound | Metrics scraping |

### 9.2 Internal Module Integration

All modules within the Payment Orchestration Service communicate via:
- **Direct Method Calls**: Zero network overhead, shared TransactionContext
- **Shared Memory**: TransactionContext passed by reference
- **In-Process Events**: For loose coupling where needed

---

## 10. Security Design

### 10.1 Authentication & Authorization

| Layer | Mechanism |
|-------|-----------|
| API Gateway | OAuth 2.0 / JWT validation |
| Service-to-Service | mTLS + Service Auth |
| Database | Role-based access control |
| Encryption | TLS 1.3 in transit, AES-256 at rest |

### 10.2 Data Protection

- **PII Masking**: Account numbers, names masked in logs
- **Field-Level Encryption**: Sensitive fields encrypted in database
- **Audit Trail**: Immutable audit log with tamper detection
- **Key Management**: HSM-backed key management

---

## 11. Non-Functional Requirements

### 11.1 Performance Requirements

| Metric | Requirement |
|--------|-------------|
| End-to-End Latency (p99) | < 150ms (real-time) |
| Throughput | 7,500+ TPS |
| Availability | 99.99% |
| Recovery Time (RTO) | < 5 minutes |
| Recovery Point (RPO) | < 1 second |

### 11.2 Scalability

| Dimension | Capability |
|-----------|------------|
| Horizontal Scaling | Auto-scale 2-20 instances |
| Database | Read replicas, connection pooling |
| Cache | Redis cluster with sharding |
| Queue | Kafka with partitioning |

---

## 12. Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         KUBERNETES CLUSTER                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    INGRESS / LOAD BALANCER                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│  ┌─────────────────────────────────▼───────────────────────────────────┐   │
│  │              PAYMENT ORCHESTRATION SERVICE                           │   │
│  │                    (Deployment: 3-10 replicas)                       │   │
│  │                                                                      │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │   │
│  │  │   Pod 1      │  │   Pod 2      │  │   Pod N      │               │   │
│  │  │              │  │              │  │              │               │   │
│  │  │ • Gateway    │  │ • Gateway    │  │ • Gateway    │               │   │
│  │  │ • Validation │  │ • Validation │  │ • Validation │               │   │
│  │  │ • Limits     │  │ • Limits     │  │ • Limits     │               │   │
│  │  │ • Queue      │  │ • Queue      │  │ • Queue      │               │   │
│  │  │ • Status     │  │ • Status     │  │ • Status     │               │   │
│  │  │ • Routing    │  │ • Routing    │  │ • Routing    │               │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐         │
│  │  PostgreSQL      │  │  Redis Cluster   │  │  Kafka Cluster   │         │
│  │  (Primary +      │  │  (3 nodes)       │  │  (3 brokers)     │         │
│  │   Replicas)      │  │                  │  │                  │         │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 12.1 Resource Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 2 cores | 4 cores |
| Memory | 4 GB | 8 GB |
| Replicas | 3 | 5-10 |
| Storage | 100 GB SSD | 500 GB SSD |

---

## 13. Monitoring & Alerting

### 13.1 Key Metrics

| Category | Metrics |
|----------|---------|
| **Throughput** | Requests/sec, TPS by channel, batch processing rate |
| **Latency** | p50, p95, p99 for each stage and overall |
| **Errors** | Error rate by stage, validation failure rate, execution failures |
| **Business** | Payment volume, success rate by rail, cost per transaction |
| **Resources** | CPU, memory, connection pool, cache hit rate |

### 13.2 Alert Thresholds

| Alert | Condition | Severity |
|-------|-----------|----------|
| High Latency | p99 > 200ms | Warning |
| Error Spike | Error rate > 1% | Critical |
| Queue Backlog | Depth > 10,000 | Warning |
| Rail Degradation | Success rate < 95% | Critical |
| SLA Breach Risk | Approaching deadline | High |

### 13.3 Dashboards

- **Operations Dashboard**: Real-time throughput, latency, error rates
- **Business Dashboard**: Payment volumes, success rates, costs
- **SLA Dashboard**: SLA compliance, breach tracking
- **Rail Health Dashboard**: Rail availability, performance by rail

---

## 14. Test Strategy

### 14.1 Test Levels

| Level | Scope | Tools |
|-------|-------|-------|
| Unit Tests | Individual components | JUnit, Mockito |
| Integration Tests | Module interactions | TestContainers |
| Contract Tests | API contracts | Pact |
| E2E Tests | Full payment flow | Cucumber, Karate |
| Performance Tests | Load, stress | Gatling, k6 |
| Chaos Tests | Failure scenarios | Chaos Monkey |

### 14.2 Test Coverage Targets

| Component | Coverage Target |
|-----------|-----------------|
| Core Business Logic | > 90% |
| API Layer | > 85% |
| Integration Points | > 80% |
| Overall Service | > 85% |

---

## 15. Implementation Roadmap

### Phase 1: Foundation (Weeks 1-4)
- [ ] TransactionContext implementation
- [ ] Payment Gateway Core with REST API
- [ ] Basic validation pipeline (Schema + Business Rules)
- [ ] Database schema and migrations

### Phase 2: Core Processing (Weeks 5-8)
- [ ] Complete validation pipeline (4 stages)
- [ ] Limit Control Engine (3 steps)
- [ ] Queue Management
- [ ] Status & Lifecycle Management

### Phase 3: Routing & Execution (Weeks 9-12)
- [ ] Embedded Routing Engine
- [ ] Rail adapters (IPP, IPI)
- [ ] Fallback handling
- [ ] Event publishing

### Phase 4: Production Readiness (Weeks 13-16)
- [ ] Additional rail adapters (FTS-NG, SWIFT)
- [ ] Batch processing
- [ ] Performance optimization
- [ ] Monitoring & alerting
- [ ] Documentation & training

---

## Appendix A: Related Documents

| Document | Location |
|----------|----------|
| ADR-005a: Unified Orchestration Domain | /docs/adr/ADR-005a.md |
| Payment Gateway Context Overview | PaymentGateway_Context_Overview.md |
| Configuration Domain Design | ConfigurationDomain/ |
| Risk & Compliance Domain Design | Risk&ComplinceDomain/ |

---

## Appendix B: Glossary

| Term | Definition |
|------|------------|
| **TransactionContext** | Shared state object containing all transaction data |
| **Rail** | Payment network/channel (IPP, IPI, SWIFT, etc.) |
| **Idempotency** | Guaranteeing same result for duplicate requests |
| **DLQ** | Dead Letter Queue for failed transactions |
| **SLA** | Service Level Agreement for processing time |
| **Circuit Breaker** | Pattern to prevent cascading failures |

---

*Document Version: 2.0 | Last Updated: February 23, 2026*
