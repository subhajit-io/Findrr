# 01 - System Overview & Architecture Layers

## 1. Architectural Philosophy

Findrr follows a strict **Clean Architecture + Unidirectional Data Flow (UDF)** paradigm optimized for mobile edge compute. The design guarantees that the Android client is completely autonomous: it can function indefinitely without network access, while seamlessly coordinating with external edge services when connectivity is present.

```
┌────────────────────────────────────────────────────────┐
│                   PRESENTATION LAYER                   │
│   Jetpack Compose (Material 3) • Screens • Dialogs     │
└───────────────────────────┬────────────────────────────┘
                            │ User Actions (Intents)
                            ▼
┌────────────────────────────────────────────────────────┐
│                      UI STATE LAYER                    │
│   ViewModel • StateFlow • MutableStateFlow • Coroutines│
└───────────────────────────┬────────────────────────────┘
                            │ Domain Requests
                            ▼
┌────────────────────────────────────────────────────────┐
│                      DOMAIN LAYER                      │
│   Lyra NLU Parser • Score Equalizer • ZK Crypto Engine │
└───────────────────────────┬────────────────────────────┘
                            │ CRUD Operations
                            ▼
┌────────────────────────────────────────────────────────┐
│                       DATA LAYER                       │
│   Room Database (SQLite) • Local Repositories • DAOs   │
└───────────────────────────┬────────────────────────────┘
                            │ Encrypted Sync (Opt-In)
                            ▼
┌────────────────────────────────────────────────────────┐
│                 EXTERNAL CLOUD BOUNDARY                │
│   Supabase (Ciphertext Only) • Razorpay UPI • Cuelinks │
└────────────────────────────────────────────────────────┘
```

---

## 2. Layer-by-Layer Breakdown

### 2.1. Presentation Layer (Jetpack Compose Material 3)
The entire user interface is constructed using **Jetpack Compose**, Google's declarative UI toolkit for Android.
- **Pure Declarative Hierarchy:** Eliminates legacy Android XML layouts, fragments, and view binding. UI state is passed down as immutable data classes, and events bubble up through lambda callbacks.
- **Material Design 3 (M3):** Implements dynamic color theming, adaptive dark/light palettes, and fluid spring animations.
- **Predictable Recomposition:** State is scoped to atomic composable functions (`AiChatScreen`, `ArenaMapScreen`, `SavingsHubScreen`), preventing unnecessary recomposition cycles across unrelated UI trees.

### 2.2. State Management & Concurrency
- **Kotlin StateFlow & SharedFlow:** Every screen is paired with an architectural `ViewModel`. The `ViewModel` exposes a single, immutable `StateFlow<UiState>` that the Compose UI collects using `collectAsStateWithLifecycle()`.
- **Coroutines Structured Concurrency:** All I/O operations (SQLite queries, cryptographic operations, file writes) are dispatched on `Dispatchers.IO` or custom bounded thread pools, ensuring that the Android main UI thread never drops below 60/120 FPS.

### 2.3. Domain & Intelligence Layer
This layer holds business logic isolated from Android framework dependencies:
- **Lyra NLU Parser:** Deterministic natural language tokenization and intent extraction engine.
- **Discipline Score Equalizer:** Mathematical normalization engine that calculates game scores without exposing raw salary or income data.
- **Zero-Knowledge Cryptographic Engine:** Coordinates key derivation (PBKDF2) and AES-256-GCM authenticated cipher operations.

### 2.4. Data Layer (Room SQLite Engine)
- **Local-First SQLite:** All financial data (transactions, budgets, categories, user settings, streak logs) is persisted locally via **Android Jetpack Room**.
- **Reactive Queries:** DAOs return `Flow<List<TransactionEntity>>`, allowing changes made by transactions, deletions, or Lyra AI interactions to immediately trigger reactive UI updates.
- **Zero Remote Dependencies for Core Operations:** Adding an expense, computing monthly totals, or chatting with Lyra requires zero network requests.

---

## 3. Technology Stack & Architectural Rationale

