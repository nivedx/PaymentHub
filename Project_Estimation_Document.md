# Payment Hub - Project Estimation Document
## Configuration Domain & Payment Orchestration Domain

---

### Document Control

| Attribute | Value |
|-----------|-------|
| **Document Title** | Development Effort Estimation |
| **Version** | 1.0 |
| **Date** | February 23, 2026 |
| **Prepared By** | Senior Software Architect |
| **Status** | Draft for Client Review |

---

## Executive Summary

This document provides a comprehensive effort estimation for the development of two core domains of the Payment Hub system:

1. **Configuration Management Service** (Configuration Domain)
2. **Payment Orchestration Service** (Payment Orchestration Domain)

| Summary Metric | Value |
|----------------|-------|
| **Total Estimated Effort (with Copilot)** | **621 Person-Days** |
| **Contingency Buffer (15%)** | 93 Person-Days |
| **Grand Total with Buffer** | **714 Person-Days** |
| **Recommended Team Size** | 6-8 Engineers |
| **Estimated Duration** | 14-16 Weeks |
| **Productivity Gain (vs Traditional)** | ~35% |
| **Development Approach** | GitHub Copilot Agent-Assisted |

---

## 1. Estimation Methodology

### 1.1 Approach

This estimation employs a **hybrid methodology** combining:

| Method | Application | Weight |
|--------|-------------|--------|
| **Expert Judgment** | Base estimates from similar payment systems | 50% |
| **Analogous Estimation** | Comparison with prior payment gateway implementations | 30% |
| **Bottom-Up Decomposition** | WBS-based task breakdown per module | 20% |

### 1.2 AI-Assisted Development (GitHub Copilot Agent)

This estimation incorporates productivity gains from **GitHub Copilot Agent** assisted development:

| Task Category | Copilot Impact | Productivity Gain |
|---------------|----------------|-------------------|
| Boilerplate/CRUD Code | High | 40-50% |
| Unit Test Generation | High | 35-45% |
| API Development | Medium-High | 30-40% |
| Documentation | High | 35-45% |
| Standard Patterns | High | 40-50% |
| Database Schema/Queries | Medium | 25-35% |
| Complex Business Logic | Low-Medium | 15-20% |
| Integration Code | Low-Medium | 15-25% |
| DevOps/Infrastructure | Medium | 20-30% |
| Debugging/Troubleshooting | Low | 10-15% |

**Note**: Estimates below reflect Copilot-assisted productivity. Complex algorithmic work, architecture decisions, and integration debugging retain conservative estimates.

### 1.3 Estimation Assumptions

| Category | Assumption |
|----------|------------|
| **Team Composition** | Senior (30%) / Mid-Level (50%) / Junior (20%) engineers |
| **Working Hours** | 8 productive hours per person-day |
| **Technology Stack** | .NET Core/ASP.NET, SQL Server, Redis, Kafka, Kubernetes |
| **Infrastructure** | Cloud-native, infrastructure pre-provisioned |
| **Documentation** | Standard technical documentation included |
| **Code Quality** | Unit tests (>80% coverage), code reviews mandatory |
| **AI Tooling** | GitHub Copilot Agent enabled for all developers |
| **Copilot Workspace** | Agents configured with project context and coding standards |

### 1.4 Complexity Assessment Criteria

| Complexity | Criteria | Copilot Benefit |
|------------|----------|----------------|
| **Low** | Standard CRUD, simple validation, single integration | High (40-50%) |
| **Medium** | Business logic, multiple integrations, state management | Medium (25-35%) |
| **High** | Complex algorithms, real-time processing, multi-system orchestration | Low (15-20%) |

---

## 2. Configuration Domain - Module Breakdown

### 2.1 Service Overview

| Attribute | Value |
|-----------|-------|
| Service Name | Configuration Management Service |
| Port | 8002 |
| Database | Microsoft SQL Server |
| Cache | Redis |
| Message Broker | Kafka |

### 2.2 Module 1: Time & Calendar Management

| Task | Description | Base PD | Copilot Gain | Final PD | Complexity |
|------|-------------|--------:|-------------:|---------:|------------|
| **Backend Development** | | | | | |
| Domain Model Implementation | HolidayCalendar, Holiday, CutoffRule, CutoffSchedule entities | 4 | 45% | 2 | Medium |
| Holiday Calendar Service | CRUD operations, multi-region calendar support | 6 | 45% | 3 | Medium |
| Cut-off Time Configuration | Rail/currency/client-specific cut-off management | 5 | 35% | 3 | Medium |
| Business Day Calculator | T+N calculations, settlement date computation | 5 | 20% | 4 | High |
| Timezone Management | Global timezone handling, DST support | 3 | 35% | 2 | Medium |
| Cut-off Validation Engine | Real-time cut-off checking with warnings | 5 | 20% | 4 | High |
| **Database** | | | | | |
| Schema Design & Implementation | Tables: holiday_calendars, holidays, cutoff_rules, cutoff_schedules | 3 | 35% | 2 | Medium |
| Stored Procedures | Business day calculations, bulk operations | 2 | 30% | 1 | Low |
| Data Migration Scripts | Initial calendar and cut-off data seeding | 2 | 40% | 1 | Low |
| **APIs** | | | | | |
| REST API Development | 7 endpoints for calendar/cut-off operations | 4 | 40% | 2 | Medium |
| API Documentation | OpenAPI/Swagger documentation | 1 | 50% | 0.5 | Low |
| **Integration** | | | | | |
| Redis Cache Integration | Calendar and cut-off caching with TTL | 3 | 25% | 2 | Medium |
| Kafka Event Publishing | Holiday/cut-off change events | 2 | 30% | 1.5 | Medium |
| **Testing** | | | | | |
| Unit Tests | >80% coverage for business logic | 4 | 40% | 2.5 | Medium |
| Integration Tests | API and database integration tests | 3 | 25% | 2 | Medium |
| **Documentation** | | | | | |
| Technical Documentation | Module design and API documentation | 2 | 45% | 1 | Low |
| **Subtotal** | | **54** | **~37%** | **33.5** | |

