# AML Screening (Anti-Money Laundering) Module
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

The **AML Screening (Anti-Money Laundering)** module is a critical component of the Risk & Compliance Domain responsible for detecting, preventing, and reporting potential money laundering activities within payment transactions. This module implements comprehensive screening capabilities against sanctions lists, PEP (Politically Exposed Persons) databases, adverse media sources, and configurable risk rules to ensure regulatory compliance across multiple jurisdictions.

### Key Capabilities

| Capability | Description |
|------------|-------------|
| Sanctions List Screening | Real-time screening against OFAC, EU, UN, and country-specific sanctions lists |
| PEP Screening | Detection of Politically Exposed Persons and their associates |
| Watchlist Management | Centralized management of internal and external watchlists |
| Name Matching Engine | Fuzzy matching with configurable algorithms (Jaro-Winkler, Soundex, Levenshtein) |
| Transaction Pattern Analysis | Detection of structuring, layering, and suspicious patterns |
| Risk Scoring Engine | Multi-factor risk score calculation for transactions and parties |
| Case Management Integration | Automated case creation for investigation workflows |
| Regulatory Reporting | SAR/STR generation and filing automation |
| Real-Time & Batch Screening | Support for synchronous transaction screening and batch file processing |
| Audit Trail | Complete compliance audit logging for regulatory examination |

### Business Value

- **Regulatory Compliance**: Meet AML/CFT requirements across FATF, FinCEN, FCA, and regional regulations
- **Risk Mitigation**: Prevent processing of payments involving sanctioned entities
- **Fraud Prevention**: Detect and block suspicious transaction patterns
- **Operational Efficiency**: Automated screening reduces manual review workload
- **Cost Reduction**: False positive optimization minimizes unnecessary investigations
- **Reputation Protection**: Prevent association with money laundering activities
- **Real-Time Protection**: Sub-second screening enables STP (Straight-Through Processing)
- **Auditability**: Complete screening history for regulatory examinations

### Architecture Context (ADR-005a)

The AML Screening module operates within the **Risk & Compliance Domain** and is triggered by the **Limit Control Engine** within the Payment Orchestration Domain when transactions meet AML screening thresholds:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PAYMENT ORCHESTRATION DOMAIN                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Payment Gateway Core → Validation Pipeline → Limit Control Engine          │
│                                                        │                    │
│                                                        │ (AML Trigger)      │
│                                                        ▼                    │
└────────────────────────────────────────────────────────┼────────────────────┘
                                                         │
                    ┌────────────────────────────────────┼────────────────────┐
                    │              RISK & COMPLIANCE DOMAIN                    │
                    ├────────────────────────────────────┼────────────────────┤
                    │                                    ▼                    │
                    │  ┌────────────────────────────────────────────────────┐ │
                    │  │           AML SCREENING MODULE (This Module)        │ │
                    │  │                                                     │ │
                    │  │   ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │ │
                    │  │   │  Sanctions  │  │     PEP     │  │  Watchlist │ │ │
                    │  │   │  Screening  │  │  Screening  │  │  Screening │ │ │
                    │  │   └─────────────┘  └─────────────┘  └────────────┘ │ │
                    │  │          │               │               │         │ │
                    │  │          └───────────────┼───────────────┘         │ │
                    │  │                          ▼                         │ │
                    │  │   ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │ │
                    │  │   │    Name     │  │Transaction  │  │    Risk    │ │ │
                    │  │   │   Matching  │  │  Patterns   │  │   Scoring  │ │ │
                    │  │   └─────────────┘  └─────────────┘  └────────────┘ │ │
                    │  │                          │                         │ │
                    │  │                          ▼                         │ │
                    │  │   ┌─────────────────────────────────────────────┐ │ │
                    │  │   │         SCREENING DECISION ENGINE           │ │ │
                    │  │   │  • PASS (Clear)  • REVIEW (Alert)  • BLOCK  │ │ │
                    │  │   └─────────────────────────────────────────────┘ │ │
                    │  │          │               │               │         │ │
                    │  │          ▼               ▼               ▼         │ │
                    │  │      Continue       Case Mgmt         Reject       │ │
                    │  │      Payment        Alert Queue       Payment      │ │
                    │  │                                                     │ │
                    │  └─────────────────────────────────────────────────────┘ │
                    │                                                          │
                    │  ┌──────────────────┐  ┌──────────────────────────────┐ │
                    │  │  Fraud Detection │  │  Regulatory Reporting (SAR)  │ │
                    │  └──────────────────┘  └──────────────────────────────┘ │
                    │                                                          │
                    └──────────────────────────────────────────────────────────┘
```

### Performance Metrics

| Metric | Target | Description |
|--------|--------|-------------|
| Screening Latency (p99) | < 200ms | End-to-end screening response time |
| Name Match Latency (p99) | < 50ms | Single name matching operation time |
| Sanctions Check (p99) | < 100ms | Sanctions list lookup time |
| PEP Check (p99) | < 80ms | PEP database lookup time |
| Risk Score Calculation | < 30ms | Multi-factor risk scoring time |
| Throughput | > 5,000 TPS | Screening checks per second |
| List Update Latency | < 5 min | Time to propagate list updates |
| False Positive Rate | < 3% | Target false positive percentage |
| True Positive Detection | > 99.9% | Sanctioned entity detection rate |

### Regulatory Compliance Coverage

| Jurisdiction | Regulation | Coverage |
|--------------|------------|----------|
| Global | FATF Recommendations | Full |
| USA | Bank Secrecy Act (BSA), OFAC | Full |
| USA | FinCEN AML Requirements | Full |
| EU | 6th AML Directive (6AMLD) | Full |
| UK | Money Laundering Regulations 2017 | Full |
| Singapore | MAS Notice 626 | Full |
| Hong Kong | AMLO | Full |
| Australia | AML/CTF Act | Full |

---

## 2. Module Overview

### 2.1 Bounded Context

The AML Screening module belongs to the **Risk & Compliance Domain** bounded context and operates as a dedicated **AML Screening Service** (Service Port: 8003).

### 2.2 Module Scope

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AML SCREENING MODULE                                  │
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
│  │  SANCTIONS      │  │  PEP SCREENING     │  │  WATCHLIST SCREENING    │ │
│  │  SCREENING      │  │                    │  │                         │ │
│  │                 │  │ • PEP Database     │  │ • Internal Watchlist    │ │
│  │ • OFAC SDN      │  │ • Family/Associate │  │ • Customer Blacklist    │ │
│  │ • EU Sanctions  │  │ • RCA Detection    │  │ • Negative News         │ │
│  │ • UN Sanctions  │  │ • SOE Detection    │  │ • Industry Watchlist    │ │
│  │ • UK Sanctions  │  │ • Risk Categories  │  │ • Custom Lists          │ │
│  │ • Country Lists │  │                    │  │                         │ │
│  │                 │  │ Sources:           │  │ Sources:                │ │
│  │ Sources:        │  │ • World-Check      │  │ • Internal DB           │ │
│  │ • OFAC API      │  │ • Dow Jones        │  │ • Partner Feeds         │ │
│  │ • EU API        │  │ • LexisNexis       │  │ • Manual Entries        │ │
│  │ • Downloaded    │  │ • Custom PEP DB    │  │ • Regulatory Feeds      │ │
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
│  │  │ • Tokenized     │  │ • Jaccard       │  │ • Double Metaphone  │  │   │
│  │  │ • Alias Match   │  │ • N-Gram        │  │ • NYSIIS            │  │   │
│  │  │                 │  │ • Edit Distance │  │ • Soundex (Arabic)  │  │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
│  │                                                                      │   │
│  │  Configuration: Match Thresholds, Algorithm Weights, Name Parsing    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  TRANSACTION PATTERN ANALYSIS                        │   │
│  │                                                                      │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │   │
│  │  │ Structuring     │  │ Velocity        │  │ Behavioral          │  │   │
│  │  │ Detection       │  │ Analysis        │  │ Analysis            │  │   │
│  │  │                 │  │                 │  │                     │  │   │
│  │  │ • Threshold     │  │ • Frequency     │  │ • Deviation from    │  │   │
│  │  │   Avoidance     │  │   Patterns      │  │   Normal Pattern    │  │   │
│  │  │ • Round-Trip    │  │ • Volume Spikes │  │ • New Beneficiaries │  │   │
│  │  │ • Split Trans   │  │ • Time Patterns │  │ • Geographic Risk   │  │   │
│  │  │ • Layering      │  │ • Dormant Wake  │  │ • Industry Risk     │  │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  RISK SCORING ENGINE                                 │   │
│  │                                                                      │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │   │
│  │  │ Customer Risk   │  │ Transaction     │  │ Geographic Risk     │  │   │
│  │  │ Factors         │  │ Risk Factors    │  │ Factors             │  │   │
│  │  │                 │  │                 │  │                     │  │   │
│  │  │ • Customer Type │  │ • Amount        │  │ • Country Risk      │  │   │
│  │  │ • Occupation    │  │ • Payment Type  │  │ • Jurisdiction      │  │   │
│  │  │ • Source Funds  │  │ • Channel       │  │ • Corridor Risk     │  │   │
│  │  │ • Tenure        │  │ • Beneficiary   │  │ • Tax Haven         │  │   │
│  │  │ • KYC Status    │  │ • Frequency     │  │ • High-Risk Region  │  │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
│  │                                                                      │   │
│  │  Output: Composite Risk Score (0-100) with Factor Breakdown          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  SUPPORTING COMPONENTS                               │   │
│  │                                                                      │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │   │
│  │  │ List        │  │ Match       │  │ Alert       │  │ Audit       │ │   │
│  │  │ Manager     │  │ Cache       │  │ Generator   │  │ Logger      │ │   │
│  │  │             │  │             │  │             │  │             │ │   │
│  │  │ • Load      │  │ • Store     │  │ • Create    │  │ • Screen    │ │   │
│  │  │ • Update    │  │ • Lookup    │  │ • Priority  │  │ • Match     │ │   │
│  │  │ • Version   │  │ • Invalidate│  │ • Route     │  │ • Decision  │ │   │
│  │  │ • Distribute│  │ • TTL       │  │ • Escalate  │  │ • Alert     │ │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘ │   │
│  │                                                                      │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │   │
│  │  │ Case        │  │ SAR/STR     │  │ Metrics     │  │ Config      │ │   │
│  │  │ Connector   │  │ Generator   │  │ Publisher   │  │ Manager     │ │   │
│  │  │             │  │             │  │             │  │             │ │   │
│  │  │ • Create    │  │ • Generate  │  │ • Counters  │  │ • Rules     │ │   │
│  │  │ • Link      │  │ • Validate  │  │ • Latency   │  │ • Thresholds│ │   │
│  │  │ • Update    │  │ • Submit    │  │ • Hit Rates │  │ • Algorithms│ │   │
│  │  │ • Close     │  │ • Track     │  │ • Dashboard │  │ • Weights   │ │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘ │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 Module Responsibilities

| Responsibility | Description |
|----------------|-------------|
| Sanctions Screening | Screen parties against global sanctions lists (OFAC, EU, UN, UK) |
| PEP Detection | Identify politically exposed persons and their associates |
| Watchlist Matching | Check against internal and external watchlists |
| Name Matching | Apply fuzzy, phonetic, and exact matching algorithms |
| Pattern Detection | Identify suspicious transaction patterns (structuring, layering) |
| Risk Scoring | Calculate composite risk scores for transactions and parties |
| Hit Resolution | Provide match details for analyst review |
| Alert Generation | Create alerts for potential matches requiring investigation |
| Case Creation | Integrate with case management for investigation workflows |
| SAR Generation | Generate Suspicious Activity Reports for regulatory filing |
| List Management | Maintain and update screening lists from multiple sources |
| Audit Logging | Record complete screening history for compliance |

### 2.4 Screening Flow Design

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AML SCREENING FLOW                                        │
└─────────────────────────────────────────────────────────────────────────────┘

    ScreeningRequest (from Limit Control Engine / Batch Processor)
                │
                ▼
    ┌───────────────────────────────────────────────────────────────────┐
    │                     SCREENING ORCHESTRATOR                         │
    │                                                                    │
    │   screenTransaction(ScreeningRequest request)                      │
    │   {                                                                │
    │       // Step 1: Extract screening subjects                        │
    │       List<Subject> subjects = extractSubjects(request);          │
    │       // - Originator (name, address, DOB, nationality)           │
    │       // - Beneficiary (name, address, account)                   │
    │       // - Intermediary banks (BIC, name)                         │
    │       // - Connected parties (directors, owners)                  │
    │                                                                    │
    │       // Step 2: Parallel Screening (fan-out)                      │
    │       CompletableFuture<ScreeningResult>[] futures = {            │
    │           sanctionsScreener.screenAsync(subjects),                │
    │           pepScreener.screenAsync(subjects),                      │
    │           watchlistScreener.screenAsync(subjects),                │
    │           patternAnalyzer.analyzeAsync(request.transaction)       │
    │       };                                                           │
    │                                                                    │
    │       // Step 3: Aggregate Results                                 │
    │       AggregatedResult results = waitAndAggregate(futures);       │
    │                                                                    │
    │       // Step 4: Calculate Risk Score                              │
    │       RiskScore score = riskScorer.calculate(request, results);   │
    │                                                                    │
    │       // Step 5: Make Decision                                     │
    │       ScreeningDecision decision = decisionEngine.evaluate(       │
    │           results, score, request.context                         │
    │       );                                                           │
    │                                                                    │
    │       // Step 6: Post-Decision Actions                             │
    │       if (decision == BLOCK || decision == REVIEW) {              │
    │           alertGenerator.createAlert(request, results, decision); │
    │           caseConnector.createOrUpdateCase(request, results);     │
    │       }                                                            │
    │                                                                    │
    │       // Step 7: Audit & Return                                    │
    │       auditLogger.log(request, results, decision);                │
    │       return buildResponse(decision, results, score);             │
    │   }                                                                │
    │                                                                    │
    └───────────────────────────────────────────────────────────────────┘
                │
                ▼
        ScreeningResponse (to Payment Orchestration / Case Management)
        - Decision: PASS | REVIEW | BLOCK
        - Risk Score: 0-100
        - Match Details: List of potential hits
        - Alert Reference: If alert created
```

### 2.5 Screening Decision Matrix

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SCREENING DECISION MATRIX                                 │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────┬─────────────────────────────────────────────────────────┐
│ Match Type          │ Score Range  → Decision                                 │
├─────────────────────┼─────────────────────────────────────────────────────────┤
│                     │ 100%         → BLOCK (Exact sanctioned entity match)   │
│ SANCTIONS MATCH     │ 90-99%       → BLOCK (High-confidence match)           │
│                     │ 80-89%       → REVIEW (Requires analyst verification)  │
│                     │ <80%         → PASS (Below threshold, log only)        │
├─────────────────────┼─────────────────────────────────────────────────────────┤
│                     │ 100%         → REVIEW (Exact PEP match, enhanced DD)   │
│ PEP MATCH           │ 90-99%       → REVIEW (High-confidence PEP match)      │
│                     │ 80-89%       → REVIEW (Potential PEP, verify)          │
│                     │ <80%         → PASS (Below threshold)                  │
├─────────────────────┼─────────────────────────────────────────────────────────┤
│                     │ 100%         → BLOCK (Exact blacklist match)           │
│ WATCHLIST MATCH     │ 90-99%       → REVIEW (High-confidence match)          │
│                     │ <90%         → PASS (Below threshold)                  │
├─────────────────────┼─────────────────────────────────────────────────────────┤
│                     │ High (>80)   → REVIEW (Suspicious pattern detected)    │
│ PATTERN RISK        │ Medium (50-80)→ FLAG  (Enhanced monitoring)            │
│                     │ Low (<50)    → PASS  (Normal pattern)                  │
├─────────────────────┼─────────────────────────────────────────────────────────┤
│                     │ High (>70)   → REVIEW + Enhanced Due Diligence         │
│ COMPOSITE RISK      │ Medium (40-70)→ PASS + Periodic Review Flag            │
│ SCORE               │ Low (<40)    → PASS  (Standard processing)             │
└─────────────────────┴─────────────────────────────────────────────────────────┘

Decision Precedence: BLOCK > REVIEW > FLAG > PASS

Combined Decision Logic:
┌─────────────────────────────────────────────────────────────────────────────┐
│ IF any_match == SANCTIONS_BLOCK                                             │
│    THEN final_decision = BLOCK, reason = "Sanctions hit"                   │
│ ELSE IF any_match == WATCHLIST_BLOCK                                        │
│    THEN final_decision = BLOCK, reason = "Watchlist hit"                   │
│ ELSE IF any_match == SANCTIONS_REVIEW OR PEP_REVIEW OR WATCHLIST_REVIEW     │
│    THEN final_decision = REVIEW, reason = "Potential match requiring review"│
│ ELSE IF pattern_risk == HIGH OR composite_score > 70                        │
│    THEN final_decision = REVIEW, reason = "High risk score"                │
│ ELSE                                                                         │
│    final_decision = PASS                                                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Functional Requirements Summary

### 3.1 Sanctions Screening Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| AML-S-001 | System shall screen all parties against OFAC SDN (Specially Designated Nationals) list | P1 |
| AML-S-002 | System shall screen all parties against EU Consolidated Sanctions List | P1 |
| AML-S-003 | System shall screen all parties against UN Security Council Sanctions List | P1 |
| AML-S-004 | System shall screen all parties against UK HM Treasury Sanctions List | P1 |
| AML-S-005 | System shall support country-specific sanctions lists (configurable per jurisdiction) | P1 |
| AML-S-006 | System shall screen originator name, beneficiary name, and intermediary bank names | P1 |
| AML-S-007 | System shall screen against sanctioned vessel names and registration numbers | P2 |
| AML-S-008 | System shall screen against sanctioned aircraft tail numbers | P2 |
| AML-S-009 | System shall automatically update sanctions lists within 4 hours of publication | P1 |
| AML-S-010 | System shall maintain sanctions list version history for audit purposes | P1 |
| AML-S-011 | System shall support secondary sanctions screening (e.g., Iran secondary) | P2 |
| AML-S-012 | System shall detect sanctions evasion through alias and variation matching | P1 |

### 3.2 PEP Screening Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| AML-P-001 | System shall screen parties against PEP databases (domestic and foreign) | P1 |
| AML-P-002 | System shall identify Relatives and Close Associates (RCAs) of PEPs | P1 |
| AML-P-003 | System shall categorize PEPs by risk level (domestic, foreign, international org) | P1 |
| AML-P-004 | System shall identify State-Owned Enterprise (SOE) connections | P2 |
| AML-P-005 | System shall support configurable PEP risk categories and thresholds | P1 |
| AML-P-006 | System shall maintain PEP status history (former PEP tracking) | P2 |
| AML-P-007 | System shall screen against adverse media related to PEP status | P2 |
| AML-P-008 | System shall support multiple PEP data providers (World-Check, Dow Jones) | P1 |

### 3.3 Watchlist Management Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| AML-W-001 | System shall support internal blacklist/whitelist management | P1 |
| AML-W-002 | System shall support customer-defined watchlists | P1 |
| AML-W-003 | System shall support industry-specific watchlists (e.g., MSB, MVTS) | P2 |
| AML-W-004 | System shall support negative news/adverse media screening | P2 |
| AML-W-005 | System shall support law enforcement watchlists (with appropriate access) | P2 |
| AML-W-006 | System shall maintain watchlist entry audit trail (add, modify, remove) | P1 |
| AML-W-007 | System shall support bulk watchlist import/export (CSV, XML) | P2 |
| AML-W-008 | System shall support watchlist entry expiration and review dates | P2 |

### 3.4 Name Matching Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| AML-N-001 | System shall support exact name matching with normalization | P1 |
| AML-N-002 | System shall support fuzzy matching using Jaro-Winkler algorithm | P1 |
| AML-N-003 | System shall support fuzzy matching using Levenshtein distance | P1 |
| AML-N-004 | System shall support phonetic matching using Soundex algorithm | P1 |
| AML-N-005 | System shall support phonetic matching using Double Metaphone | P2 |
| AML-N-006 | System shall handle name variations (initials, prefixes, suffixes) | P1 |
| AML-N-007 | System shall handle transliteration from non-Latin scripts | P1 |
| AML-N-008 | System shall handle nickname and alias matching | P1 |
| AML-N-009 | System shall support configurable match thresholds per list type | P1 |
| AML-N-010 | System shall support weighted scoring across multiple algorithms | P2 |
| AML-N-011 | System shall handle name order variations (first/last name swapping) | P1 |
| AML-N-012 | System shall support language-specific name matching rules | P2 |

### 3.5 Pattern Detection Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| AML-T-001 | System shall detect structuring/smurfing patterns (threshold avoidance) | P1 |
| AML-T-002 | System shall detect rapid movement of funds (velocity patterns) | P1 |
| AML-T-003 | System shall detect round-trip transactions | P1 |
| AML-T-004 | System shall detect layering patterns (multiple intermediaries) | P2 |
| AML-T-005 | System shall detect unusual transaction timing patterns | P2 |
| AML-T-006 | System shall detect deviation from established customer patterns | P1 |
| AML-T-007 | System shall detect new beneficiary patterns for existing customers | P2 |
| AML-T-008 | System shall detect dormant account reactivation with high activity | P2 |
| AML-T-009 | System shall detect high-risk geographic patterns | P1 |
| AML-T-010 | System shall support configurable pattern detection rules | P1 |

### 3.6 Risk Scoring Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| AML-R-001 | System shall calculate composite risk score (0-100) for each transaction | P1 |
| AML-R-002 | System shall include customer risk factors in scoring | P1 |
| AML-R-003 | System shall include geographic risk factors in scoring | P1 |
| AML-R-004 | System shall include transaction risk factors in scoring | P1 |
| AML-R-005 | System shall include product/channel risk factors in scoring | P2 |
| AML-R-006 | System shall support configurable risk factor weights | P1 |
| AML-R-007 | System shall provide risk score breakdown by factor category | P1 |
| AML-R-008 | System shall maintain historical risk score trends per customer | P2 |

### 3.7 Alert & Case Management Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| AML-A-001 | System shall generate alerts for REVIEW and BLOCK decisions | P1 |
| AML-A-002 | System shall assign alert priority based on risk score and match confidence | P1 |
| AML-A-003 | System shall support automatic case creation for alerts | P1 |
| AML-A-004 | System shall support alert consolidation for related transactions | P2 |
| AML-A-005 | System shall provide match details and evidence in alerts | P1 |
| AML-A-006 | System shall support alert routing based on risk type and amount | P2 |
| AML-A-007 | System shall track alert resolution outcomes for tuning | P1 |
| AML-A-008 | System shall support SLA tracking for alert resolution | P2 |

### 3.8 Regulatory Reporting Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| AML-REG-001 | System shall support SAR (Suspicious Activity Report) generation | P1 |
| AML-REG-002 | System shall support STR (Suspicious Transaction Report) generation | P1 |
| AML-REG-003 | System shall support CTR (Currency Transaction Report) generation | P2 |
| AML-REG-004 | System shall support regulatory report submission tracking | P1 |
| AML-REG-005 | System shall maintain regulatory filing history | P1 |
| AML-REG-006 | System shall support jurisdiction-specific report formats | P2 |

---

## 4. Domain Model

