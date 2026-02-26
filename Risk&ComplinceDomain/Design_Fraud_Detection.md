# Fraud Detection Module
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

The **Fraud Detection** module is a critical component of the Risk & Compliance Domain responsible for identifying, preventing, and mitigating fraudulent activities within payment transactions in real-time. This module implements comprehensive fraud detection capabilities using machine learning models, rule-based engines, behavioral analytics, and device fingerprinting to protect the payment ecosystem from various fraud types including account takeover, payment fraud, identity theft, and synthetic fraud.

### Key Capabilities

| Capability | Description |
|------------|-------------|
| Real-Time Scoring | Sub-second fraud risk scoring for every transaction |
| Machine Learning Engine | ML models for pattern recognition and anomaly detection |
| Rule-Based Detection | Configurable business rules for known fraud patterns |
| Behavioral Analytics | Customer behavior profiling and deviation detection |
| Device Intelligence | Device fingerprinting and reputation scoring |
| Velocity Controls | Transaction frequency and amount velocity tracking |
| Network Analysis | Detection of fraud rings and collusion patterns |
| Entity Resolution | Linking related accounts and identities across channels |
| Adaptive Authentication | Step-up authentication triggers based on risk |
| Case Management Integration | Automated case creation for fraud investigation |
| Real-Time Alerts | Instant notifications for high-risk transactions |
| Fraud Analytics Dashboard | Comprehensive reporting and trend analysis |

### Business Value

- **Loss Prevention**: Reduce fraud losses through real-time detection and blocking
- **Customer Protection**: Shield customers from unauthorized transactions and account takeover
- **Operational Efficiency**: Automated detection reduces manual review workload
- **False Positive Optimization**: ML-driven accuracy minimizes customer friction
- **Regulatory Compliance**: Meet PSD2 SCA, FFIEC, and regulatory fraud requirements
- **Reputation Protection**: Prevent fraud-related reputational damage
- **Real-Time Protection**: Sub-200ms decision time enables seamless customer experience
- **Adaptive Defense**: Continuously learning models adapt to evolving fraud patterns
- **Cost Reduction**: Lower chargeback rates and dispute handling costs
- **Trust Building**: Enhanced security increases customer confidence

### Architecture Context (ADR-005a)

The Fraud Detection module operates within the **Risk & Compliance Domain** and is triggered by the **Validation Pipeline** (Stage 4: Duplicate Detection) within the Payment Orchestration Domain when transactions require fraud assessment:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PAYMENT ORCHESTRATION DOMAIN                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Payment Gateway Core → Validation Pipeline (Stage 4) → Limit Control      │
│                                    │                                        │
│                                    │ (Fraud Check Trigger)                  │
│                                    ▼                                        │
└────────────────────────────────────┼────────────────────────────────────────┘
                                     │
                    ┌────────────────┼────────────────────────────────────────┐
                    │              RISK & COMPLIANCE DOMAIN                    │
                    ├────────────────┼────────────────────────────────────────┤
                    │                ▼                                        │
                    │  ┌────────────────────────────────────────────────────┐ │
                    │  │         FRAUD DETECTION MODULE (This Module)        │ │
                    │  │                                                     │ │
                    │  │   ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │ │
                    │  │   │   ML Model  │  │ Rule Engine │  │ Behavioral │ │ │
                    │  │   │   Scoring   │  │  Evaluation │  │  Analytics │ │ │
                    │  │   └─────────────┘  └─────────────┘  └────────────┘ │ │
                    │  │          │               │               │         │ │
                    │  │          └───────────────┼───────────────┘         │ │
                    │  │                          ▼                         │ │
                    │  │   ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │ │
                    │  │   │   Device    │  │  Velocity   │  │  Network   │ │ │
                    │  │   │  Analysis   │  │   Checks    │  │  Analysis  │ │ │
                    │  │   └─────────────┘  └─────────────┘  └────────────┘ │ │
                    │  │                          │                         │ │
                    │  │                          ▼                         │ │
                    │  │   ┌─────────────────────────────────────────────┐ │ │
                    │  │   │           FRAUD DECISION ENGINE             │ │ │
                    │  │   │  • ALLOW • CHALLENGE • REVIEW • BLOCK       │ │ │
                    │  │   └─────────────────────────────────────────────┘ │ │
                    │  │          │               │               │         │ │
                    │  │          ▼               ▼               ▼         │ │
                    │  │      Continue       Step-Up Auth      Reject       │ │
                    │  │      Payment        Challenge         Payment      │ │
                    │  │                                                     │ │
                    │  └─────────────────────────────────────────────────────┘ │
                    │                                                          │
                    │  ┌──────────────────┐  ┌──────────────────────────────┐ │
                    │  │   AML Screening  │  │  Regulatory Reporting (SAR)  │ │
                    │  └──────────────────┘  └──────────────────────────────┘ │
                    │                                                          │
                    └──────────────────────────────────────────────────────────┘
```

### Fraud Type Coverage

| Fraud Type | Detection Method | Description |
|------------|------------------|-------------|
| Account Takeover (ATO) | Behavioral + Device | Unauthorized access to customer accounts |
| Payment Fraud | ML + Rules | Unauthorized or fraudulent payment transactions |
| Card-Not-Present (CNP) | ML + Velocity | Online card fraud without physical card |
| Card-Present Fraud | Device + Location | In-person fraud with stolen/cloned cards |
| Identity Theft | Entity Resolution | Use of stolen personal information |
| Synthetic Identity | Network Analysis | Fabricated identities combining real/fake data |
| First-Party Fraud | Behavioral | Fraud committed by legitimate account holders |
| Authorized Push Payment | Behavioral + Rules | Social engineering scams |
| Business Email Compromise | Pattern + Rules | Corporate email impersonation |
| Money Mule Detection | Network + Velocity | Detection of mule account activity |
| Credential Stuffing | Device + Velocity | Automated login attempts with stolen credentials |
| SIM Swap Fraud | Device + Behavioral | Mobile number hijacking for 2FA bypass |

### Performance Metrics

| Metric | Target | Description |
|--------|--------|-------------|
| Scoring Latency (p99) | < 150ms | End-to-end fraud scoring response time |
| ML Model Inference | < 50ms | Single model inference time |
| Rule Engine Evaluation | < 30ms | Rule evaluation completion time |
| Behavioral Check | < 40ms | Profile comparison time |
| Device Analysis | < 25ms | Device fingerprint analysis time |
| Throughput | > 10,000 TPS | Fraud checks per second |
| Model Update Latency | < 1 hour | Time to deploy model updates |
| False Positive Rate | < 2% | Target false positive percentage |
| True Positive Detection | > 95% | Fraud detection rate |
| Model Accuracy (AUC) | > 0.95 | Area under ROC curve |

### Regulatory Compliance Coverage

| Regulation | Jurisdiction | Coverage |
|------------|--------------|----------|
| PSD2 SCA | EU | Strong Customer Authentication triggers |
| FFIEC Guidance | USA | Online banking fraud prevention |
| UK Payment Services | UK | APP fraud protection measures |
| MAS TRM Guidelines | Singapore | Technology risk management |
| APRA CPG 234 | Australia | Information security fraud controls |
| PCI DSS | Global | Card data security and fraud prevention |
| GDPR | EU | Data protection in fraud processing |
| CCPA | California | Consumer data rights in fraud systems |

---

## 2. Module Overview

### 2.1 Bounded Context

The Fraud Detection module belongs to the **Risk & Compliance Domain** bounded context and operates as a dedicated **Fraud Detection Service** (Service Port: 8004).

### 2.2 Module Scope

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        FRAUD DETECTION MODULE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  FRAUD ORCHESTRATOR                                  │   │
│  │  • Request Handling  • Parallel Evaluation  • Result Aggregation    │   │
│  │  • Decision Making   • Alert Generation     • Response Formation    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│         ┌──────────────────────────┼──────────────────────────┐            │
│         │                          │                          │            │
│         ▼                          ▼                          ▼            │
│  ┌─────────────────┐  ┌────────────────────┐  ┌─────────────────────────┐ │
│  │  ML MODEL       │  │  RULE ENGINE       │  │  BEHAVIORAL            │ │
│  │  SCORING        │  │                    │  │  ANALYTICS             │ │
│  │                 │  │ • Fraud Rules      │  │                         │ │
│  │ • XGBoost       │  │ • Velocity Rules   │  │ • Profile Comparison   │ │
│  │ • Neural Network│  │ • Threshold Rules  │  │ • Deviation Scoring    │ │
│  │ • Ensemble      │  │ • Conditional Rules│  │ • Habit Analysis       │ │
│  │ • Real-time     │  │ • Blocklist Rules  │  │ • Peer Group Compare   │ │
│  │ • Batch Model   │  │ • Whitelist Rules  │  │ • Historical Patterns  │ │
│  │                 │  │                    │  │                         │ │
│  │ Features:       │  │ Sources:           │  │ Signals:                │ │
│  │ • Transaction   │  │ • Internal DB      │  │ • Login patterns       │ │
│  │ • Customer      │  │ • External Feeds   │  │ • Transaction habits   │ │
│  │ • Device        │  │ • Manual Config    │  │ • Channel preferences  │ │
│  │ • Contextual    │  │ • API Imports      │  │ • Beneficiary patterns │ │
│  └─────────────────┘  └────────────────────┘  └─────────────────────────┘ │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  DEVICE INTELLIGENCE ENGINE                          │   │
│  │                                                                      │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │   │
│  │  │ Device          │  │ Device          │  │ Device              │  │   │
│  │  │ Fingerprinting  │  │ Reputation      │  │ Binding             │  │   │
│  │  │                 │  │                 │  │                     │  │   │
│  │  │ • Browser FP    │  │ • Trust Score   │  │ • Known Devices     │  │   │
│  │  │ • Mobile FP     │  │ • Risk Level    │  │ • Device History    │  │   │
│  │  │ • Hardware ID   │  │ • Fraud History │  │ • Account Links     │  │   │
│  │  │ • Canvas FP     │  │ • Age Score     │  │ • Multi-Account     │  │   │
│  │  │ • WebGL FP      │  │ • Velocity Score│  │ • Registration Flow │  │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
│  │                                                                      │   │
│  │  Integration: Device ID providers, Browser fingerprint libraries     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  VELOCITY & AGGREGATION ENGINE                       │   │
│  │                                                                      │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │   │
│  │  │ Transaction     │  │ Amount          │  │ Entity              │  │   │
│  │  │ Velocity        │  │ Aggregation     │  │ Velocity            │  │   │
│  │  │                 │  │                 │  │                     │  │   │
│  │  │ • Count/Hour    │  │ • Sum/Day       │  │ • Device/Hour       │  │   │
│  │  │ • Count/Day     │  │ • Sum/Week      │  │ • IP/Hour           │  │   │
│  │  │ • Count/Week    │  │ • Sum/Month     │  │ • Account/Day       │  │   │
│  │  │ • Beneficiary   │  │ • Avg Amount    │  │ • Card/Day          │  │   │
│  │  │ • Channel Mix   │  │ • Max Amount    │  │ • Email/Day         │  │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
│  │                                                                      │   │
│  │  Storage: Redis Cluster for real-time aggregations with TTL          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  NETWORK ANALYSIS ENGINE                             │   │
│  │                                                                      │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │   │
│  │  │ Entity          │  │ Fraud Ring      │  │ Link                │  │   │
│  │  │ Resolution      │  │ Detection       │  │ Analysis            │  │   │
│  │  │                 │  │                 │  │                     │  │   │
│  │  │ • Name Matching │  │ • Graph Traverse│  │ • Account Links     │  │   │
│  │  │ • Address Match │  │ • Community     │  │ • Beneficiary Links │  │   │
│  │  │ • Device Links  │  │   Detection     │  │ • Device Sharing    │  │   │
│  │  │ • Phone/Email   │  │ • Anomaly Score │  │ • IP Correlation    │  │   │
│  │  │ • Card Links    │  │ • Mule Detection│  │ • Temporal Links    │  │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
│  │                                                                      │   │
│  │  Graph DB: Neo4j for entity relationships and fraud network analysis │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  SUPPORTING COMPONENTS                               │   │
│  │                                                                      │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │   │
│  │  │ Feature     │  │ Model       │  │ Decision    │  │ Alert       │ │   │
│  │  │ Store       │  │ Registry    │  │ Cache       │  │ Generator   │ │   │
│  │  │             │  │             │  │             │  │             │ │   │
│  │  │ • Real-time │  │ • Versioning│  │ • Recent    │  │ • Create    │ │   │
│  │  │ • Historical│  │ • A/B Test  │  │   Decisions │  │ • Priority  │ │   │
│  │  │ • Derived   │  │ • Champion/ │  │ • Whitelist │  │ • Route     │ │   │
│  │  │ • External  │  │   Challenger│  │ • Blocklist │  │ • Escalate  │ │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘ │   │
│  │                                                                      │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │   │
│  │  │ Case        │  │ Feedback    │  │ Metrics     │  │ Config      │ │   │
│  │  │ Connector   │  │ Loop        │  │ Publisher   │  │ Manager     │ │   │
│  │  │             │  │             │  │             │  │             │ │   │
│  │  │ • Create    │  │ • Confirmed │  │ • Counters  │  │ • Rules     │ │   │
│  │  │ • Link      │  │ • False Pos │  │ • Latency   │  │ • Thresholds│ │   │
│  │  │ • Update    │  │ • Retrain   │  │ • Hit Rates │  │ • Models    │ │   │
│  │  │ • Close     │  │ • Labels    │  │ • Dashboard │  │ • Weights   │ │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘ │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 Module Responsibilities

| Responsibility | Description |
|----------------|-------------|
| Real-Time Scoring | Provide instant fraud risk scores for transactions |
| ML Model Management | Deploy, version, and serve fraud detection models |
| Rule Evaluation | Execute configurable fraud detection rules |
| Behavioral Profiling | Build and maintain customer behavior profiles |
| Device Intelligence | Fingerprint devices and assess device risk |
| Velocity Monitoring | Track transaction velocity across multiple dimensions |
| Network Analysis | Detect fraud rings and linked suspicious entities |
| Feature Engineering | Compute and store features for ML models |
| Decision Making | Combine signals to make ALLOW/CHALLENGE/REVIEW/BLOCK decisions |
| Alert Generation | Create alerts for suspicious transactions |
| Case Creation | Integrate with case management for investigations |
| Feedback Processing | Incorporate investigation outcomes for model improvement |
| Audit Logging | Record complete fraud screening history |
| Analytics & Reporting | Provide fraud metrics and trend analysis |

### 2.4 Fraud Detection Flow Design

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FRAUD DETECTION FLOW                                      │
└─────────────────────────────────────────────────────────────────────────────┘

    FraudCheckRequest (from Validation Pipeline / Gateway)
                │
                ▼
    ┌───────────────────────────────────────────────────────────────────┐
    │                     FRAUD ORCHESTRATOR                             │
    │                                                                    │
    │   evaluateTransaction(FraudCheckRequest request)                   │
    │   {                                                                │
    │       // Step 1: Extract & Enrich Context                          │
    │       FraudContext context = contextBuilder.build(request);        │
    │       // - Transaction details (amount, currency, type)           │
    │       // - Customer profile (ID, history, risk tier)              │
    │       // - Device information (fingerprint, IP, geolocation)      │
    │       // - Session data (login time, actions, channel)            │
    │                                                                    │
    │       // Step 2: Feature Computation                               │
    │       FeatureVector features = featureEngine.compute(context);     │
    │       // - Real-time aggregations from Redis                      │
    │       // - Historical features from Feature Store                 │
    │       // - Derived features (ratios, deltas, flags)               │
    │                                                                    │
    │       // Step 3: Parallel Evaluation (fan-out)                     │
    │       CompletableFuture<EvalResult>[] futures = {                 │
    │           mlEngine.scoreAsync(features),                          │
    │           ruleEngine.evaluateAsync(context, features),            │
    │           behavioralEngine.analyzeAsync(context),                 │
    │           deviceEngine.assessAsync(context.device),               │
    │           velocityEngine.checkAsync(context),                     │
    │           networkEngine.analyzeAsync(context.entities)            │
    │       };                                                           │
    │                                                                    │
    │       // Step 4: Aggregate Results                                 │
    │       AggregatedResult results = waitAndAggregate(futures);       │
    │                                                                    │
    │       // Step 5: Calculate Composite Score                         │
    │       FraudScore score = scoreCalculator.compute(results);        │
    │                                                                    │
    │       // Step 6: Make Decision                                     │
    │       FraudDecision decision = decisionEngine.evaluate(           │
    │           results, score, context                                  │
    │       );                                                           │
    │                                                                    │
    │       // Step 7: Post-Decision Actions                             │
    │       if (decision == BLOCK || decision == REVIEW) {              │
    │           alertGenerator.createAlert(context, results, decision); │
    │           caseConnector.createOrUpdateCase(context, results);     │
    │       }                                                            │
    │       if (decision == CHALLENGE) {                                 │
    │           authService.triggerStepUp(context, decision.authMethod);│
    │       }                                                            │
    │                                                                    │
    │       // Step 8: Update Velocities & Audit                         │
    │       velocityEngine.updateCounters(context, decision);           │
    │       auditLogger.log(context, results, decision);                │
    │       metricsPublisher.recordDecision(context, decision);         │
    │                                                                    │
    │       return buildResponse(decision, score, results);             │
    │   }                                                                │
    │                                                                    │
    └───────────────────────────────────────────────────────────────────┘
                │
                ▼
        FraudCheckResponse (to Payment Orchestration / Auth Service)
        - Decision: ALLOW | CHALLENGE | REVIEW | BLOCK
        - Fraud Score: 0-1000
        - Risk Level: LOW | MEDIUM | HIGH | CRITICAL
        - Triggered Rules: List of matched rules
        - Challenge Method: If step-up required (OTP, Biometric, etc.)
        - Alert Reference: If alert created
```

### 2.5 Fraud Decision Matrix

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FRAUD DECISION MATRIX                                     │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────┬─────────────────────────────────────────────────────────┐
│ Signal Type         │ Score Range  → Decision                                 │
├─────────────────────┼─────────────────────────────────────────────────────────┤
│                     │ > 900        → BLOCK (Very high fraud probability)     │
│ ML MODEL SCORE      │ 800-900      → REVIEW (High fraud probability)         │
│ (0-1000 scale)      │ 600-799      → CHALLENGE (Moderate risk, verify)       │
│                     │ < 600        → ALLOW (Low fraud probability)           │
├─────────────────────┼─────────────────────────────────────────────────────────┤
│                     │ BLOCK rule   → BLOCK (Hard rule match)                 │
│ RULE ENGINE         │ REVIEW rule  → REVIEW (Requires manual review)         │
│                     │ CHALLENGE    → CHALLENGE (Step-up authentication)      │
│                     │ FLAG rule    → ALLOW + Flag (Enhanced monitoring)      │
├─────────────────────┼─────────────────────────────────────────────────────────┤
│                     │ > 3σ         → REVIEW (Extreme deviation)              │
│ BEHAVIORAL          │ 2σ - 3σ      → CHALLENGE (Significant deviation)       │
│ DEVIATION           │ 1σ - 2σ      → FLAG (Notable deviation)                │
│                     │ < 1σ         → ALLOW (Within normal behavior)          │
├─────────────────────┼─────────────────────────────────────────────────────────┤
│                     │ Unknown + High Risk → CHALLENGE (New device)           │
│ DEVICE              │ Known + Low Risk    → ALLOW (Trusted device)           │
│ ASSESSMENT          │ Blacklisted         → BLOCK (Fraud-linked device)      │
│                     │ Suspicious          → REVIEW (Anomalous device)        │
├─────────────────────┼─────────────────────────────────────────────────────────┤
│                     │ > Threshold  → BLOCK or REVIEW (Velocity breach)       │
│ VELOCITY            │ > Warning    → CHALLENGE (Approaching limit)           │
│ CHECKS              │ ≤ Threshold  → ALLOW (Within normal velocity)          │
├─────────────────────┼─────────────────────────────────────────────────────────┤
│                     │ Fraud Ring   → BLOCK (Connected to fraud network)      │
│ NETWORK             │ Mule Account → REVIEW (Money mule indicators)          │
│ ANALYSIS            │ Suspicious Links → CHALLENGE (Unusual connections)     │
│                     │ No Issues    → ALLOW (No network concerns)             │
└─────────────────────┴─────────────────────────────────────────────────────────┘

Decision Precedence: BLOCK > REVIEW > CHALLENGE > FLAG > ALLOW

