# Findrr Architecture Diagrams Catalog

This document provides a visual catalog of the core architectural patterns, state machines, and sequence workflows operating across the Findrr ecosystem.

---

## 1. End-to-End System Topology

```mermaid
flowchart TB
    subgraph MobileDevice ["Android Client Architecture"]
        subgraph UI_Layer ["Presentation Layer"]
            ComposeUI["Jetpack Compose UI (Material 3)"]
            ChatScreen["AiChatScreen (Lyra UI)"]
            ArenaScreen["ArenaMapScreen (World Map)"]
            HubScreen["SavingsHubScreen (Razorpay UI)"]
        end

        subgraph ViewModel_Layer ["State & Concurrency Layer"]
            VM["FinanceViewModel"]
            StateFlow["StateFlow & SharedFlow (Immutable)"]
            Coroutines["Kotlin Coroutines (Dispatchers.IO)"]
        end

        subgraph Core_Engines ["Domain Core Logic"]
            LyraNLP["Lyra NLU Intent Engine (<1MB)"]
            ScoreEngine["Discipline Score Normalizer"]
            ZKCrypto["AES-256-GCM Envelope Encryption"]
        end

        subgraph Storage_Layer ["On-Device Persistence"]
            RoomDB[("Android Room (SQLite Database)")]
            ScopedPass["Downloads/Findrr_Security_Pass.txt"]
        end

        subgraph Local_IPC ["Edge IPC Interfaces"]
            MCPServer["Localhost MCP Server (Port 8080)"]
            UPIHandler["Native UPI Scheme Interceptor"]
        end
    end

    subgraph Cloud_Infrastructure ["Zero-Knowledge Cloud Relays"]
        SupabaseAuth["Supabase Auth (One-Way Hashes)"]
        SupabaseVault[("Encrypted Backup Vault (Ciphertext)")]
        RazorpayEngine["Razorpay Payment Gateway"]
        MerchantAffiliates["Cuelinks Deals Engine"]
    end

    subgraph Ops_Control ["Operations Portal (Web)"]
        OpsPortal["Findrr Operations Portal (Vanilla SPA)"]
        EdgeWorkers["Cloudflare Pages & Functions"]
        PostgresOps[("Neon Serverless PostgreSQL (RBAC & Audit)")]
    end

    %% Client Internal Connections
    ComposeUI <--> VM
    ChatScreen <--> VM
    ArenaScreen <--> VM
    HubScreen <--> VM
    VM <--> StateFlow
    VM <--> Coroutines

    VM <--> LyraNLP
    VM <--> ScoreEngine
    VM <--> ZKCrypto

    LyraNLP -->|Parameterized Queries| RoomDB
    ScoreEngine <--> RoomDB
    ZKCrypto -->|Generates Offline Pass| ScopedPass
    VM <--> RoomDB

    MCPServer <-->|Local Bearer Token| RoomDB
    HubScreen --> UPIHandler

    %% Cloud Network Connections
    ZKCrypto -.->|Encrypted Sync (AES-256)| SupabaseVault
    VM -.->|Auth Verification (SHA-256)| SupabaseAuth
    UPIHandler -.->|UPI Intent Handshake| RazorpayEngine
    VM -.->|Fetch Deal Catalog| MerchantAffiliates

    %% Operations Portal Connections
    OpsPortal <--> EdgeWorkers
    EdgeWorkers <--> PostgresOps
```

---

## 2. Zero-Knowledge Cryptographic Registration & Sync Flow

```mermaid
sequenceDiagram
    autonumber
    actor User as User Device (Client)
    participant UI as Onboarding UI
    participant Crypto as ZK Crypto Engine
    participant Storage as Android Scoped Storage
    participant Cloud as Supabase Cloud Vault

    User->>UI: Launch First-Time Setup
    UI->>Crypto: Generate Cryptographic Keypair
    Crypto->>Crypto: 128-bit CSPRNG Entropy Generation
    Crypto->>Crypto: Map to 12 BIP-39 English Words
    Crypto->>Crypto: Compute Hash = SHA256(Phrase)
    Crypto->>Crypto: Derive AES-256 Master Key via PBKDF2(Phrase, Salt, 100k)
    
    Crypto->>Storage: Write "Findrr_Security_Pass.txt" via MediaStore.Downloads
    Crypto->>Cloud: POST /auth/register (Email, SHA256_Hash)
    Note over Cloud: Server records the SHA-256 hash.<br/>Server NEVER receives the 12 words or master key!
    Cloud-->>UI: Registration Confirmed

    rect rgb(240, 248, 255)
        Note over User, Cloud: Optional Cloud Backup Flow
        User->>UI: Trigger Cloud Backup
        UI->>Crypto: Encrypt Local SQLite Dump
        Crypto->>Crypto: Generate 96-bit Random Nonce (IV)
        Crypto->>Crypto: Encrypt Payload using AES-256-GCM
        Crypto->>Cloud: PUT /vault/backup (Encrypted Blob + IV + Auth Tag)
        Note over Cloud: Cloud stores opaque ciphertext.<br/>Admins cannot decrypt user records.
        Cloud-->>UI: 200 Backup Synchronized
    end
```

