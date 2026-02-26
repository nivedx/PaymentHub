# Payment Gateway Context - Detailed Architecture

## Overview
This document provides a comprehensive view of the **Payment Orchestration Bounded Context**, detailing the complete payment processing flow from entry channels through orchestration, validation, routing, and execution. Following **ADR-005a**, the architecture now implements a **unified Payment Orchestration Domain** that consolidates Payment Orchestration and Routing & Decision into a single high-performance service, eliminating inter-service network latency and enabling shared persistent context with caching across all processing stages for improved disaster recovery.

## Architecture Flow
```
Entry Channels → ┌─────────────────────────────────────────────────────────────┐
                 │              PAYMENT ORCHESTRATION DOMAIN                   │
                 │                                                             │
                 │  Payment Gateway Core → Validation Pipeline (4 stages) →   │
                 │  Limit Control Engine (3 steps) → Queue Management →       │
                 │  Status & Lifecycle → Embedded Routing Engine →            │
                 │  Rail Execution                                            │
                 └─────────────────────────────────────────────────────────────┘
                                    ↓                     ↓                    ↓
                         Configuration Domain   Risk & Compliance   Platform Services
                                                          ↓
                                                   Payment Rails
```

### Performance Benefits (ADR-005a)
| Metric | Before (Separate) | After (Unified) | Improvement |
|--------|-------------------|-----------------|-------------|
| **Latency (p99)** | 200ms | 150ms | 25% faster |
| **Throughput** | 5,000 TPS | 7,500 TPS | 50% increase |
| **Network Hops** | 3 network calls | 0 network calls | 100% reduction |
| **Infrastructure** | 2 services | 1 service | ~30% cost reduction |

---

## 1. Entry Channels

The system accepts payment requests through multiple entry points:

### 🚪 API Gateway
- **Protocol**: REST/GraphQL
- **Usage**: Real-time payment initiation
- **Features**: Synchronous request-response, immediate validation feedback

### 📨 Message Queue Gateway
- **Protocol**: Async/Event-Driven (Kafka, RabbitMQ)
- **Usage**: High-volume asynchronous processing
- **Features**: Decoupled processing, guaranteed delivery, event streaming

### 📁 File Gateway
- **Protocol**: Batch/SFTP
- **Usage**: Bulk payment file processing
- **Features**: Scheduled batch processing, file format validation, reconciliation

---

## 2. Payment Orchestration Domain

> **⚡ Architecture Update (ADR-005a)**: The Payment Orchestration Domain and Routing & Decision Domain have been consolidated into a **unified Payment Orchestration Domain** for improved performance. This service now handles the complete transaction lifecycle from ingestion through routing and execution using a shared **TransactionContext** object that flows through all stages without serialization.

> **📚 Detailed Documentation**: For comprehensive implementation details including domain models, API specifications, data architecture, and detailed module specifications, see [Payment Orchestration Service Design](PaymentOrchestrationDomain/Design_Payment_Orchestration_Service.md). This overview document provides a high-level summary; the unified service design document contains complete technical specifications.

### 2.1 Payment Gateway Core 🔄

The central orchestrator that receives payment requests from all entry channels and coordinates the entire payment lifecycle.

**Key Responsibilities:**
- **Request Reception**: Accept requests from API, MQ, and File channels
- **Protocol Translation**: Normalize different protocols (REST, AMQP, CSV) into unified internal format
- **Transaction Initialization**: Generate unique transaction IDs, timestamps, and initial state
- **Request Enrichment**: Add contextual data (customer info, channel metadata, trace IDs)
- **Orchestration**: Coordinate downstream validation, limit checks, and queue placement

**Integration Points:**
- Receives from: All Entry Channels
- Sends to: Validation Pipeline (first stage)
- Publishes to: Audit & Logging (all actions)

---

### 2.2 Validation Pipeline (4-Stage Sequential Process)

All payment requests pass through four mandatory validation stages in sequence. Failure at any stage rejects the transaction immediately.

#### 1️⃣ Stage 1: Schema Validation
**Purpose**: Ensure structural and format integrity
- Data format checks (JSON/XML schema compliance)
- Required field validation (mandatory data present)
- Data type verification (numeric, string, date formats)
- Structure compliance (nested object validation)
- **Uses**: Cache Service for schema definitions

#### 2️⃣ Stage 2: Business Rules Validation
**Purpose**: Apply business logic and constraints
- Payment type rules (P2P, B2B, payroll rules)
- Channel-specific rules (web vs mobile limits)
- Amount thresholds (min/max amount checks)
- Time-window restrictions (operating hours)
- **Integrates With**: Rules Repository (Configuration Domain)

#### 3️⃣ Stage 3: Reference Data Validation
**Purpose**: Validate against external reference data
- **Holiday calendar check** → Holiday Calendar (Config)
- **Cut-off time verification** → Cut-off Management (Config)
- **Participant availability** → Participant Directory (Config)
- Currency support validation

#### 4️⃣ Stage 4: Duplicate Detection
**Purpose**: Prevent duplicate payment processing
- Transaction ID uniqueness checks
- Idempotency key validation
- Amount-beneficiary-date matching
- Time-window duplicate detection (configurable window)
- **Triggers**: Fraud Detection (Risk & Compliance)

---

### 2.3 Limit Control Engine (3-Step Process) ⚖️

After successful validation, all payments undergo limit control checks to ensure they comply with defined thresholds.

#### 🔍 Step 1: Limit Lookup
**Purpose**: Retrieve applicable limits from configuration
- User-level limits (per-user daily/monthly limits)
- Account-level limits (account balance, transaction count)
- Channel-level limits (web, mobile, API limits)
- Product-level limits (payment type specific)
- **Integrates With**: 
  - Limit Definitions (Configuration Domain)
  - Cache Service (for performance optimization)

#### ⚖️ Step 2: Limit Calculation
**Purpose**: Calculate current usage and available capacity
- Current usage aggregation (today/this month totals)
- Available balance calculation (limit - usage)
- Utilization percentage (used/total × 100)
- Time-based resets (daily, weekly, monthly boundaries)
- **Integrates With**: Pricing Engine (Configuration Domain) for fee calculations

#### 🚦 Step 3: Limit Decision
**Purpose**: Make allow/block/warn decisions
- **Allow**: Proceed to queue management
- **Block**: Reject transaction, return error
- **Warn**: Proceed but notify user/ops
- Temporary limit holds (reserve limit during processing)
- Breach alert generation (notifications for threshold violations)
- Approval routing (escalate to approver if needed)
- **Triggers**: AML Screening (Risk & Compliance)

---

### 2.4 Queue Management (3-Phase Process) 📥

Approved payments are intelligently queued for processing based on priority and scheduling requirements.

#### 📋 Phase 1: Priority Assignment
**Purpose**: Calculate processing priority
- Urgency score calculation (time-sensitive payments)
- Customer segment weighting (VIP, enterprise, retail)
- Amount-based priority (high-value transactions first)
- SLA requirements (contractual processing times)
- **References**: Workflow Configs (Configuration Domain)

#### 📥 Phase 2: Queue Placement
**Purpose**: Place transaction in appropriate queue
- **Priority Queue**: Urgent/high-value transactions
- **Standard Queue**: Normal processing
- **Batch Queue**: Scheduled/low-priority bulk payments
- **Dead Letter Queue (DLQ)**: Failed items requiring manual intervention
- **Note**: Retry Handler can re-route failed payments back to appropriate queues

#### ⏱️ Phase 3: Scheduling
**Purpose**: Determine execution timing
- **Immediate**: Process now (real-time payments)
- **Deferred**: Timed release (process at specific time)
- **Scheduled**: Future-dated (standing orders, recurring payments)
- **Retry Scheduling**: Exponential backoff for failed transactions

---

### 2.5 Status & Lifecycle Management 📍

Manages the complete lifecycle of payment transactions with state tracking, event publishing, retry handling, and **inline routing** (embedded within the service).