### 2.3 Module 2: Reference Data Management

| Task | Description | Base PD | Copilot Gain | Final PD | Complexity |
|------|-------------|--------:|-------------:|---------:|------------|
| **Backend Development** | | | | | |
| Domain Model Implementation | Participant, Classification, SettlementAccount, RailCapability | 5 | 45% | 3 | Medium |
| Participant Directory Service | Bank registry, BIC/SWIFT codes, routing numbers | 8 | 20% | 6 | High |
| Participant Classification | Tier levels, risk ratings, service levels | 4 | 40% | 2.5 | Medium |
| Participant Status Monitor | Operational status, health tracking, availability | 5 | 30% | 3.5 | Medium |
| Banking Validation Service | BIC validation, IBAN validation (MOD97), account format | 8 | 25% | 6 | High |
| Settlement Account Management | Nostro/Vostro accounts, GL mappings | 5 | 35% | 3 | Medium |
| Currency/Country Management | ISO codes, sanctioned country flagging | 3 | 50% | 1.5 | Low |
| **Database** | | | | | |
| Schema Design & Implementation | Tables: participants, classifications, settlement_accounts, currencies, countries, bic_directory, iban_rules | 4 | 35% | 2.5 | Medium |
| Indexes & Optimization | Performance indexes for lookups | 2 | 30% | 1.5 | Medium |
| Data Migration | BIC directory import, currency/country seeding | 3 | 40% | 2 | Medium |
| **APIs** | | | | | |
| REST API Development | 12 endpoints for participant/validation operations | 5 | 40% | 3 | Medium |
| API Documentation | OpenAPI/Swagger documentation | 1 | 50% | 0.5 | Low |
| **Integration** | | | | | |
| Redis Cache Integration | Participant and reference data caching | 3 | 25% | 2 | Medium |
| SWIFT Network Integration | BIC directory sync interface | 4 | 15% | 3.5 | High |
| Core Banking Integration | Account validation interface | 4 | 15% | 3.5 | High |
| Kafka Event Publishing | Participant status change events | 2 | 30% | 1.5 | Medium |
| **Testing** | | | | | |
| Unit Tests | >80% coverage including IBAN/BIC validation | 5 | 40% | 3 | Medium |
| Integration Tests | API, database, external system mocks | 4 | 25% | 3 | Medium |
| **Documentation** | | | | | |
| Technical Documentation | Module design and integration guides | 2 | 45% | 1 | Low |
| **Subtotal** | | **77** | **~35%** | **52** | |

### 2.4 Module 3: Business Rules & Workflows

| Task | Description | Base PD | Copilot Gain | Final PD | Complexity |
|------|-------------|--------:|-------------:|---------:|------------|
| **Backend Development** | | | | | |
| Domain Model Implementation | RuleDefinition, RuleCondition, RuleAction, WorkflowDefinition, WorkflowState | 6 | 40% | 4 | High |
| Rules Repository | Rule storage with versioning and effective dating | 6 | 40% | 3.5 | Medium |
| Rule Engine | Condition parsing, rule evaluation, priority handling | 12 | 15% | 10 | High |
| Workflow Engine | State machine, transition rules, guard conditions | 14 | 15% | 12 | High |
| Approval Management | Maker-checker pattern, multi-level approvals, delegation | 8 | 20% | 6 | High |
| Escalation Management | Time-based escalation, auto-escalation, notifications | 6 | 20% | 5 | High |
| SLA Management | SLA definitions, breach detection, performance tracking | 5 | 30% | 3.5 | Medium |
| Notification Templates | Email/SMS/Push/Webhook template engine | 5 | 40% | 3 | Medium |
| Decision Audit | Rule execution logging, decision analytics | 4 | 35% | 2.5 | Medium |
| **Database** | | | | | |
| Schema Design & Implementation | Tables: rule_definitions, rule_conditions, rule_actions, workflow_definitions, workflow_states, workflow_instances, escalation_rules, sla_definitions, notification_templates | 5 | 30% | 3.5 | High |
| Stored Procedures | Rule evaluation optimization, workflow state queries | 3 | 25% | 2 | Medium |
| **APIs** | | | | | |
| REST API Development | 15 endpoints for rules/workflows/escalations/SLAs | 6 | 40% | 3.5 | Medium |
| API Documentation | OpenAPI/Swagger documentation | 2 | 50% | 1 | Low |
| **Integration** | | | | | |
| Redis Cache Integration | Rule and workflow caching | 3 | 25% | 2 | Medium |
| Kafka Event Publishing | Workflow events, escalation triggers, SLA breaches | 4 | 30% | 3 | Medium |
| Notification Service Integration | Multi-channel notification dispatch | 4 | 20% | 3 | High |
| **Testing** | | | | | |
| Unit Tests | >80% coverage for rule engine and workflow logic | 8 | 35% | 5 | High |
| Integration Tests | Workflow state transitions, approval flows | 6 | 20% | 5 | High |
| **Documentation** | | | | | |
| Technical Documentation | Rule DSL guide, workflow design patterns | 3 | 40% | 2 | Medium |
| **Subtotal** | | **110** | **~29%** | **79.5** | |

### 2.5 Module 4: Financial Configuration