---

## 3. Lyra Micro-AI Query Execution Lifecycle

```mermaid
flowchart TD
    A["User Inputs Natural Query\n('How much spent on groceries this week?')"] --> B["Step 1: Normalization & Tokenization\n(Lowercase, strip symbols, tokenize words)"]
    B --> C["Step 2: Semantic Intent Classifier\n(Vector keyword matching)"]
    
    C --> D{"Intent Identified?"}
    D -- "No" --> E["Fallback: Contextual Help Chips & Guidance"]
    D -- "Yes" --> F["Intent: GET_CATEGORY_EXPENSE"]
    
    F --> G["Step 3: Entity Extraction Engine"]
    G --> H["Extract Category: 'Groceries'\nExtract Timeframe: 'THIS_WEEK' (Epoch Bounds)"]
    
    H --> I["Step 4: Parameterized Room SQL Synthesizer"]
    I --> J[("Execute Local SQLite Query\nSELECT SUM(amount) FROM transactions\nWHERE category = :cat AND timestamp >= :start")]
    
    J --> K["Step 5: Natural Language Generator (NLG)\nFormat localized response string: '₹1,240.00'"]
    K --> L["Step 6: Render in AiChatScreen with 500ms Typing Simulation"]
```

---

## 4. Arena Territory State Machine

```mermaid
stateDiagram-v2
    [*] --> Unclaimed : Initial World Generation
    
    Unclaimed --> SovereignMonarchy : Player logs top weekly Discipline Score
    
    state SovereignMonarchy {
        [*] --> StrongDefense
        StrongDefense --> StableRule : Monarch logs transactions & meets budget
        StableRule --> WarningZone : Monarch exceeds monthly budget
        WarningZone --> StableRule : Monarch restores financial discipline
        WarningZone --> RevolutionTriggered : Monarch slacks for 14 consecutive days
    }

    state CitizensAndFortress {
        SovereignMonarchy --> GuildFormed : Citizens pledge discipline
        GuildFormed --> FortressShieldActive : Combined score builds Fortress Shield
        FortressShieldActive --> ShieldWeakened : Citizens overspend
        ShieldWeakened --> FortressShieldActive : Citizens cut unnecessary spending
    }

    RevolutionTriggered --> DemocraticVote : Citizens initiate automatic revolution vote
    DemocraticVote --> SovereignMonarchy : Highest discipline citizen crowned as New Monarch
```

---

## 5. Native UPI Payment Lifecycle

```mermaid
sequenceDiagram
    autonumber
    actor User as User
    participant App as Findrr Android App
    participant SDK as Razorpay SDK
    participant UPI as Native UPI App (PhonePe / GPay)
    participant Gateway as Razorpay Settlement Engine
    participant Room as Local Room Database

    User->>App: Tap "Unlock ₹2 Deal" (Zerodha/Kuvera)
    App->>SDK: Checkout.open(OrderPayload, ₹2.00)
    Note over SDK: Inspects installed UPI apps via Manifest <queries>
    SDK->>User: Surface Native App Sheet (Google Pay, PhonePe, Paytm)
    User->>SDK: Choose PhonePe
    
    SDK->>UPI: Intent(ACTION_VIEW, "upi://pay?pa=findrr@bank&am=2.00...")
    Note over UPI: User authenticates with bank-grade biometric/PIN
    UPI->>Gateway: Direct IMPS/NPCI Bank Transfer
    Gateway-->>UPI: Bank Settlement Success
    UPI-->>SDK: onActivityResult(RESULT_OK, PaymentData)
    
    SDK-->>App: onPaymentSuccess(PaymentID)
    App->>Room: Update Local Deal Record (status = UNLOCKED)
    App-->>User: Reveal Voucher Code & Open Partner App
```

---

## 6. Operations Portal Default-Deny Middleware Pipeline

```mermaid
sequenceDiagram
    autonumber
    actor Staff as Portal User
    participant Edge as Cloudflare Edge Worker
    participant Auth as Session & Security Engine
    participant DB as Neon PostgreSQL
    participant Route as Operational Route Handler

    Staff->>Edge: Request /api/staff/activity
    Edge->>Auth: Extract "findrr_session" Cookie
    
    alt Cookie Missing or Tampered
        Auth-->>Staff: 401 Unauthorized (Redirect to /login)
    else Valid Cookie Present
        Auth->>DB: Query Session Validity & Staff Permissions
        alt Session Expired
            DB-->>Auth: Invalid Session Record
            Auth-->>Staff: 401 Session Expired
        else Valid Active Session
            DB-->>Auth: Staff Found (Role: owner / staff)
            alt Route Requires "owner" but User is "staff"
                Auth-->>Staff: 403 Forbidden (Insufficient Privileges)
            else Permission Check Passed
                Auth->>Route: Execute Route Controller
                Route->>DB: Fetch Data / Update Record
                Route->>DB: INSERT INTO audit_log (staff_id, action, timestamp)
                Route-->>Staff: 200 OK (Clean JSON Payload)
            end
        end
    end
```