### 4.1 Core Domain Entities

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AML SCREENING DOMAIN MODEL                           │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                         AGGREGATE ROOTS                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    ScreeningRequest [Aggregate Root]                 │   │
│  │                                                                      │   │
│  │  - requestId: UUID (Primary Identifier)                             │   │
│  │  - transactionId: UUID (Link to Payment Transaction)                │   │
│  │  - requestType: ScreeningRequestType (REAL_TIME, BATCH, PERIODIC)   │   │
│  │  - priority: ScreeningPriority (HIGH, MEDIUM, LOW)                  │   │
│  │  - subjects: List<ScreeningSubject>                                 │   │
│  │  - transactionDetails: TransactionDetails                           │   │
│  │  - requestedAt: Timestamp                                           │   │
│  │  - requestedBy: String (Service/User ID)                            │   │
│  │  - screeningConfig: ScreeningConfiguration                          │   │
│  │  - status: ScreeningStatus                                          │   │
│  │                                                                      │   │
│  │  Methods:                                                            │   │
│  │  + addSubject(subject: ScreeningSubject): void                      │   │
│  │  + updateStatus(status: ScreeningStatus): void                      │   │
│  │  + getSubjectsByType(type: SubjectType): List<ScreeningSubject>     │   │
│  │  + isHighPriority(): boolean                                        │   │
│  │  + requiresEnhancedScreening(): boolean                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    ScreeningResult [Aggregate Root]                  │   │
│  │                                                                      │   │
│  │  - resultId: UUID (Primary Identifier)                              │   │
│  │  - requestId: UUID (Link to ScreeningRequest)                       │   │
│  │  - decision: ScreeningDecision (PASS, REVIEW, BLOCK)                │   │
│  │  - compositeRiskScore: Integer (0-100)                              │   │
│  │  - matches: List<ScreeningMatch>                                    │   │
│  │  - patternAnalysisResult: PatternAnalysisResult                     │   │
│  │  - riskScoreBreakdown: RiskScoreBreakdown                           │   │
│  │  - screenedAt: Timestamp                                            │   │
│  │  - screeningDurationMs: Long                                        │   │
│  │  - listsScreened: List<ListReference>                               │   │
│  │  - alertReference: String (if alert created)                        │   │
│  │  - caseReference: String (if case created)                          │   │
│  │                                                                      │   │
│  │  Methods:                                                            │   │
│  │  + addMatch(match: ScreeningMatch): void                            │   │
│  │  + hasBlockingMatch(): boolean                                      │   │
│  │  + hasReviewableMatch(): boolean                                    │   │
│  │  + getMatchesByType(type: MatchType): List<ScreeningMatch>          │   │
│  │  + getHighestConfidenceMatch(): Optional<ScreeningMatch>            │   │
│  │  + requiresInvestigation(): boolean                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    SanctionsList [Aggregate Root]                    │   │
│  │                                                                      │   │
│  │  - listId: UUID (Primary Identifier)                                │   │
│  │  - listCode: String (OFAC_SDN, EU_CONSOLIDATED, UN_SANCTIONS, etc.) │   │
│  │  - listName: String                                                  │   │
│  │  - listType: ListType (SANCTIONS, PEP, WATCHLIST, ADVERSE_MEDIA)    │   │
│  │  - jurisdiction: String                                              │   │
│  │  - sourceUrl: String                                                 │   │
│  │  - version: String                                                   │   │
│  │  - publishedAt: Timestamp                                            │   │
│  │  - loadedAt: Timestamp                                               │   │
│  │  - entryCount: Integer                                               │   │
│  │  - status: ListStatus (ACTIVE, LOADING, FAILED, DEPRECATED)         │   │
│  │  - entries: List<ListEntry> (loaded on demand)                      │   │
│  │                                                                      │   │
│  │  Methods:                                                            │   │
│  │  + addEntry(entry: ListEntry): void                                 │   │
│  │  + removeEntry(entryId: UUID): void                                 │   │
│  │  + searchEntries(criteria: SearchCriteria): List<ListEntry>         │   │
│  │  + getEntriesByType(type: EntryType): List<ListEntry>               │   │
│  │  + isActive(): boolean                                               │   │
│  │  + needsUpdate(): boolean                                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    AmlAlert [Aggregate Root]                         │   │
│  │                                                                      │   │
│  │  - alertId: UUID (Primary Identifier)                               │   │
│  │  - alertReference: String (Human-readable reference)                │   │
│  │  - resultId: UUID (Link to ScreeningResult)                         │   │
│  │  - transactionId: UUID                                               │   │
│  │  - alertType: AlertType (SANCTIONS_HIT, PEP_HIT, PATTERN, RISK)     │   │
│  │  - priority: AlertPriority (CRITICAL, HIGH, MEDIUM, LOW)            │   │
│  │  - status: AlertStatus (OPEN, ASSIGNED, INVESTIGATING, CLOSED)      │   │
│  │  - matchDetails: List<MatchDetail>                                  │   │
│  │  - riskScore: Integer                                                │   │
│  │  - createdAt: Timestamp                                              │   │
│  │  - assignedTo: String                                                │   │
│  │  - assignedAt: Timestamp                                             │   │
│  │  - dueAt: Timestamp (SLA deadline)                                   │   │
│  │  - resolution: AlertResolution                                       │   │
│  │  - linkedCaseId: UUID                                                │   │
│  │                                                                      │   │
│  │  Methods:                                                            │   │
│  │  + assign(analyst: String): void                                    │   │
│  │  + escalate(reason: String): void                                   │   │
│  │  + resolve(resolution: AlertResolution): void                       │   │
│  │  + linkToCase(caseId: UUID): void                                   │   │
│  │  + addNote(note: AlertNote): void                                   │   │
│  │  + isOverdue(): boolean                                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Entity Definitions

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              ENTITIES                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    ScreeningSubject [Entity]                         │   │
│  │                                                                      │   │
│  │  - subjectId: UUID                                                   │   │
│  │  - subjectType: SubjectType (ORIGINATOR, BENEFICIARY, INTERMEDIARY,  │   │
│  │                              CONNECTED_PARTY, VESSEL, AIRCRAFT)      │   │
│  │  - entityType: EntityType (INDIVIDUAL, ORGANIZATION, VESSEL, etc.)  │   │
│  │  - name: NameDetails                                                 │   │
│  │  - dateOfBirth: LocalDate (for individuals)                         │   │
│  │  - nationality: List<String>                                         │   │
│  │  - countryOfResidence: String                                        │   │
│  │  - addresses: List<Address>                                          │   │
│  │  - identifiers: List<Identifier> (passport, national ID, BIC, etc.) │   │
│  │  - aliases: List<String>                                             │   │
│  │  - additionalInfo: Map<String, Object>                               │   │
│  │                                                                      │   │
│  │  Methods:                                                            │   │
│  │  + getNormalizedName(): String                                       │   │
│  │  + getAllNameVariations(): List<String>                              │   │
│  │  + getSearchableTokens(): List<String>                               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    ListEntry [Entity]                                │   │
│  │                                                                      │   │
│  │  - entryId: UUID                                                     │   │
│  │  - externalId: String (ID from source list)                         │   │
│  │  - listId: UUID (parent list)                                        │   │
│  │  - entryType: EntryType (INDIVIDUAL, ENTITY, VESSEL, AIRCRAFT)      │   │
│  │  - primaryName: String                                               │   │
│  │  - aliases: List<AliasEntry>                                         │   │
│  │  - dateOfBirth: LocalDate                                            │   │
│  │  - placeOfBirth: String                                              │   │
│  │  - nationalities: List<String>                                       │   │
│  │  - addresses: List<Address>                                          │   │
│  │  - identifiers: List<Identifier>                                     │   │
│  │  - sanctionPrograms: List<String> (e.g., SDGT, NPWMD)               │   │
│  │  - listingReason: String                                             │   │
│  │  - listedDate: LocalDate                                             │   │
│  │  - lastUpdated: Timestamp                                            │   │
│  │  - remarks: String                                                   │   │
│  │  - status: EntryStatus (ACTIVE, REMOVED, UPDATED)                   │   │
│  │                                                                      │   │
│  │  Methods:                                                            │   │
│  │  + getAllNames(): List<String>                                       │   │
│  │  + getSearchableFields(): Map<String, String>                        │   │
│  │  + matchesCriteria(criteria: SearchCriteria): boolean                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    ScreeningMatch [Entity]                           │   │
│  │                                                                      │   │
│  │  - matchId: UUID                                                     │   │
│  │  - subjectId: UUID (which subject matched)                          │   │
│  │  - listEntryId: UUID (which list entry matched)                     │   │
│  │  - listCode: String                                                  │   │
│  │  - matchType: MatchType (SANCTIONS, PEP, WATCHLIST, ADVERSE_MEDIA)  │   │
│  │  - matchConfidence: Integer (0-100)                                  │   │
│  │  - matchAlgorithm: String (EXACT, FUZZY, PHONETIC)                  │   │
│  │  - matchedFields: List<MatchedField>                                │   │
│  │  - subjectName: String (name that was screened)                     │   │
│  │  - matchedName: String (name from list)                             │   │
│  │  - additionalMatches: Map<String, MatchedField> (DOB, nationality)  │   │
│  │  - recommendedAction: RecommendedAction (BLOCK, REVIEW, PASS)       │   │
│  │  - matchedAt: Timestamp                                              │   │
│  │                                                                      │   │
│  │  Methods:                                                            │   │
│  │  + isConfirmedMatch(): boolean                                       │   │
│  │  + isPotentialMatch(): boolean                                       │   │
│  │  + getMatchSummary(): String                                         │   │
│  │  + requiresManualReview(): boolean                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    PatternAnalysisResult [Entity]                    │   │
│  │                                                                      │   │
│  │  - analysisId: UUID                                                  │   │
│  │  - transactionId: UUID                                               │   │
│  │  - customerId: UUID                                                  │   │
│  │  - patternsDetected: List<DetectedPattern>                          │   │
│  │  - structuringScore: Integer (0-100)                                │   │
│  │  - velocityScore: Integer (0-100)                                   │   │
│  │  - behaviorDeviationScore: Integer (0-100)                          │   │
│  │  - overallPatternRisk: RiskLevel (LOW, MEDIUM, HIGH, CRITICAL)      │   │
│  │  - analysisContext: PatternContext                                   │   │
│  │  - analyzedAt: Timestamp                                             │   │
│  │                                                                      │   │
│  │  Methods:                                                            │   │
│  │  + hasHighRiskPattern(): boolean                                     │   │
│  │  + getTopPatterns(n: int): List<DetectedPattern>                    │   │
│  │  + getPatternsByCategory(cat: PatternCategory): List<DetectedPattern>│   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    RiskScoreBreakdown [Entity]                       │   │
│  │                                                                      │   │
│  │  - breakdownId: UUID                                                 │   │
│  │  - compositeScore: Integer (0-100)                                  │   │
│  │  - customerRiskScore: Integer                                        │   │
│  │  - customerRiskFactors: List<RiskFactor>                            │   │
│  │  - transactionRiskScore: Integer                                     │   │
│  │  - transactionRiskFactors: List<RiskFactor>                         │   │
│  │  - geographicRiskScore: Integer                                      │   │
│  │  - geographicRiskFactors: List<RiskFactor>                          │   │
│  │  - productRiskScore: Integer                                         │   │
│  │  - productRiskFactors: List<RiskFactor>                             │   │
│  │  - matchRiskScore: Integer (from screening matches)                 │   │
│  │  - patternRiskScore: Integer (from pattern analysis)               │   │
│  │  - calculatedAt: Timestamp                                           │   │
│  │  - modelVersion: String                                              │   │
│  │                                                                      │   │
│  │  Methods:                                                            │   │
│  │  + getTopContributingFactors(n: int): List<RiskFactor>              │   │
│  │  + getFactorsByCategory(cat: RiskCategory): List<RiskFactor>        │   │
│  │  + getRiskLevel(): RiskLevel                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 Value Objects

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           VALUE OBJECTS                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌────────────────────┐│
│  │ NameDetails          │  │ TransactionDetails   │  │ MatchedField       ││
│  │                      │  │                      │  │                    ││
│  │ - fullName: String   │  │ - amount: Money      │  │ - fieldName: String││
│  │ - firstName: String  │  │ - currency: Currency │  │ - subjectValue: Str││
│  │ - middleName: String │  │ - paymentType: String│  │ - listValue: String││
│  │ - lastName: String   │  │ - channel: String    │  │ - matchScore: Int  ││
│  │ - prefix: String     │  │ - originCountry: Str │  │ - algorithm: String││
│  │ - suffix: String     │  │ - destCountry: String│  │                    ││
│  │ - script: String     │  │ - valueDate: Date    │  │                    ││
│  └──────────────────────┘  └──────────────────────┘  └────────────────────┘│
│                                                                             │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌────────────────────┐│
│  │ Address              │  │ Identifier           │  │ RiskFactor         ││
│  │                      │  │                      │  │                    ││
│  │ - addressType: Enum  │  │ - type: IdType       │  │ - factorCode: Str  ││
│  │ - line1: String      │  │ - value: String      │  │ - factorName: Str  ││
│  │ - line2: String      │  │ - issuingCountry: Str│  │ - weight: Decimal  ││
│  │ - city: String       │  │ - issueDate: Date    │  │ - score: Integer   ││
│  │ - state: String      │  │ - expiryDate: Date   │  │ - category: Enum   ││
│  │ - postalCode: String │  │                      │  │ - description: Str ││
│  │ - country: String    │  │                      │  │                    ││
│  └──────────────────────┘  └──────────────────────┘  └────────────────────┘│
│                                                                             │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌────────────────────┐│
│  │ DetectedPattern      │  │ AlertResolution      │  │ ListReference      ││
│  │                      │  │                      │  │                    ││
│  │ - patternType: Enum  │  │ - outcome: Enum      │  │ - listCode: String ││
│  │ - confidence: Integer│  │ - resolvedBy: String │  │ - listVersion: Str ││
│  │ - description: String│  │ - resolvedAt: Time   │  │ - screenedAt: Time ││
│  │ - evidence: List<Str>│  │ - notes: String      │  │ - entryCount: Int  ││
│  │ - riskContribution: I│  │ - sarFiled: boolean  │  │                    ││
│  │ - timeWindow: Duration│ │ - sarReference: Str  │  │                    ││
│  └──────────────────────┘  └──────────────────────┘  └────────────────────┘│
│                                                                             │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌────────────────────┐│
│  │ ScreeningConfig      │  │ MatchThreshold       │  │ AliasEntry         ││
│  │                      │  │                      │  │                    ││
│  │ - listsToScreen: []  │  │ - listType: Enum     │  │ - aliasType: Enum  ││
│  │ - matchThresholds: []│  │ - minScore: Integer  │  │ - aliasName: String││
│  │ - algorithms: []     │  │ - blockThreshold: Int│  │ - quality: String  ││
│  │ - includePatterns: bo│ │ - reviewThreshold:Int│  │ - script: String   ││
│  │ - riskScoring: bool  │  │                      │  │                    ││
│  │ - timeout: Duration  │  │                      │  │                    ││
│  └──────────────────────┘  └──────────────────────┘  └────────────────────┘│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.4 Enumerations

```java
// Screening Request & Result Enums
public enum ScreeningRequestType {
    REAL_TIME,      // Synchronous screening during payment
    BATCH,          // Batch file screening
    PERIODIC,       // Scheduled re-screening
    ON_DEMAND,      // Manual screening request
    ONBOARDING      // Customer onboarding screening
}

public enum ScreeningPriority {
    CRITICAL,   // Immediate processing, <100ms SLA
    HIGH,       // Priority processing, <200ms SLA
    MEDIUM,     // Standard processing, <500ms SLA
    LOW         // Background processing, <2s SLA
}

public enum ScreeningDecision {
    PASS,       // Clear to proceed
    REVIEW,     // Requires analyst review
    BLOCK,      // Transaction must be blocked
    PENDING     // Awaiting external response
}

public enum ScreeningStatus {
    RECEIVED,       // Request received
    PROCESSING,     // Currently being screened
    COMPLETED,      // Screening complete
    FAILED,         // Screening failed (technical)
    TIMEOUT         // Screening timed out
}

// Subject & Entity Enums
public enum SubjectType {
    ORIGINATOR,         // Payment sender
    BENEFICIARY,        // Payment receiver
    INTERMEDIARY,       // Intermediary bank
    CONNECTED_PARTY,    // Director, owner, etc.
    VESSEL,             // Ship/vessel
    AIRCRAFT            // Aircraft
}

public enum EntityType {
    INDIVIDUAL,         // Natural person
    ORGANIZATION,       // Legal entity
    VESSEL,             // Ship/vessel
    AIRCRAFT,           // Aircraft
    UNKNOWN             // Unknown entity type
}

// List & Entry Enums
public enum ListType {
    SANCTIONS,          // Government sanctions list
    PEP,                // Politically exposed persons
    WATCHLIST,          // Internal/external watchlist
    ADVERSE_MEDIA,      // Negative news
    LAW_ENFORCEMENT     // Law enforcement lists
}

public enum EntryType {
    INDIVIDUAL,
    ENTITY,
    VESSEL,
    AIRCRAFT,
    CRYPTO_WALLET       // Cryptocurrency address
}

public enum ListStatus {
    ACTIVE,         // Currently in use
    LOADING,        // Being loaded/updated
    FAILED,         // Load/update failed
    DEPRECATED,     // No longer in use
    PENDING         // Pending activation
}

// Match Enums
public enum MatchType {
    SANCTIONS_EXACT,        // Exact sanctions match
    SANCTIONS_FUZZY,        // Fuzzy sanctions match
    PEP_MATCH,              // PEP database match
    WATCHLIST_MATCH,        // Internal watchlist match
    ADVERSE_MEDIA_MATCH,    // Negative news match
    PATTERN_MATCH           // Suspicious pattern detected
}

public enum RecommendedAction {
    BLOCK,      // Block the transaction
    REVIEW,     // Manual review required
    PASS,       // Clear to proceed
    ESCALATE    // Escalate to senior analyst
}

// Alert Enums
public enum AlertType {
    SANCTIONS_HIT,      // Sanctions list match
    PEP_HIT,            // PEP match
    WATCHLIST_HIT,      // Watchlist match
    ADVERSE_MEDIA_HIT,  // Negative news match
    PATTERN_ALERT,      // Suspicious pattern
    HIGH_RISK_SCORE,    // High composite risk
    THRESHOLD_BREACH    // Reporting threshold exceeded
}

public enum AlertPriority {
    CRITICAL,   // Requires immediate attention (<1 hour SLA)
    HIGH,       // Same-day resolution (<8 hours SLA)
    MEDIUM,     // 24-hour resolution SLA
    LOW         // 72-hour resolution SLA
}

public enum AlertStatus {
    OPEN,           // Newly created
    ASSIGNED,       // Assigned to analyst
    INVESTIGATING,  // Under investigation
    PENDING_INFO,   // Awaiting additional information
    ESCALATED,      // Escalated to senior
    CLOSED          // Resolved and closed
}

public enum AlertResolutionOutcome {
    TRUE_POSITIVE,          // Confirmed match - action taken
    FALSE_POSITIVE,         // Not a real match
    INCONCLUSIVE,           // Unable to determine
    SAR_FILED,              // SAR filed
    TRANSACTION_BLOCKED,    // Transaction was blocked
    CUSTOMER_EXITED         // Customer relationship terminated
}

// Pattern Enums
public enum PatternCategory {
    STRUCTURING,        // Threshold avoidance
    VELOCITY,           // Rapid transactions
    BEHAVIORAL,         // Deviation from normal
    GEOGRAPHIC,         // High-risk geography
    RELATIONSHIP,       // Suspicious relationships
    TEMPORAL            // Time-based patterns
}

// Risk Enums
public enum RiskLevel {
    LOW,        // Score 0-29
    MEDIUM,     // Score 30-59
    HIGH,       // Score 60-79
    CRITICAL    // Score 80-100
}

public enum RiskCategory {
    CUSTOMER,       // Customer-related risk
    TRANSACTION,    // Transaction-related risk
    GEOGRAPHIC,     // Geography-related risk
    PRODUCT,        // Product/channel risk
    MATCH,          // Screening match risk
    PATTERN         // Pattern detection risk
}

// Identifier Enums
public enum IdentifierType {
    PASSPORT,
    NATIONAL_ID,
    DRIVING_LICENSE,
    TAX_ID,
    BIC,
    LEI,
    REGISTRATION_NUMBER,
    VESSEL_IMO,
    AIRCRAFT_TAIL,
    CRYPTO_ADDRESS
}
```

### 4.5 Domain Services

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DOMAIN SERVICES                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  ScreeningOrchestrationService                       │   │
│  │                                                                      │   │
│  │  Responsibility: Orchestrate the end-to-end screening process       │   │
│  │                                                                      │   │
│  │  + screenTransaction(request: ScreeningRequest): ScreeningResult    │   │
│  │  + screenBatch(requests: List<ScreeningRequest>): List<Result>      │   │
│  │  + rescreenCustomer(customerId: UUID): List<ScreeningResult>        │   │
│  │  + getScreeningStatus(requestId: UUID): ScreeningStatus             │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  SanctionsScreeningService                           │   │
│  │                                                                      │   │
│  │  Responsibility: Screen subjects against sanctions lists            │   │
│  │                                                                      │   │
│  │  + screenAgainstSanctions(subjects: List<Subject>): List<Match>     │   │
│  │  + screenAgainstList(subject: Subject, list: String): List<Match>   │   │
│  │  + getActiveListVersions(): Map<String, String>                      │   │
│  │  + validateSanctionsCompliance(txn: Transaction): ComplianceResult  │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  PepScreeningService                                 │   │
│  │                                                                      │   │
│  │  Responsibility: Screen subjects against PEP databases              │   │
│  │                                                                      │   │
│  │  + screenAgainstPep(subjects: List<Subject>): List<PepMatch>        │   │
│  │  + identifyRcaConnections(subject: Subject): List<RcaConnection>    │   │
│  │  + getPepRiskCategory(pepMatch: PepMatch): PepRiskCategory          │   │
│  │  + checkSoeConnection(subject: Subject): Optional<SoeConnection>    │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  NameMatchingService                                 │   │
│  │                                                                      │   │
│  │  Responsibility: Execute name matching algorithms                   │   │
│  │                                                                      │   │
│  │  + matchName(input: String, target: String): MatchScore             │   │
│  │  + matchWithAlgorithm(input: String, target: String, algo): Score   │   │
│  │  + calculateCompositeScore(scores: List<MatchScore>): Integer       │   │
│  │  + normalizeName(name: String): String                               │   │
│  │  + tokenizeName(name: String): List<String>                         │   │
│  │  + transliterate(name: String, sourceScript: String): String        │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  PatternAnalysisService                              │   │
│  │                                                                      │   │
│  │  Responsibility: Detect suspicious transaction patterns             │   │
│  │                                                                      │   │
│  │  + analyzeTransaction(txn: Transaction): PatternAnalysisResult      │   │
│  │  + detectStructuring(txns: List<Transaction>): StructuringResult    │   │
│  │  + analyzeVelocity(customerId: UUID, window: Duration): VelocityRes │   │
│  │  + detectBehavioralDeviation(txn: Transaction): DeviationResult     │   │
│  │  + analyzeGeographicRisk(txn: Transaction): GeographicRiskResult    │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  RiskScoringService                                  │   │
│  │                                                                      │   │
│  │  Responsibility: Calculate composite risk scores                    │   │
│  │                                                                      │   │
│  │  + calculateRiskScore(request: ScreeningRequest,                    │   │
│  │                       matches: List<Match>,                         │   │
│  │                       patterns: PatternResult): RiskScoreBreakdown  │   │
│  │  + getCustomerRiskFactors(customerId: UUID): List<RiskFactor>       │   │
│  │  + getTransactionRiskFactors(txn: Transaction): List<RiskFactor>    │   │
│  │  + getGeographicRiskFactors(countries: List<String>): List<Factor>  │   │
│  │  + applyRiskWeights(factors: List<Factor>, weights: Config): Score  │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  ListManagementService                               │   │
│  │                                                                      │   │
│  │  Responsibility: Manage screening lists lifecycle                   │   │
│  │                                                                      │   │
│  │  + loadList(source: ListSource): SanctionsList                      │   │
│  │  + updateList(listCode: String): UpdateResult                       │   │
│  │  + addWatchlistEntry(entry: ListEntry): void                        │   │
│  │  + removeWatchlistEntry(entryId: UUID): void                        │   │
│  │  + searchListEntries(criteria: SearchCriteria): List<ListEntry>     │   │
│  │  + getListStatistics(listCode: String): ListStatistics              │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  AlertManagementService                              │   │
│  │                                                                      │   │
│  │  Responsibility: Manage screening alerts                            │   │
│  │                                                                      │   │
│  │  + createAlert(result: ScreeningResult): AmlAlert                   │   │
│  │  + assignAlert(alertId: UUID, analyst: String): void                │   │
│  │  + escalateAlert(alertId: UUID, reason: String): void               │   │
│  │  + resolveAlert(alertId: UUID, resolution: Resolution): void        │   │
│  │  + linkAlertToCase(alertId: UUID, caseId: UUID): void               │   │
│  │  + getAlertsByStatus(status: AlertStatus): List<AmlAlert>           │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  RegulatoryReportingService                          │   │
│  │                                                                      │   │
│  │  Responsibility: Generate and submit regulatory reports             │   │
│  │                                                                      │   │
│  │  + generateSar(alert: AmlAlert): SarReport                          │   │
│  │  + generateStr(alert: AmlAlert): StrReport                          │   │
│  │  + generateCtr(transaction: Transaction): CtrReport                 │   │
│  │  + submitReport(report: RegulatoryReport): SubmissionResult         │   │
│  │  + getReportStatus(reportId: UUID): ReportStatus                    │   │
│  │  + getFilingHistory(criteria: FilingCriteria): List<Filing>         │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Data Architecture