| Layer / Component | Technology Selected | Version / Standard | Architectural Rationale |
| :--- | :--- | :--- | :--- |
| **Operating System** | Android OS | Android 10+ (API 29+) | Targets 95%+ of global active Android devices while leveraging modern Scoped Storage and Hardware Keystore. |
| **Language** | Kotlin | 2.0+ | Modern type safety, null safety, coroutines, and seamless Jetpack Compose compiler support. |
| **UI Framework** | Jetpack Compose | Material 3 (1.3+) | Declarative, frictionless state synchronization; eliminates XML boilerplate and UI sync bugs. |
| **Local Database** | Room Persistence Library | SQLite 3.x | Type-safe SQL validation at compile-time, first-class Kotlin Coroutines & Flow integration, robust migrations. |
| **On-Device NLP** | Lyra Hybrid Engine | Custom Kotlin Engine | <1MB APK overhead, zero cloud latency (<10ms), zero API fees, operates 100% offline in Airplane mode. |
| **Cryptography** | Java Cryptography Architecture (JCA) | AES-256-GCM + PBKDF2 | Authenticated envelope encryption ensuring confidentiality and integrity; zero plaintext cloud exposure. |
| **Cloud Vault / Auth** | Supabase | PostgREST / GoTrue Auth | Open-source backend supporting row-level security (RLS) and custom Postgres cryptographic RPCs. |
| **Payment Gateway** | Razorpay Android SDK | Native Checkout Standard | Industry-standard Indian payment gateway with native UPI Intent routing (Google Pay, PhonePe, Paytm). |
| **Local IPC Interface** | Model Context Protocol (MCP) | Localhost TCP Socket (8080) | Allows desktop coding agents and AI IDEs to securely interface with the phone's budget data via local authorization. |
| **Operations Portal** | Cloudflare Pages + Functions | V8 Worker Isolates | Serverless edge deployment with zero cold starts, global distribution, and zero frontend build dependencies. |
| **Ops Database** | Neon Serverless PostgreSQL | PostgreSQL 16 | ACID-compliant enterprise storage for portal RBAC, team workloads, and immutable audit logging. |

---

## 4. Client vs. Cloud Boundaries

Findrr establishes a strict cryptographic boundary separating the client device from cloud infrastructure:

```
┌────────────────────────────────────────────────────────┐
│                   CLIENT BOUNDARY                      │
│                                                        │
│  [Plaintext Financial Records]                         │
│  [User Balances & Categories]                          │
│  [12-Word Secret Recovery Pass]                        │
│  [Lyra NLU Intent State]                               │
└───────────────────────────┬────────────────────────────┘
                            │
               CRYPTOGRAPHIC AIR-GAP (AES-256)
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│                    CLOUD BOUNDARY                      │
│                                                        │
│  [AES-256-GCM Encrypted Blobs (Ciphertext)]            │
│  [SHA-256 Hash of Recovery Pass (Auth Only)]           │
│  [Anonymous Arena Player IDs & Discipline Scores]      │
│  [Payment Order IDs & Verification Signatures]         │
└────────────────────────────────────────────────────────┘
```

### What NEVER Leaves the Phone:
1. Transaction amounts, descriptions, merchant names, and timestamps in plaintext.
2. Unhashed recovery passwords, account credentials, or master encryption keys.
3. Chat history with Lyra AI or natural language prompts.
4. Exact account balances, bank account details, or income figures.

### What the Cloud Sees:
1. High-entropy encrypted binary blobs (indistinguishable from random noise without the client's key).
2. One-way SHA-256 cryptographic hashes for identity authentication.
3. Anonymized integer `Discipline Scores` used for multiplayer Arena matchmaking.
4. Razorpay transaction IDs for ₹2 micro-fee validation.

---

## 5. Build, Linting & Quality Verification

To ensure enterprise-grade reliability, the Findrr codebase enforces strict automated verification:

- **Lint Checks:** Passes Android Gradle Lint (`:app:lintDebug`) with **0 warnings and 0 errors**.
- **Binary Footprint:** Total compiled APK size is maintained at approximately **52.5 MB**, including the complete Material 3 Compose runtime, Razorpay native binaries, Room SQLite engine, and the Lyra NLU model.
- **Memory Footprint:** Baseline runtime RAM consumption stays under **65 MB**, allowing seamless background operation on budget Android devices.