| Task | Description | Base PD | Copilot Gain | Final PD | Complexity |
|------|-------------|--------:|-------------:|---------:|------------|
| **Backend Development** | | | | | |
| Domain Model Implementation | ChargeDefinition, ChargeSlab, VostroAccount, LimitDefinition, LimitUtilization, GLAccount | 6 | 40% | 4 | High |
| Charge & Pricing Engine | Flat/percentage/tiered pricing, customer-specific rates | 10 | 15% | 8.5 | High |
| Fee Calculation Service | Multi-slab calculation, promotional rates | 6 | 20% | 5 | High |
| Vostro Account Management | Account registry, multi-currency support | 6 | 35% | 4 | Medium |
| Real-time Balance Tracking | Balance updates, held amount management | 8 | 15% | 7 | High |
| Limit Definition Management | Multi-dimensional limits (user/account/channel/product) | 6 | 35% | 4 | Medium |
| Real-time Limit Checking | Concurrent checks, utilization tracking, hold management | 10 | 15% | 8.5 | High |
| Limit Breach Handling | Breach actions, override workflows, escalation | 6 | 20% | 5 | High |
| GL Integration | GL code mapping, entry generation, reconciliation | 6 | 30% | 4 | Medium |
| **Database** | | | | | |
| Schema Design & Implementation | Tables: charge_definitions, charge_slabs, vostro_accounts, account_balances, gl_accounts, gl_mappings, limit_definitions, limit_utilization, limit_breaches | 5 | 30% | 3.5 | High |
| Stored Procedures | Limit utilization updates, balance calculations | 4 | 20% | 3 | High |
| **APIs** | | | | | |
| REST API Development | 14 endpoints for charges/limits/vostro/GL operations | 6 | 40% | 3.5 | Medium |
| API Documentation | OpenAPI/Swagger documentation | 2 | 50% | 1 | Low |
| **Integration** | | | | | |
| Redis Cache Integration | Limit definitions, balance caching (30s TTL) | 4 | 20% | 3 | High |
| Kafka Event Publishing | Limit breaches, balance alerts, charge events | 3 | 30% | 2 | Medium |
| GL System Integration | External GL posting interface | 5 | 15% | 4 | High |
| **Testing** | | | | | |
| Unit Tests | >80% coverage for pricing and limit calculations | 7 | 35% | 4.5 | High |
| Integration Tests | End-to-end charge calculation, limit enforcement | 5 | 20% | 4 | High |
| Concurrent Limit Tests | Race condition testing for limit updates | 3 | 10% | 2.5 | High |
| **Documentation** | | | | | |
| Technical Documentation | Pricing model documentation, limit hierarchy guide | 3 | 40% | 2 | Medium |
| **Subtotal** | | **111** | **~31%** | **83** | |

### 2.6 Configuration Domain - Cross-Cutting Concerns

| Task | Description | Base PD | Copilot Gain | Final PD | Complexity |
|------|-------------|--------:|-------------:|---------:|------------|
| **Service Infrastructure** | | | | | |
| Service Scaffolding | .NET Core project setup, NuGet dependency management | 3 | 50% | 1.5 | Low |
| API Gateway Layer | REST/GraphQL/gRPC/WebSocket configuration | 5 | 35% | 3 | Medium |
| Security Implementation | OAuth 2.0, RBAC, API key validation, TLS | 8 | 20% | 6 | High |
| Audit Logging | Comprehensive audit trail for all changes | 4 | 35% | 2.5 | Medium |
| **DevOps** | | | | | |
| Docker Containerization | Dockerfile, multi-stage builds | 2 | 40% | 1 | Low |
| Kubernetes Deployment | Helm charts, ConfigMaps, Secrets | 4 | 30% | 3 | Medium |
| CI/CD Pipeline | Build, test, deploy automation | 4 | 30% | 3 | Medium |
| Monitoring Setup | Prometheus metrics, Grafana dashboards | 4 | 30% | 3 | Medium |
| Alerting Configuration | Alert rules, notification channels | 3 | 35% | 2 | Medium |
| **Performance** | | | | | |
| Performance Optimization | Query optimization, caching strategies | 5 | 15% | 4 | High |
| Load Testing | JMeter/Gatling test scripts and execution | 4 | 25% | 3 | Medium |
| **Subtotal** | | **46** | **~33%** | **32** | |

### 2.7 Configuration Domain Summary

| Module | Base PD | With Copilot | Savings | Complexity |
|--------|--------:|-------------:|--------:|------------|
| Time & Calendar Management | 54 | 33.5 | 38% | Medium |
| Reference Data Management | 77 | 52 | 32% | Medium-High |
| Business Rules & Workflows | 110 | 79.5 | 28% | High |
| Financial Configuration | 111 | 83 | 25% | High |
| Cross-Cutting Concerns | 46 | 32 | 30% | Medium |
| **Total Configuration Domain** | **398** | **280** | **30%** | |

---

## 3. Payment Orchestration Domain - Module Breakdown

### 3.1 Service Overview

| Attribute | Value |
|-----------|-------|
| Service Name | Payment Orchestration Service |
| Port | 8001 |
| Database | SQL Server |
| Cache | Redis |
| Message Broker | Kafka |
| Target Performance | 7,500+ TPS, <150ms p99 latency |

### 3.2 Module 1: Payment Gateway Core

| Task | Description | Base PD | Copilot Gain | Final PD | Complexity |
|------|-------------|--------:|-------------:|---------:|------------|
| **Backend Development** | | | | | |
| Channel Receivers | REST Controller, GraphQL Resolver, MQ Listener, File Processor | 10 | 35% | 6.5 | High |
| Protocol Translators | JSON, XML/ISO20022, CSV, CAMT/PAIN translation | 10 | 30% | 7 | High |
| Request Validators | Schema validation, format validation, rate limiting | 5 | 40% | 3 | Medium |
| Transaction Initializer | UUID v7 generation, timestamp management, correlation IDs | 4 | 45% | 2 | Medium |
| Request Enrichment | Customer lookup, channel metadata, compliance flags | 6 | 35% | 4 | Medium |
| Idempotency Manager | Key generation, duplicate check, response caching | 5 | 25% | 4 | High |
| Orchestration Engine | Flow coordination, stage dispatch, error handling, circuit breaker | 10 | 15% | 8.5 | High |
| Batch Processing Engine | File parsing, chunk processing, progress tracking, error aggregation | 10 | 25% | 7.5 | High |
| Response Builder | Sync/async/batch response formatting | 4 | 45% | 2 | Medium |
| **APIs** | | | | | |
| REST API Development | Payment submission, status, batch endpoints | 5 | 40% | 3 | Medium |
| GraphQL Schema & Resolvers | Query, Mutation, Subscription types | 5 | 40% | 3 | Medium |
| API Documentation | OpenAPI/Swagger + GraphQL documentation | 2 | 50% | 1 | Low |
| **Testing** | | | | | |
| Unit Tests | >80% coverage for all components | 7 | 40% | 4 | Medium |
| Integration Tests | Multi-channel reception tests | 5 | 25% | 4 | Medium |
| **Subtotal** | | **88** | **~33%** | **59.5** | |