Combined Decision Logic:
┌─────────────────────────────────────────────────────────────────────────────┐
│ IF any_signal == BLOCK                                                       │
│    THEN final_decision = BLOCK                                               │
│    reason = highest_priority_block_reason                                   │
│                                                                              │
│ ELSE IF any_signal == REVIEW                                                 │
│    THEN final_decision = REVIEW                                              │
│    reason = "Requires manual investigation"                                 │
│                                                                              │
│ ELSE IF any_signal == CHALLENGE                                              │
│    THEN final_decision = CHALLENGE                                           │
│    challenge_method = selectAuthMethod(context, risk_factors)               │
│    // Methods: OTP_SMS, OTP_EMAIL, BIOMETRIC, SECURITY_QUESTIONS, CALLBACK  │
│                                                                              │
│ ELSE IF composite_score > configurable_review_threshold                      │
│    THEN final_decision = REVIEW                                              │
│                                                                              │
│ ELSE IF composite_score > configurable_challenge_threshold                   │
│    THEN final_decision = CHALLENGE                                           │
│                                                                              │
│ ELSE                                                                         │
│    final_decision = ALLOW                                                   │
│    apply_any_flags(monitoring_flags)                                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.6 Challenge Authentication Matrix

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  STEP-UP AUTHENTICATION SELECTION                            │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────┬─────────────────────────────────────────────────────────┐
│ Risk Context        │ Authentication Method                                  │
├─────────────────────┼─────────────────────────────────────────────────────────┤
│ New Device + High   │ BIOMETRIC + OTP_SMS (Multi-factor)                     │
│ Amount              │                                                         │
├─────────────────────┼─────────────────────────────────────────────────────────┤
│ New Beneficiary +   │ OTP_SMS + TRANSACTION_CONFIRMATION                     │
│ Large Transfer      │                                                         │
├─────────────────────┼─────────────────────────────────────────────────────────┤
│ Unusual Time +      │ BIOMETRIC (Fast, secure verification)                  │
│ Moderate Amount     │                                                         │
├─────────────────────┼─────────────────────────────────────────────────────────┤
│ Velocity Warning    │ OTP_EMAIL (Additional verification)                    │
├─────────────────────┼─────────────────────────────────────────────────────────┤
│ Geographic Anomaly  │ SECURITY_QUESTIONS + OTP_SMS                           │
├─────────────────────┼─────────────────────────────────────────────────────────┤
│ Session Risk        │ RE_AUTHENTICATE (Password + OTP)                       │
├─────────────────────┼─────────────────────────────────────────────────────────┤
│ High-Value Transfer │ CALLBACK (Phone verification by agent)                 │
│ (> $50,000)         │                                                         │
├─────────────────────┼─────────────────────────────────────────────────────────┤
│ PSD2 SCA Required   │ BIOMETRIC or DEVICE_BINDING_CONFIRM                    │
└─────────────────────┴─────────────────────────────────────────────────────────┘
```

---

## 3. Functional Requirements Summary

### 3.1 ML Model Scoring Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| FRD-ML-001 | System shall score every transaction using ensemble ML models within 50ms | P1 |
| FRD-ML-002 | System shall support multiple concurrent model versions (champion/challenger) | P1 |
| FRD-ML-003 | System shall provide model inference with feature vector input of up to 500 features | P1 |
| FRD-ML-004 | System shall support XGBoost, LightGBM, and neural network model types | P1 |
| FRD-ML-005 | System shall enable A/B testing with configurable traffic split percentages | P2 |
| FRD-ML-006 | System shall provide model performance metrics (AUC, precision, recall) per model version | P1 |
| FRD-ML-007 | System shall support model rollback within 5 minutes | P1 |
| FRD-ML-008 | System shall refresh model predictions when customer profile changes | P2 |
| FRD-ML-009 | System shall support batch scoring for historical analysis | P2 |
| FRD-ML-010 | System shall log all model inputs/outputs for model explainability | P1 |

### 3.2 Rule Engine Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| FRD-RE-001 | System shall evaluate configurable rules against transaction and context data | P1 |
| FRD-RE-002 | System shall support rule conditions using transaction, customer, device, and velocity attributes | P1 |
| FRD-RE-003 | System shall support rule actions: BLOCK, REVIEW, CHALLENGE, FLAG, ALLOW | P1 |
| FRD-RE-004 | System shall execute rules with priority ordering and short-circuit evaluation | P1 |
| FRD-RE-005 | System shall complete rule evaluation within 30ms for up to 500 active rules | P1 |
| FRD-RE-006 | System shall support rule versioning with effective date ranges | P2 |
| FRD-RE-007 | System shall provide rule simulation capability (dry-run mode) | P2 |
| FRD-RE-008 | System shall support compound rules with AND/OR/NOT logic | P1 |
| FRD-RE-009 | System shall provide rule hit statistics and effectiveness metrics | P2 |
| FRD-RE-010 | System shall support rule import/export for environment migration | P3 |

### 3.3 Behavioral Analytics Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| FRD-BA-001 | System shall maintain behavioral profiles for all active customers | P1 |
| FRD-BA-002 | System shall detect deviations from established behavioral patterns | P1 |
| FRD-BA-003 | System shall track transaction amount patterns (mean, std dev, percentiles) | P1 |
| FRD-BA-004 | System shall track transaction timing patterns (hour of day, day of week) | P1 |
| FRD-BA-005 | System shall track channel usage patterns (web, mobile, API) | P1 |
| FRD-BA-006 | System shall track beneficiary patterns (new vs. known recipients) | P1 |
| FRD-BA-007 | System shall track geographic patterns (login locations, payment destinations) | P1 |
| FRD-BA-008 | System shall compute behavioral deviation scores within 40ms | P1 |
| FRD-BA-009 | System shall support peer group comparison for anomaly detection | P2 |
| FRD-BA-010 | System shall age and decay old behavioral data based on configurable windows | P2 |

### 3.4 Device Intelligence Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| FRD-DI-001 | System shall fingerprint devices using browser and hardware attributes | P1 |
| FRD-DI-002 | System shall maintain device reputation scores based on historical behavior | P1 |
| FRD-DI-003 | System shall detect device spoofing and emulation attempts | P1 |
| FRD-DI-004 | System shall track device-to-account bindings and detect anomalies | P1 |
| FRD-DI-005 | System shall detect VPN, proxy, and Tor usage | P1 |
| FRD-DI-006 | System shall assess device age and first-seen timestamps | P2 |
| FRD-DI-007 | System shall detect jailbroken/rooted mobile devices | P2 |
| FRD-DI-008 | System shall support device blacklisting and whitelisting | P1 |
| FRD-DI-009 | System shall provide device risk assessment within 25ms | P1 |
| FRD-DI-010 | System shall integrate with third-party device intelligence providers | P2 |

### 3.5 Velocity Control Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| FRD-VC-001 | System shall track transaction counts per entity (account, card, device, IP) | P1 |
| FRD-VC-002 | System shall track transaction amounts per entity over configurable time windows | P1 |
| FRD-VC-003 | System shall support velocity windows: 1min, 5min, 1hour, 1day, 7day, 30day | P1 |
| FRD-VC-004 | System shall update velocity counters in real-time with sub-second latency | P1 |
| FRD-VC-005 | System shall support configurable velocity thresholds per customer segment | P1 |
| FRD-VC-006 | System shall detect rapid successive transactions (burst detection) | P1 |
| FRD-VC-007 | System shall track unique beneficiary counts per time window | P2 |
| FRD-VC-008 | System shall support velocity checks across multiple payment types | P1 |
| FRD-VC-009 | System shall provide velocity breach alerts with threshold details | P2 |
| FRD-VC-010 | System shall handle velocity counter failures gracefully (fail-open/fail-close) | P1 |

### 3.6 Network Analysis Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| FRD-NA-001 | System shall maintain entity relationship graphs (account, device, beneficiary links) | P1 |
| FRD-NA-002 | System shall detect fraud rings using graph traversal algorithms | P1 |
| FRD-NA-003 | System shall identify accounts sharing devices, IPs, or beneficiaries | P1 |
| FRD-NA-004 | System shall detect money mule account patterns | P1 |
| FRD-NA-005 | System shall compute network risk scores based on connected entity risk | P2 |
| FRD-NA-006 | System shall support real-time graph updates as new transactions occur | P2 |
| FRD-NA-007 | System shall detect rapid fund movement through account chains | P2 |
| FRD-NA-008 | System shall provide fraud network visualization for investigators | P3 |
| FRD-NA-009 | System shall score new entities based on their network connections | P2 |
| FRD-NA-010 | System shall integrate network scores into overall fraud decision | P1 |

### 3.7 Decision & Alert Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| FRD-DA-001 | System shall make fraud decisions (ALLOW/CHALLENGE/REVIEW/BLOCK) for all transactions | P1 |
| FRD-DA-002 | System shall provide composite fraud scores combining all signals | P1 |
| FRD-DA-003 | System shall support configurable decision thresholds per transaction type | P1 |
| FRD-DA-004 | System shall generate alerts for REVIEW and BLOCK decisions | P1 |
| FRD-DA-005 | System shall route alerts to appropriate queues based on fraud type and amount | P2 |
| FRD-DA-006 | System shall provide decision explanations with contributing factors | P1 |
| FRD-DA-007 | System shall support override capabilities for authorized users | P2 |
| FRD-DA-008 | System shall maintain decision audit trail with timestamps | P1 |
| FRD-DA-009 | System shall integrate with case management for investigation workflow | P1 |
| FRD-DA-010 | System shall support real-time decision statistics and monitoring | P1 |

### 3.8 Step-Up Authentication Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| FRD-SU-001 | System shall trigger step-up authentication for CHALLENGE decisions | P1 |
| FRD-SU-002 | System shall support multiple authentication methods (OTP, biometric, security questions) | P1 |
| FRD-SU-003 | System shall select authentication method based on risk context and availability | P1 |
| FRD-SU-004 | System shall track step-up challenge outcomes (success, failure, timeout) | P1 |
| FRD-SU-005 | System shall escalate to BLOCK on repeated authentication failures | P1 |
| FRD-SU-006 | System shall support PSD2 SCA (Strong Customer Authentication) requirements | P1 |
| FRD-SU-007 | System shall provide configurable challenge timeout periods | P2 |
| FRD-SU-008 | System shall support fallback authentication methods | P2 |
| FRD-SU-009 | System shall exempt low-risk transactions from SCA per regulatory allowances | P2 |
| FRD-SU-010 | System shall log all authentication challenges and outcomes | P1 |

### 3.9 Feedback & Learning Requirements

| Req ID | Requirement | Priority |
|--------|-------------|----------|
| FRD-FL-001 | System shall accept feedback on fraud decisions (confirmed fraud, false positive) | P1 |
| FRD-FL-002 | System shall maintain labeled datasets for model training | P1 |
| FRD-FL-003 | System shall automatically update customer risk profiles based on feedback | P2 |
| FRD-FL-004 | System shall track model performance drift over time | P2 |
| FRD-FL-005 | System shall support scheduled model retraining pipelines | P2 |
| FRD-FL-006 | System shall flag potential training data quality issues | P3 |
| FRD-FL-007 | System shall provide feedback submission API for case management | P1 |
| FRD-FL-008 | System shall support feedback categories (fraud confirmed, customer dispute, etc.) | P2 |
| FRD-FL-009 | System shall generate model retraining recommendations based on feedback volume | P3 |
| FRD-FL-010 | System shall track feedback processing latency and completion rates | P2 |

---

## 4. Domain Model

### 4.1 Core Aggregates

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FRAUD DETECTION DOMAIN MODEL                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                         FRAUD CHECK AGGREGATE                                │
│                                                                             │
│  FraudCheck (Aggregate Root)                                                │
│  ├── checkId: UUID                                                         │
│  ├── transactionId: UUID                                                   │
│  ├── checkType: CheckType (REAL_TIME, BATCH, RECHECK)                     │
│  ├── requestTimestamp: Instant                                             │
│  ├── responseTimestamp: Instant                                            │
│  ├── latencyMs: Integer                                                    │
│  │                                                                          │
│  ├── FraudContext (Value Object)                                           │
│  │   ├── transaction: TransactionDetails                                   │
│  │   ├── customer: CustomerProfile                                         │
│  │   ├── device: DeviceInfo                                                │
│  │   ├── session: SessionContext                                           │
│  │   └── channel: ChannelInfo                                              │
│  │                                                                          │
│  ├── FeatureVector (Value Object)                                          │
│  │   ├── featureVersion: String                                            │
│  │   ├── transactionFeatures: Map<String, Double>                         │
│  │   ├── customerFeatures: Map<String, Double>                            │
│  │   ├── deviceFeatures: Map<String, Double>                              │
│  │   ├── velocityFeatures: Map<String, Double>                            │
│  │   └── derivedFeatures: Map<String, Double>                             │
│  │                                                                          │
│  ├── EvaluationResults (Value Object)                                      │
│  │   ├── mlResult: MLModelResult                                           │
│  │   ├── ruleResult: RuleEngineResult                                      │
│  │   ├── behavioralResult: BehavioralResult                                │
│  │   ├── deviceResult: DeviceAssessmentResult                              │
│  │   ├── velocityResult: VelocityCheckResult                               │
│  │   └── networkResult: NetworkAnalysisResult                              │
│  │                                                                          │
│  ├── FraudDecision (Value Object)                                          │
│  │   ├── decision: DecisionType (ALLOW, CHALLENGE, REVIEW, BLOCK)         │
│  │   ├── fraudScore: Integer (0-1000)                                      │
│  │   ├── riskLevel: RiskLevel (LOW, MEDIUM, HIGH, CRITICAL)               │
│  │   ├── primaryReason: String                                             │
│  │   ├── reasonCodes: List<ReasonCode>                                     │
│  │   ├── triggeredRules: List<RuleMatch>                                   │
│  │   └── challengeMethod: ChallengeMethod (if CHALLENGE)                   │
│  │                                                                          │
│  ├── alertId: UUID (nullable, if alert created)                            │
│  ├── caseId: UUID (nullable, if case created)                              │
│  └── auditTrail: List<AuditEntry>                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                      CUSTOMER PROFILE AGGREGATE                              │
│                                                                             │
│  CustomerProfile (Aggregate Root)                                           │
│  ├── customerId: UUID                                                      │
│  ├── customerNumber: String                                                │
│  ├── profileVersion: Long                                                  │
│  ├── riskTier: RiskTier (LOW, MEDIUM, HIGH, VIP, RESTRICTED)              │
│  ├── onboardingDate: LocalDate                                             │
│  ├── lastActivityDate: LocalDate                                           │
│  │                                                                          │
│  ├── BehavioralProfile (Entity)                                            │
│  │   ├── profileId: UUID                                                   │
│  │   ├── lastUpdated: Instant                                              │
│  │   ├── transactionPattern: TransactionPatternStats                       │
│  │   │   ├── avgAmount: BigDecimal                                         │
│  │   │   ├── stdDevAmount: BigDecimal                                      │
│  │   │   ├── maxAmount: BigDecimal                                         │
│  │   │   ├── p50Amount: BigDecimal                                         │
│  │   │   ├── p95Amount: BigDecimal                                         │
│  │   │   └── transactionCount: Long                                        │
│  │   ├── timingPattern: TimingPatternStats                                 │
│  │   │   ├── preferredHours: List<Integer>                                 │
│  │   │   ├── preferredDays: List<DayOfWeek>                               │
│  │   │   └── avgTimeBetweenTx: Duration                                   │
│  │   ├── channelPattern: ChannelPatternStats                               │
│  │   │   ├── channelDistribution: Map<Channel, Double>                    │
│  │   │   └── preferredChannel: Channel                                     │
│  │   ├── beneficiaryPattern: BeneficiaryPatternStats                       │
│  │   │   ├── knownBeneficiaries: Set<String>                              │
│  │   │   ├── avgBeneficiariesPerMonth: Double                             │
│  │   │   └── recurringPayments: List<RecurringPattern>                    │
│  │   └── geographicPattern: GeographicPatternStats                         │
│  │       ├── homeCountry: String                                           │
│  │       ├── frequentCountries: Set<String>                               │
│  │       └── unusualCountries: Set<String>                                │
│  │                                                                          │
│  ├── DeviceBindings (Entity)                                               │
│  │   ├── trustedDevices: List<TrustedDevice>                              │
│  │   │   ├── deviceId: String                                              │
│  │   │   ├── deviceFingerprint: String                                     │
│  │   │   ├── firstSeen: Instant                                            │
│  │   │   ├── lastSeen: Instant                                             │
│  │   │   ├── trustScore: Integer                                           │
│  │   │   └── status: DeviceStatus (ACTIVE, SUSPENDED, REVOKED)            │
│  │   └── deviceCount: Integer                                              │
│  │                                                                          │
│  ├── fraudHistory: FraudHistory                                            │
│  │   ├── confirmedFraudCount: Integer                                      │
│  │   ├── falsePositiveCount: Integer                                       │
│  │   ├── chargebackCount: Integer                                          │
│  │   ├── suspiciousActivityCount: Integer                                  │
│  │   └── lastIncidentDate: LocalDate                                       │
│  │                                                                          │
│  └── restrictions: List<Restriction>                                       │
│      ├── restrictionType: RestrictionType                                  │
│      ├── effectiveFrom: Instant                                            │
│      ├── effectiveUntil: Instant                                           │
│      └── reason: String                                                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                         DEVICE AGGREGATE                                     │
│                                                                             │
│  DeviceRecord (Aggregate Root)                                              │
│  ├── deviceId: String (hash of fingerprint)                                │
│  ├── fingerprint: DeviceFingerprint                                        │
│  │   ├── browserHash: String                                               │
│  │   ├── canvasHash: String                                                │
│  │   ├── webGLHash: String                                                 │
│  │   ├── audioHash: String                                                 │
│  │   ├── screenResolution: String                                          │
│  │   ├── timezone: String                                                  │
│  │   ├── language: String                                                  │
│  │   ├── platform: String                                                  │
│  │   └── plugins: List<String>                                             │
│  │                                                                          │
│  ├── deviceType: DeviceType (DESKTOP, MOBILE, TABLET, UNKNOWN)            │
│  ├── operatingSystem: String                                               │
│  ├── browser: String                                                       │
│  ├── firstSeen: Instant                                                    │
│  ├── lastSeen: Instant                                                     │
│  │                                                                          │
│  ├── reputation: DeviceReputation                                          │
│  │   ├── trustScore: Integer (0-100)                                       │
│  │   ├── riskLevel: RiskLevel                                              │
│  │   ├── fraudIndicators: List<FraudIndicator>                            │
│  │   ├── accountCount: Integer                                             │
│  │   ├── transactionCount: Long                                            │
│  │   └── lastRiskAssessment: Instant                                       │
│  │                                                                          │
│  ├── linkedAccounts: List<LinkedAccount>                                   │
│  │   ├── customerId: UUID                                                  │
│  │   ├── firstLinked: Instant                                              │
│  │   └── lastUsed: Instant                                                 │
│  │                                                                          │
│  ├── ipHistory: List<IPRecord>                                             │
│  │   ├── ipAddress: String                                                 │
│  │   ├── geolocation: GeoLocation                                          │
│  │   ├── isVPN: Boolean                                                    │
│  │   ├── isProxy: Boolean                                                  │
│  │   ├── isTor: Boolean                                                    │
│  │   └── seenAt: Instant                                                   │
│  │                                                                          │
│  └── status: DeviceStatus (TRUSTED, UNKNOWN, SUSPICIOUS, BLOCKED)         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                      FRAUD RULE AGGREGATE                                    │
│                                                                             │
│  FraudRule (Aggregate Root)                                                 │
│  ├── ruleId: UUID                                                          │
│  ├── ruleCode: String (unique identifier)                                  │
│  ├── ruleName: String                                                      │
│  ├── description: String                                                   │
│  ├── category: RuleCategory (VELOCITY, AMOUNT, BEHAVIOR, DEVICE, NETWORK) │
│  ├── priority: Integer (1-1000, lower = higher priority)                  │
│  │                                                                          │
│  ├── conditions: List<RuleCondition>                                       │
│  │   ├── field: String (e.g., "transaction.amount")                       │
│  │   ├── operator: Operator (EQ, NE, GT, LT, GTE, LTE, IN, CONTAINS, etc.)│
│  │   ├── value: Object                                                     │
│  │   └── logicalOperator: LogicalOp (AND, OR)                             │
│  │                                                                          │
│  ├── action: RuleAction                                                    │
│  │   ├── actionType: ActionType (BLOCK, REVIEW, CHALLENGE, FLAG, ALLOW)   │
│  │   ├── reasonCode: String                                                │
│  │   ├── message: String                                                   │
│  │   └── challengeMethod: ChallengeMethod (if CHALLENGE)                   │
│  │                                                                          │
│  ├── effectiveFrom: Instant                                                │
│  ├── effectiveUntil: Instant (nullable for no expiry)                     │
│  ├── status: RuleStatus (ACTIVE, INACTIVE, TESTING)                       │
│  ├── version: Integer                                                      │
│  │                                                                          │
│  └── statistics: RuleStatistics                                            │
│      ├── hitCount: Long                                                    │
│      ├── lastHit: Instant                                                  │
│      ├── truePositiveCount: Long                                           │
│      └── falsePositiveCount: Long                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                      ML MODEL AGGREGATE                                      │
│                                                                             │
│  MLModel (Aggregate Root)                                                   │
│  ├── modelId: UUID                                                         │
│  ├── modelName: String                                                     │
│  ├── modelType: ModelType (XGBOOST, LIGHTGBM, NEURAL_NET, ENSEMBLE)       │
│  ├── version: String (semantic versioning)                                 │
│  ├── description: String                                                   │
│  │                                                                          │
│  ├── artifact: ModelArtifact                                               │
│  │   ├── artifactPath: String (S3/GCS path)                               │
│  │   ├── artifactHash: String (SHA-256)                                   │
│  │   ├── artifactSize: Long                                                │
│  │   └── framework: String (e.g., "sklearn", "pytorch")                   │
│  │                                                                          │
│  ├── featureSpec: FeatureSpecification                                     │
│  │   ├── featureCount: Integer                                             │
│  │   ├── featureNames: List<String>                                       │
│  │   ├── featureTypes: Map<String, DataType>                              │
│  │   └── featureStats: Map<String, FeatureStats>                          │
│  │                                                                          │
│  ├── performance: ModelPerformance                                         │
│  │   ├── auc: Double                                                       │
│  │   ├── precision: Double                                                 │
│  │   ├── recall: Double                                                    │
│  │   ├── f1Score: Double                                                   │
│  │   ├── falsePositiveRate: Double                                        │
│  │   └── evaluatedAt: Instant                                              │
│  │                                                                          │
│  ├── deployment: DeploymentConfig                                          │
│  │   ├── status: DeploymentStatus (TRAINING, STAGING, CHAMPION, RETIRED)  │
│  │   ├── trafficPercentage: Integer (0-100 for A/B testing)              │
│  │   ├── deployedAt: Instant                                               │
│  │   └── deployedBy: String                                                │
│  │                                                                          │
│  ├── trainingMetadata: TrainingMetadata                                    │
│  │   ├── trainedAt: Instant                                                │
│  │   ├── trainingDataStart: LocalDate                                     │
│  │   ├── trainingDataEnd: LocalDate                                       │
│  │   ├── sampleCount: Long                                                 │
│  │   ├── fraudRatio: Double                                                │
│  │   └── hyperparameters: Map<String, Object>                             │
│  │                                                                          │
│  └── monitoringConfig: MonitoringConfig                                    │
│      ├── driftThreshold: Double                                            │
│      ├── performanceThreshold: Double                                      │
│      └── alertRecipients: List<String>                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                       FRAUD ALERT AGGREGATE                                  │
│                                                                             │
│  FraudAlert (Aggregate Root)                                                │
│  ├── alertId: UUID                                                         │
│  ├── alertNumber: String (human-readable ID)                               │
│  ├── transactionId: UUID                                                   │
│  ├── fraudCheckId: UUID                                                    │
│  ├── customerId: UUID                                                      │
│  │                                                                          │
│  ├── alertType: AlertType (BLOCK, REVIEW, PATTERN, NETWORK, VELOCITY)     │
│  ├── severity: Severity (LOW, MEDIUM, HIGH, CRITICAL)                     │
│  ├── priority: Integer                                                     │
│  ├── queue: String (assignment queue)                                      │
│  │                                                                          │
│  ├── fraudScore: Integer                                                   │
│  ├── triggeredRules: List<String>                                          │
│  ├── reasonCodes: List<String>                                             │
│  ├── description: String                                                   │
│  │                                                                          │
│  ├── status: AlertStatus (OPEN, ASSIGNED, INVESTIGATING, RESOLVED)        │
│  ├── resolution: AlertResolution (nullable)                                │
│  │   ├── resolutionType: ResolutionType (CONFIRMED_FRAUD, FALSE_POSITIVE, │
│  │   │                    CUSTOMER_DISPUTE, INSUFFICIENT_INFO)             │
│  │   ├── resolvedBy: String                                                │
│  │   ├── resolvedAt: Instant                                               │
│  │   └── notes: String                                                     │
│  │                                                                          │
│  ├── assignee: String (nullable)                                           │
│  ├── createdAt: Instant                                                    │
│  ├── updatedAt: Instant                                                    │
│  ├── slaDeadline: Instant                                                  │
│  │                                                                          │
│  └── linkedAlerts: List<UUID>  (related alerts)                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Value Objects

```java
// Decision Types
public enum DecisionType {
    ALLOW,      // Transaction approved
    CHALLENGE,  // Step-up authentication required
    REVIEW,     // Manual review required
    BLOCK       // Transaction rejected
}