### 5.1 Database Strategy

The AML Screening module employs a **polyglot persistence** strategy optimized for different data access patterns:

| Data Store | Technology | Purpose |
|------------|------------|---------|
| Primary Database | PostgreSQL | Screening results, alerts, audit logs |
| List Store | Elasticsearch | Sanctions/PEP list entries for fast search |
| Match Cache | Redis Cluster | Recent match results, list entry cache |
| Pattern Store | TimescaleDB | Time-series transaction data for pattern analysis |
| Document Store | MongoDB | Regulatory reports, case documents |
| Graph Database | Neo4j | Relationship mapping (PEP connections, entity links) |

### 5.2 Primary Database Schema (PostgreSQL)

```sql
-- ============================================================================
-- AML SCREENING MODULE - PostgreSQL Schema
-- ============================================================================

-- Schema for AML Screening
CREATE SCHEMA IF NOT EXISTS aml_screening;

-- ============================================================================
-- SCREENING REQUESTS & RESULTS
-- ============================================================================

-- Screening Request Table
CREATE TABLE aml_screening.screening_request (
    request_id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_id          UUID NOT NULL,
    request_type            VARCHAR(20) NOT NULL,  -- REAL_TIME, BATCH, PERIODIC
    priority                VARCHAR(20) NOT NULL DEFAULT 'MEDIUM',
    status                  VARCHAR(20) NOT NULL DEFAULT 'RECEIVED',
    requested_at            TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    requested_by            VARCHAR(100) NOT NULL,
    completed_at            TIMESTAMP WITH TIME ZONE,
    screening_duration_ms   INTEGER,
    config_snapshot         JSONB,                 -- Screening config at time of request
    metadata                JSONB,
    created_at              TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    CONSTRAINT chk_request_type CHECK (request_type IN ('REAL_TIME', 'BATCH', 'PERIODIC', 'ON_DEMAND', 'ONBOARDING')),
    CONSTRAINT chk_priority CHECK (priority IN ('CRITICAL', 'HIGH', 'MEDIUM', 'LOW')),
    CONSTRAINT chk_status CHECK (status IN ('RECEIVED', 'PROCESSING', 'COMPLETED', 'FAILED', 'TIMEOUT'))
);

-- Index for transaction lookups
CREATE INDEX idx_screening_request_transaction ON aml_screening.screening_request(transaction_id);
CREATE INDEX idx_screening_request_status ON aml_screening.screening_request(status);
CREATE INDEX idx_screening_request_requested_at ON aml_screening.screening_request(requested_at);

-- Screening Subject Table
CREATE TABLE aml_screening.screening_subject (
    subject_id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id              UUID NOT NULL REFERENCES aml_screening.screening_request(request_id),
    subject_type            VARCHAR(30) NOT NULL,  -- ORIGINATOR, BENEFICIARY, etc.
    entity_type             VARCHAR(20) NOT NULL,  -- INDIVIDUAL, ORGANIZATION
    full_name               VARCHAR(500) NOT NULL,
    first_name              VARCHAR(200),
    middle_name             VARCHAR(200),
    last_name               VARCHAR(200),
    date_of_birth           DATE,
    nationalities           VARCHAR(100)[] DEFAULT '{}',
    country_of_residence    VARCHAR(3),
    addresses               JSONB,
    identifiers             JSONB,                 -- Array of {type, value, country}
    aliases                 VARCHAR(500)[] DEFAULT '{}',
    normalized_name         VARCHAR(500),          -- Pre-computed for matching
    name_tokens             TSVECTOR,              -- Full-text search tokens
    additional_info         JSONB,
    created_at              TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    CONSTRAINT chk_subject_type CHECK (subject_type IN ('ORIGINATOR', 'BENEFICIARY', 'INTERMEDIARY', 'CONNECTED_PARTY', 'VESSEL', 'AIRCRAFT')),
    CONSTRAINT chk_entity_type CHECK (entity_type IN ('INDIVIDUAL', 'ORGANIZATION', 'VESSEL', 'AIRCRAFT', 'UNKNOWN'))
);

CREATE INDEX idx_screening_subject_request ON aml_screening.screening_subject(request_id);
CREATE INDEX idx_screening_subject_name ON aml_screening.screening_subject USING gin(name_tokens);
CREATE INDEX idx_screening_subject_normalized ON aml_screening.screening_subject(normalized_name);

-- Screening Result Table
CREATE TABLE aml_screening.screening_result (
    result_id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id              UUID NOT NULL REFERENCES aml_screening.screening_request(request_id),
    decision                VARCHAR(20) NOT NULL,
    composite_risk_score    INTEGER NOT NULL CHECK (composite_risk_score BETWEEN 0 AND 100),
    customer_risk_score     INTEGER CHECK (customer_risk_score BETWEEN 0 AND 100),
    transaction_risk_score  INTEGER CHECK (transaction_risk_score BETWEEN 0 AND 100),
    geographic_risk_score   INTEGER CHECK (geographic_risk_score BETWEEN 0 AND 100),
    match_risk_score        INTEGER CHECK (match_risk_score BETWEEN 0 AND 100),
    pattern_risk_score      INTEGER CHECK (pattern_risk_score BETWEEN 0 AND 100),
    risk_factors            JSONB,                 -- Detailed factor breakdown
    total_matches           INTEGER NOT NULL DEFAULT 0,
    blocking_matches        INTEGER NOT NULL DEFAULT 0,
    review_matches          INTEGER NOT NULL DEFAULT 0,
    lists_screened          VARCHAR(50)[] DEFAULT '{}',
    list_versions           JSONB,                 -- {listCode: version}
    screened_at             TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    alert_reference         VARCHAR(50),
    case_reference          VARCHAR(50),
    decision_reason         TEXT,
    created_at              TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    CONSTRAINT chk_decision CHECK (decision IN ('PASS', 'REVIEW', 'BLOCK', 'PENDING'))
);

CREATE UNIQUE INDEX idx_screening_result_request ON aml_screening.screening_result(request_id);
CREATE INDEX idx_screening_result_decision ON aml_screening.screening_result(decision);
CREATE INDEX idx_screening_result_risk_score ON aml_screening.screening_result(composite_risk_score);
CREATE INDEX idx_screening_result_screened_at ON aml_screening.screening_result(screened_at);

-- Screening Match Table
CREATE TABLE aml_screening.screening_match (
    match_id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    result_id               UUID NOT NULL REFERENCES aml_screening.screening_result(result_id),
    subject_id              UUID NOT NULL REFERENCES aml_screening.screening_subject(subject_id),
    list_entry_id           VARCHAR(100) NOT NULL, -- External list entry ID
    list_code               VARCHAR(50) NOT NULL,
    list_version            VARCHAR(50),
    match_type              VARCHAR(30) NOT NULL,
    match_confidence        INTEGER NOT NULL CHECK (match_confidence BETWEEN 0 AND 100),
    match_algorithm         VARCHAR(30) NOT NULL,
    subject_name            VARCHAR(500) NOT NULL,
    matched_name            VARCHAR(500) NOT NULL,
    matched_fields          JSONB,                 -- Array of {field, subjectVal, listVal, score}
    additional_matches      JSONB,                 -- DOB, nationality matches
    recommended_action      VARCHAR(20) NOT NULL,
    match_details           JSONB,                 -- Full list entry details
    matched_at              TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    CONSTRAINT chk_match_type CHECK (match_type IN ('SANCTIONS_EXACT', 'SANCTIONS_FUZZY', 'PEP_MATCH', 'WATCHLIST_MATCH', 'ADVERSE_MEDIA_MATCH')),
    CONSTRAINT chk_algorithm CHECK (match_algorithm IN ('EXACT', 'JARO_WINKLER', 'LEVENSHTEIN', 'SOUNDEX', 'METAPHONE', 'COMPOSITE')),
    CONSTRAINT chk_recommended_action CHECK (recommended_action IN ('BLOCK', 'REVIEW', 'PASS', 'ESCALATE'))
);

CREATE INDEX idx_screening_match_result ON aml_screening.screening_match(result_id);
CREATE INDEX idx_screening_match_subject ON aml_screening.screening_match(subject_id);
CREATE INDEX idx_screening_match_list_code ON aml_screening.screening_match(list_code);
CREATE INDEX idx_screening_match_confidence ON aml_screening.screening_match(match_confidence);

-- ============================================================================
-- PATTERN ANALYSIS
-- ============================================================================

-- Pattern Analysis Result Table
CREATE TABLE aml_screening.pattern_analysis (
    analysis_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    result_id               UUID NOT NULL REFERENCES aml_screening.screening_result(result_id),
    transaction_id          UUID NOT NULL,
    customer_id             UUID NOT NULL,
    structuring_score       INTEGER CHECK (structuring_score BETWEEN 0 AND 100),
    velocity_score          INTEGER CHECK (velocity_score BETWEEN 0 AND 100),
    behavior_deviation_score INTEGER CHECK (behavior_deviation_score BETWEEN 0 AND 100),
    geographic_risk_score   INTEGER CHECK (geographic_risk_score BETWEEN 0 AND 100),
    overall_pattern_risk    VARCHAR(20) NOT NULL,
    analysis_window_hours   INTEGER NOT NULL DEFAULT 24,
    transactions_analyzed   INTEGER NOT NULL DEFAULT 0,
    patterns_detected       JSONB,                 -- Array of detected patterns
    analysis_context        JSONB,                 -- Historical context used
    analyzed_at             TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    CONSTRAINT chk_pattern_risk CHECK (overall_pattern_risk IN ('LOW', 'MEDIUM', 'HIGH', 'CRITICAL'))
);

CREATE INDEX idx_pattern_analysis_result ON aml_screening.pattern_analysis(result_id);
CREATE INDEX idx_pattern_analysis_customer ON aml_screening.pattern_analysis(customer_id);
CREATE INDEX idx_pattern_analysis_transaction ON aml_screening.pattern_analysis(transaction_id);

-- ============================================================================
-- ALERTS & CASES
-- ============================================================================

-- AML Alert Table
CREATE TABLE aml_screening.aml_alert (
    alert_id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_reference         VARCHAR(50) NOT NULL UNIQUE,
    result_id               UUID NOT NULL REFERENCES aml_screening.screening_result(result_id),
    transaction_id          UUID NOT NULL,
    customer_id             UUID,
    alert_type              VARCHAR(30) NOT NULL,
    priority                VARCHAR(20) NOT NULL,
    status                  VARCHAR(20) NOT NULL DEFAULT 'OPEN',
    risk_score              INTEGER NOT NULL,
    match_count             INTEGER NOT NULL DEFAULT 0,
    match_summary           TEXT,
    assigned_to             VARCHAR(100),
    assigned_at             TIMESTAMP WITH TIME ZONE,
    due_at                  TIMESTAMP WITH TIME ZONE,
    escalated_to            VARCHAR(100),
    escalated_at            TIMESTAMP WITH TIME ZONE,
    escalation_reason       TEXT,
    resolution_outcome      VARCHAR(30),
    resolution_notes        TEXT,
    resolved_by             VARCHAR(100),
    resolved_at             TIMESTAMP WITH TIME ZONE,
    sar_filed               BOOLEAN DEFAULT FALSE,
    sar_reference           VARCHAR(50),
    linked_case_id          UUID,
    linked_alerts           UUID[] DEFAULT '{}',
    created_at              TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    CONSTRAINT chk_alert_type CHECK (alert_type IN ('SANCTIONS_HIT', 'PEP_HIT', 'WATCHLIST_HIT', 'ADVERSE_MEDIA_HIT', 'PATTERN_ALERT', 'HIGH_RISK_SCORE', 'THRESHOLD_BREACH')),
    CONSTRAINT chk_alert_priority CHECK (priority IN ('CRITICAL', 'HIGH', 'MEDIUM', 'LOW')),
    CONSTRAINT chk_alert_status CHECK (status IN ('OPEN', 'ASSIGNED', 'INVESTIGATING', 'PENDING_INFO', 'ESCALATED', 'CLOSED')),
    CONSTRAINT chk_resolution CHECK (resolution_outcome IS NULL OR resolution_outcome IN ('TRUE_POSITIVE', 'FALSE_POSITIVE', 'INCONCLUSIVE', 'SAR_FILED', 'TRANSACTION_BLOCKED', 'CUSTOMER_EXITED'))
);

CREATE INDEX idx_aml_alert_reference ON aml_screening.aml_alert(alert_reference);
CREATE INDEX idx_aml_alert_status ON aml_screening.aml_alert(status);
CREATE INDEX idx_aml_alert_priority ON aml_screening.aml_alert(priority);
CREATE INDEX idx_aml_alert_assigned ON aml_screening.aml_alert(assigned_to);
CREATE INDEX idx_aml_alert_customer ON aml_screening.aml_alert(customer_id);
CREATE INDEX idx_aml_alert_due ON aml_screening.aml_alert(due_at) WHERE status NOT IN ('CLOSED');
CREATE INDEX idx_aml_alert_created ON aml_screening.aml_alert(created_at);

-- Alert Notes Table
CREATE TABLE aml_screening.alert_note (
    note_id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_id                UUID NOT NULL REFERENCES aml_screening.aml_alert(alert_id),
    note_type               VARCHAR(30) NOT NULL,  -- INVESTIGATION, ESCALATION, RESOLUTION
    content                 TEXT NOT NULL,
    created_by              VARCHAR(100) NOT NULL,
    created_at              TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_alert_note_alert ON aml_screening.alert_note(alert_id);

-- ============================================================================
-- LIST MANAGEMENT
-- ============================================================================

-- Sanctions/Watchlist Metadata Table
CREATE TABLE aml_screening.screening_list (
    list_id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    list_code               VARCHAR(50) NOT NULL UNIQUE,
    list_name               VARCHAR(200) NOT NULL,
    list_type               VARCHAR(30) NOT NULL,
    jurisdiction            VARCHAR(50),
    source_url              VARCHAR(500),
    update_frequency        VARCHAR(30),          -- DAILY, HOURLY, etc.
    current_version         VARCHAR(50),
    published_at            TIMESTAMP WITH TIME ZONE,
    loaded_at               TIMESTAMP WITH TIME ZONE,
    entry_count             INTEGER DEFAULT 0,
    status                  VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    is_mandatory            BOOLEAN DEFAULT TRUE,
    priority_order          INTEGER DEFAULT 100,
    config                  JSONB,                -- List-specific config
    created_at              TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    CONSTRAINT chk_list_type CHECK (list_type IN ('SANCTIONS', 'PEP', 'WATCHLIST', 'ADVERSE_MEDIA', 'LAW_ENFORCEMENT')),
    CONSTRAINT chk_list_status CHECK (status IN ('ACTIVE', 'LOADING', 'FAILED', 'DEPRECATED', 'PENDING'))
);

-- List Version History Table
CREATE TABLE aml_screening.list_version_history (
    version_id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    list_id                 UUID NOT NULL REFERENCES aml_screening.screening_list(list_id),
    version                 VARCHAR(50) NOT NULL,
    published_at            TIMESTAMP WITH TIME ZONE,
    loaded_at               TIMESTAMP WITH TIME ZONE NOT NULL,
    entry_count             INTEGER NOT NULL,
    entries_added           INTEGER DEFAULT 0,
    entries_removed         INTEGER DEFAULT 0,
    entries_modified        INTEGER DEFAULT 0,
    load_duration_ms        INTEGER,
    status                  VARCHAR(20) NOT NULL,
    error_message           TEXT,
    
    CONSTRAINT chk_version_status CHECK (status IN ('SUCCESS', 'FAILED', 'PARTIAL'))
);

CREATE INDEX idx_list_version_list ON aml_screening.list_version_history(list_id);
CREATE INDEX idx_list_version_loaded ON aml_screening.list_version_history(loaded_at);

-- Internal Watchlist Entries Table (for custom entries)
CREATE TABLE aml_screening.internal_watchlist_entry (
    entry_id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    list_code               VARCHAR(50) NOT NULL,
    entry_type              VARCHAR(20) NOT NULL,
    primary_name            VARCHAR(500) NOT NULL,
    aliases                 VARCHAR(500)[] DEFAULT '{}',
    date_of_birth           DATE,
    nationalities           VARCHAR(100)[] DEFAULT '{}',
    identifiers             JSONB,
    addresses               JSONB,
    reason                  TEXT NOT NULL,
    risk_level              VARCHAR(20) NOT NULL,
    added_by                VARCHAR(100) NOT NULL,
    added_at                TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    reviewed_by             VARCHAR(100),
    reviewed_at             TIMESTAMP WITH TIME ZONE,
    expires_at              TIMESTAMP WITH TIME ZONE,
    status                  VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    normalized_name         VARCHAR(500),
    name_tokens             TSVECTOR,
    
    CONSTRAINT chk_entry_type CHECK (entry_type IN ('INDIVIDUAL', 'ENTITY', 'VESSEL', 'AIRCRAFT')),
    CONSTRAINT chk_risk_level CHECK (risk_level IN ('LOW', 'MEDIUM', 'HIGH', 'CRITICAL')),
    CONSTRAINT chk_entry_status CHECK (status IN ('ACTIVE', 'REMOVED', 'EXPIRED', 'PENDING_REVIEW'))
);

CREATE INDEX idx_internal_watchlist_name ON aml_screening.internal_watchlist_entry USING gin(name_tokens);
CREATE INDEX idx_internal_watchlist_normalized ON aml_screening.internal_watchlist_entry(normalized_name);
CREATE INDEX idx_internal_watchlist_status ON aml_screening.internal_watchlist_entry(status);

-- ============================================================================
-- REGULATORY REPORTING
-- ============================================================================

-- SAR/STR Report Table
CREATE TABLE aml_screening.regulatory_report (
    report_id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_reference        VARCHAR(50) NOT NULL UNIQUE,
    report_type             VARCHAR(20) NOT NULL,  -- SAR, STR, CTR
    jurisdiction            VARCHAR(50) NOT NULL,
    alert_id                UUID REFERENCES aml_screening.aml_alert(alert_id),
    case_id                 UUID,
    customer_id             UUID,
    transaction_ids         UUID[] DEFAULT '{}',
    report_status           VARCHAR(30) NOT NULL DEFAULT 'DRAFT',
    narrative               TEXT,
    report_data             JSONB NOT NULL,        -- Full report content
    filing_deadline         DATE,
    submitted_at            TIMESTAMP WITH TIME ZONE,
    submission_reference    VARCHAR(100),          -- Reference from regulator
    acknowledgement_at      TIMESTAMP WITH TIME ZONE,
    created_by              VARCHAR(100) NOT NULL,
    created_at              TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    CONSTRAINT chk_report_type CHECK (report_type IN ('SAR', 'STR', 'CTR')),
    CONSTRAINT chk_report_status CHECK (report_status IN ('DRAFT', 'PENDING_REVIEW', 'APPROVED', 'SUBMITTED', 'ACKNOWLEDGED', 'REJECTED'))
);

CREATE INDEX idx_regulatory_report_type ON aml_screening.regulatory_report(report_type);
CREATE INDEX idx_regulatory_report_status ON aml_screening.regulatory_report(report_status);
CREATE INDEX idx_regulatory_report_alert ON aml_screening.regulatory_report(alert_id);
CREATE INDEX idx_regulatory_report_customer ON aml_screening.regulatory_report(customer_id);

-- ============================================================================
-- AUDIT LOGGING
-- ============================================================================

-- Screening Audit Log Table
CREATE TABLE aml_screening.screening_audit_log (
    audit_id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type              VARCHAR(50) NOT NULL,
    entity_type             VARCHAR(50) NOT NULL,
    entity_id               UUID NOT NULL,
    action                  VARCHAR(50) NOT NULL,
    actor                   VARCHAR(100) NOT NULL,
    actor_type              VARCHAR(30) NOT NULL,  -- SYSTEM, USER, API
    before_state            JSONB,
    after_state             JSONB,
    metadata                JSONB,
    ip_address              INET,
    user_agent              VARCHAR(500),
    event_timestamp         TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Partition by month for efficient management
CREATE INDEX idx_screening_audit_entity ON aml_screening.screening_audit_log(entity_type, entity_id);
CREATE INDEX idx_screening_audit_event ON aml_screening.screening_audit_log(event_type);
CREATE INDEX idx_screening_audit_timestamp ON aml_screening.screening_audit_log(event_timestamp);
CREATE INDEX idx_screening_audit_actor ON aml_screening.screening_audit_log(actor);

-- ============================================================================
-- CONFIGURATION TABLES
-- ============================================================================

-- Risk Factor Configuration
CREATE TABLE aml_screening.risk_factor_config (
    factor_id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    factor_code             VARCHAR(50) NOT NULL UNIQUE,
    factor_name             VARCHAR(200) NOT NULL,
    category                VARCHAR(30) NOT NULL,
    description             TEXT,
    weight                  DECIMAL(5,2) NOT NULL DEFAULT 1.0,
    score_mapping           JSONB NOT NULL,        -- Value to score mapping
    is_active               BOOLEAN DEFAULT TRUE,
    effective_from          DATE NOT NULL,
    effective_to            DATE,
    created_at              TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    CONSTRAINT chk_factor_category CHECK (category IN ('CUSTOMER', 'TRANSACTION', 'GEOGRAPHIC', 'PRODUCT', 'PATTERN'))
);

-- Match Threshold Configuration
CREATE TABLE aml_screening.match_threshold_config (
    threshold_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    list_type               VARCHAR(30) NOT NULL,
    match_algorithm         VARCHAR(30) NOT NULL,
    block_threshold         INTEGER NOT NULL CHECK (block_threshold BETWEEN 0 AND 100),
    review_threshold        INTEGER NOT NULL CHECK (review_threshold BETWEEN 0 AND 100),
    pass_threshold          INTEGER NOT NULL CHECK (pass_threshold BETWEEN 0 AND 100),
    is_active               BOOLEAN DEFAULT TRUE,
    effective_from          DATE NOT NULL,
    effective_to            DATE,
    created_at              TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    CONSTRAINT chk_threshold_order CHECK (block_threshold >= review_threshold AND review_threshold >= pass_threshold)
);

-- Country Risk Configuration
CREATE TABLE aml_screening.country_risk_config (
    country_code            VARCHAR(3) PRIMARY KEY,
    country_name            VARCHAR(200) NOT NULL,
    risk_level              VARCHAR(20) NOT NULL,
    risk_score              INTEGER NOT NULL CHECK (risk_score BETWEEN 0 AND 100),
    is_sanctioned           BOOLEAN DEFAULT FALSE,
    is_high_risk            BOOLEAN DEFAULT FALSE,
    is_tax_haven            BOOLEAN DEFAULT FALSE,
    fatf_status             VARCHAR(50),           -- GREY_LIST, BLACK_LIST, etc.
    requires_enhanced_dd    BOOLEAN DEFAULT FALSE,
    effective_from          DATE NOT NULL,
    notes                   TEXT,
    
    CONSTRAINT chk_country_risk CHECK (risk_level IN ('LOW', 'MEDIUM', 'HIGH', 'CRITICAL', 'PROHIBITED'))
);
```

### 5.3 Elasticsearch Index Schema (List Entries)

```json
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 2,
    "analysis": {
      "analyzer": {
        "name_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "asciifolding", "name_synonyms", "phonetic_filter"]
        },
        "phonetic_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "phonetic_filter"]
        }
      },
      "filter": {
        "phonetic_filter": {
          "type": "phonetic",
          "encoder": "double_metaphone",
          "replace": false
        },
        "name_synonyms": {
          "type": "synonym",
          "synonyms_path": "analysis/name_synonyms.txt"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "entry_id": { "type": "keyword" },
      "external_id": { "type": "keyword" },
      "list_code": { "type": "keyword" },
      "list_version": { "type": "keyword" },
      "entry_type": { "type": "keyword" },
      "primary_name": {
        "type": "text",
        "analyzer": "name_analyzer",
        "fields": {
          "keyword": { "type": "keyword" },
          "phonetic": { "type": "text", "analyzer": "phonetic_analyzer" }
        }
      },
      "aliases": {
        "type": "text",
        "analyzer": "name_analyzer",
        "fields": {
          "keyword": { "type": "keyword" },
          "phonetic": { "type": "text", "analyzer": "phonetic_analyzer" }
        }
      },
      "all_names": {
        "type": "text",
        "analyzer": "name_analyzer",
        "fields": {
          "phonetic": { "type": "text", "analyzer": "phonetic_analyzer" }
        }
      },
      "date_of_birth": { "type": "date", "format": "yyyy-MM-dd||yyyy" },
      "place_of_birth": { "type": "text" },
      "nationalities": { "type": "keyword" },
      "countries": { "type": "keyword" },
      "identifiers": {
        "type": "nested",
        "properties": {
          "type": { "type": "keyword" },
          "value": { "type": "keyword" },
          "country": { "type": "keyword" }
        }
      },
      "sanction_programs": { "type": "keyword" },
      "listing_reason": { "type": "text" },
      "listed_date": { "type": "date" },
      "last_updated": { "type": "date" },
      "status": { "type": "keyword" },
      "remarks": { "type": "text" },
      "risk_level": { "type": "keyword" },
      "pep_category": { "type": "keyword" },
      "pep_position": { "type": "text" },
      "name_suggest": {
        "type": "completion",
        "analyzer": "simple",
        "preserve_separators": true,
        "preserve_position_increments": true,
        "max_input_length": 50
      }
    }
  }
}
```