### 3.3 Module 2: TransactionContext Management

| Task | Description | Base PD | Copilot Gain | Final PD | Complexity |
|------|-------------|--------:|-------------:|---------:|------------|
| **Backend Development** | | | | | |
| Context Factory | Context creation, ID generation, initial state, templates | 4 | 45% | 2 | Medium |
| Context Registry | Active context tracking, lookup by TxnId, memory management | 5 | 20% | 4 | High |
| Context Lifecycle Manager | State transitions, TTL management, cleanup, recovery | 6 | 15% | 5 | High |
| State Holder (Core Data) | Transaction ID, payment details, account info, timestamps | 4 | 45% | 2 | Medium |
| Enrichment Container | Customer profile, channel metadata, risk indicators | 4 | 45% | 2 | Medium |
| Stage Result Aggregator | Validation, limit, queue, routing, execution results | 5 | 40% | 3 | Medium |
| Audit Buffer | Entry buffering, batch flush, compliance records | 5 | 25% | 4 | High |
| Snapshot Manager | Point-in-time snapshots, delta tracking, restoration | 6 | 15% | 5 | High |
| Concurrency Controller | Read/write locks, optimistic locking, deadlock prevention | 6 | 10% | 5.5 | High |
| Write-Through Cache | Redis persistence with consistency guarantees | 6 | 20% | 5 | High |
| **Database** | | | | | |
| Schema Design | Transactions, status_history, idempotency_records tables | 4 | 35% | 2.5 | Medium |
| Indexes & Optimization | Performance indexes for high-throughput queries | 3 | 25% | 2 | Medium |
| **Testing** | | | | | |
| Unit Tests | Context lifecycle, concurrency tests | 5 | 35% | 3 | High |
| Integration Tests | Cache consistency, persistence tests | 4 | 20% | 3 | High |
| **Subtotal** | | **67** | **~29%** | **48** | |

### 3.4 Module 3: Validation Pipeline

| Task | Description | Base PD | Copilot Gain | Final PD | Complexity |
|------|-------------|--------:|-------------:|---------:|------------|
| **Backend Development** | | | | | |
| Pipeline Orchestrator | Stage sequencing, fail-fast control, result aggregation | 6 | 20% | 5 | High |
| Stage 1: Schema Validation | JSON/XML schema validation, required fields, data types | 5 | 40% | 3 | Medium |
| Stage 2: Business Rules Validation | Payment type rules, channel rules, amount thresholds, time windows | 8 | 20% | 6 | High |
| Stage 3: Reference Data Validation | Holiday calendar, cut-off times, participant check, BIC/IBAN | 8 | 20% | 6 | High |
| Stage 4: Duplicate Detection | Transaction ID uniqueness, idempotency key, content-based matching | 6 | 25% | 4.5 | High |
| Validation Result Aggregator | Error collection, warning flags, validation metadata | 3 | 45% | 1.5 | Medium |
| **Integration** | | | | | |
| Configuration Domain Integration | Rules repository, reference data, calendars | 5 | 15% | 4 | High |
| Risk & Compliance Integration | Fraud Detection service integration | 4 | 15% | 3.5 | High |
| Cache Integration | Schema cache, validation cache | 3 | 30% | 2 | Medium |
| **Testing** | | | | | |
| Unit Tests | Per-stage validation logic tests | 6 | 40% | 3.5 | Medium |
| Integration Tests | Full pipeline integration tests | 4 | 20% | 3 | High |
| **Subtotal** | | **58** | **~28%** | **42** | |

### 3.5 Module 4: Limit Control Engine

| Task | Description | Base PD | Copilot Gain | Final PD | Complexity |
|------|-------------|--------:|-------------:|---------:|------------|
| **Backend Development** | | | | | |
| Engine Orchestrator | Step sequencing, decision coordination, hold management | 5 | 20% | 4 | High |
| Step 1: Limit Lookup | User/account/channel/product/corporate hierarchy limits | 6 | 20% | 5 | High |
| Step 2: Limit Calculation | Usage aggregation, available balance, utilization %, time-based reset | 8 | 15% | 7 | High |
| Step 3: Limit Decision | Allow/block/warn decisions, temporary holds, breach alerts | 6 | 20% | 5 | High |
| Multi-Level Limit Hierarchy | User → Account → Customer → Channel → Product limits | 5 | 20% | 4 | High |
| Usage Counter Management | Real-time counter updates with atomicity | 6 | 10% | 5.5 | High |
| Hold Management | Reserve limit during processing, release on completion | 4 | 20% | 3 | High |
| **Integration** | | | | | |
| Configuration Domain Integration | Limit definitions, pricing engine | 4 | 15% | 3.5 | High |
| AML Screening Integration | Trigger AML checks for limit breaches | 3 | 20% | 2.5 | Medium |
| Approval Workflow Integration | Route to approval for threshold breaches | 3 | 25% | 2 | Medium |
| Cache Integration | Limit cache, usage counters | 3 | 30% | 2 | Medium |
| **Testing** | | | | | |
| Unit Tests | Limit calculation, decision logic | 5 | 35% | 3 | High |
| Integration Tests | End-to-end limit enforcement | 4 | 20% | 3 | High |
| Concurrent Tests | Race condition handling | 3 | 10% | 2.5 | High |
| **Subtotal** | | **65** | **~22%** | **52** | |