// Risk Levels
public enum RiskLevel {
    LOW(0, 300),
    MEDIUM(301, 600),
    HIGH(601, 850),
    CRITICAL(851, 1000);
    
    private final int minScore;
    private final int maxScore;
}

// Challenge Methods
public enum ChallengeMethod {
    OTP_SMS,
    OTP_EMAIL,
    BIOMETRIC_FINGERPRINT,
    BIOMETRIC_FACE,
    SECURITY_QUESTIONS,
    DEVICE_BINDING_CONFIRM,
    CALLBACK,
    RE_AUTHENTICATE,
    PUSH_NOTIFICATION
}

// Rule Categories
public enum RuleCategory {
    VELOCITY,       // Transaction frequency rules
    AMOUNT,         // Transaction amount rules
    BEHAVIOR,       // Behavioral deviation rules
    DEVICE,         // Device-related rules
    NETWORK,        // Network/relationship rules
    GEOGRAPHIC,     // Location-based rules
    TIME,           // Time-based rules
    IDENTITY,       // Identity verification rules
    BLOCKLIST,      // Blocklist matching rules
    WHITELIST       // Whitelist/trust rules
}

// Reason Codes
@Value
public class ReasonCode {
    String code;          // e.g., "FRD-VEL-001"
    String category;      // e.g., "VELOCITY"
    String description;   // e.g., "Transaction count exceeded daily limit"
    int contribution;     // Score contribution (0-100)
}

// Transaction Details
@Value
public class TransactionDetails {
    UUID transactionId;
    String transactionType;     // PAYMENT, TRANSFER, WITHDRAWAL
    BigDecimal amount;
    String currency;
    String originatorAccount;
    String beneficiaryAccount;
    String beneficiaryName;
    String beneficiaryBank;
    String beneficiaryCountry;
    String paymentPurpose;
    Instant initiatedAt;
}

// Session Context
@Value
public class SessionContext {
    String sessionId;
    Instant sessionStart;
    String loginMethod;         // PASSWORD, BIOMETRIC, SSO
    int actionsInSession;
    GeoLocation loginLocation;
    boolean isNewSession;
}