### 5.4 Redis Cache Schema

```
# ============================================================================
# AML SCREENING - REDIS CACHE PATTERNS
# ============================================================================

# Recent Screening Results Cache (TTL: 1 hour)
# Key: aml:result:{transactionId}
# Value: JSON serialized ScreeningResult
aml:result:{transactionId} = {
    "resultId": "uuid",
    "decision": "PASS|REVIEW|BLOCK",
    "compositeRiskScore": 45,
    "matchCount": 0,
    "screenedAt": "2026-02-20T10:30:00Z",
    "listsScreened": ["OFAC_SDN", "EU_CONSOLIDATED", "UN_SANCTIONS"]
}

# List Entry Cache (TTL: 24 hours)
# Key: aml:list:{listCode}:entry:{entryId}
# Value: JSON serialized ListEntry
aml:list:OFAC_SDN:entry:{entryId} = {
    "entryId": "12345",
    "primaryName": "John Doe",
    "aliases": ["J. Doe", "Johnny Doe"],
    "dateOfBirth": "1970-01-01",
    "nationalities": ["US", "UK"],
    "identifiers": [...],
    "status": "ACTIVE"
}

# List Version Tracker
# Key: aml:list:version:{listCode}
# Value: Version string
aml:list:version:OFAC_SDN = "2026-02-20-001"

# Name Match Cache (TTL: 6 hours)
# Key: aml:match:{normalizedName}:{listCode}
# Value: Array of potential match entry IDs with scores
aml:match:{normalizedName}:{listCode} = [
    {"entryId": "123", "score": 95, "algorithm": "JARO_WINKLER"},
    {"entryId": "456", "score": 87, "algorithm": "SOUNDEX"}
]

# Customer Risk Score Cache (TTL: 4 hours)
# Key: aml:customer:risk:{customerId}
# Value: Customer risk profile
aml:customer:risk:{customerId} = {
    "customerId": "uuid",
    "riskScore": 35,
    "riskLevel": "MEDIUM",
    "factors": [...],
    "calculatedAt": "2026-02-20T08:00:00Z"
}

# Country Risk Cache (TTL: 24 hours)
# Key: aml:country:risk:{countryCode}
# Value: Country risk data
aml:country:risk:IR = {
    "countryCode": "IR",
    "riskLevel": "PROHIBITED",
    "riskScore": 100,
    "isSanctioned": true,
    "fatfStatus": "BLACK_LIST"
}

# Idempotency Cache (TTL: 24 hours)
# Key: aml:idempotency:{requestHash}
# Value: Result ID
aml:idempotency:{requestHash} = "result-uuid"

# Bloom Filter for Quick Negative Checks
# Key: aml:bloomfilter:{listCode}
# Purpose: Quick check if name definitely NOT in list
aml:bloomfilter:OFAC_SDN = [bloom filter binary data]

# Rate Limiting
# Key: aml:ratelimit:{clientId}:{window}
# Value: Request count
aml:ratelimit:client123:2026-02-20-10 = 1500
```

### 5.5 TimescaleDB Schema (Pattern Analysis)

```sql
-- ============================================================================
-- AML PATTERN ANALYSIS - TimescaleDB Hypertables
-- ============================================================================

-- Enable TimescaleDB extension
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Customer Transaction History (Hypertable)
CREATE TABLE aml_patterns.customer_transaction_history (
    time                    TIMESTAMPTZ NOT NULL,
    customer_id             UUID NOT NULL,
    transaction_id          UUID NOT NULL,
    amount                  DECIMAL(19,4) NOT NULL,
    currency                VARCHAR(3) NOT NULL,
    amount_usd              DECIMAL(19,4) NOT NULL,  -- Normalized amount
    payment_type            VARCHAR(30) NOT NULL,
    channel                 VARCHAR(30) NOT NULL,
    origin_country          VARCHAR(3),
    destination_country     VARCHAR(3),
    beneficiary_id          UUID,
    is_new_beneficiary      BOOLEAN DEFAULT FALSE,
    screening_decision      VARCHAR(20),
    risk_score              INTEGER
);

-- Convert to hypertable partitioned by time
SELECT create_hypertable('aml_patterns.customer_transaction_history', 'time',
    chunk_time_interval => INTERVAL '1 day');

-- Compression policy (compress chunks older than 7 days)
ALTER TABLE aml_patterns.customer_transaction_history SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'customer_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('aml_patterns.customer_transaction_history', INTERVAL '7 days');

-- Retention policy (drop chunks older than 2 years)
SELECT add_retention_policy('aml_patterns.customer_transaction_history', INTERVAL '2 years');

-- Continuous Aggregates for Pattern Detection
-- Hourly aggregates
CREATE MATERIALIZED VIEW aml_patterns.customer_hourly_stats
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    customer_id,
    COUNT(*) AS transaction_count,
    SUM(amount_usd) AS total_amount_usd,
    AVG(amount_usd) AS avg_amount_usd,
    MAX(amount_usd) AS max_amount_usd,
    COUNT(DISTINCT beneficiary_id) AS unique_beneficiaries,
    COUNT(DISTINCT destination_country) AS unique_destinations,
    SUM(CASE WHEN is_new_beneficiary THEN 1 ELSE 0 END) AS new_beneficiary_count
FROM aml_patterns.customer_transaction_history
GROUP BY time_bucket('1 hour', time), customer_id;

-- Daily aggregates
CREATE MATERIALIZED VIEW aml_patterns.customer_daily_stats
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    customer_id,
    COUNT(*) AS transaction_count,
    SUM(amount_usd) AS total_amount_usd,
    AVG(amount_usd) AS avg_amount_usd,
    MAX(amount_usd) AS max_amount_usd,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY amount_usd) AS p95_amount,
    COUNT(DISTINCT beneficiary_id) AS unique_beneficiaries,
    COUNT(DISTINCT destination_country) AS unique_destinations
FROM aml_patterns.customer_transaction_history
GROUP BY time_bucket('1 day', time), customer_id;

-- Structuring Detection View
CREATE MATERIALIZED VIEW aml_patterns.structuring_detection
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('24 hours', time) AS bucket,
    customer_id,
    COUNT(*) AS transaction_count,
    SUM(amount_usd) AS total_amount_usd,
    COUNT(*) FILTER (WHERE amount_usd BETWEEN 9000 AND 10000) AS near_threshold_count,
    SUM(amount_usd) FILTER (WHERE amount_usd BETWEEN 9000 AND 10000) AS near_threshold_total
FROM aml_patterns.customer_transaction_history
GROUP BY time_bucket('24 hours', time), customer_id
HAVING COUNT(*) >= 3;  -- At least 3 transactions
```

### 5.6 Data Flow Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       AML DATA FLOW ARCHITECTURE                             │
└─────────────────────────────────────────────────────────────────────────────┘

                        ┌───────────────────────────────────────┐
                        │       LIST DATA SOURCES               │
                        │                                       │
                        │  OFAC  │  EU  │  UN  │  World-Check  │
                        │  API   │  API │  XML │     API       │
                        └───────────────────────────────────────┘
                                        │
                                        ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                           LIST INGESTION PIPELINE                             │
│                                                                               │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────────────┐   │
│  │  Fetcher   │──►│  Parser    │──►│  Normalizer│──►│   Elasticsearch    │   │
│  │  (Pull)    │   │  (Transform)│   │  (Index)   │   │   Index Writer     │   │
│  └────────────┘   └────────────┘   └────────────┘   └────────────────────┘   │
│                                                               │               │
│                                           ┌───────────────────┘               │
│                                           ▼                                   │
│                              ┌─────────────────────────┐                     │
│                              │  Redis Cache Warming    │                     │
│                              │  (Bloom Filter Update)  │                     │
│                              └─────────────────────────┘                     │
└───────────────────────────────────────────────────────────────────────────────┘

                                        │
                                        ▼

┌───────────────────────────────────────────────────────────────────────────────┐
│                        REAL-TIME SCREENING FLOW                               │
│                                                                               │
│   Payment Orchestration                                                       │
│         │                                                                     │
│         ▼                                                                     │
│   ┌───────────┐                                                              │
│   │ Screening │    ┌────────────────────────────────────────────────────┐   │
│   │ Request   │───►│              SCREENING ORCHESTRATOR                │   │
│   └───────────┘    │                                                    │   │
│                    │  ┌──────────────────────────────────────────────┐ │   │
│                    │  │          PARALLEL SCREENING                  │ │   │
│                    │  │                                              │ │   │
│                    │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  │ │   │
│                    │  │  │Sanctions │  │   PEP    │  │Watchlist │  │ │   │
│                    │  │  │Screener  │  │ Screener │  │ Screener │  │ │   │
│                    │  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  │ │   │
│                    │  │       │             │             │        │ │   │
│                    │  │       ▼             ▼             ▼        │ │   │
│                    │  │  ┌─────────────────────────────────────┐  │ │   │
│                    │  │  │          ELASTICSEARCH               │  │ │   │
│                    │  │  │       (Name Matching Queries)        │  │ │   │
│                    │  │  └─────────────────────────────────────┘  │ │   │
│                    │  │                     │                      │ │   │
│                    │  └─────────────────────┼──────────────────────┘ │   │
│                    │                        │                        │   │
│                    │  ┌─────────────────────▼──────────────────────┐ │   │
│                    │  │           RESULT AGGREGATION               │ │   │
│                    │  │  • Merge matches  • Calculate scores       │ │   │
│                    │  │  • Apply decision logic                    │ │   │
│                    │  └─────────────────────┬──────────────────────┘ │   │
│                    │                        │                        │   │
│                    └────────────────────────┼────────────────────────┘   │
│                                             │                             │
│         ┌───────────────────────────────────┼───────────────────────┐    │
│         │                                   ▼                       │    │
│         │   ┌───────────────┐   ┌───────────────┐   ┌─────────────┐│    │
│         │   │  PostgreSQL   │   │   Redis       │   │ TimescaleDB ││    │
│         │   │  (Results,    │   │   (Cache)     │   │ (Patterns)  ││    │
│         │   │   Alerts)     │   │               │   │             ││    │
│         │   └───────────────┘   └───────────────┘   └─────────────┘│    │
│         │                                                           │    │
│         └───────────────────────────────────────────────────────────┘    │
│                                                                           │
│                                             │                             │
│                                             ▼                             │
│                                   ┌─────────────────┐                    │
│                                   │ Screening       │                    │
│                                   │ Response        │                    │
│                                   └─────────────────┘                    │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 6. API Design

### 6.1 API Overview

| API Endpoint | Method | Description | SLA |
|--------------|--------|-------------|-----|
| `/api/v1/screening/transaction` | POST | Screen transaction in real-time | < 200ms |
| `/api/v1/screening/batch` | POST | Submit batch screening request | < 5s |
| `/api/v1/screening/subject` | POST | Screen individual subject | < 100ms |
| `/api/v1/screening/{requestId}` | GET | Get screening result | < 50ms |
| `/api/v1/screening/{requestId}/matches` | GET | Get match details | < 50ms |
| `/api/v1/alerts` | GET | List alerts with filters | < 100ms |
| `/api/v1/alerts/{alertId}` | GET | Get alert details | < 50ms |
| `/api/v1/alerts/{alertId}/assign` | POST | Assign alert to analyst | < 50ms |
| `/api/v1/alerts/{alertId}/resolve` | POST | Resolve alert | < 100ms |
| `/api/v1/lists` | GET | List available screening lists | < 50ms |
| `/api/v1/lists/{listCode}/entries` | GET | Search list entries | < 100ms |
| `/api/v1/watchlist/entries` | POST | Add internal watchlist entry | < 100ms |
| `/api/v1/reports/sar` | POST | Generate SAR report | < 5s |

### 6.2 Screening API Endpoints

#### 6.2.1 Screen Transaction (Real-Time)

```yaml
POST /api/v1/screening/transaction
Content-Type: application/json
Authorization: Bearer {token}
X-Request-ID: {correlation-id}
X-Idempotency-Key: {idempotency-key}

Request:
{
  "transactionId": "550e8400-e29b-41d4-a716-446655440000",
  "requestType": "REAL_TIME",
  "priority": "HIGH",
  "transactionDetails": {
    "amount": 50000.00,
    "currency": "USD",
    "paymentType": "WIRE_TRANSFER",
    "channel": "ONLINE_BANKING",
    "valueDate": "2026-02-20",
    "originCountry": "US",
    "destinationCountry": "DE"
  },
  "subjects": [
    {
      "subjectType": "ORIGINATOR",
      "entityType": "INDIVIDUAL",
      "name": {
        "fullName": "John Michael Smith",
        "firstName": "John",
        "middleName": "Michael",
        "lastName": "Smith"
      },
      "dateOfBirth": "1975-03-15",
      "nationalities": ["US"],
      "countryOfResidence": "US",
      "addresses": [
        {
          "addressType": "RESIDENCE",
          "line1": "123 Main Street",
          "city": "New York",
          "state": "NY",
          "postalCode": "10001",
          "country": "US"
        }
      ],
      "identifiers": [
        {
          "type": "PASSPORT",
          "value": "123456789",
          "issuingCountry": "US"
        }
      ]
    },
    {
      "subjectType": "BENEFICIARY",
      "entityType": "ORGANIZATION",
      "name": {
        "fullName": "Acme GmbH"
      },
      "countryOfResidence": "DE",
      "addresses": [
        {
          "addressType": "BUSINESS",
          "line1": "Friedrichstrasse 123",
          "city": "Berlin",
          "country": "DE"
        }
      ],
      "identifiers": [
        {
          "type": "LEI",
          "value": "529900T8BM49AURSDO55"
        }
      ]
    }
  ],
  "screeningConfig": {
    "listsToScreen": ["OFAC_SDN", "EU_CONSOLIDATED", "UN_SANCTIONS", "PEP_GLOBAL"],
    "includePatternAnalysis": true,
    "includeRiskScoring": true,
    "matchThresholds": {
      "sanctions": {
        "blockThreshold": 90,
        "reviewThreshold": 80
      },
      "pep": {
        "reviewThreshold": 85
      }
    }
  },
  "context": {
    "customerId": "cust-12345",
    "accountId": "acct-67890",
    "customerRiskProfile": "MEDIUM",
    "customerSince": "2020-01-15"
  }
}

Response (200 OK):
{
  "requestId": "req-550e8400-e29b-41d4-a716-446655440001",
  "transactionId": "550e8400-e29b-41d4-a716-446655440000",
  "decision": "PASS",
  "compositeRiskScore": 35,
  "riskLevel": "MEDIUM",
  "screeningDurationMs": 156,
  "screenedAt": "2026-02-20T10:30:15.123Z",
  "summary": {
    "totalSubjectsScreened": 2,
    "totalListsScreened": 4,
    "matchesFound": 0,
    "blockingMatches": 0,
    "reviewMatches": 0
  },
  "listsScreened": [
    {
      "listCode": "OFAC_SDN",
      "version": "2026-02-20-001",
      "entriesChecked": 12500
    },
    {
      "listCode": "EU_CONSOLIDATED",
      "version": "2026-02-19-002",
      "entriesChecked": 8200
    },
    {
      "listCode": "UN_SANCTIONS",
      "version": "2026-02-18-001",
      "entriesChecked": 3100
    },
    {
      "listCode": "PEP_GLOBAL",
      "version": "2026-02-20-001",
      "entriesChecked": 1500000
    }
  ],
  "riskScoreBreakdown": {
    "customerRiskScore": 30,
    "transactionRiskScore": 40,
    "geographicRiskScore": 25,
    "matchRiskScore": 0,
    "patternRiskScore": 45
  },
  "patternAnalysis": {
    "overallPatternRisk": "LOW",
    "structuringScore": 10,
    "velocityScore": 25,
    "behaviorDeviationScore": 15,
    "patternsDetected": []
  },
  "matches": [],
  "alertReference": null,
  "caseReference": null
}

Response (200 OK - With Match):
{
  "requestId": "req-550e8400-e29b-41d4-a716-446655440002",
  "transactionId": "550e8400-e29b-41d4-a716-446655440003",
  "decision": "REVIEW",
  "compositeRiskScore": 72,
  "riskLevel": "HIGH",
  "screeningDurationMs": 189,
  "screenedAt": "2026-02-20T10:31:22.456Z",
  "summary": {
    "totalSubjectsScreened": 2,
    "totalListsScreened": 4,
    "matchesFound": 1,
    "blockingMatches": 0,
    "reviewMatches": 1
  },
  "matches": [
    {
      "matchId": "match-12345",
      "subjectType": "BENEFICIARY",
      "subjectName": "Acme GmbH",
      "matchType": "PEP_MATCH",
      "matchConfidence": 87,
      "matchAlgorithm": "JARO_WINKLER",
      "recommendedAction": "REVIEW",
      "matchedEntry": {
        "listCode": "PEP_GLOBAL",
        "entryId": "pep-67890",
        "matchedName": "ACME Holdings GmbH",
        "pepCategory": "STATE_OWNED_ENTERPRISE",
        "relatedPep": "Minister of Trade, Germany",
        "listingDate": "2024-05-15"
      },
      "matchedFields": [
        {
          "fieldName": "name",
          "subjectValue": "Acme GmbH",
          "listValue": "ACME Holdings GmbH",
          "matchScore": 87,
          "algorithm": "JARO_WINKLER"
        }
      ]
    }
  ],
  "alertReference": "AML-2026-02-20-00145",
  "caseReference": null
}
```

#### 6.2.2 Screen Individual Subject

```yaml
POST /api/v1/screening/subject
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
  "requestType": "ON_DEMAND",
  "priority": "MEDIUM",
  "subject": {
    "subjectType": "ORIGINATOR",
    "entityType": "INDIVIDUAL",
    "name": {
      "fullName": "Vladimir Petrovich Ivanov",
      "firstName": "Vladimir",
      "middleName": "Petrovich",
      "lastName": "Ivanov"
    },
    "dateOfBirth": "1965-08-20",
    "nationalities": ["RU"],
    "countryOfResidence": "RU"
  },
  "listsToScreen": ["OFAC_SDN", "EU_CONSOLIDATED", "UK_SANCTIONS"],
  "context": {
    "purpose": "CUSTOMER_ONBOARDING",
    "requestedBy": "kyc-team"
  }
}

Response (200 OK):
{
  "requestId": "req-abc123",
  "decision": "BLOCK",
  "compositeRiskScore": 98,
  "matches": [
    {
      "matchId": "match-xyz789",
      "matchType": "SANCTIONS_EXACT",
      "matchConfidence": 98,
      "recommendedAction": "BLOCK",
      "matchedEntry": {
        "listCode": "OFAC_SDN",
        "entryId": "ofac-12345",
        "matchedName": "IVANOV, Vladimir Petrovich",
        "sanctionPrograms": ["RUSSIA-EO14024", "UKRAINE-EO13661"],
        "listingReason": "Government official involved in sanctions evasion",
        "listedDate": "2022-03-15"
      }
    }
  ],
  "alertReference": "AML-2026-02-20-00146"
}
```

#### 6.2.3 Batch Screening Request

```yaml
POST /api/v1/screening/batch
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
  "batchId": "batch-20260220-001",
  "requestType": "BATCH",
  "priority": "LOW",
  "callbackUrl": "https://api.internal/callbacks/aml-batch",
  "subjects": [
    {
      "externalRef": "cust-001",
      "entityType": "INDIVIDUAL",
      "name": { "fullName": "John Doe" },
      "nationalities": ["US"]
    },
    {
      "externalRef": "cust-002",
      "entityType": "ORGANIZATION",
      "name": { "fullName": "Global Trading Ltd" },
      "countryOfResidence": "UK"
    }
    // ... up to 10,000 subjects per batch
  ],
  "listsToScreen": ["OFAC_SDN", "EU_CONSOLIDATED", "UN_SANCTIONS"],
  "options": {
    "includeNoMatchRecords": false,
    "matchThreshold": 80
  }
}

Response (202 Accepted):
{
  "batchId": "batch-20260220-001",
  "requestId": "req-batch-001",
  "status": "ACCEPTED",
  "totalSubjects": 2,
  "estimatedCompletionTime": "2026-02-20T11:00:00Z",
  "statusUrl": "/api/v1/screening/batch/batch-20260220-001/status"
}
```

### 6.3 Alert Management API

#### 6.3.1 List Alerts

```yaml
GET /api/v1/alerts?status=OPEN&priority=HIGH&limit=50&offset=0
Authorization: Bearer {token}

Response (200 OK):
{
  "alerts": [
    {
      "alertId": "alert-12345",
      "alertReference": "AML-2026-02-20-00145",
      "transactionId": "txn-67890",
      "alertType": "PEP_HIT",
      "priority": "HIGH",
      "status": "OPEN",
      "riskScore": 72,
      "matchCount": 1,
      "createdAt": "2026-02-20T10:31:22Z",
      "dueAt": "2026-02-20T18:31:22Z",
      "assignedTo": null,
      "summary": "Potential PEP match on beneficiary: Acme GmbH"
    }
  ],
  "pagination": {
    "total": 127,
    "limit": 50,
    "offset": 0,
    "hasMore": true
  }
}
```

#### 6.3.2 Get Alert Details

```yaml
GET /api/v1/alerts/{alertId}
Authorization: Bearer {token}

Response (200 OK):
{
  "alertId": "alert-12345",
  "alertReference": "AML-2026-02-20-00145",
  "transactionId": "txn-67890",
  "alertType": "PEP_HIT",
  "priority": "HIGH",
  "status": "OPEN",
  "riskScore": 72,
  "createdAt": "2026-02-20T10:31:22Z",
  "dueAt": "2026-02-20T18:31:22Z",
  "transaction": {
    "transactionId": "txn-67890",
    "amount": 50000.00,
    "currency": "USD",
    "paymentType": "WIRE_TRANSFER",
    "originCountry": "US",
    "destinationCountry": "DE",
    "valueDate": "2026-02-20"
  },
  "customer": {
    "customerId": "cust-12345",
    "customerName": "John Smith",
    "customerRiskProfile": "MEDIUM",
    "customerSince": "2020-01-15"
  },
  "matches": [
    {
      "matchId": "match-xyz789",
      "subjectType": "BENEFICIARY",
      "subjectName": "Acme GmbH",
      "matchType": "PEP_MATCH",
      "matchConfidence": 87,
      "matchedEntry": {
        "listCode": "PEP_GLOBAL",
        "entryId": "pep-67890",
        "matchedName": "ACME Holdings GmbH",
        "pepCategory": "STATE_OWNED_ENTERPRISE",
        "relatedPep": "Minister of Trade, Germany",
        "pepDetails": {
          "position": "Minister of Trade",
          "country": "Germany",
          "startDate": "2021-03-01",
          "riskLevel": "HIGH"
        }
      },
      "evidence": {
        "nameMatchScore": 87,
        "algorithm": "JARO_WINKLER",
        "countryMatch": true,
        "additionalFactors": ["SOE connection", "High-value transaction"]
      }
    }
  ],
  "riskFactors": [
    { "factor": "PEP_SOE_CONNECTION", "weight": 0.3, "contribution": 25 },
    { "factor": "HIGH_VALUE_TRANSACTION", "weight": 0.2, "contribution": 20 },
    { "factor": "CROSS_BORDER", "weight": 0.1, "contribution": 10 }
  ],
  "notes": [],
  "auditTrail": [
    {
      "action": "ALERT_CREATED",
      "actor": "SYSTEM",
      "timestamp": "2026-02-20T10:31:22Z"
    }
  ]
}
```

#### 6.3.3 Assign Alert

```yaml
POST /api/v1/alerts/{alertId}/assign
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
  "assignTo": "analyst.john@company.com",
  "notes": "Assigned for expedited review due to high-value transaction"
}

Response (200 OK):
{
  "alertId": "alert-12345",
  "status": "ASSIGNED",
  "assignedTo": "analyst.john@company.com",
  "assignedAt": "2026-02-20T10:45:00Z",
  "dueAt": "2026-02-20T18:31:22Z"
}
```