#### 📍 Status Manager
**Purpose**: Track and manage transaction state with integrated routing
- Current state tracking (PENDING, VALIDATED, QUEUED, PROCESSING, ROUTING, EXECUTING, etc.)
- State transition validation (enforce valid state machine transitions)
- History maintenance (audit trail of all state changes)
- Timeline tracking (timestamps for SLA monitoring)
- **Inline routing trigger**: When status reaches READY_FOR_ROUTING, directly invokes Embedded Routing Engine (no network call)
- **Integrates With**:
  - Currency Management (Configuration Domain) for multi-currency tracking
  - Sanctions Check (Risk & Compliance) for compliance verification
  - Monitoring & Metrics (Platform Services) for dashboards
  - **Embedded Routing Engine** (direct method call within same service)

#### 🔔 Event Publisher
**Purpose**: Broadcast transaction status changes
- Status change events (publish to event bus)
- Notification triggers (alerts for critical states)
- Webhook dispatching (notify external systems)
- Event streaming (real-time status updates)
- **Integrates With**: Notification Service (Platform Services)

#### 🔄 Retry Handler
**Purpose**: Handle failures with intelligent retry logic
- Automatic retry logic (configurable retry attempts)
- Exponential backoff strategy (1s, 2s, 4s, 8s, ...)
- Max retry limits (prevent infinite loops)
- Dead letter queue routing (move to DLQ after max retries)
- **Flow**: Payment Rails failures → Retry Handler → Queue Placement (Phase 2) → Retry scheduling

---

### 2.6 Embedded Routing Engine 🎯

The routing logic is now embedded within the Payment Orchestration Domain, eliminating network latency and enabling direct access to the **TransactionContext**.

#### 📋 Rule Engine
**Purpose**: Evaluate routing criteria using shared context
- Rule evaluation (match transaction attributes from TransactionContext)
- Score calculation (rank available rails)
- Constraint checking (amount limits, currency support)
- Time-based routing (office hours vs after-hours)
- Rules cached with TTL-based refresh
- **Input**: TransactionContext (direct cached access with persistent backing)
- **Output**: Evaluated rules and scores for Smart Router

#### 🧭 Smart Router
**Purpose**: Select optimal payment rail with multi-criteria optimization
- Rail selection logic (IPP for instant, FTS for batch, SWIFT for cross-border)
- Multi-criteria scoring: cost, speed, reliability, health
- Load balancing (distribute across duplicate rails)
- Rail health checking (avoid degraded rails via HealthChecker)
- **Input**: Rule Engine scores + TransactionContext
- **Output**: Ranked RailScore list for Decision Maker

#### ✅ Decision Maker
**Purpose**: Final rail selection with fallback handling
- Final selection decision (choose best rail from scored options)
- Fallback handling (secondary rail if primary fails)
- Cost-benefit analysis (optimize for cost vs speed)
- Updates TransactionContext with routing decision
- **Output**: RoutingDecision with primary + fallback rail

#### 🚂 Rail Executor
**Purpose**: Execute payment on selected rail
- Execute transaction on primary rail
- Automatic fallback execution if primary fails
- Adapter pattern for different rail protocols (IPP, IPI, FTS-NG, SWIFT)
- Reports execution result back to Status Manager
- **Adapters**: IPPAdapter, IPIAdapter, FTSAdapter, SWIFTAdapter

---

### 2.7 TransactionContext (Shared State Object)

The key optimization enabling the service is the **TransactionContext** that flows through all stages without serialization:

```
TransactionContext
├── Core Transaction Data
│   ├── transactionId: String
│   ├── amount: Decimal
│   ├── currency: String
│   ├── sourceAccount: String
│   └── destinationAccount: String
│
├── Enriched Data (populated by stages)
│   ├── validationResult: ValidationResult
│   ├── limitResult: LimitCheckResult
│   ├── queueAssignment: QueueAssignment
│   └── currentStatus: StatusInfo
│
├── Routing Decision (populated by embedded routing)
│   ├── railScores: List<RailScore>
│   ├── primaryRail: SelectedRail
│   ├── fallbackRail: SelectedRail
│   └── finalDecision: RoutingDecision
│
├── Execution Tracking
│   ├── executionResult: ExecutionResult
│   └── retryHistory: List<RetryAttempt>
│
└── Metadata
    ├── createdAt: Timestamp
    ├── lastUpdatedAt: Timestamp
    └── extensionData: Map<String, Object>
```

**Benefits of TransactionContext**:
- Persistent storage with write-through caching for disaster recovery
- All stages enrich the same context object
- Single database write includes status + routing decision atomically
- Complete audit trail in one place
- Automatic state recovery on service restart or failover

---

## 3. Configuration Domain ⚙️

> **Consolidated Service**: All configuration capabilities are now delivered through the unified **Configuration Management Service** (Port 8002). See [Configuration Management Service Design](ConfigurationDomain/Design_Configuration_Management_Service.md) for complete details.

Provides static and semi-static configuration data that drives validation, limits, and business logic across the payment orchestration through a single consolidated service.

### Service Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│               CONFIGURATION MANAGEMENT SERVICE (Port 8002)                       │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  ┌───────────┐  │
│  │  Time & Calendar│  │  Reference Data │  │  Business Rules │  │ Financial │  │
│  │     Module      │  │     Module      │  │  & Workflows    │  │  Config   │  │
│  │                 │  │                 │  │     Module      │  │  Module   │  │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤  ├───────────┤  │
│  │• Holiday Cals   │  │• Participants   │  │• Rules Engine   │  │• Charges  │  │
│  │• Cut-off Times  │  │• BIC/IBAN Valid │  │• Workflows      │  │• Limits   │  │
│  │• Business Days  │  │• Currencies     │  │• Approvals      │  │• Vostro   │  │
│  │• Timezones      │  │• Bank Directory │  │• Escalations    │  │• GL Accts │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  └───────────┘  │
│                                                                                 │
│                             Shared Data Layer                                   │
│                   SQL Server │ Redis Cache │ Kafka Events                       │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 3.1 Time & Calendar Management Module

#### 📅 Holiday Calendar
**Purpose**: Multi-country, multi-region business day logic
- National, regional, and bank-specific calendars
- Working day calculations
- Weekend definitions (Fri-Sat for Middle East, Sat-Sun for others)
- Recurring holiday rules
- Rail-specific applicability
- **API**: `/api/v1/calendars`
- **Used By**: Validation Stage 3 (Reference Data Validation)

#### ⏰ Cut-off Management
**Purpose**: Time-based processing windows
- Rail-specific cut-off times (IPP: 17:00, IPI: 23:59, FTS: 18:30)
- Currency-specific cut-off times
- Customer/client-specific overrides (VIP extended windows)
- Channel-specific cut-offs
- Warning and breach alerts
- Holiday calendar integration
- **API**: `/api/v1/cutoffs`
- **Used By**: Validation Stage 3 (Reference Data Validation)

#### 📆 Business Day Calculator
**Purpose**: Settlement date computation
- Next business day calculation
- T+N calculations across jurisdictions
- Multi-calendar overlap handling
- **API**: `/api/v1/business-days`

---

### 3.2 Reference Data Management Module

#### 🏦 Participant Directory
**Purpose**: Bank and financial institution registry
- Bank identification codes (BIC, SWIFT, Routing Numbers, IFSC, Sort Codes)
- Participant classifications (Tier levels, Risk ratings, Service levels)
- Supported payment rails and capabilities per participant
- Real-time operational status monitoring
- Settlement account configuration
- **API**: `/api/v1/participants`
- **Used By**: Validation Stage 3 (Reference Data Validation)

#### 🔍 Banking Validation
**Purpose**: Payment credential validation
- BIC/SWIFT code validation
- IBAN format and checksum validation
- Country-specific account format validation
- Bank-rail capability checking
- **API**: `/api/v1/validation`