### 3.6 Module 5: Queue Management

| Task | Description | Base PD | Copilot Gain | Final PD | Complexity |
|------|-------------|--------:|-------------:|---------:|------------|
| **Backend Development** | | | | | |
| Priority Calculator | Urgency scorer, segment weigher, amount scorer, SLA calculator | 6 | 20% | 5 | High |
| Queue Router | Queue selector, priority/standard/batch/DLQ placement | 5 | 35% | 3 | Medium |
| Scheduling Engine | Immediate handler, deferred scheduler, future date manager, retry scheduler | 8 | 20% | 6 | High |
| Queue Consumer | Dequeue manager, batch poller, priority poller, consumer group | 6 | 25% | 4.5 | High |
| Backpressure Controller | Rate limiter, load shedder, circuit breaker, throttle manager | 6 | 15% | 5 | High |
| SLA Monitor | SLA tracker, breach predictor, escalation engine, alert generator | 5 | 30% | 3.5 | Medium |
| Dead Letter Queue Handler | Failed transaction handling, manual intervention workflow | 4 | 35% | 2.5 | Medium |
| **Integration** | | | | | |
| Kafka Queue Integration | Queue topics setup, partitioning strategy | 4 | 25% | 3 | Medium |
| Configuration Domain Integration | Workflow configs, SLA definitions | 3 | 25% | 2 | Medium |
| **Testing** | | | | | |
| Unit Tests | Priority calculation, scheduling logic | 5 | 40% | 3 | Medium |
| Integration Tests | Queue operations, backpressure tests | 4 | 20% | 3 | High |
| Load Tests | High-volume queue throughput | 3 | 10% | 2.5 | High |
| **Subtotal** | | **59** | **~29%** | **43** | |

### 3.7 Module 6: Status & Lifecycle Management

| Task | Description | Base PD | Copilot Gain | Final PD | Complexity |
|------|-------------|--------:|-------------:|---------:|------------|
| **Backend Development** | | | | | |
| Status Manager | Status CRUD, current state, state validation, inline routing | 5 | 25% | 4 | High |
| State Machine | State definitions, transition rules, guard conditions, action triggers | 8 | 15% | 7 | High |
| Event Publisher | Event generation, topic routing, webhook dispatch, delivery guarantee | 6 | 25% | 4.5 | High |
| Retry Handler | Retry policy, backoff strategy, max attempts, circuit breaker, DLQ routing | 6 | 20% | 5 | High |
| SLA Monitor | SLA definitions, threshold check, breach alerting, escalation | 5 | 30% | 3.5 | Medium |
| Notification Engine | Alert triggers, channel routing, template engine, rate limiting | 5 | 35% | 3 | Medium |
| **Integration** | | | | | |
| Kafka Event Publishing | Status change events, audit events | 4 | 30% | 3 | Medium |
| Notification Service Integration | Multi-channel notifications | 3 | 25% | 2 | Medium |
| Monitoring Integration | Metrics export, dashboard integration | 3 | 35% | 2 | Medium |
| **Testing** | | | | | |
| Unit Tests | State machine transitions, event publishing | 5 | 40% | 3 | High |
| Integration Tests | End-to-end status flow tests | 4 | 25% | 3 | Medium |
| **Subtotal** | | **54** | **~26%** | **40** | |

### 3.8 Module 7: Embedded Routing Engine