#### 6.3.4 Resolve Alert

```yaml
POST /api/v1/alerts/{alertId}/resolve
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
  "resolution": "FALSE_POSITIVE",
  "notes": "Verified beneficiary is not connected to the PEP identified. Different legal entity with similar name.",
  "evidence": {
    "verificationMethod": "MANUAL_REVIEW",
    "documentsReviewed": ["corporate_registry", "beneficial_ownership"],
    "additionalChecks": ["LEI verification confirmed different entity"]
  },
  "fileSar": false
}

Response (200 OK):
{
  "alertId": "alert-12345",
  "status": "CLOSED",
  "resolution": "FALSE_POSITIVE",
  "resolvedBy": "analyst.john@company.com",
  "resolvedAt": "2026-02-20T14:30:00Z",
  "transactionReleased": true
}
```

### 6.4 List Management API

#### 6.4.1 List Available Screening Lists

```yaml
GET /api/v1/lists
Authorization: Bearer {token}

Response (200 OK):
{
  "lists": [
    {
      "listCode": "OFAC_SDN",
      "listName": "OFAC Specially Designated Nationals",
      "listType": "SANCTIONS",
      "jurisdiction": "US",
      "currentVersion": "2026-02-20-001",
      "lastUpdated": "2026-02-20T06:00:00Z",
      "entryCount": 12500,
      "status": "ACTIVE",
      "isMandatory": true
    },
    {
      "listCode": "EU_CONSOLIDATED",
      "listName": "EU Consolidated Sanctions List",
      "listType": "SANCTIONS",
      "jurisdiction": "EU",
      "currentVersion": "2026-02-19-002",
      "lastUpdated": "2026-02-19T18:00:00Z",
      "entryCount": 8200,
      "status": "ACTIVE",
      "isMandatory": true
    },
    {
      "listCode": "PEP_GLOBAL",
      "listName": "Global PEP Database",
      "listType": "PEP",
      "jurisdiction": "GLOBAL",
      "currentVersion": "2026-02-20-001",
      "lastUpdated": "2026-02-20T04:00:00Z",
      "entryCount": 1500000,
      "status": "ACTIVE",
      "isMandatory": true
    }
  ]
}
```

#### 6.4.2 Search List Entries

```yaml
GET /api/v1/lists/{listCode}/entries?name=vladimir&type=INDIVIDUAL&limit=20
Authorization: Bearer {token}

Response (200 OK):
{
  "listCode": "OFAC_SDN",
  "searchQuery": {
    "name": "vladimir",
    "type": "INDIVIDUAL"
  },
  "results": [
    {
      "entryId": "ofac-12345",
      "externalId": "12345",
      "entryType": "INDIVIDUAL",
      "primaryName": "IVANOV, Vladimir Petrovich",
      "aliases": ["IVANOV, V.P.", "Vladimir IVANOV"],
      "dateOfBirth": "1965-08-20",
      "nationalities": ["RU"],
      "sanctionPrograms": ["RUSSIA-EO14024"],
      "listingReason": "Government official",
      "listedDate": "2022-03-15",
      "status": "ACTIVE"
    }
  ],
  "total": 1,
  "limit": 20
}
```

#### 6.4.3 Add Internal Watchlist Entry

```yaml
POST /api/v1/watchlist/entries
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
  "listCode": "INTERNAL_BLACKLIST",
  "entryType": "INDIVIDUAL",
  "primaryName": "John Bad Actor",
  "aliases": ["J. Actor", "Johnny Bad"],
  "dateOfBirth": "1980-05-10",
  "nationalities": ["US"],
  "identifiers": [
    {
      "type": "PASSPORT",
      "value": "AB1234567",
      "issuingCountry": "US"
    }
  ],
  "reason": "Confirmed involvement in fraudulent activity per case CASE-2026-001",
  "riskLevel": "CRITICAL",
  "expiresAt": null,
  "caseReference": "CASE-2026-001"
}

Response (201 Created):
{
  "entryId": "wl-entry-12345",
  "listCode": "INTERNAL_BLACKLIST",
  "primaryName": "John Bad Actor",
  "status": "ACTIVE",
  "addedBy": "compliance.officer@company.com",
  "addedAt": "2026-02-20T15:00:00Z",
  "effectiveImmediately": true
}
```

### 6.5 Regulatory Reporting API

#### 6.5.1 Generate SAR Report

```yaml
POST /api/v1/reports/sar
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
  "alertId": "alert-12345",
  "jurisdiction": "US",
  "filingType": "INITIAL",
  "suspiciousActivity": {
    "activityType": ["MONEY_LAUNDERING", "STRUCTURING"],
    "activityDescription": "Multiple transactions below reporting threshold to known high-risk jurisdiction",
    "dateRangeStart": "2026-01-01",
    "dateRangeEnd": "2026-02-20"
  },
  "subjectInformation": {
    "subjectType": "INDIVIDUAL",
    "name": "John Smith",
    "dateOfBirth": "1975-03-15",
    "identifiers": [
      { "type": "SSN", "value": "XXX-XX-1234" }
    ],
    "addresses": [
      {
        "line1": "123 Main Street",
        "city": "New York",
        "state": "NY",
        "postalCode": "10001",
        "country": "US"
      }
    ],
    "accountNumbers": ["ACCT-12345", "ACCT-67890"],
    "relationshipToFI": "CUSTOMER"
  },
  "transactionSummary": {
    "totalTransactions": 15,
    "totalAmount": 145000.00,
    "currency": "USD",
    "transactionTypes": ["WIRE_TRANSFER", "ACH"]
  },
  "narrative": "Customer conducted 15 wire transfers over 6-week period...",
  "lawEnforcementContact": false
}

Response (201 Created):
{
  "reportId": "sar-2026-001234",
  "reportReference": "SAR-2026-02-20-00001",
  "reportType": "SAR",
  "status": "DRAFT",
  "filingDeadline": "2026-03-22",
  "createdAt": "2026-02-20T16:00:00Z",
  "createdBy": "compliance.officer@company.com",
  "reviewUrl": "/api/v1/reports/sar/sar-2026-001234"
}
```

### 6.6 Error Responses

```yaml
# Standard Error Response Format
{
  "error": {
    "code": "AML_SCREENING_TIMEOUT",
    "message": "Screening request timed out after 30 seconds",
    "details": {
      "requestId": "req-12345",
      "timeoutMs": 30000,
      "completedLists": ["OFAC_SDN", "EU_CONSOLIDATED"],
      "pendingLists": ["PEP_GLOBAL"]
    },
    "timestamp": "2026-02-20T10:30:00Z",
    "traceId": "x-trace-12345"
  }
}

# Error Codes
| Code | HTTP Status | Description |
|------|-------------|-------------|
| AML_INVALID_REQUEST | 400 | Invalid request format or missing required fields |
| AML_SUBJECT_REQUIRED | 400 | At least one screening subject is required |
| AML_INVALID_LIST_CODE | 400 | Unknown or inactive screening list code |
| AML_UNAUTHORIZED | 401 | Invalid or missing authentication |
| AML_FORBIDDEN | 403 | Insufficient permissions for operation |
| AML_REQUEST_NOT_FOUND | 404 | Screening request not found |
| AML_ALERT_NOT_FOUND | 404 | Alert not found |
| AML_LIST_NOT_FOUND | 404 | Screening list not found |
| AML_DUPLICATE_REQUEST | 409 | Duplicate idempotency key |
| AML_ALERT_ALREADY_RESOLVED | 409 | Alert has already been resolved |
| AML_RATE_LIMITED | 429 | Too many requests, rate limit exceeded |
| AML_SCREENING_TIMEOUT | 504 | Screening request timed out |
| AML_LIST_UNAVAILABLE | 503 | Screening list temporarily unavailable |
| AML_INTERNAL_ERROR | 500 | Internal server error |
```

### 6.7 API Authentication & Rate Limiting

```yaml
# Authentication
- All endpoints require Bearer token authentication
- Token must be obtained via OAuth 2.0 client credentials flow
- Token scopes: aml:screen, aml:alerts:read, aml:alerts:write, aml:lists:read, aml:lists:write, aml:reports:write

# Rate Limits
| Tier | Real-Time Screening | Batch Screening | Alerts | Lists |
|------|---------------------|-----------------|--------|-------|
| Standard | 1,000/min | 10/min | 500/min | 100/min |
| Premium | 5,000/min | 50/min | 2,000/min | 500/min |
| Enterprise | 20,000/min | 200/min | 10,000/min | 2,000/min |

# Rate Limit Headers
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4832
X-RateLimit-Reset: 1645358400
```

---

## 7. Event-Driven Architecture

### 7.1 Event Catalog

| Event Name | Publisher | Description | Consumers |
|------------|-----------|-------------|-----------|
| `ScreeningRequested` | Limit Control Engine | Transaction requires AML screening | AML Screening Service |
| `ScreeningCompleted` | AML Screening Service | Screening finished with decision | Payment Orchestration, Audit |
| `ScreeningFailed` | AML Screening Service | Screening failed (technical error) | Payment Orchestration, Monitoring |
| `MatchFound` | AML Screening Service | Potential match detected | Alert Service, Case Management |
| `AlertCreated` | AML Screening Service | New alert generated | Case Management, Notifications |
| `AlertAssigned` | Alert Service | Alert assigned to analyst | Notifications, Dashboard |
| `AlertEscalated` | Alert Service | Alert escalated | Notifications, Management Dashboard |
| `AlertResolved` | Alert Service | Alert investigation completed | Payment Orchestration, Audit |
| `SarGenerated` | Reporting Service | SAR report generated | Compliance Dashboard, Audit |
| `SarSubmitted` | Reporting Service | SAR submitted to regulator | Compliance Dashboard, Audit |
| `ListUpdated` | List Management Service | Screening list updated | AML Screening Service, Cache |
| `CustomerRescreenRequired` | Customer Risk Service | Customer requires re-screening | AML Screening Service |

### 7.2 Event Schemas

```json
// ScreeningRequested Event
{
  "eventId": "evt-550e8400-e29b-41d4-a716-446655440000",
  "eventType": "ScreeningRequested",
  "eventVersion": "1.0",
  "timestamp": "2026-02-20T10:30:00.123Z",
  "source": "limit-control-engine",
  "correlationId": "corr-12345",
  "data": {
    "transactionId": "txn-67890",
    "requestPriority": "HIGH",
    "triggerReason": "AMOUNT_THRESHOLD_EXCEEDED",
    "triggerDetails": {
      "threshold": 10000,
      "transactionAmount": 50000,
      "currency": "USD"
    },
    "subjects": [
      {
        "subjectType": "ORIGINATOR",
        "name": "John Smith",
        "entityType": "INDIVIDUAL"
      },
      {
        "subjectType": "BENEFICIARY",
        "name": "Acme GmbH",
        "entityType": "ORGANIZATION"
      }
    ],
    "screeningConfig": {
      "listsRequired": ["OFAC_SDN", "EU_CONSOLIDATED", "UN_SANCTIONS", "PEP_GLOBAL"],
      "includePatternAnalysis": true,
      "timeoutMs": 30000
    }
  }
}

// ScreeningCompleted Event
{
  "eventId": "evt-550e8400-e29b-41d4-a716-446655440001",
  "eventType": "ScreeningCompleted",
  "eventVersion": "1.0",
  "timestamp": "2026-02-20T10:30:00.289Z",
  "source": "aml-screening-service",
  "correlationId": "corr-12345",
  "data": {
    "requestId": "req-12345",
    "transactionId": "txn-67890",
    "decision": "PASS",
    "compositeRiskScore": 35,
    "riskLevel": "MEDIUM",
    "screeningDurationMs": 156,
    "summary": {
      "subjectsScreened": 2,
      "listsScreened": 4,
      "matchesFound": 0,
      "blockingMatches": 0,
      "reviewMatches": 0
    },
    "continuation": {
      "action": "PROCEED",
      "nextStep": "CONTINUE_TO_ROUTING"
    }
  }
}

// MatchFound Event
{
  "eventId": "evt-550e8400-e29b-41d4-a716-446655440002",
  "eventType": "MatchFound",
  "eventVersion": "1.0",
  "timestamp": "2026-02-20T10:30:00.245Z",
  "source": "aml-screening-service",
  "correlationId": "corr-12345",
  "data": {
    "requestId": "req-12345",
    "transactionId": "txn-67890",
    "matchId": "match-xyz789",
    "matchType": "PEP_MATCH",
    "matchConfidence": 87,
    "recommendedAction": "REVIEW",
    "subject": {
      "subjectType": "BENEFICIARY",
      "name": "Acme GmbH"
    },
    "matchedEntry": {
      "listCode": "PEP_GLOBAL",
      "entryId": "pep-67890",
      "matchedName": "ACME Holdings GmbH",
      "category": "STATE_OWNED_ENTERPRISE"
    }
  }
}

// AlertCreated Event
{
  "eventId": "evt-550e8400-e29b-41d4-a716-446655440003",
  "eventType": "AlertCreated",
  "eventVersion": "1.0",
  "timestamp": "2026-02-20T10:30:00.350Z",
  "source": "aml-screening-service",
  "correlationId": "corr-12345",
  "data": {
    "alertId": "alert-12345",
    "alertReference": "AML-2026-02-20-00145",
    "transactionId": "txn-67890",
    "customerId": "cust-12345",
    "alertType": "PEP_HIT",
    "priority": "HIGH",
    "riskScore": 72,
    "matchCount": 1,
    "slaDeadline": "2026-02-20T18:30:00Z",
    "requiresImmediateAction": false,
    "transactionHeld": true
  }
}

// AlertResolved Event
{
  "eventId": "evt-550e8400-e29b-41d4-a716-446655440004",
  "eventType": "AlertResolved",
  "eventVersion": "1.0",
  "timestamp": "2026-02-20T14:30:00.000Z",
  "source": "alert-management-service",
  "correlationId": "corr-12345",
  "data": {
    "alertId": "alert-12345",
    "alertReference": "AML-2026-02-20-00145",
    "transactionId": "txn-67890",
    "resolution": "FALSE_POSITIVE",
    "resolvedBy": "analyst.john@company.com",
    "resolutionNotes": "Verified different legal entity",
    "sarFiled": false,
    "transactionAction": "RELEASE",
    "continuation": {
      "action": "RELEASE_TRANSACTION",
      "transactionId": "txn-67890"
    }
  }
}

// ListUpdated Event
{
  "eventId": "evt-550e8400-e29b-41d4-a716-446655440005",
  "eventType": "ListUpdated",
  "eventVersion": "1.0",
  "timestamp": "2026-02-20T06:00:00.000Z",
  "source": "list-management-service",
  "data": {
    "listCode": "OFAC_SDN",
    "listName": "OFAC SDN List",
    "previousVersion": "2026-02-19-001",
    "newVersion": "2026-02-20-001",
    "updateSummary": {
      "entriesAdded": 15,
      "entriesRemoved": 3,
      "entriesModified": 22,
      "totalEntries": 12500
    },
    "significantChanges": [
      {
        "changeType": "ADDED",
        "entryId": "ofac-new-001",
        "name": "New Sanctioned Entity Ltd",
        "programs": ["RUSSIA-EO14024"]
      }
    ],
    "rescreenRequired": true,
    "rescreenWindow": "24h"
  }
}
```

### 7.3 Event Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AML EVENT FLOW ARCHITECTURE                           │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐                           ┌──────────────────────────────┐
│ Payment          │                           │     KAFKA TOPICS             │
│ Orchestration    │                           │                              │
│ Domain           │                           │  ┌────────────────────────┐ │
│                  │   ScreeningRequested      │  │ aml.screening.requests │ │
│  Limit Control   │──────────────────────────►│  └────────────────────────┘ │
│  Engine          │                           │                              │
│                  │                           │  ┌────────────────────────┐ │
│                  │◄──────────────────────────│  │ aml.screening.results  │ │
│                  │   ScreeningCompleted      │  └────────────────────────┘ │
└──────────────────┘                           │                              │
                                               │  ┌────────────────────────┐ │
┌──────────────────┐                           │  │ aml.matches.found      │ │
│ AML Screening    │                           │  └────────────────────────┘ │
│ Service          │                           │                              │
│                  │   ScreeningCompleted      │  ┌────────────────────────┐ │
│  Screening       │──────────────────────────►│  │ aml.alerts.lifecycle   │ │
│  Orchestrator    │                           │  └────────────────────────┘ │
│                  │   MatchFound              │                              │
│  Sanctions       │──────────────────────────►│  ┌────────────────────────┐ │
│  Screener        │                           │  │ aml.lists.updates      │ │
│                  │   AlertCreated            │  └────────────────────────┘ │
│  PEP Screener    │──────────────────────────►│                              │
│                  │                           │  ┌────────────────────────┐ │
│  Pattern         │◄──────────────────────────│  │ aml.reports.lifecycle  │ │
│  Analyzer        │   ListUpdated             │  └────────────────────────┘ │
└──────────────────┘                           │                              │
                                               └──────────────────────────────┘
┌──────────────────┐                                        │
│ Case Management  │◄───────────────────────────────────────┤ AlertCreated
│ System           │                                        │ MatchFound
│                  │                                        │
└──────────────────┘                                        │
                                                            │
┌──────────────────┐                                        │
│ Notification     │◄───────────────────────────────────────┤ AlertCreated
│ Service          │                                        │ AlertAssigned
│                  │                                        │ AlertEscalated
└──────────────────┘                                        │
                                                            │
┌──────────────────┐                                        │
│ Audit & Logging  │◄───────────────────────────────────────┤ All Events
│ Service          │                                        │
│                  │                                        │
└──────────────────┘                                        │
                                                            │
┌──────────────────┐                                        │
│ Compliance       │◄───────────────────────────────────────┤ SarGenerated
│ Dashboard        │                                        │ AlertResolved
│                  │                                        │ ListUpdated
└──────────────────┘                                        │
```

### 7.4 Kafka Topic Configuration

```yaml
# Topic: aml.screening.requests
aml.screening.requests:
  partitions: 24
  replication-factor: 3
  config:
    retention.ms: 604800000  # 7 days
    cleanup.policy: delete
    min.insync.replicas: 2
    compression.type: snappy

# Topic: aml.screening.results
aml.screening.results:
  partitions: 24
  replication-factor: 3
  config:
    retention.ms: 2592000000  # 30 days
    cleanup.policy: delete
    min.insync.replicas: 2

# Topic: aml.matches.found
aml.matches.found:
  partitions: 12
  replication-factor: 3
  config:
    retention.ms: 7776000000  # 90 days
    cleanup.policy: delete

# Topic: aml.alerts.lifecycle
aml.alerts.lifecycle:
  partitions: 12
  replication-factor: 3
  config:
    retention.ms: 31536000000  # 1 year
    cleanup.policy: compact,delete
    min.compaction.lag.ms: 3600000

# Topic: aml.lists.updates
aml.lists.updates:
  partitions: 6
  replication-factor: 3
  config:
    retention.ms: 2592000000  # 30 days
    cleanup.policy: compact

# Topic: aml.reports.lifecycle
aml.reports.lifecycle:
  partitions: 6
  replication-factor: 3
  config:
    retention.ms: 157680000000  # 5 years
    cleanup.policy: delete
```

### 7.5 Consumer Group Configuration

```yaml
# Consumer Groups
consumer-groups:
  aml-screening-processor:
    topics: [aml.screening.requests]
    instances: 8
    max-poll-records: 100
    session-timeout-ms: 30000
    auto-offset-reset: earliest
    
  aml-result-handler:
    topics: [aml.screening.results]
    instances: 4
    max-poll-records: 200
    
  alert-generator:
    topics: [aml.matches.found]
    instances: 4
    max-poll-records: 50
    
  case-management-listener:
    topics: [aml.alerts.lifecycle]
    instances: 2
    max-poll-records: 50
    
  list-update-handler:
    topics: [aml.lists.updates]
    instances: 2
    max-poll-records: 10
    
  audit-logger:
    topics: [aml.screening.results, aml.alerts.lifecycle, aml.reports.lifecycle]
    instances: 4
    max-poll-records: 500
```

---

## 8. Sequence Diagrams

### 8.1 Real-Time Screening Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    REAL-TIME AML SCREENING SEQUENCE                          │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────┐ ┌──────────┐ ┌───────────┐ ┌─────────┐ ┌────────┐ ┌─────────────┐
│Limit    │ │Screening │ │Sanctions  │ │PEP      │ │Pattern │ │Elasticsearch│
│Control  │ │Orchestrator│ │Screener   │ │Screener │ │Analyzer│ │             │
└────┬────┘ └─────┬─────┘ └─────┬─────┘ └────┬────┘ └───┬────┘ └──────┬──────┘
     │            │             │            │          │             │
     │  screenTransaction()     │            │          │             │
     │───────────────────────►│            │          │             │
     │            │             │            │          │             │
     │            │  extractSubjects()       │          │             │
     │            │─────────┐  │            │          │             │
     │            │         │  │            │          │             │
     │            │◄────────┘  │            │          │             │
     │            │             │            │          │             │
     │            │     PARALLEL SCREENING (fan-out)    │             │
     │            │─────────────────────────────────────│             │
     │            │             │            │          │             │
     │            │  screenAsync()           │          │             │
     │            │────────────►│            │          │             │
     │            │             │            │          │             │
     │            │             │  searchNames()        │             │
     │            │             │─────────────────────────────────────►│
     │            │             │            │          │             │
     │            │             │◄─────────────────────────────────────│
     │            │             │   matches  │          │             │
     │            │  screenAsync()           │          │             │
     │            │─────────────────────────►│          │             │
     │            │             │            │          │             │
     │            │             │            │searchPEP()│             │
     │            │             │            │──────────────────────────►│
     │            │             │            │          │             │
     │            │             │            │◄─────────────────────────│
     │            │             │            │ pepMatches│             │
     │            │  analyzeAsync()          │          │             │
     │            │─────────────────────────────────────►│             │
     │            │             │            │          │             │
     │            │             │            │          │ queryHistory()
     │            │             │            │          │──────┐      │
     │            │             │            │          │      │      │
     │            │             │            │          │◄─────┘      │
     │            │             │            │          │  patterns   │
     │            │◄────────────────────────────────────│             │
     │            │     ALL RESULTS COLLECTED           │             │
     │            │─────────────────────────────────────│             │
     │            │             │            │          │             │
     │            │  aggregateResults()      │          │             │
     │            │─────────┐  │            │          │             │
     │            │         │  │            │          │             │
     │            │◄────────┘  │            │          │             │
     │            │             │            │          │             │
     │            │  calculateRiskScore()    │          │             │
     │            │─────────┐  │            │          │             │
     │            │         │  │            │          │             │
     │            │◄────────┘  │            │          │             │
     │            │             │            │          │             │
     │            │  makeDecision()          │          │             │
     │            │─────────┐  │            │          │             │
     │            │         │  │            │          │             │
     │            │◄────────┘  │            │          │             │
     │            │  [PASS/REVIEW/BLOCK]     │          │             │
     │            │             │            │          │             │
     │  ScreeningResult        │            │          │             │
     │◄───────────────────────│            │          │             │
     │            │             │            │          │             │
```

### 8.2 Match Detection and Alert Creation Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MATCH DETECTION AND ALERT FLOW                            │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────┐ ┌──────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────┐
│Screening│ │Decision  │ │Alert    │ │Case     │ │Notific- │ │PostgreSQL│ │Kafka│
│Orchestrator│ │Engine    │ │Generator│ │Connector│ │ation    │ │         │ │     │
└────┬────┘ └────┬─────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └──┬──┘
     │           │            │           │           │           │         │
     │ evaluate(results)      │           │           │           │         │
     │──────────►│            │           │           │           │         │
     │           │            │           │           │           │         │
     │           │ decision=REVIEW        │           │           │         │
     │◄──────────│            │           │           │           │         │
     │           │            │           │           │           │         │
     │ createAlert()          │           │           │           │         │
     │───────────────────────►│           │           │           │         │
     │           │            │           │           │           │         │
     │           │            │ saveAlert()           │           │         │
     │           │            │───────────────────────────────────►│         │
     │           │            │           │           │           │         │
     │           │            │◄───────────────────────────────────│         │
     │           │            │ alertId   │           │           │         │
     │           │            │           │           │           │         │
     │           │            │ publishEvent(AlertCreated)        │         │
     │           │            │──────────────────────────────────────────────►│
     │           │            │           │           │           │         │
     │           │            │           │◄──────────────────────────────────│
     │           │            │           │ AlertCreated          │         │
     │           │            │           │           │           │         │
     │           │            │           │ createCase()          │         │
     │           │            │           │───────────┐│           │         │
     │           │            │           │           ││           │         │
     │           │            │           │◄──────────┘│           │         │
     │           │            │           │ caseId    │           │         │
     │           │            │           │           │           │         │
     │           │            │           │           │◄──────────────────────│
     │           │            │           │           │ AlertCreated         │
     │           │            │           │           │           │         │
     │           │            │           │           │ sendNotification()   │
     │           │            │           │           │───────────┐│         │
     │           │            │           │           │           ││         │
     │           │            │           │           │◄──────────┘│         │
     │           │            │           │           │  email/SMS │         │
     │           │            │           │           │           │         │
     │◄───────────────────────│           │           │           │         │
     │ AlertReference         │           │           │           │         │
     │           │            │           │           │           │         │