// GeoLocation
@Value
public class GeoLocation {
    String ipAddress;
    String country;
    String region;
    String city;
    double latitude;
    double longitude;
    String isp;
    boolean isVPN;
    boolean isProxy;
    boolean isTor;
    boolean isDataCenter;
}
```

### 4.3 Domain Events

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       FRAUD DETECTION DOMAIN EVENTS                          │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ FraudCheckCompleted                                                          │
│ ├── eventId: UUID                                                           │
│ ├── timestamp: Instant                                                      │
│ ├── checkId: UUID                                                           │
│ ├── transactionId: UUID                                                     │
│ ├── customerId: UUID                                                        │
│ ├── decision: DecisionType                                                  │
│ ├── fraudScore: Integer                                                     │
│ ├── riskLevel: RiskLevel                                                    │
│ ├── reasonCodes: List<String>                                               │
│ ├── triggeredRules: List<String>                                            │
│ ├── latencyMs: Long                                                         │
│ └── challengeMethod: ChallengeMethod (if CHALLENGE)                         │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ FraudAlertCreated                                                            │
│ ├── eventId: UUID                                                           │
│ ├── timestamp: Instant                                                      │
│ ├── alertId: UUID                                                           │
│ ├── alertNumber: String                                                     │
│ ├── transactionId: UUID                                                     │
│ ├── customerId: UUID                                                        │
│ ├── alertType: AlertType                                                    │
│ ├── severity: Severity                                                      │
│ ├── queue: String                                                           │
│ └── slaDeadline: Instant                                                    │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ FraudAlertResolved                                                           │
│ ├── eventId: UUID                                                           │
│ ├── timestamp: Instant                                                      │
│ ├── alertId: UUID                                                           │
│ ├── resolutionType: ResolutionType                                          │
│ ├── resolvedBy: String                                                      │
│ ├── notes: String                                                           │
│ └── resolutionTimeMinutes: Long                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ CustomerRiskProfileUpdated                                                   │
│ ├── eventId: UUID                                                           │
│ ├── timestamp: Instant                                                      │
│ ├── customerId: UUID                                                        │
│ ├── previousRiskTier: RiskTier                                              │
│ ├── newRiskTier: RiskTier                                                   │
│ ├── reason: String                                                          │
│ └── triggeredBy: String (FEEDBACK, SYSTEM, MANUAL)                          │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ DeviceBlacklisted                                                            │
│ ├── eventId: UUID                                                           │
│ ├── timestamp: Instant                                                      │
│ ├── deviceId: String                                                        │
│ ├── reason: String                                                          │
│ ├── linkedAccounts: List<UUID>                                              │
│ └── blacklistedBy: String                                                   │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ MLModelDeployed                                                              │
│ ├── eventId: UUID                                                           │
│ ├── timestamp: Instant                                                      │
│ ├── modelId: UUID                                                           │
│ ├── modelName: String                                                       │
│ ├── version: String                                                         │
│ ├── deploymentStatus: DeploymentStatus                                      │
│ ├── trafficPercentage: Integer                                              │
│ └── deployedBy: String                                                      │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ FraudRuleTriggered                                                           │
│ ├── eventId: UUID                                                           │
│ ├── timestamp: Instant                                                      │
│ ├── ruleId: UUID                                                            │
│ ├── ruleCode: String                                                        │
│ ├── transactionId: UUID                                                     │
│ ├── customerId: UUID                                                        │
│ ├── actionTaken: ActionType                                                 │
│ └── matchedConditions: Map<String, Object>                                  │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ VelocityBreachDetected                                                       │
│ ├── eventId: UUID                                                           │
│ ├── timestamp: Instant                                                      │
│ ├── entityType: String (ACCOUNT, DEVICE, IP, CARD)                         │
│ ├── entityId: String                                                        │
│ ├── velocityType: String (COUNT, AMOUNT)                                   │
│ ├── window: String (1H, 1D, 7D)                                            │
│ ├── threshold: Number                                                       │
│ ├── currentValue: Number                                                    │
│ └── transactionId: UUID                                                     │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ FraudNetworkDetected                                                         │
│ ├── eventId: UUID                                                           │
│ ├── timestamp: Instant                                                      │
│ ├── networkId: UUID                                                         │
│ ├── nodeCount: Integer                                                      │
│ ├── linkedAccounts: List<UUID>                                              │
│ ├── sharedDevices: List<String>                                             │
│ ├── riskScore: Integer                                                      │
│ └── detectionMethod: String                                                 │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ StepUpChallengeCompleted                                                     │
│ ├── eventId: UUID                                                           │
│ ├── timestamp: Instant                                                      │
│ ├── transactionId: UUID                                                     │
│ ├── customerId: UUID                                                        │
│ ├── challengeMethod: ChallengeMethod                                        │
│ ├── result: ChallengeResult (SUCCESS, FAILED, TIMEOUT, CANCELLED)          │
│ ├── attemptCount: Integer                                                   │
│ └── completionTimeMs: Long                                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Data Architecture

### 5.1 Data Store Strategy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FRAUD DETECTION DATA ARCHITECTURE                         │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                         PRIMARY DATABASES                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ PostgreSQL (fraud_db)                                                │   │
│  │                                                                      │   │
│  │ Purpose: Primary transactional data store for fraud checks          │   │
│  │                                                                      │   │
│  │ Tables:                                                              │   │
│  │ • fraud_checks          - Fraud check requests and decisions        │   │
│  │ • fraud_alerts          - Generated fraud alerts                    │   │
│  │ • fraud_rules           - Rule definitions and versions             │   │
│  │ • ml_models             - Model registry and metadata               │   │
│  │ • customer_profiles     - Customer risk profiles                    │   │
│  │ • behavioral_profiles   - Behavioral statistics                     │   │
│  │ • device_records        - Device fingerprints and reputation        │   │
│  │ • decision_feedback     - Investigation outcomes                    │   │
│  │ • audit_log             - Compliance audit trail                    │   │
│  │                                                                      │   │
│  │ Config: Primary-Replica, synchronous replication                    │   │
│  │ Capacity: Hot (90 days), Warm (1 year), Cold (7 years)             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Redis Cluster (fraud_cache)                                          │   │
│  │                                                                      │   │
│  │ Purpose: Real-time velocity counters, caching, feature store        │   │
│  │                                                                      │   │
│  │ Data Structures:                                                     │   │
│  │ • velocity:{entity}:{window}      - HyperLogLog/Counter             │   │
│  │ • profile:{customerId}            - Hash (cached profile)           │   │
│  │ • device:{deviceId}               - Hash (device reputation)        │   │
│  │ • blocklist:device                - Set (blocked devices)           │   │
│  │ • blocklist:ip                    - Set (blocked IPs)               │   │
│  │ • whitelist:customer              - Set (trusted customers)         │   │
│  │ • features:{transactionId}        - Hash (computed features)        │   │
│  │ • decision:{transactionId}        - String (recent decision)        │   │
│  │                                                                      │   │
│  │ Config: 6-node cluster, 3 masters + 3 replicas                      │   │
│  │ TTL: Configurable per key type (1h - 30d)                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Neo4j (fraud_graph)                                                  │   │
│  │                                                                      │   │
│  │ Purpose: Entity relationship graph for network analysis             │   │
│  │                                                                      │   │
│  │ Node Types:                                                          │   │
│  │ • (:Customer)           - Customer accounts                         │   │
│  │ • (:Device)             - Device fingerprints                       │   │
│  │ • (:IP)                 - IP addresses                              │   │
│  │ • (:Beneficiary)        - Payment beneficiaries                     │   │
│  │ • (:Card)               - Payment cards                             │   │
│  │ • (:Email)              - Email addresses                           │   │
│  │ • (:Phone)              - Phone numbers                             │   │
│  │ • (:FraudCase)          - Confirmed fraud cases                     │   │
│  │                                                                      │   │
│  │ Relationships:                                                       │   │
│  │ • [:USES_DEVICE]        - Customer → Device                         │   │
│  │ • [:LOGGED_FROM]        - Customer → IP                             │   │
│  │ • [:PAYS_TO]            - Customer → Beneficiary                    │   │
│  │ • [:HAS_CARD]           - Customer → Card                           │   │
│  │ • [:SHARES_DEVICE]      - Customer → Customer                       │   │
│  │ • [:LINKED_TO]          - Entity → Entity                           │   │
│  │ • [:CONFIRMED_FRAUD]    - Customer → FraudCase                      │   │
│  │                                                                      │   │
│  │ Config: Causal Cluster with 3 cores                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Elasticsearch (fraud_analytics)                                      │   │
│  │                                                                      │   │
│  │ Purpose: Full-text search, analytics, and dashboards                │   │
│  │                                                                      │   │
│  │ Indices:                                                             │   │
│  │ • fraud-checks-*        - Fraud check documents (daily indices)    │   │
│  │ • fraud-alerts-*        - Alert documents                           │   │
│  │ • fraud-metrics-*       - Aggregated metrics                        │   │
│  │ • model-predictions-*   - ML prediction logs                        │   │
│  │ • rule-hits-*           - Rule trigger logs                         │   │
│  │                                                                      │   │
│  │ Config: 3 master + 6 data nodes, hot-warm-cold tiers                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ AWS S3 / Azure Blob (fraud_artifacts)                                │   │
│  │                                                                      │   │
│  │ Purpose: ML model artifacts and large file storage                  │   │
│  │                                                                      │   │
│  │ Buckets:                                                             │   │
│  │ • fraud-models/         - Serialized ML models                       │   │
│  │ • training-data/        - Training datasets                          │   │
│  │ • feature-snapshots/    - Historical feature data                    │   │
│  │ • audit-archives/       - Archived audit logs                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Schema Definitions

#### 5.2.1 Fraud Checks Table

```sql
CREATE TABLE fraud_checks (
    check_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    check_type VARCHAR(20) NOT NULL,  -- REAL_TIME, BATCH, RECHECK
    
    -- Request Context
    request_payload JSONB NOT NULL,
    transaction_amount DECIMAL(18,4) NOT NULL,
    transaction_currency VARCHAR(3) NOT NULL,
    transaction_type VARCHAR(50) NOT NULL,
    channel VARCHAR(50) NOT NULL,
    
    -- Feature Data
    feature_vector JSONB,
    feature_version VARCHAR(20),
    
    -- Evaluation Results
    ml_score INTEGER,
    ml_model_id UUID,
    ml_model_version VARCHAR(20),
    rule_results JSONB,
    behavioral_score DECIMAL(5,2),
    device_score INTEGER,
    velocity_results JSONB,
    network_score INTEGER,
    
    -- Decision
    decision VARCHAR(20) NOT NULL,  -- ALLOW, CHALLENGE, REVIEW, BLOCK
    fraud_score INTEGER NOT NULL,
    risk_level VARCHAR(20) NOT NULL,  -- LOW, MEDIUM, HIGH, CRITICAL
    reason_codes TEXT[],
    triggered_rules TEXT[],
    challenge_method VARCHAR(50),
    
    -- Alert/Case Reference
    alert_id UUID,
    case_id UUID,
    
    -- Timestamps
    request_timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    response_timestamp TIMESTAMPTZ,
    latency_ms INTEGER,
    
    -- Audit
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_fraud_checks_transaction ON fraud_checks(transaction_id);
CREATE INDEX idx_fraud_checks_customer ON fraud_checks(customer_id);
CREATE INDEX idx_fraud_checks_decision ON fraud_checks(decision);
CREATE INDEX idx_fraud_checks_timestamp ON fraud_checks(request_timestamp);
CREATE INDEX idx_fraud_checks_score ON fraud_checks(fraud_score);
CREATE INDEX idx_fraud_checks_alert ON fraud_checks(alert_id) WHERE alert_id IS NOT NULL;

-- Partitioning by month
CREATE TABLE fraud_checks_y2026m02 PARTITION OF fraud_checks
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
```

#### 5.2.2 Customer Profiles Table

```sql
CREATE TABLE customer_profiles (
    customer_id UUID PRIMARY KEY,
    customer_number VARCHAR(50) NOT NULL UNIQUE,
    profile_version BIGINT NOT NULL DEFAULT 1,
    risk_tier VARCHAR(20) NOT NULL DEFAULT 'MEDIUM',  -- LOW, MEDIUM, HIGH, VIP, RESTRICTED
    
    -- Basic Info
    onboarding_date DATE NOT NULL,
    last_activity_date DATE,
    account_status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    
    -- Behavioral Profile (JSON for flexibility)
    transaction_pattern JSONB,
    timing_pattern JSONB,
    channel_pattern JSONB,
    beneficiary_pattern JSONB,
    geographic_pattern JSONB,
    
    -- Fraud History
    confirmed_fraud_count INTEGER NOT NULL DEFAULT 0,
    false_positive_count INTEGER NOT NULL DEFAULT 0,
    chargeback_count INTEGER NOT NULL DEFAULT 0,
    suspicious_activity_count INTEGER NOT NULL DEFAULT 0,
    last_incident_date DATE,
    
    -- Restrictions
    restrictions JSONB,
    
    -- Timestamps
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    CONSTRAINT chk_risk_tier CHECK (risk_tier IN ('LOW', 'MEDIUM', 'HIGH', 'VIP', 'RESTRICTED'))
);

-- Indexes
CREATE INDEX idx_customer_profiles_risk ON customer_profiles(risk_tier);
CREATE INDEX idx_customer_profiles_fraud ON customer_profiles(confirmed_fraud_count) WHERE confirmed_fraud_count > 0;
```

#### 5.2.3 Device Records Table

```sql
CREATE TABLE device_records (
    device_id VARCHAR(64) PRIMARY KEY,  -- SHA-256 hash
    fingerprint JSONB NOT NULL,
    
    -- Device Info
    device_type VARCHAR(20) NOT NULL,  -- DESKTOP, MOBILE, TABLET, UNKNOWN
    operating_system VARCHAR(100),
    browser VARCHAR(100),
    
    -- Reputation
    trust_score INTEGER NOT NULL DEFAULT 50,  -- 0-100
    risk_level VARCHAR(20) NOT NULL DEFAULT 'UNKNOWN',
    fraud_indicators TEXT[],
    
    -- Usage Stats
    account_count INTEGER NOT NULL DEFAULT 0,
    transaction_count BIGINT NOT NULL DEFAULT 0,
    
    -- Linked Data
    linked_accounts UUID[],
    ip_history JSONB,
    
    -- Status
    status VARCHAR(20) NOT NULL DEFAULT 'UNKNOWN',  -- TRUSTED, UNKNOWN, SUSPICIOUS, BLOCKED
    blocked_reason TEXT,
    blocked_at TIMESTAMPTZ,
    
    -- Timestamps
    first_seen TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_seen TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_risk_assessment TIMESTAMPTZ,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_device_records_status ON device_records(status);
CREATE INDEX idx_device_records_risk ON device_records(risk_level);
CREATE INDEX idx_device_records_trust ON device_records(trust_score);
CREATE INDEX idx_device_records_blocked ON device_records(status) WHERE status = 'BLOCKED';
```

#### 5.2.4 Fraud Rules Table

```sql
CREATE TABLE fraud_rules (
    rule_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_code VARCHAR(50) NOT NULL UNIQUE,
    rule_name VARCHAR(200) NOT NULL,
    description TEXT,
    category VARCHAR(50) NOT NULL,
    priority INTEGER NOT NULL DEFAULT 500,
    
    -- Rule Definition
    conditions JSONB NOT NULL,
    action_type VARCHAR(20) NOT NULL,  -- BLOCK, REVIEW, CHALLENGE, FLAG, ALLOW
    reason_code VARCHAR(50),
    action_message TEXT,
    challenge_method VARCHAR(50),
    
    -- Validity
    effective_from TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    effective_until TIMESTAMPTZ,
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',  -- ACTIVE, INACTIVE, TESTING
    version INTEGER NOT NULL DEFAULT 1,
    
    -- Statistics
    hit_count BIGINT NOT NULL DEFAULT 0,
    last_hit TIMESTAMPTZ,
    true_positive_count BIGINT NOT NULL DEFAULT 0,
    false_positive_count BIGINT NOT NULL DEFAULT 0,
    
    -- Audit
    created_by VARCHAR(100) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by VARCHAR(100),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    CONSTRAINT chk_category CHECK (category IN ('VELOCITY', 'AMOUNT', 'BEHAVIOR', 'DEVICE', 'NETWORK', 'GEOGRAPHIC', 'TIME', 'IDENTITY', 'BLOCKLIST', 'WHITELIST')),
    CONSTRAINT chk_action CHECK (action_type IN ('BLOCK', 'REVIEW', 'CHALLENGE', 'FLAG', 'ALLOW'))
);

-- Indexes
CREATE INDEX idx_fraud_rules_status ON fraud_rules(status);
CREATE INDEX idx_fraud_rules_category ON fraud_rules(category);
CREATE INDEX idx_fraud_rules_priority ON fraud_rules(priority);
CREATE INDEX idx_fraud_rules_active ON fraud_rules(status, effective_from, effective_until) 
    WHERE status = 'ACTIVE';
```

#### 5.2.5 ML Models Registry Table

```sql
CREATE TABLE ml_models (
    model_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_name VARCHAR(100) NOT NULL,
    model_type VARCHAR(50) NOT NULL,  -- XGBOOST, LIGHTGBM, NEURAL_NET, ENSEMBLE
    version VARCHAR(20) NOT NULL,
    description TEXT,
    
    -- Artifact
    artifact_path VARCHAR(500) NOT NULL,
    artifact_hash VARCHAR(64) NOT NULL,  -- SHA-256
    artifact_size BIGINT NOT NULL,
    framework VARCHAR(50) NOT NULL,
    
    -- Feature Specification
    feature_count INTEGER NOT NULL,
    feature_names TEXT[],
    feature_spec JSONB,
    
    -- Performance Metrics
    auc DECIMAL(5,4),
    precision_score DECIMAL(5,4),
    recall DECIMAL(5,4),
    f1_score DECIMAL(5,4),
    false_positive_rate DECIMAL(5,4),
    evaluated_at TIMESTAMPTZ,
    
    -- Deployment
    deployment_status VARCHAR(20) NOT NULL DEFAULT 'TRAINING',
    traffic_percentage INTEGER NOT NULL DEFAULT 0,
    deployed_at TIMESTAMPTZ,
    deployed_by VARCHAR(100),
    
    -- Training Metadata
    trained_at TIMESTAMPTZ NOT NULL,
    training_data_start DATE,
    training_data_end DATE,
    sample_count BIGINT,
    fraud_ratio DECIMAL(5,4),
    hyperparameters JSONB,
    
    -- Monitoring
    drift_threshold DECIMAL(5,4) DEFAULT 0.05,
    performance_threshold DECIMAL(5,4) DEFAULT 0.90,
    alert_recipients TEXT[],
    
    -- Audit
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    CONSTRAINT chk_deployment_status CHECK (deployment_status IN ('TRAINING', 'STAGING', 'CHAMPION', 'CHALLENGER', 'RETIRED')),
    CONSTRAINT chk_traffic CHECK (traffic_percentage >= 0 AND traffic_percentage <= 100),
    CONSTRAINT uk_model_version UNIQUE (model_name, version)
);

-- Indexes
CREATE INDEX idx_ml_models_status ON ml_models(deployment_status);
CREATE INDEX idx_ml_models_champion ON ml_models(model_name, deployment_status) 
    WHERE deployment_status = 'CHAMPION';
```

#### 5.2.6 Fraud Alerts Table

```sql
CREATE TABLE fraud_alerts (
    alert_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_number VARCHAR(20) NOT NULL UNIQUE,
    transaction_id UUID NOT NULL,
    fraud_check_id UUID NOT NULL REFERENCES fraud_checks(check_id),
    customer_id UUID NOT NULL,
    
    -- Alert Details
    alert_type VARCHAR(50) NOT NULL,
    severity VARCHAR(20) NOT NULL,  -- LOW, MEDIUM, HIGH, CRITICAL
    priority INTEGER NOT NULL DEFAULT 500,
    queue VARCHAR(50) NOT NULL,
    
    -- Fraud Details
    fraud_score INTEGER NOT NULL,
    triggered_rules TEXT[],
    reason_codes TEXT[],
    description TEXT,
    
    -- Status & Workflow
    status VARCHAR(20) NOT NULL DEFAULT 'OPEN',
    assignee VARCHAR(100),
    assigned_at TIMESTAMPTZ,
    
    -- Resolution
    resolution_type VARCHAR(50),
    resolved_by VARCHAR(100),
    resolved_at TIMESTAMPTZ,
    resolution_notes TEXT,
    
    -- SLA
    sla_deadline TIMESTAMPTZ NOT NULL,
    sla_breached BOOLEAN NOT NULL DEFAULT FALSE,
    
    -- Related Alerts
    linked_alerts UUID[],
    
    -- Timestamps
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    CONSTRAINT chk_alert_status CHECK (status IN ('OPEN', 'ASSIGNED', 'INVESTIGATING', 'ESCALATED', 'RESOLVED', 'CLOSED')),
    CONSTRAINT chk_resolution CHECK (resolution_type IN ('CONFIRMED_FRAUD', 'FALSE_POSITIVE', 'CUSTOMER_DISPUTE', 'INSUFFICIENT_INFO', 'DUPLICATE') OR resolution_type IS NULL)
);

-- Indexes
CREATE INDEX idx_fraud_alerts_status ON fraud_alerts(status);
CREATE INDEX idx_fraud_alerts_queue ON fraud_alerts(queue, status);
CREATE INDEX idx_fraud_alerts_assignee ON fraud_alerts(assignee) WHERE assignee IS NOT NULL;
CREATE INDEX idx_fraud_alerts_sla ON fraud_alerts(sla_deadline) WHERE status NOT IN ('RESOLVED', 'CLOSED');
CREATE INDEX idx_fraud_alerts_customer ON fraud_alerts(customer_id);
CREATE INDEX idx_fraud_alerts_created ON fraud_alerts(created_at);

-- Sequence for alert numbers
CREATE SEQUENCE fraud_alert_number_seq START 1000000;
```

### 5.3 Redis Data Structures

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      REDIS VELOCITY COUNTERS                                 │
└─────────────────────────────────────────────────────────────────────────────┘

Key Patterns:

1. Transaction Count Velocity
   Key: velocity:count:{entityType}:{entityId}:{window}
   Type: String (atomic counter)
   TTL: Based on window
   Examples:
   - velocity:count:account:ACC123:1h     → "15"
   - velocity:count:device:DEV456:1d      → "42"
   - velocity:count:ip:192.168.1.1:5m     → "3"

2. Transaction Amount Velocity
   Key: velocity:amount:{entityType}:{entityId}:{window}
   Type: String (decimal amount)
   TTL: Based on window
   Examples:
   - velocity:amount:account:ACC123:1d    → "15000.00"
   - velocity:amount:card:CARD789:7d      → "45000.00"

3. Unique Beneficiary Count
   Key: velocity:beneficiary:{accountId}:{window}
   Type: HyperLogLog
   TTL: Based on window
   Commands:
   - PFADD velocity:beneficiary:ACC123:1d "BEN001"
   - PFCOUNT velocity:beneficiary:ACC123:1d

4. Customer Profile Cache
   Key: profile:{customerId}
   Type: Hash
   TTL: 1 hour (refreshed on access)
   Fields:
   - risk_tier: "MEDIUM"
   - onboarding_days: "365"
   - avg_amount: "1500.00"
   - std_dev_amount: "500.00"
   - preferred_channel: "WEB"
   - trusted_devices: "3"
   - fraud_history_count: "0"

5. Device Record Cache
   Key: device:{deviceId}
   Type: Hash
   TTL: 24 hours
   Fields:
   - trust_score: "75"
   - risk_level: "LOW"
   - status: "TRUSTED"
   - account_count: "1"
   - first_seen: "2025-01-15T10:30:00Z"
   - last_seen: "2026-02-20T14:22:00Z"

6. Recent Decision Cache
   Key: decision:{transactionId}
   Type: String (JSON)
   TTL: 1 hour
   Value: {"decision":"ALLOW","score":250,"timestamp":"..."}

7. Blocklists
   Key: blocklist:{type}
   Type: Set
   TTL: None (persistent)
   Examples:
   - blocklist:device  → SMEMBERS returns blocked device IDs
   - blocklist:ip      → SMEMBERS returns blocked IP addresses
   - blocklist:card    → SMEMBERS returns blocked card hashes

8. Whitelists
   Key: whitelist:{type}
   Type: Set
   TTL: None (persistent)
   Examples:
   - whitelist:customer → Trusted customers (skip some checks)
   - whitelist:device   → Pre-approved devices

9. Feature Cache
   Key: features:{transactionId}
   Type: Hash
   TTL: 5 minutes
   Fields: All computed features as key-value pairs

10. Model Warm-up Cache
    Key: model:warmup:{modelId}
    Type: String (timestamp)
    TTL: None
    Purpose: Track model initialization status
```

### 5.4 Neo4j Graph Schema

```cypher
// Node Labels and Properties

// Customer Node
CREATE CONSTRAINT customer_id IF NOT EXISTS
FOR (c:Customer) REQUIRE c.customerId IS UNIQUE;

(:Customer {
    customerId: UUID,
    customerNumber: String,
    riskTier: String,
    onboardingDate: Date,
    status: String,
    fraudCount: Integer,
    lastActivity: DateTime
})

// Device Node
CREATE CONSTRAINT device_id IF NOT EXISTS
FOR (d:Device) REQUIRE d.deviceId IS UNIQUE;

(:Device {
    deviceId: String,
    deviceType: String,
    trustScore: Integer,
    riskLevel: String,
    status: String,
    firstSeen: DateTime,
    lastSeen: DateTime
})

// IP Node
CREATE CONSTRAINT ip_address IF NOT EXISTS
FOR (i:IP) REQUIRE i.ipAddress IS UNIQUE;

(:IP {
    ipAddress: String,
    country: String,
    isVPN: Boolean,
    isProxy: Boolean,
    riskScore: Integer,
    firstSeen: DateTime
})

// Beneficiary Node
CREATE CONSTRAINT beneficiary_id IF NOT EXISTS
FOR (b:Beneficiary) REQUIRE b.beneficiaryId IS UNIQUE;

(:Beneficiary {
    beneficiaryId: String,
    accountNumber: String,
    bankCode: String,
    country: String,
    paymentCount: Integer,
    riskScore: Integer
})

// FraudCase Node
(:FraudCase {
    caseId: UUID,
    caseNumber: String,
    fraudType: String,
    confirmedAt: DateTime,
    amount: Decimal
})

// Relationships

// Customer uses device
(:Customer)-[:USES_DEVICE {
    firstUsed: DateTime,
    lastUsed: DateTime,
    transactionCount: Integer
}]->(:Device)

// Customer logged from IP
(:Customer)-[:LOGGED_FROM {
    firstSeen: DateTime,
    lastSeen: DateTime,
    loginCount: Integer
}]->(:IP)

// Customer pays beneficiary
(:Customer)-[:PAYS_TO {
    firstPayment: DateTime,
    lastPayment: DateTime,
    paymentCount: Integer,
    totalAmount: Decimal
}]->(:Beneficiary)

// Device shares between customers
(:Customer)-[:SHARES_DEVICE {
    sharedDeviceId: String,
    overlapPeriod: Duration
}]->(:Customer)

// Fraud case linkage
(:Customer)-[:CONFIRMED_FRAUD {
    confirmedAt: DateTime,
    role: String
}]->(:FraudCase)

// Example Fraud Ring Detection Query
MATCH (c:Customer)-[:USES_DEVICE]->(d:Device)<-[:USES_DEVICE]-(other:Customer)
WHERE c.customerId <> other.customerId
  AND d.status = 'SUSPICIOUS'
WITH c, d, COLLECT(DISTINCT other) AS sharedUsers
WHERE SIZE(sharedUsers) >= 2
MATCH (c)-[:CONFIRMED_FRAUD]->(:FraudCase)
RETURN c, d, sharedUsers
```

---

## 6. API Design

### 6.1 REST API Endpoints

```yaml
openapi: 3.0.3
info:
  title: Fraud Detection API
  version: 1.0.0
  description: API for real-time fraud detection and management

servers:
  - url: https://api.paymenthub.com/fraud/v1
    description: Production
  - url: https://api-staging.paymenthub.com/fraud/v1
    description: Staging

tags:
  - name: Fraud Check
    description: Real-time fraud evaluation endpoints
  - name: Alerts
    description: Fraud alert management
  - name: Rules
    description: Fraud rule management
  - name: Models
    description: ML model management
  - name: Profiles
    description: Customer risk profile endpoints
  - name: Devices
    description: Device management endpoints
  - name: Feedback
    description: Decision feedback endpoints
```

### 6.2 Core Fraud Check Endpoints

#### 6.2.1 Evaluate Transaction

```yaml
POST /evaluate
summary: Evaluate a transaction for fraud risk
description: |
  Performs real-time fraud evaluation including ML scoring, rule evaluation,
  behavioral analysis, device assessment, and velocity checks.
  
tags:
  - Fraud Check

requestBody:
  required: true
  content:
    application/json:
      schema:
        $ref: '#/components/schemas/FraudCheckRequest'

responses:
  200:
    description: Fraud evaluation completed
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/FraudCheckResponse'
  400:
    description: Invalid request
  429:
    description: Rate limit exceeded
  500:
    description: Internal server error
  503:
    description: Service unavailable

x-timeout: 200ms
x-rate-limit: 10000/minute
```

#### 6.2.2 Request/Response Schemas

```yaml
components:
  schemas:
    FraudCheckRequest:
      type: object
      required:
        - transactionId
        - customerId
        - transaction
      properties:
        transactionId:
          type: string
          format: uuid
          description: Unique transaction identifier
        customerId:
          type: string
          format: uuid
          description: Customer identifier
        checkType:
          type: string
          enum: [REAL_TIME, BATCH, RECHECK]
          default: REAL_TIME
        transaction:
          $ref: '#/components/schemas/TransactionDetails'
        device:
          $ref: '#/components/schemas/DeviceInfo'
        session:
          $ref: '#/components/schemas/SessionContext'
        channel:
          type: string
          enum: [WEB, MOBILE_APP, API, BRANCH, ATM]
        metadata:
          type: object
          additionalProperties: true

    TransactionDetails:
      type: object
      required:
        - amount
        - currency
        - type
      properties:
        amount:
          type: number
          format: decimal
          example: 1500.00
        currency:
          type: string
          pattern: ^[A-Z]{3}$
          example: USD
        type:
          type: string
          enum: [PAYMENT, TRANSFER, WITHDRAWAL, DEPOSIT]
        originatorAccount:
          type: string
        beneficiaryAccount:
          type: string
        beneficiaryName:
          type: string
        beneficiaryBank:
          type: string
        beneficiaryCountry:
          type: string
        paymentPurpose:
          type: string
        isRecurring:
          type: boolean
          default: false
        isFirstPaymentToBeneficiary:
          type: boolean

    DeviceInfo:
      type: object
      properties:
        deviceId:
          type: string
          description: Device fingerprint hash
        fingerprint:
          type: object
          description: Raw fingerprint data
        deviceType:
          type: string
          enum: [DESKTOP, MOBILE, TABLET, UNKNOWN]
        operatingSystem:
          type: string
        browser:
          type: string
        ipAddress:
          type: string
        geolocation:
          $ref: '#/components/schemas/GeoLocation'
        userAgent:
          type: string

    SessionContext:
      type: object
      properties:
        sessionId:
          type: string
        sessionStart:
          type: string
          format: date-time
        loginMethod:
          type: string
          enum: [PASSWORD, BIOMETRIC, SSO, OTP]
        actionsInSession:
          type: integer
        isNewSession:
          type: boolean

    GeoLocation:
      type: object
      properties:
        country:
          type: string
        region:
          type: string
        city:
          type: string
        latitude:
          type: number
        longitude:
          type: number
        isVPN:
          type: boolean
        isProxy:
          type: boolean

    FraudCheckResponse:
      type: object
      required:
        - checkId
        - transactionId
        - decision
        - fraudScore
        - riskLevel
      properties:
        checkId:
          type: string
          format: uuid
        transactionId:
          type: string
          format: uuid
        decision:
          type: string
          enum: [ALLOW, CHALLENGE, REVIEW, BLOCK]
        fraudScore:
          type: integer
          minimum: 0
          maximum: 1000
          description: Composite fraud score
        riskLevel:
          type: string
          enum: [LOW, MEDIUM, HIGH, CRITICAL]
        reasonCodes:
          type: array
          items:
            $ref: '#/components/schemas/ReasonCode'
        triggeredRules:
          type: array
          items:
            type: string
        challengeDetails:
          $ref: '#/components/schemas/ChallengeDetails'
        alertId:
          type: string
          format: uuid
          description: Alert ID if alert was created
        evaluationDetails:
          $ref: '#/components/schemas/EvaluationDetails'
        latencyMs:
          type: integer
          description: Total evaluation time in milliseconds

    ReasonCode:
      type: object
      properties:
        code:
          type: string
          example: FRD-VEL-001
        category:
          type: string
        description:
          type: string
        scoreContribution:
          type: integer

    ChallengeDetails:
      type: object
      properties:
        challengeMethod:
          type: string
          enum: [OTP_SMS, OTP_EMAIL, BIOMETRIC, SECURITY_QUESTIONS, CALLBACK]
        challengeId:
          type: string
        expiresAt:
          type: string
          format: date-time
        fallbackMethods:
          type: array
          items:
            type: string

    EvaluationDetails:
      type: object
      properties:
        mlScore:
          type: integer
        mlModelVersion:
          type: string
        behavioralDeviation:
          type: number
        deviceRiskScore:
          type: integer
        velocityFlags:
          type: array
          items:
            type: string
        networkRiskScore:
          type: integer
```

### 6.3 Alert Management Endpoints

```yaml
# Get Alert
GET /alerts/{alertId}
summary: Retrieve fraud alert details
parameters:
  - name: alertId
    in: path
    required: true
    schema:
      type: string
      format: uuid
responses:
  200:
    description: Alert details
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/FraudAlert'

# List Alerts
GET /alerts
summary: List fraud alerts with filtering
parameters:
  - name: status
    in: query
    schema:
      type: string
      enum: [OPEN, ASSIGNED, INVESTIGATING, RESOLVED]
  - name: queue
    in: query
    schema:
      type: string
  - name: assignee
    in: query
    schema:
      type: string
  - name: severity
    in: query
    schema:
      type: string
      enum: [LOW, MEDIUM, HIGH, CRITICAL]
  - name: fromDate
    in: query
    schema:
      type: string
      format: date
  - name: toDate
    in: query
    schema:
      type: string
      format: date
  - name: page
    in: query
    schema:
      type: integer
      default: 0
  - name: size
    in: query
    schema:
      type: integer
      default: 20
      maximum: 100
responses:
  200:
    description: Paginated alert list
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/AlertListResponse'

# Assign Alert
POST /alerts/{alertId}/assign
summary: Assign alert to investigator
requestBody:
  content:
    application/json:
      schema:
        type: object
        required:
          - assignee
        properties:
          assignee:
            type: string
          notes:
            type: string
responses:
  200:
    description: Alert assigned

# Resolve Alert
POST /alerts/{alertId}/resolve
summary: Resolve fraud alert
requestBody:
  content:
    application/json:
      schema:
        type: object
        required:
          - resolutionType
        properties:
          resolutionType:
            type: string
            enum: [CONFIRMED_FRAUD, FALSE_POSITIVE, CUSTOMER_DISPUTE, INSUFFICIENT_INFO]
          notes:
            type: string
          updateCustomerProfile:
            type: boolean
            default: true
responses:
  200:
    description: Alert resolved
```

### 6.4 Rule Management Endpoints

```yaml
# List Rules
GET /rules
summary: List fraud rules
parameters:
  - name: category
    in: query
    schema:
      type: string
  - name: status
    in: query
    schema:
      type: string
      enum: [ACTIVE, INACTIVE, TESTING]
responses:
  200:
    description: List of fraud rules

# Create Rule
POST /rules
summary: Create new fraud rule
requestBody:
  content:
    application/json:
      schema:
        $ref: '#/components/schemas/CreateRuleRequest'
responses:
  201:
    description: Rule created
  400:
    description: Invalid rule definition

# Update Rule
PUT /rules/{ruleId}
summary: Update fraud rule
parameters:
  - name: ruleId
    in: path
    required: true
    schema:
      type: string
      format: uuid
requestBody:
  content:
    application/json:
      schema:
        $ref: '#/components/schemas/UpdateRuleRequest'
responses:
  200:
    description: Rule updated

# Test Rule
POST /rules/{ruleId}/test
summary: Test rule against sample transactions
requestBody:
  content:
    application/json:
      schema:
        type: object
        properties:
          sampleSize:
            type: integer
            default: 1000
          dateRange:
            type: object
            properties:
              from:
                type: string
                format: date
              to:
                type: string
                format: date
responses:
  200:
    description: Test results
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/RuleTestResult'

# Rule Schemas
CreateRuleRequest:
  type: object
  required:
    - ruleCode
    - ruleName
    - category
    - conditions
    - action
  properties:
    ruleCode:
      type: string
      pattern: ^[A-Z]{3}-[A-Z]{3}-[0-9]{3}$
    ruleName:
      type: string
    description:
      type: string
    category:
      type: string
      enum: [VELOCITY, AMOUNT, BEHAVIOR, DEVICE, NETWORK, GEOGRAPHIC, TIME]
    priority:
      type: integer
      minimum: 1
      maximum: 1000
      default: 500
    conditions:
      type: array
      items:
        $ref: '#/components/schemas/RuleCondition'
    action:
      $ref: '#/components/schemas/RuleAction'
    effectiveFrom:
      type: string
      format: date-time
    effectiveUntil:
      type: string
      format: date-time

RuleCondition:
  type: object
  required:
    - field
    - operator
    - value
  properties:
    field:
      type: string
      example: transaction.amount
    operator:
      type: string
      enum: [EQ, NE, GT, LT, GTE, LTE, IN, NOT_IN, CONTAINS, STARTS_WITH, REGEX]
    value:
      oneOf:
        - type: string
        - type: number
        - type: array
    logicalOperator:
      type: string
      enum: [AND, OR]

RuleAction:
  type: object
  required:
    - actionType
  properties:
    actionType:
      type: string
      enum: [BLOCK, REVIEW, CHALLENGE, FLAG, ALLOW]
    reasonCode:
      type: string
    message:
      type: string
    challengeMethod:
      type: string
```

### 6.5 Model Management Endpoints

```yaml
# List Models
GET /models
summary: List ML models
parameters:
  - name: status
    in: query
    schema:
      type: string
      enum: [TRAINING, STAGING, CHAMPION, CHALLENGER, RETIRED]
responses:
  200:
    description: List of models

# Get Model Details
GET /models/{modelId}
summary: Get ML model details
responses:
  200:
    description: Model details

# Deploy Model
POST /models/{modelId}/deploy
summary: Deploy model to production
requestBody:
  content:
    application/json:
      schema:
        type: object
        properties:
          deploymentStatus:
            type: string
            enum: [STAGING, CHAMPION, CHALLENGER]
          trafficPercentage:
            type: integer
            minimum: 0
            maximum: 100
responses:
  200:
    description: Model deployed

# Get Model Performance
GET /models/{modelId}/performance
summary: Get model performance metrics
parameters:
  - name: period
    in: query
    schema:
      type: string
      enum: [1D, 7D, 30D, 90D]
responses:
  200:
    description: Performance metrics
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/ModelPerformance'
```

### 6.6 Feedback Endpoint

```yaml
POST /feedback
summary: Submit decision feedback
description: |
  Submit feedback on fraud decisions to improve model accuracy.
  Used by case management after investigation completion.

requestBody:
  required: true
  content:
    application/json:
      schema:
        type: object
        required:
          - transactionId
          - feedbackType
        properties:
          transactionId:
            type: string
            format: uuid
          checkId:
            type: string
            format: uuid
          alertId:
            type: string
            format: uuid
          feedbackType:
            type: string
            enum: [CONFIRMED_FRAUD, FALSE_POSITIVE, INCONCLUSIVE]
          fraudType:
            type: string
            description: Specific fraud type if confirmed
          notes:
            type: string
          investigatorId:
            type: string

responses:
  200:
    description: Feedback recorded
  400:
    description: Invalid feedback
```

### 6.7 gRPC Service Definition

```protobuf
syntax = "proto3";

package fraud.v1;

option java_package = "com.paymenthub.fraud.grpc";
option java_outer_classname = "FraudServiceProto";

// Fraud Detection Service - High-performance gRPC interface
service FraudService {
  // Real-time fraud evaluation
  rpc EvaluateTransaction(FraudCheckRequest) returns (FraudCheckResponse);
  
  // Batch evaluation
  rpc EvaluateBatch(stream FraudCheckRequest) returns (stream FraudCheckResponse);
  
  // Get customer risk profile
  rpc GetCustomerProfile(GetProfileRequest) returns (CustomerProfile);
  
  // Update customer risk tier
  rpc UpdateRiskTier(UpdateRiskTierRequest) returns (UpdateRiskTierResponse);
  
  // Check device reputation
  rpc CheckDevice(DeviceCheckRequest) returns (DeviceCheckResponse);
  
  // Block device
  rpc BlockDevice(BlockDeviceRequest) returns (BlockDeviceResponse);
}

message FraudCheckRequest {
  string transaction_id = 1;
  string customer_id = 2;
  CheckType check_type = 3;
  TransactionDetails transaction = 4;
  DeviceInfo device = 5;
  SessionContext session = 6;
  string channel = 7;
  map<string, string> metadata = 8;
}

message FraudCheckResponse {
  string check_id = 1;
  string transaction_id = 2;
  Decision decision = 3;
  int32 fraud_score = 4;
  RiskLevel risk_level = 5;
  repeated ReasonCode reason_codes = 6;
  repeated string triggered_rules = 7;
  ChallengeDetails challenge = 8;
  string alert_id = 9;
  int64 latency_ms = 10;
}

enum Decision {
  DECISION_UNSPECIFIED = 0;
  ALLOW = 1;
  CHALLENGE = 2;
  REVIEW = 3;
  BLOCK = 4;
}

enum RiskLevel {
  RISK_LEVEL_UNSPECIFIED = 0;
  LOW = 1;
  MEDIUM = 2;
  HIGH = 3;
  CRITICAL = 4;
}

enum CheckType {
  CHECK_TYPE_UNSPECIFIED = 0;
  REAL_TIME = 1;
  BATCH = 2;
  RECHECK = 3;
}

message TransactionDetails {
  string amount = 1;  // Decimal as string for precision
  string currency = 2;
  string type = 3;
  string originator_account = 4;
  string beneficiary_account = 5;
  string beneficiary_name = 6;
  string beneficiary_country = 7;
  bool is_first_payment_to_beneficiary = 8;
}

message DeviceInfo {
  string device_id = 1;
  string device_type = 2;
  string ip_address = 3;
  GeoLocation geolocation = 4;
  string user_agent = 5;
}

message GeoLocation {
  string country = 1;
  string city = 2;
  double latitude = 3;
  double longitude = 4;
  bool is_vpn = 5;
}

message SessionContext {
  string session_id = 1;
  int64 session_start_epoch = 2;
  string login_method = 3;
  int32 actions_in_session = 4;
}

message ReasonCode {
  string code = 1;
  string category = 2;
  string description = 3;
  int32 score_contribution = 4;
}

message ChallengeDetails {
  string challenge_method = 1;
  string challenge_id = 2;
  int64 expires_at_epoch = 3;
}

message GetProfileRequest {
  string customer_id = 1;
}

message CustomerProfile {
  string customer_id = 1;
  string risk_tier = 2;
  BehavioralProfile behavioral = 3;
  FraudHistory fraud_history = 4;
}

message BehavioralProfile {
  double avg_transaction_amount = 1;
  double std_dev_amount = 2;
  repeated int32 preferred_hours = 3;
  string preferred_channel = 4;
}

message FraudHistory {
  int32 confirmed_fraud_count = 1;
  int32 false_positive_count = 2;
  int32 chargeback_count = 3;
}

message UpdateRiskTierRequest {
  string customer_id = 1;
  string new_risk_tier = 2;
  string reason = 3;
}

message UpdateRiskTierResponse {
  bool success = 1;
  string previous_tier = 2;
  string new_tier = 3;
}

message DeviceCheckRequest {
  string device_id = 1;
  DeviceInfo device_info = 2;
}

message DeviceCheckResponse {
  string device_id = 1;
  int32 trust_score = 2;
  string risk_level = 3;
  string status = 4;
  repeated string fraud_indicators = 5;
}

message BlockDeviceRequest {
  string device_id = 1;
  string reason = 2;
  repeated string linked_accounts = 3;
}

message BlockDeviceResponse {
  bool success = 1;
  string blocked_at = 2;
}
```

---

## 7. Event-Driven Architecture

### 7.1 Kafka Topic Design

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FRAUD DETECTION EVENT TOPICS                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ INBOUND TOPICS (Consumed by Fraud Detection)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│ payment.transactions.initiated                                              │
│ ├── Description: New payment transactions requiring fraud checks           │
│ ├── Partitions: 24                                                         │
│ ├── Replication: 3                                                         │
│ ├── Retention: 7 days                                                      │
│ ├── Key: transactionId                                                     │
│ └── Consumer Group: fraud-detection-service                                │
│                                                                             │
│ customer.profile.updated                                                    │
│ ├── Description: Customer profile changes affecting risk assessment        │
│ ├── Partitions: 12                                                         │
│ ├── Replication: 3                                                         │
│ ├── Retention: 3 days                                                      │
│ ├── Key: customerId                                                        │
│ └── Consumer Group: fraud-detection-service                                │
│                                                                             │
│ auth.login.events                                                           │
│ ├── Description: Login events for session and device tracking              │
│ ├── Partitions: 12                                                         │
│ ├── Key: customerId                                                        │
│ └── Consumer Group: fraud-detection-service                                │
│                                                                             │
│ case.resolution.completed                                                   │
│ ├── Description: Case resolution outcomes for feedback loop                │
│ ├── Partitions: 6                                                          │
│ ├── Key: alertId                                                           │
│ └── Consumer Group: fraud-feedback-processor                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ OUTBOUND TOPICS (Published by Fraud Detection)                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│ fraud.check.completed                                                       │
│ ├── Description: Fraud evaluation results                                  │
│ ├── Partitions: 24                                                         │
│ ├── Replication: 3                                                         │
│ ├── Retention: 30 days                                                     │
│ ├── Key: transactionId                                                     │
│ └── Consumers:                                                             │
│     ├── payment-orchestration-service                                      │
│     ├── analytics-service                                                  │
│     └── audit-service                                                      │
│                                                                             │
│ fraud.alert.created                                                         │
│ ├── Description: New fraud alerts requiring investigation                  │
│ ├── Partitions: 12                                                         │
│ ├── Replication: 3                                                         │
│ ├── Retention: 30 days                                                     │
│ ├── Key: alertId                                                           │
│ └── Consumers:                                                             │
│     ├── case-management-service                                            │
│     ├── notification-service                                               │
│     └── analytics-service                                                  │
│                                                                             │
│ fraud.alert.resolved                                                        │
│ ├── Description: Alert resolution events                                   │
│ ├── Partitions: 6                                                          │
│ ├── Key: alertId                                                           │
│ └── Consumers:                                                             │
│     ├── analytics-service                                                  │
│     └── ml-training-pipeline                                               │
│                                                                             │
│ fraud.device.blocked                                                        │
│ ├── Description: Device blocking events                                    │
│ ├── Partitions: 6                                                          │
│ ├── Key: deviceId                                                          │
│ └── Consumers:                                                             │
│     ├── auth-service                                                       │
│     └── notification-service                                               │
│                                                                             │
│ fraud.customer.risk.updated                                                 │
│ ├── Description: Customer risk tier changes                                │
│ ├── Partitions: 12                                                         │
│ ├── Key: customerId                                                        │
│ └── Consumers:                                                             │
│     ├── customer-service                                                   │
│     ├── limits-service                                                     │
│     └── aml-screening-service                                              │
│                                                                             │
│ fraud.velocity.breach                                                       │
│ ├── Description: Velocity threshold breaches                               │
│ ├── Partitions: 6                                                          │
│ ├── Key: entityId                                                          │
│ └── Consumers:                                                             │
│     ├── monitoring-service                                                 │
│     └── notification-service                                               │
│                                                                             │
│ fraud.network.detected                                                      │
│ ├── Description: Fraud network/ring detection events                       │
│ ├── Partitions: 3                                                          │
│ ├── Key: networkId                                                         │
│ └── Consumers:                                                             │
│     ├── investigation-service                                              │
│     └── aml-screening-service                                              │
│                                                                             │
│ fraud.model.metrics                                                         │
│ ├── Description: ML model performance metrics                              │
│ ├── Partitions: 3                                                          │
│ ├── Key: modelId                                                           │
│ └── Consumers:                                                             │
│     ├── ml-monitoring-service                                              │
│     └── analytics-service                                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Event Schemas (Avro)

```avro
// FraudCheckCompleted Event
{
  "type": "record",
  "name": "FraudCheckCompleted",
  "namespace": "com.paymenthub.fraud.events",
  "fields": [
    {"name": "eventId", "type": "string"},
    {"name": "eventTimestamp", "type": "long", "logicalType": "timestamp-millis"},
    {"name": "eventVersion", "type": "string", "default": "1.0"},
    {"name": "checkId", "type": "string"},
    {"name": "transactionId", "type": "string"},
    {"name": "customerId", "type": "string"},
    {"name": "decision", "type": {"type": "enum", "name": "Decision", 
      "symbols": ["ALLOW", "CHALLENGE", "REVIEW", "BLOCK"]}},
    {"name": "fraudScore", "type": "int"},
    {"name": "riskLevel", "type": {"type": "enum", "name": "RiskLevel",
      "symbols": ["LOW", "MEDIUM", "HIGH", "CRITICAL"]}},
    {"name": "reasonCodes", "type": {"type": "array", "items": "string"}},
    {"name": "triggeredRules", "type": {"type": "array", "items": "string"}},
    {"name": "mlModelId", "type": ["null", "string"], "default": null},
    {"name": "mlModelVersion", "type": ["null", "string"], "default": null},
    {"name": "mlScore", "type": ["null", "int"], "default": null},
    {"name": "challengeMethod", "type": ["null", "string"], "default": null},
    {"name": "alertId", "type": ["null", "string"], "default": null},
    {"name": "latencyMs", "type": "long"},
    {"name": "channel", "type": "string"},
    {"name": "transactionAmount", "type": "string"},
    {"name": "transactionCurrency", "type": "string"}
  ]
}

// FraudAlertCreated Event
{
  "type": "record",
  "name": "FraudAlertCreated",
  "namespace": "com.paymenthub.fraud.events",
  "fields": [
    {"name": "eventId", "type": "string"},
    {"name": "eventTimestamp", "type": "long", "logicalType": "timestamp-millis"},
    {"name": "alertId", "type": "string"},
    {"name": "alertNumber", "type": "string"},
    {"name": "transactionId", "type": "string"},
    {"name": "customerId", "type": "string"},
    {"name": "alertType", "type": "string"},
    {"name": "severity", "type": {"type": "enum", "name": "Severity",
      "symbols": ["LOW", "MEDIUM", "HIGH", "CRITICAL"]}},
    {"name": "priority", "type": "int"},
    {"name": "queue", "type": "string"},
    {"name": "fraudScore", "type": "int"},
    {"name": "triggeredRules", "type": {"type": "array", "items": "string"}},
    {"name": "reasonCodes", "type": {"type": "array", "items": "string"}},
    {"name": "description", "type": "string"},
    {"name": "slaDeadline", "type": "long", "logicalType": "timestamp-millis"},
    {"name": "transactionAmount", "type": "string"},
    {"name": "transactionCurrency", "type": "string"}
  ]
}

// CustomerRiskUpdated Event
{
  "type": "record",
  "name": "CustomerRiskUpdated",
  "namespace": "com.paymenthub.fraud.events",
  "fields": [
    {"name": "eventId", "type": "string"},
    {"name": "eventTimestamp", "type": "long", "logicalType": "timestamp-millis"},
    {"name": "customerId", "type": "string"},
    {"name": "previousRiskTier", "type": "string"},
    {"name": "newRiskTier", "type": "string"},
    {"name": "reason", "type": "string"},
    {"name": "triggeredBy", "type": {"type": "enum", "name": "TriggerSource",
      "symbols": ["FEEDBACK", "SYSTEM", "MANUAL", "MODEL"]}},
    {"name": "effectiveFrom", "type": "long", "logicalType": "timestamp-millis"}
  ]
}

// VelocityBreachDetected Event
{
  "type": "record",
  "name": "VelocityBreachDetected",
  "namespace": "com.paymenthub.fraud.events",
  "fields": [
    {"name": "eventId", "type": "string"},
    {"name": "eventTimestamp", "type": "long", "logicalType": "timestamp-millis"},
    {"name": "entityType", "type": "string"},
    {"name": "entityId", "type": "string"},
    {"name": "velocityType", "type": "string"},
    {"name": "window", "type": "string"},
    {"name": "threshold", "type": "double"},
    {"name": "currentValue", "type": "double"},
    {"name": "breachPercentage", "type": "double"},
    {"name": "transactionId", "type": "string"},
    {"name": "customerId", "type": "string"}
  ]
}
```

### 7.3 Event Flow Patterns

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    REAL-TIME FRAUD CHECK EVENT FLOW                          │
└─────────────────────────────────────────────────────────────────────────────┘

       ┌───────────────┐
       │  Payment      │
       │  Gateway      │
       └───────┬───────┘
               │ Sync API Call
               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         FRAUD DETECTION SERVICE                              │
│                                                                             │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌───────────┐ │
│  │ Request │    │Feature  │    │ Parallel│    │Decision │    │ Response  │ │
│  │ Handler │───▶│ Builder │───▶│ Eval    │───▶│ Engine  │───▶│ Builder   │ │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘    └───────────┘ │
│                                                     │                       │
│                              ┌──────────────────────┘                       │
│                              │ (If REVIEW or BLOCK)                         │
│                              ▼                                              │
│                    ┌─────────────────┐                                      │
│                    │ Alert Generator │                                      │
│                    └────────┬────────┘                                      │
│                             │                                               │
└─────────────────────────────┼───────────────────────────────────────────────┘
               │              │
               │ Response     │ Async Publish
               ▼              ▼
       ┌───────────────┐   ┌─────────────────────┐
       │  Payment      │   │ fraud.check.        │
       │  Gateway      │   │ completed           │────┐
       └───────────────┘   └─────────────────────┘    │
                                                       │
                           ┌─────────────────────┐    │
                           │ fraud.alert.        │◀───┤ (If alert created)
                           │ created             │    │
                           └──────────┬──────────┘    │
                                      │               │
                     ┌────────────────┼───────────────┼────────────────┐
                     ▼                ▼               ▼                ▼
              ┌───────────┐   ┌───────────┐   ┌───────────┐    ┌───────────┐
              │ Case Mgmt │   │ Notifier  │   │ Analytics │    │  Audit    │
              │ Service   │   │ Service   │   │ Service   │    │  Service  │
              └───────────┘   └───────────┘   └───────────┘    └───────────┘
```

### 7.4 Consumer Configuration

```yaml
# Kafka Consumer Configuration
fraud-detection-consumer:
  bootstrap-servers: ${KAFKA_BROKERS}
  group-id: fraud-detection-service
  auto-offset-reset: latest
  enable-auto-commit: false
  max-poll-records: 500
  session-timeout-ms: 30000
  heartbeat-interval-ms: 10000
  
  # Exactly-once semantics
  isolation-level: read_committed
  
  # Deserializers
  key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
  value-deserializer: io.confluent.kafka.serializers.KafkaAvroDeserializer
  
  # Schema Registry
  schema-registry-url: ${SCHEMA_REGISTRY_URL}
  specific-avro-reader: true
  
  # Consumer specific
  topics:
    - payment.transactions.initiated
    - customer.profile.updated
    - auth.login.events
    - case.resolution.completed

# Producer Configuration  
fraud-detection-producer:
  bootstrap-servers: ${KAFKA_BROKERS}
  acks: all
  retries: 3
  retry-backoff-ms: 100
  
  # Exactly-once semantics
  enable-idempotence: true
  transactional-id: fraud-detection-tx-${POD_NAME}
  
  # Serializers
  key-serializer: org.apache.kafka.common.serialization.StringSerializer
  value-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
  
  # Performance
  batch-size: 16384
  linger-ms: 5
  buffer-memory: 33554432
  compression-type: lz4
```

---

## 8. Sequence Diagrams

### 8.1 Real-Time Fraud Check Sequence

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  REAL-TIME FRAUD CHECK SEQUENCE DIAGRAM                      │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────┐  ┌──────────┐  ┌─────────┐  ┌────────┐  ┌─────────┐  ┌──────────┐
│ Payment │  │ Fraud    │  │ Feature │  │ Redis  │  │ ML      │  │ Rule     │
│ Gateway │  │ Service  │  │ Engine  │  │ Cache  │  │ Engine  │  │ Engine   │
└────┬────┘  └────┬─────┘  └────┬────┘  └───┬────┘  └────┬────┘  └────┬─────┘
     │            │             │           │            │            │
     │ POST /evaluate           │           │            │            │
     │───────────▶│             │           │            │            │
     │            │             │           │            │            │
     │            │ buildContext│           │            │            │
     │            │────────────▶│           │            │            │
     │            │             │           │            │            │
     │            │             │ GET profile:{customerId}            │
     │            │             │──────────▶│            │            │
     │            │             │◀──────────│            │            │
     │            │             │           │            │            │
     │            │             │ GET velocity:* (parallel)           │
     │            │             │──────────▶│            │            │
     │            │             │◀──────────│            │            │
     │            │             │           │            │            │
     │            │◀────────────│           │            │            │
     │            │ FeatureVector           │            │            │
     │            │             │           │            │            │
     │            ├─────────────────────────────────────▶│            │
     │            │             │           │ score(features)         │
     │            │             │           │            │            │
     │            ├──────────────────────────────────────────────────▶│
     │            │             │           │            │ evaluate() │
     │            │             │           │            │            │
     │            │◀────────────────────────────────────┤            │
     │            │ MLScore = 350           │            │            │
     │            │             │           │            │            │
     │            │◀─────────────────────────────────────────────────┤
     │            │ RuleResult (no matches) │            │            │
     │            │             │           │            │            │
     │            │ aggregateResults()      │            │            │
     │            │────────────▶│           │            │            │
     │            │             │           │            │            │
     │            │ updateVelocity          │            │            │
     │            │─────────────────────────▶│            │            │
     │            │             │           │            │            │
     │            │ publishEvent           │            │            │
     │            │ (async)    │            │            │            │
     │            │─────────────────────────────────────▶ Kafka       │
     │            │             │           │            │            │
     │◀───────────│             │           │            │            │
     │ FraudCheckResponse       │           │            │            │
     │ decision=ALLOW           │           │            │            │
     │ score=350, risk=MEDIUM   │           │            │            │
     │            │             │           │            │            │

Timeline: ~120ms total
- Context building: 10ms
- Feature computation: 25ms (including Redis calls)
- ML scoring: 40ms
- Rule evaluation: 25ms
- Decision & response: 20ms
```

### 8.2 High-Risk Transaction with Step-Up Authentication

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              HIGH-RISK TRANSACTION WITH STEP-UP AUTH                         │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────┐ ┌──────────┐ ┌─────────┐ ┌────────┐ ┌─────────┐ ┌─────────┐ ┌─────┐
│ Payment │ │ Fraud    │ │ Device  │ │ Behav  │ │ Network │ │ Auth    │ │Alert│
│ Gateway │ │ Service  │ │ Engine  │ │ Engine │ │ Engine  │ │ Service │ │ Gen │
└────┬────┘ └────┬─────┘ └────┬────┘ └───┬────┘ └────┬────┘ └────┬────┘ └──┬──┘
     │           │            │          │           │           │         │
     │ POST /evaluate         │          │           │           │         │
     │──────────▶│            │          │           │           │         │
     │           │            │          │           │           │         │
     │           │ assessDevice          │           │           │         │
     │           │───────────▶│          │           │           │         │
     │           │◀───────────│          │           │           │         │
     │           │ DeviceScore=HIGH      │           │           │         │
     │           │ (new device, VPN)     │           │           │         │
     │           │            │          │           │           │         │
     │           │ analyzeBehavior       │           │           │         │
     │           │───────────────────────▶│           │           │         │
     │           │◀───────────────────────│           │           │         │
     │           │ BehavioralDeviation=2.5σ          │           │         │
     │           │ (unusual amount & time)           │           │         │
     │           │            │          │           │           │         │
     │           │ checkNetwork           │           │           │         │
     │           │───────────────────────────────────▶│           │         │
     │           │◀───────────────────────────────────│           │         │
     │           │ NetworkScore=MODERATE │           │           │         │
     │           │            │          │           │           │         │
     │           │ Decision = CHALLENGE  │           │           │         │
     │           │ Method = BIOMETRIC + OTP          │           │         │
     │           │            │          │           │           │         │
     │           │ triggerStepUp         │           │           │         │
     │           │───────────────────────────────────────────────▶│         │
     │           │◀───────────────────────────────────────────────│         │
     │           │ challengeId=xxx       │           │           │         │
     │           │            │          │           │           │         │
     │           │ createAlert (async)   │           │           │         │
     │           │────────────────────────────────────────────────────────▶│
     │           │            │          │           │           │         │
     │◀──────────│            │          │           │           │         │
     │ FraudCheckResponse     │          │           │           │         │
     │ decision=CHALLENGE     │          │           │           │         │
     │ challengeMethod=[BIOMETRIC,OTP_SMS]           │           │         │
     │ challengeId=xxx        │          │           │           │         │
     │           │            │          │           │           │         │
     │           │            │          │           │           │         │
     │ (User completes biometric)        │           │           │         │
     │           │            │          │           │           │         │
     │ POST /challenge/complete          │           │           │         │
     │──────────▶│            │          │           │           │         │
     │           │ validateChallenge     │           │           │         │
     │           │───────────────────────────────────────────────▶│         │
     │           │◀───────────────────────────────────────────────│         │
     │           │ Success    │          │           │           │         │
     │           │            │          │           │           │         │
     │           │ Update decision: ALLOW│           │           │         │
     │           │            │          │           │           │         │
     │◀──────────│            │          │           │           │         │
     │ ChallengeResult=SUCCESS│          │           │           │         │
     │ finalDecision=ALLOW    │          │           │           │         │
```

### 8.3 Blocked Transaction with Alert Creation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                BLOCKED TRANSACTION - FRAUD RING DETECTED                     │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────┐ ┌──────────┐ ┌────────┐ ┌─────────┐ ┌───────┐ ┌─────────┐ ┌───────┐
│ Payment │ │ Fraud    │ │ ML     │ │ Rule    │ │Graph  │ │ Alert   │ │ Case  │
│ Gateway │ │ Service  │ │ Engine │ │ Engine  │ │DB     │ │Generator│ │ Mgmt  │
└────┬────┘ └────┬─────┘ └───┬────┘ └────┬────┘ └───┬───┘ └────┬────┘ └───┬───┘
     │           │           │           │          │          │          │
     │ POST /evaluate        │           │          │          │          │
     │──────────▶│           │           │          │          │          │
     │           │           │           │          │          │          │
     │           │ score()   │           │          │          │          │
     │           │──────────▶│           │          │          │          │
     │           │◀──────────│           │          │          │          │
     │           │ MLScore=920 (CRITICAL)│          │          │          │
     │           │           │           │          │          │          │
     │           │ evaluate()│           │          │          │          │
     │           │───────────────────────▶│          │          │          │
     │           │◀───────────────────────│          │          │          │
     │           │ Rule FRD-NET-001 triggered       │          │          │
     │           │ (device linked to fraud account) │          │          │
     │           │           │           │          │          │          │
     │           │ queryFraudNetwork     │          │          │          │
     │           │──────────────────────────────────▶│          │          │
     │           │◀──────────────────────────────────│          │          │
     │           │ FraudNetwork found: 5 accounts   │          │          │
     │           │ via shared device + IP           │          │          │
     │           │           │           │          │          │          │
     │           │ Decision = BLOCK      │          │          │          │
     │           │ Reason = Fraud ring detected     │          │          │
     │           │           │           │          │          │          │
     │           │ createHighPriorityAlert          │          │          │
     │           │─────────────────────────────────────────────▶│          │
     │           │           │           │          │          │          │
     │           │           │           │          │ createCase(alertId)  │
     │           │           │           │          │          │─────────▶│
     │           │           │           │          │          │◀─────────│
     │           │           │           │          │          │ caseId   │
     │           │◀────────────────────────────────────────────│          │
     │           │ alertId, caseId       │          │          │          │
     │           │           │           │          │          │          │
     │           │ blockDevice(deviceId) │          │          │          │
     │           │ (async)   │           │          │          │          │
     │           │           │           │          │          │          │
     │           │ notifyFraudTeam       │          │          │          │
     │           │ (async)   │           │          │          │          │
     │           │           │           │          │          │          │
     │◀──────────│           │           │          │          │          │
     │ FraudCheckResponse    │           │          │          │          │
     │ decision=BLOCK        │           │          │          │          │
     │ score=920             │           │          │          │          │
     │ riskLevel=CRITICAL    │           │          │          │          │
     │ reasonCodes=[FRD-NET-001, FRD-ML-HIGH]       │          │          │
     │ alertId, caseId       │           │          │          │          │
```

### 8.4 Feedback Loop - Model Retraining Trigger

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FEEDBACK LOOP - FALSE POSITIVE CORRECTION                 │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────┐ ┌──────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────────────┐
│ Case    │ │ Fraud    │ │Customer │ │Feature  │ │ Kafka   │ │ ML Training     │
│ Mgmt    │ │ Service  │ │ Profile │ │ Store   │ │         │ │ Pipeline        │
└────┬────┘ └────┬─────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────────┬────────┘
     │           │            │           │           │               │
     │ POST /feedback         │           │           │               │
     │ type=FALSE_POSITIVE    │           │           │               │
     │──────────▶│            │           │           │               │
     │           │            │           │           │               │
     │           │ recordFeedback         │           │               │
     │           │───────────▶│           │           │               │
     │           │ (update fraud_checks)  │           │               │
     │           │            │           │           │               │
     │           │ updateCustomerProfile  │           │               │
     │           │───────────────────────▶│           │               │
     │           │ (increment false_positive_count)  │               │
     │           │            │           │           │               │
     │           │ adjustRiskTier         │           │               │
     │           │ (if many FPs, lower risk)         │               │
     │           │            │           │           │               │
     │           │ storeLabeled Feature   │           │               │
     │           │──────────────────────────────────▶│               │
     │           │ (for retraining)       │           │               │
     │           │            │           │           │               │
     │           │ publishFeedbackEvent   │           │               │
     │           │────────────────────────────────────▶│               │
     │           │            │           │           │               │
     │           │            │           │           │ consumeEvent  │
     │           │            │           │           │──────────────▶│
     │           │            │           │           │               │
     │           │            │           │           │     (Check    │
     │           │            │           │           │      batch    │
     │           │            │           │           │      threshol)│
     │           │            │           │           │               │
     │           │            │           │           │ If threshold  │
     │           │            │           │           │ reached:      │
     │           │            │           │           │ triggerRetrain│
     │           │            │           │           │──────────────▶│
     │           │            │           │           │               │
     │◀──────────│            │           │           │               │
     │ FeedbackRecorded       │           │           │               │
     │           │            │           │           │               │

Retraining Thresholds:
• 100+ new labeled samples
• Drift detection > 5%
• Weekly scheduled (if above minimum samples)
```

---

## 9. Integration Points

### 9.1 Internal Service Integration

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FRAUD DETECTION INTEGRATION MAP                           │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                         UPSTREAM SERVICES                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Payment Gateway Core (Payment Orchestration Domain)                  │   │
│  │                                                                      │   │
│  │ Integration: Synchronous gRPC / REST                                │   │
│  │ Endpoint: POST /fraud/v1/evaluate                                   │   │
│  │ Purpose: Real-time fraud check for payment transactions             │   │
│  │ SLA: < 150ms p99                                                    │   │
│  │ Fallback: ALLOW with logging (configurable)                         │   │
│  │                                                                      │   │
│  │ Data Exchanged:                                                     │   │
│  │ → Transaction details, customer ID, device info, session context   │   │
│  │ ← Decision, score, risk level, challenge details (if applicable)   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Validation Pipeline (Payment Orchestration Domain)                   │   │
│  │                                                                      │   │
│  │ Integration: Event-driven (Kafka)                                   │   │
│  │ Topic: payment.transactions.initiated                               │   │
│  │ Purpose: Async fraud check for batch/file-based payments           │   │
│  │                                                                      │   │
│  │ Data Exchanged:                                                     │   │
│  │ → Full transaction context                                          │   │
│  │ ← Fraud check result via fraud.check.completed topic               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Authentication Service (Platform Services)                          │   │
│  │                                                                      │   │
│  │ Integration: Kafka + REST                                           │   │
│  │ Topic: auth.login.events                                            │   │
│  │ Purpose: Device/session tracking, step-up authentication triggers  │   │
│  │                                                                      │   │
│  │ Data Exchanged:                                                     │   │
│  │ → Login events, device details, session info                       │   │
│  │ ← Step-up challenge requests                                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                         DOWNSTREAM SERVICES                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Case Management Service                                              │   │
│  │                                                                      │   │
│  │ Integration: Kafka + REST API                                       │   │
│  │ Topic Published: fraud.alert.created                                │   │
│  │ Topic Consumed: case.resolution.completed                           │   │
│  │ Purpose: Create investigation cases, receive resolution feedback   │   │
│  │                                                                      │   │
│  │ Data Exchanged:                                                     │   │
│  │ → Alert details, transaction info, investigation context           │   │
│  │ ← Resolution outcome, fraud confirmation/rejection                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Notification Service (Platform Services)                            │   │
│  │                                                                      │   │
│  │ Integration: Kafka                                                  │   │
│  │ Topics: fraud.alert.created, fraud.device.blocked                  │   │
│  │ Purpose: Alert fraud team, notify customers of suspicious activity │   │
│  │                                                                      │   │
│  │ Notifications:                                                      │   │
│  │ • SMS/Email alerts to fraud analysts                               │   │
│  │ • Push notifications to fraud ops dashboard                        │   │
│  │ • Customer notifications (when configured)                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ AML Screening Service (Risk & Compliance Domain)                    │   │
│  │                                                                      │   │
│  │ Integration: Kafka                                                  │   │
│  │ Topic Published: fraud.network.detected                             │   │
│  │ Purpose: Share fraud network insights for AML correlation          │   │
│  │                                                                      │   │
│  │ Data Exchanged:                                                     │   │
│  │ → Detected fraud rings, shared entity identifiers                  │   │
│  │ ← Enhanced screening for linked entities                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Customer Service                                                     │   │
│  │                                                                      │   │
│  │ Integration: Kafka                                                  │   │
│  │ Topic: fraud.customer.risk.updated                                  │   │
│  │ Purpose: Sync customer risk tier changes                           │   │
│  │                                                                      │   │
│  │ Data Exchanged:                                                     │   │
│  │ → Risk tier updates, restriction notifications                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Analytics Service (Platform Services)                               │   │
│  │                                                                      │   │
│  │ Integration: Kafka                                                  │   │
│  │ Topics: All fraud.* topics                                          │   │
│  │ Purpose: Fraud analytics, reporting, dashboards                    │   │
│  │                                                                      │   │
│  │ Data Exchanged:                                                     │   │
│  │ → All fraud events for analysis and reporting                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 9.2 External Service Integration

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    EXTERNAL SERVICE INTEGRATIONS                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ Device Intelligence Providers                                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Provider: ThreatMetrix / LexisNexis                                        │
│  ├── Purpose: Device fingerprinting and reputation                         │
│  ├── Integration: REST API (async enrichment)                              │
│  ├── Timeout: 500ms (cached locally)                                       │
│  ├── Fallback: Use local fingerprint with reduced accuracy                │
│  └── Data: Device ID, trust score, fraud signals, bot detection           │
│                                                                             │
│  Provider: Iovation / TransUnion                                            │
│  ├── Purpose: Device reputation and shared intelligence                    │
│  ├── Integration: REST API                                                 │
│  └── Data: Device risk score, cross-network fraud signals                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ IP Intelligence Providers                                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Provider: MaxMind GeoIP                                                    │
│  ├── Purpose: IP geolocation and VPN/proxy detection                       │
│  ├── Integration: Local database (updated daily)                           │
│  └── Data: Country, city, ISP, VPN/proxy flags, data center flags         │
│                                                                             │
│  Provider: IPQualityScore                                                   │
│  ├── Purpose: IP reputation and bot detection                              │
│  ├── Integration: REST API with caching                                    │
│  └── Data: Fraud score, bot likelihood, abuse velocity                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ Email Intelligence Providers                                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Provider: Emailage / LexisNexis                                            │
│  ├── Purpose: Email risk scoring and identity verification                 │
│  ├── Integration: REST API (batch and real-time)                           │
│  └── Data: Email age, domain risk, identity linkage                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ Consortium Data Providers                                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Provider: FICO Falcon Consortium                                           │
│  ├── Purpose: Cross-industry fraud pattern intelligence                    │
│  ├── Integration: Batch feed (daily) + real-time API                       │
│  └── Data: Consortium fraud scores, shared fraud patterns                  │
│                                                                             │
│  Provider: Early Warning Services                                           │
│  ├── Purpose: Account abuse and fraud sharing network                      │
│  ├── Integration: REST API                                                 │
│  └── Data: Account risk, historical fraud, multiple institution view      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 9.3 Configuration Domain Integration

| Configuration | Source | Update Frequency | Description |
|---------------|--------|------------------|-------------|
| Fraud Rules | Rules Repository | Real-time (event-driven) | Rule definitions and conditions |
| Velocity Limits | Limit Definitions | Near real-time | Threshold configurations per segment |
| Risk Tier Thresholds | Business Config | On-demand | Score thresholds for risk levels |
| Challenge Methods | Auth Config | On-deployment | Available authentication methods |
| Model Configuration | ML Config | On model deployment | Model parameters and feature specs |
| Country Risk Scores | Reference Data | Daily | Geographic risk weighting |

---

## 10. Security Design

### 10.1 Authentication & Authorization

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SECURITY ARCHITECTURE                                     │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ API Security                                                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Authentication:                                                            │
│  ├── Internal APIs: mTLS + Service mesh (Istio)                            │
│  ├── External APIs: OAuth 2.0 / JWT Bearer tokens                          │
│  ├── Admin APIs: OAuth 2.0 + MFA                                           │
│  └── gRPC: mTLS with certificate rotation                                  │
│                                                                             │
│  Authorization (RBAC):                                                      │
│  ├── fraud.check:read          - Read fraud check results                  │
│  ├── fraud.check:write         - Submit fraud checks (service accounts)   │
│  ├── fraud.alert:read          - View fraud alerts                        │
│  ├── fraud.alert:manage        - Assign, resolve alerts                   │
│  ├── fraud.rule:read           - View fraud rules                         │
│  ├── fraud.rule:manage         - Create, update, delete rules             │
│  ├── fraud.model:read          - View model information                   │
│  ├── fraud.model:deploy        - Deploy/retire models                     │
│  ├── fraud.device:manage       - Block/unblock devices                    │
│  ├── fraud.customer:manage     - Update customer risk tiers               │
│  ├── fraud.feedback:write      - Submit decision feedback                 │
│  └── fraud.admin:full          - Full administrative access               │
│                                                                             │
│  Role Mappings:                                                             │
│  ├── FRAUD_ANALYST: alert:read, alert:manage, feedback:write              │
│  ├── FRAUD_MANAGER: ANALYST + rule:read, customer:manage                  │
│  ├── FRAUD_ADMIN: MANAGER + rule:manage, model:deploy, device:manage      │
│  ├── ML_ENGINEER: model:read, model:deploy                                │
│  ├── SERVICE_ACCOUNT: check:write, check:read                             │
│  └── AUDITOR: *.read (all read permissions)                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 10.2 Data Protection

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Data Classification & Protection                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ HIGHLY SENSITIVE (Encryption Required)                             │    │
│  │                                                                     │    │
│  │ • Customer PII (name, address, DOB)                                │    │
│  │ • Account numbers                                                   │    │
│  │ • Device fingerprints                                              │    │
│  │ • IP addresses                                                      │    │
│  │ • Transaction details                                               │    │
│  │                                                                     │    │
│  │ Protection:                                                         │    │
│  │ ├── At-rest: AES-256 encryption                                    │    │
│  │ ├── In-transit: TLS 1.3                                            │    │
│  │ ├── In-memory: Secure enclave where available                      │    │
│  │ └── Logs: PII masking/tokenization                                 │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ SENSITIVE (Access Controlled)                                      │    │
│  │                                                                     │    │
│  │ • Fraud scores                                                      │    │
│  │ • Risk assessments                                                  │    │
│  │ • Rule configurations                                               │    │
│  │ • Model parameters                                                  │    │
│  │ • Alert details                                                     │    │
│  │                                                                     │    │
│  │ Protection:                                                         │    │
│  │ ├── Role-based access control                                      │    │
│  │ ├── Audit logging                                                  │    │
│  │ └── Data retention policies                                        │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ PII Handling                                                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Field                │ Storage          │ Logging        │ API Response   │
│  ─────────────────────┼──────────────────┼────────────────┼────────────────│
│  Customer Name        │ Encrypted        │ First initial  │ Masked         │
│  Account Number       │ Encrypted        │ Last 4 digits  │ Last 4 digits  │
│  Card Number          │ Tokenized        │ Not logged     │ Not returned   │
│  IP Address           │ Hashed           │ Anonymized     │ Country only   │
│  Device Fingerprint   │ Hashed           │ Hash only      │ Hash only      │
│  Email                │ Encrypted        │ Domain only    │ Partial mask   │
│  Phone Number         │ Encrypted        │ Last 4 digits  │ Masked         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 10.3 Fraud Detection Security Controls

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Security Controls Against Internal Threats                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Rule Changes:                                                              │
│  ├── All rule changes require approval workflow                            │
│  ├── Changes logged with user identification                               │
│  ├── Rule testing required before production deployment                    │
│  └── Automated anomaly detection on rule changes                          │
│                                                                             │
│  Model Deployment:                                                          │
│  ├── Model artifact integrity verification (hash validation)              │
│  ├── Staged rollout with traffic control                                  │
│  ├── Performance monitoring with automatic rollback                       │
│  └── Signed model artifacts                                               │
│                                                                             │
│  Whitelist/Blocklist Management:                                            │
│  ├── Dual authorization for bulk changes                                  │
│  ├── Time-limited whitelist entries                                       │
│  ├── Audit trail for all list modifications                               │
│  └── Automated review of expiring entries                                 │
│                                                                             │
│  Alert Handling:                                                            │
│  ├── Separation of duties (creator vs. resolver)                          │
│  ├── Resolution notes mandatory                                           │
│  ├── Random sampling for quality review                                   │
│  └── Pattern analysis for investigator performance                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 10.4 Audit Trail Requirements

| Event Type | Data Captured | Retention | Purpose |
|------------|---------------|-----------|---------|
| Fraud Check | Full request/response, decision, scores | 7 years | Regulatory compliance |
| Alert Creation | Transaction, alert details, triggered rules | 7 years | Investigation support |
| Alert Resolution | Resolution type, notes, resolver ID | 7 years | Quality assurance |
| Rule Changes | Before/after state, change reason, approver | 7 years | Change management |
| Model Deployment | Model version, performance metrics, deployer | 7 years | Model governance |
| Device Actions | Block/unblock, reason, authorizer | 7 years | Device management audit |
| Whitelist Changes | Entity, duration, reason, approver | 7 years | Access control audit |
| Customer Risk Changes | Previous/new tier, reason, trigger | 7 years | Customer impact audit |

---

## 11. Non-Functional Requirements

### 11.1 Performance Requirements

| Requirement ID | Category | Requirement | Target | Critical |
|----------------|----------|-------------|--------|----------|
| NFR-PERF-001 | Latency | Real-time fraud check p50 latency | ≤ 50ms | Yes |
| NFR-PERF-002 | Latency | Real-time fraud check p95 latency | ≤ 100ms | Yes |
| NFR-PERF-003 | Latency | Real-time fraud check p99 latency | ≤ 150ms | Yes |
| NFR-PERF-004 | Latency | Batch evaluation per transaction | ≤ 25ms | No |
| NFR-PERF-005 | Throughput | Peak transactions per second | 10,000 TPS | Yes |
| NFR-PERF-006 | Throughput | Sustained transactions per second | 5,000 TPS | Yes |
| NFR-PERF-007 | Throughput | Batch processing rate | 100,000/hour | No |
| NFR-PERF-008 | ML Inference | Model inference time | ≤ 20ms | Yes |
| NFR-PERF-009 | Cache | Cache hit rate for profiles | ≥ 95% | Yes |
| NFR-PERF-010 | Database | Database query time p95 | ≤ 15ms | Yes |

### 11.2 Availability & Reliability

| Requirement ID | Category | Requirement | Target |
|----------------|----------|-------------|--------|
| NFR-AVL-001 | Uptime | Service availability | 99.99% |
| NFR-AVL-002 | Recovery | RPO (Recovery Point Objective) | ≤ 1 minute |
| NFR-AVL-003 | Recovery | RTO (Recovery Time Objective) | ≤ 5 minutes |
| NFR-AVL-004 | Failover | Active-active datacenter support | Required |
| NFR-AVL-005 | Graceful Degradation | Continue with default decision on failure | Required |
| NFR-REL-001 | Error Rate | Error rate (5xx responses) | < 0.01% |
| NFR-REL-002 | Error Rate | Circuit breaker activation | < 0.1% triggers |
| NFR-REL-003 | Data Integrity | Fraud check data consistency | 100% |
| NFR-REL-004 | Message Delivery | Kafka message reliability | At-least-once |

### 11.3 Scalability

| Requirement ID | Dimension | Requirement | Specification |
|----------------|-----------|-------------|---------------|
| NFR-SCL-001 | Horizontal | Pod scaling capability | 3-50 pods |
| NFR-SCL-002 | Horizontal | Scale-up trigger | CPU > 70% or memory > 75% |
| NFR-SCL-003 | Horizontal | Scale-up time | ≤ 30 seconds |
| NFR-SCL-004 | Vertical | Max pod resources | 8 CPU, 16GB RAM |
| NFR-SCL-005 | Database | Read replica count | 3+ |
| NFR-SCL-006 | Cache | Redis cluster nodes | 6+ (3 primary, 3 replica) |
| NFR-SCL-007 | Graph DB | Neo4j cluster | 3-node causal cluster |
| NFR-SCL-008 | Event Stream | Kafka partition count | 16+ per topic |

### 11.4 Data Management

| Requirement ID | Category | Requirement | Specification |
|----------------|----------|-------------|---------------|
| NFR-DAT-001 | Retention | Fraud check records | 7 years |
| NFR-DAT-002 | Retention | Device fingerprints | 3 years |
| NFR-DAT-003 | Retention | Customer profiles | Active + 7 years |
| NFR-DAT-004 | Retention | ML model artifacts | Indefinite |
| NFR-DAT-005 | Archival | Old records to cold storage | > 1 year |
| NFR-DAT-006 | Purge | PII data on request (GDPR) | ≤ 72 hours |
| NFR-DAT-007 | Backup | Full database backup | Daily |
| NFR-DAT-008 | Backup | Incremental backup | Hourly |

---

## 12. Deployment Architecture

### 12.1 Kubernetes Deployment

```yaml
# fraud-detection-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fraud-detection-service
  namespace: risk-compliance
  labels:
    app: fraud-detection
    version: v1
    tier: backend
spec:
  replicas: 5
  selector:
    matchLabels:
      app: fraud-detection
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: fraud-detection
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - fraud-detection
                topologyKey: "topology.kubernetes.io/zone"
      containers:
        - name: fraud-detection
          image: registry.internal/fraud-detection:v1.0.0
          ports:
            - name: http
              containerPort: 8004
            - name: grpc
              containerPort: 50054
            - name: metrics
              containerPort: 9090
          env:
            - name: ENVIRONMENT
              valueFrom:
                configMapKeyRef:
                  name: fraud-detection-config
                  key: environment
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: fraud-detection-secrets
                  key: db-host
            - name: REDIS_CLUSTER
              valueFrom:
                secretKeyRef:
                  name: fraud-detection-secrets
                  key: redis-cluster
            - name: KAFKA_BOOTSTRAP
              valueFrom:
                configMapKeyRef:
                  name: fraud-detection-config
                  key: kafka-bootstrap
          resources:
            requests:
              cpu: "2"
              memory: "4Gi"
            limits:
              cpu: "8"
              memory: "16Gi"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8004
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8004
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /health/started
              port: 8004
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 30
          volumeMounts:
            - name: ml-models
              mountPath: /app/models
              readOnly: true
            - name: config
              mountPath: /app/config
              readOnly: true
      volumes:
        - name: ml-models
          persistentVolumeClaim:
            claimName: fraud-ml-models-pvc
        - name: config
          configMap:
            name: fraud-detection-config
```

### 12.2 Horizontal Pod Autoscaler

```yaml
# fraud-detection-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fraud-detection-hpa
  namespace: risk-compliance
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: fraud-detection-service
  minReplicas: 5
  maxReplicas: 50
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
          name: fraud_check_queue_depth
        target:
          type: AverageValue
          averageValue: "100"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
        - type: Pods
          value: 4
          periodSeconds: 15
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

### 12.3 Service Mesh Configuration

```yaml
# fraud-detection-istio.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: fraud-detection
  namespace: risk-compliance
spec:
  hosts:
    - fraud-detection
  http:
    - match:
        - headers:
            x-api-version:
              exact: "v2"
      route:
        - destination:
            host: fraud-detection
            subset: v2
          weight: 100
    - route:
        - destination:
            host: fraud-detection
            subset: v1
          weight: 90
        - destination:
            host: fraud-detection
            subset: v2
          weight: 10
  retries:
    attempts: 3
    perTryTimeout: 100ms
    retryOn: 5xx,reset,connect-failure

---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: fraud-detection
  namespace: risk-compliance
spec:
  host: fraud-detection
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1000
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 1000
        http2MaxRequests: 2000
        maxRequestsPerConnection: 100
    loadBalancer:
      simple: LEAST_CONN
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

### 12.4 Infrastructure Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FRAUD DETECTION INFRASTRUCTURE                            │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                              REGION A (Primary)                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Kubernetes Cluster                                                   │   │
│  │ ├── fraud-detection-service (5-25 pods)                             │   │
│  │ ├── ml-inference-service (3-10 pods)                                │   │
│  │ └── fraud-worker-service (3-15 pods)                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐   │
│  │ PostgreSQL Primary │  │ Redis Cluster      │  │ Neo4j Primary      │   │
│  │ (fraud_db)         │  │ (6 nodes)          │  │ (3-node cluster)   │   │
│  │ • Write operations │  │ • Session cache    │  │ • Graph traversal  │   │
│  │ • 2 sync replicas  │  │ • Profile cache    │  │ • Network analysis │   │
│  └────────────────────┘  └────────────────────┘  └────────────────────┘   │
│                                                                             │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ Kafka Cluster (shared)                                              │    │
│  │ • 6 brokers                                                         │    │
│  │ • Fraud topics: 16 partitions each                                  │    │
│  │ • Replication factor: 3                                             │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Elasticsearch Cluster                                                │   │
│  │ • 3 master nodes                                                     │   │
│  │ • 6 data nodes                                                       │   │
│  │ • fraud_* indices                                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                            REGION B (DR/Active)                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Kubernetes Cluster (Mirror)                                          │   │
│  │ ├── fraud-detection-service (3-15 pods)                             │   │
│  │ ├── ml-inference-service (2-5 pods)                                 │   │
│  │ └── fraud-worker-service (2-8 pods)                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐   │
│  │ PostgreSQL Replica │  │ Redis Cluster      │  │ Neo4j Read Replica │   │
│  │ (async from A)     │  │ (6 nodes)          │  │ (async from A)     │   │
│  │ • Read operations  │  │ • Local cache      │  │ • Graph queries    │   │
│  │ • Promotion ready  │  │ • Independent      │  │ • Promotion ready  │   │
│  └────────────────────┘  └────────────────────┘  └────────────────────┘   │
│                                                                             │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ Kafka MirrorMaker 2                                                 │    │
│  │ • Active-active replication                                         │    │
│  │ • Fraud topics mirrored                                             │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                          SHARED SERVICES                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐   │
│  │ S3/Azure Blob      │  │ MLflow Registry    │  │ Vault              │   │
│  │ • Model artifacts  │  │ • Model versions   │  │ • Secrets mgmt     │   │
│  │ • Training data    │  │ • Experiment track │  │ • DB credentials   │   │
│  │ • Audit logs       │  │ • Feature specs    │  │ • API keys         │   │
│  └────────────────────┘  └────────────────────┘  └────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 12.5 Model Deployment Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       ML MODEL DEPLOYMENT PIPELINE                           │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐           │
│    │          │    │          │    │          │    │          │           │
│    │  Model   │───▶│  Model   │───▶│ Staging  │───▶│  Shadow  │           │
│    │ Training │    │ Registry │    │  Deploy  │    │   Mode   │           │
│    │          │    │          │    │          │    │          │           │
│    └──────────┘    └──────────┘    └──────────┘    └──────────┘           │
│         │              │               │               │                   │
│         ▼              ▼               ▼               ▼                   │
│    - Feature eng  - Version tag   - Canary 5%    - Compare vs            │
│    - Training     - Artifact hash - Monitor      - Champion model        │
│    - Validation   - Sign artifact - A/B metrics  - No decisions          │
│    - Unit tests   - Approval gate - Rollback     - Log scores            │
│                                                                             │
│    ┌──────────┐    ┌──────────┐    ┌──────────┐                           │
│    │          │    │          │    │          │                           │
│    │  Canary  │───▶│ Gradual  │───▶│  Full    │                           │
│    │  Release │    │ Rollout  │    │  Deploy  │                           │
│    │          │    │          │    │          │                           │
│    └──────────┘    └──────────┘    └──────────┘                           │
│         │              │               │                                   │
│         ▼              ▼               ▼                                   │
│    - 10% traffic  - 25/50/75%    - 100% traffic                          │
│    - Monitor KPIs - Auto-pause   - Archive old                           │
│    - Error rate   - Performance  - Update config                         │
│    - Rollback     - gates        - Notify teams                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

Model Deployment Gates:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Gate 1: Quality                                                            │
│  ├── AUC-ROC > 0.85                                                        │
│  ├── Precision at 5% FPR > 70%                                             │
│  ├── Recall at target threshold > 85%                                      │
│  └── No significant bias detected                                          │
│                                                                             │
│  Gate 2: Performance                                                        │
│  ├── Inference time p99 < 20ms                                             │
│  ├── Memory footprint < 2GB                                                │
│  └── CPU utilization stable                                                │
│                                                                             │
│  Gate 3: Safety                                                             │
│  ├── No increase in false positives > 5%                                   │
│  ├── No increase in false negatives > 2%                                   │
│  └── Approval from ML lead                                                 │
│                                                                             │
│  Gate 4: Production                                                         │
│  ├── Canary success for 24 hours                                           │
│  ├── No anomalies in business metrics                                      │
│  └── Rollback plan verified                                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 13. Monitoring & Observability

### 13.1 Key Metrics

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         FRAUD DETECTION METRICS                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ Business Metrics (Golden Signals)                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Fraud Detection Rate                                                       │
│  ├── fraud_detection_total{decision="BLOCK|CHALLENGE|ALLOW"}               │
│  ├── fraud_detection_rate{type="confirmed_fraud"}                          │
│  ├── fraud_prevention_amount_total                                         │
│  └── fraud_loss_amount_total                                               │
│                                                                             │
│  False Positive/Negative Rate                                               │
│  ├── fraud_false_positive_rate                                             │
│  ├── fraud_false_negative_rate                                             │
│  ├── fraud_precision                                                       │
│  └── fraud_recall                                                          │
│                                                                             │
│  Customer Impact                                                            │
│  ├── fraud_challenge_success_rate                                          │
│  ├── fraud_customer_friction_score                                         │
│  └── fraud_legitimate_transaction_blocked_rate                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ Technical Metrics                                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Latency                                                                    │
│  ├── fraud_check_duration_seconds{quantile="0.5|0.95|0.99"}               │
│  ├── ml_inference_duration_seconds{model="*"}                              │
│  ├── rule_evaluation_duration_seconds                                      │
│  └── external_api_duration_seconds{provider="*"}                           │
│                                                                             │
│  Throughput                                                                 │
│  ├── fraud_checks_total                                                    │
│  ├── fraud_checks_per_second                                               │
│  ├── batch_evaluations_total                                               │
│  └── events_processed_total{topic="*"}                                     │
│                                                                             │
│  Error Rate                                                                 │
│  ├── fraud_check_errors_total{type="*"}                                   │
│  ├── external_api_errors_total{provider="*"}                              │
│  └── kafka_consumer_errors_total                                           │
│                                                                             │
│  Saturation                                                                 │
│  ├── fraud_service_queue_depth                                             │
│  ├── db_connection_pool_usage                                              │
│  ├── redis_memory_usage_percent                                            │
│  └── kafka_consumer_lag{topic="*", partition="*"}                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ ML Model Metrics                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Model Performance                                                          │
│  ├── ml_model_auc_roc{model="*", version="*"}                             │
│  ├── ml_model_precision{model="*", threshold="*"}                         │
│  ├── ml_model_recall{model="*", threshold="*"}                            │
│  └── ml_model_f1_score{model="*"}                                         │
│                                                                             │
│  Feature Health                                                             │
│  ├── ml_feature_missing_rate{feature="*"}                                 │
│  ├── ml_feature_drift_score{feature="*"}                                  │
│  └── ml_feature_cardinality{feature="*"}                                  │
│                                                                             │
│  Inference                                                                  │
│  ├── ml_inference_count{model="*"}                                        │
│  ├── ml_score_distribution{model="*", bucket="*"}                         │
│  └── ml_model_staleness_hours{model="*"}                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 13.2 Service Level Objectives (SLOs)

| SLO ID | Name | Indicator | Target | Measurement Window |
|--------|------|-----------|--------|-------------------|
| SLO-001 | Availability | Successful responses / Total requests | 99.99% | Rolling 30 days |
| SLO-002 | Latency P50 | 50th percentile response time | ≤ 50ms | Rolling 24 hours |
| SLO-003 | Latency P99 | 99th percentile response time | ≤ 150ms | Rolling 24 hours |
| SLO-004 | Error Rate | 5xx errors / Total requests | < 0.01% | Rolling 24 hours |
| SLO-005 | Detection Rate | Fraud caught / Total confirmed fraud | ≥ 95% | Rolling 7 days |
| SLO-006 | False Positive Rate | False blocks / Total blocks | < 5% | Rolling 7 days |
| SLO-007 | Model Accuracy | AUC-ROC score | ≥ 0.85 | Per model version |

### 13.3 Alerting Rules

```yaml
# fraud-detection-alerts.yaml
groups:
  - name: fraud-detection-critical
    rules:
      - alert: FraudServiceDown
        expr: up{job="fraud-detection"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Fraud Detection Service is down"
          description: "Fraud Detection Service instance {{ $labels.instance }} is down"

      - alert: FraudCheckLatencyHigh
        expr: histogram_quantile(0.99, fraud_check_duration_seconds) > 0.2
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Fraud check latency exceeds SLO"
          description: "P99 fraud check latency is {{ $value }}s, target is 0.15s"

      - alert: FraudCheckErrorRateHigh
        expr: rate(fraud_check_errors_total[5m]) / rate(fraud_checks_total[5m]) > 0.001
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Fraud check error rate exceeds threshold"
          description: "Error rate is {{ $value | humanizePercentage }}, threshold is 0.1%"

      - alert: FraudDetectionRateDrop
        expr: fraud_detection_rate{decision="BLOCK"} < 0.02
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Fraud block rate dropped significantly"
          description: "Block rate is {{ $value | humanizePercentage }}, may indicate model issue"

      - alert: FraudFalsePositiveRateHigh
        expr: fraud_false_positive_rate > 0.10
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "False positive rate exceeds threshold"
          description: "False positive rate is {{ $value | humanizePercentage }}, threshold is 10%"

  - name: fraud-detection-ml
    rules:
      - alert: MLModelInferenceSlow
        expr: histogram_quantile(0.99, ml_inference_duration_seconds) > 0.025
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "ML model inference is slow"
          description: "Model {{ $labels.model }} P99 inference time is {{ $value }}s"

      - alert: MLFeatureDriftDetected
        expr: ml_feature_drift_score > 0.1
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Feature drift detected"
          description: "Feature {{ $labels.feature }} drift score is {{ $value }}"

      - alert: MLModelPerformanceDegraded
        expr: ml_model_auc_roc < 0.80
        for: 4h
        labels:
          severity: warning
        annotations:
          summary: "ML model performance below threshold"
          description: "Model {{ $labels.model }} AUC-ROC is {{ $value }}, threshold is 0.85"

  - name: fraud-detection-infrastructure
    rules:
      - alert: KafkaConsumerLagHigh
        expr: kafka_consumer_lag > 10000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Kafka consumer lag is high"
          description: "Consumer lag for {{ $labels.topic }} is {{ $value }}"

      - alert: RedisMemoryHigh
        expr: redis_memory_usage_percent > 85
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Redis memory usage is high"
          description: "Redis memory usage is {{ $value }}%"

      - alert: DatabaseConnectionPoolExhausted
        expr: db_connection_pool_usage > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Database connection pool near exhaustion"
          description: "Connection pool usage is {{ $value }}%"
```

### 13.4 Dashboards

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      FRAUD DETECTION DASHBOARDS                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ Dashboard 1: Operations Overview                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Row 1: Key Indicators                                                      │
│  ├── Total Transactions Evaluated (today)                                  │
│  ├── Fraud Detected ($$ amount)                                            │
│  ├── Block Rate (%)                                                        │
│  └── False Positive Rate (%)                                               │
│                                                                             │
│  Row 2: Decision Breakdown                                                  │
│  ├── Decisions by Type (ALLOW/CHALLENGE/BLOCK) - Pie chart               │
│  ├── Decisions Over Time - Stacked area chart                             │
│  └── Challenge Success Rate - Line chart                                  │
│                                                                             │
│  Row 3: Performance                                                         │
│  ├── Latency Distribution (P50/P95/P99) - Heatmap                         │
│  ├── Throughput (TPS) - Line chart                                        │
│  └── Error Rate - Line chart                                              │
│                                                                             │
│  Row 4: Alerts & Investigations                                            │
│  ├── Open Alerts by Priority - Bar chart                                  │
│  ├── Alert Resolution Time - Histogram                                    │
│  └── Investigation Queue Depth - Gauge                                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ Dashboard 2: ML Model Performance                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Row 1: Model Health                                                        │
│  ├── Active Models by Type                                                 │
│  ├── Model AUC-ROC Trend - Line chart                                     │
│  └── Inference Volume by Model - Stacked bar                              │
│                                                                             │
│  Row 2: Prediction Quality                                                  │
│  ├── Score Distribution - Histogram                                       │
│  ├── Precision/Recall Trend - Dual-axis line                              │
│  └── Confusion Matrix (last 24h) - Heatmap                                │
│                                                                             │
│  Row 3: Feature Health                                                      │
│  ├── Feature Missing Rates - Table                                        │
│  ├── Feature Drift Scores - Bar chart                                     │
│  └── Feature Importance - Horizontal bar                                  │
│                                                                             │
│  Row 4: Model Comparison                                                    │
│  ├── Champion vs Challenger Performance                                    │
│  ├── A/B Test Results                                                      │
│  └── Model Version History                                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ Dashboard 3: Fraud Analytics                                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Row 1: Fraud Trends                                                        │
│  ├── Fraud by Type (Card-Not-Present, ATO, etc.) - Pie chart             │
│  ├── Fraud by Channel - Bar chart                                         │
│  └── Fraud by Geography - Map                                             │
│                                                                             │
│  Row 2: Attack Patterns                                                     │
│  ├── Top Triggered Rules - Table                                          │
│  ├── Velocity Pattern Anomalies - Heatmap                                 │
│  └── Device Risk Distribution - Histogram                                 │
│                                                                             │
│  Row 3: Customer Risk                                                       │
│  ├── Risk Tier Distribution - Donut chart                                 │
│  ├── Risk Tier Changes Over Time - Stacked area                           │
│  └── High-Risk Customer Segments                                          │
│                                                                             │
│  Row 4: Network Analysis                                                    │
│  ├── Connected Entities Graph - Network viz                              │
│  ├── Fraud Ring Detections (this week)                                    │
│  └── Beneficiary Risk Clusters                                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 14. Test Strategy

### 14.1 Test Pyramid

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         TEST PYRAMID                                         │
└─────────────────────────────────────────────────────────────────────────────┘

                              ┌───────────┐
                             /│   E2E     │\
                            / │  Tests    │ \
                           /  │  (5%)     │  \
                          /   └───────────┘   \
                         /                     \
                        /    ┌───────────────┐  \
                       /     │  Integration  │   \
                      /      │    Tests      │    \
                     /       │    (25%)      │     \
                    /        └───────────────┘      \
                   /                                 \
                  /       ┌───────────────────┐      \
                 /        │    Unit Tests     │       \
                /         │      (70%)        │        \
               /          └───────────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│ Test Categories                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Unit Tests (~70%)                                                          │
│  ├── Domain logic (fraud score calculation, rule evaluation)              │
│  ├── Value object validation                                               │
│  ├── ML feature extraction                                                 │
│  ├── Velocity counter logic                                                │
│  └── Decision engine logic                                                 │
│                                                                             │
│  Integration Tests (~25%)                                                   │
│  ├── API endpoint testing (REST, gRPC)                                     │
│  ├── Database operations (CRUD, queries)                                   │
│  ├── Redis cache operations                                                │
│  ├── Kafka producer/consumer                                               │
│  ├── External API mocks                                                    │
│  └── ML model loading & inference                                          │
│                                                                             │
│  E2E Tests (~5%)                                                            │
│  ├── Full fraud check flow                                                 │
│  ├── Step-up authentication flow                                           │
│  ├── Alert creation and routing                                            │
│  └── Multi-service integration                                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 14.2 Test Scenarios

| Category | Test Case | Description | Priority |
|----------|-----------|-------------|----------|
| Core Flow | Basic fraud check | Valid transaction returns ALLOW decision | P0 |
| Core Flow | Block fraudulent transaction | Known fraud pattern returns BLOCK | P0 |
| Core Flow | Challenge medium risk | Medium risk returns CHALLENGE with auth method | P0 |
| ML Models | Model inference | All models return valid scores | P0 |
| ML Models | Model fallback | Service degrades gracefully when model fails | P1 |
| ML Models | Feature extraction | All features extracted correctly from transaction | P1 |
| Rules Engine | Rule evaluation | Rules evaluate correctly against transactions | P0 |
| Rules Engine | Rule priority | Higher priority rules override lower | P1 |
| Velocity | Counter increment | Velocity counters update correctly | P0 |
| Velocity | Window expiration | Counters reset after time window | P1 |
| Device | Fingerprint validation | Device fingerprint is validated/registered | P1 |
| Device | Blocked device | Blocked device returns BLOCK decision | P0 |
| Network | Graph query | Related entities retrieved from graph | P2 |
| Network | Fraud ring detection | Connected fraud entities are identified | P2 |
| Performance | Latency under load | P99 < 150ms at 5000 TPS | P0 |
| Performance | Throughput peak | System handles 10,000 TPS | P1 |
| Resilience | Database failure | Service degrades gracefully | P1 |
| Resilience | Cache failure | Service continues with DB fallback | P1 |
| Resilience | Kafka failure | Events queued locally until recovery | P1 |

### 14.3 Performance Testing

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PERFORMANCE TEST SCENARIOS                                │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ Load Test Configuration                                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Test Type: Sustained Load                                                  │
│  ├── Target TPS: 5,000                                                     │
│  ├── Duration: 60 minutes                                                  │
│  ├── Ramp-up: 10 minutes                                                   │
│  └── Success Criteria:                                                     │
│      • P99 latency < 150ms                                                 │
│      • Error rate < 0.01%                                                  │
│      • No memory leaks                                                     │
│                                                                             │
│  Test Type: Spike Test                                                      │
│  ├── Base TPS: 2,000                                                       │
│  ├── Spike TPS: 10,000                                                     │
│  ├── Spike Duration: 5 minutes                                             │
│  └── Success Criteria:                                                     │
│      • System handles spike without failure                                │
│      • Recovery to normal latency < 2 minutes                             │
│      • No dropped requests                                                 │
│                                                                             │
│  Test Type: Soak Test                                                       │
│  ├── Target TPS: 3,000                                                     │
│  ├── Duration: 24 hours                                                    │
│  └── Success Criteria:                                                     │
│      • No performance degradation                                          │
│      • Memory usage stable                                                 │
│      • No connection leaks                                                 │
│                                                                             │
│  Test Type: Capacity Test                                                   │
│  ├── Start TPS: 1,000                                                      │
│  ├── Increment: 1,000 TPS every 5 minutes                                 │
│  ├── Until: Error rate > 1% or P99 > 500ms                                │
│  └── Success Criteria:                                                     │
│      • Identify max capacity                                               │
│      • Document breaking point                                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

Performance Test Data Profile:
┌─────────────────────────────────────────────────────────────────────────────┐
│  Transaction Mix                                                            │
│  ├── Low Risk (ALLOW): 85%                                                 │
│  ├── Medium Risk (CHALLENGE): 10%                                          │
│  ├── High Risk (BLOCK): 4%                                                 │
│  └── Known Fraud: 1%                                                       │
│                                                                             │
│  Customer Profile Mix                                                       │
│  ├── New Customers (no history): 10%                                       │
│  ├── Established (1-50 transactions): 60%                                  │
│  └── High Volume (50+ transactions): 30%                                   │
│                                                                             │
│  Device Mix                                                                 │
│  ├── Known Trusted: 70%                                                    │
│  ├── Known Untrusted: 5%                                                   │
│  └── New Device: 25%                                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 14.4 Chaos Engineering

| Chaos Experiment | Hypothesis | Method | Recovery Expectation |
|------------------|------------|--------|---------------------|
| Pod Kill | Service remains available | Randomly kill fraud-detection pods | New pods spawn < 30s, no user impact |
| Network Partition | Service degrades gracefully | Isolate DB from service pods | Cache serves requests, errors logged |
| Redis Failure | Service continues with DB | Kill Redis cluster | Fallback to DB, latency increase < 200ms |
| Model Server Failure | Default decision applied | Kill ML inference pods | ALLOW with logging, alert fired |
| CPU Exhaustion | System throttles gracefully | Inject CPU stress | HPA scales up, latency increases temporarily |
| Memory Pressure | OOM handled without data loss | Inject memory stress | Pod restarts, transactions retried |
| Kafka Partition Loss | Events buffered locally | Kill Kafka brokers | Local buffer, events published on recovery |
| Zone Failure | Traffic reroutes to other zones | Simulate availability zone failure | Traffic shifts, no user-visible impact |

---

## 15. Implementation Roadmap

### 15.1 Delivery Phases

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FRAUD DETECTION IMPLEMENTATION ROADMAP                    │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ PHASE 1: FOUNDATION (Months 1-3)                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Sprint 1-2: Core Service Setup                                             │
│  ├── Project scaffolding and CI/CD pipeline                                │
│  ├── Database schema creation (PostgreSQL)                                 │
│  ├── Redis cluster setup and connection pooling                            │
│  ├── Basic API endpoints (health, fraud check)                             │
│  └── Kafka producer/consumer setup                                         │
│                                                                             │
│  Sprint 3-4: Basic Rule Engine                                              │
│  ├── Rule definition schema and CRUD operations                            │
│  ├── Simple rule evaluation logic                                          │
│  ├── Velocity counter implementation                                       │
│  ├── Blocklist/whitelist functionality                                     │
│  └── Basic decision engine (rules-only)                                    │
│                                                                             │
│  Sprint 5-6: Device Tracking                                                │
│  ├── Device fingerprinting integration                                     │
│  ├── Device registration and history                                       │
│  ├── Device risk scoring (basic)                                           │
│  └── Device block/trust functionality                                      │
│                                                                             │
│  Milestone 1: Rules-based fraud detection operational                      │
│  Success Criteria:                                                         │
│  ├── < 200ms p99 latency                                                   │
│  ├── Basic rules evaluation working                                        │
│  ├── Velocity checks for amount/count                                      │
│  └── Device registration and blocking                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ PHASE 2: ML INTEGRATION (Months 4-6)                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Sprint 7-8: ML Infrastructure                                              │
│  ├── MLflow setup for model registry                                       │
│  ├── Model serving infrastructure (inference service)                      │
│  ├── Feature extraction pipeline                                           │
│  ├── Model A/B testing framework                                           │
│  └── Shadow mode for model evaluation                                      │
│                                                                             │
│  Sprint 9-10: Initial ML Models                                             │
│  ├── Transaction scoring model (v1)                                        │
│  ├── Behavioral anomaly model                                              │
│  ├── Model ensemble framework                                              │
│  ├── Feature store integration                                             │
│  └── Champion/challenger deployment                                        │
│                                                                             │
│  Sprint 11-12: Decision Engine Enhancement                                  │
│  ├── Hybrid decision engine (rules + ML)                                   │
│  ├── Score blending logic                                                  │
│  ├── Dynamic threshold configuration                                       │
│  ├── Segment-based decisioning                                             │
│  └── Decision explanation generation                                       │
│                                                                             │
│  Milestone 2: ML-powered fraud detection                                   │
│  Success Criteria:                                                         │
│  ├── < 150ms p99 latency with ML                                           │
│  ├── AUC-ROC > 0.85 for main model                                         │
│  ├── 10% improvement over rules-only                                       │
│  └── Automated model deployment pipeline                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ PHASE 3: ADVANCED CAPABILITIES (Months 7-9)                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Sprint 13-14: Step-Up Authentication                                       │
│  ├── Challenge type selection logic                                        │
│  ├── Authentication service integration                                    │
│  ├── Challenge outcome processing                                          │
│  ├── Challenge effectiveness tracking                                      │
│  └── OTP/biometric/security question flows                                │
│                                                                             │
│  Sprint 15-16: Network Analysis (Neo4j)                                     │
│  ├── Graph database setup and schema                                       │
│  ├── Entity relationship ingestion                                         │
│  ├── Fraud ring detection algorithms                                       │
│  ├── Link analysis queries                                                 │
│  └── Graph-based risk scoring                                              │
│                                                                             │
│  Sprint 17-18: Case Management Integration                                  │
│  ├── Alert routing to case management                                      │
│  ├── Investigation workflow triggers                                       │
│  ├── Feedback loop implementation                                          │
│  ├── Model retraining pipeline                                             │
│  └── Resolution outcome ingestion                                          │
│                                                                             │
│  Milestone 3: Full fraud ecosystem                                         │
│  Success Criteria:                                                         │
│  ├── Step-up auth reducing false positives by 30%                          │
│  ├── Network analysis detecting 5+ fraud rings/month                       │
│  ├── Closed-loop feedback operational                                      │
│  └── < 100ms p99 latency maintained                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ PHASE 4: OPTIMIZATION & SCALE (Months 10-12)                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Sprint 19-20: Performance Optimization                                     │
│  ├── Caching strategy optimization                                         │
│  ├── Database query tuning                                                 │
│  ├── ML inference optimization (model compression)                         │
│  ├── Connection pooling tuning                                             │
│  └── Async processing improvements                                         │
│                                                                             │
│  Sprint 21-22: Advanced ML Models                                           │
│  ├── Deep learning models for pattern detection                            │
│  ├── Real-time feature engineering                                         │
│  ├── Online learning capabilities                                          │
│  ├── Explainable AI integration                                            │
│  └── Automated feature selection                                           │
│                                                                             │
│  Sprint 23-24: Operational Excellence                                       │
│  ├── Advanced monitoring dashboards                                        │
│  ├── Self-healing capabilities                                             │
│  ├── Chaos engineering exercises                                           │
│  ├── Documentation and runbooks                                            │
│  └── DR testing and validation                                             │
│                                                                             │
│  Milestone 4: Production-ready at scale                                    │
│  Success Criteria:                                                         │
│  ├── 10,000 TPS capacity                                                   │
│  ├── 99.99% availability                                                   │
│  ├── < 100ms p99 latency                                                   │
│  ├── Full DR capability tested                                             │
│  └── Fraud detection rate > 95%                                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 15.2 Dependencies

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DEPENDENCY MATRIX                                    │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ Platform Dependencies                                                        │
├───────────────────────────────────────┬─────────────────────────────────────┤
│ Dependency                            │ Required By Phase                   │
├───────────────────────────────────────┼─────────────────────────────────────┤
│ Kubernetes cluster (3 nodes min)      │ Phase 1, Sprint 1                   │
│ PostgreSQL 14+ (HA setup)             │ Phase 1, Sprint 1                   │
│ Redis Cluster 7+                      │ Phase 1, Sprint 1                   │
│ Kafka cluster (3+ brokers)            │ Phase 1, Sprint 2                   │
│ Neo4j 5+ (causal cluster)             │ Phase 3, Sprint 15                  │
│ Elasticsearch 8+ (3 nodes min)        │ Phase 2, Sprint 7                   │
│ MLflow server                         │ Phase 2, Sprint 7                   │
│ Object storage (S3/Azure Blob)        │ Phase 1, Sprint 1                   │
│ HashiCorp Vault                       │ Phase 1, Sprint 1                   │
│ Istio service mesh                    │ Phase 1, Sprint 2                   │
└───────────────────────────────────────┴─────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ Service Dependencies                                                         │
├───────────────────────────────────────┬─────────────────────────────────────┤
│ Dependency                            │ Integration Point                   │
├───────────────────────────────────────┼─────────────────────────────────────┤
│ Payment Gateway Core                  │ Phase 1, Sprint 3 (sync API)        │
│ Validation Pipeline                   │ Phase 1, Sprint 4 (event-driven)    │
│ Authentication Service                │ Phase 3, Sprint 13 (step-up auth)   │
│ Case Management Service               │ Phase 3, Sprint 17 (alerts)         │
│ Customer Service                      │ Phase 2, Sprint 9 (profile data)    │
│ Notification Service                  │ Phase 1, Sprint 5 (alerts)          │
│ AML Screening Service                 │ Phase 3, Sprint 17 (cross-ref)      │
└───────────────────────────────────────┴─────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ External Vendor Dependencies                                                 │
├───────────────────────────────────────┬─────────────────────────────────────┤
│ Dependency                            │ Required By Phase                   │
├───────────────────────────────────────┼─────────────────────────────────────┤
│ ThreatMetrix/Device Intelligence      │ Phase 1, Sprint 5 (optional)        │
│ MaxMind GeoIP Database                │ Phase 1, Sprint 3                   │
│ IP Intelligence Provider              │ Phase 2, Sprint 8 (optional)        │
│ Consortium Data Provider              │ Phase 4, Sprint 21 (optional)       │
└───────────────────────────────────────┴─────────────────────────────────────┘
```

### 15.3 Resource Requirements

| Phase | Backend Engineers | ML Engineers | DevOps | QA | Total |
|-------|-------------------|--------------|--------|-----|-------|
| Phase 1 | 3 | 0.5 | 1 | 1 | 5.5 |
| Phase 2 | 2 | 2 | 1 | 1.5 | 6.5 |
| Phase 3 | 3 | 1.5 | 0.5 | 1.5 | 6.5 |
| Phase 4 | 2 | 2 | 1 | 1 | 6 |

### 15.4 Risk Register

| Risk ID | Risk | Likelihood | Impact | Mitigation |
|---------|------|------------|--------|------------|
| R001 | ML model performance below target | Medium | High | Shadow mode testing, champion/challenger |
| R002 | Latency SLO breach | Medium | Critical | Aggressive caching, async processing |
| R003 | External vendor unavailability | Low | High | Local fallback, multi-vendor strategy |
| R004 | Data quality issues | Medium | Medium | Data validation, monitoring, alerting |
| R005 | Integration delays with upstream | Medium | High | Early integration testing, mock services |
| R006 | Neo4j scaling challenges | Low | Medium | Query optimization, read replicas |
| R007 | False positive rate too high | Medium | High | Dynamic thresholds, step-up auth |
| R008 | Model drift undetected | Medium | High | Drift monitoring, automated retraining |
| R009 | Security vulnerabilities | Low | Critical | Security reviews, pen testing, WAF |
| R010 | Key personnel unavailability | Medium | Medium | Knowledge sharing, documentation |

---

## 16. Appendix

### 16.1 Glossary

| Term | Definition |
|------|------------|
| ATO | Account Takeover - unauthorized access to customer account |
| AUC-ROC | Area Under Receiver Operating Characteristic Curve - model performance metric |
| CNP | Card-Not-Present - transaction where card is not physically present |
| Champion Model | Current production ML model |
| Challenger Model | New model being evaluated against champion |
| Device Fingerprint | Unique identifier for a device based on characteristics |
| False Positive | Legitimate transaction incorrectly flagged as fraud |
| False Negative | Fraudulent transaction incorrectly allowed |
| Feature | Input variable used by ML model for prediction |
| Fraud Ring | Group of connected fraudulent entities |
| Step-Up Auth | Additional authentication challenge for risky transactions |
| Velocity | Rate of transactions over a time window |
| SHAP | SHapley Additive exPlanations - model interpretability method |

### 16.2 References

| Document | Location |
|----------|----------|
| ADR-005a: Payment Orchestration Domain | `/architecture/decisions/ADR-005a.md` |
| AML Screening Design | `/Risk&ComplianceDomain/Design_AML_Screening.md` |
| Payment Gateway Context Overview | `/PaymentGateway_Context_Overview.md` |
| API Standards Guide | `/standards/API_Standards.md` |
| Event Schema Standards | `/standards/Event_Schema_Standards.md` |
| Security Guidelines | `/security/Security_Guidelines.md` |

### 16.3 Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | 2024-01-15 | Architecture Team | Initial draft |
| 0.2 | 2024-02-01 | Architecture Team | Added ML model architecture |
| 0.3 | 2024-02-15 | Architecture Team | Added network analysis design |
| 1.0 | 2024-03-01 | Architecture Team | First release |

---

**Document Status**: APPROVED  
**Next Review Date**: 2024-09-01  
**Owner**: Risk & Compliance Architecture Team