| Task | Description | Base PD | Copilot Gain | Final PD | Complexity |
|------|-------------|--------:|-------------:|---------:|------------|
| **Backend Development** | | | | | |
| Rule Engine | Rule loader, rule evaluator, constraint checker, score aggregator | 8 | 15% | 7 | High |
| Smart Router | Score calculator, cost analyzer, speed evaluator, health integrator | 10 | 10% | 9 | High |
| Decision Maker | Primary selector, fallback handler, cost-benefit calculation, decision auditor | 6 | 15% | 5 | High |
| Rail Executor | Adapter factory, execution orchestration | 5 | 30% | 3.5 | Medium |
| IPP Rail Adapter | Instant Payment Platform integration | 8 | 20% | 6 | High |
| IPI Rail Adapter | Interbank Payment Interface integration | 8 | 20% | 6 | High |
| **FTS-NG Integration — Financial Messages (Inward & Outward, file-based SFTP, 48hr TAT)** | | | | | |
| FTS MT103 Adapter | Single customer-to-customer payment transfers (inward & outward) | 8 | 20% | 6 | High |
| FTS MT102 Adapter | Bulk customer transfers — salary/bonus payouts, 1-to-many (inward & outward) | 8 | 20% | 6 | High |
| FTS MT202 Adapter | Single institution-to-institution transfer (inward & outward) | 8 | 20% | 6 | High |
| FTS MT203 Adapter | Bulk institution-to-many transfers (inward & outward) | 8 | 20% | 6 | High |
| FTS MT2C2 Cover Adapter | Cross-border cover transfer messages (inward & outward) | 8 | 20% | 6 | High |
| **FTS-NG Integration — Non-Financial Messages (Inward & Outward, file-based SFTP, 48hr TAT)** | | | | | |
| FTS CB182 Refund Adapter | Refund messages to/from Central Bank (inward & outward) | 5 | 25% | 4 | Medium |
| FTS Message Query/Answer | Payment enquiry messages and corresponding answer responses (inward & outward) | 6 | 20% | 5 | High |
| FTS Free Format Message | Unstructured free-text inter-institution messages (inward & outward) | 4 | 30% | 3 | Medium |
| FTS Charge Claim | Transaction charge claim messages (inward & outward) | 4 | 25% | 3 | Medium |
| FTS CB002/CB003 Adapter | Exchange house gateway — CB002 received from exchange house, CB003 sent to CB (inward & outward) | 6 | 20% | 5 | High |
| FTS Statement Request | Account statement request messages (inward & outward) | 3 | 30% | 2 | Medium |
| FTS CAD Daily Report | Customer Activity Details — daily report to CB of all customer changes; sent even when empty | 5 | 20% | 4 | High |
| FTS CB101/CB103 Adapter | CB101 sent from bank to CB; CB103 received as acceptance confirmation | 5 | 20% | 4 | High |
| FTS Return 202 | Return of institution transfer messages (inward & outward) | 4 | 20% | 3 | High |
| **FTS-NG Shared Infrastructure** | | | | | |
| FTS SFTP Integration Layer | File-based SFTP upload/download to/from CB server, retry handling, 48hr TAT tracking, async response correlation | 8 | 15% | 7 | High |
| FTS Message Parser/Generator | File format parsing and generation for all FTS message types (inward & outward) | 6 | 15% | 5 | High |
| WPS Rail Adapter | Wages Protection System integration (SIF/SFTP) | 8 | 20% | 6 | High |
| Health Checker | Health monitor, availability check, performance stats, degradation alerts | 5 | 35% | 3 | Medium |
| Fallback Manager | Fallback strategy, retry coordinator, circuit breaker, recovery | 6 | 15% | 5 | High |
| Load Balancer | Round robin, weighted distribution, capacity management | 4 | 30% | 3 | Medium |
| Cost Optimizer | Fee calculator, FX cost analyzer, optimal path calculation | 5 | 15% | 4 | High |
| Routing Cache | Rule cache, score cache, health cache, TTL management | 4 | 30% | 3 | Medium |
| **Integration** | | | | | |
| Configuration Domain Integration | Participant directory, routing rules, charges | 4 | 15% | 3.5 | High |
| Risk & Compliance Integration | Sanctions check integration | 3 | 20% | 2.5 | Medium |
| **Testing** | | | | | |
| Unit Tests | Routing algorithm, scoring logic, adapter logic | 8 | 35% | 5 | High |
| Integration Tests | End-to-end routing, rail execution | 6 | 15% | 5 | High |
| Rail Simulation Tests | Mock rail responses, failure scenarios | 5 | 20% | 4 | High |
| **Subtotal** | | **199** | **~22%** | **155.5** | |

> **FTS Technical Architecture Note:** All FTS messages are file-based, uploaded to and downloaded from the Central Bank SFTP server. Responses are also file-based via the same SFTP mechanism. A 48-hour Turnaround Time (TAT) applies for responses. Both outward (sending) and inward (receiving) flows are required for all 14 FTS message types.

### 3.9 Payment Orchestration Domain - Cross-Cutting Concerns

| Task | Description | Base PD | Copilot Gain | Final PD | Complexity |
|------|-------------|--------:|-------------:|---------:|------------|
| **Service Infrastructure** | | | | | |
| Service Scaffolding | .NET Core project setup, module organization | 3 | 50% | 1.5 | Low |
| Security Implementation | OAuth 2.0, mTLS, field-level encryption, PII masking | 10 | 15% | 8.5 | High |
| Audit & Compliance | Immutable audit log, tamper detection | 5 | 20% | 4 | High |
| **DevOps** | | | | | |
| Docker Containerization | Multi-stage builds, optimization | 2 | 40% | 1 | Low |
| Kubernetes Deployment | Helm charts, HPA, PDB, resource quotas | 5 | 25% | 4 | High |
| CI/CD Pipeline | Build, test, deploy, rollback automation | 5 | 30% | 3.5 | Medium |
| Monitoring Setup | Prometheus metrics, Grafana dashboards (4 dashboards) | 5 | 30% | 3.5 | Medium |
| Alerting Configuration | Alert rules, PagerDuty/Slack integration | 3 | 35% | 2 | Medium |
| Log Aggregation | ELK/Loki setup, log correlation | 3 | 30% | 2 | Medium |
| **Performance** | | | | | |
| Performance Optimization | Query optimization, caching, connection pooling | 8 | 10% | 7 | High |
| Load Testing | High-volume tests (7,500+ TPS target) | 5 | 15% | 4 | High |
| Chaos Engineering | Failure injection, recovery validation | 4 | 10% | 3.5 | High |
| **Subtotal** | | **58** | **~24%** | **44.5** | |

### 3.10 Payment Orchestration Domain Summary

| Module | Base PD | With Copilot | Savings | Complexity |
|--------|--------:|-------------:|--------:|------------|
| Payment Gateway Core | 88 | 59.5 | 32% | High |
| TransactionContext Management | 67 | 48 | 28% | High |
| Validation Pipeline | 58 | 42 | 28% | High |
| Limit Control Engine | 65 | 52 | 20% | High |
| Queue Management | 59 | 43 | 27% | High |
| Status & Lifecycle Management | 54 | 40 | 26% | High |
| Embedded Routing Engine | 199 | 155.5 | 22% | High |
| Cross-Cutting Concerns | 58 | 44.5 | 23% | Medium-High |
| **Total Payment Orchestration Domain** | **648** | **484.5** | **25%** | |

---

## 4. Consolidated Estimation Summary

### 4.1 Domain Summary (With GitHub Copilot Optimization)

| Domain | Base PD | With Copilot | Savings | % of Total |
|--------|--------:|-------------:|--------:|------------|
| Configuration Domain | 398 | 280 | 30% | 40% |
| Payment Orchestration Domain | 648 | 484.5 | 25% | 60% |
| **Subtotal** | **1,046** | **764.5** | **27%** | 100% |

### 4.2 Effort Distribution by Work Type (Copilot-Optimized)