```

### 8.3 Alert Resolution Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       ALERT RESOLUTION SEQUENCE                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────┐
│Analyst  │ │Alert    │ │Alert    │ │Payment  │ │Audit    │ │PostgreSQL│ │Kafka│
│UI       │ │API      │ │Service  │ │Orchestr │ │Logger   │ │         │ │     │
└────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └──┬──┘
     │           │           │           │           │           │         │
     │ GET /alerts/{id}      │           │           │           │         │
     │──────────►│           │           │           │           │         │
     │           │ getAlertDetails()     │           │           │         │
     │           │──────────►│           │           │           │         │
     │           │           │ query()   │           │           │         │
     │           │           │───────────────────────────────────►│         │
     │           │           │◄───────────────────────────────────│         │
     │           │◄──────────│           │           │           │         │
     │◄──────────│ alertDetails          │           │           │         │
     │           │           │           │           │           │         │
     │    [Analyst reviews match evidence]           │           │         │
     │           │           │           │           │           │         │
     │ POST /alerts/{id}/resolve         │           │           │         │
     │ {resolution: FALSE_POSITIVE}      │           │           │         │
     │──────────►│           │           │           │           │         │
     │           │ resolveAlert()        │           │           │         │
     │           │──────────►│           │           │           │         │
     │           │           │           │           │           │         │
     │           │           │ validateResolution()  │           │         │
     │           │           │───────────┐│           │           │         │
     │           │           │           ││           │           │         │
     │           │           │◄──────────┘│           │           │         │
     │           │           │           │           │           │         │
     │           │           │ updateAlert()         │           │         │
     │           │           │───────────────────────────────────►│         │
     │           │           │◄───────────────────────────────────│         │
     │           │           │           │           │           │         │
     │           │           │ publishEvent(AlertResolved)       │         │
     │           │           │──────────────────────────────────────────────►│
     │           │           │           │           │           │         │
     │           │           │           │◄──────────────────────────────────│
     │           │           │           │ AlertResolved         │         │
     │           │           │           │           │           │         │
     │           │           │           │ releaseTransaction()  │         │
     │           │           │           │───────────┐│           │         │
     │           │           │           │           ││           │         │
     │           │           │           │◄──────────┘│           │         │
     │           │           │           │ released  │           │         │
     │           │           │           │           │           │         │
     │           │           │           │           │◄──────────────────────│
     │           │           │           │           │ AlertResolved        │
     │           │           │           │           │           │         │
     │           │           │           │           │ logResolution()      │
     │           │           │           │           │───────────┐│         │
     │           │           │           │           │           ││         │
     │           │           │           │           │◄──────────┘│         │
     │           │           │           │           │  logged   │         │
     │           │◄──────────│           │           │           │         │
     │◄──────────│ resolution confirmed  │           │           │         │
     │           │           │           │           │           │         │
```

### 8.4 List Update and Re-screening Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     LIST UPDATE AND RE-SCREENING FLOW                        │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
│Scheduler│ │List     │ │OFAC     │ │Elastic- │ │Redis    │ │Screening│
│         │ │Manager  │ │Source   │ │search   │ │Cache    │ │Service  │
└────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘
     │           │           │           │           │           │
     │ triggerListUpdate()   │           │           │           │
     │──────────►│           │           │           │           │
     │           │           │           │           │           │
     │           │ fetchLatest()         │           │           │
     │           │──────────►│           │           │           │
     │           │           │           │           │           │
     │           │◄──────────│           │           │           │
     │           │ listData  │           │           │           │
     │           │           │           │           │           │
     │           │ compareVersions()     │           │           │
     │           │───────────┐│           │           │           │
     │           │           ││           │           │           │
     │           │◄──────────┘│           │           │           │
     │           │ deltaChanges          │           │           │
     │           │           │           │           │           │
     │           │ bulkIndex(newEntries) │           │           │
     │           │───────────────────────►│           │           │
     │           │           │           │           │           │
     │           │◄───────────────────────│           │           │
     │           │ indexed   │           │           │           │
     │           │           │           │           │           │
     │           │ invalidateCache()     │           │           │
     │           │───────────────────────────────────►│           │
     │           │           │           │           │           │
     │           │◄───────────────────────────────────│           │
     │           │ invalidated           │           │           │
     │           │           │           │           │           │
     │           │ updateBloomFilter()   │           │           │
     │           │───────────────────────────────────►│           │
     │           │           │           │           │           │
     │           │◄───────────────────────────────────│           │
     │           │ updated   │           │           │           │
     │           │           │           │           │           │
     │           │ publishEvent(ListUpdated)         │           │
     │           │────────────────────────────────────────────────►│
     │           │           │           │           │           │
     │           │           │           │           │           │
     │           │           │           │           │ triggerRescreen()
     │           │           │           │           │ (for affected
     │           │           │           │           │  customers)
     │           │           │           │           │───────────┐
     │           │           │           │           │           │
     │           │           │           │           │◄──────────┘
     │           │           │           │           │ scheduled │
     │◄──────────│           │           │           │           │
     │ completed │           │           │           │           │
     │           │           │           │           │           │
```

---

## 9. Integration Points

### 9.1 Integration Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     AML MODULE INTEGRATION ARCHITECTURE                      │
└─────────────────────────────────────────────────────────────────────────────┘

                    ┌─────────────────────────────────────┐
                    │        EXTERNAL DATA PROVIDERS       │
                    │                                     │
                    │  ┌─────────┐ ┌─────────┐ ┌────────┐│
                    │  │  OFAC   │ │   EU    │ │World-  ││
                    │  │  API    │ │  API    │ │Check   ││
                    │  └────┬────┘ └────┬────┘ └───┬────┘│
                    │       │          │          │     │
                    └───────┼──────────┼──────────┼─────┘
                            │          │          │
                            ▼          ▼          ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                          AML SCREENING MODULE                                  │
│                                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │   List       │  │  Screening   │  │    Alert     │  │   Regulatory     │ │
│  │   Ingestion  │  │  Engine      │  │   Management │  │   Reporting      │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────────┘ │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
         │                  │                  │                  │
         ▼                  ▼                  ▼                  ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐
│ Payment     │    │ Customer    │    │ Case        │    │ Regulatory      │
│ Orchestration│    │ Domain      │    │ Management  │    │ Portal          │
│ Domain      │    │             │    │ System      │    │ (FinCEN BSA)    │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────────┘
```

### 9.2 Internal Integration Points

#### 9.2.1 Payment Orchestration Domain Integration

| Integration | Type | Protocol | Description |
|-------------|------|----------|-------------|
| Screening Trigger | Sync/Async | gRPC/Kafka | Limit Control Engine triggers AML screening |
| Screening Response | Sync/Async | gRPC/Kafka | Return screening decision to orchestration |
| Transaction Hold | Event | Kafka | Hold transaction pending alert resolution |
| Transaction Release | Event | Kafka | Release held transaction after resolution |

```protobuf
// gRPC Service Definition
service AmlScreeningService {
    // Synchronous screening for real-time payments
    rpc ScreenTransaction(ScreeningRequest) returns (ScreeningResponse);
    
    // Async screening with callback
    rpc ScreenTransactionAsync(ScreeningRequest) returns (ScreeningRequestAck);
    
    // Get screening status
    rpc GetScreeningStatus(GetStatusRequest) returns (ScreeningStatus);
    
    // Cancel pending screening
    rpc CancelScreening(CancelRequest) returns (CancelResponse);
}

message ScreeningRequest {
    string transaction_id = 1;
    string request_type = 2;
    string priority = 3;
    repeated Subject subjects = 4;
    TransactionDetails transaction_details = 5;
    ScreeningConfig config = 6;
}

message ScreeningResponse {
    string request_id = 1;
    string decision = 2;  // PASS, REVIEW, BLOCK
    int32 risk_score = 3;
    repeated Match matches = 4;
    string alert_reference = 5;
    int64 screening_duration_ms = 6;
}
```

#### 9.2.2 Customer Domain Integration

| Integration | Type | Protocol | Description |
|-------------|------|----------|-------------|
| Customer Risk Profile | Sync | REST | Retrieve customer risk classification |
| Customer KYC Status | Sync | REST | Get KYC/EDD completion status |
| Customer History | Sync | REST | Retrieve customer transaction history |
| Risk Profile Update | Event | Kafka | Update customer risk after screening |

```yaml
# Customer Domain API Integration
GET /api/v1/customers/{customerId}/risk-profile
Response:
{
  "customerId": "cust-12345",
  "riskClassification": "MEDIUM",
  "riskScore": 45,
  "kycStatus": "VERIFIED",
  "eddRequired": false,
  "lastReviewDate": "2025-12-15",
  "nextReviewDate": "2026-06-15",
  "riskFactors": [
    {"factor": "CUSTOMER_TYPE", "value": "INDIVIDUAL", "score": 20},
    {"factor": "TENURE", "value": "5_YEARS", "score": 10},
    {"factor": "PRODUCT_MIX", "value": "STANDARD", "score": 15}
  ]
}
```

#### 9.2.3 Case Management System Integration

| Integration | Type | Protocol | Description |
|-------------|------|----------|-------------|
| Case Creation | Sync | REST | Create investigation case from alert |
| Case Update | Sync | REST | Update case with new evidence |
| Case Status | Sync | REST | Get case investigation status |
| Case Events | Event | Kafka | Receive case status change events |

```yaml
# Case Management API Integration
POST /api/v1/cases
Request:
{
  "caseType": "AML_INVESTIGATION",
  "priority": "HIGH",
  "source": "AML_SCREENING",
  "sourceReference": "AML-2026-02-20-00145",
  "subject": {
    "type": "CUSTOMER",
    "id": "cust-12345",
    "name": "John Smith"
  },
  "relatedEntities": [
    {
      "type": "TRANSACTION",
      "id": "txn-67890"
    },
    {
      "type": "ALERT",
      "id": "alert-12345"
    }
  ],
  "summary": "PEP match detected on wire transfer beneficiary",
  "initialEvidence": {
    "matchType": "PEP_HIT",
    "matchConfidence": 87,
    "riskScore": 72,
    "matchedEntity": "ACME Holdings GmbH"
  }
}

Response:
{
  "caseId": "case-abc123",
  "caseReference": "INV-2026-00234",
  "status": "OPEN",
  "assignedTeam": "AML_INVESTIGATIONS",
  "slaDeadline": "2026-02-27T18:00:00Z"
}
```

### 9.3 External Integration Points

#### 9.3.1 Sanctions List Providers

| Provider | List Type | Protocol | Update Frequency | Integration Method |
|----------|-----------|----------|------------------|-------------------|
| OFAC | SDN, Consolidated | REST API | Daily | Pull with delta updates |
| EU | Consolidated | REST API | Daily | Pull with full refresh |
| UN | Security Council | XML/FTP | Daily | File download |
| UK HMT | Consolidated | REST API | Daily | Pull with delta |
| World-Check | PEP, Sanctions | REST API | Continuous | Streaming/Webhook |
| Dow Jones | Risk & Compliance | REST API | Continuous | API Query |
| LexisNexis | Adverse Media | REST API | On-demand | Real-time API |

```yaml
# OFAC SDN API Integration
Base URL: https://api.ofac.treasury.gov/v1

# Get SDN List Updates
GET /sdn/delta?since=2026-02-19
Headers:
  Authorization: Bearer {api_key}
  Accept: application/json

Response:
{
  "version": "2026-02-20-001",
  "publishedAt": "2026-02-20T06:00:00Z",
  "entriesAdded": 15,
  "entriesRemoved": 3,
  "entriesModified": 22,
  "entries": [
    {
      "entryId": "12345",
      "action": "ADD",
      "entryType": "INDIVIDUAL",
      "firstName": "VLADIMIR",
      "lastName": "IVANOV",
      "programs": ["RUSSIA-EO14024"],
      "dateOfBirth": "1965-08-20",
      "nationalities": ["RU"],
      "identifiers": [
        {"type": "PASSPORT", "value": "AB1234567"}
      ]
    }
  ]
}
```

#### 9.3.2 PEP Data Providers

```yaml
# World-Check One API Integration
Base URL: https://api.refinitiv.com/worldcheck/v1

# Screen Subject
POST /screen/subjects
Headers:
  Authorization: Bearer {oauth_token}
  Content-Type: application/json

Request:
{
  "subject": {
    "fullName": "Vladimir Ivanov",
    "dateOfBirth": "1965-08-20",
    "nationalities": ["RU"],
    "entityType": "INDIVIDUAL"
  },
  "searchConfiguration": {
    "pepCategories": ["DOMESTIC", "FOREIGN", "INTERNATIONAL"],
    "includeRca": true,
    "includeSoe": true,
    "threshold": 80
  }
}

Response:
{
  "screeningId": "scr-12345",
  "matchesFound": 1,
  "matches": [
    {
      "matchId": "wc-67890",
      "matchScore": 95,
      "profile": {
        "profileId": "prf-11111",
        "name": "IVANOV, Vladimir Petrovich",
        "category": "PEP",
        "subCategory": "DOMESTIC_PEP",
        "position": "Deputy Minister of Finance",
        "country": "RU",
        "status": "ACTIVE",
        "since": "2020-01-15"
      },
      "rca": [
        {
          "name": "IVANOVA, Maria",
          "relationship": "SPOUSE",
          "profileId": "prf-22222"
        }
      ]
    }
  ]
}
```

#### 9.3.3 Regulatory Reporting Integration

| Regulator | Report Type | Protocol | Submission Method |
|-----------|-------------|----------|-------------------|
| FinCEN (US) | SAR, CTR | HTTPS | BSA E-Filing System |
| FCA (UK) | SAR | HTTPS | SAR Online Portal |
| BaFin (DE) | STR | HTTPS | goAML |
| MAS (SG) | STR | HTTPS | STRO Portal |

```yaml
# FinCEN BSA E-Filing Integration
Base URL: https://bsaefiling.fincen.treas.gov/api/v1

# Submit SAR
POST /filings/sar
Headers:
  Authorization: Bearer {api_key}
  Content-Type: application/xml

Request: [SAR XML Document per FinCEN Schema]

Response:
{
  "trackingId": "BSA-2026-00123456",
  "status": "SUBMITTED",
  "acknowledgementExpected": "2026-02-21T12:00:00Z"
}

# Check Filing Status
GET /filings/{trackingId}/status
Response:
{
  "trackingId": "BSA-2026-00123456",
  "status": "ACKNOWLEDGED",
  "acknowledgedAt": "2026-02-20T18:45:00Z",
  "filingReference": "FinCEN-2026-SAR-789012"
}
```

### 9.4 Integration Resilience Patterns

```yaml
# Circuit Breaker Configuration
circuit-breakers:
  ofac-api:
    failure-rate-threshold: 50
    slow-call-rate-threshold: 100
    slow-call-duration: 5s
    wait-duration-in-open-state: 30s
    permitted-calls-in-half-open: 5
    sliding-window-size: 10
    fallback: use-cached-list
    
  world-check-api:
    failure-rate-threshold: 50
    slow-call-rate-threshold: 100
    slow-call-duration: 3s
    wait-duration-in-open-state: 60s
    fallback: queue-for-retry
    
  case-management:
    failure-rate-threshold: 70
    wait-duration-in-open-state: 30s
    fallback: store-and-forward

# Retry Configuration
retry-policies:
  external-list-fetch:
    max-attempts: 3
    delay: 1s
    multiplier: 2
    max-delay: 10s
    
  screening-request:
    max-attempts: 2
    delay: 100ms
    
  regulatory-submission:
    max-attempts: 5
    delay: 5s
    multiplier: 2
    max-delay: 5m

# Timeout Configuration
timeouts:
  sanctions-screening: 100ms
  pep-screening: 80ms
  pattern-analysis: 50ms
  total-screening: 200ms
  list-update: 5m
  regulatory-submission: 30s
```

---

## 10. Security Design

### 10.1 Security Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AML MODULE SECURITY ARCHITECTURE                      │
└─────────────────────────────────────────────────────────────────────────────┘

                           ┌─────────────────────┐
                           │   API Gateway       │
                           │   (Authentication)  │
                           └──────────┬──────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                 │
                    ▼                 ▼                 ▼
           ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
           │   OAuth2/   │   │    mTLS     │   │   API Key   │
           │   OIDC      │   │   (Service) │   │  (External) │
           └─────────────┘   └─────────────┘   └─────────────┘
                    │                 │                 │
                    └─────────────────┼─────────────────┘
                                      │
                           ┌──────────▼──────────┐
                           │  Authorization      │
                           │  (RBAC + ABAC)      │
                           └──────────┬──────────┘
                                      │
┌─────────────────────────────────────┼─────────────────────────────────────────┐
│                      AML SCREENING SERVICE                                    │
│                                     │                                         │
│  ┌─────────────┐   ┌─────────────┐  │  ┌─────────────┐   ┌─────────────────┐ │
│  │ Input       │   │ Data        │  │  │ Audit       │   │ Encryption      │ │
│  │ Validation  │   │ Masking     │  │  │ Logging     │   │ (At-Rest/Transit)│ │
│  └─────────────┘   └─────────────┘  │  └─────────────┘   └─────────────────┘ │
│                                     │                                         │
│  ┌─────────────┐   ┌─────────────┐  │  ┌─────────────┐   ┌─────────────────┐ │
│  │ Secret      │   │ Key         │  │  │ Data        │   │ Threat          │ │
│  │ Management  │   │ Management  │  │  │ Classification│   │ Detection       │ │
│  └─────────────┘   └─────────────┘  │  └─────────────┘   └─────────────────┘ │
│                                     │                                         │
└─────────────────────────────────────┼─────────────────────────────────────────┘
                                      │
                           ┌──────────▼──────────┐
                           │  Data Stores        │
                           │  (Encrypted)        │
                           └─────────────────────┘
```

### 10.2 Authentication & Authorization

```yaml
# OAuth2/OIDC Configuration
oauth2:
  issuer: https://auth.company.com
  token-endpoint: /oauth2/token
  jwks-uri: /.well-known/jwks.json
  scopes:
    - aml:screen           # Perform screening
    - aml:screen:read      # Read screening results
    - aml:alerts:read      # Read alerts
    - aml:alerts:write     # Manage alerts
    - aml:lists:read       # Read screening lists
    - aml:lists:write      # Manage watchlists
    - aml:reports:write    # Generate regulatory reports
    - aml:admin            # Administrative functions

# RBAC Role Definitions
roles:
  aml-screener:
    description: "Automated screening service"
    scopes: [aml:screen, aml:screen:read]
    
  aml-analyst:
    description: "AML investigation analyst"
    scopes: [aml:screen:read, aml:alerts:read, aml:alerts:write]
    
  aml-senior-analyst:
    description: "Senior AML analyst"
    scopes: [aml:screen:read, aml:alerts:read, aml:alerts:write, aml:reports:write]
    
  aml-compliance-officer:
    description: "Compliance officer"
    scopes: [aml:screen:read, aml:alerts:read, aml:alerts:write, aml:lists:read, aml:lists:write, aml:reports:write]
    
  aml-administrator:
    description: "AML system administrator"
    scopes: [aml:admin, aml:lists:write]

# ABAC Policy Examples
policies:
  - name: "alert-assignment-by-amount"
    description: "High-value alerts require senior analyst"
    condition: "alert.amount > 100000 AND alert.priority == 'HIGH'"
    effect: "require_role('aml-senior-analyst')"
    
  - name: "sar-approval-dual-control"
    description: "SAR filing requires dual approval"
    condition: "action == 'SAR_SUBMIT'"
    effect: "require_approval(['aml-senior-analyst', 'aml-compliance-officer'])"
```

### 10.3 Data Protection

```yaml
# Data Classification
data-classification:
  HIGHLY_SENSITIVE:
    - customer_identifiers (SSN, passport)
    - sanctions_match_details
    - sar_reports
    encryption: AES-256-GCM
    masking: full
    retention: 7 years
    access: aml-compliance-officer only
    
  SENSITIVE:
    - customer_names
    - transaction_amounts
    - screening_results
    encryption: AES-256-GCM
    masking: partial (last 4 chars visible)
    retention: 5 years
    access: aml-analyst and above
    
  INTERNAL:
    - alert_metadata
    - screening_statistics
    - list_versions
    encryption: AES-256-GCM
    masking: none
    retention: 3 years
    access: all aml roles

# Encryption Configuration
encryption:
  at-rest:
    algorithm: AES-256-GCM
    key-rotation: 90 days
    key-source: AWS KMS / HashiCorp Vault
    
  in-transit:
    protocol: TLS 1.3
    cipher-suites:
      - TLS_AES_256_GCM_SHA384
      - TLS_CHACHA20_POLY1305_SHA256
    certificate-rotation: 365 days
    mtls-required: true (service-to-service)

# Data Masking Rules
masking-rules:
  ssn:
    pattern: '\d{3}-\d{2}-\d{4}'
    mask: 'XXX-XX-{last4}'
    
  passport:
    pattern: '[A-Z]{2}\d{7}'
    mask: 'XX{last5}'
    
  name:
    mask: '{first_initial}****{last_initial}'
    
  account-number:
    mask: '****{last4}'
```

### 10.4 Audit & Compliance Logging

```yaml
# Audit Log Configuration
audit-logging:
  enabled: true
  retention: 7 years
  immutable: true
  encryption: AES-256-GCM
  
  events:
    # Screening Events
    - event: SCREENING_REQUESTED
      log: [request_id, transaction_id, subject_count, requester]
      
    - event: SCREENING_COMPLETED
      log: [request_id, decision, risk_score, match_count, duration_ms]
      
    - event: MATCH_DETECTED
      log: [match_id, match_type, confidence, list_code, subject_type]
      
    # Alert Events
    - event: ALERT_CREATED
      log: [alert_id, alert_type, priority, risk_score]
      
    - event: ALERT_ASSIGNED
      log: [alert_id, assigned_to, assigned_by]
      
    - event: ALERT_RESOLVED
      log: [alert_id, resolution, resolved_by, sar_filed]
      
    # List Events
    - event: LIST_UPDATED
      log: [list_code, version, entries_changed]
      
    - event: WATCHLIST_ENTRY_ADDED
      log: [entry_id, list_code, added_by, reason]
      
    # Access Events
    - event: ALERT_VIEWED
      log: [alert_id, viewer, timestamp]
      
    - event: CUSTOMER_DATA_ACCESSED
      log: [customer_id, data_type, accessor, purpose]

# Compliance Requirements Mapping
compliance-mapping:
  GDPR:
    - data-minimization
    - right-to-erasure (with legal hold exception)
    - consent-tracking
    - data-portability
    
  PCI-DSS:
    - encryption-at-rest
    - encryption-in-transit
    - access-logging
    - key-management
    
  SOX:
    - audit-trail-immutability
    - dual-control-for-changes
    - access-reviews