#### 💱 Reference Data
**Purpose**: Supporting reference data
- Currency codes and decimal precision (JPY: 0, USD: 2, BHD: 3)
- Country codes and sanction flags
- GL accounts and product codes
- **API**: `/api/v1/currencies`, `/api/v1/countries`
- **Used By**: Status Manager for multi-currency transaction tracking

---

### 3.3 Business Rules & Workflows Module

#### 📋 Rules Repository & Engine
**Purpose**: Centralized business rules with real-time evaluation
- Validation rules (format, range, business logic)
- Routing rules (rail selection criteria)
- Decision rules (approval thresholds, escalations)
- Transformation and enrichment rules
- Rule versioning, effective dating, and A/B testing
- Priority-based rule execution
- **API**: `/api/v1/rules`
- **Used By**: Validation Stage 2 (Business Rules Validation)

#### 🔄 Workflow Engine
**Purpose**: Process orchestration with state management
- Workflow definitions with state machines
- Approval workflows (maker-checker, multi-level approvals)
- Task assignment and delegation
- Parallel and conditional task execution
- **API**: `/api/v1/workflows`
- **Used By**: Queue Management Phase 1 (Priority Assignment)

#### ⬆️ Escalation Management
**Purpose**: Automated escalation handling
- Time-based escalation rules
- Multi-level escalation chains
- Auto-approval on timeout (configurable)
- Notification integration
- **API**: `/api/v1/escalations`

#### ⏱️ SLA Management
**Purpose**: Service level tracking
- SLA definitions per payment type/channel/customer
- Milestone-based tracking
- Breach detection and alerting
- Performance reporting
- **API**: `/api/v1/slas`

---

### 3.4 Financial Configuration Module

#### 💲 Charge & Pricing Engine
**Purpose**: Dynamic fee calculation
- Charge definitions by payment type/rail/currency
- Tiered/slab-based pricing models
- Customer-specific pricing (negotiated rates)
- Promotional pricing with date ranges
- Pre/post-processing charge timing
- **API**: `/api/v1/charges`
- **Used By**: Limit Calculation (Step 2)

#### 🎯 Limit Management
**Purpose**: Multi-dimensional limit enforcement
- Limit levels: User, Account, Customer, Channel, Product, System
- Limit types: Transaction count, Amount, Cumulative, Velocity, Concentration
- Time dimensions: Per-transaction, Daily, Weekly, Monthly, Yearly
- Real-time utilization tracking
- Breach actions: Block, Warn, Route to Approval, Escalate
- Temporary limit overrides with approval workflow
- **API**: `/api/v1/limits`
- **Used By**: Limit Lookup (Step 1) and Cache Service

#### 🏦 Vostro Account Management
**Purpose**: Settlement account configuration
- Multi-currency account registry
- Real-time balance tracking (Book, Available, Held)
- Balance thresholds and alerts
- Daily limit configuration
- **API**: `/api/v1/vostro`

#### 📊 GL Integration
**Purpose**: General Ledger mapping
- Payment type to GL code mapping
- Automated entry generation
- Currency position tracking
- **API**: `/api/v1/gl`

---

## 4. Routing & Decision (Embedded in Service) 🎯

> **⚡ Architecture Note (ADR-005a)**: The Routing & Decision capabilities are now **embedded within the Payment Orchestration Domain** as described in Section 2.6. This section documents the routing logic for reference, which is implemented as internal components rather than a separate domain.

Makes intelligent decisions on which payment rail to use based on transaction attributes, participant capabilities, and optimization criteria. All routing components have direct cached access to the **TransactionContext** with persistent backing for disaster recovery.

### Embedded Routing Components

The following components are internal modules within the Payment Orchestration Domain:

### 📋 Rule Engine (Embedded)
**Purpose**: Evaluate routing criteria and calculate scores
- Rule evaluation (match transaction attributes from shared context)
- Score calculation (rank available rails based on weighted criteria)
- Constraint checking (amount limits, currency support, rail availability)
- Time-based routing (office hours vs after-hours rail selection)
- **Caching**: Rules loaded with @Cacheable for performance
- **Input**: TransactionContext (direct access - no serialization)
- **Output**: List<RuleEvaluation> with scores for Smart Router

### 🧭 Smart Router (Embedded)
**Purpose**: Select optimal payment rail with multi-criteria optimization
- Rail selection logic (IPP for instant, FTS for batch, SWIFT for cross-border)
- Multi-criteria optimization:
  - **Rule Score**: Base score from rule evaluation
  - **Health Score**: Real-time rail availability from HealthChecker
  - **Cost Score**: Transaction cost calculation
  - **Speed Score**: Average processing time
  - **Weighted Final Score**: Combined scoring with configurable weights
- Load balancing (distribute across equivalent rails)
- Rail health checking (avoid degraded rails)
- **Input**: Rule Engine evaluations + TransactionContext
- **Output**: List<RailScore> ranked by finalScore for Decision Maker

### ✅ Decision Maker (Embedded)
**Purpose**: Final rail selection with fallback handling
- Final selection decision (choose highest-scored rail)
- Fallback handling (select second-best rail as backup)
- Cost-benefit analysis (optimize for cost vs speed based on transaction type)
- Updates TransactionContext with RoutingDecision directly
- **Output**: RoutingDecision containing:
  - primaryRail + primaryScore
  - fallbackRail + fallbackScore
  - decisionTimestamp
  - costEstimate
  - expectedProcessingTime

### Performance Benefits (vs Separate Domain)

| Aspect | Separate Domain | Embedded Routing | Improvement |
|--------|-----------------|------------------|-------------|
| **Network Latency** | 10-30ms per call | 0ms | 100% eliminated |
| **Serialization** | JSON encode/decode | Direct object pass | 5ms saved |
| **Data Transfer** | Full context re-transmitted | Shared context | Zero overhead |
| **Transaction Scope** | Distributed | Single atomic | Consistency |
| **Scaling** | Independent scaling | Unified scaling | Simplified |