| Work Type | Configuration Domain | Payment Orchestration | Total PD | Copilot Gain | % |
|-----------|---------------------:|----------------------:|--------:|-------------:|--:|
| Backend Development | 189 | 358 | 547 | 27% | 72% |
| Database | 23 | 8 | 31 | 30% | 4% |
| APIs | 22 | 10 | 32 | 29% | 4% |
| Integration | 31 | 44 | 75 | 27% | 10% |
| Testing | 37 | 49 | 86 | 31% | 11% |
| Documentation | 10 | 8 | 18 | 33% | 2% |
| DevOps/Infra | - | 15 | 15 | 25% | 2% |
| **Total** | **280** | **484.5** | **764.5** | **27%** | 100% |

### 4.3 Complexity Distribution (Copilot-Optimized)

| Complexity | Modules | Base PD | With Copilot | Savings |
|------------|--------:|--------:|-------------:|--------:|
| Low | 2 | 46 | 28 | 39% |
| Medium | 4 | 255 | 174 | 32% |
| High | 8 | 745 | 562.5 | 25% |
| **Total** | **14** | **1,046** | **764.5** | **27%** |

*Note: Copilot provides greater productivity gains for lower-complexity work (boilerplate, CRUD, documentation) while complex algorithms and integrations retain most of their original estimates.*

---

## 5. Risk Assessment & Contingency

### 5.1 Identified Risks

| Risk ID | Risk Description | Probability | Impact | Mitigation |
|---------|------------------|-------------|--------|------------|
| R1 | Complex routing algorithm exceeds performance targets | Medium | High | Early performance prototyping, load testing |
| R2 | Rail integration delays due to external API changes | Medium | High | Mock rail interfaces early, buffer for integration |
| R3 | Concurrent limit checking race conditions | Medium | Medium | Atomic operations design, thorough concurrent testing |
| R4 | Workflow engine complexity underestimated | Medium | Medium | Iterative development, scope management |
| R5 | Cross-domain integration complexity | Medium | High | Early integration testing, well-defined contracts |
| R6 | Performance requirements (7,500 TPS) challenging | Medium | High | Performance-first design, continuous benchmarking |
| R7 | FTS integration complexity — 14 message types, SFTP file-based, 48hr TAT, inward & outward flows | High | High | Phased delivery (financial messages first), mock-file test suite for all message types |
| R8 | Copilot-generated code quality variance | Low | Low | Mandatory code reviews, test coverage thresholds |

*Note: With GitHub Copilot agent-assisted development, some risks are reduced (R4 reduced from High to Medium probability) due to faster prototyping and iteration capabilities.*

### 5.2 Contingency Calculation (Copilot-Optimized)

| Category | Base PD | With Copilot | Risk Factor | Contingency PD |
|----------|--------:|-------------:|------------:|---------------:|
| High Complexity Modules | 745 | 562.5 | 18% | 101.5 |
| Medium Complexity Modules | 255 | 174 | 12% | 21 |
| Low Complexity Modules | 46 | 28 | 8% | 2 |
| **Total Contingency** | | **764.5** | | **124.5** |
| **Applied Contingency (15%)** | | | | **115** |

### 5.3 Buffer Justification

A **15% contingency buffer** (reduced from 20%) is applied based on:
- GitHub Copilot accelerates prototyping and iteration, reducing discovery risk
- Faster feedback loops through AI-assisted development
- Reduced boilerplate errors through code generation
- However, maintaining buffer for:
  - Real-time performance requirements (7,500+ TPS)
  - Multiple external system integrations (4 payment rails)
  - First-time implementation of unified orchestration pattern

---

## 6. Dependencies

### 6.1 Technical Dependencies

| Dependency | Required By | Owner | Risk Level |
|------------|-------------|-------|------------|
| SQL Server cluster provisioning | Week 1 | Infrastructure | Low |
| Redis cluster setup | Week 1 | Infrastructure | Low |
| Kafka cluster setup | Week 1 | Infrastructure | Low |
| Kubernetes namespace & RBAC | Week 1 | Platform | Low |
| IPP Rail sandbox access | Week 8 | External - IPP Provider | Medium |
| IPI Rail sandbox access | Week 8 | External - IPI Provider | Medium |
| WPS sandbox access | Week 9 | External - WPS/Ministry of Labour | Medium |
| CB SFTP sandbox access | Week 10 | External - Central Bank | High |
| FTS message format specifications | Week 8 | External - Central Bank | Medium |
| Core Banking integration specs | Week 4 | Core Banking Team | Medium |

### 6.2 Domain Dependencies

```
Payment Orchestration Domain
    └──► Configuration Domain
            ├── Limit Definitions (Limit Control Engine)
            ├── Rules Repository (Validation Pipeline)
            ├── Holiday Calendar (Validation Pipeline)
            ├── Participant Directory (Routing Engine)
            └── Cut-off Times (Validation Pipeline)
```

**Recommendation**: Develop Configuration Domain core modules first, with Payment Orchestration following 2-3 weeks behind.

---

## 7. Team Structure & Timeline

### 7.1 Recommended Team Composition (Copilot-Optimized)

| Role | Count | Allocation | Responsibility |
|------|------:|------------|----------------|
| Tech Lead / Architect | 1 | 100% | Architecture, code reviews, integration design, Copilot best practices |
| Senior Backend Engineer | 2 | 100% | Core module development, complex algorithms, Copilot workflow optimization |
| Backend Engineer (Mid) | 2 | 100% | Module development, API implementation |
| Backend Engineer (Junior) | 1 | 100% | CRUD operations, documentation, testing (high Copilot leverage) |
| DevOps Engineer | 1 | 75% | CI/CD, Kubernetes, monitoring |
| QA Engineer | 1 | 100% | Test automation, performance testing |
| **Total** | **8** | | |