```

---

## 11. Non-Functional Requirements

### 11.1 Performance Requirements

| Requirement | Metric | Target | Measurement |
|-------------|--------|--------|-------------|
| NFR-P-001 | Real-time screening latency (p50) | < 100ms | Response time for single transaction |
| NFR-P-002 | Real-time screening latency (p95) | < 150ms | Response time for single transaction |
| NFR-P-003 | Real-time screening latency (p99) | < 200ms | Response time for single transaction |
| NFR-P-004 | Batch screening throughput | > 10,000/min | Subjects processed per minute |
| NFR-P-005 | Name matching latency | < 50ms | Single name match operation |
| NFR-P-006 | Risk score calculation | < 30ms | Score computation time |
| NFR-P-007 | Alert creation latency | < 100ms | Time from match to alert |
| NFR-P-008 | List update propagation | < 5 min | Time to propagate updates |
| NFR-P-009 | Concurrent screening capacity | > 5,000 TPS | Simultaneous screening requests |
| NFR-P-010 | Database query latency (p99) | < 20ms | Query response time |

### 11.2 Availability Requirements

| Requirement | Metric | Target | Description |
|-------------|--------|--------|-------------|
| NFR-A-001 | Service availability | 99.99% | Monthly uptime |
| NFR-A-002 | Planned maintenance window | < 4 hours/month | Scheduled downtime |
| NFR-A-003 | Recovery Time Objective (RTO) | < 15 min | Time to restore service |
| NFR-A-004 | Recovery Point Objective (RPO) | < 1 min | Maximum data loss |
| NFR-A-005 | Failover time | < 30 sec | Automatic failover duration |
| NFR-A-006 | Database availability | 99.99% | Database cluster uptime |
| NFR-A-007 | List data availability | 99.999% | Screening list availability |

### 11.3 Scalability Requirements

| Requirement | Metric | Target | Description |
|-------------|--------|--------|-------------|
| NFR-S-001 | Horizontal scaling | Auto-scale | Scale pods based on load |
| NFR-S-002 | Peak load handling | 3x normal | Handle 3x baseline throughput |
| NFR-S-003 | List entry capacity | > 10M entries | Total list entries supported |
| NFR-S-004 | Concurrent users | > 1,000 | Simultaneous analyst users |
| NFR-S-005 | Alert queue capacity | > 1M | Pending alerts storage |
| NFR-S-006 | Historical data retention | 7 years | Screening history storage |

### 11.4 Detection Accuracy Requirements

| Requirement | Metric | Target | Description |
|-------------|--------|--------|-------------|
| NFR-D-001 | True positive rate | > 99.9% | Sanctioned entity detection |
| NFR-D-002 | False positive rate | < 3% | Incorrect match rate |
| NFR-D-003 | Name matching accuracy | > 95% | Correct name match rate |
| NFR-D-004 | Pattern detection accuracy | > 90% | Suspicious pattern detection |
| NFR-D-005 | PEP identification rate | > 98% | PEP detection accuracy |

### 11.5 Compliance Requirements

| Requirement | Regulation | Description |
|-------------|------------|-------------|
| NFR-C-001 | FATF | Implement all 40 FATF recommendations |
| NFR-C-002 | BSA/AML | Comply with Bank Secrecy Act requirements |
| NFR-C-003 | OFAC | Screen against all OFAC lists |
| NFR-C-004 | 6AMLD | Meet EU 6th AML Directive requirements |
| NFR-C-005 | GDPR | Data protection and privacy compliance |
| NFR-C-006 | PCI-DSS | Payment card data security |
| NFR-C-007 | SOX | Financial reporting controls |

---

## 12. Deployment Architecture

### 12.1 Kubernetes Deployment Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      AML MODULE DEPLOYMENT ARCHITECTURE                      │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                            KUBERNETES CLUSTER                                │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                      NAMESPACE: aml-screening                           ││
│  │                                                                         ││
│  │  ┌───────────────────────────────────────────────────────────────────┐ ││
│  │  │                    DEPLOYMENT: aml-screening-api                   │ ││
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐ │ ││
│  │  │  │ Pod 1   │  │ Pod 2   │  │ Pod 3   │  │ Pod N   │  │ ...     │ │ ││
│  │  │  │ (8003)  │  │ (8003)  │  │ (8003)  │  │ (8003)  │  │         │ │ ││
│  │  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘  └─────────┘ │ ││
│  │  └───────────────────────────────────────────────────────────────────┘ ││
│  │                                                                         ││
│  │  ┌───────────────────────────────────────────────────────────────────┐ ││
│  │  │                 DEPLOYMENT: aml-screening-worker                   │ ││
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐              │ ││
│  │  │  │ Worker  │  │ Worker  │  │ Worker  │  │ Worker  │              │ ││
│  │  │  │    1    │  │    2    │  │    3    │  │   N     │              │ ││
│  │  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘              │ ││
│  │  └───────────────────────────────────────────────────────────────────┘ ││
│  │                                                                         ││
│  │  ┌──────────────────────┐  ┌──────────────────────┐                   ││
│  │  │ CronJob: list-sync   │  │ CronJob: batch-screen│                   ││
│  │  └──────────────────────┘  └──────────────────────┘                   ││
│  │                                                                         ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                    SERVICES & NETWORKING                                ││
│  │                                                                         ││
│  │  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐               ││
│  │  │ ClusterIP    │   │ Ingress      │   │ Network      │               ││
│  │  │ Service      │   │ Controller   │   │ Policies     │               ││
│  │  └──────────────┘   └──────────────┘   └──────────────┘               ││
│  │                                                                         ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 12.2 Kubernetes Manifests

```yaml
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: aml-screening
  labels:
    app: aml-screening
    compliance: high-security
---
# API Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aml-screening-api
  namespace: aml-screening
  labels:
    app: aml-screening
    component: api
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: aml-screening
      component: api
  template:
    metadata:
      labels:
        app: aml-screening
        component: api
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8003"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      serviceAccountName: aml-screening-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
        - name: aml-screening-api
          image: payment-hub/aml-screening:v2.7.0
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8003
              protocol: TCP
            - name: grpc
              containerPort: 9003
              protocol: TCP
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "production,k8s"
            - name: JAVA_OPTS
              value: "-Xms2g -Xmx4g -XX:+UseG1GC -XX:MaxGCPauseMillis=100"
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: aml-db-credentials
                  key: host
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: aml-db-credentials
                  key: password
          resources:
            requests:
              cpu: "1000m"
              memory: "4Gi"
            limits:
              cpu: "2000m"
              memory: "8Gi"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8003
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8003
            initialDelaySeconds: 20
            periodSeconds: 5
            failureThreshold: 3
          volumeMounts:
            - name: config-volume
              mountPath: /app/config
              readOnly: true
            - name: secrets-volume
              mountPath: /app/secrets
              readOnly: true
      volumes:
        - name: config-volume
          configMap:
            name: aml-screening-config
        - name: secrets-volume
          secret:
            secretName: aml-screening-secrets
---
# Worker Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aml-screening-worker
  namespace: aml-screening
  labels:
    app: aml-screening
    component: worker
spec:
  replicas: 5
  selector:
    matchLabels:
      app: aml-screening
      component: worker
  template:
    metadata:
      labels:
        app: aml-screening
        component: worker
    spec:
      containers:
        - name: aml-screening-worker
          image: payment-hub/aml-screening-worker:v2.7.0
          env:
            - name: KAFKA_BOOTSTRAP_SERVERS
              value: "kafka-cluster:9092"
            - name: CONSUMER_GROUP_ID
              value: "aml-screening-workers"
          resources:
            requests:
              cpu: "500m"
              memory: "2Gi"
            limits:
              cpu: "1000m"
              memory: "4Gi"
---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: aml-screening-api-hpa
  namespace: aml-screening
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: aml-screening-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 75
    - type: Pods
      pods:
        metric:
          name: screening_requests_per_second
        target:
          type: AverageValue
          averageValue: "500"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Pods
          value: 4
          periodSeconds: 30
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
---
# List Sync CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: aml-list-sync
  namespace: aml-screening
spec:
  schedule: "0 */4 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: list-sync
              image: payment-hub/aml-list-sync:v2.7.0
              resources:
                requests:
                  cpu: "500m"
                  memory: "1Gi"
                limits:
                  cpu: "1000m"
                  memory: "2Gi"
---
# Service
apiVersion: v1
kind: Service
metadata:
  name: aml-screening-service
  namespace: aml-screening
spec:
  type: ClusterIP
  selector:
    app: aml-screening
    component: api
  ports:
    - name: http
      port: 8003
      targetPort: 8003
      protocol: TCP
    - name: grpc
      port: 9003
      targetPort: 9003
      protocol: TCP
---
# Network Policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: aml-screening-network-policy
  namespace: aml-screening
spec:
  podSelector:
    matchLabels:
      app: aml-screening
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: payment-orchestration
        - namespaceSelector:
            matchLabels:
              name: api-gateway
      ports:
        - protocol: TCP
          port: 8003
        - protocol: TCP
          port: 9003
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: database
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - namespaceSelector:
            matchLabels:
              name: kafka
      ports:
        - protocol: TCP
          port: 9092
```

### 12.3 Environment Configuration

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: aml-screening-config
  namespace: aml-screening
data:
  application.yml: |
    server:
      port: 8003
      
    aml:
      screening:
        default-timeout-ms: 200
        parallel-list-screening: true
        cache:
          enabled: true
          ttl-seconds: 300
          
      name-matching:
        algorithm: HYBRID
        threshold: 85
        fuzzy-enabled: true
        
      lists:
        auto-sync: true
        sync-interval: PT4H
        
      alerts:
        auto-creation: true
        default-priority: MEDIUM
        sla:
          critical: PT1H
          high: PT4H
          medium: PT24H
          low: PT72H
          
    spring:
      datasource:
        url: jdbc:postgresql://${DB_HOST}:5432/aml_screening
        hikari:
          maximum-pool-size: 50
          minimum-idle: 10
          
      kafka:
        bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS}
        consumer:
          group-id: aml-screening-service
          auto-offset-reset: earliest
          
      elasticsearch:
        uris: ${ES_HOSTS}
        
      redis:
        host: ${REDIS_HOST}
        port: 6379
        
    management:
      endpoints:
        web:
          exposure:
            include: health,metrics,prometheus
      metrics:
        tags:
          application: aml-screening
```

---

## 13. Monitoring & Observability

### 13.1 Observability Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     AML MODULE OBSERVABILITY STACK                           │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   APPLICATION    │────▶│    COLLECTORS    │────▶│    BACKENDS      │
└──────────────────┘     └──────────────────┘     └──────────────────┘
         │                        │                        │
         │                        │                        ▼
┌────────┴────────┐      ┌────────┴────────┐     ┌─────────────────┐
│                 │      │                 │     │                 │
│  ┌───────────┐ │      │  ┌───────────┐ │     │  ┌───────────┐  │
│  │  Metrics  │ │──────│▶│ Prometheus │ │─────│▶│  Grafana   │  │
│  │  (Micrometer)│     │  │  Agent     │ │     │  │  Dashboard │  │
│  └───────────┘ │      │  └───────────┘ │     │  └───────────┘  │
│                 │      │                 │     │                 │
│  ┌───────────┐ │      │  ┌───────────┐ │     │  ┌───────────┐  │
│  │  Traces   │ │──────│▶│   Jaeger   │ │─────│▶│   Jaeger   │  │
│  │(OpenTelemetry)│    │  │  Collector │ │     │  │    UI      │  │
│  └───────────┘ │      │  └───────────┘ │     │  └───────────┘  │
│                 │      │                 │     │                 │
│  ┌───────────┐ │      │  ┌───────────┐ │     │  ┌───────────┐  │
│  │   Logs    │ │──────│▶│  Fluentd/  │ │─────│▶│Elasticsearch│  │
│  │ (Logback) │ │      │  │  Vector   │ │     │  │ + Kibana   │  │
│  └───────────┘ │      │  └───────────┘ │     │  └───────────┘  │
│                 │      │                 │     │                 │
└─────────────────┘      └─────────────────┘     └─────────────────┘
                                                         │
                                                         ▼
                                               ┌─────────────────┐
                                               │   ALERTING      │
                                               │  PagerDuty/     │
                                               │  OpsGenie/Slack │
                                               └─────────────────┘
```

### 13.2 Key Metrics

```yaml
# Custom Metrics Definition
metrics:
  # Screening Metrics
  - name: aml_screening_requests_total
    type: counter
    description: "Total number of screening requests"
    labels: [request_type, subject_type, priority]
    
  - name: aml_screening_duration_seconds
    type: histogram
    description: "Screening request duration in seconds"
    labels: [request_type, decision]
    buckets: [0.01, 0.025, 0.05, 0.075, 0.1, 0.15, 0.2, 0.3, 0.5, 1.0]
    
  - name: aml_screening_decisions_total
    type: counter
    description: "Screening decisions by type"
    labels: [decision, risk_category]
    
  # Match Metrics
  - name: aml_matches_total
    type: counter
    description: "Total matches detected"
    labels: [match_type, list_code, match_status]
    
  - name: aml_match_confidence
    type: histogram
    description: "Match confidence score distribution"
    labels: [match_type]
    buckets: [50, 60, 70, 80, 85, 90, 95, 100]
    
  - name: aml_false_positive_rate
    type: gauge
    description: "Current false positive rate"
    labels: [list_code]
    
  # Alert Metrics
  - name: aml_alerts_created_total
    type: counter
    description: "Total alerts created"
    labels: [alert_type, priority]
    
  - name: aml_alerts_open
    type: gauge
    description: "Current open alerts"
    labels: [alert_type, priority, assigned_team]
    
  - name: aml_alert_resolution_time_seconds
    type: histogram
    description: "Alert resolution time"
    labels: [alert_type, resolution]
    buckets: [300, 900, 1800, 3600, 14400, 28800, 86400, 259200]
    
  - name: aml_alert_sla_breach_total
    type: counter
    description: "SLA breaches by alert type"
    labels: [alert_type, priority]
    
  # List Metrics
  - name: aml_list_entries_total
    type: gauge
    description: "Total entries in screening lists"
    labels: [list_code, list_type, status]
    
  - name: aml_list_sync_duration_seconds
    type: histogram
    description: "List synchronization duration"
    labels: [list_code, sync_type]
    
  - name: aml_list_sync_errors_total
    type: counter
    description: "List sync errors"
    labels: [list_code, error_type]
    
  # Regulatory Reporting Metrics
  - name: aml_sar_filed_total
    type: counter
    description: "SARs filed by type"
    labels: [sar_type, regulator]
    
  - name: aml_sar_pending
    type: gauge
    description: "Pending SAR filings"
    labels: [sar_type]
```

### 13.3 Grafana Dashboard Configuration

```json
{
  "dashboard": {
    "title": "AML Screening Module Dashboard",
    "tags": ["aml", "screening", "compliance"],
    "timezone": "UTC",
    "panels": [
      {
        "title": "Screening Request Rate",
        "type": "graph",
        "gridPos": {"x": 0, "y": 0, "w": 12, "h": 8},
        "targets": [
          {
            "expr": "rate(aml_screening_requests_total[5m])",
            "legendFormat": "{{request_type}}"
          }
        ]
      },
      {
        "title": "Screening Latency (p95)",
        "type": "graph",
        "gridPos": {"x": 12, "y": 0, "w": 12, "h": 8},
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(aml_screening_duration_seconds_bucket[5m]))",
            "legendFormat": "p95 latency"
          }
        ],
        "alert": {
          "name": "High Screening Latency",
          "conditions": [
            {
              "evaluator": {"params": [0.15], "type": "gt"},
              "operator": {"type": "and"},
              "query": {"params": ["A", "5m", "now"]},
              "reducer": {"type": "avg"}
            }
          ],
          "notifications": [{"uid": "aml-alerts-channel"}]
        }
      },
      {
        "title": "Screening Decisions",
        "type": "piechart",
        "gridPos": {"x": 0, "y": 8, "w": 8, "h": 8},
        "targets": [
          {
            "expr": "sum(increase(aml_screening_decisions_total[24h])) by (decision)",
            "legendFormat": "{{decision}}"
          }
        ]
      },
      {
        "title": "Match Types Distribution",
        "type": "piechart",
        "gridPos": {"x": 8, "y": 8, "w": 8, "h": 8},
        "targets": [
          {
            "expr": "sum(increase(aml_matches_total[24h])) by (match_type)",
            "legendFormat": "{{match_type}}"
          }
        ]
      },
      {
        "title": "Open Alerts by Priority",
        "type": "bargauge",
        "gridPos": {"x": 16, "y": 8, "w": 8, "h": 8},
        "targets": [
          {
            "expr": "sum(aml_alerts_open) by (priority)",
            "legendFormat": "{{priority}}"
          }
        ]
      },
      {
        "title": "False Positive Rate",
        "type": "stat",
        "gridPos": {"x": 0, "y": 16, "w": 6, "h": 4},
        "targets": [
          {
            "expr": "avg(aml_false_positive_rate)",
            "legendFormat": "FP Rate"
          }
        ],
        "thresholds": [
          {"value": 0, "color": "green"},
          {"value": 0.02, "color": "yellow"},
          {"value": 0.03, "color": "red"}
        ]
      },
      {
        "title": "SLA Compliance",
        "type": "gauge",
        "gridPos": {"x": 6, "y": 16, "w": 6, "h": 4},
        "targets": [
          {
            "expr": "1 - (sum(increase(aml_alert_sla_breach_total[24h])) / sum(increase(aml_alerts_created_total[24h])))",
            "legendFormat": "SLA Compliance"
          }
        ]
      }
    ]
  }
}
```

### 13.4 Alert Rules

```yaml
# Prometheus Alert Rules
groups:
  - name: aml-screening-alerts
    interval: 30s
    rules:
      # Critical Alerts
      - alert: AMLScreeningServiceDown
        expr: up{job="aml-screening"} == 0
        for: 1m
        labels:
          severity: critical
          team: aml-operations
        annotations:
          summary: "AML Screening service is down"
          description: "AML Screening service has been unavailable for more than 1 minute"
          runbook_url: "https://runbooks.company.com/aml/service-down"
          
      - alert: AMLScreeningHighLatency
        expr: histogram_quantile(0.95, rate(aml_screening_duration_seconds_bucket[5m])) > 0.2
        for: 5m
        labels:
          severity: critical
          team: aml-operations
        annotations:
          summary: "AML Screening latency is too high"
          description: "p95 latency exceeds 200ms threshold: {{ $value }}s"
          
      - alert: AMLListSyncFailure
        expr: increase(aml_list_sync_errors_total[1h]) > 3
        for: 5m
        labels:
          severity: critical
          team: aml-operations
        annotations:
          summary: "AML list synchronization failing"
          description: "Multiple list sync failures detected for {{ $labels.list_code }}"
          
      # High Priority Alerts
      - alert: AMLHighFalsePositiveRate
        expr: aml_false_positive_rate > 0.05
        for: 30m
        labels:
          severity: high
          team: aml-compliance
        annotations:
          summary: "False positive rate exceeds threshold"
          description: "FP rate is {{ $value | humanizePercentage }} for {{ $labels.list_code }}"
          
      - alert: AMLAlertBacklog
        expr: sum(aml_alerts_open{priority=~"CRITICAL|HIGH"}) > 100
        for: 15m
        labels:
          severity: high
          team: aml-investigations
        annotations:
          summary: "High priority alert backlog is growing"
          description: "{{ $value }} critical/high priority alerts pending"
          
      - alert: AMLSLABreachRisk
        expr: |
          sum(aml_alerts_open{priority="CRITICAL"} 
            * on(alert_id) 
            (time() - aml_alert_created_timestamp > 3600)
          ) > 0
        for: 5m
        labels:
          severity: high
          team: aml-investigations
        annotations:
          summary: "Critical alerts approaching SLA breach"
          description: "Critical alerts close to SLA deadline"
          
      # Warning Alerts
      - alert: AMLScreeningErrorRate
        expr: |
          sum(rate(aml_screening_requests_total{status="ERROR"}[5m])) 
          / sum(rate(aml_screening_requests_total[5m])) > 0.01
        for: 10m
        labels:
          severity: warning
          team: aml-operations
        annotations:
          summary: "Screening error rate elevated"
          description: "Error rate is {{ $value | humanizePercentage }}"
          
      - alert: AMLListStale
        expr: |
          (time() - aml_list_last_sync_timestamp) > 86400
        for: 1h
        labels:
          severity: warning
          team: aml-operations
        annotations:
          summary: "Screening list not updated"
          description: "{{ $labels.list_code }} not synced for > 24 hours"
          
      - alert: AMLMatchRateAnomaly
        expr: |
          abs(
            rate(aml_matches_total[1h]) 
            - rate(aml_matches_total[24h] offset 1d)
          ) / rate(aml_matches_total[24h] offset 1d) > 0.5
        for: 1h
        labels:
          severity: warning
          team: aml-compliance
        annotations:
          summary: "Unusual match rate detected"
          description: "Match rate deviation > 50% from baseline"
```

### 13.5 Distributed Tracing

```yaml
# OpenTelemetry Configuration
opentelemetry:
  service-name: aml-screening
  
  traces:
    enabled: true
    sampler:
      type: parentbased_traceidratio
      ratio: 0.1  # 10% sampling
      # Always sample errors and slow requests
      rules:
        - type: always_on
          condition: "status.code == ERROR"
        - type: always_on
          condition: "duration > 200ms"
    
    exporters:
      - type: otlp
        endpoint: otel-collector:4317
        
  spans:
    # Custom span attributes
    attributes:
      request.type: screening_request.request_type
      risk.score: screening_result.risk_score
      match.count: screening_result.match_count
      decision: screening_result.decision
      
    # Span naming
    naming:
      pattern: "aml.{operation}"
      operations:
        - screen_transaction
        - match_subject
        - calculate_risk_score
        - create_alert
        - resolve_alert
        - sync_list

# Trace Context Propagation
context-propagation:
  carriers:
    - W3C Trace Context
    - B3 Multi
  
  # Headers to propagate
  headers:
    - traceparent
    - tracestate
    - X-B3-TraceId
    - X-B3-SpanId
    - X-B3-Sampled
    - X-Request-Id
    - X-Correlation-Id
```

---

## 14. Test Strategy

### 14.1 Test Pyramid

```
                            ┌─────────────┐
                            │   E2E &     │
                            │ Compliance  │     5%
                            │   Tests     │
                           ┌┴─────────────┴┐
                           │  Integration  │
                           │    Tests      │    15%
                          ┌┴───────────────┴┐
                          │   Component     │
                          │    Tests        │   25%
                         ┌┴─────────────────┴┐
                         │    Unit Tests     │
                         │                   │   55%
                         └───────────────────┘
```

### 14.2 Unit Tests

```java
// Example: ScreeningEngineTest.java
@ExtendWith(MockitoExtension.class)
class ScreeningEngineTest {

    @Mock
    private SanctionsListRepository sanctionsListRepository;
    
    @Mock
    private NameMatchingService nameMatchingService;
    
    @Mock
    private RiskScoringService riskScoringService;
    
    @InjectMocks
    private ScreeningEngine screeningEngine;
    
    @Test
    @DisplayName("Should return PASS when no matches found")
    void shouldReturnPassWhenNoMatches() {
        // Given
        Subject subject = Subject.builder()
            .type(SubjectType.INDIVIDUAL)
            .name(PersonName.of("John", "Smith"))
            .build();
            
        ScreeningRequest request = ScreeningRequest.builder()
            .requestId(RequestId.generate())
            .subjects(List.of(subject))
            .config(ScreeningConfig.defaults())
            .build();
            
        when(nameMatchingService.findMatches(any(), any()))
            .thenReturn(Collections.emptyList());
            
        // When
        ScreeningResult result = screeningEngine.screen(request);
        
        // Then
        assertThat(result.getDecision()).isEqualTo(ScreeningDecision.PASS);
        assertThat(result.getMatches()).isEmpty();
        assertThat(result.getRiskScore()).isLessThan(30);
    }
    
    @Test
    @DisplayName("Should return BLOCK when exact sanctions match found")
    void shouldReturnBlockOnExactSanctionsMatch() {
        // Given
        Subject subject = Subject.builder()
            .type(SubjectType.INDIVIDUAL)
            .name(PersonName.of("VLADIMIR", "IVANOV"))
            .dateOfBirth(LocalDate.of(1965, 8, 20))
            .build();
            
        Match exactMatch = Match.builder()
            .matchType(MatchType.SANCTIONS)
            .matchScore(MatchScore.of(98))
            .listCode(ListCode.of("OFAC_SDN"))
            .matchedEntry(createSanctionedEntry())
            .build();
            
        when(nameMatchingService.findMatches(any(), any()))
            .thenReturn(List.of(exactMatch));
            
        when(riskScoringService.calculateScore(any()))
            .thenReturn(RiskScore.of(95));
            
        // When
        ScreeningResult result = screeningEngine.screen(createRequest(subject));
        
        // Then
        assertThat(result.getDecision()).isEqualTo(ScreeningDecision.BLOCK);
        assertThat(result.getMatches()).hasSize(1);
        assertThat(result.getRiskScore()).isGreaterThanOrEqualTo(85);
    }
    
    @Test
    @DisplayName("Should apply all screening types in parallel")
    void shouldApplyAllScreeningTypesInParallel() {
        // Given
        ScreeningConfig config = ScreeningConfig.builder()
            .enableSanctionsScreening(true)
            .enablePepScreening(true)
            .enableWatchlistScreening(true)
            .enableAdverseMediaScreening(true)
            .build();
            
        Subject subject = createTestSubject();
        ScreeningRequest request = createRequest(subject, config);
        
        // When
        StopWatch stopWatch = StopWatch.createStarted();
        ScreeningResult result = screeningEngine.screen(request);
        stopWatch.stop();
        
        // Then
        verify(nameMatchingService, times(4)).findMatches(any(), any());
        // Parallel execution should be faster than sequential
        assertThat(stopWatch.getTime()).isLessThan(500);
    }
}