**See Also**: 
- [Section 2.6: Embedded Routing Engine](#26-embedded-routing-engine-) for implementation details
- [Payment Orchestration Service Design](PaymentOrchestrationDomain/Design_Payment_Orchestration_Service.md) for full architecture

---

## 5. Risk & Compliance Domain 🛡️

Ensures regulatory compliance and fraud prevention across all payment transactions.

### 🕵️ AML Screening (Anti-Money Laundering)
**Purpose**: Detect suspicious financial activity
- Pattern analysis (structuring, smurfing detection)
- Threshold monitoring (large transaction alerts)
- Watch list screening (known bad actors)
- Regulatory reporting (CTR, SAR filing)
- **Triggered By**: Limit Decision (Step 3)

### 🚨 Fraud Detection
**Purpose**: Real-time fraud scoring and prevention
- Behavioral analytics (anomaly detection)
- Device fingerprinting (recognize suspicious devices)
- Velocity checks (rapid-fire transactions)
- ML-based scoring (fraud probability models)
- **Triggered By**: Duplicate Detection (Validation Stage 4)

### 🚫 Sanctions Check
**Purpose**: Compliance with international sanctions
- OFAC screening (US sanctions)
- UN/EU sanctions lists
- Country/region restrictions
- Entity and individual screening
- **Triggered By**: Status Manager (before routing)

---

## 6. Platform Services Domain 🔧

Cross-cutting concerns that support the entire payment processing ecosystem.

### 📧 Notification Service
**Purpose**: Multi-channel alerting and communication
- Email notifications (transaction receipts, alerts)
- SMS alerts (OTP, status updates)
- Push notifications (mobile app alerts)
- Webhook delivery (external system integration)
- **Triggered By**: Event Publisher (Status & Lifecycle)

### 📝 Audit & Logging
**Purpose**: Immutable audit trail and compliance logging
- Complete transaction logging (all actions and decisions)
- Change tracking (who, what, when, why)
- Regulatory compliance (tamper-proof logs)
- Log retention policies (archive and purge)
- Long-term storage (cold storage for historical data)
- **Triggered By**: Payment Gateway Core (logs all actions)

### 📊 Monitoring & Metrics
**Purpose**: Real-time performance tracking and alerting
- Performance monitoring (latency, throughput, error rates)
- Business metrics (success rates, revenue, volumes)
- Infrastructure health (CPU, memory, disk, network)
- Alerting and dashboards (Grafana, Prometheus, Datadog)
- **Triggered By**: Status Manager for real-time metrics

### ⚡ Cache Service
**Purpose**: Performance optimization through caching
- Hot data caching (frequently accessed config)
- TTL-based expiration (time-to-live for each cache type)
- Cache invalidation (refresh on config changes)
- Distributed caching (Redis, Hazelcast)
- **Used By**: 
  - Schema Validation (cache schema definitions)
  - Limit Lookup (cache limit configurations)

---

## 7. Payment Processing Rails 💳

The final execution layer where payments are processed through various payment networks.

### 💸 IPP (Instant Payment Platform)
- **Type**: Real-time payment rail
- **Speed**: Instant (seconds)
- **Use Cases**: P2P transfers, instant payments, urgent transactions
- **Cut-off**: 24/7 (no cut-off)

### ⚡ IPI (Interbank Payment Interface)
- **Type**: Batch real-time payment
- **Speed**: Near real-time (minutes)
- **Use Cases**: Interbank transfers, domestic payments
- **Cut-off**: 23:59 daily

### 📦 FTS-NG (Funds Transfer System - Next Gen)
- **Type**: Batch processing rail
- **Speed**: Same-day or next-day
- **Use Cases**: Bulk payments, payroll, scheduled transfers
- **Cut-off**: 18:30 daily

### 🌐 SWIFT (Society for Worldwide Interbank Financial Telecommunication)
- **Type**: International payment network
- **Speed**: 1-5 business days
- **Use Cases**: Cross-border payments, international transfers
- **Cut-off**: Varies by correspondent bank

**Retry Flow**: Any rail failure \u2192 Retry Handler \u2192 Re-queue \u2192 Retry with exponential backoff

---

## 8. Key Data Flows

### 8.1 Main Payment Processing Flow (Payment Orchestration Domain)
```
Entry Channels (API/MQ/File) 
  ↓
┌─────────────────────────────────────────────────────────────────────┐
│                   PAYMENT ORCHESTRATION DOMAIN                     │
│                                                                     │
│  ① Payment Gateway Core [logs to Audit]                            │
│     └─ Create TransactionContext (persistent storage with cache)   │
│           │                                                         │
│  ② Validation Pipeline (4 stages)                                   │
│     ├─ Schema Validation [uses Cache]                               │
│     ├─ Business Rules Validation [applies Rules Repository]         │
│     ├─ Reference Data Validation [Holiday, Cutoff, Participant]     │
│     └─ Duplicate Detection [triggers Fraud Detection]                │
│           │                                                         │
│  ③ Limit Control Engine (3 steps)                                   │
│     ├─ Limit Lookup [fetches Limits, uses Cache]                    │
│     ├─ Limit Calculation [calculates with Pricing]                  │
│     └─ Limit Decision [triggers AML Screening]                       │
│           │                                                         │
│  ④ Queue Management (3 phases)                                      │
│     ├─ Priority Assignment [references Workflow Configs]             │
│     ├─ Queue Placement                                              │
│     └─ Scheduling                                                     │
│           │                                                         │
│  ⑤ Status Manager                                                   │
│     └─ Triggers Sanctions Check, sends to Monitoring                │
│     └─ Event Publisher [sends to Notification Service]              │
│           │                                                         │
│  ⑥ Embedded Routing Engine  ◀── NO NETWORK HOP                     │
│     ├─ Rule Engine: Evaluate rules using TransactionContext        │
│     ├─ Smart Router: Calculate optimal rail scores                  │
│     └─ Decision Maker: Select rail with fallback                    │
│           │                                                         │
│  ⑦ Rail Execution                                                   │
│     └─ Execute on selected Payment Rail                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
  ↓
Payment Rails (IPP/IPI/FTS-NG/SWIFT)
```

**Key Optimization**: Steps ①-⑦ execute within a single service boundary using shared TransactionContext, eliminating 3 network hops and enabling atomic database transactions.

### 8.2 Configuration Integration Flows (Reference - Dotted Lines in Diagram)
These are lookup/reference relationships, not sequential processing steps:

**Validation Stage 3 Integration:**
```
Reference Data Validation ←→ Holiday Calendar (check business days)
Reference Data Validation ←→ Cut-off Management (verify time windows)
Reference Data Validation ←→ Participant Directory (lookup bank details)
```

**Limit Control Integration:**
```
Limit Lookup ←→ Limit Definitions (fetch applicable limits)
Limit Lookup ←→ Cache Service (performance optimization)
Limit Calculation ←→ Pricing Engine (calculate fees)
```

**Queue & Status Integration:**
```
Priority Assignment ←→ Workflow Configs (reference SLA definitions)
Status Manager ←→ Currency Management (multi-currency tracking)
```

**Performance Optimization:**
```
Schema Validation ←→ Cache Service (schema caching)
Limit Lookup ←→ Cache Service (limit caching)
```

### 8.3 Risk & Compliance Trigger Flows
Risk checks are triggered at specific points in the main flow:

```
Validation Stage 4 (Duplicate Detection) → Fraud Detection
  ↓
Limit Decision (Step 3) → AML Screening
  ↓
Status Manager → Sanctions Check
```

### 8.4 Event & Notification Flow
```
Payment Gateway Core → Audit & Logging (all actions)
  ↓
Status Manager → Monitoring & Metrics (real-time metrics)
  ↓
Event Publisher → Notification Service → Multi-channel delivery (Email/SMS/Push/Webhook)
```

### 8.5 Error Handling & Retry Flow
```
Payment Rail Failure
  → Retry Handler (decide retry strategy)
    → Backoff Calculation (exponential: 1s, 2s, 4s, 8s...)
      → Queue Placement (Phase 2 - re-queue)
        → Retry Scheduling (Phase 3 - schedule next attempt)
          → [If max retries exceeded] → Dead Letter Queue
```

### 8.6 Platform Services Integration
Cross-cutting concerns integrated throughout:

```
Payment Gateway → Audit & Logging (all transactions)
Event Publisher → Notification Service (status updates)
Status Manager → Monitoring & Metrics (dashboards)
Validation Stage 1 → Cache Service (schema definitions)
Limit Lookup → Cache Service (limit configurations)
```

---

## 9. Integration Points

### 9.1 Inbound Integrations
Entry points for payment requests into the system:

| Integration Point | Protocol | Usage Pattern | Features |
|------------------|----------|---------------|----------|
| **API Gateway** | REST/GraphQL | Synchronous | Real-time response, immediate validation feedback |
| **Message Queue Gateway** | Kafka/RabbitMQ | Asynchronous | Event-driven, guaranteed delivery, high throughput |
| **File Gateway** | SFTP/Batch | Scheduled Batch | Bulk processing, file format validation, reconciliation |

### 9.2 Outbound Integrations
Payment execution and supporting services:

| Integration Point | Purpose | Protocol | Trigger Point |
|------------------|---------|----------|---------------|
| **IPP Rail** | Instant payments | ISO 20022 | Rail Executor (embedded) |
| **IPI Rail** | Interbank transfers | Proprietary | Rail Executor (embedded) |
| **FTS-NG Rail** | Batch processing | Batch file | Rail Executor (embedded) |
| **SWIFT Network** | Cross-border | MT/MX Messages | Rail Executor (embedded) |
| **Fraud Detection** | Risk scoring | REST API | Duplicate Detection |
| **AML Screening** | Compliance check | REST API | Limit Decision |
| **Sanctions Check** | Watch list | REST API | Status Manager |
| **Notification Service** | Alerts | Message Queue | Event Publisher |
| **Audit & Logging** | Compliance logging | Event Stream | Payment Gateway |
| **Monitoring** | Metrics & alerts | Prometheus/Grafana | Status Manager |

> **Note**: Payment rail execution is now handled by the embedded Rail Executor within the Payment Orchestration Domain, using adapter pattern for different rail protocols (IPPAdapter, IPIAdapter, FTSAdapter, SWIFTAdapter).

### 9.3 Configuration Management Service Integration
All configuration lookups are served by the unified Configuration Management Service (Port 8002):

| Module | API Endpoint | Used By | Cache TTL | Update Frequency |
|--------|-------------|---------|-----------|------------------|
| **Holiday Calendar** | `/api/v1/calendars` | Reference Data Validation | 24 hours | Daily |
| **Cut-off Management** | `/api/v1/cutoffs` | Reference Data Validation | 60 seconds | Real-time |
| **Business Day Calc** | `/api/v1/business-days` | Reference Data Validation | 24 hours | On-demand |
| **Participant Directory** | `/api/v1/participants` | Reference Data Validation | 1 hour | On-demand |
| **Banking Validation** | `/api/v1/validation` | Reference Data Validation | 24 hours | On-demand |
| **Currency/Country** | `/api/v1/currencies` | Status Manager | 24 hours | Daily |
| **Rules Engine** | `/api/v1/rules` | Business Rules Validation | 15 minutes | On-demand |
| **Workflows** | `/api/v1/workflows` | Priority Assignment | 1 hour | On-demand |
| **Charge Calculation** | `/api/v1/charges` | Limit Calculation | 1 hour | On-demand |
| **Limit Definitions** | `/api/v1/limits` | Limit Lookup | 15 minutes | Near real-time |
| **Vostro Accounts** | `/api/v1/vostro` | Settlement Processing | 30 seconds | Real-time |

**Service Details**: See [Configuration Management Service Design](ConfigurationDomain/Design_Configuration_Management_Service.md)

---

## 10. Design Patterns Used

### Architectural Patterns

1. **Pipeline Pattern** (Validation & Processing)
   - **Implementation**: 4-stage validation pipeline executed sequentially
   - **Benefit**: Clear separation of concerns, fail-fast approach
   - **Components**: Schema → Business Rules → Reference Data → Duplicate Detection

2. **Strategy Pattern** (Limit Calculation & Routing)
   - **Implementation**: Different limit calculation strategies based on customer segment
   - **Benefit**: Flexible algorithm selection without code changes
   - **Components**: User limits, Account limits, Channel limits with different calculation methods

3. **Priority Queue Pattern** (Queue Management)
   - **Implementation**: Multiple priority levels for transaction processing
   - **Benefit**: Ensures high-priority transactions are processed first
   - **Components**: Priority, Standard, Batch, and Dead Letter Queues

4. **State Machine Pattern** (Status & Lifecycle)
   - **Implementation**: Payment lifecycle with defined states and transitions
   - **Benefit**: Predictable state management, prevents invalid transitions
   - **States**: PENDING → VALIDATED → QUEUED → PROCESSING → COMPLETED/FAILED

5. **Retry Pattern with Exponential Backoff** (Error Handling)
   - **Implementation**: Automatic retry with increasing delay (1s, 2s, 4s, 8s...)
   - **Benefit**: Handles transient failures, prevents thundering herd
   - **Components**: Retry Handler, Backoff Calculator, Max Retry Limiter

6. **Event Sourcing** (Status Tracking)
   - **Implementation**: Status changes captured as immutable events
   - **Benefit**: Complete audit trail, ability to replay events
   - **Components**: Status Manager, Event Publisher, Event Store

7. **Cache-Aside Pattern** (Configuration Data)
   - **Implementation**: Check cache first, load from DB on miss, update cache
   - **Benefit**: Reduced database load, improved response times
   - **Components**: Redis/Hazelcast cache with TTL-based expiration

8. **Circuit Breaker Pattern** (External Service Calls)
   - **Implementation**: Automatic circuit opening when failure threshold reached
   - **Benefit**: Prevents cascading failures, fast failure response
   - **Components**: Fraud Detection, AML Screening, Sanctions Check calls

9. **Saga Pattern** (Distributed Transaction)
   - **Implementation**: Orchestrated saga for multi-step payment processing
   - **Benefit**: Eventual consistency, compensating transactions for rollback
   - **Components**: Payment Gateway orchestrates validation → limits → queue → routing

10. **Strangler Fig Pattern** (Legacy Integration)
    - **Implementation**: Gradual replacement of legacy payment systems
    - **Benefit**: Incremental migration, parallel operation during transition
    - **Components**: Routing layer can direct to new or legacy rails

---

## 11. Performance Considerations

### Payment Orchestration Domain (Primary Optimization)

The most significant performance optimization is the consolidation of Payment Orchestration and Routing & Decision into a **unified Payment Orchestration Domain** (ADR-005a):

| Optimization | Before (Separate) | After (Unified) | Impact |
|-------------|-------------------|-----------------|--------|
| **Inter-service calls** | Network hop to Routing Service | Inline method call | 20-50ms saved |
| **Serialization** | JSON encode/decode | Direct object passing | ~5ms saved |
| **Connection overhead** | Separate connection pools | N/A | Resource reduction |
| **Transaction scope** | Distributed across services | Single atomic transaction | Consistency |
| **End-to-End Latency (p99)** | 200ms | 150ms | 25% faster |
| **Throughput** | 5,000 TPS | 7,500 TPS | 50% increase |
| **Infrastructure Cost** | 2 services | 1 service | ~30% savings |

**Implementation Details**:
- **TransactionContext**: Shared persistent context with caching flows through all stages, enabling disaster recovery
- **Embedded Routing Engine**: Rule Engine, Smart Router, Decision Maker as internal components
- **Single Database Write**: Status update + routing decision saved atomically
- **Unified Metrics**: Single service tracing with internal spans for each stage

See [Payment Orchestration Service Design](PaymentOrchestrationDomain/Design_Payment_Orchestration_Service.md) for full architecture details.

### Caching Strategy
Aggressive caching for reference data to minimize database calls:

| Cache Type | Data | TTL | Impact |
|-----------|------|-----|--------|
| **Configuration Cache** | Holiday Calendar, Cut-offs | 1-24 hours | 90% DB load reduction |
| **Participant Cache** | Bank details, capabilities | 4 hours | 85% lookup time reduction |
| **Limit Cache** | Limit definitions | 15 minutes | 95% DB query elimination |
| **Schema Cache** | Validation schemas | 1 hour | 99% validation speedup |
| **Rule Cache** | Business rules | 30 minutes | 80% rule evaluation speedup |

### Asynchronous Processing
Queue-based architecture for non-blocking operations:
- **Message Queue Gateway**: Accepts high-volume async requests
- **Queue Management**: Decouples ingestion from processing
- **Event Publisher**: Asynchronous status notifications
- **Impact**: 10x throughput increase vs synchronous processing

### Parallel Validation
Independent validation stages can run concurrently (when enabled):
- Schema and Business Rules validation (parallel)
- Reference Data checks can be parallelized (Holiday + Cutoff + Participant)
- **Impact**: 40% reduction in validation time

### Batch Optimization
Grouped processing for batch payments:
- **File Gateway**: Single file with 1000s of payments processed together
- **Batch Queue**: Dedicated queue for bulk processing
- **Bulk DB Operations**: Batch inserts/updates instead of individual
- **Impact**: 100x efficiency for bulk payments

### Circuit Breaker
Prevents cascading failures in external service calls:
- **Timeout Configuration**: Fast failure (2-5 seconds)
- **Failure Threshold**: Open circuit after 5 consecutive failures
- **Half-Open Testing**: Gradual recovery with test requests
- **Impact**: System stability during downstream failures

### Database Optimization
- **Connection Pooling**: Reuse database connections
- **Read Replicas**: Route configuration reads to replicas
- **Partitioning**: Transaction tables partitioned by date
- **Indexing**: Strategic indexes on transaction ID, customer ID, status, timestamp

---

## 12. Scalability Features

### Horizontal Scaling
Each component designed for independent scaling:

| Component | Scaling Strategy | Trigger Metric | Target Instances |
|-----------|------------------|----------------|------------------|
| **Transaction Processor** | Load-balanced instances | TPS > 500 per pod | 3-50 instances |
| **Validation Workers** | Worker pool per stage | Queue depth > 100 | 5-20 workers |
| **Limit Control Engine** | Stateless compute (embedded) | Request rate > 1000 TPS | Scaled with processor |
| **Queue Workers** | Auto-scaling consumers | Queue lag > 10s | 5-50 workers |
| **Routing Engine** | Embedded (scales with processor) | N/A | Scaled with processor |

### Queue-Based Decoupling
Asynchronous processing prevents bottlenecks:
- **Ingestion Queue**: 100K+ messages/sec capacity (Kafka)
- **Processing Queues**: Multiple priority levels with independent consumers
- **Event Bus**: Pub/sub for status events (10K+ events/sec)
- **DLQ**: Isolated failed message handling

### Stateless Design
No session affinity required for load balancing:
- **Transaction Processor**: Fully stateless, TransactionContext persisted with cache per request
- **Validation Services**: No local state, can restart anytime with state recovery from persistent storage
- **Embedded Routing Engine**: Stateless decision making using TransactionContext
- **Benefit**: Simple load balancing, easy failover, fast deployments, unified scaling, disaster recovery

### Database Partitioning & Sharding
Distributed data storage for scale:

| Partition Strategy | Tables | Key | Benefit |
|-------------------|---------|-----|---------|
| **Date Partitioning** | Transactions, Status History | Transaction Date | Fast archive, efficient queries |
| **Customer Sharding** | Account Limits, User Profiles | Customer ID | Distributed load, tenant isolation |
| **Geographic Sharding** | Configuration, Participants | Region Code | Low latency, data residency compliance |

### Read Replicas
Configuration data served from read-only replicas:
- **Master**: Single write endpoint
- **Replicas**: 3-5 read replicas per region
- **Lag**: < 1 second replication lag
- **Usage**: Validation, Limit Lookup, Routing Rules

### Auto-Scaling Policies
Dynamic resource allocation based on demand:

```
Scale Up Triggers:
- CPU utilization > 70% for 5 minutes
- Queue depth > 1000 messages for 2 minutes
- Response time > 200ms (p99) for 3 minutes

Scale Down Triggers:
- CPU utilization < 30% for 15 minutes
- Queue depth < 100 messages for 10 minutes
- Min instances maintained: 2 per component
```

---

## 13. Monitoring & Observability

### Key Performance Indicators (KPIs)

#### Transaction Metrics
| Metric | Target | Alert Threshold | Dashboard |
|--------|--------|-----------------|-----------|
| **Request Throughput** | 1000-5000 TPS | < 500 TPS | Real-time graph |
| **Validation Pass Rate** | > 95% | < 90% | Stage-wise breakdown |
| **Limit Approval Rate** | > 98% | < 95% | By customer segment |
| **Queue Depth** | < 100 messages | > 1000 messages | Per queue type |
| **Queue Wait Time** | < 5 seconds | > 30 seconds | Histogram |
| **End-to-End Latency** | < 200ms (p99) | > 500ms | Percentile graph |
| **Error Rate** | < 0.5% | > 2% | By error type |

#### Validation Stage Metrics
```
Stage 1 (Schema):     Pass Rate, Avg Time, Top Failure Reasons
Stage 2 (Business):   Rule Hit Rate, Failed Rules, Processing Time
Stage 3 (Reference):  Cache Hit Rate, External Call Time, Failures
Stage 4 (Duplicate):  Detection Rate, False Positive Rate, Lookup Time
```

#### Limit Control Metrics
```
Lookup Performance:    Cache Hit %, DB Query Time, Concurrent Checks
Calculation Accuracy:  Utilization %, Breach Count, Hold Duration
Decision Distribution: Allow/Block/Warn Ratio, Approval Escalations
```

#### Status & Lifecycle Metrics
```
State Transitions:     State Duration, Invalid Transition Attempts
Event Publishing:      Event Lag, Failed Events, Retry Success Rate
Retry Performance:     Retry Attempt Distribution, Success After Retry %
```

### Observable Components

#### Distributed Tracing
- **Tool**: OpenTelemetry + Jaeger/Zipkin
- **Trace Span**: Single service tracing with internal spans for each stage
- **Coverage**: Entry → Gateway → Validation → Limits → Queue → Status-Routing (combined) → Execution
- **Benefit**: End-to-end visibility, bottleneck identification, simplified tracing (single service)

**Service Trace Structure**:
```
transaction-processor
├── span: gateway-receive (5ms)
├── span: validation-pipeline (25ms)
│   ├── span: schema-validation (5ms)
│   ├── span: business-rules (8ms)
│   ├── span: reference-data (7ms)
│   └── span: duplicate-check (5ms)
├── span: limit-control (15ms)
├── span: queue-management (5ms)
├── span: status-routing (10ms)  ◀── Combined span (no network boundary)
│   ├── span: rule-engine (3ms)
│   ├── span: smart-router (4ms)
│   └── span: decision-maker (3ms)
└── span: rail-execution (100ms)
```

#### Structured Logging
- **Format**: JSON with correlation ID
- **Levels**: DEBUG, INFO, WARN, ERROR, FATAL
- **Context**: Transaction ID, Customer ID, Channel, Amount, Status
- **Storage**: Elasticsearch (7 days hot, 90 days warm, 2 years cold)

#### Real-time Dashboards
```
Executive Dashboard:   Daily volume, Success rate, Revenue, Top errors
Operations Dashboard:  Queue depths, Processing times, System health
Technical Dashboard:   CPU/Memory/Disk, API latencies, Cache hit rates
Business Dashboard:    Payment rails usage, Customer segments, Peak hours
```

#### Alerting Strategy
| Severity | Examples | Response Time | Notification |
|----------|----------|---------------|--------------|
| **P1 Critical** | System down, 100% error rate | < 5 minutes | PagerDuty, Phone, SMS |
| **P2 High** | Error rate > 5%, Latency > 1s | < 15 minutes | PagerDuty, Email |
| **P3 Medium** | Queue depth high, Cache miss > 20% | < 1 hour | Email, Slack |
| **P4 Low** | Config change, Slow queries | < 4 hours | Email |

### Health Checks
```
Liveness Probe:  /health/live (Is service running?)
Readiness Probe: /health/ready (Can service accept traffic?)
Startup Probe:   /health/startup (Has service completed initialization?)

Health Check Includes:
- Application status (UP/DOWN)
- Database connectivity (Primary + Replicas)
- Cache connectivity (Redis/Hazelcast)
- Message Queue connectivity (Kafka)
- External service availability (Fraud, AML, Sanctions)
```

---

## 14. Security Controls

### Authentication & Authorization

#### Entry Point Security
| Channel | Authentication | Authorization |
|---------|---------------|---------------|
| **API Gateway** | OAuth 2.0, JWT | Role-Based Access Control (RBAC) |
| **Message Queue** | mTLS, SASL | Topic-level permissions |
| **File Gateway** | SSH Keys, IP Whitelist | Directory-level access |

#### Service-to-Service
- **Protocol**: Mutual TLS (mTLS)
- **Certificates**: Rotated every 90 days
- **Service Mesh**: Istio/Linkerd for zero-trust networking

### Data Protection

#### Encryption
| Data State | Method | Key Management |
|-----------|--------|----------------|
| **In-Transit** | TLS 1.3 | Automated cert rotation |
| **At-Rest** | AES-256 | AWS KMS / Azure Key Vault |
| **PII Fields** | Field-level encryption | Customer Master Keys (CMK) |

#### Data Masking
Sensitive data protection in logs and non-production environments:
```
Card Numbers:     Only last 4 digits (************1234)
Account Numbers:  Masked except last 4 (******4567)
Customer Names:   First name + last initial (John S.)
Amounts:         Masked in non-production logs
```

### Audit Trail
Complete immutable logging of all actions:
- **What**: Action performed (CREATE, UPDATE, APPROVE, REJECT)
- **Who**: User/Service ID, IP address
- **When**: Timestamp with millisecond precision
- **Why**: Business reason, approval reference
- **Where**: System component, geographic location
- **Before/After**: State changes captured

### Rate Limiting
Protection against abuse and DDoS:

| Level | Limit | Window | Action |
|-------|-------|--------|--------|
| **Per Customer** | 100 requests | 1 minute | HTTP 429 + Wait header |
| **Per IP** | 1000 requests | 1 minute | Temporary block (5 min) |
| **Per API Key** | 10000 requests | 1 hour | Throttle to 50% speed |
| **Global** | 100K requests | 1 minute | Circuit breaker |

### Vulnerability Management
- **SAST**: Static code analysis (SonarQube, Checkmarx)
- **DAST**: Dynamic testing (OWASP ZAP)
- **SCA**: Dependency scanning (Snyk, Dependabot)
- **Penetration Testing**: Annual third-party pen tests
- **Patch Management**: Critical patches within 48 hours

### Compliance & Standards
- **PCI-DSS**: Level 1 compliance for card data
- **PSD2**: Strong Customer Authentication (SCA)
- **GDPR**: Data privacy and right to erasure
- **SOC 2 Type II**: Annual audit
- **ISO 27001**: Information security management

---

## 15. Disaster Recovery & Business Continuity

### Backup Strategy
| Component | Backup Frequency | Retention | RTO | RPO |
|-----------|------------------|-----------|-----|-----|
| **Transaction DB** | Continuous (WAL) | 30 days | 15 min | 5 min |
| **Configuration DB** | Every 6 hours | 90 days | 30 min | 6 hours |
| **Audit Logs** | Real-time replication | 7 years | 1 hour | Near-zero |
| **Message Queues** | Replicated (3 copies) | N/A | Instant | Near-zero |

### High Availability
- **Multi-AZ Deployment**: Active-active across 3 availability zones
- **Load Balancing**: Automatic traffic distribution
- **Auto-Healing**: Failed instances replaced automatically
- **Database**: Multi-master or Primary-Replica with auto-failover
- **Uptime SLA**: 99.99% (< 53 minutes downtime/year)

### Disaster Recovery
- **DR Site**: Active standby in secondary region
- **Data Replication**: Asynchronous cross-region replication
- **Failover Strategy**: Automated DNS failover or manual cutover
- **Recovery Time Objective (RTO)**: < 1 hour
- **Recovery Point Objective (RPO)**: < 15 minutes
- **DR Testing**: Quarterly failover drills

---

## 16. Future Enhancements

### Phase 1: Machine Learning Integration (Q2 2026)
**Objective**: Intelligent automation and predictive capabilities

1. **ML-Based Fraud Scoring**
   - Real-time fraud probability scoring using neural networks
   - Behavioral biometrics and anomaly detection
   - Adaptive learning from false positives/negatives
   - Expected Impact: 40% reduction in fraud losses, 30% fewer false positives

2. **Dynamic Limit Optimization**
   - ML-driven personalized limits based on user behavior
   - Predictive utilization forecasting
   - Automatic limit adjustment recommendations
   - Expected Impact: 15% increase in approval rates, 25% reduction in breaches

3. **Intelligent Routing**
   - ML model for optimal rail selection
   - Cost-speed-reliability optimization
   - Learning from historical success rates
   - Expected Impact: 10% cost reduction, 20% faster processing

### Phase 2: Advanced Analytics (Q3 2026)
**Objective**: Data-driven insights and predictive operations

1. **Predictive Queue Management**
   - Forecast peak load times
   - Proactive resource scaling
   - Capacity planning automation
   - Expected Impact: 30% reduction in queue wait times

2. **Real-time Streaming Analytics**
   - Live business metrics dashboard
   - Anomaly detection in transaction patterns
   - Trend analysis and forecasting
   - Expected Impact: Instant visibility, faster incident response

3. **Customer Behavior Analytics**
   - Payment pattern analysis
   - Customer segmentation and personalization
   - Churn prediction and retention strategies
   - Expected Impact: 20% improvement in customer satisfaction

### Phase 3: Advanced Features (Q4 2026)
**Objective**: Next-generation capabilities

1. **Smart Retry with Reinforcement Learning**
   - RL-based retry strategy optimization
   - Context-aware backoff calculation
   - Success probability prediction
   - Expected Impact: 25% higher retry success rate

2. **A/B Testing Framework**
   - Routing strategy experimentation
   - Validation rule testing
   - Limit policy optimization
   - Expected Impact: Evidence-based decision making

3. **GraphQL Federation**
   - Unified API across payment domains
   - Flexible querying capabilities
   - Real-time subscriptions for status updates
   - Expected Impact: 50% reduction in API calls, better developer experience

### Phase 4: Ecosystem Expansion (2027)
**Objective**: Broader payment capabilities

1. **Cryptocurrency Support**
   - Bitcoin, Ethereum, stablecoin payments
   - Blockchain-based settlement
   - DeFi integration
   - Expected Impact: New revenue streams, younger demographic

2. **Buy Now Pay Later (BNPL)**
   - Installment payment options
   - Credit assessment integration
   - Partner lending network
   - Expected Impact: 30% increase in high-value transactions

3. **Open Banking Integration**
   - Account information services
   - Payment initiation services
   - PSD2/Open Banking compliance
   - Expected Impact: Direct account-to-account payments

4. **Cross-Border Expansion**
   - Multi-currency wallet
   - Foreign exchange optimization
   - Regional payment method support (Alipay, WeChat Pay, UPI)
   - Expected Impact: Global market reach

---

## 17. Architecture Decision Records (ADRs)

### ADR-001: Event-Driven Architecture for Status Updates
**Decision**: Use event sourcing for status changes instead of direct database updates  
**Rationale**: Complete audit trail, eventual consistency, decoupled consumers  
**Trade-offs**: Increased complexity, eventual consistency challenges  
**Status**: Accepted

### ADR-002: Multi-Stage Validation Pipeline
**Decision**: 4 sequential validation stages instead of single monolithic validation  
**Rationale**: Fail-fast, clear separation of concerns, easier debugging  
**Trade-offs**: Slight latency increase (40ms), more components to manage  
**Status**: Accepted

### ADR-003: Queue-Based Decoupling
**Decision**: Message queues between major components  
**Rationale**: Scalability, resilience, async processing  
**Trade-offs**: Complexity, message ordering challenges  
**Status**: Accepted

### ADR-004: Redis for Distributed Caching
**Decision**: Redis Cluster for configuration and limit caching  
**Rationale**: High performance, TTL support, distributed setup  
**Trade-offs**: Additional infrastructure, cache invalidation complexity  
**Status**: Accepted

### ADR-005: Separate Routing & Decision Domain
**Decision**: Dedicated domain for rail selection logic  
**Rationale**: Complex routing rules, independent evolution, reusable across channels  
**Trade-offs**: Additional service, network hop  
**Status**: ~~Accepted~~ **SUPERSEDED by ADR-005a**

### ADR-005a: Payment Orchestration Domain (Performance Optimization)
**Decision**: Consolidate Payment Orchestration Domain and Routing & Decision Domain into a unified Payment Orchestration Domain  
**Rationale**: 
- Eliminate 20-50ms network latency between orchestration and routing
- Optimize serialization with efficient binary encoding
- Enable shared TransactionContext (persistent storage with cache) across all stages for disaster recovery
- Atomic database transactions for status + routing decisions
- Unified scaling based on TPS (single service to scale)
- Simplified operational model (one service to deploy, monitor, troubleshoot)

**Performance Improvements**:
| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Latency (p99) | 200ms | 150ms | 25% faster |
| Throughput | 5,000 TPS | 7,500 TPS | 50% increase |
| Network Hops | 3 | 0 | 100% reduction |
| Infrastructure | 2 services | 1 service | 50% reduction |

**Trade-offs**: Larger service footprint, routing changes require redeployment (mitigated by configuration-driven routing rules)  
**Status**: Accepted  
**Documentation**: See [Payment Orchestration Service Design](PaymentOrchestrationDomain/Design_Payment_Orchestration_Service.md)

---

## 18. Integration with Other Bounded Contexts

### Upstream Contexts (Provide Data/Requests to Payment Gateway)
```
Customer Management → Customer profiles, authentication
Account Management → Account balances, account details
Channel Services → Web/Mobile/API entry points
File Processing → Batch payment files
```

### Downstream Contexts (Consume from Payment Gateway)
```
Settlement & Reconciliation → Payment execution details
Reporting & Analytics → Transaction data, metrics
Customer Notifications → Status updates, receipts
Financial Accounting → General ledger entries
```

### Peer Contexts (Bi-directional Integration)
```
Risk & Compliance ↔ Screening results, risk scores
Configuration Domain ↔ Reference data, rules
Platform Services ↔ Monitoring, logging, notifications
```

---

## 19. Technology Stack Recommendations

### Application Layer
| Component | Technology Options | Recommendation |
|-----------|-------------------|----------------|
| **API Gateway** | Kong, NGINX, AWS API Gateway | Kong (feature-rich, extensible) |
| **Payment Gateway Core** | .NET Core/ASP.NET, Java/Spring Boot, Go | .NET Core/ASP.NET (enterprise maturity, performance) |
| **Validation Services** | Microservices (.NET, Java, Go) | .NET Core (consistent with core) |
| **Message Queue** | Kafka, RabbitMQ, AWS SQS | Kafka (high throughput, event replay) |
| **Cache** | Redis, Hazelcast, Memcached | Redis Cluster (feature-rich, proven) |
| **Search** | Elasticsearch, OpenSearch | Elasticsearch (logging, analytics) |

### Data Layer
| Purpose | Technology Options | Recommendation |
|---------|-------------------|----------------|
| **Transactional DB** | SQL Server, PostgreSQL, Oracle | SQL Server (ACID, enterprise features, integration) |
| **Configuration DB** | SQL Server, PostgreSQL | SQL Server (consistency critical, unified platform) |
| **Event Store** | Event Store, Kafka | Kafka (already in stack) |
| **Audit Logs** | MongoDB, Cassandra, S3 | Elasticsearch (search capabilities) |

### Infrastructure & Observability
| Purpose | Technology Options | Recommendation |
|---------|-------------------|----------------|
| **Container Orchestration** | Kubernetes, ECS, Docker Swarm | Kubernetes (industry standard) |
| **Service Mesh** | Istio, Linkerd, Consul | Istio (feature-rich) |
| **Monitoring** | Prometheus, Datadog, New Relic | Prometheus + Grafana (open source) |
| **Tracing** | Jaeger, Zipkin, AWS X-Ray | Jaeger (OpenTelemetry native) |
| **Logging** | ELK Stack, Splunk, Datadog | ELK Stack (cost-effective) |
| **CI/CD** | Jenkins, GitLab CI, GitHub Actions | GitLab CI (integrated) |

---

## 20. Summary

The Payment Gateway Context represents a comprehensive, scalable, and resilient payment processing architecture designed to handle high-volume transaction processing with real-time validation, intelligent routing, and robust error handling.

### Payment Orchestration Domain (ADR-005a)

Following ADR-005a, the architecture now implements a **unified Payment Orchestration Domain** that consolidates Payment Orchestration and Routing & Decision into a single high-performance service:

| Optimization | Before | After | Impact |
|-------------|--------|-------|--------|
| **Latency (p99)** | 200ms | 150ms | 25% faster |
| **Throughput** | 5,000 TPS | 7,500 TPS | 50% increase |
| **Network Hops** | 3 calls | 0 calls | 100% eliminated |
| **Infrastructure** | 2 services | 1 service | ~30% cost reduction |
| **Transaction Scope** | Distributed | Atomic | Improved consistency |

The unified service uses a shared **TransactionContext** object that flows through all processing stages (validation, limits, queue, status, routing, execution) without serialization, enabling:
- Zero serialization overhead
- Single atomic database transaction for status + routing
- Unified scaling based on TPS
- Simplified operational model

### Key Architectural Highlights

✅ **Unified Architecture**: Consolidated orchestration + routing in single high-performance service  
✅ **TransactionContext**: Shared persistent state object with caching across all processing stages for disaster recovery  
✅ **Sequential Processing Pipeline**: 4-stage validation → 3-step limit control → 3-phase queue management → embedded routing → execution  
✅ **Event-Driven**: Status changes and notifications via event streaming for decoupled, scalable operations  
✅ **Configuration-Driven**: Business rules, limits, and workflows externalized for flexibility  
✅ **Multi-Channel Support**: API, Message Queue, and File-based entry points  
✅ **Intelligent Routing**: Embedded rule-based rail selection with multi-criteria scoring and fallback  
✅ **Comprehensive Risk Controls**: Fraud detection, AML screening, sanctions checking  
✅ **Observability**: Distributed tracing with internal spans, structured logging, real-time monitoring  
✅ **Resilience**: Retry with exponential backoff, circuit breakers, dead letter queues  
✅ **Security**: End-to-end encryption, RBAC, audit trails, compliance controls

### Performance Characteristics
- **Throughput**: 7,500 TPS baseline (scalable to 50K+ TPS with horizontal scaling)
- **Latency**: < 150ms (p99) end-to-end (25% improvement from unified architecture)
- **Availability**: 99.99% uptime SLA
- **Scalability**: Unified horizontal scaling across orchestration + routing
- **Data Consistency**: Atomic transactions within unified service, eventual consistency for external events

### Business Benefits
🎯 **Faster Time-to-Market**: Configuration-driven rules reduce deployment cycles  
💰 **Cost Optimization**: Intelligent routing minimizes transaction costs  
🛡️ **Risk Mitigation**: Multi-layered validation and screening  
📊 **Data-Driven Decisions**: Comprehensive metrics and analytics  
🚀 **Future-Ready**: Extensible architecture for ML, crypto, open banking

---

## Related Documentation

### Domain Design Documents
- [Configuration Management Service](ConfigurationDomain/Design_Configuration_Management_Service.md) - Unified configuration service (Time/Calendar, Reference Data, Rules/Workflows, Financial Config)
- [Payment Orchestration Service Design](PaymentOrchestrationDomain/Design_Payment_Orchestration_Service.md) - Unified service design consolidating Gateway Core, Validation Pipeline, Limit Control, Queue Management, Status & Lifecycle, Embedded Routing Engine, and TransactionContext modules

### Architecture Diagrams
- [Payment Gateway Context Diagram](PaymentGateway_Context.mmd) - Architecture context diagram
- [Main Application Architecture](../Application_Architecture.mmd) - Overall system context

### Business & Planning
- [PaymentHub Discovery Report](../PaymentHub_Discovery_Report.md) - Initial analysis
- [Business Requirements Document](../PaymentHub_BRD.md) - Functional requirements
- [Microservices Quick Reference](../Microservices_Quick_Reference.md) - Service catalog
- [Implementation Roadmap](../Implementation_Roadmap.md) - Delivery plan

---

**Document Version**: 2.5  
**Last Updated**: February 23, 2026  
**Status**: Updated to align with unified Payment Orchestration Service design  
**Author**: PaymentHub Architecture Team  
**Review Cycle**: Quarterly