*Note: Team size reduced from 12 to 8 engineers due to Copilot productivity gains. Junior engineers can leverage Copilot effectively for boilerplate tasks, allowing seniors to focus on complex algorithms and architecture.*

### 7.2 Phase-wise Timeline (Copilot-Optimized)

| Phase | Duration | Weeks | Focus Areas |
|-------|----------|-------|-------------|
| **Phase 1: Foundation** | 3 weeks | 1-3 | Service scaffolding, database schemas, TransactionContext, Time & Calendar module, Reference Data module |
| **Phase 2: Business Logic** | 3 weeks | 4-6 | Business Rules engine, Workflow engine, Validation Pipeline, Gateway Core |
| **Phase 3: Core Processing** | 4 weeks | 7-10 | Limit Control Engine, Queue Management, Status & Lifecycle, Financial Configuration |
| **Phase 4: Routing & Execution** | 6 weeks | 11-16 | Embedded Routing Engine, Rail adapters (IPP, IPI, FTS-NG, WPS) |
| **Phase 5: Production Readiness** | 2 weeks | 17-18 | Performance optimization, security hardening, monitoring, documentation |

**Total Duration: 18 weeks** (reduced from 20 weeks)

### 7.3 Gantt Chart Summary

```
Week:        1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18
             ───────────────────────────────────────────────────────
Phase 1      ████████████
Phase 2                  ████████████
Phase 3                              ████████████████
Phase 4                                              ████████████████████████
Phase 5                                                                      ████████
             ───────────────────────────────────────────────────────
Config Domain├───────████████████████████████────────────────────────────────┤
Orchestration├──────────────███████████████████████████████████████████████████┤
```

---

## 8. Cost Estimation

*Note: Actual costs depend on team location and seniority mix. Below is a reference calculation optimized for GitHub Copilot-assisted development.*

| Resource Type | Daily Rate (USD) | Days | Total (USD) |
|---------------|----------------:|-----:|------------:|
| Tech Lead / Architect | $800 | 90 | $72,000 |
| Senior Backend Engineer (2) | $600 | 180 | $108,000 |
| Backend Engineer Mid (2) | $450 | 180 | $81,000 |
| Backend Engineer Junior (1) | $300 | 80 | $24,000 |
| DevOps Engineer (0.75 FTE) | $550 | 60 | $33,000 |
| QA Engineer (1) | $400 | 90 | $36,000 |
| **Subtotal** | | | **$354,000** |
| Contingency (15%) | | | $53,100 |
| GitHub Copilot Licenses (8 users × 4 months) | $19/month | 32 | $608 |
| **Total Estimated Cost** | | | **$407,708** |

### Cost Savings Summary

| Metric | Traditional | With Copilot | Savings |
|--------|------------:|-------------:|--------:|
| Team Size | 12 | 8 | 33% |
| Duration | 20 weeks | 18 weeks | 10% |
| Personnel Cost | $590,000 | $354,000 | $236,000 |
| Contingency | $118,000 | $53,100 | $64,900 |
| **Total Cost** | **$708,000** | **$407,708** | **$300,292 (42%)** |

---

## 9. Assumptions & Exclusions

### 9.1 Key Assumptions

| # | Assumption |
|---|------------|
| 1 | Development team has experience with .NET Core/ASP.NET, Kafka, Redis, SQL Server |
| 2 | Infrastructure (Kubernetes, databases) is pre-provisioned |
| 3 | External rail sandbox environments will be available on schedule |
| 4 | Requirements are stable; major changes will require re-estimation |
| 5 | Standard working hours (8h/day, 5 days/week) apply |
| 6 | Code reviews and documentation are included in task estimates |
| 7 | UAT support is included; production support is excluded |
| 8 | Third-party library licensing is pre-approved |

### 9.2 Exclusions

| # | Exclusion |
|---|-----------|
| 1 | Risk & Compliance Domain development (separate estimate) |
| 2 | UI/Frontend development (API-only scope) |
| 3 | Production environment setup and migration |
| 4 | Post-go-live production support |
| 5 | Training program development and delivery |
| 6 | Regulatory compliance certification |
| 7 | Data migration from legacy systems |
| 8 | Security penetration testing (assuming third-party service) |

---

## 10. Final Summary

| Metric | Traditional | With GitHub Copilot |
|--------|------------:|--------------------:|
| **Configuration Domain** | 398 Person-Days | 280 Person-Days |
| **Payment Orchestration Domain** | 648 Person-Days | 484.5 Person-Days |
| **Base Effort Total** | 1,046 Person-Days | 764.5 Person-Days |
| **Contingency** | 209 PD (20%) | 115 PD (15%) |
| **Grand Total** | **1,255 Person-Days** | **879.5 Person-Days** |
| **Recommended Duration** | **20 Weeks** | **18 Weeks** |
| **Recommended Team Size** | **12 Engineers** | **8 Engineers** |
| **Parallel Tracks** | 2 | 2 |
| **Estimated Cost** | **$708,000** | **$407,708** |

### Productivity Gains Summary

| Category | Productivity Gain |
|----------|------------------:|
| Overall Effort Reduction | 27% |
| Timeline Reduction | 10% |
| Team Size Reduction | 33% |
| Cost Reduction | 42% |

### Confidence Level

| Aspect | Confidence |
|--------|------------|
| Scope Definition | 85% |
| Technical Complexity Assessment | 82% |
| Copilot Productivity Estimates | 80% |
| Integration Points | 78% |
| External Dependencies | 70% |
| **Overall Estimate Confidence** | **80%** |

---

## Approval & Sign-off

| Role | Name | Signature | Date |
|------|------|-----------|------|
| Project Manager | | | |
| Technical Lead | | | |
| Client Representative | | | |

---

*Document End*

*This estimation document is provided for planning purposes. Actual effort may vary based on detailed technical discovery, team capability assessment, and requirement refinement.*