// Example: NameMatchingServiceTest.java
@ExtendWith(MockitoExtension.class)
class NameMatchingServiceTest {

    @InjectMocks
    private NameMatchingService nameMatchingService;
    
    @ParameterizedTest
    @CsvSource({
        "VLADIMIR IVANOV, IVANOV VLADIMIR, 95",
        "JOHN SMITH, JON SMYTH, 82",
        "MOHAMMED ALI, MUHAMMAD ALI, 90",
        "MUELLER HANS, MULLER HANS, 88",
        "中文名字, 中文名字, 100"
    })
    @DisplayName("Should calculate correct match scores for name variations")
    void shouldCalculateCorrectMatchScores(
            String inputName, String listName, int expectedMinScore) {
        // Given
        PersonName input = PersonName.parse(inputName);
        ListEntry entry = createEntryWithName(listName);
        
        // When
        Optional<Match> match = nameMatchingService.match(input, entry);
        
        // Then
        assertThat(match).isPresent();
        assertThat(match.get().getMatchScore().getValue())
            .isGreaterThanOrEqualTo(expectedMinScore);
    }
    
    @Test
    @DisplayName("Should apply phonetic matching for similar sounding names")
    void shouldApplyPhoneticMatching() {
        // Given
        PersonName input = PersonName.of("STEVEN", "JOBS");
        ListEntry entry = createEntryWithName("STEPHEN JOBS");
        
        // When
        Optional<Match> match = nameMatchingService.matchWithPhonetics(input, entry);
        
        // Then
        assertThat(match).isPresent();
        assertThat(match.get().getMatchingAlgorithms())
            .contains(MatchingAlgorithm.SOUNDEX, MatchingAlgorithm.METAPHONE);
    }
}
```

### 14.3 Integration Tests

```java
// Example: ScreeningIntegrationTest.java
@SpringBootTest
@Testcontainers
@AutoConfigureTestDatabase(replace = Replace.NONE)
class ScreeningIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("aml_test")
        .withInitScript("sql/init-test-data.sql");
        
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7")
        .withExposedPorts(6379);
        
    @Container
    static ElasticsearchContainer elasticsearch = 
        new ElasticsearchContainer("docker.elastic.co/elasticsearch/elasticsearch:8.11.0");
    
    @Autowired
    private ScreeningService screeningService;
    
    @Autowired
    private AlertRepository alertRepository;
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", () -> redis.getMappedPort(6379));
        registry.add("spring.elasticsearch.uris", elasticsearch::getHttpHostAddress);
    }
    
    @Test
    @DisplayName("Should screen transaction end-to-end and create alert")
    void shouldScreenTransactionAndCreateAlert() {
        // Given
        ScreenTransactionCommand command = ScreenTransactionCommand.builder()
            .transactionId(TransactionId.of("txn-12345"))
            .originator(Subject.builder()
                .name(PersonName.of("VLADIMIR", "IVANOV"))
                .type(SubjectType.INDIVIDUAL)
                .build())
            .beneficiary(Subject.builder()
                .name(PersonName.of("ACME", "CORPORATION"))
                .type(SubjectType.ORGANIZATION)
                .build())
            .amount(Money.of(100000, "USD"))
            .build();
            
        // When
        ScreeningResultDto result = screeningService.screenTransaction(command);
        
        // Then
        assertThat(result.getDecision()).isEqualTo(ScreeningDecision.REVIEW);
        assertThat(result.getAlertReference()).isNotNull();
        
        // Verify alert was created
        Optional<Alert> alert = alertRepository.findByReference(result.getAlertReference());
        assertThat(alert).isPresent();
        assertThat(alert.get().getStatus()).isEqualTo(AlertStatus.OPEN);
    }
    
    @Test
    @DisplayName("Should process batch screening with multiple subjects")
    void shouldProcessBatchScreening() {
        // Given
        List<Subject> subjects = IntStream.range(0, 100)
            .mapToObj(i -> createRandomSubject())
            .collect(Collectors.toList());
            
        BatchScreeningCommand command = BatchScreeningCommand.builder()
            .batchId(BatchId.generate())
            .subjects(subjects)
            .priority(Priority.NORMAL)
            .build();
            
        // When
        Instant start = Instant.now();
        BatchScreeningResultDto result = screeningService.screenBatch(command);
        Duration duration = Duration.between(start, Instant.now());
        
        // Then
        assertThat(result.getProcessedCount()).isEqualTo(100);
        assertThat(duration).isLessThan(Duration.ofSeconds(30));
    }
}

// Example: KafkaIntegrationTest.java
@SpringBootTest
@EmbeddedKafka(partitions = 3, topics = {
    "aml.screening.requests",
    "aml.screening.results",
    "aml.matches.found"
})
class KafkaIntegrationTest {

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Autowired
    private EmbeddedKafkaBroker embeddedKafka;
    
    @SpyBean
    private ScreeningEventHandler screeningEventHandler;
    
    @Test
    @DisplayName("Should process screening request from Kafka topic")
    void shouldProcessScreeningRequestFromKafka() throws Exception {
        // Given
        ScreeningRequestedEvent event = ScreeningRequestedEvent.builder()
            .requestId("req-12345")
            .transactionId("txn-67890")
            .subjects(List.of(createTestSubjectDto()))
            .timestamp(Instant.now())
            .build();
            
        // When
        kafkaTemplate.send("aml.screening.requests", event.getRequestId(), event);
        
        // Then
        await().atMost(Duration.ofSeconds(10))
            .untilAsserted(() -> {
                verify(screeningEventHandler).handle(argThat(e -> 
                    e.getRequestId().equals("req-12345")));
            });
    }
    
    @Test
    @DisplayName("Should publish match event when match detected")
    void shouldPublishMatchEventWhenMatchDetected() throws Exception {
        // Setup consumer
        Map<String, Object> consumerProps = KafkaTestUtils.consumerProps(
            "test-group", "true", embeddedKafka);
        ConsumerFactory<String, MatchDetectedEvent> cf = 
            new DefaultKafkaConsumerFactory<>(consumerProps);
        Consumer<String, MatchDetectedEvent> consumer = cf.createConsumer();
        embeddedKafka.consumeFromAnEmbeddedTopic(consumer, "aml.matches.found");
        
        // Given - Request with known sanctioned entity
        ScreeningRequestedEvent event = createSanctionedEntityRequest();
        
        // When
        kafkaTemplate.send("aml.screening.requests", event.getRequestId(), event);
        
        // Then
        ConsumerRecords<String, MatchDetectedEvent> records = 
            KafkaTestUtils.getRecords(consumer, Duration.ofSeconds(10));
        assertThat(records.count()).isGreaterThan(0);
        
        MatchDetectedEvent matchEvent = records.iterator().next().value();
        assertThat(matchEvent.getMatchType()).isEqualTo("SANCTIONS");
    }
}
```

### 14.4 Compliance Tests

```java
// Example: SanctionsComplianceTest.java
@SpringBootTest
@Tag("compliance")
class SanctionsComplianceTest {

    @Autowired
    private ScreeningService screeningService;
    
    @ParameterizedTest
    @MethodSource("ofacTestCases")
    @DisplayName("Should detect all OFAC SDN list entries")
    void shouldDetectOFACEntries(String name, String expectedId) {
        // Given
        Subject subject = Subject.builder()
            .name(PersonName.parse(name))
            .type(SubjectType.INDIVIDUAL)
            .build();
            
        // When
        ScreeningResult result = screeningService.screen(
            ScreeningRequest.forSubject(subject));
            
        // Then
        assertThat(result.getDecision())
            .as("Should block sanctioned entity: %s", name)
            .isIn(ScreeningDecision.BLOCK, ScreeningDecision.REVIEW);
        assertThat(result.getMatches())
            .anyMatch(m -> m.getListEntryId().equals(expectedId));
    }
    
    static Stream<Arguments> ofacTestCases() {
        return Stream.of(
            Arguments.of("VLADIMIR IVANOV", "SDN-12345"),
            Arguments.of("V. IVANOV", "SDN-12345"),
            Arguments.of("IVANOV, VLADIMIR P.", "SDN-12345"),
            // Name transliterations
            Arguments.of("WLADIMIR IWANOW", "SDN-12345"),
            // Partial matches
            Arguments.of("VLADIMIR PETROV IVANOV", "SDN-12345")
        );
    }
    
    @Test
    @DisplayName("Should detect all PEP categories as per regulatory requirements")
    void shouldDetectAllPEPCategories() {
        // Given - Test subjects for each PEP category
        Map<String, PEPCategory> testCases = Map.of(
            "HEAD_OF_STATE", PEPCategory.DOMESTIC_PEP,
            "CABINET_MINISTER", PEPCategory.DOMESTIC_PEP,
            "AMBASSADOR", PEPCategory.FOREIGN_PEP,
            "SENIOR_EXECUTIVE_IO", PEPCategory.INTERNATIONAL_ORG
        );
        
        // When/Then
        testCases.forEach((testName, expectedCategory) -> {
            Subject subject = loadPEPTestSubject(testName);
            ScreeningResult result = screeningService.screen(
                ScreeningRequest.forSubject(subject));
                
            assertThat(result.getMatches())
                .as("Should detect PEP: %s", testName)
                .anyMatch(m -> 
                    m.getMatchType() == MatchType.PEP &&
                    m.getPepCategory() == expectedCategory);
        });
    }
    
    @Test
    @DisplayName("Should maintain false positive rate below threshold")
    void shouldMaintainFalsePositiveRateBelowThreshold() {
        // Given - Load known good customer dataset
        List<Subject> knownGoodCustomers = loadKnownGoodCustomers(1000);
        
        // When
        List<ScreeningResult> results = knownGoodCustomers.stream()
            .map(s -> screeningService.screen(ScreeningRequest.forSubject(s)))
            .collect(Collectors.toList());
            
        // Then
        long falsePositives = results.stream()
            .filter(r -> r.getDecision() != ScreeningDecision.PASS)
            .count();
            
        double falsePositiveRate = (double) falsePositives / knownGoodCustomers.size();
        
        assertThat(falsePositiveRate)
            .as("False positive rate should be below 3%")
            .isLessThan(0.03);
    }
}

// Example: RegulatoryReportingComplianceTest.java
@SpringBootTest
@Tag("compliance")
class RegulatoryReportingComplianceTest {

    @Autowired
    private SARFilingService sarFilingService;
    
    @Autowired
    private SARValidator sarValidator;
    
    @Test
    @DisplayName("SAR should contain all required FinCEN fields")
    void sarShouldContainAllRequiredFields() {
        // Given
        Alert alert = createAlertRequiringSAR();
        
        // When
        SAR sar = sarFilingService.generateSAR(alert);
        
        // Then - Validate against FinCEN schema
        ValidationResult validation = sarValidator.validate(sar);
        
        assertThat(validation.isValid()).isTrue();
        assertThat(validation.getMissingFields()).isEmpty();
    }
    
    @Test
    @DisplayName("Should meet SAR filing deadline requirements")
    void shouldMeetSARFilingDeadlines() {
        // Given
        Alert alert = createHighRiskAlert();
        Instant alertCreatedAt = alert.getCreatedAt();
        
        // When
        sarFilingService.processForSAR(alert);
        Optional<SAR> sar = sarRepository.findByAlertId(alert.getId());
        
        // Then - SAR should be filed within 30 days
        assertThat(sar).isPresent();
        assertThat(sar.get().getFiledAt())
            .isBefore(alertCreatedAt.plus(Duration.ofDays(30)));
    }
}
```

### 14.5 Performance Tests

```java
// Example: ScreeningPerformanceTest.java
@SpringBootTest
@Tag("performance")
class ScreeningPerformanceTest {

    @Autowired
    private ScreeningService screeningService;
    
    @Test
    @DisplayName("Should process single screening within 200ms SLA")
    void shouldMeetLatencySLA() {
        // Given
        Subject subject = createTypicalCustomer();
        ScreeningRequest request = ScreeningRequest.forSubject(subject);
        
        // Warm up
        IntStream.range(0, 100).forEach(i -> 
            screeningService.screen(request));
        
        // When - Measure p99 latency
        List<Long> latencies = IntStream.range(0, 1000)
            .mapToObj(i -> {
                Instant start = Instant.now();
                screeningService.screen(request);
                return Duration.between(start, Instant.now()).toMillis();
            })
            .sorted()
            .collect(Collectors.toList());
            
        long p50 = latencies.get(499);
        long p95 = latencies.get(949);
        long p99 = latencies.get(989);
        
        // Then
        assertThat(p50).as("p50 latency").isLessThan(100);
        assertThat(p95).as("p95 latency").isLessThan(150);
        assertThat(p99).as("p99 latency").isLessThan(200);
    }
    
    @Test
    @DisplayName("Should handle 5000 TPS throughput")
    void shouldHandleTargetThroughput() throws Exception {
        // Given
        int targetTPS = 5000;
        Duration testDuration = Duration.ofSeconds(30);
        AtomicLong successCount = new AtomicLong(0);
        AtomicLong errorCount = new AtomicLong(0);
        
        ExecutorService executor = Executors.newFixedThreadPool(100);
        RateLimiter rateLimiter = RateLimiter.create(targetTPS);
        
        // When
        Instant end = Instant.now().plus(testDuration);
        List<Future<?>> futures = new ArrayList<>();
        
        while (Instant.now().isBefore(end)) {
            rateLimiter.acquire();
            futures.add(executor.submit(() -> {
                try {
                    screeningService.screen(createRandomRequest());
                    successCount.incrementAndGet();
                } catch (Exception e) {
                    errorCount.incrementAndGet();
                }
            }));
        }
        
        // Wait for completion
        executor.shutdown();
        executor.awaitTermination(1, TimeUnit.MINUTES);
        
        // Then
        double actualTPS = successCount.get() / testDuration.toSeconds();
        double errorRate = (double) errorCount.get() / 
            (successCount.get() + errorCount.get());
        
        assertThat(actualTPS)
            .as("Should achieve at least 90% of target TPS")
            .isGreaterThan(targetTPS * 0.9);
        assertThat(errorRate)
            .as("Error rate should be below 0.1%")
            .isLessThan(0.001);
    }
    
    @Test
    @DisplayName("Should scale batch processing linearly")
    void shouldScaleBatchProcessingLinearly() {
        // Given
        List<Integer> batchSizes = List.of(100, 500, 1000, 5000);
        Map<Integer, Long> processingTimes = new LinkedHashMap<>();
        
        // When
        for (int size : batchSizes) {
            List<Subject> subjects = IntStream.range(0, size)
                .mapToObj(i -> createRandomSubject())
                .collect(Collectors.toList());
                
            Instant start = Instant.now();
            screeningService.screenBatch(BatchScreeningCommand.of(subjects));
            long duration = Duration.between(start, Instant.now()).toMillis();
            
            processingTimes.put(size, duration);
        }
        
        // Then - Processing time should scale linearly (not exponentially)
        // Time for 5000 should be roughly 50x time for 100 (with some overhead)
        double scalingFactor = (double) processingTimes.get(5000) / 
            processingTimes.get(100);
        
        assertThat(scalingFactor)
            .as("Should scale roughly linearly")
            .isBetween(40.0, 60.0);
    }
}
```

### 14.6 Test Data Management

```yaml
# Test Data Strategy
test-data:
  synthetic:
    # Faker-based generation for most tests
    customers:
      generator: com.company.test.CustomerDataGenerator
      seed: 12345
      
  golden:
    # Known-good datasets for compliance tests
    sanctions-list: classpath:test-data/ofac-sdn-test-subset.json
    pep-list: classpath:test-data/pep-test-subset.json
    known-good-customers: classpath:test-data/known-good-customers.json
    
  anonymized:
    # Anonymized production data for realistic testing
    source: production-sanitized
    refresh: weekly
    encryption: required
    
  edge-cases:
    # Specific edge cases for boundary testing
    source: classpath:test-data/edge-cases/
    categories:
      - name-variations
      - unicode-names
      - special-characters
      - transliterations
```

---

## 15. Implementation Roadmap

### 15.1 Phased Implementation Plan

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AML MODULE IMPLEMENTATION ROADMAP                         │
└─────────────────────────────────────────────────────────────────────────────┘

Phase 1: Foundation (Weeks 1-6)
├── Core Domain Model
├── Basic Name Matching (Exact + Fuzzy)
├── OFAC/EU List Integration
├── Simple Alert Creation
└── Basic API Endpoints

Phase 2: Enhanced Matching (Weeks 7-10)
├── Advanced Name Matching Algorithms
├── PEP Screening Integration
├── Watchlist Management
├── Match Confidence Scoring
└── Caching Layer

Phase 3: Alert Management (Weeks 11-14)
├── Full Alert Lifecycle
├── Case Management Integration
├── Investigation Workflow
├── Disposition & Resolution
└── Analytics Dashboard

Phase 4: Pattern Detection (Weeks 15-18)
├── Transaction Pattern Analysis
├── Velocity Rules
├── Geographic Risk Analysis
├── Behavioral Analytics
└── ML Model Integration

Phase 5: Regulatory Reporting (Weeks 19-22)
├── SAR Generation
├── CTR Automation
├── Multi-Regulator Support
├── Audit Trail Enhancement
└── Compliance Dashboard

Phase 6: Optimization (Weeks 23-26)
├── Performance Tuning
├── False Positive Reduction
├── Model Retraining Pipeline
├── Production Stabilization
└── Documentation & Training
```

### 15.2 Detailed Phase Breakdown

#### Phase 1: Foundation (Weeks 1-6)

| Week | Deliverable | Dependencies | Team |
|------|-------------|--------------|------|
| 1-2 | Domain Model & Repository Layer | None | Backend |
| 2-3 | Basic Screening Engine | Domain Model | Backend |
| 3-4 | OFAC/EU List Ingestion | Database | Backend + DevOps |
| 4-5 | REST API Implementation | Screening Engine | Backend |
| 5-6 | Basic Alert Creation | API | Backend |
| 6 | Integration Testing | All above | QA |

**Phase 1 Exit Criteria:**
- [ ] Screen individual/organization against OFAC SDN list
- [ ] Return pass/review/block decision
- [ ] Create basic alerts for matches
- [ ] API contracts published and documented
- [ ] 80% unit test coverage

#### Phase 2: Enhanced Matching (Weeks 7-10)

| Week | Deliverable | Dependencies | Team |
|------|-------------|--------------|------|
| 7 | Phonetic Matching (Soundex/Metaphone) | Phase 1 | Backend |
| 7-8 | Transliteration Support | Phase 1 | Backend |
| 8-9 | World-Check PEP Integration | API contracts | Backend |
| 9 | Redis Caching Layer | Infrastructure | Backend + DevOps |
| 9-10 | Watchlist CRUD & Management | Database | Backend |
| 10 | Performance Testing | All above | QA + Performance |

**Phase 2 Exit Criteria:**
- [ ] Name matching accuracy > 95%
- [ ] Screening latency p99 < 200ms
- [ ] PEP screening operational
- [ ] Watchlist management functional
- [ ] Cache hit rate > 80%

#### Phase 3: Alert Management (Weeks 11-14)

| Week | Deliverable | Dependencies | Team |
|------|-------------|--------------|------|
| 11 | Alert State Machine | Phase 2 | Backend |
| 11-12 | Alert Assignment & Routing | Phase 2 | Backend |
| 12-13 | Case Management Integration | External API | Backend + Integration |
| 13 | Investigation UI Components | API | Frontend |
| 13-14 | Disposition Workflow | All above | Backend + Frontend |
| 14 | UAT for Alert Workflow | All above | QA + Business |

**Phase 3 Exit Criteria:**
- [ ] Full alert lifecycle implemented
- [ ] Case management integration working
- [ ] Investigation workflow usable
- [ ] SLA tracking operational
- [ ] User acceptance testing passed

#### Phase 4: Pattern Detection (Weeks 15-18)

| Week | Deliverable | Dependencies | Team |
|------|-------------|--------------|------|
| 15-16 | TimescaleDB Integration | Infrastructure | Backend + DevOps |
| 16-17 | Transaction Pattern Rules | TimescaleDB | Backend |
| 17 | Velocity Detection | Pattern Rules | Backend |
| 17-18 | Geographic Risk Engine | Reference Data | Backend |
| 18 | ML Model Integration | All above | Data Science + Backend |

**Phase 4 Exit Criteria:**
- [ ] Pattern detection accuracy > 90%
- [ ] Velocity rules triggering correctly
- [ ] Geographic risk scoring operational
- [ ] ML model deployed and monitored
- [ ] Pattern analysis latency < 50ms

#### Phase 5: Regulatory Reporting (Weeks 19-22)

| Week | Deliverable | Dependencies | Team |
|------|-------------|--------------|------|
| 19-20 | SAR Generation Engine | Phase 3 | Backend |
| 20 | FinCEN E-Filing Integration | SAR Engine | Backend + Compliance |
| 21 | CTR Automation | Transaction Data | Backend |
| 21-22 | Multi-Regulator Support | SAR Engine | Backend |
| 22 | Compliance Dashboard | All above | Frontend |

**Phase 5 Exit Criteria:**
- [ ] SAR auto-generation working
- [ ] E-Filing integration tested
- [ ] CTR automation operational
- [ ] Support for US, UK, EU regulators
- [ ] Compliance audit passed

#### Phase 6: Optimization (Weeks 23-26)

| Week | Deliverable | Dependencies | Team |
|------|-------------|--------------|------|
| 23 | Performance Optimization | All phases | Backend + Performance |
| 23-24 | False Positive Tuning | Production data | Data Science |
| 24-25 | Model Retraining Pipeline | ML Infrastructure | Data Science + MLOps |
| 25 | Production Hardening | All above | DevOps + SRE |
| 26 | Documentation & Training | All above | All teams |

**Phase 6 Exit Criteria:**
- [ ] All NFRs met
- [ ] False positive rate < 3%
- [ ] Model retraining automated
- [ ] Production monitoring complete
- [ ] Team trained and documented

### 15.3 Risk Registry

| Risk ID | Description | Probability | Impact | Mitigation |
|---------|-------------|-------------|--------|------------|
| R-001 | External list provider API instability | Medium | High | Implement caching, fallback to cached lists |
| R-002 | High false positive rate impacting operations | High | Medium | Iterative threshold tuning, ML model training |
| R-003 | Performance degradation under peak load | Medium | High | Early performance testing, horizontal scaling |
| R-004 | Regulatory requirement changes | Medium | High | Modular design, configurable rule engine |
| R-005 | Integration delays with Case Management | Medium | Medium | Mock services, async integration |
| R-006 | Name matching accuracy below target | Low | High | Multiple algorithm fallback, continuous improvement |
| R-007 | Data privacy compliance issues | Low | Critical | Privacy by design, encryption, access controls |

### 15.4 Success Metrics

| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| Screening Latency (p99) | < 200ms | Application metrics |
| True Positive Rate | > 99.9% | Compliance testing |
| False Positive Rate | < 3% | Production monitoring |
| Alert Resolution Time (avg) | < 24 hours | Alert metrics |
| SAR Filing Timeliness | 100% on-time | Compliance dashboard |
| System Availability | 99.99% | Infrastructure monitoring |
| Team Training Completion | 100% | Training records |

---

## Appendix A: Glossary

| Term | Definition |
|------|------------|
| AML | Anti-Money Laundering |
| BSA | Bank Secrecy Act |
| CTR | Currency Transaction Report |
| EDD | Enhanced Due Diligence |
| FATF | Financial Action Task Force |
| KYC | Know Your Customer |
| OFAC | Office of Foreign Assets Control |
| PEP | Politically Exposed Person |
| RCA | Relatives and Close Associates |
| SAR | Suspicious Activity Report |
| SDN | Specially Designated Nationals |
| SOE | State-Owned Enterprise |
| STR | Suspicious Transaction Report |

---

## Appendix B: Reference Documents

| Document | Description |
|----------|-------------|
| FATF Recommendations | 40 FATF recommendations for AML/CFT |
| FinCEN BSA Regulations | US Bank Secrecy Act requirements |
| 6AMLD | EU 6th Anti-Money Laundering Directive |
| OFAC SDN List | Specially Designated Nationals list |
| World-Check API Documentation | PEP screening integration guide |
| ISO 20022 Pain Messages | Payment message standards |

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-20 | Architecture Team | Initial version |

---

*This document is part of the Payment Gateway Context documentation suite.*
