# Configuration Management Service
## Unified End-to-End Design Document

---

### Document Control

| Version | Date | Author | Description |
|---------|------|--------|-------------|
| 1.0 | February 20, 2026 | Payment Hub Team | Initial Release |
| 2.0 | February 23, 2026 | Payment Hub Team | Consolidated into single service document |

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Service Overview](#2-service-overview)
3. [Module Architecture](#3-module-architecture)
4. [Functional Requirements Summary](#4-functional-requirements-summary)
5. [Domain Model](#5-domain-model)
6. [Data Architecture](#6-data-architecture)
7. [API Design](#7-api-design)
8. [Event-Driven Architecture](#8-event-driven-architecture)
9. [Sequence Diagrams](#9-sequence-diagrams)
10. [Integration Points](#10-integration-points)
11. [Security Design](#11-security-design)
12. [Non-Functional Requirements](#12-non-functional-requirements)
13. [Deployment Architecture](#13-deployment-architecture)
14. [Monitoring & Alerting](#14-monitoring--alerting)
15. [Test Strategy](#15-test-strategy)
16. [Implementation Roadmap](#16-implementation-roadmap)

---

## 1. Executive Summary

The **Configuration Management Service** is a foundational microservice of the Payment Hub system responsible for managing all configuration, reference data, business rules, financial settings, and time/calendar operations required for payment processing. This unified service consolidates four critical modules into a cohesive configuration layer that supports the entire payment ecosystem.

### Service Information

| Attribute | Value |
|-----------|-------|
| **Service Name** | Configuration Management Service |
| **Service Port** | 8002 |
| **Bounded Context** | Configuration Management Domain |
| **Database** | Microsoft SQL Server |
| **Cache Layer** | Redis |
| **Message Broker** | Apache Kafka |

### Consolidated Modules

| Module | Description |
|--------|-------------|
| **Time & Calendar Management** | Holiday calendars, cut-off times, business day calculations |
| **Reference Data Management** | Participant directory, bank codes, validation rules |
| **Business Rules & Workflows** | Rules engine, approval workflows, escalation management |
| **Financial Configuration** | Charges, pricing, limits, Vostro/GL management |

### Key Capabilities

| Capability | Module | Description |
|------------|--------|-------------|
| Holiday Calendar Management | Time & Calendar | Multi-country, multi-region holiday calendars |
| Cut-off Time Configuration | Time & Calendar | Configurable cut-offs per rail, currency, payment type |
| Business Day Calculation | Time & Calendar | Settlement date computation across jurisdictions |
| Participant Directory | Reference Data | Banks, financial institutions, payment providers |
| Banking Validation | Reference Data | BIC, IBAN, account number validation |
| Rules Repository | Business Rules | Centralized storage of business rules with versioning |
| Workflow Orchestration | Business Rules | Approval workflows with maker-checker support |
| Charge & Pricing Engine | Financial Config | Fee calculation with tiered pricing models |
| Limit Management | Financial Config | Multi-dimensional transaction limits |
| Vostro Account Management | Financial Config | Settlement accounts with real-time balance tracking |

### Business Value

- **Centralized Configuration**: Single source of truth for all payment configuration
- **Operational Efficiency**: Automated processing reduces manual intervention
- **Regulatory Compliance**: Complete audit trails and approval workflows
- **Global Operations**: Multi-country, multi-currency support
- **Real-time Processing**: Sub-millisecond configuration lookups
- **Business Agility**: Configuration changes without code deployments

---

## 2. Service Overview

### 2.1 Service Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                       CONFIGURATION MANAGEMENT SERVICE (Port 8002)                   │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                              API GATEWAY LAYER                               │   │
│  │  REST APIs │ GraphQL │ gRPC │ WebSocket (Real-time Updates)                 │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                            │
│  ┌─────────────────────────────────────┼───────────────────────────────────────┐   │
│  │                           SERVICE LAYER                                      │   │
│  ├──────────────────┬──────────────────┼──────────────────┬─────────────────────┤   │
│  │                  │                  │                  │                     │   │
│  │  ┌────────────┐  │  ┌────────────┐  │  ┌────────────┐  │  ┌────────────┐    │   │
│  │  │   TIME &   │  │  │ REFERENCE  │  │  │  BUSINESS  │  │  │ FINANCIAL  │    │   │
│  │  │  CALENDAR  │  │  │    DATA    │  │  │   RULES &  │  │  │   CONFIG   │    │   │
│  │  │   MODULE   │  │  │   MODULE   │  │  │  WORKFLOWS │  │  │   MODULE   │    │   │
│  │  │            │  │  │            │  │  │   MODULE   │  │  │            │    │   │
│  │  ├────────────┤  │  ├────────────┤  │  ├────────────┤  │  ├────────────┤    │   │
│  │  │• Calendars │  │  │• Partici-  │  │  │• Rules     │  │  │• Charges   │    │   │
│  │  │• Holidays  │  │  │  pants     │  │  │• Workflows │  │  │• Limits    │    │   │
│  │  │• Cut-offs  │  │  │• Validators│  │  │• Approvals │  │  │• Vostro    │    │   │
│  │  │• Timezones │  │  │• BIC/IBAN  │  │  │• Escalation│  │  │• GL/Accts  │    │   │
│  │  │• Bus. Days │  │  │• Currencies│  │  │• SLAs      │  │  │• Pricing   │    │   │
│  │  └────────────┘  │  └────────────┘  │  └────────────┘  │  └────────────┘    │   │
│  │                  │                  │                  │                     │   │
│  └──────────────────┴──────────────────┴──────────────────┴─────────────────────┘   │
│                                        │                                            │
│  ┌─────────────────────────────────────┼───────────────────────────────────────┐   │
│  │                        DATA ACCESS LAYER                                     │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │   │
│  │  │ Repository  │  │   Cache     │  │   Event     │  │   Audit Logger      │ │   │
│  │  │   Layer     │  │   Manager   │  │   Producer  │  │                     │ │   │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘ │   │
│  └─────────┼────────────────┼────────────────┼───────────────────┼─────────────┘   │
│            │                │                │                   │                  │
└────────────┼────────────────┼────────────────┼───────────────────┼──────────────────┘
             │                │                │                   │
             ▼                ▼                ▼                   ▼
      ┌────────────┐   ┌────────────┐   ┌────────────┐      ┌────────────┐
      │ SQL Server │   │   Redis    │   │   Kafka    │      │  Audit DB  │
      │  Primary   │   │  Cluster   │   │  Cluster   │      │            │
      └────────────┘   └────────────┘   └────────────┘      └────────────┘
```

### 2.2 Module Responsibilities

| Module | Responsibilities |
|--------|------------------|
| **Time & Calendar** | Calendar maintenance, cut-off configuration, holiday lookups, business day calculation, cut-off validation, time conversions, calendar publishing |
| **Reference Data** | Participant registry, classification management, status monitoring, settlement configuration, banking validation, reference data lookup, rail capability management |
| **Business Rules** | Rule management, rule evaluation, workflow orchestration, approval processing, escalation handling, SLA monitoring, notification delivery, decision auditing |
| **Financial Config** | Charge configuration, fee calculation, Vostro management, GL mapping, limit definition, limit enforcement, breach handling, financial reporting |

### 2.3 Integration with Payment Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              PAYMENT PROCESSING FLOW                                 │
└─────────────────────────────────────────────────────────────────────────────────────┘

                              Payment Request
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                         VALIDATION PIPELINE                                           │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐              │
│  │   Schema    │──▶│  Business   │──▶│  Reference  │──▶│  Duplicate  │              │
│  │ Validation  │   │   Rules     │   │    Data     │   │  Detection  │              │
│  └─────────────┘   └──────┬──────┘   └──────┬──────┘   └─────────────┘              │
│                           │                 │                                        │
│                           ▼                 ▼                                        │
│                    ┌──────────────────────────────────────┐                         │
│                    │   CONFIGURATION MANAGEMENT SERVICE    │                         │
│                    │  • Rules Repository & Engine          │                         │
│                    │  • Reference Data Validation          │                         │
│                    │  • Cut-off & Calendar Checks          │                         │
│                    └──────────────────────────────────────┘                         │
└──────────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                         LIMIT CONTROL ENGINE                                          │
│                                                                                      │
│  ┌─────────────┐        ┌──────────────────────────────────────┐                    │
│  │Limit Breach?│───────▶│   CONFIGURATION MANAGEMENT SERVICE    │                    │
│  └─────────────┘        │  • Limit Definitions & Checks         │                    │
│        │                │  • Workflow Engine for Approvals      │                    │
│        ▼                │  • Escalation Management              │                    │
│  Route to Approval      └──────────────────────────────────────┘                    │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                         ROUTING ENGINE                                                │
│                                                                                      │
│  ┌─────────────┐        ┌──────────────────────────────────────┐                    │
│  │   Select    │───────▶│   CONFIGURATION MANAGEMENT SERVICE    │                    │
│  │    Rail     │        │  • Participant Directory & Status     │                    │
│  └─────────────┘        │  • Routing Rules                      │                    │
│        │                │  • Cut-off Status                     │                    │
│        ▼                │  • Charge Calculation                 │                    │
│  Route Payment          └──────────────────────────────────────┘                    │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Module Architecture

### 3.1 Time & Calendar Management Module

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TIME & CALENDAR MANAGEMENT MODULE                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │  Holiday Calendar   │  │   Cut-off Time      │  │   Business Day      │ │
│  │    Management       │  │   Configuration     │  │   Calculator        │ │
│  │                     │  │                     │  │                     │ │
│  │ • Country Calendars │  │ • Rail Cut-offs     │  │ • Next Business Day │ │
│  │ • Regional Calendars│  │ • Currency Cut-offs │  │ • Settlement Date   │ │
│  │ • Bank Calendars    │  │ • Client Cut-offs   │  │ • Value Date Calc   │ │
│  │ • Holiday Types     │  │ • Channel Cut-offs  │  │ • T+N Calculations  │ │
│  │ • Recurring Rules   │  │ • Internal/External │  │ • Working Days      │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │   Timezone          │  │   Cut-off           │  │   Calendar          │ │
│  │   Management        │  │   Validation        │  │   Publishing        │ │
│  │                     │  │                     │  │                     │ │
│  │ • Global Timezones  │  │ • Real-time Check   │  │ • Client Export     │ │
│  │ • DST Handling      │  │ • Warning Alerts    │  │ • API Publishing    │ │
│  │ • Time Conversions  │  │ • Breach Detection  │  │ • Notifications     │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Reference Data Management Module

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    REFERENCE DATA MANAGEMENT MODULE                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │  Participant        │  │   Participant       │  │   Settlement        │ │
│  │    Directory        │  │   Classification    │  │   Accounts          │ │
│  │                     │  │                     │  │                     │ │
│  │ • Bank Registry     │  │ • Tier Levels       │  │ • Nostro Accounts   │ │
│  │ • BIC/SWIFT Codes   │  │ • Risk Ratings      │  │ • Vostro Accounts   │ │
│  │ • Routing Numbers   │  │ • Service Levels    │  │ • GL Mappings       │ │
│  │ • Contact Info      │  │ • Relationship Type │  │ • Currency Accounts │ │
│  │ • Operational Hours │  │ • Geographic Scope  │  │ • Account Status    │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │   Participant       │  │   Banking &         │  │   Reference Data    │ │
│  │   Status Monitor    │  │   Routing Valid.    │  │   Validation        │ │
│  │                     │  │                     │  │                     │ │
│  │ • Operational Status│  │ • IBAN Validation   │  │ • Currency Codes    │ │
│  │ • Service Health    │  │ • BIC Validation    │  │ • Country Codes     │ │
│  │ • Availability      │  │ • Account Format    │  │ • GL Accounts       │ │
│  │ • Routing Impact    │  │ • Routing Path      │  │ • Product Codes     │ │
│  │ • Uptime Stats      │  │ • Bank-Rail Check   │  │ • Branch Codes      │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Business Rules & Workflows Module

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    BUSINESS RULES & WORKFLOWS MODULE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │  Rules Repository   │  │   Rule Engine       │  │   Rule Versioning   │ │
│  │                     │  │                     │  │                     │ │
│  │ • Validation Rules  │  │ • Rule Evaluation   │  │ • Version Control   │ │
│  │ • Routing Rules     │  │ • Condition Parsing │  │ • Effective Dating  │ │
│  │ • Decision Rules    │  │ • Action Execution  │  │ • Rollback Support  │ │
│  │ • Threshold Rules   │  │ • Priority Handling │  │ • A/B Testing       │ │
│  │ • Transformation    │  │ • Caching           │  │ • Audit History     │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │  Workflow Engine    │  │   Approval          │  │   Escalation        │ │
│  │                     │  │   Management        │  │   Management        │ │
│  │                     │  │                     │  │                     │ │
│  │ • Workflow Defs     │  │ • Maker-Checker     │  │ • Escalation Rules  │ │
│  │ • State Machine     │  │ • Multi-Level       │  │ • Timeout Handling  │ │
│  │ • Task Assignment   │  │ • Delegation        │  │ • Auto-Escalation   │ │
│  │ • Parallel Tasks    │  │ • Override Workflow │  │ • Notification      │ │
│  │ • Conditional Flow  │  │ • Approval History  │  │ • SLA Monitoring    │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │   SLA Management    │  │   Notification      │  │   Decision          │ │
│  │                     │  │   Templates         │  │   Audit             │ │
│  │                     │  │                     │  │                     │ │
│  │ • SLA Definitions   │  │ • Email Templates   │  │ • Rule Execution    │ │
│  │ • Breach Detection  │  │ • SMS Templates     │  │ • Decision Logs     │ │
│  │ • Performance Track │  │ • Push Notifications│  │ • Audit Trail       │ │
│  │ • Reporting         │  │ • Webhook Templates │  │ • Analytics         │ │
│  │ • Alerting          │  │ • Dynamic Content   │  │ • Compliance        │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.4 Financial Configuration Module

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FINANCIAL CONFIGURATION MODULE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │  Charge & Pricing   │  │   Vostro Account    │  │   GL Integration    │ │
│  │       Engine        │  │    Management       │  │                     │ │
│  │                     │  │                     │  │                     │ │
│  │ • Charge Definition │  │ • Account Registry  │  │ • GL Code Mapping   │ │
│  │ • Tiered Pricing    │  │ • Balance Tracking  │  │ • Entry Generation  │ │
│  │ • Customer Pricing  │  │ • Multi-Currency    │  │ • Reconciliation    │ │
│  │ • Promotional Rates │  │ • Limit Management  │  │ • Currency Accounts │ │
│  │ • Fee Calculation   │  │ • Alert Management  │  │ • Account Hierarchy │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │   Limit Definition  │  │   Real-time Limit   │  │   Limit Breach      │ │
│  │    & Management     │  │      Checking       │  │    Handling         │ │
│  │                     │  │                     │  │                     │ │
│  │ • User Limits       │  │ • Concurrent Check  │  │ • Breach Actions    │ │
│  │ • Account Limits    │  │ • Utilization Track │  │ • Alert Generation  │ │
│  │ • Customer Limits   │  │ • Hold Management   │  │ • Override Workflow │ │
│  │ • Channel Limits    │  │ • Release Logic     │  │ • Escalation Rules  │ │
│  │ • System Limits     │  │ • Hierarchy Check   │  │ • Breach Analytics  │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Functional Requirements Summary

### 4.1 Time & Calendar Management Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| FR-CFG-001 | Create and maintain multiple holiday calendars (country, region, bank-specific) | High |
| FR-CFG-002 | Check holiday status during cut-off validation | High |
| FR-CFG-003 | Define cut-off times against multiple parameters | High |
| FR-CFG-004 | Define client-specific cut-off times overriding defaults | High |
| FR-CFG-005 | Block payments beyond cut-off (unless overridden) | High |
| FR-CFG-006 | Display current cut-off status for all rails and currencies | High |
| FR-CFG-037 | Check payment against applicable cut-off times | High |

### 4.2 Reference Data Management Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| FR-CFG-010 | Maintain comprehensive participant information | High |
| FR-CFG-011 | Define participant categories (Tier 1, Tier 2, Correspondent, Direct) | High |
| FR-CFG-012 | Track participant operational status | High |
| FR-CFG-033 | Account number and bank code validation | High |
| FR-CFG-034 | Internal GL account and reference data validations | High |

### 4.3 Business Rules & Workflows Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| FR-CFG-013 | Define validation rules with conditions and actions | High |
| FR-CFG-014 | Maintain version history for all rules | High |
| FR-CFG-015 | Evaluate rules in real-time during payment processing | High |
| FR-CFG-016 | Define workflow templates with states and transitions | High |
| FR-CFG-017 | Support maker-checker approval pattern | High |
| FR-CFG-018 | Define time-based escalation rules | High |
| FR-CFG-019 | Define SLA targets per payment type, channel, customer | High |
| FR-CFG-020 | Define notification templates per event type | High |
| FR-CFG-021 | Log all rule evaluations with input/output | High |

### 4.4 Financial Configuration Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| FR-CFG-007 | Define charge slabs with flat and percentage-based fees | High |
| FR-CFG-008 | Define pre and post-processing charges | High |
| FR-CFG-009 | Generate revenue reports by charge type, customer, product | High |
| FR-CFG-023 | Maintain Vostro account registry with balance, limits, status | High |
| FR-CFG-024 | Maintain mapping between payment types and GL codes | High |
| FR-CFG-025 | Maintain separate accounts per currency | High |
| FR-CFG-026 | Define limits at user, account, customer, channel, product levels | High |
| FR-CFG-027 | Check all applicable limits before payment authorization | High |
| FR-CFG-028 | Configure breach actions: block, warn, route to approval | High |

---

## 5. Domain Model

### 5.1 Time & Calendar Domain Entities

```
┌─────────────────────────┐      1        *  ┌─────────────────────────┐
│    HolidayCalendar      │──────────────────│        Holiday          │
├─────────────────────────┤                  ├─────────────────────────┤
│ - calendarId: UUID      │                  │ - holidayId: UUID       │
│ - calendarCode: String  │                  │ - holidayDate: Date     │
│ - calendarName: String  │                  │ - holidayName: String   │
│ - calendarType: Enum    │                  │ - holidayType: Enum     │
│ - countryCode: String   │                  │ - isRecurring: Boolean  │
│ - regionCode: String    │                  │ - recurringRule: String │
│ - effectiveFrom: Date   │                  │ - applicableRails: List │
│ - effectiveTo: Date     │                  │ - effectiveFrom: Date   │
│ - version: Integer      │                  │ - status: Enum          │
│ - status: Enum          │                  └─────────────────────────┘
└─────────────────────────┘

┌─────────────────────────┐      1        *  ┌─────────────────────────┐
│      CutoffRule         │──────────────────│    CutoffSchedule       │
├─────────────────────────┤                  ├─────────────────────────┤
│ - cutoffId: UUID        │                  │ - scheduleId: UUID      │
│ - cutoffCode: String    │                  │ - dayOfWeek: Enum       │
│ - cutoffType: Enum      │                  │ - cutoffTime: Time      │
│ - paymentType: Enum     │                  │ - timezone: String      │
│ - paymentRail: Enum     │                  │ - isPreHoliday: Boolean │
│ - currency: String      │                  │ - effectiveFrom: Date   │
│ - clientId: String      │                  │ - effectiveTo: Date     │
│ - priority: Integer     │                  │ - status: Enum          │
│ - warningMinutes: Int   │                  └─────────────────────────┘
│ - status: Enum          │
└─────────────────────────┘
```

### 5.2 Reference Data Domain Entities

```
┌─────────────────────────┐      1        *  ┌─────────────────────────┐
│      Participant        │──────────────────│   SettlementAccount     │
├─────────────────────────┤                  ├─────────────────────────┤
│ - participantId: UUID   │                  │ - accountId: UUID       │
│ - participantCode: Str  │                  │ - accountNumber: String │
│ - legalName: String     │                  │ - accountType: Enum     │
│ - participantType: Enum │                  │ - currency: String      │
│ - bicCode: String       │                  │ - glCode: String        │
│ - countryCode: String   │                  │ - accountStatus: Enum   │
│ - primaryCurrency: Str  │                  │ - effectiveFrom: Date   │
│ - timezone: String      │                  └─────────────────────────┘
│ - status: Enum          │
│ - version: Integer      │       1        *  ┌─────────────────────────┐
└─────────────────────────┘──────────────────│   RailCapability        │
                                             ├─────────────────────────┤
┌─────────────────────────┐                  │ - capabilityId: UUID    │
│ ParticipantClassification│                 │ - paymentRail: Enum     │
├─────────────────────────┤                  │ - channelType: Enum     │
│ - classificationId: UUID│                  │ - maxAmount: Decimal    │
│ - tierLevel: Enum       │                  │ - minAmount: Decimal    │
│ - relationshipType: Enum│                  │ - isActive: Boolean     │
│ - riskRating: Enum      │                  └─────────────────────────┘
│ - serviceLevel: Enum    │
│ - geographicScope: Enum │
└─────────────────────────┘
```

### 5.3 Business Rules Domain Entities

```
┌─────────────────────────┐      1        *  ┌─────────────────────────┐
│     RuleDefinition      │──────────────────│     RuleCondition       │
├─────────────────────────┤                  ├─────────────────────────┤
│ - ruleId: UUID          │                  │ - conditionId: UUID     │
│ - ruleCode: String      │                  │ - conditionSequence: Int│
│ - ruleName: String      │                  │ - fieldName: String     │
│ - ruleCategory: Enum    │                  │ - operator: Enum        │
│ - ruleType: Enum        │                  │ - value: String         │
│ - priority: Integer     │                  │ - logicalOperator: Enum │
│ - isActive: Boolean     │                  └─────────────────────────┘
│ - effectiveFrom: Date   │
│ - effectiveTo: Date     │       1        *  ┌─────────────────────────┐
│ - version: Integer      │──────────────────│      RuleAction         │
│ - status: Enum          │                  ├─────────────────────────┤
└─────────────────────────┘                  │ - actionId: UUID        │
                                             │ - actionSequence: Int   │
                                             │ - actionType: Enum      │
                                             │ - actionConfig: JSON    │
                                             │ - stopOnAction: Boolean │
                                             └─────────────────────────┘

┌─────────────────────────┐      1        *  ┌─────────────────────────┐
│   WorkflowDefinition    │──────────────────│     WorkflowState       │
├─────────────────────────┤                  ├─────────────────────────┤
│ - workflowId: UUID      │                  │ - stateId: UUID         │
│ - workflowCode: String  │                  │ - stateName: String     │
│ - workflowName: String  │                  │ - stateType: Enum       │
│ - workflowType: Enum    │                  │ - stateOrder: Integer   │
│ - triggerEvent: String  │                  │ - entryAction: String   │
│ - initialState: String  │                  │ - timeoutMinutes: Int   │
│ - finalStates: List     │                  │ - isTerminal: Boolean   │
│ - version: Integer      │                  └─────────────────────────┘
│ - status: Enum          │
└─────────────────────────┘       1        *  ┌─────────────────────────┐
                                 ────────────│   WorkflowTransition    │
                                             ├─────────────────────────┤
                                             │ - transitionId: UUID    │
                                             │ - fromState: String     │
                                             │ - toState: String       │
                                             │ - triggerEvent: String  │
                                             │ - guardCondition: String│
                                             │ - requiredRole: String  │
                                             └─────────────────────────┘

┌─────────────────────────┐      1        *  ┌─────────────────────────┐
│   EscalationRule        │──────────────────│   EscalationLevel       │
├─────────────────────────┤                  ├─────────────────────────┤
│ - escalationId: UUID    │                  │ - levelId: UUID         │
│ - escalationCode: String│                  │ - levelNumber: Integer  │
│ - escalationName: String│                  │ - escalateAfterMins: Int│
│ - targetEntityType: Str │                  │ - assignToRole: String  │
│ - triggerCondition: JSON│                  │ - notificationTemplate: │
│ - isActive: Boolean     │                  │   String                │
└─────────────────────────┘                  │ - autoApproveOnTimeout: │
                                             │   Boolean               │
                                             └─────────────────────────┘

┌─────────────────────────┐      1        *  ┌─────────────────────────┐
│   SLADefinition         │──────────────────│   SLAMilestone          │
├─────────────────────────┤                  ├─────────────────────────┤
│ - slaId: UUID           │                  │ - milestoneId: UUID     │
│ - slaCode: String       │                  │ - milestoneName: String │
│ - slaName: String       │                  │ - fromStatus: String    │
│ - entityType: String    │                  │ - toStatus: String      │
│ - paymentType: String   │                  │ - targetMinutes: Int    │
│ - overallTargetMins: Int│                  │ - warningMinutes: Int   │
│ - measurementType: Enum │                  │ - breachAction: String  │
│ - isActive: Boolean     │                  └─────────────────────────┘
└─────────────────────────┘
```

### 5.4 Financial Configuration Domain Entities

```
┌─────────────────────────┐      1        *  ┌─────────────────────────┐
│     ChargeDefinition    │──────────────────│       ChargeSlab        │
├─────────────────────────┤                  ├─────────────────────────┤
│ - chargeId: UUID        │                  │ - slabId: UUID          │
│ - chargeCode: String    │                  │ - slabSequence: Int     │
│ - chargeName: String    │                  │ - minAmount: Decimal    │
│ - chargeType: Enum      │                  │ - maxAmount: Decimal    │
│ - calculationType: Enum │                  │ - flatFee: Decimal      │
│ - paymentType: Enum     │                  │ - percentageFee: Decimal│
│ - paymentRail: Enum     │                  │ - minimumCharge: Decimal│
│ - currency: String      │                  │ - maximumCharge: Decimal│
│ - debitTiming: Enum     │                  │ - effectiveFrom: Date   │
│ - effectiveFrom: Date   │                  └─────────────────────────┘
│ - version: Integer      │
│ - status: Enum          │      1        *  ┌─────────────────────────┐
└─────────────────────────┘──────────────────│   CustomerPricing       │
                                             ├─────────────────────────┤
                                             │ - pricingId: UUID       │
                                             │ - customerId: String    │
                                             │ - customerSegment: Enum │
                                             │ - discountPercent: Dec  │
                                             │ - agreementRef: String  │
                                             │ - effectiveFrom: Date   │
                                             └─────────────────────────┘

┌─────────────────────────┐      1        *  ┌─────────────────────────┐
│      VostroAccount      │──────────────────│    AccountBalance       │
├─────────────────────────┤                  ├─────────────────────────┤
│ - accountId: UUID       │                  │ - balanceId: UUID       │
│ - accountNumber: String │                  │ - balanceType: Enum     │
│ - accountName: String   │                  │ - currentBalance: Dec   │
│ - accountType: Enum     │                  │ - availableBalance: Dec │
│ - currency: String      │                  │ - heldAmount: Decimal   │
│ - holdingBankId: UUID   │                  │ - asOfTimestamp: TS     │
│ - dailyLimit: Decimal   │                  └─────────────────────────┘
│ - minBalanceThreshold:  │
│   Decimal               │
│ - status: Enum          │
└─────────────────────────┘

┌─────────────────────────┐      1        *  ┌─────────────────────────┐
│     LimitDefinition     │──────────────────│    LimitThreshold       │
├─────────────────────────┤                  ├─────────────────────────┤
│ - limitId: UUID         │                  │ - thresholdId: UUID     │
│ - limitCode: String     │                  │ - thresholdType: Enum   │
│ - limitName: String     │                  │ - thresholdValue: Dec   │
│ - limitLevel: Enum      │                  │ - warningPercent: Int   │
│ - limitType: Enum       │                  │ - action: Enum          │
│ - entityType: Enum      │                  └─────────────────────────┘
│ - timeDimension: Enum   │
│ - limitAmount: Decimal  │       1        *  ┌─────────────────────────┐
│ - limitCount: Integer   │──────────────────│    LimitUtilization     │
│ - currency: String      │                  ├─────────────────────────┤
│ - breachAction: Enum    │                  │ - utilizationId: UUID   │
│ - overrideAllowed: Bool │                  │ - periodStart: DateTime │
│ - effectiveFrom: Date   │                  │ - utilizedAmount: Dec   │
│ - status: Enum          │                  │ - utilizedCount: Int    │
└─────────────────────────┘                  │ - remainingAmount: Dec  │
                                             │ - lastUpdated: TS       │
                                             └─────────────────────────┘
```

### 5.5 Core Enumerations

```java
// Time & Calendar Enums
enum CalendarType { NATIONAL, REGIONAL, BANK_SPECIFIC, CURRENCY, CUSTOM }
enum HolidayType { NATIONAL_HOLIDAY, REGIONAL_HOLIDAY, BANK_HOLIDAY, SETTLEMENT_HOLIDAY, HALF_DAY }
enum CutoffType { HARD_CUTOFF, SOFT_CUTOFF, WARNING_CUTOFF, INTERNAL_CUTOFF, EXTERNAL_CUTOFF }
enum CutoffValidationResult { ALLOWED, WARNING, BLOCKED, ADJUSTED, OVERRIDE_REQUIRED, HOLIDAY_IMPACT }

// Reference Data Enums
enum ParticipantType { BANK, CENTRAL_BANK, FINANCIAL_INSTITUTION, PAYMENT_SERVICE_PROVIDER, CLEARING_HOUSE, CORRESPONDENT_BANK }
enum OperationalStatus { ACTIVE, DEGRADED, MAINTENANCE, SUSPENDED, INACTIVE }
enum TierLevel { TIER_1, TIER_2, TIER_3, TIER_4 }
enum RiskRating { LOW, MEDIUM, HIGH, CRITICAL }
enum BankCodeType { BIC, IFSC, ROUTING_NUMBER, SORT_CODE, BSB, FEDWIRE, CHIPS_UID }

// Business Rules Enums
enum RuleCategory { VALIDATION, ROUTING, DECISION, TRANSFORMATION, ENRICHMENT, NOTIFICATION }
enum RuleType { SIMPLE, COMPOSITE, DECISION_TABLE, DECISION_TREE, EXPRESSION, SCRIPT }
enum ConditionOperator { EQUALS, NOT_EQUALS, GREATER_THAN, LESS_THAN, CONTAINS, IN, BETWEEN, MATCHES_REGEX }
enum ActionType { VALIDATE, REJECT, ROUTE, TRANSFORM, ENRICH, NOTIFY, ESCALATE, HOLD, APPROVE, CUSTOM }
enum WorkflowType { APPROVAL, REVIEW, EXCEPTION, ONBOARDING, AMENDMENT, CANCELLATION }
enum TaskStatus { PENDING, ASSIGNED, IN_PROGRESS, COMPLETED, REJECTED, ESCALATED, EXPIRED, CANCELLED }

// Financial Configuration Enums
enum ChargeType { TRANSACTION_FEE, SERVICE_FEE, PROCESSING_FEE, SWIFT_FEE, CORRESPONDENT_FEE, CURRENCY_FEE, URGENT_FEE }
enum CalculationType { FLAT, PERCENTAGE, TIERED, COMBINED, VOLUME_BASED, MINIMUM_MAXIMUM }
enum DebitTiming { PRE_PROCESSING, POST_PROCESSING, ON_SUCCESS, ON_FAILURE, SCHEDULED }
enum LimitLevel { USER, ACCOUNT, CUSTOMER, CHANNEL, PRODUCT, PARTICIPANT, CURRENCY, SYSTEM }
enum LimitType { TRANSACTION_COUNT, TRANSACTION_AMOUNT, CUMULATIVE_AMOUNT, VELOCITY, CONCENTRATION, BALANCE, EXPOSURE }
enum TimeDimension { PER_TRANSACTION, HOURLY, DAILY, WEEKLY, MONTHLY, QUARTERLY, YEARLY }
enum BreachAction { BLOCK, WARN, ROUTE_TO_APPROVAL, MANUAL_REVIEW, ESCALATE, THROTTLE, ALERT_ONLY }

// Payment Rails
enum PaymentRail { IPP, FTS, IPI, FTS_4_0, SWIFT, RTGS, ACH }

// Common Enums
enum Status { ACTIVE, INACTIVE, PENDING_APPROVAL, EXPIRED, DRAFT }
```

---

## 6. Data Architecture

### 6.1 Database Schema Overview

```sql
-- ============================================================================
-- CONFIGURATION MANAGEMENT SERVICE - DATABASE SCHEMA
-- Database: Microsoft SQL Server
-- Service: Configuration Management Service (Port 8002)
-- ============================================================================

-- Note: Complete schema is split by module for maintainability
-- All tables are in the same database: ConfigurationDB
-- Schemas used: tcm (Time/Calendar), rdm (Reference Data), brw (Rules/Workflows), fin (Financial)
```

### 6.2 Time & Calendar Management Tables

```sql
-- Schema: tcm (Time & Calendar Management)
CREATE SCHEMA tcm;
GO

-- Holiday Calendars Master
CREATE TABLE tcm.holiday_calendars (
    calendar_id         UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    calendar_code       VARCHAR(20) NOT NULL UNIQUE,
    calendar_name       NVARCHAR(100) NOT NULL,
    calendar_type       VARCHAR(20) NOT NULL,
    country_code        VARCHAR(3) NOT NULL,
    region_code         VARCHAR(10),
    effective_from      DATE NOT NULL,
    effective_to        DATE,
    version             INT NOT NULL DEFAULT 1,
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    created_by          VARCHAR(50) NOT NULL,
    created_at          DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    modified_by         VARCHAR(50),
    modified_at         DATETIME2
);

-- Holidays
CREATE TABLE tcm.holidays (
    holiday_id          UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    calendar_id         UNIQUEIDENTIFIER NOT NULL REFERENCES tcm.holiday_calendars(calendar_id),
    holiday_date        DATE NOT NULL,
    holiday_name        NVARCHAR(100) NOT NULL,
    holiday_type        VARCHAR(30) NOT NULL,
    is_recurring        BIT DEFAULT 0,
    recurring_rule      VARCHAR(100),
    is_half_day         BIT DEFAULT 0,
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    created_by          VARCHAR(50) NOT NULL,
    created_at          DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

-- Cut-off Rules
CREATE TABLE tcm.cutoff_rules (
    cutoff_id           UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    cutoff_code         VARCHAR(30) NOT NULL UNIQUE,
    cutoff_name         NVARCHAR(100) NOT NULL,
    cutoff_type         VARCHAR(20) NOT NULL,
    payment_type        VARCHAR(30),
    payment_rail        VARCHAR(20),
    currency_code       VARCHAR(3),
    client_id           VARCHAR(50),
    channel_id          VARCHAR(30),
    priority            INT NOT NULL DEFAULT 100,
    calendar_id         UNIQUEIDENTIFIER REFERENCES tcm.holiday_calendars(calendar_id),
    default_cutoff_time TIME NOT NULL,
    default_timezone    VARCHAR(50) NOT NULL DEFAULT 'UTC',
    warning_minutes     INT DEFAULT 30,
    effective_from      DATE NOT NULL,
    effective_to        DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    created_at          DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

-- Cut-off Validation Log
CREATE TABLE tcm.cutoff_validations (
    validation_id       UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    payment_id          UNIQUEIDENTIFIER NOT NULL,
    cutoff_id           UNIQUEIDENTIFIER NOT NULL,
    validation_time     DATETIME2 NOT NULL,
    cutoff_time         TIME NOT NULL,
    remaining_minutes   INT NOT NULL,
    validation_result   VARCHAR(30) NOT NULL,
    original_value_date DATE,
    adjusted_value_date DATE,
    holiday_impact      BIT DEFAULT 0,
    warning_issued      BIT DEFAULT 0,
    created_at          DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);
```

### 6.3 Reference Data Management Tables

```sql
-- Schema: rdm (Reference Data Management)
CREATE SCHEMA rdm;
GO

-- Participants Master
CREATE TABLE rdm.participants (
    participant_id          UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    participant_code        VARCHAR(20) NOT NULL UNIQUE,
    legal_name              NVARCHAR(200) NOT NULL,
    short_name              NVARCHAR(50) NOT NULL,
    participant_type        VARCHAR(30) NOT NULL,
    bic_code                VARCHAR(11),
    country_code            VARCHAR(3) NOT NULL,
    primary_currency        VARCHAR(3) NOT NULL,
    timezone                VARCHAR(50) NOT NULL,
    effective_from          DATE NOT NULL,
    effective_to            DATE,
    version                 INT NOT NULL DEFAULT 1,
    status                  VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    created_by              VARCHAR(50) NOT NULL,
    created_at              DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

-- Participant Classifications
CREATE TABLE rdm.participant_classifications (
    classification_id       UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    participant_id          UNIQUEIDENTIFIER NOT NULL REFERENCES rdm.participants(participant_id),
    tier_level              VARCHAR(10) NOT NULL,
    relationship_type       VARCHAR(20) NOT NULL,
    risk_rating             VARCHAR(10) NOT NULL,
    service_level           VARCHAR(20) NOT NULL,
    effective_from          DATE NOT NULL,
    effective_to            DATE,
    created_at              DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

-- Settlement Accounts
CREATE TABLE rdm.settlement_accounts (
    account_id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    participant_id          UNIQUEIDENTIFIER NOT NULL REFERENCES rdm.participants(participant_id),
    account_number          VARCHAR(50) NOT NULL,
    account_type            VARCHAR(20) NOT NULL,
    currency_code           VARCHAR(3) NOT NULL,
    gl_code                 VARCHAR(30),
    is_default              BIT DEFAULT 0,
    account_status          VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    effective_from          DATE NOT NULL,
    created_at              DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

-- Currencies
CREATE TABLE rdm.currencies (
    currency_id             UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    currency_code           VARCHAR(3) NOT NULL UNIQUE,
    currency_name           NVARCHAR(100) NOT NULL,
    numeric_code            VARCHAR(3),
    decimal_places          INT NOT NULL DEFAULT 2,
    is_active               BIT DEFAULT 1
);

-- Countries
CREATE TABLE rdm.countries (
    country_id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    country_code            VARCHAR(2) NOT NULL UNIQUE,
    alpha3_code             VARCHAR(3) NOT NULL UNIQUE,
    country_name            NVARCHAR(100) NOT NULL,
    primary_currency        VARCHAR(3),
    is_sanctioned           BIT DEFAULT 0,
    is_active               BIT DEFAULT 1
);

-- BIC Directory
CREATE TABLE rdm.bic_directory (
    bic_id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    bic_code                VARCHAR(11) NOT NULL UNIQUE,
    institution_name        NVARCHAR(200) NOT NULL,
    country_code            VARCHAR(2) NOT NULL,
    city                    NVARCHAR(100),
    is_active               BIT DEFAULT 1
);

-- IBAN Rules
CREATE TABLE rdm.iban_rules (
    rule_id                 UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    country_code            VARCHAR(2) NOT NULL UNIQUE,
    iban_length             INT NOT NULL,
    bank_code_position      INT NOT NULL,
    bank_code_length        INT NOT NULL,
    account_position        INT NOT NULL,
    account_length          INT NOT NULL,
    check_digit_algo        VARCHAR(50) NOT NULL DEFAULT 'MOD97',
    format_pattern          VARCHAR(100) NOT NULL
);
```

### 6.4 Business Rules & Workflows Tables

```sql
-- Schema: brw (Business Rules & Workflows)
CREATE SCHEMA brw;
GO

-- Rule Definitions Master
CREATE TABLE brw.rule_definitions (
    rule_id             UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    rule_code           VARCHAR(50) NOT NULL UNIQUE,
    rule_name           NVARCHAR(200) NOT NULL,
    rule_category       VARCHAR(30) NOT NULL,
    rule_type           VARCHAR(30) NOT NULL,
    description         NVARCHAR(MAX),
    priority            INT NOT NULL DEFAULT 100,
    is_active           BIT DEFAULT 1,
    effective_from      DATE NOT NULL,
    effective_to        DATE,
    version             INT NOT NULL DEFAULT 1,
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    created_by          VARCHAR(50) NOT NULL,
    created_at          DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

-- Rule Conditions
CREATE TABLE brw.rule_conditions (
    condition_id        UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    rule_id             UNIQUEIDENTIFIER NOT NULL REFERENCES brw.rule_definitions(rule_id),
    condition_sequence  INT NOT NULL,
    field_name          VARCHAR(100) NOT NULL,
    operator            VARCHAR(30) NOT NULL,
    value               NVARCHAR(500),
    value_type          VARCHAR(30) NOT NULL DEFAULT 'STRING',
    logical_operator    VARCHAR(5) DEFAULT 'AND',
    created_at          DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

-- Rule Actions
CREATE TABLE brw.rule_actions (
    action_id           UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    rule_id             UNIQUEIDENTIFIER NOT NULL REFERENCES brw.rule_definitions(rule_id),
    action_sequence     INT NOT NULL,
    action_type         VARCHAR(30) NOT NULL,
    action_config       NVARCHAR(MAX),
    stop_on_action      BIT DEFAULT 0,
    created_at          DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

-- Workflow Definitions
CREATE TABLE brw.workflow_definitions (
    workflow_id         UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    workflow_code       VARCHAR(50) NOT NULL UNIQUE,
    workflow_name       NVARCHAR(200) NOT NULL,
    workflow_type       VARCHAR(30) NOT NULL,
    trigger_event       VARCHAR(100),
    initial_state       VARCHAR(50) NOT NULL,
    final_states        NVARCHAR(500) NOT NULL,
    version             INT NOT NULL DEFAULT 1,
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    effective_from      DATE NOT NULL,
    created_at          DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

-- Workflow States
CREATE TABLE brw.workflow_states (
    state_id            UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    workflow_id         UNIQUEIDENTIFIER NOT NULL REFERENCES brw.workflow_definitions(workflow_id),
    state_name          VARCHAR(50) NOT NULL,
    state_type          VARCHAR(30) NOT NULL,
    state_order         INT NOT NULL,
    timeout_minutes     INT,
    is_terminal         BIT DEFAULT 0,
    created_at          DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

-- Workflow Instances (Runtime)
CREATE TABLE brw.workflow_instances (
    instance_id         UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    workflow_id         UNIQUEIDENTIFIER NOT NULL REFERENCES brw.workflow_definitions(workflow_id),
    entity_type         VARCHAR(50) NOT NULL,
    entity_id           VARCHAR(100) NOT NULL,
    current_state       VARCHAR(50) NOT NULL,
    previous_state      VARCHAR(50),
    started_at          DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    completed_at        DATETIME2,
    initiated_by        VARCHAR(50) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE'
);

-- Escalation Rules
CREATE TABLE brw.escalation_rules (
    escalation_id       UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    escalation_code     VARCHAR(50) NOT NULL UNIQUE,
    escalation_name     NVARCHAR(200) NOT NULL,
    target_entity_type  VARCHAR(50) NOT NULL,
    trigger_condition   NVARCHAR(MAX),
    is_active           BIT DEFAULT 1,
    effective_from      DATE NOT NULL,
    created_at          DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

-- SLA Definitions
CREATE TABLE brw.sla_definitions (
    sla_id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    sla_code            VARCHAR(50) NOT NULL UNIQUE,
    sla_name            NVARCHAR(200) NOT NULL,
    entity_type         VARCHAR(50) NOT NULL,
    payment_type        VARCHAR(50),
    overall_target_mins INT NOT NULL,
    warning_threshold_percent INT DEFAULT 80,
    measurement_type    VARCHAR(30) NOT NULL DEFAULT 'BUSINESS_TIME',
    is_active           BIT DEFAULT 1,
    effective_from      DATE NOT NULL,
    created_at          DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

-- Notification Templates
CREATE TABLE brw.notification_templates (
    template_id         UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    template_code       VARCHAR(50) NOT NULL UNIQUE,
    template_name       NVARCHAR(200) NOT NULL,
    event_type          VARCHAR(100) NOT NULL,
    recipient_type      VARCHAR(30) NOT NULL,
    is_active           BIT DEFAULT 1,
    version             INT NOT NULL DEFAULT 1,
    effective_from      DATE NOT NULL,
    created_at          DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);
```

### 6.5 Financial Configuration Tables

```sql
-- Schema: fin (Financial Configuration)
CREATE SCHEMA fin;
GO

-- Charge Definitions Master
CREATE TABLE fin.charge_definitions (
    charge_id           UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    charge_code         VARCHAR(30) NOT NULL UNIQUE,
    charge_name         NVARCHAR(100) NOT NULL,
    charge_type         VARCHAR(30) NOT NULL,
    calculation_type    VARCHAR(20) NOT NULL,
    payment_type        VARCHAR(30),
    payment_rail        VARCHAR(20),
    currency            VARCHAR(3),
    debit_timing        VARCHAR(20) NOT NULL DEFAULT 'PRE_PROCESSING',
    effective_from      DATE NOT NULL,
    effective_to        DATE,
    version             INT NOT NULL DEFAULT 1,
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    created_by          VARCHAR(50) NOT NULL,
    created_at          DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

-- Charge Slabs (Tiered Pricing)
CREATE TABLE fin.charge_slabs (
    slab_id             UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    charge_id           UNIQUEIDENTIFIER NOT NULL REFERENCES fin.charge_definitions(charge_id),
    slab_sequence       INT NOT NULL,
    min_amount          DECIMAL(18,4) NOT NULL,
    max_amount          DECIMAL(18,4),
    flat_fee            DECIMAL(18,4) DEFAULT 0,
    percentage_fee      DECIMAL(8,6) DEFAULT 0,
    minimum_charge      DECIMAL(18,4),
    maximum_charge      DECIMAL(18,4),
    effective_from      DATE NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    created_at          DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

-- Vostro Accounts
CREATE TABLE fin.vostro_accounts (
    account_id          UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    account_number      VARCHAR(50) NOT NULL UNIQUE,
    account_name        NVARCHAR(100) NOT NULL,
    account_type        VARCHAR(20) NOT NULL,
    currency            VARCHAR(3) NOT NULL,
    holding_bank_id     UNIQUEIDENTIFIER,
    daily_limit         DECIMAL(18,4),
    min_balance_threshold DECIMAL(18,4),
    low_balance_threshold DECIMAL(18,4),
    gl_code             VARCHAR(20),
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    effective_from      DATE NOT NULL,
    created_at          DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

-- Account Balances (Real-time)
CREATE TABLE fin.account_balances (
    balance_id          UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    account_id          UNIQUEIDENTIFIER NOT NULL REFERENCES fin.vostro_accounts(account_id),
    balance_type        VARCHAR(20) NOT NULL DEFAULT 'BOOK',
    current_balance     DECIMAL(18,4) NOT NULL,
    available_balance   DECIMAL(18,4) NOT NULL,
    held_amount         DECIMAL(18,4) DEFAULT 0,
    balance_date        DATE NOT NULL,
    last_updated        DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

-- GL Accounts
CREATE TABLE fin.gl_accounts (
    gl_account_id       UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    gl_code             VARCHAR(20) NOT NULL UNIQUE,
    gl_name             NVARCHAR(100) NOT NULL,
    account_category    VARCHAR(20) NOT NULL,
    parent_gl_id        UNIQUEIDENTIFIER,
    currency            VARCHAR(3),
    is_active           BIT DEFAULT 1,
    effective_from      DATE NOT NULL,
    created_at          DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

-- GL Mappings
CREATE TABLE fin.gl_mappings (
    mapping_id          UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    payment_type        VARCHAR(30) NOT NULL,
    transaction_type    VARCHAR(30) NOT NULL,
    payment_rail        VARCHAR(20),
    currency            VARCHAR(3),
    debit_gl_code       VARCHAR(20) NOT NULL,
    credit_gl_code      VARCHAR(20) NOT NULL,
    effective_from      DATE NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    created_at          DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

-- Limit Definitions
CREATE TABLE fin.limit_definitions (
    limit_id            UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    limit_code          VARCHAR(30) NOT NULL UNIQUE,
    limit_name          NVARCHAR(100) NOT NULL,
    limit_level         VARCHAR(20) NOT NULL,
    limit_type          VARCHAR(30) NOT NULL,
    entity_type         VARCHAR(30),
    entity_id           VARCHAR(50),
    time_dimension      VARCHAR(20) NOT NULL,
    limit_amount        DECIMAL(18,4),
    limit_count         INT,
    currency            VARCHAR(3),
    breach_action       VARCHAR(30) NOT NULL,
    override_allowed    BIT DEFAULT 0,
    warning_threshold   INT DEFAULT 80,
    effective_from      DATE NOT NULL,
    effective_to        DATE,
    version             INT NOT NULL DEFAULT 1,
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    created_by          VARCHAR(50) NOT NULL,
    created_at          DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

-- Limit Utilization (Real-time tracking)
CREATE TABLE fin.limit_utilization (
    utilization_id      UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    limit_id            UNIQUEIDENTIFIER NOT NULL REFERENCES fin.limit_definitions(limit_id),
    entity_id           VARCHAR(50) NOT NULL,
    period_start        DATETIME2 NOT NULL,
    period_end          DATETIME2 NOT NULL,
    utilized_amount     DECIMAL(18,4) DEFAULT 0,
    utilized_count      INT DEFAULT 0,
    remaining_amount    DECIMAL(18,4),
    remaining_count     INT,
    utilization_percent DECIMAL(5,2),
    last_updated        DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE'
);

-- Limit Breaches
CREATE TABLE fin.limit_breaches (
    breach_id           UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    limit_id            UNIQUEIDENTIFIER NOT NULL REFERENCES fin.limit_definitions(limit_id),
    payment_id          UNIQUEIDENTIFIER,
    breach_time         DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    breach_type         VARCHAR(30) NOT NULL,
    attempted_amount    DECIMAL(18,4),
    limit_amount        DECIMAL(18,4),
    utilized_before     DECIMAL(18,4),
    excess_amount       DECIMAL(18,4),
    action_taken        VARCHAR(30) NOT NULL,
    override_applied    BIT DEFAULT 0,
    override_by         VARCHAR(50),
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    created_at          DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);
```

### 6.6 Redis Cache Schema

```
# ============================================================================
# REDIS CACHE SCHEMA - CONFIGURATION MANAGEMENT SERVICE
# ============================================================================

# Time & Calendar Module
Key: calendar:{country_code}:{calendar_type}
Value: JSON serialized HolidayCalendar
TTL: 24 hours

Key: holiday:{country_code}:{date:yyyy-MM-dd}
Value: JSON array of Holiday objects
TTL: 24 hours

Key: cutoff:status:{rail}:{currency}
Value: JSON serialized CutoffStatus
TTL: 60 seconds

Key: businessday:{country_code}:{date}:{days}
Value: Calculated business day date
TTL: 24 hours

# Reference Data Module
Key: participant:{participant_code}
Value: JSON serialized Participant with classifications
TTL: 1 hour

Key: participant:bic:{bic_code}
Value: Participant code
TTL: 24 hours

Key: participant:status:{participant_code}
Value: JSON serialized ParticipantStatus
TTL: 60 seconds

Key: currency:{currency_code}
Value: JSON serialized Currency
TTL: 24 hours

Key: country:{country_code}
Value: JSON serialized Country
TTL: 24 hours

Key: bic:{bic_code}
Value: JSON serialized BIC entry
TTL: 24 hours

Key: iban:rule:{country_code}
Value: JSON serialized IBAN validation rule
TTL: 24 hours

# Business Rules Module
Key: rule:{rule_code}
Value: JSON serialized RuleDefinition with conditions/actions
TTL: 15 minutes

Key: rule:category:{category}
Value: JSON array of active rule codes
TTL: 15 minutes

Key: workflow:{workflow_code}
Value: JSON serialized WorkflowDefinition
TTL: 1 hour

Key: sla:{sla_code}
Value: JSON serialized SLADefinition
TTL: 1 hour

Key: notification:template:{template_code}
Value: JSON serialized NotificationTemplate
TTL: 1 hour

# Financial Configuration Module
Key: charge:def:{chargeCode}
Value: JSON serialized ChargeDefinition with slabs
TTL: 1 hour

Key: charge:customer:{customerId}:{chargeCode}
Value: JSON serialized CustomerPricing
TTL: 30 minutes

Key: vostro:account:{accountId}
Value: JSON serialized VostroAccount
TTL: 5 minutes

Key: vostro:balance:{accountId}
Value: JSON serialized BalanceSummary
TTL: 30 seconds

Key: limit:def:{limitCode}
Value: JSON serialized LimitDefinition
TTL: 15 minutes

Key: limit:util:{limitId}:{entityId}:{periodKey}
Value: JSON serialized LimitUtilization
TTL: 30 seconds
```

---

## 7. API Design

### 7.1 API Endpoint Summary

| Module | Base Path | Description |
|--------|-----------|-------------|
| Time & Calendar | `/api/v1/calendars`, `/api/v1/cutoffs`, `/api/v1/business-days` | Holiday and cut-off management |
| Reference Data | `/api/v1/participants`, `/api/v1/validation`, `/api/v1/currencies`, `/api/v1/countries` | Participant and reference data |
| Business Rules | `/api/v1/rules`, `/api/v1/workflows`, `/api/v1/escalations`, `/api/v1/slas` | Rules and workflow management |
| Financial Config | `/api/v1/charges`, `/api/v1/vostro`, `/api/v1/limits`, `/api/v1/gl` | Financial configuration |

### 7.2 Time & Calendar APIs

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/calendars` | List all calendars |
| GET | `/api/v1/calendars/{countryCode}/is-holiday` | Check if date is holiday |
| GET | `/api/v1/calendars/{countryCode}/next-business-day` | Get next business day |
| POST | `/api/v1/cutoffs/validate` | Validate payment against cut-off |
| GET | `/api/v1/cutoffs/status` | Get all cut-off statuses |
| GET | `/api/v1/business-days/next` | Calculate next business day |
| GET | `/api/v1/business-days/add` | Add business days to date |

### 7.3 Reference Data APIs

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/participants` | List all participants |
| GET | `/api/v1/participants/{participantId}` | Get participant by ID |
| GET | `/api/v1/participants/bic/{bicCode}` | Get participant by BIC |
| POST | `/api/v1/validation/bic` | Validate BIC code |
| POST | `/api/v1/validation/iban` | Validate IBAN |
| POST | `/api/v1/validation/banking` | Full banking validation |
| GET | `/api/v1/currencies` | List all currencies |
| GET | `/api/v1/countries` | List all countries |

### 7.4 Business Rules & Workflows APIs

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/rules` | List all rules |
| GET | `/api/v1/rules/{ruleId}` | Get rule by ID |
| POST | `/api/v1/rules/evaluate` | Evaluate rules against context |
| GET | `/api/v1/workflows` | List all workflows |
| POST | `/api/v1/workflows/{workflowId}/instances` | Start workflow instance |
| POST | `/api/v1/workflows/instances/{instanceId}/transition` | Transition workflow state |
| GET | `/api/v1/escalations` | List escalation rules |
| GET | `/api/v1/slas` | List SLA definitions |

### 7.5 Financial Configuration APIs

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/charges` | List all charges |
| POST | `/api/v1/charges/calculate` | Calculate charges for payment |
| GET | `/api/v1/vostro` | List Vostro accounts |
| GET | `/api/v1/vostro/{accountId}/balance` | Get account balance |
| GET | `/api/v1/limits` | List all limits |
| POST | `/api/v1/limits/check` | Check limits for payment |
| GET | `/api/v1/gl/mappings` | List GL mappings |
| POST | `/api/v1/gl/entries` | Generate GL entries |

### 7.6 Sample API Requests/Responses

#### Validate Cut-off

```http
POST /api/v1/cutoffs/validate
Content-Type: application/json

{
  "paymentId": "PAY-2026022300001",
  "paymentType": "DOMESTIC",
  "paymentRail": "FTS",
  "currency": "USD",
  "clientId": "CLIENT001",
  "requestedValueDate": "2026-02-23"
}
```

**Response:**
```json
{
  "validationId": "VAL-550e8400-e29b",
  "paymentId": "PAY-2026022300001",
  "validationResult": "ALLOWED",
  "cutoffDetails": {
    "cutoffTime": "16:00:00",
    "currentTime": "14:30:00",
    "remainingMinutes": 90
  },
  "validatedAt": "2026-02-23T14:30:00Z"
}
```

#### Validate Banking Information

```http
POST /api/v1/validation/banking
Content-Type: application/json

{
  "iban": "DE89370400440532013000",
  "bicCode": "COBADEFFXXX",
  "paymentRail": "SWIFT",
  "currency": "EUR"
}
```

**Response:**
```json
{
  "isValid": true,
  "ibanValidation": {
    "isValid": true,
    "countryCode": "DE",
    "bankCode": "37040044"
  },
  "bicValidation": {
    "isValid": true,
    "institutionName": "Commerzbank AG"
  },
  "participantValidation": {
    "status": "ACTIVE",
    "supportsRail": true
  }
}
```

#### Evaluate Business Rules

```http
POST /api/v1/rules/evaluate
Content-Type: application/json

{
  "context": {
    "paymentType": "INTERNATIONAL",
    "amount": 50000.00,
    "currency": "USD",
    "customerSegment": "CORPORATE"
  },
  "ruleCategories": ["VALIDATION", "ROUTING"]
}
```

**Response:**
```json
{
  "evaluationId": "EVAL-550e8400",
  "results": [
    {
      "ruleCode": "VAL-INTL-001",
      "matched": true,
      "actions": [
        {"type": "VALIDATE", "result": "PASS"}
      ]
    },
    {
      "ruleCode": "ROUTE-SWIFT-001",
      "matched": true,
      "actions": [
        {"type": "ROUTE", "rail": "SWIFT"}
      ]
    }
  ],
  "evaluatedAt": "2026-02-23T14:30:00Z"
}
```

#### Check Limits

```http
POST /api/v1/limits/check
Content-Type: application/json

{
  "paymentId": "PAY-2026022300001",
  "customerId": "CUST001",
  "accountId": "ACCT001",
  "amount": 100000.00,
  "currency": "USD",
  "paymentType": "INTERNATIONAL"
}
```

**Response:**
```json
{
  "isAllowed": true,
  "checkedLimits": [
    {
      "limitCode": "CUST-DAILY-USD",
      "limitAmount": 500000.00,
      "utilizedAmount": 150000.00,
      "remainingAmount": 350000.00,
      "utilizationPercent": 30,
      "isBreached": false
    }
  ],
  "holdReference": "HOLD-2026022300001"
}
```

---

## 8. Event-Driven Architecture

### 8.1 Events Published

```yaml
# Time & Calendar Events
HolidayAdded:
  topic: payment-hub.calendar.holiday-added
  schema: { holidayId, calendarId, countryCode, holidayDate, holidayName, timestamp }

CutoffApproaching:
  topic: payment-hub.cutoff.approaching
  schema: { cutoffId, paymentRail, currency, remainingMinutes, alertLevel, timestamp }

CutoffBreached:
  topic: payment-hub.cutoff.breached
  schema: { cutoffId, paymentRail, currency, breachTime, affectedPayments, timestamp }

# Reference Data Events
ParticipantCreated:
  topic: payment-hub.participant.created
  schema: { participantId, participantCode, bicCode, countryCode, timestamp }

ParticipantStatusChanged:
  topic: payment-hub.participant.status-changed
  schema: { participantId, previousStatus, newStatus, routingImpact, timestamp }

# Business Rules Events
RuleActivated:
  topic: payment-hub.rule.activated
  schema: { ruleId, ruleCode, ruleCategory, version, activatedBy, timestamp }

WorkflowStarted:
  topic: payment-hub.workflow.started
  schema: { instanceId, workflowCode, entityType, entityId, initiatedBy, timestamp }

WorkflowCompleted:
  topic: payment-hub.workflow.completed
  schema: { instanceId, workflowCode, finalState, completedAt, timestamp }

ApprovalRequired:
  topic: payment-hub.approval.required
  schema: { taskId, instanceId, entityType, entityId, assignedTo, dueDate, timestamp }

EscalationTriggered:
  topic: payment-hub.escalation.triggered
  schema: { escalationId, entityType, entityId, level, assignedTo, timestamp }

SLABreached:
  topic: payment-hub.sla.breached
  schema: { slaId, entityType, entityId, targetMinutes, actualMinutes, timestamp }

# Financial Configuration Events
ChargeCalculated:
  topic: payment-hub.charge.calculated
  schema: { paymentId, chargeCode, totalCharge, currency, timestamp }

LimitBreached:
  topic: payment-hub.limit.breached
  schema: { limitId, entityId, breachType, attemptedAmount, limitAmount, actionTaken, timestamp }

LimitUtilizationWarning:
  topic: payment-hub.limit.warning
  schema: { limitId, entityId, utilizationPercent, warningThreshold, timestamp }

BalanceAlert:
  topic: payment-hub.vostro.balance-alert
  schema: { accountId, currentBalance, threshold, alertType, timestamp }
```

### 8.2 Events Consumed

```yaml
PaymentInitiated:
  source: payment-hub.payment.initiated
  handlers:
    - CutoffValidationHandler
    - LimitCheckHandler
    - ChargeCalculationHandler

PaymentCompleted:
  source: payment-hub.payment.completed
  handlers:
    - LimitReleaseHandler
    - GLEntryHandler
    - SLATrackingHandler

PaymentFailed:
  source: payment-hub.payment.failed
  handlers:
    - LimitReleaseHandler
    - ChargeReversalHandler
```

---

## 9. Sequence Diagrams

### 9.1 Payment Configuration Validation Flow

```
┌────────┐  ┌──────────┐  ┌──────────────────────────────────────────────┐
│Payment │  │   API    │  │       CONFIGURATION MANAGEMENT SERVICE        │
│Service │  │ Gateway  │  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────┐│
│        │  │          │  │  │Time/Cal │ │Ref Data │ │ Rules   │ │ Fin ││
└───┬────┘  └────┬─────┘  │  └────┬────┘ └────┬────┘ └────┬────┘ └──┬──┘│
    │            │        └───────┼──────────┼──────────┼─────────┼────┘
    │ Validate Payment Config     │          │          │         │
    │───────────>│                │          │          │         │
    │            │ Forward        │          │          │         │
    │            │────────────────┼──────────┼──────────┼─────────┤
    │            │                │          │          │         │
    │            │                │ Cut-off Check       │         │
    │            │                │◄─────────┤          │         │
    │            │                │          │          │         │
    │            │                │ Validate Participant│         │
    │            │                │──────────►          │         │
    │            │                │          │          │         │
    │            │                │          │ Evaluate Rules     │
    │            │                │          │─────────►│         │
    │            │                │          │          │         │
    │            │                │          │          │ Check Limits
    │            │                │          │          │────────►│
    │            │                │          │          │         │
    │            │ Consolidated Validation Result       │         │
    │            │◄───────────────┼──────────┼──────────┼─────────│
    │            │                │          │          │         │
    │ 200 OK {cutoffOk, participantValid, rulesPass, limitsOk}   │
    │◄───────────│                │          │          │         │
```

### 9.2 Approval Workflow Flow

```
┌────────┐  ┌──────────┐  ┌──────────────────────────┐  ┌─────────┐  ┌─────────┐
│Payment │  │  Config  │  │    Workflow Engine       │  │Notifica-│  │ Kafka   │
│Service │  │  Service │  │                          │  │tion Svc │  │         │
└───┬────┘  └────┬─────┘  └────────────┬─────────────┘  └────┬────┘  └────┬────┘
    │            │                     │                     │            │
    │ Limit Breach - Route to Approval │                     │            │
    │───────────>│                     │                     │            │
    │            │                     │                     │            │
    │            │ Start Approval Workflow                   │            │
    │            │────────────────────>│                     │            │
    │            │                     │                     │            │
    │            │                     │ Create Task Instance│            │
    │            │                     │──────┐              │            │
    │            │                     │      │              │            │
    │            │                     │◄─────┘              │            │
    │            │                     │                     │            │
    │            │                     │ Send Notification   │            │
    │            │                     │────────────────────>│            │
    │            │                     │                     │            │
    │            │                     │ Publish ApprovalRequired         │
    │            │                     │────────────────────────────────>│
    │            │                     │                     │            │
    │            │ Instance Created    │                     │            │
    │            │◄────────────────────│                     │            │
    │            │                     │                     │            │
    │ Queued for Approval              │                     │            │
    │◄───────────│                     │                     │            │
```

---

## 10. Integration Points

### 10.1 Internal Service Integration

| Consuming Service | APIs Used | Purpose |
|-------------------|-----------|---------|
| Payment Gateway Core | Cut-off validation, Participant lookup, Rule evaluation, Limit check | Payment processing |
| Routing Engine | Participant status, Rail capabilities, Routing rules | Payment routing decisions |
| Validation Pipeline | Business rules, Reference data validation | Payment validation |
| Limit Control Engine | Limit definitions, Limit checks, Breach handling | Limit enforcement |
| Queue Management | SLA definitions, Workflow status | Queue prioritization |
| Settlement Service | Vostro balances, GL mappings | Settlement processing |

### 10.2 External System Integration

| External System | Integration Type | Purpose |
|-----------------|------------------|---------|
| SWIFT Network | REST/File | BIC directory sync |
| Core Banking | REST/MQ | Account validation, Balance sync |
| GL System | REST/File | GL code sync, Entry posting |
| Notification Gateway | REST/Webhook | Email, SMS, Push notifications |
| Compliance Systems | REST | Sanction list sync |

---

## 11. Security Design

### 11.1 Authentication & Authorization

| Aspect | Implementation |
|--------|----------------|
| Authentication | OAuth 2.0 / JWT tokens |
| Authorization | Role-Based Access Control (RBAC) |
| API Security | TLS 1.3, API key validation |
| Data Encryption | AES-256 at rest, TLS in transit |
| Audit Logging | All configuration changes logged |

### 11.2 Role Permissions

| Role | Permissions |
|------|-------------|
| CONFIG_VIEWER | Read all configurations |
| CONFIG_MANAGER | Create/Update configurations (approval required) |
| CONFIG_ADMIN | Full access, approve changes |
| WORKFLOW_ADMIN | Manage workflows and approvals |
| FINANCIAL_ADMIN | Manage charges, limits, accounts |
| SYSTEM_ADMIN | Full system access |

---

## 12. Non-Functional Requirements

### 12.1 Performance Requirements

| Metric | Target |
|--------|--------|
| API Response Time (P99) | < 100ms |
| Configuration Lookup | < 10ms (cached) |
| Rule Evaluation | < 50ms |
| Limit Check | < 20ms |
| Throughput | 10,000 TPS |
| Availability | 99.99% |

### 12.2 Scalability

| Aspect | Approach |
|--------|----------|
| Horizontal Scaling | Kubernetes pod autoscaling |
| Database | Read replicas, connection pooling |
| Caching | Redis cluster with replication |
| Event Processing | Kafka partitioning |

---

## 13. Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              KUBERNETES CLUSTER                                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │                    CONFIGURATION MANAGEMENT SERVICE                           │  │
│  │                                                                              │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐            │  │
│  │  │   Pod 1    │  │   Pod 2    │  │   Pod 3    │  │   Pod N    │            │  │
│  │  │ (Primary)  │  │ (Replica)  │  │ (Replica)  │  │   (Auto)   │            │  │
│  │  └────────────┘  └────────────┘  └────────────┘  └────────────┘            │  │
│  │                                                                              │  │
│  │  ┌──────────────────────────────────────────────────────────────────────┐   │  │
│  │  │                     KUBERNETES SERVICE (LoadBalancer)                │   │  │
│  │  │                            Port: 8002                                │   │  │
│  │  └──────────────────────────────────────────────────────────────────────┘   │  │
│  │                                                                              │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐                 │
│  │   SQL Server     │  │   Redis Cluster  │  │   Kafka Cluster  │                 │
│  │   (Primary +     │  │   (3 nodes)      │  │   (3 brokers)    │                 │
│  │    Replicas)     │  │                  │  │                  │                 │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘                 │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 14. Monitoring & Alerting

### 14.1 Key Metrics

| Category | Metrics |
|----------|---------|
| API | Response time, error rate, throughput |
| Cache | Hit rate, miss rate, evictions |
| Database | Query time, connection pool usage |
| Business | Rule evaluations, workflow completions, limit breaches |
| Financial | Charge calculations, balance alerts |

### 14.2 Alerting Rules

| Alert | Condition | Severity |
|-------|-----------|----------|
| High API Latency | P99 > 200ms for 5 min | Warning |
| High Error Rate | Error rate > 1% for 5 min | Critical |
| Cache Miss High | Miss rate > 20% for 10 min | Warning |
| Cut-off Approaching | < 30 min to cut-off | Info |
| Limit Utilization High | > 90% utilization | Warning |
| Low Vostro Balance | Below threshold | Critical |

---

## 15. Test Strategy

### 15.1 Test Types

| Type | Coverage | Tools |
|------|----------|-------|
| Unit Tests | > 80% code coverage | JUnit, Mockito |
| Integration Tests | API endpoints, DB | TestContainers |
| Performance Tests | Load, stress testing | JMeter, Gatling |
| Contract Tests | API contracts | Pact |
| E2E Tests | Full payment flows | Selenium, Cypress |

### 15.2 Test Scenarios

| Module | Key Scenarios |
|--------|---------------|
| Time/Calendar | Holiday lookup, cut-off validation, business day calculation |
| Reference Data | Participant lookup, BIC/IBAN validation, status monitoring |
| Business Rules | Rule evaluation, workflow transitions, escalation triggers |
| Financial Config | Charge calculation, limit checks, balance updates |

---

## 16. Implementation Roadmap

### Phase 1: Foundation (Weeks 1-4)
- Service scaffolding and infrastructure setup
- Time & Calendar module implementation
- Reference Data module implementation
- Core database schema deployment

### Phase 2: Business Logic (Weeks 5-8)
- Business Rules engine implementation
- Workflow engine implementation
- Escalation and SLA management
- Notification framework

### Phase 3: Financial (Weeks 9-12)
- Charge & pricing engine
- Limit management system
- Vostro account management
- GL integration

### Phase 4: Integration & Testing (Weeks 13-16)
- Internal service integration
- External system integration
- Performance testing
- UAT and documentation

---

## Appendix A: Error Codes

| Code | Module | Description |
|------|--------|-------------|
| CFG-TCM-001 | Time/Calendar | Invalid date format |
| CFG-TCM-002 | Time/Calendar | Calendar not found |
| CFG-TCM-003 | Time/Calendar | Cut-off passed |
| CFG-RDM-001 | Reference Data | Participant not found |
| CFG-RDM-002 | Reference Data | Invalid BIC code |
| CFG-RDM-003 | Reference Data | Invalid IBAN |
| CFG-BRW-001 | Rules/Workflow | Rule not found |
| CFG-BRW-002 | Rules/Workflow | Rule evaluation failed |
| CFG-BRW-003 | Rules/Workflow | Workflow instance not found |
| CFG-FIN-001 | Financial | Charge not found |
| CFG-FIN-002 | Financial | Limit breached |
| CFG-FIN-003 | Financial | Insufficient balance |

---

## Appendix B: Configuration Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| cache.ttl.calendar | 24h | Calendar cache TTL |
| cache.ttl.participant | 1h | Participant cache TTL |
| cache.ttl.rule | 15m | Rule cache TTL |
| cache.ttl.balance | 30s | Balance cache TTL |
| cutoff.warning.minutes | 30 | Cut-off warning threshold |
| limit.warning.percent | 80 | Limit warning threshold |
| workflow.task.timeout | 60m | Default task timeout |
| escalation.check.interval | 5m | Escalation check interval |

---

*Document End*
