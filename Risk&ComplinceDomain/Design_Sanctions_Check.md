# Sanctions Check Module
## End-to-End Design Document

---

### Document Control

| Version | Date | Author | Description |
|---------|------|--------|-------------|
| 1.0 | February 20, 2026 | Payment Hub Team | Initial Release |

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Module Overview](#2-module-overview)
3. [Functional Requirements Summary](#3-functional-requirements-summary)
4. [Domain Model](#4-domain-model)
5. [Data Architecture](#5-data-architecture)
6. [API Design](#6-api-design)
7. [Event-Driven Architecture](#7-event-driven-architecture)
8. [Sequence Diagrams](#8-sequence-diagrams)
9. [Integration Points](#9-integration-points)
10. [Security Design](#10-security-design)
11. [Non-Functional Requirements](#11-non-functional-requirements)
12. [Deployment Architecture](#12-deployment-architecture)
13. [Monitoring & Alerting](#13-monitoring--alerting)
14. [Test Strategy](#14-test-strategy)
15. [Implementation Roadmap](#15-implementation-roadmap)

---

## 1. Executive Summary

The **Sanctions Check** module is a mission-critical component of the Risk & Compliance Domain responsible for screening payment transactions, counterparties, and related entities against global sanctions lists, embargoed country restrictions, and regulatory prohibition databases. This module ensures the organization does not process payments that violate international trade sanctions, export controls, or financial restrictions imposed by governmental and intergovernmental bodies.

### Key Capabilities

| Capability | Description |
|------------|-------------|
| Multi-List Screening | Simultaneous screening against OFAC, EU, UN, UK, and 30+ country-specific sanctions lists |
| Real-Time Transaction Screening | Sub-100ms screening for all payment transactions |
| Batch Screening | High-volume batch processing for customer onboarding and periodic reviews |
| Name Matching Engine | Advanced fuzzy matching with configurable algorithms and thresholds |
| Vessel & Aircraft Screening | Specialized screening for maritime and aviation sector transactions |
| Country & Region Screening | Embargo and restricted country/region detection |
| BIC/SWIFT Screening | Financial institution identifier screening |
| Address Screening | Geographic and address-based sanctions detection |
| Dual-Use Goods Detection | Export control screening for dual-use items |
| Real-Time List Updates | Continuous synchronization with official sanctions sources |
| Hit Disposition Workflow | Automated and manual review workflows for potential matches |
| Regulatory Reporting | Automated OFAC/SDN reporting and record retention |
| Audit Trail | Complete screening history for regulatory examination |
| Secondary Sanctions | Detection of indirect exposure through correspondent banking |

### Business Value

- **Regulatory Compliance**: Meet OFAC, EU, UN, and global sanctions requirements
- **License Protection**: Prevent regulatory penalties and license revocation
- **Risk Mitigation**: Block transactions involving sanctioned entities
- **Reputation Protection**: Avoid association with sanctioned parties
- **Operational Efficiency**: Automated screening reduces manual workload
- **False Positive Optimization**: Intelligent matching minimizes unnecessary reviews
- **Real-Time Protection**: Sub-second screening enables STP processing
- **Global Coverage**: Comprehensive multi-jurisdiction sanctions compliance
- **Audit Readiness**: Complete documentation for regulatory examinations
- **Cost Avoidance**: Prevent multi-million dollar sanctions violations penalties

### Architecture Context (ADR-005a)

The Sanctions Check module operates within the **Risk & Compliance Domain** and is triggered at multiple points in the payment lifecycle:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PAYMENT ORCHESTRATION DOMAIN                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Payment Gateway Core → Validation Pipeline → Limit Control Engine          │
│                              │                        │                     │
│                              │ (Pre-Validation)       │ (Post-Limit)        │
│                              ▼                        ▼                     │
└──────────────────────────────┼────────────────────────┼─────────────────────┘
                               │                        │
                    ┌──────────┼────────────────────────┼─────────────────────┐
                    │          │    RISK & COMPLIANCE DOMAIN                   │
                    ├──────────┼────────────────────────┼─────────────────────┤
                    │          ▼                        ▼                     │
                    │  ┌────────────────────────────────────────────────────┐ │
                    │  │        SANCTIONS CHECK MODULE (This Module)         │ │
                    │  │                                                     │ │
                    │  │   ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │ │
                    │  │   │   OFAC/SDN  │  │ EU Sanctions│  │ UN UNSCR   │ │ │
                    │  │   │  Screening  │  │  Screening  │  │ Screening  │ │ │
                    │  │   └─────────────┘  └─────────────┘  └────────────┘ │ │
                    │  │          │               │               │         │ │
                    │  │   ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │ │
                    │  │   │ UK Sanctions│  │Country Lists│  │ Custom     │ │ │
                    │  │   │  (OFSI)     │  │  Embargo    │  │ Watchlists │ │ │
                    │  │   └─────────────┘  └─────────────┘  └────────────┘ │ │
                    │  │          │               │               │         │ │
                    │  │          └───────────────┼───────────────┘         │ │
                    │  │                          ▼                         │ │
                    │  │   ┌─────────────────────────────────────────────┐ │ │
                    │  │   │           NAME MATCHING ENGINE              │ │ │
                    │  │   │  • Exact • Fuzzy • Phonetic • Transliterated│ │ │
                    │  │   └─────────────────────────────────────────────┘ │ │
                    │  │                          │                         │ │
                    │  │                          ▼                         │ │
                    │  │   ┌─────────────────────────────────────────────┐ │ │
                    │  │   │         SANCTIONS DECISION ENGINE           │ │ │
                    │  │   │  • CLEAR (Pass)  • REVIEW (Alert)  • BLOCK  │ │ │
                    │  │   └─────────────────────────────────────────────┘ │ │
                    │  │          │               │               │         │ │
                    │  │          ▼               ▼               ▼         │ │
                    │  │      Continue       Case Mgmt         Reject       │ │
                    │  │      Payment        Alert Queue       Payment      │ │
                    │  │                                                     │ │
                    │  └─────────────────────────────────────────────────────┘ │
                    │                                                          │
                    │  ┌──────────────────┐  ┌──────────────────────────────┐ │
                    │  │   AML Screening  │  │     Fraud Detection          │ │
                    │  └──────────────────┘  └──────────────────────────────┘ │
                    │                                                          │
                    └──────────────────────────────────────────────────────────┘
```

### Sanctions Lists Coverage

| List Authority | List Name | Update Frequency | Coverage |
|----------------|-----------|------------------|----------|
| OFAC (USA) | SDN List | Daily | Individuals, Entities, Vessels |
| OFAC (USA) | Consolidated Sanctions List | Daily | All OFAC Programs |
| OFAC (USA) | Sectoral Sanctions (SSI) | Daily | Russian/Ukrainian sectors |
| EU | Consolidated Financial Sanctions | Daily | EU Sanctions Regime |
| UN | UN Security Council Consolidated List | Weekly | Global Terrorism, Proliferation |
| UK (OFSI) | UK Sanctions List | Daily | UK Autonomous Sanctions |
| Australia (DFAT) | Consolidated List | Weekly | Australian Sanctions |
| Canada (OSFI) | Consolidated List | Weekly | Canadian Sanctions |
| Singapore (MAS) | Lists of Designated Individuals | Weekly | Singapore Sanctions |
| Hong Kong (HKMA) | Designated List | Weekly | HK Sanctions |
| Japan (METI) | End User List | Monthly | Export Controls |
| Switzerland (SECO) | SECO Sanctions List | Weekly | Swiss Sanctions |
| France (DGT) | National Freezing List | Weekly | French Sanctions |
| World-Check | Refinitiv World-Check | Daily | Comprehensive Global |
| Dow Jones | Risk & Compliance | Daily | Multi-Source Aggregated |

### Performance Metrics

| Metric | Target | Description |
|--------|--------|-------------|
| Single-Party Screening (p99) | < 80ms | Individual name screening time |
| Transaction Screening (p99) | < 150ms | Full transaction with all parties |
| Batch Screening Throughput | > 100,000/hour | Bulk screening capacity |
| List Load Time | < 30 seconds | Full list refresh time |
| Hit Match Latency | < 20ms | Single match comparison time |
| Decision Latency | < 10ms | Post-match decision time |
| Throughput | > 8,000 TPS | Screening checks per second |
| List Update Propagation | < 2 minutes | Time to activate new list entries |
| False Positive Rate | < 5% | Target false positive percentage |
| True Positive Detection | 100% | Sanctioned entity detection rate (zero tolerance) |

### Regulatory Compliance Coverage

| Jurisdiction | Regulation/Authority | Requirements | Coverage |
|--------------|---------------------|--------------|----------|
| USA | OFAC Sanctions Programs | SDN, SSI, FSE Screening | Full |
| USA | IEEPA (International Emergency Economic Powers Act) | Sanctions Compliance | Full |
| USA | Bank Secrecy Act | Sanctions Record Keeping | Full |
| EU | EU Sanctions Regulation | Restrictive Measures | Full |
| UK | Sanctions and Anti-Money Laundering Act 2018 | OFSI Compliance | Full |
| UN | UN Security Council Resolutions | UNSCR Compliance | Full |
| Singapore | MAS Notice 626 | Sanctions Screening | Full |
| Hong Kong | United Nations Sanctions Ordinance | Sanctions Compliance | Full |
| Australia | Autonomous Sanctions Act 2011 | DFAT Compliance | Full |
| Canada | Special Economic Measures Act | OSFI Compliance | Full |
| Multi-Jurisdiction | FATF Recommendations | Targeted Financial Sanctions | Full |

---

## 2. Module Overview

### 2.1 Bounded Context

The Sanctions Check module belongs to the **Risk & Compliance Domain** bounded context and operates as a dedicated **Sanctions Screening Service** (Service Port: 8005).

### 2.2 Module Scope

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        SANCTIONS CHECK MODULE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  SCREENING ORCHESTRATOR                              │   │
│  │  • Request Handling  • Parallel Screening  • Result Aggregation     │   │
│  │  • Decision Making   • Alert Generation    • Response Formation     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│         ┌──────────────────────────┼──────────────────────────┐            │
│         │                          │                          │            │
│         ▼                          ▼                          ▼            │
│  ┌─────────────────┐  ┌────────────────────┐  ┌─────────────────────────┐ │
│  │  SANCTIONS      │  │  EMBARGO           │  │  ENTITY SCREENING       │ │
│  │  LIST SCREENING │  │  SCREENING         │  │                         │ │
│  │                 │  │                    │  │ • Corporations          │ │
│  │ • OFAC SDN      │  │ • Country Embargo  │  │ • Financial Institutions│ │
│  │ • EU Sanctions  │  │ • Region Embargo   │  │ • Government Entities   │ │
│  │ • UN UNSCR      │  │ • Sectoral Embargo │  │ • NGOs/Charities        │ │
│  │ • UK OFSI       │  │ • Trade Embargo    │  │ • Vessels/Aircraft      │ │
│  │ • Country Lists │  │ • Port Embargo     │  │ • Crypto Addresses      │ │
│  │                 │  │                    │  │                         │ │
│  │ Types:          │  │ Programs:          │  │ Identifiers:            │ │
│  │ • Individuals   │  │ • Crimea/Sevastopol│  │ • BIC/SWIFT Codes       │ │
│  │ • Entities      │  │ • North Korea      │  │ • LEI Identifiers       │ │
│  │ • Vessels       │  │ • Iran             │  │ • IMO Numbers           │ │
│  │ • Aircraft      │  │ • Syria            │  │ • National IDs          │ │
│  │ • Addresses     │  │ • Cuba             │  │ • Passport Numbers      │ │
│  └─────────────────┘  └────────────────────┘  └─────────────────────────┘ │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  NAME MATCHING ENGINE                                │   │
│  │                                                                      │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │   │
│  │  │ Exact Match     │  │ Fuzzy Match     │  │ Phonetic Match      │  │   │
│  │  │                 │  │                 │  │                     │  │   │
│  │  │ • Direct Lookup │  │ • Jaro-Winkler  │  │ • Soundex           │  │   │
│  │  │ • Normalized    │  │ • Levenshtein   │  │ • Metaphone         │  │   │
│  │  │ • Case-folded   │  │ • Jaccard       │  │ • Double Metaphone  │  │   │
│  │  │ • Whitespace    │  │ • N-Gram        │  │ • NYSIIS            │  │   │
│  │  │                 │  │ • Damerau-Lev   │  │ • Caverphone        │  │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
│  │                                                                      │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │   │
│  │  │ Transliteration │  │ Alias Expansion │  │ Token Analysis      │  │   │
│  │  │                 │  │                 │  │                     │  │   │
│  │  │ • Arabic→Latin  │  │ • AKA Matching  │  │ • Name Parsing      │  │   │
│  │  │ • Cyrillic→Latin│  │ • FKA Matching  │  │ • Word Reordering   │  │   │
│  │  │ • Chinese→Pinyin│  │ • DBA Matching  │  │ • Title Removal     │  │   │
│  │  │ • Japanese→Roma │  │ • Nickname Match│  │ • Honorific Strip   │  │   │
│  │  │ • Korean→Roman  │  │ • Abbreviations │  │ • Suffix Handling   │  │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
│  │                                                                      │   │
│  │  Configuration: Match Thresholds, Algorithm Weights, Script Handling │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  HIT DISPOSITION ENGINE                              │   │
│  │                                                                      │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │   │
│  │  │ Auto-Clearance  │  │ Review Queue    │  │ Escalation Manager  │  │   │
│  │  │                 │  │                 │  │                     │  │   │
│  │  │ • Whitelist     │  │ • L1 Analyst    │  │ • L2 Escalation     │  │   │
│  │  │ • Prior Clear   │  │ • L2 Senior     │  │ • Compliance Officer│  │   │
│  │  │ • Low Score     │  │ • Compliance    │  │ • Legal/External    │  │   │
│  │  │ • False Pos DB  │  │ • Investigation │  │ • Regulatory Report │  │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
│  │                                                                      │   │
│  │  Workflow: Auto-Clear → L1 Review → L2 Review → Compliance → Legal   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  SANCTIONS DECISION ENGINE                           │   │
│  │                                                                      │   │
│  │  Decision Matrix:                                                    │   │
│  │  ┌─────────────┬──────────────┬─────────────┬──────────────────┐    │   │
│  │  │ Match Score │ List Type    │ Entity Type │ Decision         │    │   │
│  │  ├─────────────┼──────────────┼─────────────┼──────────────────┤    │   │
│  │  │ 100%        │ Any          │ Any         │ BLOCK            │    │   │
│  │  │ 95-99%      │ SDN/UNSCR    │ Any         │ BLOCK            │    │   │
│  │  │ 90-99%      │ Other Lists  │ Individual  │ REVIEW_URGENT    │    │   │
│  │  │ 80-89%      │ Any          │ Any         │ REVIEW_STANDARD  │    │   │
│  │  │ 70-79%      │ Any          │ Any         │ REVIEW_LOW       │    │   │
│  │  │ < 70%       │ Cleared      │ Any         │ CLEAR            │    │   │
│  │  └─────────────┴──────────────┴─────────────┴──────────────────┘    │   │
│  │                                                                      │   │
│  │  Actions: CLEAR → Continue | REVIEW → Hold + Alert | BLOCK → Reject │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  LIST MANAGEMENT ENGINE                              │   │
│  │                                                                      │   │
│  │  • List Import (XML, CSV, JSON, API)     • Version Control          │   │
│  │  • Delta Updates (incremental sync)       • Rollback Capability     │   │
│  │  • List Indexing (Trie, Inverted Index)  • Search Optimization      │   │
│  │  • Schedule Management (auto-refresh)     • Manual Import Override  │   │
│  │  • List Validation (format, duplicates)  • Audit Logging            │   │
│  │  • Multi-Source Reconciliation           • Conflict Resolution      │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 In-Scope Capabilities

| Category | Capabilities |
|----------|-------------|
| **Transaction Screening** | Real-time screening of all payment parties (originator, beneficiary, intermediaries) |
| **Customer Onboarding** | KYC sanctions screening during customer registration |
| **Periodic Review** | Scheduled batch screening of existing customer base |
| **List Management** | Import, update, version control of sanctions lists |
| **Name Matching** | Multi-algorithm name matching with configurable thresholds |
| **Hit Disposition** | Workflow management for potential match review |
| **Decision Engine** | Automated clear/review/block decisions |
| **Embargo Screening** | Country, region, and port embargo detection |
| **Secondary Sanctions** | Indirect exposure detection through correspondent chains |
| **Vessel/Aircraft Screening** | Maritime and aviation sector specialized screening |
| **Crypto Screening** | Blockchain address and wallet screening |
| **Regulatory Reporting** | OFAC, OFSI, and jurisdiction-specific reporting |
| **Audit & Compliance** | Complete screening history and examination support |

### 2.4 Out-of-Scope

| Item | Rationale |
|------|-----------|
| PEP Screening | Handled by AML Screening module |
| Adverse Media Screening | Handled by AML Screening module |
| Fraud Detection | Handled by Fraud Detection module |
| Transaction Monitoring | Handled by AML Screening module |
| Customer Risk Rating | Handled by AML Screening module (risk scoring component) |
| SAR Filing | Handled by AML Screening module (regulatory reporting) |

### 2.5 Module Dependencies

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       SANCTIONS CHECK MODULE                                 │
│                         (Service Port: 8005)                                │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
         ┌──────────────────────────┼──────────────────────────┐
         │                          │                          │
         ▼                          ▼                          ▼
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────────────┐
│ CONFIGURATION   │      │ PLATFORM        │      │ EXTERNAL SERVICES       │
│ DOMAIN          │      │ SERVICES        │      │                         │
│                 │      │                 │      │ • OFAC API              │
│ • Screening     │      │ • Cache Service │      │ • EU Sanctions API      │
│   Rules Config  │      │ • Event Bus     │      │ • World-Check API       │
│ • List Sources  │      │ • Audit Logger  │      │ • Dow Jones API         │
│ • Match Config  │      │ • Secret Vault  │      │ • UN List Download      │
│ • Threshold     │      │ • Notification  │      │ • Vessel Tracking       │
│   Settings      │      │   Service       │      │ • Case Management       │
└─────────────────┘      └─────────────────┘      └─────────────────────────┘
         │                          │                          │
         └──────────────────────────┼──────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       PAYMENT ORCHESTRATION DOMAIN                           │
│                                                                             │
│   Payment Gateway Core ← Sanctions Results                                  │
│   Validation Pipeline  ← Screening Decisions                                │
│   Transaction Context  ← Sanctions Metadata                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Functional Requirements Summary

### 3.1 Core Functional Requirements

| Req ID | Requirement | Priority | Description |
|--------|-------------|----------|-------------|
| SCR-001 | Real-Time Transaction Screening | Critical | Screen all payment transactions against sanctions lists in real-time |
| SCR-002 | Multi-Party Screening | Critical | Screen all parties: originator, beneficiary, ordering party, correspondent banks |
| SCR-003 | Multi-List Support | Critical | Support screening against 30+ sanctions lists simultaneously |
| SCR-004 | Name Matching Engine | Critical | Implement fuzzy, phonetic, and exact matching algorithms |
| SCR-005 | Hit Scoring | Critical | Calculate match confidence scores for potential hits |
| SCR-006 | Auto-Clearance | High | Automatically clear low-score matches and known false positives |
| SCR-007 | Review Workflow | High | Route uncertain matches to analyst review queues |
| SCR-008 | Block on True Match | Critical | Immediately block transactions with confirmed sanctions matches |
| SCR-009 | List Management | Critical | Import, update, and version control sanctions lists |
| SCR-010 | Embargo Screening | Critical | Detect embargoed countries, regions, and ports |

### 3.2 Screening Functional Requirements

| Req ID | Requirement | Priority | Description |
|--------|-------------|----------|-------------|
| SCR-011 | Individual Screening | Critical | Screen natural persons against SDN and other individual lists |
| SCR-012 | Entity Screening | Critical | Screen corporations, institutions against entity lists |
| SCR-013 | Vessel Screening | High | Screen vessel names and IMO numbers against vessel lists |
| SCR-014 | Aircraft Screening | High | Screen aircraft tail numbers against aviation sanctions |
| SCR-015 | BIC/SWIFT Screening | Critical | Screen bank identifiers against financial institution lists |
| SCR-016 | Address Screening | High | Screen addresses against sanctioned locations |
| SCR-017 | ID Document Screening | High | Screen passport, national ID against known identifiers |
| SCR-018 | Crypto Address Screening | Medium | Screen blockchain addresses against OFAC crypto list |

### 3.3 Name Matching Requirements

| Req ID | Requirement | Priority | Description |
|--------|-------------|----------|-------------|
| SCR-019 | Exact Matching | Critical | 100% character-for-character matching |
| SCR-020 | Fuzzy Matching | Critical | Support Jaro-Winkler, Levenshtein, Jaccard algorithms |
| SCR-021 | Phonetic Matching | Critical | Support Soundex, Metaphone, Double Metaphone |
| SCR-022 | Transliteration | High | Convert non-Latin scripts to Latin equivalents |
| SCR-023 | Alias Expansion | High | Match against AKA, FKA, DBA aliases |
| SCR-024 | Name Normalization | Critical | Standardize names (remove titles, honorifics, punctuation) |
| SCR-025 | Token Reordering | High | Match names regardless of word order |
| SCR-026 | Configurable Thresholds | Critical | Allow jurisdiction-specific match thresholds |

### 3.4 List Management Requirements

| Req ID | Requirement | Priority | Description |
|--------|-------------|----------|-------------|
| SCR-027 | List Import | Critical | Support XML, CSV, JSON, API list imports |
| SCR-028 | Incremental Updates | Critical | Support delta updates without full list reload |
| SCR-029 | List Versioning | Critical | Maintain version history for audit and rollback |
| SCR-030 | Scheduled Refresh | High | Automatic list updates on configurable schedule |
| SCR-031 | Manual Override | High | Allow manual list imports outside schedule |
| SCR-032 | List Validation | High | Validate list format, detect duplicates, log anomalies |
| SCR-033 | Multi-Source Reconciliation | Medium | Merge entries from multiple list providers |
| SCR-034 | List Activation Control | High | Control when new list versions become active |

### 3.5 Decision & Workflow Requirements

| Req ID | Requirement | Priority | Description |
|--------|-------------|----------|-------------|
| SCR-035 | Auto-Clear Rules | High | Configure rules for automatic false positive clearing |
| SCR-036 | Review Queue Management | High | Route hits to appropriate analyst queues by severity |
| SCR-037 | Escalation Workflow | High | Support multi-level escalation (L1→L2→Compliance→Legal) |
| SCR-038 | Decision Recording | Critical | Record all decisions with analyst ID, timestamp, rationale |
| SCR-039 | Bulk Disposition | Medium | Allow bulk clearing of similar false positives |
| SCR-040 | SLA Management | High | Track and alert on review SLA breaches |
| SCR-041 | Four-Eyes Principle | High | Require dual approval for sensitive decisions |

### 3.6 Reporting & Audit Requirements

| Req ID | Requirement | Priority | Description |
|--------|-------------|----------|-------------|
| SCR-042 | Screening Audit Log | Critical | Log all screening requests, results, decisions |
| SCR-043 | Regulatory Reports | Critical | Generate OFAC, OFSI jurisdiction-specific reports |
| SCR-044 | Hit Statistics | High | Dashboard for hit rates, false positives, review times |
| SCR-045 | List Update Reports | High | Document all list changes for examination |
| SCR-046 | Analyst Performance | Medium | Track analyst decision accuracy and throughput |
| SCR-047 | Examination Support | Critical | Export data packages for regulatory examinations |
| SCR-048 | Record Retention | Critical | Retain screening records per regulatory requirements (5-7 years) |

---

## 4. Domain Model

### 4.1 Core Domain Entities

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                          SANCTIONS CHECK DOMAIN MODEL                           │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                        AGGREGATE ROOTS                                   │  │
│  │                                                                          │  │
│  │  ┌──────────────────────┐  ┌──────────────────────┐                     │  │
│  │  │   ScreeningRequest   │  │   SanctionsList      │                     │  │
│  │  │   <<Aggregate Root>> │  │   <<Aggregate Root>> │                     │  │
│  │  │                      │  │                      │                     │  │
│  │  │ • requestId: UUID    │  │ • listId: UUID       │                     │  │
│  │  │ • transactionId: Str │  │ • listCode: String   │                     │  │
│  │  │ • screeningType: Enum│  │ • listName: String   │                     │  │
│  │  │ • requestSource: Str │  │ • authority: String  │                     │  │
│  │  │ • priority: Enum     │  │ • jurisdiction: Str  │                     │  │
│  │  │ • parties: List      │  │ • version: String    │                     │  │
│  │  │ • status: Enum       │  │ • effectiveDate: Date│                     │  │
│  │  │ • createdAt: DateTime│  │ • entryCount: Int    │                     │  │
│  │  │ • completedAt: Date  │  │ • status: Enum       │                     │  │
│  │  └──────────┬───────────┘  └──────────┬───────────┘                     │  │
│  │             │                         │                                  │  │
│  │             │ contains                │ contains                         │  │
│  │             ▼                         ▼                                  │  │
│  │  ┌──────────────────────┐  ┌──────────────────────┐                     │  │
│  │  │   ScreeningParty     │  │   SanctionsEntry     │                     │  │
│  │  │   <<Entity>>         │  │   <<Entity>>         │                     │  │
│  │  │                      │  │                      │                     │  │
│  │  │ • partyId: UUID      │  │ • entryId: UUID      │                     │  │
│  │  │ • partyType: Enum    │  │ • entryType: Enum    │                     │  │
│  │  │ • partyRole: Enum    │  │ • primaryName: String│                     │  │
│  │  │ • name: String       │  │ • aliases: List      │                     │  │
│  │  │ • normalizedName: Str│  │ • nationality: List  │                     │  │
│  │  │ • identifiers: List  │  │ • identifiers: List  │                     │  │
│  │  │ • address: Address   │  │ • addresses: List    │                     │  │
│  │  │ • country: String    │  │ • programs: List     │                     │  │
│  │  │ • dateOfBirth: Date  │  │ • remarks: String    │                     │  │
│  │  │ • nationality: String│  │ • listingDate: Date  │                     │  │
│  │  │ • bicCode: String    │  │ • delistingDate: Date│                     │  │
│  │  └──────────────────────┘  └──────────────────────┘                     │  │
│  │                                                                          │  │
│  │  ┌──────────────────────┐  ┌──────────────────────┐                     │  │
│  │  │   ScreeningResult    │  │   SanctionsHit       │                     │  │
│  │  │   <<Aggregate Root>> │  │   <<Aggregate Root>> │                     │  │
│  │  │                      │  │                      │                     │  │
│  │  │ • resultId: UUID     │  │ • hitId: UUID        │                     │  │
│  │  │ • requestId: UUID    │  │ • resultId: UUID     │                     │  │
│  │  │ • overallDecision:Enum│ │ • partyId: UUID      │                     │  │
│  │  │ • hitCount: Integer  │  │ • entryId: UUID      │                     │  │
│  │  │ • clearCount: Integer│  │ • matchScore: Decimal│                     │  │
│  │  │ • reviewCount: Integer│ │ • matchType: Enum    │                     │  │
│  │  │ • processingTime: ms │  │ • matchDetails: JSON │                     │  │
│  │  │ • screenedAt: DateTime│ │ • status: Enum       │                     │  │
│  │  │ • riskScore: Decimal │  │ • disposition: Enum  │                     │  │
│  │  │ • auditTrail: List   │  │ • reviewedBy: String │                     │  │
│  │  └──────────────────────┘  │ • reviewedAt: DateTime│                    │  │
│  │                            │ • comments: String   │                     │  │
│  │                            └──────────────────────┘                     │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                        VALUE OBJECTS                                     │  │
│  │                                                                          │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐  │  │
│  │  │ MatchScore      │  │ Identifier      │  │ SanctionsProgram        │  │  │
│  │  │ <<Value Object>>│  │ <<Value Object>>│  │ <<Value Object>>        │  │  │
│  │  │                 │  │                 │  │                         │  │  │
│  │  │ • score: Decimal│  │ • type: Enum    │  │ • programCode: String   │  │  │
│  │  │ • algorithm: Str│  │ • value: String │  │ • programName: String   │  │  │
│  │  │ • confidence:Str│  │ • country: Str  │  │ • authority: String     │  │  │
│  │  │ • matchedOn: Str│  │ • issuedDate:Dt │  │ • description: String   │  │  │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────────┘  │  │
│  │                                                                          │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐  │  │
│  │  │ Address         │  │ Alias           │  │ EmbargoDetails          │  │  │
│  │  │ <<Value Object>>│  │ <<Value Object>>│  │ <<Value Object>>        │  │  │
│  │  │                 │  │                 │  │                         │  │  │
│  │  │ • line1: String │  │ • aliasType:Enum│  │ • countryCode: String   │  │  │
│  │  │ • line2: String │  │ • aliasName: Str│  │ • embargoType: Enum     │  │  │
│  │  │ • city: String  │  │ • quality: Enum │  │ • restrictions: List    │  │  │
│  │  │ • region: String│  │ • script: String│  │ • effectiveDate: Date   │  │  │
│  │  │ • postalCode:Str│  │ • language: Str │  │ • exceptions: List      │  │  │
│  │  │ • country: Str  │  │                 │  │ • licenseRequired: Bool │  │  │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────────┘  │  │
│  │                                                                          │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                        ENUMERATIONS                                      │  │
│  │                                                                          │  │
│  │  ScreeningType:        PartyType:           PartyRole:                  │  │
│  │  • TRANSACTION         • INDIVIDUAL         • ORIGINATOR                │  │
│  │  • CUSTOMER_ONBOARD    • ENTITY             • BENEFICIARY               │  │
│  │  • PERIODIC_REVIEW     • VESSEL             • ORDERING_PARTY            │  │
│  │  • ADHOC               • AIRCRAFT           • CORRESPONDENT_BANK        │  │
│  │  • BATCH               • FINANCIAL_INST     • INTERMEDIARY_BANK         │  │
│  │                        • GOVERNMENT         • ULTIMATE_BENEFICIARY      │  │
│  │                                                                          │  │
│  │  ScreeningDecision:    HitStatus:           MatchType:                  │  │
│  │  • CLEAR               • PENDING            • EXACT                     │  │
│  │  • REVIEW_LOW          • UNDER_REVIEW       • FUZZY_HIGH                │  │
│  │  • REVIEW_STANDARD     • ESCALATED          • FUZZY_MEDIUM              │  │
│  │  • REVIEW_URGENT       • CLEARED            • PHONETIC                  │  │
│  │  • BLOCK               • CONFIRMED_MATCH    • TRANSLITERATED            │  │
│  │                        • TRUE_POSITIVE      • ALIAS                     │  │
│  │                        • FALSE_POSITIVE     • PARTIAL                   │  │
│  │                                                                          │  │
│  │  ListStatus:           EmbargoType:         IdentifierType:             │  │
│  │  • DRAFT               • COMPREHENSIVE      • PASSPORT                  │  │
│  │  • ACTIVE              • SECTORAL           • NATIONAL_ID               │  │
│  │  • SUPERSEDED          • TARGETED           • TAX_ID                    │  │
│  │  • ARCHIVED            • TRADE              • BIC_CODE                  │  │
│  │  • PENDING_ACTIVATION  • FINANCIAL          • LEI                       │  │
│  │                        • TRAVEL             • IMO_NUMBER                │  │
│  │                                             • CRYPTO_ADDRESS            │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Entity Relationship Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           ENTITY RELATIONSHIPS                                       │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌──────────────────────┐        1:N        ┌──────────────────────┐               │
│  │   ScreeningRequest   │──────────────────▶│   ScreeningParty     │               │
│  │                      │                   │                      │               │
│  │   (Transaction,      │                   │   (Originator,       │               │
│  │    Customer, Batch)  │                   │    Beneficiary, etc) │               │
│  └──────────┬───────────┘                   └──────────┬───────────┘               │
│             │                                          │                            │
│             │ 1:1                                       │ 1:N                        │
│             ▼                                          ▼                            │
│  ┌──────────────────────┐        1:N        ┌──────────────────────┐               │
│  │   ScreeningResult    │──────────────────▶│    SanctionsHit      │               │
│  │                      │                   │                      │               │
│  │   (Overall           │                   │   (Individual        │               │
│  │    Decision)         │                   │    Match Result)     │               │
│  └──────────────────────┘                   └──────────┬───────────┘               │
│                                                        │                            │
│                                                        │ N:1                        │
│                                                        ▼                            │
│  ┌──────────────────────┐        1:N        ┌──────────────────────┐               │
│  │   SanctionsList      │──────────────────▶│   SanctionsEntry     │               │
│  │                      │                   │                      │               │
│  │   (OFAC, EU,         │                   │   (SDN Entry,        │               │
│  │    UN, UK, etc)      │                   │    Entity Record)    │               │
│  └──────────────────────┘                   └──────────────────────┘               │
│                                                                                     │
│  ┌──────────────────────┐        N:N        ┌──────────────────────┐               │
│  │   SanctionsEntry     │◀─────────────────▶│   SanctionsProgram   │               │
│  │                      │                   │                      │               │
│  │   (Entry can be in   │                   │   (SDGT, IRAN,       │               │
│  │    multiple programs)│                   │    UKRAINE-EO, etc)  │               │
│  └──────────────────────┘                   └──────────────────────┘               │
│                                                                                     │
│  ┌──────────────────────┐        1:N        ┌──────────────────────┐               │
│  │   HitDisposition     │──────────────────▶│   DispositionHistory │               │
│  │                      │                   │                      │               │
│  │   (Current           │                   │   (Audit Trail       │               │
│  │    Disposition)      │                   │    of Changes)       │               │
│  └──────────────────────┘                   └──────────────────────┘               │
│                                                                                     │
│  ┌──────────────────────┐        1:1        ┌──────────────────────┐               │
│  │   ScreeningParty     │──────────────────▶│   EmbargoCheck       │               │
│  │                      │                   │                      │               │
│  │   (Country field     │                   │   (Country Embargo   │               │
│  │    triggers check)   │                   │    Result)           │               │
│  └──────────────────────┘                   └──────────────────────┘               │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 State Machine Diagrams

#### 4.3.1 Screening Request State Machine

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                      SCREENING REQUEST STATE MACHINE                                 │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│                              ┌──────────────┐                                       │
│                              │   RECEIVED   │                                       │
│                              │              │                                       │
│                              └──────┬───────┘                                       │
│                                     │ validate()                                    │
│                    ┌────────────────┴────────────────┐                             │
│                    ▼                                 ▼                             │
│           ┌──────────────┐                  ┌──────────────┐                       │
│           │    VALID     │                  │   REJECTED   │                       │
│           │              │                  │  (Invalid)   │                       │
│           └──────┬───────┘                  └──────────────┘                       │
│                  │ startScreening()                                                 │
│                  ▼                                                                  │
│           ┌──────────────┐                                                          │
│           │  SCREENING   │                                                          │
│           │              │◀─────────┐                                              │
│           └──────┬───────┘          │ retryScreening()                             │
│                  │                  │                                               │
│    ┌─────────────┴─────────────┐    │                                               │
│    ▼             │             ▼    │                                               │
│ ┌─────────┐  ┌───┴───────┐  ┌──────┴─────┐                                         │
│ │ CLEAR   │  │   HITS    │  │   ERROR    │                                         │
│ │         │  │  (Found)  │  │            │                                         │
│ └────┬────┘  └─────┬─────┘  └────────────┘                                         │
│      │             │                                                                │
│      │             │ evaluateHits()                                                 │
│      │             ▼                                                                │
│      │    ┌──────────────────────────────────────────┐                             │
│      │    │              HIT EVALUATION              │                             │
│      │    │                                          │                             │
│      │    │  ┌─────────┐ ┌─────────┐ ┌────────────┐ │                             │
│      │    │  │AUTO_CLR │ │ REVIEW  │ │   BLOCK    │ │                             │
│      │    │  └────┬────┘ └────┬────┘ └─────┬──────┘ │                             │
│      │    └───────┼───────────┼────────────┼────────┘                             │
│      │            │           │            │                                        │
│      │◀───────────┘           │            │                                        │
│      │                        │            │                                        │
│      │    ┌───────────────────┘            │                                        │
│      │    ▼                                ▼                                        │
│      │  ┌──────────────┐            ┌──────────────┐                               │
│      │  │ PENDING      │            │   BLOCKED    │                               │
│      │  │ REVIEW       │            │              │                               │
│      │  └──────┬───────┘            └──────────────┘                               │
│      │         │                                                                    │
│      │         │ analyst decision                                                   │
│      │         ▼                                                                    │
│      │  ┌────────────────────────────────────┐                                     │
│      │  │          REVIEW WORKFLOW           │                                     │
│      │  │                                    │                                     │
│      │  │  ┌────────┐  ┌────────┐  ┌──────┐ │                                     │
│      │  │  │L1 Review│→│L2 Review│→│Compl.│ │                                     │
│      │  │  └───┬────┘  └────┬───┘  └──┬───┘ │                                     │
│      │  └──────┼────────────┼─────────┼─────┘                                     │
│      │         │            │         │                                            │
│      │         ▼            ▼         ▼                                            │
│      │  ┌─────────────────────────────────────┐                                    │
│      │  │  FALSE_POSITIVE  │  TRUE_POSITIVE   │                                    │
│      │  └────────┬─────────┴────────┬─────────┘                                    │
│      │           │                  │                                               │
│      ▼           ▼                  ▼                                               │
│   ┌──────────────────┐       ┌──────────────┐                                      │
│   │    COMPLETED     │       │   BLOCKED    │                                      │
│   │    (CLEAR)       │       │ (Confirmed)  │                                      │
│   └──────────────────┘       └──────────────┘                                      │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

#### 4.3.2 Sanctions Hit State Machine

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        SANCTIONS HIT STATE MACHINE                                   │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│                              ┌──────────────┐                                       │
│                              │   DETECTED   │                                       │
│                              │              │                                       │
│                              └──────┬───────┘                                       │
│                                     │ evaluate()                                    │
│                    ┌────────────────┼────────────────┐                             │
│                    ▼                ▼                ▼                             │
│           ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                       │
│           │ AUTO_CLEARED │ │   PENDING    │ │ AUTO_BLOCKED │                       │
│           │ (Score < 70) │ │   REVIEW     │ │ (Score=100)  │                       │
│           └──────────────┘ │ (70-99%)     │ └──────┬───────┘                       │
│                            └──────┬───────┘        │                               │
│                                   │                │                               │
│                                   │ assignToQueue()│                               │
│                                   ▼                │                               │
│                           ┌──────────────┐         │                               │
│                           │   QUEUED     │         │                               │
│                           │  (L1 Queue)  │         │                               │
│                           └──────┬───────┘         │                               │
│                                  │                 │                               │
│                    ┌─────────────┴─────────────┐   │                               │
│                    ▼             │             ▼   │                               │
│           ┌──────────────┐       │    ┌──────────────┐                             │
│           │   CLAIMED    │       │    │  ESCALATED   │                             │
│           │ (by L1)      │       │    │  (to L2)     │                             │
│           └──────┬───────┘       │    └──────┬───────┘                             │
│                  │               │           │                                      │
│                  │ review()      │           │ review()                             │
│                  ▼               │           ▼                                      │
│           ┌──────────────┐       │    ┌──────────────┐                             │
│           │ L1_REVIEWED  │       │    │ L2_REVIEWED  │                             │
│           │              │       │    │              │                             │
│           └──────┬───────┘       │    └──────┬───────┘                             │
│                  │               │           │                                      │
│    ┌─────────────┴─────────────┐ │ ┌─────────┴─────────────┐                       │
│    ▼                           ▼ │ ▼                       ▼                       │
│ ┌─────────┐           ┌─────────┐│┌─────────┐      ┌────────────┐                  │
│ │ CLEARED │           │ESCALATE ││ │ CLEARED │      │   MATCH    │                  │
│ │(False +)│           │ TO L2   ││ │(False +)│      │ CONFIRMED  │                  │
│ └────┬────┘           └────┬────┘│ └────┬────┘      └─────┬──────┘                  │
│      │                     │     │      │                 │                         │
│      │                     └─────┘      │                 │                         │
│      │                                  │                 │                         │
│      ▼                                  ▼                 ▼                         │
│ ┌──────────────┐              ┌──────────────┐    ┌──────────────┐                  │
│ │FALSE_POSITIVE│              │FALSE_POSITIVE│    │TRUE_POSITIVE │◀─────────────────┘
│ │   (Final)    │              │   (Final)    │    │   (Final)    │                  │
│ └──────────────┘              └──────────────┘    └──────────────┘                  │
│                                                                                     │
│  Terminal States: FALSE_POSITIVE, TRUE_POSITIVE, AUTO_CLEARED, AUTO_BLOCKED        │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Data Architecture

### 5.1 Database Schema Design

```sql
-- ============================================================================
-- SANCTIONS CHECK MODULE - DATABASE SCHEMA
-- Database: PostgreSQL 15+ with TimescaleDB extension for time-series data
-- ============================================================================

-- ----------------------------------------------------------------------------
-- SCHEMA: sanctions
-- ----------------------------------------------------------------------------
CREATE SCHEMA IF NOT EXISTS sanctions;

-- ----------------------------------------------------------------------------
-- TABLE: sanctions_lists
-- Purpose: Master list of all sanctions list sources
-- ----------------------------------------------------------------------------
CREATE TABLE sanctions.sanctions_lists (
    list_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    list_code           VARCHAR(50) NOT NULL UNIQUE,
    list_name           VARCHAR(200) NOT NULL,
    authority           VARCHAR(100) NOT NULL,
    jurisdiction        VARCHAR(50) NOT NULL,
    list_type           VARCHAR(50) NOT NULL,
    source_url          VARCHAR(500),
    source_format       VARCHAR(20) NOT NULL,  -- XML, CSV, JSON, API
    update_frequency    VARCHAR(20) NOT NULL,  -- DAILY, WEEKLY, REALTIME
    is_active           BOOLEAN DEFAULT TRUE,
    priority_rank       INTEGER NOT NULL DEFAULT 100,
    created_at          TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_list_type CHECK (list_type IN ('SDN', 'CONSOLIDATED', 'SECTORAL', 'EMBARGO', 'PEP', 'CUSTOM')),
    CONSTRAINT chk_source_format CHECK (source_format IN ('XML', 'CSV', 'JSON', 'API'))
);

CREATE INDEX idx_lists_code ON sanctions.sanctions_lists(list_code);
CREATE INDEX idx_lists_active ON sanctions.sanctions_lists(is_active) WHERE is_active = TRUE;

-- ----------------------------------------------------------------------------
-- TABLE: list_versions
-- Purpose: Version history for each sanctions list
-- ----------------------------------------------------------------------------
CREATE TABLE sanctions.list_versions (
    version_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    list_id             UUID NOT NULL REFERENCES sanctions.sanctions_lists(list_id),
    version_number      VARCHAR(50) NOT NULL,
    published_date      DATE NOT NULL,
    effective_date      TIMESTAMP WITH TIME ZONE NOT NULL,
    entry_count         INTEGER NOT NULL,
    additions_count     INTEGER DEFAULT 0,
    modifications_count INTEGER DEFAULT 0,
    deletions_count     INTEGER DEFAULT 0,
    status              VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    imported_at         TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    activated_at        TIMESTAMP WITH TIME ZONE,
    imported_by         VARCHAR(100),
    checksum            VARCHAR(64),  -- SHA-256 hash
    source_file_path    VARCHAR(500),
    
    CONSTRAINT chk_version_status CHECK (status IN ('PENDING', 'ACTIVE', 'SUPERSEDED', 'ROLLED_BACK', 'FAILED')),
    CONSTRAINT uq_list_version UNIQUE (list_id, version_number)
);

CREATE INDEX idx_versions_list ON sanctions.list_versions(list_id);
CREATE INDEX idx_versions_status ON sanctions.list_versions(status);
CREATE INDEX idx_versions_effective ON sanctions.list_versions(effective_date DESC);

-- ----------------------------------------------------------------------------
-- TABLE: sanctions_entries
-- Purpose: Individual sanctions list entries (persons, entities, vessels, etc.)
-- ----------------------------------------------------------------------------
CREATE TABLE sanctions.sanctions_entries (
    entry_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id          UUID NOT NULL REFERENCES sanctions.list_versions(version_id),
    list_id             UUID NOT NULL REFERENCES sanctions.sanctions_lists(list_id),
    external_id         VARCHAR(100),  -- Original ID from source list
    entry_type          VARCHAR(30) NOT NULL,
    primary_name        VARCHAR(500) NOT NULL,
    normalized_name     VARCHAR(500) NOT NULL,
    name_soundex        VARCHAR(20),
    name_metaphone      VARCHAR(50),
    title               VARCHAR(200),
    gender              VARCHAR(10),
    date_of_birth       DATE,
    place_of_birth      VARCHAR(200),
    nationality         VARCHAR(100)[],
    citizenship         VARCHAR(100)[],
    programs            VARCHAR(100)[] NOT NULL,
    remarks             TEXT,
    listing_date        DATE,
    delisting_date      DATE,
    last_updated        DATE,
    is_active           BOOLEAN DEFAULT TRUE,
    search_tokens       TSVECTOR,  -- Full-text search
    created_at          TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_entry_type CHECK (entry_type IN ('INDIVIDUAL', 'ENTITY', 'VESSEL', 'AIRCRAFT', 'PROPERTY', 'CRYPTO_ADDRESS'))
);

CREATE INDEX idx_entries_list ON sanctions.sanctions_entries(list_id);
CREATE INDEX idx_entries_version ON sanctions.sanctions_entries(version_id);
CREATE INDEX idx_entries_type ON sanctions.sanctions_entries(entry_type);
CREATE INDEX idx_entries_active ON sanctions.sanctions_entries(is_active) WHERE is_active = TRUE;
CREATE INDEX idx_entries_normalized ON sanctions.sanctions_entries(normalized_name);
CREATE INDEX idx_entries_soundex ON sanctions.sanctions_entries(name_soundex);
CREATE INDEX idx_entries_metaphone ON sanctions.sanctions_entries(name_metaphone);
CREATE INDEX idx_entries_search ON sanctions.sanctions_entries USING GIN(search_tokens);
CREATE INDEX idx_entries_programs ON sanctions.sanctions_entries USING GIN(programs);

-- ----------------------------------------------------------------------------
-- TABLE: entry_aliases
-- Purpose: Alternative names/aliases for sanctions entries
-- ----------------------------------------------------------------------------
CREATE TABLE sanctions.entry_aliases (
    alias_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entry_id            UUID NOT NULL REFERENCES sanctions.sanctions_entries(entry_id),
    alias_type          VARCHAR(20) NOT NULL,  -- AKA, FKA, DBA, NICK
    alias_name          VARCHAR(500) NOT NULL,
    normalized_name     VARCHAR(500) NOT NULL,
    alias_soundex       VARCHAR(20),
    alias_metaphone     VARCHAR(50),
    quality             VARCHAR(20) DEFAULT 'GOOD',
    script              VARCHAR(20),  -- LATIN, ARABIC, CYRILLIC, etc.
    language            VARCHAR(10),
    is_primary          BOOLEAN DEFAULT FALSE,
    
    CONSTRAINT chk_alias_type CHECK (alias_type IN ('AKA', 'FKA', 'DBA', 'NICKNAME', 'SPELLING_VARIANT', 'TRANSLITERATION')),
    CONSTRAINT chk_alias_quality CHECK (quality IN ('GOOD', 'LOW', 'WEAK'))
);

CREATE INDEX idx_aliases_entry ON sanctions.entry_aliases(entry_id);
CREATE INDEX idx_aliases_normalized ON sanctions.entry_aliases(normalized_name);
CREATE INDEX idx_aliases_soundex ON sanctions.entry_aliases(alias_soundex);
CREATE INDEX idx_aliases_metaphone ON sanctions.entry_aliases(alias_metaphone);

-- ----------------------------------------------------------------------------
-- TABLE: entry_identifiers
-- Purpose: Identity documents and identifiers for sanctions entries
-- ----------------------------------------------------------------------------
CREATE TABLE sanctions.entry_identifiers (
    identifier_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entry_id            UUID NOT NULL REFERENCES sanctions.sanctions_entries(entry_id),
    identifier_type     VARCHAR(30) NOT NULL,
    identifier_value    VARCHAR(200) NOT NULL,
    issuing_country     VARCHAR(50),
    issuing_authority   VARCHAR(200),
    issue_date          DATE,
    expiry_date         DATE,
    is_primary          BOOLEAN DEFAULT FALSE,
    notes               VARCHAR(500),
    
    CONSTRAINT chk_identifier_type CHECK (identifier_type IN (
        'PASSPORT', 'NATIONAL_ID', 'TAX_ID', 'SSN', 'DRIVER_LICENSE',
        'BIC_CODE', 'LEI', 'IMO_NUMBER', 'MMSI', 'CALL_SIGN',
        'AIRCRAFT_TAIL', 'CRYPTO_ADDRESS', 'EMAIL', 'PHONE', 'WEBSITE'
    ))
);

CREATE INDEX idx_identifiers_entry ON sanctions.entry_identifiers(entry_id);
CREATE INDEX idx_identifiers_type_value ON sanctions.entry_identifiers(identifier_type, identifier_value);
CREATE INDEX idx_identifiers_value ON sanctions.entry_identifiers(identifier_value);

-- ----------------------------------------------------------------------------
-- TABLE: entry_addresses
-- Purpose: Addresses associated with sanctions entries
-- ----------------------------------------------------------------------------
CREATE TABLE sanctions.entry_addresses (
    address_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entry_id            UUID NOT NULL REFERENCES sanctions.sanctions_entries(entry_id),
    address_type        VARCHAR(20) NOT NULL DEFAULT 'PRIMARY',
    address_line1       VARCHAR(300),
    address_line2       VARCHAR(300),
    city                VARCHAR(100),
    state_province      VARCHAR(100),
    postal_code         VARCHAR(20),
    country_code        CHAR(2),
    country_name        VARCHAR(100),
    full_address        TEXT,
    normalized_address  TEXT,
    is_primary          BOOLEAN DEFAULT FALSE,
    
    CONSTRAINT chk_address_type CHECK (address_type IN ('PRIMARY', 'REGISTERED', 'MAILING', 'RESIDENCE', 'BUSINESS'))
);

CREATE INDEX idx_addresses_entry ON sanctions.entry_addresses(entry_id);
CREATE INDEX idx_addresses_country ON sanctions.entry_addresses(country_code);

-- ----------------------------------------------------------------------------
-- TABLE: embargo_countries
-- Purpose: Embargoed countries and regions
-- ----------------------------------------------------------------------------
CREATE TABLE sanctions.embargo_countries (
    embargo_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code        CHAR(2) NOT NULL,
    country_name        VARCHAR(100) NOT NULL,
    region_code         VARCHAR(10),
    region_name         VARCHAR(100),
    embargo_type        VARCHAR(30) NOT NULL,
    embargo_program     VARCHAR(100),
    authority           VARCHAR(100) NOT NULL,
    restrictions        VARCHAR(50)[] NOT NULL,
    effective_date      DATE NOT NULL,
    expiry_date         DATE,
    license_required    BOOLEAN DEFAULT TRUE,
    license_type        VARCHAR(50),
    exceptions          TEXT,
    is_active           BOOLEAN DEFAULT TRUE,
    severity            INTEGER DEFAULT 100,  -- 100 = full, lower = partial
    
    CONSTRAINT chk_embargo_type CHECK (embargo_type IN ('COMPREHENSIVE', 'SECTORAL', 'TARGETED', 'TRADE', 'FINANCIAL', 'TRAVEL', 'ARMS'))
);

CREATE INDEX idx_embargo_country ON sanctions.embargo_countries(country_code);
CREATE INDEX idx_embargo_active ON sanctions.embargo_countries(is_active) WHERE is_active = TRUE;
CREATE INDEX idx_embargo_type ON sanctions.embargo_countries(embargo_type);

-- ----------------------------------------------------------------------------
-- TABLE: screening_requests
-- Purpose: Incoming screening requests from payment system
-- ----------------------------------------------------------------------------
CREATE TABLE sanctions.screening_requests (
    request_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_id      VARCHAR(100),
    external_ref        VARCHAR(100),
    screening_type      VARCHAR(30) NOT NULL,
    request_source      VARCHAR(50) NOT NULL,
    channel             VARCHAR(30),
    priority            VARCHAR(20) NOT NULL DEFAULT 'NORMAL',
    status              VARCHAR(30) NOT NULL DEFAULT 'RECEIVED',
    party_count         INTEGER NOT NULL,
    requested_at        TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    started_at          TIMESTAMP WITH TIME ZONE,
    completed_at        TIMESTAMP WITH TIME ZONE,
    processing_time_ms  INTEGER,
    caller_system       VARCHAR(50),
    caller_user         VARCHAR(100),
    correlation_id      VARCHAR(100),
    idempotency_key     VARCHAR(100) UNIQUE,
    metadata            JSONB,
    
    CONSTRAINT chk_screening_type CHECK (screening_type IN ('TRANSACTION', 'CUSTOMER_ONBOARD', 'PERIODIC_REVIEW', 'ADHOC', 'BATCH')),
    CONSTRAINT chk_priority CHECK (priority IN ('LOW', 'NORMAL', 'HIGH', 'URGENT')),
    CONSTRAINT chk_status CHECK (status IN ('RECEIVED', 'VALIDATING', 'SCREENING', 'PENDING_REVIEW', 'COMPLETED', 'FAILED', 'CANCELLED'))
);

CREATE INDEX idx_requests_txn ON sanctions.screening_requests(transaction_id);
CREATE INDEX idx_requests_status ON sanctions.screening_requests(status);
CREATE INDEX idx_requests_type ON sanctions.screening_requests(screening_type);
CREATE INDEX idx_requests_date ON sanctions.screening_requests(requested_at DESC);
CREATE INDEX idx_requests_correlation ON sanctions.screening_requests(correlation_id);

-- ----------------------------------------------------------------------------
-- TABLE: screening_parties
-- Purpose: Parties being screened within a request
-- ----------------------------------------------------------------------------
CREATE TABLE sanctions.screening_parties (
    party_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id          UUID NOT NULL REFERENCES sanctions.screening_requests(request_id),
    party_type          VARCHAR(30) NOT NULL,
    party_role          VARCHAR(30) NOT NULL,
    party_sequence      INTEGER NOT NULL,
    full_name           VARCHAR(500) NOT NULL,
    normalized_name     VARCHAR(500) NOT NULL,
    name_soundex        VARCHAR(20),
    name_metaphone      VARCHAR(50),
    first_name          VARCHAR(200),
    last_name           VARCHAR(200),
    middle_name         VARCHAR(200),
    date_of_birth       DATE,
    place_of_birth      VARCHAR(200),
    nationality         VARCHAR(50),
    country_code        CHAR(2),
    address_line1       VARCHAR(300),
    address_line2       VARCHAR(300),
    city                VARCHAR(100),
    postal_code         VARCHAR(20),
    bic_code            VARCHAR(11),
    account_number      VARCHAR(50),
    identifiers         JSONB,  -- Array of identifier objects
    screening_status    VARCHAR(30) NOT NULL DEFAULT 'PENDING',
    hit_count           INTEGER DEFAULT 0,
    highest_score       DECIMAL(5,2),
    decision            VARCHAR(20),
    screened_at         TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT chk_party_type CHECK (party_type IN ('INDIVIDUAL', 'ENTITY', 'FINANCIAL_INSTITUTION', 'GOVERNMENT')),
    CONSTRAINT chk_party_role CHECK (party_role IN ('ORIGINATOR', 'BENEFICIARY', 'ORDERING_PARTY', 'CORRESPONDENT_BANK', 'INTERMEDIARY_BANK', 'ULTIMATE_BENEFICIARY', 'ULTIMATE_ORIGINATOR')),
    CONSTRAINT chk_party_decision CHECK (decision IN ('CLEAR', 'REVIEW', 'BLOCK'))
);

CREATE INDEX idx_parties_request ON sanctions.screening_parties(request_id);
CREATE INDEX idx_parties_name ON sanctions.screening_parties(normalized_name);
CREATE INDEX idx_parties_soundex ON sanctions.screening_parties(name_soundex);
CREATE INDEX idx_parties_bic ON sanctions.screening_parties(bic_code);
CREATE INDEX idx_parties_country ON sanctions.screening_parties(country_code);

-- ----------------------------------------------------------------------------
-- TABLE: screening_results
-- Purpose: Overall screening results for each request
-- ----------------------------------------------------------------------------
CREATE TABLE sanctions.screening_results (
    result_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id          UUID NOT NULL UNIQUE REFERENCES sanctions.screening_requests(request_id),
    overall_decision    VARCHAR(20) NOT NULL,
    total_parties       INTEGER NOT NULL,
    clear_count         INTEGER DEFAULT 0,
    review_count        INTEGER DEFAULT 0,
    block_count         INTEGER DEFAULT 0,
    total_hits          INTEGER DEFAULT 0,
    highest_score       DECIMAL(5,2),
    highest_hit_list    VARCHAR(50),
    embargo_detected    BOOLEAN DEFAULT FALSE,
    embargo_countries   VARCHAR(50)[],
    risk_score          DECIMAL(5,2),
    lists_screened      VARCHAR(50)[] NOT NULL,
    processing_time_ms  INTEGER NOT NULL,
    screened_at         TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    result_hash         VARCHAR(64),  -- For result verification
    
    CONSTRAINT chk_overall_decision CHECK (overall_decision IN ('CLEAR', 'REVIEW_LOW', 'REVIEW_STANDARD', 'REVIEW_URGENT', 'BLOCK'))
);

CREATE INDEX idx_results_decision ON sanctions.screening_results(overall_decision);
CREATE INDEX idx_results_date ON sanctions.screening_results(screened_at DESC);
CREATE INDEX idx_results_embargo ON sanctions.screening_results(embargo_detected) WHERE embargo_detected = TRUE;

-- ----------------------------------------------------------------------------
-- TABLE: sanctions_hits
-- Purpose: Individual potential matches (hits) found during screening
-- ----------------------------------------------------------------------------
CREATE TABLE sanctions.sanctions_hits (
    hit_id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    result_id           UUID NOT NULL REFERENCES sanctions.screening_results(result_id),
    party_id            UUID NOT NULL REFERENCES sanctions.screening_parties(party_id),
    entry_id            UUID NOT NULL REFERENCES sanctions.sanctions_entries(entry_id),
    list_id             UUID NOT NULL REFERENCES sanctions.sanctions_lists(list_id),
    match_score         DECIMAL(5,2) NOT NULL,
    match_type          VARCHAR(30) NOT NULL,
    match_algorithm     VARCHAR(30),
    matched_name        VARCHAR(500) NOT NULL,  -- The name that matched
    matched_field       VARCHAR(50) NOT NULL,   -- PRIMARY_NAME, ALIAS, IDENTIFIER
    match_details       JSONB,                  -- Detailed algorithm scores
    programs_matched    VARCHAR(100)[],
    status              VARCHAR(30) NOT NULL DEFAULT 'PENDING',
    disposition         VARCHAR(30),
    auto_disposition    BOOLEAN DEFAULT FALSE,
    disposition_reason  VARCHAR(500),
    reviewed_by         VARCHAR(100),
    reviewed_at         TIMESTAMP WITH TIME ZONE,
    review_comments     TEXT,
    escalation_level    INTEGER DEFAULT 0,
    sla_due_at          TIMESTAMP WITH TIME ZONE,
    created_at          TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_match_type CHECK (match_type IN ('EXACT', 'FUZZY_HIGH', 'FUZZY_MEDIUM', 'FUZZY_LOW', 'PHONETIC', 'ALIAS', 'TRANSLITERATION', 'IDENTIFIER')),
    CONSTRAINT chk_hit_status CHECK (status IN ('PENDING', 'QUEUED', 'CLAIMED', 'UNDER_REVIEW', 'ESCALATED', 'COMPLETED')),
    CONSTRAINT chk_disposition CHECK (disposition IN ('FALSE_POSITIVE', 'TRUE_POSITIVE', 'INCONCLUSIVE', 'AUTO_CLEARED', 'AUTO_BLOCKED'))
);

CREATE INDEX idx_hits_result ON sanctions.sanctions_hits(result_id);
CREATE INDEX idx_hits_party ON sanctions.sanctions_hits(party_id);
CREATE INDEX idx_hits_entry ON sanctions.sanctions_hits(entry_id);
CREATE INDEX idx_hits_score ON sanctions.sanctions_hits(match_score DESC);
CREATE INDEX idx_hits_status ON sanctions.sanctions_hits(status);
CREATE INDEX idx_hits_disposition ON sanctions.sanctions_hits(disposition);
CREATE INDEX idx_hits_sla ON sanctions.sanctions_hits(sla_due_at) WHERE status NOT IN ('COMPLETED');
CREATE INDEX idx_hits_pending ON sanctions.sanctions_hits(status, created_at) WHERE status IN ('PENDING', 'QUEUED');

-- ----------------------------------------------------------------------------
-- TABLE: hit_disposition_history
-- Purpose: Audit trail for hit disposition changes
-- ----------------------------------------------------------------------------
CREATE TABLE sanctions.hit_disposition_history (
    history_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    hit_id              UUID NOT NULL REFERENCES sanctions.sanctions_hits(hit_id),
    previous_status     VARCHAR(30),
    new_status          VARCHAR(30) NOT NULL,
    previous_disposition VARCHAR(30),
    new_disposition     VARCHAR(30),
    action_type         VARCHAR(30) NOT NULL,
    action_reason       VARCHAR(500),
    comments            TEXT,
    performed_by        VARCHAR(100) NOT NULL,
    performed_at        TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    ip_address          INET,
    user_agent          VARCHAR(300),
    
    CONSTRAINT chk_action_type CHECK (action_type IN ('CREATE', 'CLAIM', 'REVIEW', 'ESCALATE', 'CLEAR', 'CONFIRM', 'REASSIGN', 'COMMENT'))
);

CREATE INDEX idx_disposition_history_hit ON sanctions.hit_disposition_history(hit_id);
CREATE INDEX idx_disposition_history_user ON sanctions.hit_disposition_history(performed_by);
CREATE INDEX idx_disposition_history_date ON sanctions.hit_disposition_history(performed_at DESC);

-- ----------------------------------------------------------------------------
-- TABLE: false_positive_rules
-- Purpose: Rules for automatic false positive clearing
-- ----------------------------------------------------------------------------
CREATE TABLE sanctions.false_positive_rules (
    rule_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_name           VARCHAR(200) NOT NULL,
    rule_type           VARCHAR(30) NOT NULL,
    party_name          VARCHAR(500),
    party_identifier    VARCHAR(200),
    party_identifier_type VARCHAR(30),
    entry_id            UUID REFERENCES sanctions.sanctions_entries(entry_id),
    list_id             UUID REFERENCES sanctions.sanctions_lists(list_id),
    match_criteria      JSONB,
    reason              VARCHAR(500) NOT NULL,
    approved_by         VARCHAR(100) NOT NULL,
    approved_at         TIMESTAMP WITH TIME ZONE,
    valid_from          TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    valid_until         TIMESTAMP WITH TIME ZONE,
    is_active           BOOLEAN DEFAULT TRUE,
    usage_count         INTEGER DEFAULT 0,
    last_used_at        TIMESTAMP WITH TIME ZONE,
    created_at          TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_rule_type CHECK (rule_type IN ('NAME', 'IDENTIFIER', 'ENTRY', 'COMBINATION'))
);

CREATE INDEX idx_fp_rules_active ON sanctions.false_positive_rules(is_active) WHERE is_active = TRUE;
CREATE INDEX idx_fp_rules_entry ON sanctions.false_positive_rules(entry_id);
CREATE INDEX idx_fp_rules_name ON sanctions.false_positive_rules(party_name);

-- ----------------------------------------------------------------------------
-- TABLE: screening_audit_log
-- Purpose: Comprehensive audit log for all screening activities
-- ----------------------------------------------------------------------------
CREATE TABLE sanctions.screening_audit_log (
    audit_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type          VARCHAR(50) NOT NULL,
    entity_type         VARCHAR(30) NOT NULL,
    entity_id           UUID NOT NULL,
    action              VARCHAR(30) NOT NULL,
    old_values          JSONB,
    new_values          JSONB,
    request_id          UUID,
    transaction_id      VARCHAR(100),
    performed_by        VARCHAR(100),
    performed_at        TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    ip_address          INET,
    user_agent          VARCHAR(300),
    correlation_id      VARCHAR(100),
    system_component    VARCHAR(50),
    
    CONSTRAINT chk_entity_type CHECK (entity_type IN ('REQUEST', 'RESULT', 'HIT', 'LIST', 'ENTRY', 'RULE', 'CONFIG'))
);

-- Convert to TimescaleDB hypertable for efficient time-series storage
SELECT create_hypertable('sanctions.screening_audit_log', 'performed_at', 
    chunk_time_interval => INTERVAL '1 day',
    if_not_exists => TRUE);

CREATE INDEX idx_audit_entity ON sanctions.screening_audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_request ON sanctions.screening_audit_log(request_id);
CREATE INDEX idx_audit_user ON sanctions.screening_audit_log(performed_by);
CREATE INDEX idx_audit_time ON sanctions.screening_audit_log(performed_at DESC);

-- ----------------------------------------------------------------------------
-- TABLE: screening_metrics (Time-Series)
-- Purpose: Performance and volume metrics for monitoring
-- ----------------------------------------------------------------------------
CREATE TABLE sanctions.screening_metrics (
    metric_time         TIMESTAMP WITH TIME ZONE NOT NULL,
    metric_name         VARCHAR(100) NOT NULL,
    metric_value        DECIMAL(15,4) NOT NULL,
    metric_unit         VARCHAR(20),
    dimensions          JSONB,
    PRIMARY KEY (metric_time, metric_name)
);

SELECT create_hypertable('sanctions.screening_metrics', 'metric_time',
    chunk_time_interval => INTERVAL '1 hour',
    if_not_exists => TRUE);

CREATE INDEX idx_metrics_name ON sanctions.screening_metrics(metric_name, metric_time DESC);

-- ----------------------------------------------------------------------------
-- VIEWS
-- ----------------------------------------------------------------------------

-- View: Active Sanctions Entries (latest version only)
CREATE OR REPLACE VIEW sanctions.v_active_sanctions AS
SELECT 
    se.*,
    sl.list_code,
    sl.list_name,
    sl.authority,
    lv.version_number,
    lv.effective_date
FROM sanctions.sanctions_entries se
JOIN sanctions.list_versions lv ON se.version_id = lv.version_id
JOIN sanctions.sanctions_lists sl ON se.list_id = sl.list_id
WHERE lv.status = 'ACTIVE'
  AND se.is_active = TRUE;

-- View: Pending Review Queue
CREATE OR REPLACE VIEW sanctions.v_pending_reviews AS
SELECT 
    h.hit_id,
    h.match_score,
    h.match_type,
    h.status,
    h.escalation_level,
    h.sla_due_at,
    h.created_at,
    p.full_name AS party_name,
    p.party_role,
    e.primary_name AS matched_entry,
    l.list_code,
    l.authority,
    r.request_id,
    r.transaction_id,
    r.screening_type,
    EXTRACT(EPOCH FROM (CURRENT_TIMESTAMP - h.created_at))/3600 AS hours_pending
FROM sanctions.sanctions_hits h
JOIN sanctions.screening_parties p ON h.party_id = p.party_id
JOIN sanctions.sanctions_entries e ON h.entry_id = e.entry_id
JOIN sanctions.sanctions_lists l ON h.list_id = l.list_id
JOIN sanctions.screening_results sr ON h.result_id = sr.result_id
JOIN sanctions.screening_requests r ON sr.request_id = r.request_id
WHERE h.status IN ('PENDING', 'QUEUED', 'UNDER_REVIEW', 'ESCALATED')
ORDER BY h.sla_due_at ASC, h.match_score DESC;

-- View: Daily Screening Statistics
CREATE OR REPLACE VIEW sanctions.v_daily_stats AS
SELECT 
    DATE(requested_at) AS screening_date,
    screening_type,
    COUNT(*) AS total_requests,
    COUNT(*) FILTER (WHERE status = 'COMPLETED') AS completed,
    AVG(processing_time_ms) AS avg_processing_ms,
    PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY processing_time_ms) AS p99_processing_ms
FROM sanctions.screening_requests
WHERE requested_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY DATE(requested_at), screening_type
ORDER BY screening_date DESC, screening_type;
```

### 5.2 Data Retention Policy

| Data Type | Retention Period | Archive Strategy | Legal Basis |
|-----------|------------------|------------------|-------------|
| Screening Requests | 7 years | Cold storage after 1 year | OFAC Record Keeping |
| Screening Results | 7 years | Cold storage after 1 year | Regulatory Requirement |
| Sanctions Hits | 7 years | Never delete, archive only | Examination Support |
| Hit Disposition History | 7 years | Archive with hits | Audit Trail |
| Sanctions Lists (Active) | Current only | N/A | Operational |
| List Versions (Historical) | 7 years | Cold storage after 2 years | Version History |
| Audit Logs | 7 years | Compressed archive after 90 days | Compliance |
| Performance Metrics | 2 years | Aggregated after 30 days | Operational |
| False Positive Rules | Indefinite | Archive inactive rules | Reference |

### 5.3 Data Indexing Strategy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SANCTIONS DATA INDEXING ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    PRIMARY INDEXES (PostgreSQL)                      │   │
│  │                                                                      │   │
│  │  • B-Tree: Primary keys, Foreign keys, Exact lookups               │   │
│  │  • GIN: Full-text search (tsvector), Array columns (programs)      │   │
│  │  • Hash: Equality comparisons (status, type enums)                  │   │
│  │  • Partial: Conditional indexes (active records only)               │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    SPECIALIZED SEARCH INDEXES                        │   │
│  │                                                                      │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │              ELASTICSEARCH / OPENSEARCH                      │   │   │
│  │  │                                                              │   │   │
│  │  │  Index: sanctions_entries                                    │   │   │
│  │  │  • primary_name (text, phonetic analyzer)                    │   │   │
│  │  │  • aliases.alias_name (nested, phonetic)                     │   │   │
│  │  │  • identifiers (keyword, exact match)                        │   │   │
│  │  │  • addresses (text, geo_point)                               │   │   │
│  │  │  • programs (keyword, filter)                                │   │   │
│  │  │  • entry_type (keyword, filter)                              │   │   │
│  │  │                                                              │   │   │
│  │  │  Analyzers:                                                  │   │   │
│  │  │  • phonetic_soundex                                          │   │   │
│  │  │  • phonetic_metaphone                                        │   │   │
│  │  │  • phonetic_double_metaphone                                 │   │   │
│  │  │  • arabic_transliteration                                    │   │   │
│  │  │  • cyrillic_transliteration                                  │   │   │
│  │  │  • chinese_pinyin                                            │   │   │
│  │  │                                                              │   │   │
│  │  │  Similarity: Trigram, Jaro-Winkler (custom plugin)          │   │   │
│  │  │                                                              │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    IN-MEMORY CACHE (Redis)                           │   │
│  │                                                                      │   │
│  │  • Trie Index: Fast prefix matching for names                       │   │
│  │  • Hash Index: Exact identifier lookups (BIC, IMO, Passport)       │   │
│  │  • Set Index: Embargo country codes for O(1) lookup                │   │
│  │  • Sorted Set: Recently screened names (dedup optimization)        │   │
│  │  • Bloom Filter: Probabilistic negative lookup (not in any list)   │   │
│  │                                                                      │   │
│  │  TTL: List data = until next refresh | Results = 5 minutes          │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. API Design

### 6.1 API Overview

The Sanctions Check module exposes RESTful APIs for real-time screening, batch operations, list management, and administrative functions.

| API Category | Base Path | Description |
|--------------|-----------|-------------|
| Screening API | `/api/v1/sanctions/screen` | Real-time transaction and entity screening |
| Batch API | `/api/v1/sanctions/batch` | Bulk screening operations |
| List Management API | `/api/v1/sanctions/lists` | Sanctions list administration |
| Hit Management API | `/api/v1/sanctions/hits` | Hit review and disposition |
| Embargo API | `/api/v1/sanctions/embargo` | Country embargo checks |
| Admin API | `/api/v1/sanctions/admin` | Configuration and administration |
| Reports API | `/api/v1/sanctions/reports` | Reporting and analytics |

### 6.2 Screening API Endpoints

#### 6.2.1 Screen Transaction

**POST** `/api/v1/sanctions/screen/transaction`

Screen a payment transaction with all parties against sanctions lists.

**Request:**
```json
{
  "transactionId": "TXN-2026-0220-001234",
  "screeningType": "TRANSACTION",
  "priority": "HIGH",
  "channel": "API",
  "correlationId": "corr-uuid-12345",
  "idempotencyKey": "idem-key-abc123",
  "transaction": {
    "amount": 50000.00,
    "currency": "USD",
    "paymentType": "WIRE",
    "valueDate": "2026-02-20",
    "purposeCode": "TRADE"
  },
  "parties": [
    {
      "partyRole": "ORIGINATOR",
      "partyType": "INDIVIDUAL",
      "name": {
        "fullName": "MOHAMMED AL-RASHID",
        "firstName": "MOHAMMED",
        "lastName": "AL-RASHID"
      },
      "dateOfBirth": "1975-03-15",
      "nationality": "SA",
      "countryCode": "SA",
      "address": {
        "line1": "123 King Fahd Road",
        "city": "Riyadh",
        "country": "SA"
      },
      "identifiers": [
        {
          "type": "PASSPORT",
          "value": "A12345678",
          "country": "SA"
        }
      ]
    },
    {
      "partyRole": "BENEFICIARY",
      "partyType": "ENTITY",
      "name": {
        "fullName": "GLOBAL TRADING COMPANY LLC"
      },
      "countryCode": "AE",
      "address": {
        "line1": "Dubai Internet City",
        "city": "Dubai",
        "country": "AE"
      },
      "identifiers": [
        {
          "type": "LEI",
          "value": "5493001KJTIIGC8Y1R12"
        }
      ]
    },
    {
      "partyRole": "CORRESPONDENT_BANK",
      "partyType": "FINANCIAL_INSTITUTION",
      "name": {
        "fullName": "EMIRATES NBD BANK PJSC"
      },
      "countryCode": "AE",
      "identifiers": [
        {
          "type": "BIC_CODE",
          "value": "EABORXXX"
        }
      ]
    }
  ],
  "screeningConfig": {
    "lists": ["OFAC_SDN", "EU_CONSOLIDATED", "UN_CONSOLIDATED", "UK_SANCTIONS"],
    "matchThreshold": 80,
    "includeEmbargo": true,
    "enableFuzzyMatch": true,
    "enablePhoneticMatch": true
  },
  "metadata": {
    "sourceSystem": "PAYMENT_GATEWAY",
    "requestingUser": "system-auto",
    "branchCode": "HQ001"
  }
}
```

**Response (200 OK - Clear):**
```json
{
  "requestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "transactionId": "TXN-2026-0220-001234",
  "screeningResult": {
    "resultId": "res-uuid-567890",
    "overallDecision": "CLEAR",
    "decisionCode": "SC_CLEAR_001",
    "decisionMessage": "All parties cleared - no sanctions matches found",
    "riskScore": 15.5,
    "processingTimeMs": 87,
    "screenedAt": "2026-02-20T10:30:45.123Z"
  },
  "partySummary": {
    "totalParties": 3,
    "clearedParties": 3,
    "reviewParties": 0,
    "blockedParties": 0
  },
  "embargoCheck": {
    "embargoDetected": false,
    "countriesChecked": ["SA", "AE"]
  },
  "listsScreened": ["OFAC_SDN", "EU_CONSOLIDATED", "UN_CONSOLIDATED", "UK_SANCTIONS"],
  "listVersions": {
    "OFAC_SDN": "2026-02-20-v1",
    "EU_CONSOLIDATED": "2026-02-19-v3",
    "UN_CONSOLIDATED": "2026-02-15-v2",
    "UK_SANCTIONS": "2026-02-20-v1"
  },
  "auditReference": "AUD-2026-0220-001234",
  "canProceed": true
}
```

**Response (200 OK - Review Required):**
```json
{
  "requestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "transactionId": "TXN-2026-0220-001234",
  "screeningResult": {
    "resultId": "res-uuid-567890",
    "overallDecision": "REVIEW_STANDARD",
    "decisionCode": "SC_REVIEW_002",
    "decisionMessage": "Potential sanctions match detected - manual review required",
    "riskScore": 72.5,
    "processingTimeMs": 112,
    "screenedAt": "2026-02-20T10:30:45.123Z"
  },
  "partySummary": {
    "totalParties": 3,
    "clearedParties": 2,
    "reviewParties": 1,
    "blockedParties": 0
  },
  "hits": [
    {
      "hitId": "hit-uuid-123456",
      "partyRole": "BENEFICIARY",
      "partyName": "GLOBAL TRADING COMPANY LLC",
      "matchedEntry": {
        "entryId": "entry-uuid-789",
        "listCode": "OFAC_SDN",
        "entryName": "GLOBAL TRADING CO",
        "entryType": "ENTITY",
        "programs": ["SDGT", "IRAN"]
      },
      "matchScore": 87.5,
      "matchType": "FUZZY_HIGH",
      "matchDetails": {
        "jaroWinkler": 0.92,
        "levenshtein": 0.85,
        "phonetic": 0.88,
        "combined": 0.875
      },
      "status": "PENDING",
      "reviewRequired": true,
      "slaDeadline": "2026-02-20T14:30:45.123Z"
    }
  ],
  "embargoCheck": {
    "embargoDetected": false,
    "countriesChecked": ["SA", "AE"]
  },
  "reviewInfo": {
    "caseId": "CASE-2026-0220-0045",
    "assignedQueue": "L1_REVIEW",
    "priority": "STANDARD",
    "slaDueAt": "2026-02-20T14:30:45.123Z"
  },
  "canProceed": false,
  "holdPayment": true
}
```

**Response (200 OK - Block):**
```json
{
  "requestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "transactionId": "TXN-2026-0220-001234",
  "screeningResult": {
    "resultId": "res-uuid-567890",
    "overallDecision": "BLOCK",
    "decisionCode": "SC_BLOCK_001",
    "decisionMessage": "Transaction blocked - confirmed sanctions match",
    "riskScore": 100.0,
    "processingTimeMs": 45,
    "screenedAt": "2026-02-20T10:30:45.123Z"
  },
  "partySummary": {
    "totalParties": 3,
    "clearedParties": 2,
    "reviewParties": 0,
    "blockedParties": 1
  },
  "hits": [
    {
      "hitId": "hit-uuid-123456",
      "partyRole": "BENEFICIARY",
      "partyName": "GLOBAL TRADING COMPANY LLC",
      "matchedEntry": {
        "entryId": "entry-uuid-789",
        "listCode": "OFAC_SDN",
        "entryName": "GLOBAL TRADING COMPANY LLC",
        "entryType": "ENTITY",
        "programs": ["SDGT", "IRAN"],
        "listingDate": "2020-05-15"
      },
      "matchScore": 100.0,
      "matchType": "EXACT",
      "status": "AUTO_BLOCKED",
      "disposition": "TRUE_POSITIVE",
      "autoDisposition": true
    }
  ],
  "embargoCheck": {
    "embargoDetected": false,
    "countriesChecked": ["SA", "AE"]
  },
  "canProceed": false,
  "rejectPayment": true,
  "reportingRequired": true,
  "reportReference": "RPT-OFAC-2026-0220-0012"
}
```

#### 6.2.2 Screen Single Party

**POST** `/api/v1/sanctions/screen/party`

Screen an individual party (person or entity) against sanctions lists.

**Request:**
```json
{
  "screeningType": "CUSTOMER_ONBOARD",
  "priority": "NORMAL",
  "correlationId": "corr-uuid-67890",
  "party": {
    "partyType": "INDIVIDUAL",
    "name": {
      "fullName": "VLADIMIR PETROV",
      "firstName": "VLADIMIR",
      "lastName": "PETROV"
    },
    "dateOfBirth": "1968-07-22",
    "nationality": "RU",
    "countryCode": "RU",
    "address": {
      "city": "Moscow",
      "country": "RU"
    },
    "identifiers": [
      {
        "type": "PASSPORT",
        "value": "1234567890",
        "country": "RU"
      }
    ]
  },
  "screeningConfig": {
    "lists": ["ALL"],
    "matchThreshold": 75,
    "includeEmbargo": true
  }
}
```

**Response (200 OK):**
```json
{
  "requestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
  "screeningResult": {
    "resultId": "res-uuid-234567",
    "overallDecision": "CLEAR",
    "riskScore": 22.0,
    "processingTimeMs": 65
  },
  "partyResult": {
    "partyId": "party-uuid-111",
    "decision": "CLEAR",
    "hitCount": 0
  },
  "embargoCheck": {
    "embargoDetected": false,
    "countryChecked": "RU",
    "embargoPrograms": ["RUSSIA-EO14024"],
    "embargoStatus": "SECTORAL_ONLY",
    "restrictions": ["DEBT", "EQUITY"],
    "note": "Sectoral sanctions apply - check transaction type"
  },
  "canProceed": true,
  "warnings": [
    {
      "code": "WARN_SECTORAL",
      "message": "Country subject to sectoral sanctions - additional review may be required for certain transaction types"
    }
  ]
}
```

#### 6.2.3 Screen BIC/SWIFT Code

**POST** `/api/v1/sanctions/screen/bic`

Screen a bank identifier code against financial institution sanctions.

**Request:**
```json
{
  "bicCode": "VTBRRUM2XXX",
  "screeningType": "TRANSACTION",
  "includeSecondary": true
}
```

**Response:**
```json
{
  "requestId": "c3d4e5f6-a7b8-9012-cdef-123456789012",
  "bicCode": "VTBRRUM2XXX",
  "institution": {
    "name": "VTB BANK (PJSC)",
    "country": "RU",
    "city": "MOSCOW"
  },
  "screeningResult": {
    "decision": "BLOCK",
    "matchScore": 100.0
  },
  "hits": [
    {
      "listCode": "OFAC_SDN",
      "entryName": "VTB BANK PUBLIC JOINT STOCK COMPANY",
      "programs": ["RUSSIA-EO14024", "UKRAINE-EO13662"],
      "listingDate": "2022-02-24",
      "sanctionType": "COMPREHENSIVE"
    }
  ],
  "canProceed": false
}
```

#### 6.2.4 Screen Vessel

**POST** `/api/v1/sanctions/screen/vessel`

Screen a vessel against maritime sanctions lists.

**Request:**
```json
{
  "vesselName": "LADY M",
  "imoNumber": "1009613",
  "mmsi": "319866000",
  "flagCountry": "KY",
  "vesselType": "YACHT",
  "screeningType": "TRANSACTION"
}
```

**Response:**
```json
{
  "requestId": "d4e5f6a7-b8c9-0123-def0-234567890123",
  "vesselScreening": {
    "decision": "REVIEW_URGENT",
    "matchScore": 95.0
  },
  "hits": [
    {
      "listCode": "OFAC_SDN",
      "entryType": "VESSEL",
      "matchedName": "LADY M",
      "matchedIMO": "1009613",
      "registeredOwner": "SANCTIONED INDIVIDUAL",
      "programs": ["RUSSIA-EO14024"],
      "matchType": "EXACT"
    }
  ],
  "vesselDetails": {
    "previousNames": ["LADY M II", "RAGNAR"],
    "ownershipHistory": "Complex structure - review required"
  },
  "canProceed": false
}
```

### 6.3 Batch Screening API

#### 6.3.1 Submit Batch Screening Job

**POST** `/api/v1/sanctions/batch/submit`

Submit a batch of parties for screening.

**Request:**
```json
{
  "batchId": "BATCH-2026-0220-001",
  "batchType": "PERIODIC_REVIEW",
  "priority": "LOW",
  "totalRecords": 50000,
  "callbackUrl": "https://internal.api/callbacks/sanctions",
  "fileReference": {
    "storageType": "S3",
    "bucket": "sanctions-batch-input",
    "key": "2026/02/20/customer-review-batch.csv"
  },
  "screeningConfig": {
    "lists": ["ALL"],
    "matchThreshold": 75,
    "includeEmbargo": true
  },
  "outputConfig": {
    "format": "JSON",
    "includeScoreDetails": true,
    "outputBucket": "sanctions-batch-output"
  }
}
```

**Response (202 Accepted):**
```json
{
  "jobId": "JOB-2026-0220-001",
  "batchId": "BATCH-2026-0220-001",
  "status": "ACCEPTED",
  "estimatedCompletionTime": "2026-02-20T12:30:00Z",
  "trackingUrl": "/api/v1/sanctions/batch/status/JOB-2026-0220-001"
}
```

#### 6.3.2 Get Batch Job Status

**GET** `/api/v1/sanctions/batch/status/{jobId}`

**Response:**
```json
{
  "jobId": "JOB-2026-0220-001",
  "batchId": "BATCH-2026-0220-001",
  "status": "IN_PROGRESS",
  "progress": {
    "totalRecords": 50000,
    "processedRecords": 32500,
    "percentComplete": 65.0,
    "remainingTime": "PT15M"
  },
  "statistics": {
    "clearedCount": 31200,
    "reviewCount": 1150,
    "blockedCount": 150,
    "errorCount": 0
  },
  "startedAt": "2026-02-20T10:30:00Z",
  "estimatedCompletionTime": "2026-02-20T11:15:00Z"
}
```

### 6.4 List Management API

#### 6.4.1 Get Active Lists

**GET** `/api/v1/sanctions/lists`

**Response:**
```json
{
  "lists": [
    {
      "listId": "list-uuid-001",
      "listCode": "OFAC_SDN",
      "listName": "OFAC Specially Designated Nationals List",
      "authority": "U.S. Treasury - OFAC",
      "jurisdiction": "US",
      "status": "ACTIVE",
      "currentVersion": {
        "versionId": "ver-uuid-001",
        "versionNumber": "2026-02-20-v1",
        "publishedDate": "2026-02-20",
        "effectiveDate": "2026-02-20T06:00:00Z",
        "entryCount": 12543
      },
      "lastUpdated": "2026-02-20T06:15:32Z",
      "nextScheduledUpdate": "2026-02-21T06:00:00Z"
    },
    {
      "listId": "list-uuid-002",
      "listCode": "EU_CONSOLIDATED",
      "listName": "EU Consolidated List of Sanctions",
      "authority": "European Commission",
      "jurisdiction": "EU",
      "status": "ACTIVE",
      "currentVersion": {
        "versionId": "ver-uuid-002",
        "versionNumber": "2026-02-19-v3",
        "publishedDate": "2026-02-19",
        "effectiveDate": "2026-02-19T12:00:00Z",
        "entryCount": 8921
      },
      "lastUpdated": "2026-02-19T12:22:15Z",
      "nextScheduledUpdate": "2026-02-20T12:00:00Z"
    }
  ],
  "totalLists": 15,
  "totalActiveEntries": 85432
}
```

#### 6.4.2 Trigger List Update

**POST** `/api/v1/sanctions/lists/{listCode}/refresh`

**Request:**
```json
{
  "updateType": "DELTA",
  "forceRefresh": false,
  "activateImmediately": true
}
```

**Response:**
```json
{
  "updateJobId": "UPD-2026-0220-001",
  "listCode": "OFAC_SDN",
  "status": "IN_PROGRESS",
  "updateType": "DELTA",
  "previousVersion": "2026-02-19-v2",
  "targetVersion": "2026-02-20-v1",
  "startedAt": "2026-02-20T14:30:00Z"
}
```

#### 6.4.3 Search Sanctions Entries

**POST** `/api/v1/sanctions/lists/search`

**Request:**
```json
{
  "searchType": "NAME",
  "query": "AL-QAEDA",
  "filters": {
    "lists": ["OFAC_SDN", "UN_CONSOLIDATED"],
    "entryTypes": ["ENTITY", "INDIVIDUAL"],
    "programs": ["SDGT"],
    "activeOnly": true
  },
  "matchOptions": {
    "enableFuzzy": true,
    "minScore": 70
  },
  "pagination": {
    "page": 1,
    "pageSize": 50
  }
}
```

**Response:**
```json
{
  "totalResults": 127,
  "page": 1,
  "pageSize": 50,
  "results": [
    {
      "entryId": "entry-uuid-001",
      "listCode": "OFAC_SDN",
      "entryType": "ENTITY",
      "primaryName": "AL-QA'IDA",
      "aliases": [
        "AL-QAEDA",
        "AL QAIDA",
        "THE BASE"
      ],
      "programs": ["SDGT"],
      "listingDate": "2001-10-12",
      "matchScore": 100.0,
      "matchType": "EXACT"
    }
  ]
}
```

### 6.5 Hit Management API

#### 6.5.1 Get Pending Hits

**GET** `/api/v1/sanctions/hits/pending`

**Query Parameters:**
- `queue`: L1_REVIEW, L2_REVIEW, COMPLIANCE (optional)
- `status`: PENDING, QUEUED, CLAIMED, ESCALATED (optional)
- `priority`: LOW, NORMAL, HIGH, URGENT (optional)
- `page`, `pageSize`: Pagination

**Response:**
```json
{
  "totalHits": 45,
  "page": 1,
  "pageSize": 20,
  "hits": [
    {
      "hitId": "hit-uuid-001",
      "transactionId": "TXN-2026-0220-001234",
      "partyName": "GLOBAL TRADING COMPANY LLC",
      "partyRole": "BENEFICIARY",
      "matchedEntry": {
        "entryId": "entry-uuid-789",
        "listCode": "OFAC_SDN",
        "entryName": "GLOBAL TRADING CO",
        "programs": ["SDGT", "IRAN"]
      },
      "matchScore": 87.5,
      "matchType": "FUZZY_HIGH",
      "status": "QUEUED",
      "queue": "L1_REVIEW",
      "priority": "STANDARD",
      "slaDeadline": "2026-02-20T14:30:00Z",
      "slaBreach": false,
      "hoursInQueue": 2.5,
      "createdAt": "2026-02-20T10:30:00Z"
    }
  ],
  "queueStatistics": {
    "L1_REVIEW": 32,
    "L2_REVIEW": 10,
    "COMPLIANCE": 3,
    "breachingSLA": 2
  }
}
```

#### 6.5.2 Claim Hit for Review

**POST** `/api/v1/sanctions/hits/{hitId}/claim`

**Request:**
```json
{
  "analystId": "analyst-001",
  "notes": "Claiming for review"
}
```

**Response:**
```json
{
  "hitId": "hit-uuid-001",
  "status": "CLAIMED",
  "claimedBy": "analyst-001",
  "claimedAt": "2026-02-20T11:00:00Z",
  "slaRemaining": "PT3H30M"
}
```

#### 6.5.3 Disposition Hit

**POST** `/api/v1/sanctions/hits/{hitId}/disposition`

**Request:**
```json
{
  "disposition": "FALSE_POSITIVE",
  "reason": "Different entity - reviewed incorporation documents",
  "comments": "The screening subject is 'Global Trading Company LLC' incorporated in UAE 2015. The sanctions entry 'Global Trading Co' is a different entity based in Iran. Verified through official UAE company registry and LEI lookup.",
  "evidenceReferences": [
    "DOC-UAE-REGISTRY-2026-001",
    "LEI-5493001KJTIIGC8Y1R12"
  ],
  "createFalsePositiveRule": true,
  "analystId": "analyst-001"
}
```

**Response:**
```json
{
  "hitId": "hit-uuid-001",
  "disposition": "FALSE_POSITIVE",
  "status": "COMPLETED",
  "dispositionedBy": "analyst-001",
  "dispositionedAt": "2026-02-20T11:30:00Z",
  "falsePositiveRuleId": "FPR-2026-0220-001",
  "transactionReleased": true,
  "auditReference": "AUD-DISP-2026-0220-001"
}
```

#### 6.5.4 Escalate Hit

**POST** `/api/v1/sanctions/hits/{hitId}/escalate`

**Request:**
```json
{
  "escalateTo": "L2_REVIEW",
  "reason": "Complex ownership structure requires senior review",
  "notes": "Multiple shell companies involved, potential layered ownership",
  "analystId": "analyst-001"
}
```

**Response:**
```json
{
  "hitId": "hit-uuid-001",
  "status": "ESCALATED",
  "escalationLevel": 2,
  "escalatedTo": "L2_REVIEW",
  "escalatedBy": "analyst-001",
  "escalatedAt": "2026-02-20T11:15:00Z",
  "newSlaDeadline": "2026-02-20T17:15:00Z"
}
```

### 6.6 Embargo Check API

#### 6.6.1 Check Country Embargo

**POST** `/api/v1/sanctions/embargo/check`

**Request:**
```json
{
  "countries": ["RU", "IR", "CU", "KP", "SY"],
  "transactionType": "WIRE_TRANSFER",
  "purpose": "TRADE_FINANCE",
  "includeSecondary": true
}
```

**Response:**
```json
{
  "embargoResults": [
    {
      "countryCode": "RU",
      "countryName": "Russia",
      "embargoStatus": "SECTORAL",
      "canProceed": "CONDITIONAL",
      "restrictions": [
        {
          "program": "RUSSIA-EO14024",
          "authority": "OFAC",
          "restrictionType": "SECTORAL",
          "sectors": ["FINANCIAL", "ENERGY", "DEFENSE"],
          "requiresLicense": true
        }
      ],
      "guidance": "Sectoral sanctions apply. Review transaction against prohibited sectors."
    },
    {
      "countryCode": "IR",
      "countryName": "Iran",
      "embargoStatus": "COMPREHENSIVE",
      "canProceed": false,
      "restrictions": [
        {
          "program": "IRAN",
          "authority": "OFAC",
          "restrictionType": "COMPREHENSIVE",
          "requiresLicense": true,
          "licenseType": "SPECIFIC"
        }
      ],
      "guidance": "Comprehensive sanctions - transaction prohibited without specific OFAC license."
    },
    {
      "countryCode": "CU",
      "countryName": "Cuba",
      "embargoStatus": "COMPREHENSIVE",
      "canProceed": false,
      "restrictions": [
        {
          "program": "CUBA",
          "authority": "OFAC",
          "restrictionType": "COMPREHENSIVE"
        }
      ]
    },
    {
      "countryCode": "KP",
      "countryName": "North Korea",
      "embargoStatus": "COMPREHENSIVE",
      "canProceed": false,
      "restrictions": [
        {
          "program": "NORTH_KOREA",
          "authority": "OFAC",
          "restrictionType": "COMPREHENSIVE"
        }
      ]
    },
    {
      "countryCode": "SY",
      "countryName": "Syria",
      "embargoStatus": "COMPREHENSIVE",
      "canProceed": false,
      "restrictions": [
        {
          "program": "SYRIA",
          "authority": "OFAC",
          "restrictionType": "COMPREHENSIVE"
        }
      ]
    }
  ],
  "summary": {
    "totalCountries": 5,
    "blockedCountries": 4,
    "conditionalCountries": 1,
    "clearCountries": 0
  }
}
```

### 6.7 Error Responses

#### Standard Error Response Format

```json
{
  "error": {
    "code": "SCR_ERR_001",
    "type": "VALIDATION_ERROR",
    "message": "Invalid screening request",
    "details": [
      {
        "field": "parties[0].name.fullName",
        "error": "Name is required for screening"
      }
    ],
    "timestamp": "2026-02-20T10:30:00Z",
    "requestId": "req-uuid-12345",
    "traceId": "trace-abc-123"
  }
}
```

#### Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| SCR_ERR_001 | 400 | Invalid request format or missing required fields |
| SCR_ERR_002 | 400 | Invalid party type or role |
| SCR_ERR_003 | 400 | Invalid screening configuration |
| SCR_ERR_004 | 404 | Screening request not found |
| SCR_ERR_005 | 404 | Hit not found |
| SCR_ERR_006 | 409 | Hit already claimed by another analyst |
| SCR_ERR_007 | 409 | Duplicate idempotency key |
| SCR_ERR_008 | 422 | Unable to process - list not available |
| SCR_ERR_009 | 429 | Rate limit exceeded |
| SCR_ERR_010 | 500 | Internal screening engine error |
| SCR_ERR_011 | 503 | Sanctions service temporarily unavailable |

### 6.8 API Rate Limits

| API Category | Rate Limit | Burst |
|--------------|------------|-------|
| Transaction Screening | 1,000/sec | 2,000 |
| Party Screening | 2,000/sec | 4,000 |
| BIC Screening | 5,000/sec | 10,000 |
| Batch Submit | 10/min | 20 |
| Hit Management | 500/sec | 1,000 |
| List Management | 10/min | 20 |
| Search | 100/sec | 200 |

---

## 7. Event-Driven Architecture

### 7.1 Event Overview

The Sanctions Check module publishes domain events for screening activities, list updates, and hit management through the Event Bus (Kafka).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SANCTIONS CHECK EVENT ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  SANCTIONS CHECK MODULE                              │   │
│  │                                                                      │   │
│  │   Screening         List              Hit                Admin      │   │
│  │   Engine            Manager           Manager            Service    │   │
│  │      │                │                 │                   │        │   │
│  │      └────────────────┴─────────────────┴───────────────────┘        │   │
│  │                              │                                       │   │
│  │                              ▼                                       │   │
│  │               ┌────────────────────────────────┐                    │   │
│  │               │      Event Publisher           │                    │   │
│  │               │   (Domain Event Outbox)        │                    │   │
│  │               └────────────────────────────────┘                    │   │
│  │                              │                                       │   │
│  └──────────────────────────────┼───────────────────────────────────────┘   │
│                                 │                                           │
│                                 ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         KAFKA EVENT BUS                              │   │
│  │                                                                      │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │   │
│  │  │ sanctions.      │  │ sanctions.      │  │ sanctions.          │  │   │
│  │  │ screening       │  │ lists           │  │ hits                │  │   │
│  │  │ .events         │  │ .events         │  │ .events             │  │   │
│  │  └────────┬────────┘  └────────┬────────┘  └──────────┬──────────┘  │   │
│  │           │                    │                      │              │   │
│  └───────────┼────────────────────┼──────────────────────┼──────────────┘   │
│              │                    │                      │                  │
│              ▼                    ▼                      ▼                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐ │
│  │ Payment         │  │ Compliance      │  │ Case Management             │ │
│  │ Orchestration   │  │ Reporting       │  │ System                      │ │
│  │ Domain          │  │ Service         │  │                             │ │
│  │                 │  │                 │  │ • Alert Notification        │ │
│  │ • Resume Payment│  │ • OFAC Reports  │  │ • Case Creation             │ │
│  │ • Block Payment │  │ • Audit Records │  │ • Analyst Assignment        │ │
│  │ • Update Status │  │ • Statistics    │  │ • SLA Tracking              │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Domain Events

#### 7.2.1 Screening Events

| Event Name | Topic | Description |
|------------|-------|-------------|
| `ScreeningRequestReceived` | `sanctions.screening.events` | New screening request received |
| `ScreeningStarted` | `sanctions.screening.events` | Screening processing started |
| `ScreeningCompleted` | `sanctions.screening.events` | Screening completed with results |
| `ScreeningFailed` | `sanctions.screening.events` | Screening failed due to error |
| `TransactionCleared` | `sanctions.screening.events` | Transaction cleared - no matches |
| `TransactionBlocked` | `sanctions.screening.events` | Transaction blocked - confirmed match |
| `TransactionHeld` | `sanctions.screening.events` | Transaction held for review |

#### 7.2.2 Hit Events

| Event Name | Topic | Description |
|------------|-------|-------------|
| `HitDetected` | `sanctions.hits.events` | Potential match detected |
| `HitQueued` | `sanctions.hits.events` | Hit assigned to review queue |
| `HitClaimed` | `sanctions.hits.events` | Hit claimed by analyst |
| `HitEscalated` | `sanctions.hits.events` | Hit escalated to higher level |
| `HitCleared` | `sanctions.hits.events` | Hit dispositioned as false positive |
| `HitConfirmed` | `sanctions.hits.events` | Hit confirmed as true positive |
| `HitSLABreach` | `sanctions.hits.events` | Hit SLA deadline breached |

#### 7.2.3 List Events

| Event Name | Topic | Description |
|------------|-------|-------------|
| `ListUpdateStarted` | `sanctions.lists.events` | List update process started |
| `ListUpdateCompleted` | `sanctions.lists.events` | List successfully updated |
| `ListUpdateFailed` | `sanctions.lists.events` | List update failed |
| `ListVersionActivated` | `sanctions.lists.events` | New list version activated |
| `ListEntryAdded` | `sanctions.lists.events` | New entry added to list |
| `ListEntryRemoved` | `sanctions.lists.events` | Entry removed from list |
| `ListEntryModified` | `sanctions.lists.events` | Existing entry modified |

### 7.3 Event Schemas

#### 7.3.1 ScreeningCompleted Event

```json
{
  "eventId": "evt-uuid-12345",
  "eventType": "ScreeningCompleted",
  "eventVersion": "1.0",
  "aggregateType": "ScreeningResult",
  "aggregateId": "res-uuid-567890",
  "timestamp": "2026-02-20T10:30:45.123Z",
  "correlationId": "corr-uuid-12345",
  "causationId": "evt-uuid-12340",
  "payload": {
    "requestId": "req-uuid-12345",
    "transactionId": "TXN-2026-0220-001234",
    "resultId": "res-uuid-567890",
    "decision": "CLEAR",
    "decisionCode": "SC_CLEAR_001",
    "riskScore": 15.5,
    "partySummary": {
      "total": 3,
      "cleared": 3,
      "review": 0,
      "blocked": 0
    },
    "hitCount": 0,
    "embargoDetected": false,
    "processingTimeMs": 87,
    "listsScreened": ["OFAC_SDN", "EU_CONSOLIDATED", "UN_CONSOLIDATED", "UK_SANCTIONS"]
  },
  "metadata": {
    "sourceSystem": "SANCTIONS_CHECK",
    "sourceVersion": "2.1.0",
    "partitionKey": "TXN-2026-0220-001234"
  }
}
```

#### 7.3.2 HitDetected Event

```json
{
  "eventId": "evt-uuid-23456",
  "eventType": "HitDetected",
  "eventVersion": "1.0",
  "aggregateType": "SanctionsHit",
  "aggregateId": "hit-uuid-123456",
  "timestamp": "2026-02-20T10:30:45.150Z",
  "correlationId": "corr-uuid-12345",
  "payload": {
    "hitId": "hit-uuid-123456",
    "resultId": "res-uuid-567890",
    "requestId": "req-uuid-12345",
    "transactionId": "TXN-2026-0220-001234",
    "party": {
      "partyId": "party-uuid-001",
      "partyRole": "BENEFICIARY",
      "partyName": "GLOBAL TRADING COMPANY LLC"
    },
    "matchedEntry": {
      "entryId": "entry-uuid-789",
      "listCode": "OFAC_SDN",
      "entryName": "GLOBAL TRADING CO",
      "entryType": "ENTITY",
      "programs": ["SDGT", "IRAN"]
    },
    "matchScore": 87.5,
    "matchType": "FUZZY_HIGH",
    "initialStatus": "PENDING",
    "slaDeadline": "2026-02-20T14:30:45.123Z",
    "priority": "STANDARD"
  }
}
```

#### 7.3.3 ListVersionActivated Event

```json
{
  "eventId": "evt-uuid-34567",
  "eventType": "ListVersionActivated",
  "eventVersion": "1.0",
  "aggregateType": "SanctionsList",
  "aggregateId": "list-uuid-001",
  "timestamp": "2026-02-20T06:15:32Z",
  "payload": {
    "listId": "list-uuid-001",
    "listCode": "OFAC_SDN",
    "previousVersion": {
      "versionId": "ver-uuid-old",
      "versionNumber": "2026-02-19-v2",
      "entryCount": 12530
    },
    "newVersion": {
      "versionId": "ver-uuid-new",
      "versionNumber": "2026-02-20-v1",
      "entryCount": 12543,
      "publishedDate": "2026-02-20"
    },
    "changes": {
      "additions": 15,
      "modifications": 8,
      "deletions": 2
    },
    "activatedBy": "system-scheduler"
  }
}
```

### 7.4 Event Consumption

| Consumer | Events Consumed | Action |
|----------|-----------------|--------|
| Payment Orchestration | `ScreeningCompleted`, `TransactionCleared`, `TransactionBlocked` | Resume/reject payment |
| Case Management | `HitDetected`, `HitQueued`, `HitEscalated` | Create/update investigation cases |
| Notification Service | `HitDetected`, `HitSLABreach`, `TransactionBlocked` | Send alerts to analysts/compliance |
| Compliance Reporting | All screening and hit events | Generate regulatory reports |
| Audit Service | All events | Store in audit log |
| Analytics Service | All events | Update dashboards and metrics |
| Re-screening Service | `ListVersionActivated`, `ListEntryAdded` | Trigger customer re-screening |

---

## 8. Sequence Diagrams

### 8.1 Real-Time Transaction Screening

```
┌──────────┐ ┌───────────────┐ ┌─────────────────┐ ┌──────────────┐ ┌─────────────┐ ┌───────────────┐
│ Payment  │ │  Sanctions    │ │   Screening     │ │    Name      │ │   List      │ │    Cache      │
│ Gateway  │ │  Orchestrator │ │   Engine        │ │   Matcher    │ │   Store     │ │   (Redis)     │
└────┬─────┘ └───────┬───────┘ └────────┬────────┘ └──────┬───────┘ └──────┬──────┘ └───────┬───────┘
     │               │                   │                 │                │                │
     │ POST /screen  │                   │                 │                │                │
     │──────────────▶│                   │                 │                │                │
     │               │                   │                 │                │                │
     │               │ Validate Request  │                 │                │                │
     │               │──────────────────▶│                 │                │                │
     │               │                   │                 │                │                │
     │               │                   │ Check Cache     │                │                │
     │               │                   │ (recent screen) │                │                │
     │               │                   │────────────────────────────────────────────────▶│
     │               │                   │                 │                │                │
     │               │                   │◀────────────────────────────────────────────────│
     │               │                   │ Cache Miss      │                │                │
     │               │                   │                 │                │                │
     │               │                   │ Normalize Names │                │                │
     │               │                   │────────────────▶│                │                │
     │               │                   │                 │                │                │
     │               │                   │                 │ Load Active    │                │
     │               │                   │                 │ Entries        │                │
     │               │                   │                 │───────────────▶│                │
     │               │                   │                 │                │                │
     │               │                   │                 │◀───────────────│                │
     │               │                   │                 │                │                │
     │               │                   │ ┌─────────────────────────────────┐               │
     │               │                   │ │    PARALLEL NAME MATCHING       │               │
     │               │                   │ │                                 │               │
     │               │                   │ │  Party 1 ──▶ [Exact│Fuzzy│Phone]│               │
     │               │                   │ │  Party 2 ──▶ [Exact│Fuzzy│Phone]│               │
     │               │                   │ │  Party 3 ──▶ [Exact│Fuzzy│Phone]│               │
     │               │                   │ │                                 │               │
     │               │                   │ └─────────────────────────────────┘               │
     │               │                   │                 │                │                │
     │               │                   │◀────────────────│                │                │
     │               │                   │ Match Results   │                │                │
     │               │                   │                 │                │                │
     │               │                   │ Check Embargo   │                │                │
     │               │                   │ Countries       │                │                │
     │               │                   │────────────────────────────────────────────────▶│
     │               │                   │                 │                │                │
     │               │                   │◀────────────────────────────────────────────────│
     │               │                   │                 │                │                │
     │               │                   │ Calculate Scores│                │                │
     │               │                   │ Make Decision   │                │                │
     │               │                   │                 │                │                │
     │               │ Screening Result  │                 │                │                │
     │               │◀──────────────────│                 │                │                │
     │               │                   │                 │                │                │
     │               │      [DECISION = CLEAR]             │                │                │
     │               │                   │                 │                │                │
     │               │ Cache Result      │                 │                │                │
     │               │──────────────────────────────────────────────────────────────────────▶│
     │               │                   │                 │                │                │
     │               │ Publish Event     │                 │                │                │
     │               │ (ScreeningCompleted)               │                │                │
     │               │                   │                 │                │                │
     │ 200 OK        │                   │                 │                │                │
     │ (CLEAR)       │                   │                 │                │                │
     │◀──────────────│                   │                 │                │                │
     │               │                   │                 │                │                │
```

### 8.2 Screening with Hit Detection and Review

```
┌──────────┐ ┌───────────────┐ ┌──────────────┐ ┌────────────────┐ ┌─────────────┐ ┌──────────────┐
│ Payment  │ │  Sanctions    │ │  Screening   │ │  Hit           │ │  Case       │ │  Analyst     │
│ Gateway  │ │  Orchestrator │ │  Engine      │ │  Manager       │ │  Management │ │  Workstation │
└────┬─────┘ └───────┬───────┘ └──────┬───────┘ └───────┬────────┘ └──────┬──────┘ └──────┬───────┘
     │               │                │                  │                 │               │
     │ POST /screen  │                │                  │                 │               │
     │──────────────▶│                │                  │                 │               │
     │               │                │                  │                 │               │
     │               │ Screen Request │                  │                 │               │
     │               │───────────────▶│                  │                 │               │
     │               │                │                  │                 │               │
     │               │                │ [Match Found     │                 │               │
     │               │                │  Score: 87.5%]   │                 │               │
     │               │                │                  │                 │               │
     │               │ Hits Detected  │                  │                 │               │
     │               │◀───────────────│                  │                 │               │
     │               │ (Decision:     │                  │                 │               │
     │               │  REVIEW)       │                  │                 │               │
     │               │                │                  │                 │               │
     │               │ Create Hits    │                  │                 │               │
     │               │────────────────────────────────▶│                 │               │
     │               │                │                  │                 │               │
     │               │                │                  │ Assign Queue    │               │
     │               │                │                  │ Calculate SLA   │               │
     │               │                │                  │                 │               │
     │               │                │                  │ Create Case     │               │
     │               │                │                  │─────────────────▶│               │
     │               │                │                  │                 │               │
     │               │                │                  │◀────────────────│               │
     │               │                │                  │ Case Created    │               │
     │               │                │                  │                 │               │
     │               │ Hit Queued     │                  │                 │               │
     │               │◀────────────────────────────────│                 │               │
     │               │                │                  │                 │               │
     │               │ Publish Events │                  │                 │               │
     │               │ (HitDetected,  │                  │                 │               │
     │               │  TransactionHeld)                │                 │               │
     │               │                │                  │                 │               │
     │ 200 OK        │                │                  │                 │               │
     │ (REVIEW,      │                │                  │                 │               │
     │  holdPayment) │                │                  │                 │               │
     │◀──────────────│                │                  │                 │               │
     │               │                │                  │                 │               │
     │               │                │                  │                 │               │
     │               │                │                  │  ═══════════════════════════   │
     │               │                │                  │  ║ ANALYST REVIEW WORKFLOW ║   │
     │               │                │                  │  ═══════════════════════════   │
     │               │                │                  │                 │               │
     │               │                │                  │                 │ GET /hits     │
     │               │                │                  │                 │ /pending      │
     │               │                │                  │◀─────────────────────────────────│
     │               │                │                  │                 │               │
     │               │                │                  │ Pending Hits    │               │
     │               │                │                  │──────────────────────────────────▶│
     │               │                │                  │                 │               │
     │               │                │                  │ POST /hits/{id} │               │
     │               │                │                  │ /claim          │               │
     │               │                │                  │◀─────────────────────────────────│
     │               │                │                  │                 │               │
     │               │                │                  │ Hit Claimed     │               │
     │               │                │                  │──────────────────────────────────▶│
     │               │                │                  │                 │               │
     │               │                │                  │                 │               │
     │               │                │                  │ [Analyst Reviews│               │
     │               │                │                  │  Evidence]      │               │
     │               │                │                  │                 │               │
     │               │                │                  │ POST /hits/{id} │               │
     │               │                │                  │ /disposition    │               │
     │               │                │                  │ (FALSE_POSITIVE)│               │
     │               │                │                  │◀─────────────────────────────────│
     │               │                │                  │                 │               │
     │               │                │                  │ Update Hit      │               │
     │               │                │                  │ Status          │               │
     │               │                │                  │                 │               │
     │               │ Hit Cleared    │                  │                 │               │
     │               │◀────────────────────────────────│                 │               │
     │               │                │                  │                 │               │
     │               │ Publish Event  │                  │                 │               │
     │               │ (HitCleared,   │                  │                 │               │
     │               │  TransactionCleared)             │                 │               │
     │               │                │                  │                 │               │
     │               │────────────────────────────────────────────────────▶│  Close Case  │
     │               │                │                  │                 │               │
     │ Release       │                │                  │                 │               │
     │ Payment       │                │                  │                 │               │
     │◀──────────────│                │                  │                 │               │
     │               │                │                  │                 │               │
```

### 8.3 Sanctions List Update Process

```
┌────────────┐ ┌───────────────┐ ┌──────────────┐ ┌───────────────┐ ┌─────────────┐ ┌──────────────┐
│ Scheduler  │ │  List         │ │  Source      │ │  Index        │ │  Cache      │ │  Event       │
│            │ │  Manager      │ │  Connector   │ │  Builder      │ │  (Redis)    │ │  Bus         │
└─────┬──────┘ └───────┬───────┘ └──────┬───────┘ └───────┬───────┘ └──────┬──────┘ └──────┬───────┘
      │                │                │                  │                │               │
      │ Trigger Update │                │                  │                │               │
      │ (06:00 UTC)    │                │                  │                │               │
      │───────────────▶│                │                  │                │               │
      │                │                │                  │                │               │
      │                │ Publish        │                  │                │               │
      │                │ ListUpdateStarted                │                │               │
      │                │─────────────────────────────────────────────────────────────────▶│
      │                │                │                  │                │               │
      │                │ Fetch List     │                  │                │               │
      │                │ (OFAC SDN)     │                  │                │               │
      │                │───────────────▶│                  │                │               │
      │                │                │                  │                │               │
      │                │                │ GET https://     │                │               │
      │                │                │ sanctionssearch. │                │               │
      │                │                │ ofac.treas.gov/  │                │               │
      │                │                │ ──────────────▶  │                │               │
      │                │                │                  │                │               │
      │                │                │ ◀────────────────                │               │
      │                │                │ SDN XML File     │                │               │
      │                │                │                  │                │               │
      │                │ Raw List Data  │                  │                │               │
      │                │◀───────────────│                  │                │               │
      │                │                │                  │                │               │
      │                │ Validate &     │                  │                │               │
      │                │ Parse          │                  │                │               │
      │                │                │                  │                │               │
      │                │ Calculate      │                  │                │               │
      │                │ Delta Changes  │                  │                │               │
      │                │                │                  │                │               │
      │                │ Build Indexes  │                  │                │               │
      │                │─────────────────────────────────▶│                │               │
      │                │                │                  │                │               │
      │                │                │                  │ Create Trie    │               │
      │                │                │                  │ Index          │               │
      │                │                │                  │                │               │
      │                │                │                  │ Create Phonetic│               │
      │                │                │                  │ Index          │               │
      │                │                │                  │                │               │
      │                │                │                  │ Create Search  │               │
      │                │                │                  │ Index          │               │
      │                │                │                  │                │               │
      │                │ Indexes Built  │                  │                │               │
      │                │◀─────────────────────────────────│                │               │
      │                │                │                  │                │               │
      │                │ Update Cache   │                  │                │               │
      │                │ (Atomic Swap)  │                  │                │               │
      │                │─────────────────────────────────────────────────▶│               │
      │                │                │                  │                │               │
      │                │◀─────────────────────────────────────────────────│               │
      │                │ Cache Updated  │                  │                │               │
      │                │                │                  │                │               │
      │                │ Activate New   │                  │                │               │
      │                │ Version        │                  │                │               │
      │                │                │                  │                │               │
      │                │ Publish        │                  │                │               │
      │                │ ListVersionActivated             │                │               │
      │                │─────────────────────────────────────────────────────────────────▶│
      │                │                │                  │                │               │
      │                │ Publish        │                  │                │               │
      │                │ ListUpdateCompleted              │                │               │
      │                │─────────────────────────────────────────────────────────────────▶│
      │                │                │                  │                │               │
      │ Update         │                │                  │                │               │
      │ Completed      │                │                  │                │               │
      │◀───────────────│                │                  │                │               │
      │                │                │                  │                │               │
```

### 8.4 Embargo Country Check Flow

```
┌──────────┐ ┌────────────────┐ ┌──────────────┐ ┌─────────────┐
│ Payment  │ │  Embargo       │ │  Embargo     │ │  Cache      │
│ Request  │ │  Checker       │ │  Database    │ │  (Redis)    │
└────┬─────┘ └───────┬────────┘ └──────┬───────┘ └──────┬──────┘
     │               │                  │                │
     │ Check         │                  │                │
     │ Countries:    │                  │                │
     │ [SA, IR, AE]  │                  │                │
     │──────────────▶│                  │                │
     │               │                  │                │
     │               │ Check Cache      │                │
     │               │ (Embargo Set)    │                │
     │               │─────────────────────────────────▶│
     │               │                  │                │
     │               │◀─────────────────────────────────│
     │               │ Cached Result    │                │
     │               │ (IR = EMBARGO)   │                │
     │               │                  │                │
     │               │ Get Details      │                │
     │               │ for IR           │                │
     │               │─────────────────▶│                │
     │               │                  │                │
     │               │◀─────────────────│                │
     │               │ Embargo Details  │                │
     │               │ (COMPREHENSIVE,  │                │
     │               │  IRAN program)   │                │
     │               │                  │                │
     │ Embargo       │                  │                │
     │ Result        │                  │                │
     │ (IR blocked,  │                  │                │
     │  SA/AE clear) │                  │                │
     │◀──────────────│                  │                │
     │               │                  │                │
```

---

## 9. Integration Points

### 9.1 Integration Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                       SANCTIONS CHECK INTEGRATION ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                          UPSTREAM INTEGRATIONS                                 │ │
│  │                                                                                │ │
│  │   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐  │ │
│  │   │ Payment Gateway │  │ Customer        │  │ Periodic Review             │  │ │
│  │   │ Core            │  │ Onboarding      │  │ Scheduler                   │  │ │
│  │   │                 │  │                 │  │                             │  │ │
│  │   │ • Transaction   │  │ • KYC Screening │  │ • Batch Customer Review    │  │ │
│  │   │   Screening     │  │ • Account Open  │  │ • List Change Re-screen    │  │ │
│  │   └────────┬────────┘  └────────┬────────┘  └──────────────┬──────────────┘  │ │
│  │            │                    │                          │                  │ │
│  │            └────────────────────┼──────────────────────────┘                  │ │
│  │                                 │                                             │ │
│  │                                 ▼                                             │ │
│  └─────────────────────────────────┼─────────────────────────────────────────────┘ │
│                                    │                                               │
│  ┌─────────────────────────────────┼─────────────────────────────────────────────┐ │
│  │                                 │                                             │ │
│  │              ┌──────────────────┴──────────────────┐                         │ │
│  │              │     SANCTIONS CHECK MODULE          │                         │ │
│  │              │         (Port: 8005)                │                         │ │
│  │              └──────────────────┬──────────────────┘                         │ │
│  │                                 │                                             │ │
│  └─────────────────────────────────┼─────────────────────────────────────────────┘ │
│                                    │                                               │
│  ┌─────────────────────────────────┼─────────────────────────────────────────────┐ │
│  │                          DOWNSTREAM INTEGRATIONS                              │ │
│  │                                 │                                             │ │
│  │   ┌─────────────────┐  ┌───────┴─────────┐  ┌─────────────────────────────┐  │ │
│  │   │ Case Management │  │ Notification    │  │ Compliance Reporting        │  │ │
│  │   │ System          │  │ Service         │  │                             │  │ │
│  │   │                 │  │                 │  │                             │  │ │
│  │   │ • Case Creation │  │ • Email Alerts  │  │ • OFAC Reports              │  │ │
│  │   │ • Hit Tracking  │  │ • SMS Alerts    │  │ • Audit Records             │  │ │
│  │   │ • Investigation │  │ • Slack/Teams   │  │ • Regulatory Filing         │  │ │
│  │   └─────────────────┘  └─────────────────┘  └─────────────────────────────┘  │ │
│  │                                                                               │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                     │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                          EXTERNAL INTEGRATIONS                                 │ │
│  │                                                                                │ │
│  │   ┌─────────────────────────────────────────────────────────────────────────┐ │ │
│  │   │                    SANCTIONS LIST PROVIDERS                              │ │ │
│  │   │                                                                         │ │ │
│  │   │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐  │ │ │
│  │   │  │ OFAC     │ │ EU       │ │ UN       │ │ UK OFSI  │ │ World-Check  │  │ │ │
│  │   │  │ (API/FTP)│ │ (API)    │ │ (XML)    │ │ (API)    │ │ (API)        │  │ │ │
│  │   │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────────┘  │ │ │
│  │   │                                                                         │ │ │
│  │   │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐  │ │ │
│  │   │  │ Dow Jones│ │ DFAT     │ │ SECO     │ │ OSFI     │ │ MAS          │  │ │ │
│  │   │  │ (API)    │ │ (CSV)    │ │ (XML)    │ │ (XML)    │ │ (PDF/Manual) │  │ │ │
│  │   │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────────┘  │ │ │
│  │   │                                                                         │ │ │
│  │   └─────────────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                                │ │
│  │   ┌─────────────────────────────────────────────────────────────────────────┐ │ │
│  │   │                    ENRICHMENT SERVICES                                   │ │ │
│  │   │                                                                         │ │ │
│  │   │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌────────────────┐ │ │ │
│  │   │  │ GLEIF (LEI)  │ │ SWIFT BIC    │ │ IHS Markit   │ │ OpenCorporates │ │ │ │
│  │   │  │ Lookup       │ │ Directory    │ │ Vessel Data  │ │ Registry       │ │ │ │
│  │   │  └──────────────┘ └──────────────┘ └──────────────┘ └────────────────┘ │ │ │
│  │   │                                                                         │ │ │
│  │   └─────────────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                                │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 9.2 Payment Orchestration Domain Integration

| Integration Point | Type | Protocol | Description |
|-------------------|------|----------|-------------|
| Transaction Screening | Synchronous | gRPC | Real-time screening during payment processing |
| Screening Result | Synchronous | gRPC | Return decision (CLEAR/REVIEW/BLOCK) |
| Transaction Hold | Async Event | Kafka | Notify payment to hold pending review |
| Transaction Release | Async Event | Kafka | Notify payment to proceed after clearance |
| Transaction Block | Async Event | Kafka | Notify payment to reject |

**gRPC Service Definition:**

```protobuf
syntax = "proto3";

package sanctions.v1;

service SanctionsScreeningService {
  // Screen a payment transaction
  rpc ScreenTransaction(ScreenTransactionRequest) returns (ScreenTransactionResponse);
  
  // Screen a single party
  rpc ScreenParty(ScreenPartyRequest) returns (ScreenPartyResponse);
  
  // Check embargo status for countries
  rpc CheckEmbargo(EmbargoCheckRequest) returns (EmbargoCheckResponse);
  
  // Get screening result by ID
  rpc GetScreeningResult(GetResultRequest) returns (ScreeningResult);
}

message ScreenTransactionRequest {
  string transaction_id = 1;
  string correlation_id = 2;
  string idempotency_key = 3;
  ScreeningPriority priority = 4;
  repeated Party parties = 5;
  TransactionDetails transaction = 6;
  ScreeningConfig config = 7;
}

message ScreenTransactionResponse {
  string request_id = 1;
  string result_id = 2;
  ScreeningDecision decision = 3;
  string decision_code = 4;
  string decision_message = 5;
  double risk_score = 6;
  int32 hit_count = 7;
  bool embargo_detected = 8;
  int64 processing_time_ms = 9;
  bool can_proceed = 10;
  repeated HitSummary hits = 11;
}

enum ScreeningDecision {
  DECISION_UNSPECIFIED = 0;
  CLEAR = 1;
  REVIEW_LOW = 2;
  REVIEW_STANDARD = 3;
  REVIEW_URGENT = 4;
  BLOCK = 5;
}

message Party {
  string party_id = 1;
  PartyType party_type = 2;
  PartyRole party_role = 3;
  Name name = 4;
  string date_of_birth = 5;
  string nationality = 6;
  string country_code = 7;
  Address address = 8;
  repeated Identifier identifiers = 9;
}
```

### 9.3 External List Provider Integrations

#### 9.3.1 OFAC Integration

| Attribute | Value |
|-----------|-------|
| Provider | U.S. Treasury - OFAC |
| Endpoint | `https://sanctionssearch.ofac.treas.gov/` |
| Protocol | HTTPS (REST API) + FTP Download |
| Authentication | API Key (for search API) |
| Data Format | XML (SDN_ADVANCED.xml) |
| Update Frequency | Daily (9:00 AM ET) |
| SLA | 99.9% |

**OFAC API Integration:**

```yaml
ofac:
  api:
    base_url: "https://sanctionssearch.ofac.treas.gov"
    search_endpoint: "/api/SearchResults"
    api_key: "${OFAC_API_KEY}"
    timeout_ms: 5000
    retry_count: 3
    retry_delay_ms: 1000
  ftp:
    host: "ofacftp.treasury.gov"
    path: "/pub/sdn/"
    files:
      - "SDN_ADVANCED.xml"
      - "CONS_ADVANCED.xml"
    username: "anonymous"
    schedule: "0 9 * * *"  # 9 AM ET daily
  processing:
    validate_schema: true
    deduplicate: true
    normalize_names: true
```

#### 9.3.2 EU Sanctions Integration

| Attribute | Value |
|-----------|-------|
| Provider | European Commission |
| Endpoint | `https://webgate.ec.europa.eu/fsd/fsf/` |
| Protocol | HTTPS REST API |
| Authentication | None (public) |
| Data Format | XML |
| Update Frequency | Daily |

#### 9.3.3 World-Check (Refinitiv) Integration

| Attribute | Value |
|-----------|-------|
| Provider | Refinitiv (LSEG) |
| Endpoint | `https://api.refinitiv.com/permid/` |
| Protocol | HTTPS REST API |
| Authentication | OAuth 2.0 |
| Data Format | JSON |
| Update Frequency | Near real-time |
| SLA | 99.95% |

**World-Check Integration Config:**

```yaml
worldcheck:
  api:
    base_url: "https://api.refinitiv.com"
    auth_url: "https://api.refinitiv.com/auth/oauth2/v1/token"
    screening_endpoint: "/permid/screening/v2/cases"
    client_id: "${WORLDCHECK_CLIENT_ID}"
    client_secret: "${WORLDCHECK_CLIENT_SECRET}"
    scope: "trapi.screening"
    timeout_ms: 10000
  screening:
    group_id: "payment-hub-prod"
    match_strength: 80
    enable_ongoing: true
    categories:
      - "SANCTIONS"
      - "PEP"
      - "ADVERSE_MEDIA"
```

### 9.4 Case Management Integration

| Integration Type | Description |
|------------------|-------------|
| Protocol | REST API + Kafka Events |
| Case Creation | Create investigation case for hits requiring review |
| Case Update | Update case status based on disposition |
| Case Closure | Close case when hit is resolved |
| Assignment | Assign cases to analyst queues |
| Priority | Set case priority based on match score and list |

**Case Management Event:**

```json
{
  "eventType": "CaseCreationRequest",
  "payload": {
    "caseType": "SANCTIONS_HIT",
    "priority": "HIGH",
    "source": "SANCTIONS_CHECK",
    "referenceId": "hit-uuid-123456",
    "subject": {
      "type": "PARTY",
      "name": "GLOBAL TRADING COMPANY LLC",
      "identifiers": ["TXN-2026-0220-001234"]
    },
    "matchDetails": {
      "listCode": "OFAC_SDN",
      "matchedEntry": "GLOBAL TRADING CO",
      "matchScore": 87.5,
      "programs": ["SDGT", "IRAN"]
    },
    "assignTo": {
      "queue": "L1_SANCTIONS_REVIEW",
      "region": "AMERICAS"
    },
    "sla": {
      "responseTime": "PT4H",
      "resolutionTime": "PT24H"
    }
  }
}
```

### 9.5 Configuration Domain Integration

| Integration | Purpose | Protocol |
|-------------|---------|----------|
| Screening Rules Repository | Match thresholds, algorithm weights | REST/Cache |
| List Source Configuration | Source URLs, credentials, schedules | REST/Cache |
| Embargo Configuration | Country embargo rules | REST/Cache |
| False Positive Rules | Auto-clearance rules | REST/Cache |
| SLA Configuration | Review time limits | REST/Cache |

---

## 10. Security Design

### 10.1 Security Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                       SANCTIONS CHECK SECURITY ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                          PERIMETER SECURITY                                  │   │
│  │                                                                              │   │
│  │   ┌───────────────┐  ┌───────────────┐  ┌───────────────────────────────┐   │   │
│  │   │ WAF (Web      │  │ DDoS          │  │ API Gateway                   │   │   │
│  │   │ Application   │  │ Protection    │  │                               │   │   │
│  │   │ Firewall)     │  │               │  │ • Rate Limiting               │   │   │
│  │   │               │  │               │  │ • Authentication              │   │   │
│  │   │ • SQL Inject  │  │ • Volumetric  │  │ • Request Validation          │   │   │
│  │   │ • XSS         │  │ • Protocol    │  │ • mTLS Termination            │   │   │
│  │   │ • OWASP Top10 │  │ • Application │  │ • JWT Validation              │   │   │
│  │   └───────────────┘  └───────────────┘  └───────────────────────────────┘   │   │
│  │                                                                              │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                         │                                          │
│                                         ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                          APPLICATION SECURITY                                │   │
│  │                                                                              │   │
│  │   ┌───────────────────────────────────────────────────────────────────────┐ │   │
│  │   │                    AUTHENTICATION & AUTHORIZATION                     │ │   │
│  │   │                                                                       │ │   │
│  │   │  • OAuth 2.0 / OpenID Connect for API access                         │ │   │
│  │   │  • mTLS for service-to-service communication                         │ │   │
│  │   │  • JWT tokens with short expiry (15 minutes)                         │ │   │
│  │   │  • Role-Based Access Control (RBAC)                                  │ │   │
│  │   │  • Attribute-Based Access Control (ABAC) for sensitive operations    │ │   │
│  │   │                                                                       │ │   │
│  │   └───────────────────────────────────────────────────────────────────────┘ │   │
│  │                                                                              │   │
│  │   ┌───────────────────────────────────────────────────────────────────────┐ │   │
│  │   │                    RBAC ROLE DEFINITIONS                              │ │   │
│  │   │                                                                       │ │   │
│  │   │  Role                  Permissions                                   │ │   │
│  │   │  ──────────────────────────────────────────────────────────────────  │ │   │
│  │   │  SYSTEM_SERVICE        Screen transactions (read/write)              │ │   │
│  │   │  SANCTIONS_L1_ANALYST  View hits, claim, clear (low/medium score)    │ │   │
│  │   │  SANCTIONS_L2_ANALYST  All L1 + clear high-score, escalate           │ │   │
│  │   │  COMPLIANCE_OFFICER    All L2 + confirm true positive, report        │ │   │
│  │   │  SANCTIONS_ADMIN       List management, configuration                 │ │   │
│  │   │  AUDIT_READER          Read-only access to all records               │ │   │
│  │   │                                                                       │ │   │
│  │   └───────────────────────────────────────────────────────────────────────┘ │   │
│  │                                                                              │   │
│  │   ┌───────────────────────────────────────────────────────────────────────┐ │   │
│  │   │                    INPUT VALIDATION                                   │ │   │
│  │   │                                                                       │ │   │
│  │   │  • Schema validation for all API requests                             │ │   │
│  │   │  • Name sanitization (remove control characters, normalize unicode)  │ │   │
│  │   │  • Maximum field lengths enforced                                     │ │   │
│  │   │  • Country code validation against ISO 3166-1                        │ │   │
│  │   │  • BIC/SWIFT format validation                                       │ │   │
│  │   │  • Identifier format validation (passport, national ID patterns)     │ │   │
│  │   │                                                                       │ │   │
│  │   └───────────────────────────────────────────────────────────────────────┘ │   │
│  │                                                                              │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                          DATA SECURITY                                       │   │
│  │                                                                              │   │
│  │   ┌───────────────────────────────────────────────────────────────────────┐ │   │
│  │   │                    ENCRYPTION                                         │ │   │
│  │   │                                                                       │ │   │
│  │   │  In Transit:                                                          │ │   │
│  │   │  • TLS 1.3 for all external communications                           │ │   │
│  │   │  • mTLS for internal service mesh                                    │ │   │
│  │   │  • Certificate rotation every 90 days                                │ │   │
│  │   │                                                                       │ │   │
│  │   │  At Rest:                                                             │ │   │
│  │   │  • AES-256-GCM for database encryption (TDE)                         │ │   │
│  │   │  • Envelope encryption for sensitive fields (names, identifiers)     │ │   │
│  │   │  • KMS-managed encryption keys                                        │ │   │
│  │   │  • Key rotation every 365 days                                        │ │   │
│  │   │                                                                       │ │   │
│  │   │  Field-Level Encryption:                                              │ │   │
│  │   │  • Passport numbers                                                   │ │   │
│  │   │  • National ID numbers                                                │ │   │
│  │   │  • Date of birth                                                      │ │   │
│  │   │  • Full addresses                                                     │ │   │
│  │   │                                                                       │ │   │
│  │   └───────────────────────────────────────────────────────────────────────┘ │   │
│  │                                                                              │   │
│  │   ┌───────────────────────────────────────────────────────────────────────┐ │   │
│  │   │                    DATA MASKING                                       │ │   │
│  │   │                                                                       │ │   │
│  │   │  Log Masking:                                                         │ │   │
│  │   │  • Names: "John ****" (partial mask)                                  │ │   │
│  │   │  • Passport: "A***5678" (last 4 visible)                              │ │   │
│  │   │  • DOB: "****-**-15" (day only visible)                               │ │   │
│  │   │  • Account: "****1234" (last 4 visible)                               │ │   │
│  │   │                                                                       │ │   │
│  │   │  API Response Masking (by role):                                      │ │   │
│  │   │  • Full data: Compliance Officer and above                           │ │   │
│  │   │  • Partial mask: Analysts                                             │ │   │
│  │   │  • Full mask: System services (decision only)                        │ │   │
│  │   │                                                                       │ │   │
│  │   └───────────────────────────────────────────────────────────────────────┘ │   │
│  │                                                                              │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                          AUDIT & COMPLIANCE                                  │   │
│  │                                                                              │   │
│  │   • Immutable audit logs for all screening operations                       │   │
│  │   • Tamper-evident logging with cryptographic signatures                    │   │
│  │   • Complete audit trail for hit dispositions                               │   │
│  │   • User activity logging with IP address and user agent                    │   │
│  │   • List change audit (additions, modifications, deletions)                 │   │
│  │   • Configuration change logging with approval workflow                      │   │
│  │   • 7-year retention for regulatory compliance                              │   │
│  │   • WORM (Write Once Read Many) storage for audit data                     │   │
│  │                                                                              │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 10.2 Authentication & Authorization

#### 10.2.1 Service Authentication

```yaml
# Service-to-Service Authentication Configuration
service_auth:
  type: mTLS
  certificate:
    issuer: internal-ca
    validity_days: 90
    key_algorithm: ECDSA-P256
    auto_rotate: true
  allowed_services:
    - service: payment-gateway
      permissions: [SCREEN_TRANSACTION, GET_RESULT]
    - service: customer-onboarding
      permissions: [SCREEN_PARTY, GET_RESULT]
    - service: batch-processor
      permissions: [BATCH_SUBMIT, BATCH_STATUS]
```

#### 10.2.2 User Authentication

```yaml
# User Authentication Configuration
user_auth:
  provider: OAuth2/OIDC
  issuer: "https://auth.paymenthub.internal"
  token:
    type: JWT
    algorithm: RS256
    access_token_expiry: 15m
    refresh_token_expiry: 8h
  claims:
    required: [sub, iat, exp, roles, org_unit]
    optional: [region, analyst_level]
  mfa:
    required_for: [DISPOSITION, LIST_MANAGEMENT, ADMIN]
    methods: [TOTP, FIDO2]
```

#### 10.2.3 RBAC Permission Matrix

| Permission | System | L1 Analyst | L2 Analyst | Compliance | Admin | Auditor |
|------------|--------|------------|------------|------------|-------|---------|
| screen.transaction | ✓ | - | - | - | - | - |
| screen.party | ✓ | - | - | - | - | - |
| hit.view | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| hit.claim | - | ✓ | ✓ | ✓ | - | - |
| hit.clear.low | - | ✓ | ✓ | ✓ | - | - |
| hit.clear.high | - | - | ✓ | ✓ | - | - |
| hit.escalate | - | ✓ | ✓ | ✓ | - | - |
| hit.confirm | - | - | - | ✓ | - | - |
| list.view | - | ✓ | ✓ | ✓ | ✓ | ✓ |
| list.update | - | - | - | - | ✓ | - |
| list.activate | - | - | - | ✓ | ✓ | - |
| config.view | - | - | - | ✓ | ✓ | ✓ |
| config.update | - | - | - | - | ✓ | - |
| report.generate | - | - | - | ✓ | ✓ | ✓ |
| audit.view | - | - | - | - | - | ✓ |

### 10.3 Secrets Management

| Secret Type | Storage | Rotation | Access |
|-------------|---------|----------|--------|
| Database Credentials | HashiCorp Vault | 30 days | Service Account |
| API Keys (External Lists) | HashiCorp Vault | 90 days | List Manager Service |
| Encryption Keys | AWS KMS / Azure Key Vault | 365 days | Application |
| mTLS Certificates | Vault PKI | 90 days | Auto-rotate |
| JWT Signing Keys | Vault Transit | 180 days | Auth Service |

### 10.4 Network Security

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       NETWORK SECURITY ZONES                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  INTERNET ZONE (Untrusted)                                          │   │
│  │                                                                      │   │
│  │  External List Providers (OFAC, EU, World-Check)                    │   │
│  │  ↓ (HTTPS, API Keys, IP Whitelist)                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  DMZ (Semi-Trusted)                                                  │   │
│  │                                                                      │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────────┐   │   │
│  │  │ API Gateway     │  │ Load Balancer   │  │ WAF               │   │   │
│  │  └─────────────────┘  └─────────────────┘  └────────────────────┘   │   │
│  │  ↓ (mTLS)                                                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  APPLICATION ZONE (Trusted)                                          │   │
│  │                                                                      │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │ Sanctions Check Module (Service Mesh - Istio)               │    │   │
│  │  │ Port: 8005 (internal only)                                  │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │  ↓ (mTLS)                                                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  DATA ZONE (Restricted)                                              │   │
│  │                                                                      │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────┐  │   │
│  │  │ PostgreSQL  │  │ Redis       │  │ Elastics.   │  │ Kafka      │  │   │
│  │  │ (Encrypted) │  │ (Encrypted) │  │ (Encrypted) │  │ (Encrypted)│  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └────────────┘  │   │
│  │                                                                      │   │
│  │  Access: Service accounts only, no direct human access              │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 10.5 Security Compliance Requirements

| Requirement | Standard | Implementation |
|-------------|----------|----------------|
| Data Encryption | PCI-DSS 3.4, 4.1 | AES-256 at rest, TLS 1.3 in transit |
| Access Control | PCI-DSS 7.1, 7.2 | RBAC with least privilege |
| Audit Logging | PCI-DSS 10.x | Immutable audit logs, 1-year online, 7-year archive |
| Key Management | PCI-DSS 3.5, 3.6 | KMS with HSM backing |
| Vulnerability Management | PCI-DSS 6.1, 6.2 | Weekly scans, 30-day patch SLA |
| Penetration Testing | PCI-DSS 11.3 | Annual external, quarterly internal |
| Secure Development | PCI-DSS 6.3, 6.5 | SAST/DAST in CI/CD, code review |
| Network Security | PCI-DSS 1.x | Firewall, segmentation, IDS/IPS |

---

## 11. Non-Functional Requirements

### 11.1 Performance Requirements

| Requirement ID | Requirement | Target | Measurement |
|----------------|-------------|--------|-------------|
| NFR-PERF-001 | Single party screening latency (p50) | < 50ms | APM instrumentation |
| NFR-PERF-002 | Single party screening latency (p99) | < 80ms | APM instrumentation |
| NFR-PERF-003 | Transaction screening latency (p50) | < 100ms | End-to-end trace |
| NFR-PERF-004 | Transaction screening latency (p99) | < 150ms | End-to-end trace |
| NFR-PERF-005 | Name matching latency (single name) | < 20ms | Component metrics |
| NFR-PERF-006 | Embargo check latency | < 10ms | Cache hit measurement |
| NFR-PERF-007 | Throughput (real-time screening) | > 8,000 TPS | Load testing |
| NFR-PERF-008 | Throughput (batch screening) | > 100,000/hour | Batch job metrics |
| NFR-PERF-009 | List load time (full refresh) | < 30 seconds | Update job metrics |
| NFR-PERF-010 | List delta update propagation | < 2 minutes | Event timestamp delta |
| NFR-PERF-011 | Hit search response time | < 200ms | API response time |
| NFR-PERF-012 | Dashboard load time | < 3 seconds | Frontend metrics |

### 11.2 Scalability Requirements

| Requirement ID | Requirement | Target | Approach |
|----------------|-------------|--------|----------|
| NFR-SCAL-001 | Horizontal scaling | Auto-scale 2-50 pods | HPA based on CPU/RPS |
| NFR-SCAL-002 | List size support | Up to 5 million entries | Optimized indexing |
| NFR-SCAL-003 | Concurrent users (analysts) | Up to 500 | Connection pooling |
| NFR-SCAL-004 | Daily screening volume | Up to 50 million screenings | Distributed processing |
| NFR-SCAL-005 | Batch file size | Up to 1 million records | Chunked processing |
| NFR-SCAL-006 | Database storage growth | 5 years at 20% YoY | Partitioning, archival |

### 11.3 Availability Requirements

| Requirement ID | Requirement | Target | Implementation |
|----------------|-------------|--------|----------------|
| NFR-AVAIL-001 | Service availability | 99.99% (52 min/year) | Multi-AZ deployment |
| NFR-AVAIL-002 | Screening availability | 99.999% | Cached list fallback |
| NFR-AVAIL-003 | Planned maintenance window | < 15 min/month | Rolling deployments |
| NFR-AVAIL-004 | RTO (Recovery Time Objective) | < 15 minutes | Automated failover |
| NFR-AVAIL-005 | RPO (Recovery Point Objective) | < 1 minute | Synchronous replication |
| NFR-AVAIL-006 | List update availability | 99.9% | Multiple source fallback |

### 11.4 Reliability Requirements

| Requirement ID | Requirement | Target | Verification |
|----------------|-------------|--------|--------------|
| NFR-REL-001 | True positive detection rate | 100% | Regression testing |
| NFR-REL-002 | False positive rate | < 5% | Production metrics |
| NFR-REL-003 | Zero data loss | 100% | Audit verification |
| NFR-REL-004 | Idempotent processing | 100% | Integration tests |
| NFR-REL-005 | Screening consistency | Zero false negatives | Dual-screening validation |
| NFR-REL-006 | List integrity verification | Checksum validation | Update verification |

### 11.5 Accuracy Requirements

| Requirement ID | Requirement | Target | Measurement |
|----------------|-------------|--------|-------------|
| NFR-ACC-001 | Exact match detection | 100% | Test suite coverage |
| NFR-ACC-002 | Fuzzy match accuracy (> 90% score) | > 99% | Benchmark testing |
| NFR-ACC-003 | Phonetic match accuracy | > 95% | Linguistic test suite |
| NFR-ACC-004 | Transliteration accuracy | > 95% | Multi-script test suite |
| NFR-ACC-005 | Alias detection rate | > 98% | Alias test coverage |
| NFR-ACC-006 | Identifier match (exact) | 100% | Identifier test suite |

### 11.6 Compliance Requirements

| Requirement ID | Requirement | Regulation | Verification |
|----------------|-------------|------------|--------------|
| NFR-COMP-001 | Audit trail retention | 7 years (OFAC) | Retention policy |
| NFR-COMP-002 | Screening record retention | 5 years (BSA) | Data lifecycle |
| NFR-COMP-003 | List update within 24 hours | OFAC, EU | Update job SLA |
| NFR-COMP-004 | Hit review within 4 hours (urgent) | Internal policy | SLA dashboard |
| NFR-COMP-005 | Dual control for disposition | SOX, MAS | Workflow enforcement |
| NFR-COMP-006 | Geographic data residency | GDPR, local laws | Regional deployment |

### 11.7 Capacity Planning

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CAPACITY PLANNING MODEL                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Current Load Profile:                                                      │
│  ──────────────────────────────────────────────────────────────────────    │
│  • Daily Transactions:        10 million                                   │
│  • Peak TPS:                  3,000                                        │
│  • Average Parties/Txn:       2.5                                          │
│  • Daily Screenings:          25 million                                   │
│  • Hit Rate:                  0.3%                                         │
│  • Daily Hits:                75,000                                       │
│  • Review Rate:               5% of hits                                   │
│  • Daily Reviews:             3,750                                        │
│                                                                             │
│  Growth Projection (3 Years):                                              │
│  ──────────────────────────────────────────────────────────────────────    │
│  Year 1: 30% growth → 13M txn/day, 4,000 peak TPS                         │
│  Year 2: 25% growth → 16M txn/day, 5,000 peak TPS                         │
│  Year 3: 20% growth → 19M txn/day, 6,000 peak TPS                         │
│                                                                             │
│  Resource Requirements (Year 3):                                            │
│  ──────────────────────────────────────────────────────────────────────    │
│                                                                             │
│  Component              Baseline    Peak      Auto-Scale Max               │
│  ┌──────────────────────────────────────────────────────────────────┐     │
│  │ Screening Service     6 pods     12 pods    24 pods              │     │
│  │ Name Matching Engine  4 pods     8 pods     16 pods              │     │
│  │ Hit Manager           2 pods     4 pods     8 pods               │     │
│  │ List Manager          2 pods     2 pods     4 pods               │     │
│  │ Batch Processor       2 pods     8 pods     16 pods              │     │
│  └──────────────────────────────────────────────────────────────────┘     │
│                                                                             │
│  Storage Requirements (Year 3):                                             │
│  ──────────────────────────────────────────────────────────────────────    │
│                                                                             │
│  Data Type              Daily Growth   Year 3 Total   Retention           │
│  ┌──────────────────────────────────────────────────────────────────┐     │
│  │ Screening Records     2 GB/day      3.5 TB         7 years       │     │
│  │ Hit Records           500 MB/day    900 GB         7 years       │     │
│  │ Audit Logs            1 GB/day      1.8 TB         7 years       │     │
│  │ Sanctions Lists       50 MB         500 MB         Current only  │     │
│  │ Search Indexes        100 MB        1 GB           Current only  │     │
│  │ Cache Data            500 MB        500 MB         Live data     │     │
│  │ Total (with archive)  -             50 TB          7 years       │     │
│  └──────────────────────────────────────────────────────────────────┘     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 12. Deployment Architecture

### 12.1 Kubernetes Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                       SANCTIONS CHECK KUBERNETES DEPLOYMENT                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │  Namespace: risk-compliance                                                    │ │
│  │                                                                                │ │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐ │ │
│  │  │  Deployment: sanctions-screening-service                                 │ │ │
│  │  │                                                                          │ │ │
│  │  │  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐              │ │ │
│  │  │  │ Pod (replica 1)│ │ Pod (replica 2)│ │ Pod (replica N)│              │ │ │
│  │  │  │               │ │               │ │               │              │ │ │
│  │  │  │ ┌───────────┐ │ │ ┌───────────┐ │ │ ┌───────────┐ │              │ │ │
│  │  │  │ │sanctions- │ │ │ │sanctions- │ │ │ │sanctions- │ │              │ │ │
│  │  │  │ │check      │ │ │ │check      │ │ │ │check      │ │              │ │ │
│  │  │  │ │:8005      │ │ │ │:8005      │ │ │ │:8005      │ │              │ │ │
│  │  │  │ └───────────┘ │ │ └───────────┘ │ │ └───────────┘ │              │ │ │
│  │  │  │ ┌───────────┐ │ │ ┌───────────┐ │ │ ┌───────────┐ │              │ │ │
│  │  │  │ │istio-proxy│ │ │ │istio-proxy│ │ │ │istio-proxy│ │              │ │ │
│  │  │  │ └───────────┘ │ │ └───────────┘ │ │ └───────────┘ │              │ │ │
│  │  │  └────────────────┘ └────────────────┘ └────────────────┘              │ │ │
│  │  │                                                                          │ │ │
│  │  │  HPA: min=4, max=24, target CPU=70%, target RPS=500/pod                │ │ │
│  │  │  PDB: minAvailable=50%                                                  │ │ │
│  │  └─────────────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                                │ │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐ │ │
│  │  │  Deployment: name-matching-engine                                        │ │ │
│  │  │                                                                          │ │ │
│  │  │  Replicas: 2-16 (HPA)    Resources: 4 CPU, 8Gi Memory per pod          │ │ │
│  │  │  Specialized for: Fuzzy matching, phonetic analysis, transliteration   │ │ │
│  │  └─────────────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                                │ │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐ │ │
│  │  │  Deployment: hit-management-service                                      │ │ │
│  │  │                                                                          │ │ │
│  │  │  Replicas: 2-8 (HPA)    Resources: 2 CPU, 4Gi Memory per pod           │ │ │
│  │  │  Handles: Hit queue, analyst workflow, disposition                      │ │ │
│  │  └─────────────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                                │ │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐ │ │
│  │  │  Deployment: list-management-service                                     │ │ │
│  │  │                                                                          │ │ │
│  │  │  Replicas: 2-4 (static)  Resources: 2 CPU, 4Gi Memory per pod          │ │ │
│  │  │  Handles: List import, indexing, version management                     │ │ │
│  │  └─────────────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                                │ │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐ │ │
│  │  │  Deployment: batch-screening-processor                                   │ │ │
│  │  │                                                                          │ │ │
│  │  │  Replicas: 2-16 (KEDA based on queue depth)                             │ │ │
│  │  │  Resources: 4 CPU, 8Gi Memory per pod                                   │ │ │
│  │  │  Handles: Bulk screening jobs, periodic reviews                         │ │ │
│  │  └─────────────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                                │ │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐ │ │
│  │  │  Services                                                                │ │ │
│  │  │                                                                          │ │ │
│  │  │  ┌──────────────────────────────────┐                                   │ │ │
│  │  │  │ Service: sanctions-screening     │                                   │ │ │
│  │  │  │ Type: ClusterIP                  │                                   │ │ │
│  │  │  │ Port: 8005 (gRPC), 8085 (HTTP)  │                                   │ │ │
│  │  │  └──────────────────────────────────┘                                   │ │ │
│  │  │                                                                          │ │ │
│  │  └─────────────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                                │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 12.2 Multi-Region Deployment

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                       MULTI-REGION DEPLOYMENT TOPOLOGY                               │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│                    ┌─────────────────────────────────────┐                         │
│                    │         GLOBAL LOAD BALANCER         │                         │
│                    │     (GeoDNS / Traffic Manager)       │                         │
│                    └───────────────┬─────────────────────┘                         │
│                                    │                                               │
│          ┌─────────────────────────┼─────────────────────────┐                     │
│          │                         │                         │                     │
│          ▼                         ▼                         ▼                     │
│  ┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐            │
│  │   US-EAST-1       │   │   EU-WEST-1       │   │   AP-SOUTHEAST-1  │            │
│  │   (Primary)       │   │   (Regional)      │   │   (Regional)      │            │
│  │                   │   │                   │   │                   │            │
│  │ ┌───────────────┐ │   │ ┌───────────────┐ │   │ ┌───────────────┐ │            │
│  │ │  Sanctions    │ │   │ │  Sanctions    │ │   │ │  Sanctions    │ │            │
│  │ │  Check        │ │   │ │  Check        │ │   │ │  Check        │ │            │
│  │ │  Cluster      │ │   │ │  Cluster      │ │   │ │  Cluster      │ │            │
│  │ └───────────────┘ │   │ └───────────────┘ │   │ └───────────────┘ │            │
│  │                   │   │                   │   │                   │            │
│  │ ┌───────────────┐ │   │ ┌───────────────┐ │   │ ┌───────────────┐ │            │
│  │ │  PostgreSQL   │ │   │ │  PostgreSQL   │ │   │ │  PostgreSQL   │ │            │
│  │ │  (Primary)    │◀┼───┼▶│  (Replica)    │◀┼───┼▶│  (Replica)    │ │            │
│  │ └───────────────┘ │   │ └───────────────┘ │   │ └───────────────┘ │            │
│  │                   │   │                   │   │                   │            │
│  │ ┌───────────────┐ │   │ ┌───────────────┐ │   │ ┌───────────────┐ │            │
│  │ │  Redis        │ │   │ │  Redis        │ │   │ │  Redis        │ │            │
│  │ │  (Local List  │ │   │ │  (Local List  │ │   │ │  (Local List  │ │            │
│  │ │   Cache)      │ │   │ │   Cache)      │ │   │ │   Cache)      │ │            │
│  │ └───────────────┘ │   │ └───────────────┘ │   │ └───────────────┘ │            │
│  │                   │   │                   │   │                   │            │
│  └───────────────────┘   └───────────────────┘   └───────────────────┘            │
│                                                                                     │
│  Data Synchronization:                                                              │
│  • Sanctions Lists: Synchronized from Central (US-EAST-1) to all regions           │
│  • Screening Results: Written to local, async replicated to primary                 │
│  • Hits: Written to local, sync replicated for consistency                         │
│  • Configuration: Centralized in primary, cached locally                            │
│                                                                                     │
│  Failover Strategy:                                                                 │
│  • Automatic failover within region (multi-AZ)                                      │
│  • Manual failover between regions (< 15 min RTO)                                   │
│  • List cache fallback maintains screening capability during primary outage        │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 12.3 Kubernetes Deployment Manifests

#### 12.3.1 Deployment Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sanctions-screening-service
  namespace: risk-compliance
  labels:
    app: sanctions-screening
    version: v2.1.0
    tier: backend
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
  selector:
    matchLabels:
      app: sanctions-screening
  template:
    metadata:
      labels:
        app: sanctions-screening
        version: v2.1.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      serviceAccountName: sanctions-screening-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
        - name: sanctions-check
          image: paymenthub/sanctions-check:2.1.0
          imagePullPolicy: Always
          ports:
            - name: grpc
              containerPort: 8005
              protocol: TCP
            - name: http
              containerPort: 8085
              protocol: TCP
            - name: metrics
              containerPort: 9090
              protocol: TCP
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "kubernetes,prod"
            - name: JAVA_OPTS
              value: "-Xms2g -Xmx4g -XX:+UseG1GC -XX:MaxGCPauseMillis=100"
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: sanctions-db-credentials
                  key: host
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: sanctions-db-credentials
                  key: password
            - name: REDIS_HOST
              value: "sanctions-redis-cluster.risk-compliance.svc.cluster.local"
            - name: KAFKA_BROKERS
              value: "kafka-cluster.platform.svc.cluster.local:9092"
          resources:
            requests:
              cpu: "2"
              memory: "4Gi"
            limits:
              cpu: "4"
              memory: "6Gi"
          readinessProbe:
            grpc:
              port: 8005
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            grpc:
              port: 8005
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          volumeMounts:
            - name: config
              mountPath: /app/config
              readOnly: true
            - name: tls-certs
              mountPath: /app/certs
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: sanctions-screening-config
        - name: tls-certs
          secret:
            secretName: sanctions-screening-tls
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: sanctions-screening
                topologyKey: kubernetes.io/hostname
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: sanctions-screening
```

#### 12.3.2 Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: sanctions-screening-hpa
  namespace: risk-compliance
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sanctions-screening-service
  minReplicas: 4
  maxReplicas: 24
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: 500
    - type: Pods
      pods:
        metric:
          name: grpc_requests_per_second
        target:
          type: AverageValue
          averageValue: 500
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Pods
          value: 4
          periodSeconds: 30
        - type: Percent
          value: 50
          periodSeconds: 30
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
```

#### 12.3.3 Pod Disruption Budget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: sanctions-screening-pdb
  namespace: risk-compliance
spec:
  minAvailable: 50%
  selector:
    matchLabels:
      app: sanctions-screening
```

### 12.4 Infrastructure Components

| Component | Technology | Configuration |
|-----------|------------|---------------|
| Container Runtime | containerd | Latest stable |
| Kubernetes | 1.28+ | Managed (EKS/AKS/GKE) |
| Service Mesh | Istio 1.20+ | mTLS enabled |
| Ingress | Istio Gateway | TLS termination |
| Database | PostgreSQL 15+ | Multi-AZ, encrypted |
| Cache | Redis 7+ Cluster | 6-node cluster |
| Message Queue | Kafka 3.5+ | 3-broker cluster |
| Search | Elasticsearch 8+ | 3-node cluster |
| Secrets | HashiCorp Vault | Auto-unseal |
| Monitoring | Prometheus + Grafana | Custom dashboards |
| Logging | Fluentd + Elasticsearch | 30-day retention |
| Tracing | Jaeger | 7-day retention |

### 12.5 Environment Configuration

| Environment | Purpose | Replicas | Database | Cache |
|-------------|---------|----------|----------|-------|
| Development | Dev testing | 1 | Local PostgreSQL | Local Redis |
| Integration | Integration tests | 2 | Shared PostgreSQL | Shared Redis |
| Staging | Pre-production | 4 | Dedicated replica | Dedicated 3-node |
| Production | Live traffic | 4-24 (auto) | Multi-AZ primary | 6-node cluster |
| DR | Disaster recovery | 4 (standby) | Async replica | Synced cluster |

---

## 13. Monitoring & Alerting

### 13.1 Key Performance Metrics

| Metric Name | Type | Description | Target |
|-------------|------|-------------|--------|
| `sanctions_screening_latency_ms` | Histogram | End-to-end screening latency | p99 < 150ms |
| `sanctions_screening_throughput_rps` | Gauge | Screenings per second | > 8,000 |
| `sanctions_hit_count` | Counter | Total hits detected | Monitoring |
| `sanctions_hit_rate` | Gauge | Hit rate percentage | < 0.5% typical |
| `sanctions_clear_rate` | Gauge | Clear rate percentage | > 95% typical |
| `sanctions_list_update_lag_seconds` | Gauge | Time since last list update | < 86400 (24h) |
| `sanctions_cache_hit_ratio` | Gauge | Cache hit percentage | > 95% |
| `sanctions_hit_queue_depth` | Gauge | Pending review queue | < 500 |
| `sanctions_hit_sla_compliance` | Gauge | Hits reviewed within SLA | > 99% |
| `sanctions_analyst_workload` | Gauge | Hits per analyst per hour | < 50 |
| `sanctions_false_positive_rate` | Gauge | False positives percentage | < 5% |
| `sanctions_name_match_latency_ms` | Histogram | Name matching latency | p99 < 20ms |
| `sanctions_embargo_check_latency_ms` | Histogram | Embargo check latency | p99 < 5ms |
| `sanctions_db_connection_pool_usage` | Gauge | Database pool utilization | < 80% |
| `sanctions_kafka_consumer_lag` | Gauge | Kafka consumer lag | < 1000 |

### 13.2 Service Level Objectives (SLOs)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                             SERVICE LEVEL OBJECTIVES                                 │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  SLO-1: Availability                                                         │   │
│  │  ────────────────────────────────────────────────────────────────────────   │   │
│  │  • Target: 99.99% monthly                                                   │   │
│  │  • Measurement: Successful responses / Total requests                       │   │
│  │  • Error Budget: 4.32 minutes/month                                         │   │
│  │  • Burn Rate Alert: 14.4x (1 min budget) / 6x (10% budget)                 │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  SLO-2: Latency (Real-time Screening)                                        │   │
│  │  ────────────────────────────────────────────────────────────────────────   │   │
│  │  • Target: 99.5% of requests < 150ms                                        │   │
│  │  • Measurement: p99.5 latency bucket                                        │   │
│  │  • Error Budget: 0.5% of requests can exceed                                │   │
│  │  • Alert Threshold: p99 > 200ms for 5 minutes                              │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  SLO-3: Detection Accuracy                                                   │   │
│  │  ────────────────────────────────────────────────────────────────────────   │   │
│  │  • Target: 100% true positive detection                                      │   │
│  │  • Measurement: Known hits detected / Known hits total                       │   │
│  │  • Error Budget: 0 (Zero tolerance for missed hits)                         │   │
│  │  • Alert: Any regression test failure                                       │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  SLO-4: List Currency                                                        │   │
│  │  ────────────────────────────────────────────────────────────────────────   │   │
│  │  • Target: All lists updated within 24 hours of publication                 │   │
│  │  • Measurement: Update completion time - Publication time                   │   │
│  │  • Error Budget: No list older than 24 hours                                │   │
│  │  • Alert: List age > 20 hours                                               │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  SLO-5: Hit Review SLA                                                       │   │
│  │  ────────────────────────────────────────────────────────────────────────   │   │
│  │  • Target: 99% of hits reviewed within SLA                                  │   │
│  │  • Measurement: Hits reviewed on time / Total hits                          │   │
│  │  • Error Budget: 1% can exceed SLA                                          │   │
│  │  • Alert: Queue depth > 100 or approaching SLA breach                       │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 13.3 Grafana Dashboards

#### 13.3.1 Operations Dashboard

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    SANCTIONS SCREENING - OPERATIONS DASHBOARD                        │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐   │
│  │  Availability   │ │  Throughput     │ │  Latency (p99)  │ │  Error Rate     │   │
│  │    99.995%      │ │    4,523 TPS    │ │      78ms       │ │     0.02%       │   │
│  │   ▲ +0.003%     │ │   ▲ +12%       │ │   ▼ -5ms       │ │   ▼ -0.01%      │   │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘ └─────────────────┘   │
│                                                                                     │
│  Screening Volume (24h)                  │  Latency Distribution                    │
│  ┌───────────────────────────────────┐  │  ┌───────────────────────────────────┐   │
│  │ ▲                                 │  │  │                         p99 ─────│   │
│  │ │    ╭──────╮         ╭───╮       │  │  │                    p95 ────────│   │
│  │ │   ╱      ╲       ╭─╯   ╰──╮     │  │  │              p50 ─────────────│   │
│  │ │  ╱        ╲     ╱          ╲    │  │  │ ─────────────────────────────│   │
│  │ │ ╱          ╲───╯            ╲   │  │  │                               │   │
│  │ └─────────────────────────────────│  │  └─────────────────────────────────│   │
│  │   00:00  06:00  12:00  18:00  24:00│  │   0    50    100   150   200ms    │   │
│  └───────────────────────────────────┘  └───────────────────────────────────────┘   │
│                                                                                     │
│  Hit Distribution by List             │  Decision Breakdown                         │
│  ┌───────────────────────────────────┐│  ┌───────────────────────────────────┐     │
│  │ OFAC_SDN        ████████ 45%     ││  │ CLEAR        ████████████████ 94.2%│     │
│  │ EU_CONSOL       ██████ 28%       ││  │ REVIEW       ████ 5.1%             │     │
│  │ UN_CONSOL       ████ 15%         ││  │ BLOCK        █ 0.7%                │     │
│  │ UK_SANCTIONS    ██ 8%            ││  │                                    │     │
│  │ OTHER           █ 4%             ││  │                                    │     │
│  └───────────────────────────────────┘│  └───────────────────────────────────┘     │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

#### 13.3.2 Analyst Dashboard

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    SANCTIONS SCREENING - ANALYST DASHBOARD                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐   │
│  │  Queue Depth    │ │  SLA Compliance │ │  Avg Review Time│ │  Open Escalations│  │
│  │      127        │ │     98.7%       │ │    12.5 min     │ │       8          │  │
│  │   ▲ +23        │ │   ▼ -0.2%      │ │   ▼ -2.1min    │ │   ▲ +3          │  │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘ └─────────────────┘   │
│                                                                                     │
│  Queue by Priority                      │  Hits by Age                              │
│  ┌───────────────────────────────────┐  │  ┌───────────────────────────────────┐   │
│  │ 🔴 CRITICAL     ███ 12           │  │  │ < 1 hour      ██████████████ 67   │   │
│  │ 🟠 HIGH         ██████ 35         │  │  │ 1-4 hours     ████████ 38         │   │
│  │ 🟡 MEDIUM       ████████████ 58   │  │  │ 4-8 hours     ████ 15             │   │
│  │ 🟢 LOW          █████ 22          │  │  │ 8-24 hours    ██ 7                │   │
│  │                                   │  │  │ > 24 hours    ⚠ 0                 │   │
│  └───────────────────────────────────┘  │  └───────────────────────────────────┘   │
│                                                                                     │
│  Analyst Performance (Today)                                                        │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │ Analyst           │ Reviewed │ Cleared │ Confirmed │ Avg Time │ SLA %     │   │
│  │───────────────────────────────────────────────────────────────────────────│   │
│  │ john.smith        │    48    │   45    │     3     │   8.2m   │  100%     │   │
│  │ jane.doe          │    42    │   38    │     4     │  11.5m   │   97%     │   │
│  │ bob.wilson        │    35    │   33    │     2     │  15.3m   │   94%     │   │
│  │ alice.johnson     │    52    │   49    │     3     │   7.8m   │  100%     │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 13.4 Alert Configuration

#### 13.4.1 Critical Alerts (P1 - Immediate Response)

| Alert Name | Condition | Duration | Action |
|------------|-----------|----------|--------|
| `SanctionsServiceDown` | availability < 95% | 1 min | Page on-call, auto-failover |
| `SanctionsScreeningError` | error_rate > 5% | 2 min | Page on-call |
| `SanctionsListOutdated` | list_age > 24 hours | Immediate | Page compliance + on-call |
| `SanctionsTruePositiveMissed` | regression_test_fail | Immediate | Page engineering + compliance |
| `SanctionsDBConnectionFailed` | db_connections = 0 | 30 sec | Page on-call |

#### 13.4.2 High Alerts (P2 - < 30 min response)

| Alert Name | Condition | Duration | Action |
|------------|-----------|----------|--------|
| `SanctionsLatencyHigh` | p99 > 200ms | 5 min | Notify on-call |
| `SanctionsQueueBacklog` | queue_depth > 500 | 15 min | Notify analysts + manager |
| `SanctionsSLAAtRisk` | sla_compliance < 95% | 10 min | Notify compliance |
| `SanctionsCacheFailure` | cache_hit_rate < 50% | 5 min | Notify on-call |
| `SanctionsKafkaLag` | consumer_lag > 10000 | 10 min | Notify on-call |

#### 13.4.3 Medium Alerts (P3 - < 4 hour response)

| Alert Name | Condition | Duration | Action |
|------------|-----------|----------|--------|
| `SanctionsFalsePositiveHigh` | fp_rate > 10% | 1 hour | Notify tuning team |
| `SanctionsListUpdateSlow` | update_time > 30 min | - | Notify operations |
| `SanctionsDiskSpaceLow` | disk_usage > 80% | - | Create ticket |
| `SanctionsConnectionPoolHigh` | pool_usage > 90% | 30 min | Notify operations |

#### 13.4.4 Alertmanager Configuration

```yaml
groups:
  - name: sanctions-critical
    rules:
      - alert: SanctionsServiceDown
        expr: sum(up{job="sanctions-screening"}) == 0
        for: 1m
        labels:
          severity: critical
          team: compliance-engineering
        annotations:
          summary: "Sanctions Screening service is down"
          description: "All sanctions screening instances are unavailable"
          runbook_url: "https://runbooks.internal/sanctions/service-down"
          
      - alert: SanctionsListOutdated
        expr: sanctions_list_update_age_seconds > 86400
        for: 0m
        labels:
          severity: critical
          team: compliance
        annotations:
          summary: "Sanctions list is outdated (> 24 hours)"
          description: "List {{ $labels.list_code }} has not been updated for {{ $value | humanizeDuration }}"
          runbook_url: "https://runbooks.internal/sanctions/list-outdated"

  - name: sanctions-high
    rules:
      - alert: SanctionsLatencyHigh
        expr: histogram_quantile(0.99, sum(rate(sanctions_screening_latency_seconds_bucket[5m])) by (le)) > 0.2
        for: 5m
        labels:
          severity: high
          team: compliance-engineering
        annotations:
          summary: "Sanctions screening latency is high"
          description: "p99 latency is {{ $value | humanizeDuration }}"
          
      - alert: SanctionsQueueBacklog
        expr: sanctions_hit_queue_depth > 500
        for: 15m
        labels:
          severity: high
          team: compliance-operations
        annotations:
          summary: "Sanctions hit review queue backlog"
          description: "Queue depth: {{ $value }} hits pending review"
```

### 13.5 Logging Strategy

| Log Level | Content | Retention | Example |
|-----------|---------|-----------|---------|
| ERROR | Failures, exceptions, compliance events | 1 year | List load failure, DB errors |
| WARN | Degraded performance, near-threshold | 90 days | High latency, retry needed |
| INFO | Business events, decisions | 90 days | Screening completed, hit created |
| DEBUG | Detailed processing | 7 days | Match scores, algorithm steps |
| TRACE | Full data dumps | 1 day (dev only) | Request/response bodies |

#### Structured Log Format

```json
{
  "timestamp": "2026-02-20T10:30:45.123Z",
  "level": "INFO",
  "service": "sanctions-screening",
  "version": "2.1.0",
  "traceId": "abc123def456",
  "spanId": "789xyz",
  "correlationId": "corr-uuid-12345",
  "event": "SCREENING_COMPLETED",
  "transactionId": "TXN-2026-0220-001234",
  "decision": "CLEAR",
  "hitCount": 0,
  "latencyMs": 87,
  "listsScreened": 4,
  "partiesScreened": 3,
  "userId": null,
  "clientIp": "10.0.1.25",
  "message": "Transaction screening completed successfully"
}
```

---

## 14. Test Strategy

### 14.1 Test Pyramid

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              TEST PYRAMID                                            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│                              ▲                                                      │
│                             ╱ ╲                                                     │
│                            ╱   ╲     E2E / Compliance                              │
│                           ╱ 5%  ╲    (Regulatory Scenarios)                        │
│                          ╱───────╲                                                  │
│                         ╱         ╲                                                 │
│                        ╱           ╲   Integration Tests                           │
│                       ╱    15%     ╲   (API, Database, Kafka)                      │
│                      ╱───────────────╲                                             │
│                     ╱                 ╲                                             │
│                    ╱                   ╲   Component Tests                         │
│                   ╱        25%         ╲   (Service Layer)                         │
│                  ╱───────────────────────╲                                          │
│                 ╱                         ╲                                         │
│                ╱                           ╲  Unit Tests                            │
│               ╱           55%              ╲  (Business Logic)                     │
│              ╱─────────────────────────────────╲                                    │
│                                                                                     │
│  Total Test Coverage Target: > 90% (Lines), > 85% (Branches), > 95% (Critical)    │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 14.2 Unit Testing

| Test Category | Scope | Examples | Coverage Target |
|---------------|-------|----------|-----------------|
| Name Matching | Algorithm accuracy | Exact match, fuzzy, phonetic, transliteration | 100% |
| Score Calculation | Scoring logic | Weight calculation, threshold evaluation | 100% |
| Decision Engine | Decision rules | Clear/Review/Block logic, embargo rules | 100% |
| Validators | Input validation | Party data, list format, configuration | 100% |
| Domain Model | Entity behavior | State transitions, invariants | 95% |
| Event Builders | Event creation | Schema compliance, field mapping | 95% |

### 14.3 Integration Testing

| Test Category | Scope | Examples |
|---------------|-------|----------|
| API Integration | REST/gRPC endpoints | Request/response, error handling |
| Database | Repository operations | CRUD, queries, transactions |
| Cache | Redis operations | Cache hit/miss, TTL, cluster |
| Kafka | Event production/consumption | Message format, partitioning |
| External Lists | List provider integration | Download, parse, validate |
| Payment Gateway | Core integration | Screening requests, callbacks |

### 14.4 Compliance Testing

#### 14.4.1 Regulatory Test Scenarios

| Test ID | Scenario | Expected Result | Regulatory Basis |
|---------|----------|-----------------|------------------|
| COMP-001 | Known SDN listed individual | BLOCK decision | OFAC SDN |
| COMP-002 | Known EU sanctioned entity | BLOCK decision | EU Regs |
| COMP-003 | Embargoed country payment | BLOCK decision | OFAC Embargo |
| COMP-004 | 50% rule entity (subsidiary) | HIT for review | OFAC 50% Rule |
| COMP-005 | Delisted individual | CLEAR decision | Post-delisting |
| COMP-006 | Name variation (Cyrillic) | HIT detected | Multi-script |
| COMP-007 | Alias match | HIT detected | FATF guidelines |
| COMP-008 | Dual nationality | Both countries screened | MAS/OFAC |
| COMP-009 | Historic match (delisted) | CLEAR (current list) | List currency |
| COMP-010 | Sanctioned vessel | HIT detected | OFAC SDN |

#### 14.4.2 Regression Test Suite

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          REGRESSION TEST SUITE                                       │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  Golden Dataset:                                                                    │
│  ─────────────────────────────────────────────────────────────────────────────     │
│  • 10,000+ known sanctioned names (multi-source validated)                         │
│  • 5,000+ name variations (aliases, transliterations, misspellings)                │
│  • 3,000+ known false positives (business-approved clearances)                     │
│  • 500+ edge cases (partial matches, similar names)                                │
│                                                                                     │
│  Execution:                                                                         │
│  ─────────────────────────────────────────────────────────────────────────────     │
│  • Run on every deployment (blocking)                                              │
│  • Run daily against production data (monitoring)                                  │
│  • Run weekly with expanded dataset                                                │
│                                                                                     │
│  Pass Criteria:                                                                     │
│  ─────────────────────────────────────────────────────────────────────────────     │
│  • 100% detection rate for known sanctioned names                                  │
│  • 100% detection rate for all name variations                                     │
│  • No regression in false positive rate (deviation > 5% = alert)                  │
│  • All edge cases produce expected results                                         │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 14.5 Performance Testing

| Test Type | Objective | Tool | Criteria |
|-----------|-----------|------|----------|
| Load Test | Sustained capacity | k6 / Gatling | 8,000 TPS for 1 hour |
| Stress Test | Breaking point | k6 / Gatling | Identify max TPS |
| Soak Test | Memory leaks | k6 | 24 hours at 70% capacity |
| Spike Test | Auto-scaling | k6 | 0 to 10,000 TPS in 1 minute |
| Latency Test | Response time | Locust | p99 < 150ms at load |
| Batch Test | Bulk processing | Custom | 100,000 records/hour |

#### Performance Test Profile

```yaml
scenarios:
  real_time_screening:
    executor: ramping-arrival-rate
    startRate: 0
    timeUnit: 1s
    stages:
      - target: 1000
        duration: 5m
      - target: 5000
        duration: 10m
      - target: 8000
        duration: 30m
      - target: 8000
        duration: 60m
      - target: 0
        duration: 5m
    
  batch_upload:
    executor: per-vu-iterations
    vus: 10
    iterations: 100
    maxDuration: 2h
    
thresholds:
  http_req_duration:
    - p(50) < 50
    - p(95) < 100
    - p(99) < 150
  http_req_failed:
    - rate < 0.001
  iterations:
    - rate > 8000
```

### 14.6 Security Testing

| Test Type | Scope | Frequency | Tools |
|-----------|-------|-----------|-------|
| SAST | Code vulnerabilities | Every commit | SonarQube, Checkmarx |
| DAST | Runtime vulnerabilities | Weekly | OWASP ZAP |
| Dependency Scan | Library CVEs | Daily | Snyk, Dependabot |
| Container Scan | Image vulnerabilities | Every build | Trivy, Clair |
| Penetration Test | Full security assessment | Quarterly | External firm |
| API Security | API-specific attacks | Weekly | Burp Suite |

### 14.7 Test Automation

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          CI/CD TEST AUTOMATION                                       │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  Pull Request Gate:                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  □ Unit Tests (< 3 min)                                                     │   │
│  │  □ Component Tests (< 5 min)                                                │   │
│  │  □ SAST Scan (< 10 min)                                                     │   │
│  │  □ Code Coverage Check (> 90%)                                              │   │
│  │  □ Lint / Format Check                                                      │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  Merge to Main:                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  □ All PR checks                                                            │   │
│  │  □ Integration Tests (< 15 min)                                             │   │
│  │  □ Container Build + Scan                                                   │   │
│  │  □ Compliance Regression Suite                                              │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  Deploy to Staging:                                                                │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  □ All previous checks                                                      │   │
│  │  □ E2E Tests (< 30 min)                                                     │   │
│  │  □ Performance Smoke Test (< 15 min)                                        │   │
│  │  □ Security Scan (DAST)                                                     │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  Deploy to Production:                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  □ All previous checks                                                      │   │
│  │  □ Compliance Team Sign-off                                                 │   │
│  │  □ Security Team Sign-off (if security-relevant changes)                   │   │
│  │  □ Canary Deployment (5% traffic)                                          │   │
│  │  □ Full Rollout (with automated rollback)                                  │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 15. Implementation Roadmap

### 15.1 Phase Overview

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          IMPLEMENTATION ROADMAP                                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  Phase 1: Foundation (Weeks 1-8)                                                    │
│  ════════════════════════════════════════════════════════════════════════════════  │
│  │███████████████████████████████████████████████████████████████████████████│     │
│                                                                                     │
│  Phase 2: Core Screening (Weeks 9-16)                                              │
│  ════════════════════════════════════════════════════════════════════════════════  │
│          │███████████████████████████████████████████████████████████████████│     │
│                                                                                     │
│  Phase 3: Hit Management (Weeks 17-22)                                             │
│  ════════════════════════════════════════════════════════════════════════════════  │
│                  │███████████████████████████████████████████████│                 │
│                                                                                     │
│  Phase 4: Advanced Features (Weeks 23-28)                                          │
│  ════════════════════════════════════════════════════════════════════════════════  │
│                          │███████████████████████████████████████│                 │
│                                                                                     │
│  Phase 5: Production Hardening (Weeks 29-32)                                       │
│  ════════════════════════════════════════════════════════════════════════════════  │
│                                      │██████████████████████████│                   │
│                                                                                     │
│  Timeline:  W1   W4   W8   W12   W16   W20   W24   W28   W32                      │
│            └────┴────┴────┴─────┴─────┴─────┴─────┴─────┴─────┘                   │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 15.2 Phase 1: Foundation (Weeks 1-8)

| Week | Deliverable | Dependencies | Team |
|------|-------------|--------------|------|
| 1-2 | Project setup, CI/CD pipeline | ADR approval | Platform |
| 1-2 | Database schema design & setup | DBA review | Data |
| 3-4 | Domain model implementation | Schema complete | Backend |
| 3-4 | API interface design (OpenAPI/gRPC) | Design review | Backend |
| 5-6 | List Management Service (basic) | External provider setup | Backend |
| 5-6 | OFAC/EU/UN list parsers | List format specs | Backend |
| 7-8 | Name matching algorithm (exact + fuzzy) | NLP library selection | Backend |
| 7-8 | Redis cache setup | Infrastructure | Platform |

**Phase 1 Exit Criteria:**
- [ ] Service skeleton deployed to development
- [ ] Database schema created and migrated
- [ ] Basic list import working (OFAC SDN)
- [ ] Exact name matching functional
- [ ] CI/CD pipeline operational

### 15.3 Phase 2: Core Screening (Weeks 9-16)

| Week | Deliverable | Dependencies | Team |
|------|-------------|--------------|------|
| 9-10 | Phonetic matching (Soundex, Metaphone) | Phase 1 complete | Backend |
| 9-10 | Transliteration support | Unicode libraries | Backend |
| 11-12 | Full screening workflow | Matching complete | Backend |
| 11-12 | Elasticsearch integration | Infra setup | Platform |
| 13-14 | Scoring & threshold engine | Business rules | Backend |
| 13-14 | Embargo checking | Country data | Backend |
| 15-16 | Payment Gateway integration | Core team coordination | Integration |
| 15-16 | Event production (Kafka) | Kafka setup | Backend |

**Phase 2 Exit Criteria:**
- [ ] Full name matching suite operational
- [ ] Real-time screening < 150ms p99
- [ ] Integration with Payment Gateway functional
- [ ] All standard lists supported
- [ ] Events published to Kafka

### 15.4 Phase 3: Hit Management (Weeks 17-22)

| Week | Deliverable | Dependencies | Team |
|------|-------------|--------------|------|
| 17-18 | Hit creation & queue system | Phase 2 complete | Backend |
| 17-18 | Analyst UI (basic) | UX design | Frontend |
| 19-20 | Disposition workflow | Compliance review | Backend |
| 19-20 | Review assignment | Business rules | Backend |
| 21-22 | Escalation workflow | Management approval | Backend |
| 21-22 | SLA monitoring & alerts | Metrics setup | Backend |

**Phase 3 Exit Criteria:**
- [ ] Hits created for matches
- [ ] Analyst can review and disposition
- [ ] Escalation workflow functional
- [ ] SLA tracking operational
- [ ] Audit trail complete

### 15.5 Phase 4: Advanced Features (Weeks 23-28)

| Week | Deliverable | Dependencies | Team |
|------|-------------|--------------|------|
| 23-24 | Batch screening | Core screening | Backend |
| 23-24 | List delta updates | List management | Backend |
| 25-26 | False positive management | Hit management | Backend |
| 25-26 | Threshold tuning interface | Analytics | Frontend |
| 27-28 | Reporting & analytics | Data warehouse | BI |
| 27-28 | Periodic re-screening | Batch processing | Backend |

**Phase 4 Exit Criteria:**
- [ ] Batch processing 100,000/hour
- [ ] Delta updates < 5 minutes
- [ ] False positive suppression operational
- [ ] Management reports available
- [ ] Re-screening automated

### 15.6 Phase 5: Production Hardening (Weeks 29-32)

| Week | Deliverable | Dependencies | Team |
|------|-------------|--------------|------|
| 29-30 | Performance optimization | Load testing | Backend |
| 29-30 | Security hardening | Pen test results | Security |
| 31-32 | DR setup & testing | Infrastructure | Platform |
| 31-32 | Compliance certification | Audit complete | Compliance |
| 32 | Production deployment | All gates passed | All |

**Phase 5 Exit Criteria:**
- [ ] Performance targets met (8,000 TPS, <150ms p99)
- [ ] Security audit passed
- [ ] DR tested (RPO < 1 min, RTO < 15 min)
- [ ] Compliance sign-off obtained
- [ ] Production deployment complete

### 15.7 Resource Requirements

| Role | Phase 1 | Phase 2 | Phase 3 | Phase 4 | Phase 5 |
|------|---------|---------|---------|---------|---------|
| Tech Lead | 1 | 1 | 1 | 1 | 1 |
| Backend Engineers | 3 | 4 | 3 | 3 | 2 |
| Frontend Engineers | 0 | 0 | 2 | 2 | 1 |
| Platform Engineers | 2 | 1 | 1 | 1 | 2 |
| QA Engineers | 1 | 2 | 2 | 2 | 2 |
| Security Engineer | 0.5 | 0.5 | 0.5 | 0.5 | 1 |
| Compliance Analyst | 0.5 | 1 | 2 | 2 | 2 |
| **Total FTE** | **8** | **9.5** | **11.5** | **11.5** | **11** |

### 15.8 Risk Register

| Risk ID | Risk | Probability | Impact | Mitigation |
|---------|------|-------------|--------|------------|
| R-001 | List provider API changes | Medium | High | Abstract provider layer, monitor changes |
| R-002 | Performance targets not met | Medium | High | Early load testing, architecture review |
| R-003 | High false positive rate | High | Medium | Iterative tuning, ML consideration |
| R-004 | Regulatory requirement changes | Medium | High | Configurable rules, compliance monitoring |
| R-005 | Integration delays with Payment Gateway | Medium | High | Early integration, mock services |
| R-006 | Skilled resource availability | Medium | Medium | Cross-training, documentation |
| R-007 | Third-party dependency vulnerability | Medium | High | Regular scanning, update process |
| R-008 | Multi-script matching accuracy | Medium | Medium | Linguistic expertise, external validation |

### 15.9 Success Metrics

| Metric | Phase 1 | Phase 2 | Phase 3 | Phase 4 | Phase 5 |
|--------|---------|---------|---------|---------|---------|
| Code Coverage | 80% | 85% | 90% | 90% | 90% |
| API Latency (p99) | - | < 300ms | < 200ms | < 150ms | < 150ms |
| Throughput | - | 1,000 TPS | 5,000 TPS | 8,000 TPS | 8,000 TPS |
| List Currency | Daily | Daily | Daily | < 4 hours | < 4 hours |
| Match Accuracy | 95% | 99% | 99.5% | 99.9% | 99.9% |
| Hit Review SLA | - | - | 95% | 98% | 99% |
| Availability | 99% | 99.9% | 99.95% | 99.99% | 99.99% |

---

## Appendix A: Glossary

| Term | Definition |
|------|------------|
| **Sanctions** | Restrictive measures imposed by governments to change behavior of target countries, regimes, or individuals |
| **SDN** | Specially Designated Nationals - Persons and entities designated by OFAC |
| **OFAC** | Office of Foreign Assets Control - US Treasury agency administering sanctions |
| **Embargo** | Comprehensive trade and financial restrictions against a country |
| **Hit** | Potential match between screened party and sanctioned entity |
| **False Positive** | Match that appears to be a hit but is determined not to be the sanctioned entity |
| **True Positive** | Confirmed match with sanctioned entity |
| **Fuzzy Matching** | Approximate string matching to find similar but not exact names |
| **Phonetic Matching** | Matching based on pronunciation rather than spelling |
| **Disposition** | Decision made on a hit (clear, confirm, escalate) |
| **50% Rule** | OFAC rule that entities 50%+ owned by SDN are also sanctioned |

## Appendix B: Related Documents

| Document | Location | Description |
|----------|----------|-------------|
| AML Screening Design | [Design_AML_Screening.md](Design_AML_Screening.md) | Anti-Money Laundering module design |
| Fraud Detection Design | [Design_Fraud_Detection.md](Design_Fraud_Detection.md) | Fraud detection module design |
| Payment Gateway Core | [../PaymentOrchestrationDomain/Design_Payment_Gateway_Core.md](../PaymentOrchestrationDomain/Design_Payment_Gateway_Core.md) | Core payment gateway design |
| Context Overview | [../PaymentGateway_Context_Overview.md](../PaymentGateway_Context_Overview.md) | Overall system context |

## Appendix C: Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2026-02-20 | Architecture Team | Initial release |

---

*Document generated following Payment Hub design standards and ADR-005a guidelines.*
