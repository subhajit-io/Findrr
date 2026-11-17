# Findrr Public Architecture & Engineering Whitepaper
> **A 100% Local-First, Zero-Knowledge Personal Finance Operating System**  
> *Engineered for Android with Built-in On-Device AI, Multiplayer RPG Gamification, and Ethical Micro-Monetization.*

[![Platform](https://img.shields.io/badge/Platform-Android_14+-3DDC84?style=for-the-badge&logo=android&logoColor=white)](#)
[![UI Framework](https://img.shields.io/badge/UI-Jetpack_Compose_Material_3-4285F4?style=for-the-badge&logo=jetpackcompose&logoColor=white)](#)
[![Architecture](https://img.shields.io/badge/Architecture-Local--First_Room_SQLite-00599C?style=for-the-badge&logo=sqlite&logoColor=white)](#)
[![Security](https://img.shields.io/badge/Security-Zero--Knowledge_AES--256--GCM-E53935?style=for-the-badge&logo=security&logoColor=white)](#)
[![AI Engine](https://img.shields.io/badge/AI-On--Device_Lyra_NLU_(<1MB)-8E24AA?style=for-the-badge&logo=googlebard&logoColor=white)](#)
[![License](https://img.shields.io/badge/License-Proprietary_Architecture_Spec-37474F?style=for-the-badge)](#)

---

## Executive Summary

Modern consumer finance software is fundamentally broken in three critical dimensions:

1. **The Cloud Privacy Honey-pot:** Traditional apps upload unencrypted financial records, bank balances, and consumer spending patterns to centralized cloud servers. These databases are continuously mined for targeted advertising, sold to financial aggregators, and exposed to catastrophic data breaches.
2. **AI Bloat & Latency:** Emerging "AI finance assistants" send sensitive financial queries to remote large language models (LLMs). This introduces latency (1.5s–4s), costs recurring cloud API tokens, requires an active internet connection, and leaks private transactions to third-party AI labs.
3. **The Loneliness of Budgeting:** Personal finance tools treat money management as an isolating spreadsheet chore. Users face high abandonment rates because tracking expenses provides no intrinsic motivation, social encouragement, or gamified progression.

**Findrr** was created to resolve these systemic flaws through an uncompromising, privacy-first technical architecture:
- **100% Local-First Storage:** All financial transactions, budgets, and balance calculations live exclusively on the user's Android device inside a high-performance SQLite database powered by Android Jetpack Room.
- **Client-Side Zero-Knowledge Encryption:** Any cloud backup is strictly opt-in and client-side encrypted using **AES-256-GCM** before transit. Cloud servers only store indecipherable ciphertext.
- **Lyra: Sub-1MB On-Device AI:** A custom-engineered deterministic Natural Language Understanding (NLU) engine that parses user intents and queries local SQLite databases with zero cloud communication, zero API fees, and sub-10ms response times.
- **Arena Multiplayer Kingdoms:** An equalized competitive gamification engine where users conquer virtual territories based on a mathematically normalized **Discipline Score**, transforming frugal living into a collaborative RPG.
- **Ethical Micro-Monetization:** An ad-free model replacing invasive user tracking with a transparent ₹2 platform unlock fee for curated merchant perks, routed natively through the Razorpay UPI Intent SDK.

---

## High-Level System Topology

```mermaid
flowchart TD
    subgraph Client ["Android Client Device (Findrr OS)"]
        UI["Jetpack Compose UI (Material 3)"]
        VM["Kotlin ViewModels & StateFlow"]
        
        subgraph CoreEngine ["On-Device Core Engine"]
            Lyra["Lyra NLU Engine (<1MB)"]
            Arena["Arena Gamification Engine"]
            ZKCrypto["Zero-Knowledge Crypto (AES-256-GCM)"]
        end

        subgraph LocalData ["Local Storage (Isolated Sandbox)"]
            RoomDB[("Android Room (SQLite DB)")]
            ScopedStore["Scoped Storage (Downloads/.txt Pass)"]
        end
        
        subgraph ExtInterfaces ["Local Edge Interfaces"]
            MCPSocket["Localhost MCP Server (Port 8080)"]
            UPIIntent["Native UPI Intent Handler (GPay/PhonePe/Paytm)"]
        end
    end

    subgraph CloudInfra ["Zero-Knowledge Cloud Infrastructure"]
        SupabaseAuth["Supabase Auth (SHA-256 Hashes Only)"]
        SupabaseDB[("Encrypted Backup Vault (Ciphertext Only)")]
        RazorpayGateway["Razorpay Payment Gateway (UPI Server)"]
        AffiliateAPI["Cuelinks Merchant API (Curated Deals)"]
    end

    subgraph InternalOps ["Operations & Management Portal"]
        PortalUI["Findrr Ops Portal (Vanilla HTML5/ES Modules)"]
        EdgeWorkers["Cloudflare Pages & Functions (Edge API)"]
        PostgresOps[("Neon PostgreSQL (Enterprise Audit & RBAC)")]
    end

    %% Client Data Flows
    UI <--> VM
    VM <--> RoomDB
    VM <--> Lyra
    VM <--> Arena
    Lyra -->|Parameterized Queries| RoomDB
    MCPSocket <-->|Local IPC (Port 8080)| VM
    VM --> ZKCrypto
    ZKCrypto -->|Generates Offline Pass| ScopedStore

    %% External Network Flows
    ZKCrypto -.->|Encrypted Sync (AES-256)| SupabaseDB
    VM -.->|Hash Verification Only| SupabaseAuth
    UI -.->|Launch Native UPI| UPIIntent
    UPIIntent -.->|Process Micro-Fee| RazorpayGateway
    VM -.->|Fetch Deal Catalog| AffiliateAPI

    %% Operations Portal Flows
    PortalUI <--> EdgeWorkers
    EdgeWorkers <--> PostgresOps
```

---

## Architectural Principles

Findrr is built upon four unyielding engineering principles:

```
┌────────────────────────────────────────────────────────────────────────┐
│                      FINDRR ENGINEERING CREED                          │
├────────────────────────────────────────────────────────────────────────┤
│ 1. DEFAULT TO LOCAL: If an operation can be performed on the phone,    │
│    it MUST be performed on the phone. Never touch the network for math.│
│                                                                        │
│ 2. ZERO-KNOWLEDGE CLOUD: The cloud server is treated as an untrusted,   │
│    hostile relay. The server must never hold plaintext keys or data.   │
│                                                                        │
│ 3. DETERMINISTIC AI: Critical financial answers require mathematically  │
│    verifiable logic, not generative hallucinations from 70B models.    │
│                                                                        │
│ 4. ETHICAL CAPITALISM: Monetize via transparent micro-utilities (₹2),   │
│    never by brokering private behavioral profiles to data brokers.     │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Core System Pillars

### 1. True Zero-Knowledge Architecture
- **On-Device SQLite:** Transactions, balances, and budgets live exclusively on your device.
- **AES-256-GCM Encryption:** If cloud sync is enabled, data is client-side encrypted before departure.
- **12-Word Secret Recovery Pass:** Accounts are anchored to a 12-word recovery mnemonic. The server only sees `SHA256(Passphrase)`. No administrator or database leak can compromise your spending history.
- *Detailed Specification:* [02_ZERO_KNOWLEDGE_SECURITY.md](02_ZERO_KNOWLEDGE_SECURITY.md)

### 2. Lyra: The <1MB On-Device Micro AI Chatbot
- **Sub-10ms Inference:** Zero cloud API hops. Natural language queries like *"How much did I spend on food this month?"* resolve directly into parameterized Room DAO SQLite queries.
- **Zero Token Fees & 100% Offline:** Fully operational in Airplane mode.
- **Model Context Protocol (MCP) Bridge:** Exposes an optional authenticated localhost socket on port `8080` for power users wishing to query their budget via desktop AI agents (Claude Desktop, Cursor).
- *Detailed Specification:* [03_ON_DEVICE_AI_LYRA.md](03_ON_DEVICE_AI_LYRA.md)

### 3. Arena: Massively Multiplayer Budget Kingdoms
- **Normalized Discipline Score:** Solves the wealth gap in gamification. An algorithm equalizes income levels so a student saving ₹500 and an executive saving ₹50,000 compete strictly on discipline.
- **Golden Spiral World Map:** A 50-territory virtual world map conquered through weekly discipline battles.
- **Monarchies, Fortress Shields & Democratic Revolutions:** Reigning monarchs maintain defensive shields. If a monarch overspends, citizens can stage a democratic revolution to depose them.
- *Detailed Specification:* [04_ARENA_GAMIFICATION.md](04_ARENA_GAMIFICATION.md)

### 4. Ethical Monetization & UPI Intent Infrastructure
- **Ad-Free & Tracker-Free:** No analytics SDKs, no behavioral tracking pixels.
- **₹2 Micro-Transaction Coupon Unlock:** Users unlock premium financial partner discounts (Zerodha, Kuvera, Amazon) with a nominal ₹2 platform fee.
- **Native UPI Routing:** Seamless Android Manifest package visibility rules bypass Android 11+ restrictions, allowing native GPay, PhonePe, and Paytm handshakes without brittle webviews.
- *Detailed Specification:* [05_MONETIZATION_AND_PAYMENTS.md](05_MONETIZATION_AND_PAYMENTS.md)

### 5. Companion Operations & Governance Portal
- **Enterprise-Grade Admin System:** A zero-dependency vanilla HTML5/ES-module control plane deployed on Cloudflare Pages and Neon Serverless PostgreSQL.
- **Default-Deny RBAC & Audit Trail:** Strict Role-Based Access Control (`owner` vs `staff`), HttpOnly SameSite=Strict cookies, and immutable audit logs for all administrative operations.
- *Detailed Specification:* [06_ENTERPRISE_OPERATIONS_PORTAL.md](06_ENTERPRISE_OPERATIONS_PORTAL.md)

---

## Documentation Navigation Directory

| Document | Primary Audience | Key Topics Covered |
| :--- | :--- | :--- |
| [**01_SYSTEM_OVERVIEW.md**](01_SYSTEM_OVERVIEW.md) | Software Architects & Engineers | Component hierarchy, tech stack rationale, state management patterns, and build system specifications. |
| [**02_ZERO_KNOWLEDGE_SECURITY.md**](02_ZERO_KNOWLEDGE_SECURITY.md) | Security Researchers & Auditors | Cryptographic key derivation, AES-256-GCM envelope encryption, 12-word passphrases, and threat modeling. |
| [**03_ON_DEVICE_AI_LYRA.md**](03_ON_DEVICE_AI_LYRA.md) | Machine Learning & Mobile Devs | Tokenization pipeline, intent classifier, entity extraction, SQL synthesis, and Localhost MCP server socket. |
| [**04_ARENA_GAMIFICATION.md**](04_ARENA_GAMIFICATION.md) | Game Designers & Product Managers | Income equalization mathematics, territory state machines, monarchies, and democratic revolution algorithms. |
| [**05_MONETIZATION_AND_PAYMENTS.md**](05_MONETIZATION_AND_PAYMENTS.md) | Fintech Engineers & Partners | Native UPI intent schemes, Razorpay SDK integration, AndroidManifest queries, and affiliate deal pipelines. |
| [**06_ENTERPRISE_OPERATIONS_PORTAL.md**](06_ENTERPRISE_OPERATIONS_PORTAL.md) | Fullstack & DevOps Engineers | Cloudflare Pages, edge functions, Postgres RBAC schema, central middleware, and workload distribution. |
| [**ARCHITECTURE_DIAGRAMS.md**](ARCHITECTURE_DIAGRAMS.md) | Visual Learners & Presenters | Comprehensive catalog of Mermaid flowcharts, sequence diagrams, and system state transitions. |

---

## Public Security & Sanitization Statement

> [!NOTE]
> This architecture repository contains the complete engineering blueprints, state models, and design contracts of the Findrr ecosystem. All operational secrets, private cryptographic master seeds, database connection strings, live API tokens, and client credentials have been strictly sanitized and replaced with standardized architectural placeholders.

---
*Authored by Subhajit Das • Lead Architect, Findrr Solutions Drawer*  
*Document Version: 2.4.0 • Classification: Public Technical Whitepaper*
