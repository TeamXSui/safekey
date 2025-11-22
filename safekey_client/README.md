# SafeKey - Decentralized Password Manager

> **Advanced cryptographic password manager with SEAL secret sharing, Sui blockchain storage, and Walrus distributed storage**

SafeKey is a cutting-edge decentralized password manager that leverages blockchain technology, zero-knowledge authentication, and distributed cryptography to provide unparalleled security for credential management. Built on the Sui blockchain with zkLogin integration, SEAL (Secret Extended Access Layer) for distributed key derivation, and Walrus for decentralized storage.

## **Architecture Overview**

SafeKey implements a **two-tier architecture** that separates concerns for optimal security and functionality:

```
┌─────────────────────────────────┐    ┌─────────────────────────────────┐
│          WEB APPLICATION        │    │       BROWSER EXTENSION         │
│         (safekey_client/web-app)   │    │      (safekey_client/extension)    │
│                                 │    │                                 │
│  • OAuth Authentication        │    │  • Form Detection               │
│  • SEAL Master Key Derivation  │◄──►│  • Auto-fill Functionality      │
│  • Credential Management UI    │    │  • Session Synchronization      │
│  • Blockchain Transactions     │    │  • Local Credential Cache       │
│  • Session Management          │    │  • Heartbeat Service            │
└─────────────────────────────────┘    └─────────────────────────────────┘
              │                                        │
              ▼                                        ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        STORAGE LAYER                                     │
│                                                                         │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐         │
│  │  Sui Blockchain │  │ Walrus Storage  │  │ SEAL Servers    │         │
│  │   (Metadata)    │  │ (Encrypted Data)│  │ (Key Shares)    │         │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘         │
└─────────────────────────────────────────────────────────────────────────┘
```

## **Security Architecture**

SafeKey implements **multi-layered security** with innovative cryptographic techniques:

### **1. SEAL Secret Sharing (Master Key Derivation)**
```typescript
// Distributed threshold cryptography
SessionKey.create() → Authenticates with zkLogin proof
SEAL.encrypt() → Creates encrypted object + symmetric key  
Master_Key = SHA-256(address + symmetric_key)
```
- **Security**: No single key server can reconstruct master key
- **Deterministic**: Same login always produces same keys
- **Zero Knowledge**: Key servers can't access user data

### **2. Hybrid Storage Model**
```typescript
// Metadata stored on Sui blockchain
VaultEntry {
  domain_hash: HMAC-SHA256(Master_Key, domain),
  blob_ids: [walrus_blob_id],  // Points to Walrus storage
  created_at: timestamp
}

// Encrypted data stored on Walrus
BlobData: [session_nonce_len][session_nonce][iv_len][iv][ciphertext]
```

### **3. Two-Layer Encryption**
```typescript
// Layer 1: Domain isolation
Domain_Hash = HMAC-SHA256(Master_Key, domain)

// Layer 2: Credential encryption  
Session_Key = HKDF-SHA256(Master_Key, session_nonce)
Encrypted_Data = AES-256-GCM(credential_json, Session_Key)
```

## **Blockchain Integration**

### **Smart Contract** (`Safekey_SC/sources/vault.move`)
```move
struct UserVault has key, store {
    id: UID,
    owner: address,
    created_at: u64,
}

struct VaultEntry has store {
    owner: address,
    domain_hash: vector<u8>,     // HMAC-SHA256 hash
    data: vector<u8>,            // Walrus blob IDs
    entry_nonce: vector<u8>,     // Encryption IV
    session_nonce: vector<u8>,   // Key derivation nonce  
    created_at: u64,
}
```

### **Key Operations**
| Function | Purpose | Gas Cost |
|----------|---------|----------|
| `create_vault()` | Initialize user vault | ~0.001 SUI |
| `add_entry()` | Store credential metadata | ~0.002 SUI |
| `get_entry_info()` | Retrieve credential data | Free (read) |
| `update_entry()` | Modify existing credential | ~0.002 SUI |

## **Walrus Decentralized Storage**

SafeKey uses **Walrus** for storing encrypted credential data:

```typescript
// Credential storage flow
const blobData = [sessionNonceLen, sessionNonce, ivLen, iv, ciphertext]
const blobId = await storeBlob(blobData, signer)  // Store on Walrus
const vaultEntry = { domain_hash, data: [blobId] } // Reference in blockchain
```

**Benefits**:
- **Decentralized**: No single point of failure
- **Scalable**: Unlimited storage capacity  
- **Cost-effective**: Cheaper than on-chain storage
- **Durable**: Redundancy across multiple nodes

## **zkLogin Authentication Flow**

```
1. User clicks "Login with Google/Apple"
   ↓
2. OAuth redirect → JWT token + profile
   ↓  
3. Enoki wallet creates zkLogin proof
   ↓
4. SEAL SessionKey.create() with proof
   ↓
5. SEAL.encrypt() derives symmetric key
   ↓
6. Master key = SHA-256(address + symmetric_key)
   ↓
7. Session established with sponsored transactions
```

### **Supported Providers**
- Google (`@mysten/enoki`)

## **Technical Stack**

### **Frontend Architecture**
```
Web App (safekey_client/web-app/):
├── React 19 + TypeScript
├── Vite (build tool)  
├── @mysten/dapp-kit (Sui integration)
├── @mysten/enoki (zkLogin wallets)
├── @mysten/seal (secret sharing)
├── @mysten/walrus (decentralized storage)
├── Express API server (sponsored transactions)
└── TanStack Query (data fetching)
```

### **Extension Architecture**
```
Extension (safekey_client/extension/):
├── Manifest V3 (modern Chrome extension)
├── Background service worker
├── Content scripts (form detection)
├── Popup UI (credential access)
├── WebExtension polyfills
└── TweetNaCl (lightweight crypto)
```

### **Core Libraries**
| File | Purpose | Key Functions |
|------|---------|---------------|
| `crypto.ts` | Encryption/decryption | `encrypt()`, `decrypt()`, `hashDomain()` |
| `seal.ts` | SEAL integration | `deriveMasterKeyFromSeal()` |
| `vault.ts` | Blockchain operations | `addCredential()`, `getCredentialInfo()` |
| `walrus.ts` | Decentralized storage | `storeBlob()`, `retrieveBlobs()` |
| `credentials.ts` | High-level API | `saveCredential()`, `getCredential()` |

## **Project Structure**

```
├── safekey_client/
│   ├── web-app/                     # React web application
│   │   ├── src/
│   │   │   ├── lib/                 # Core libraries
│   │   │   │   ├── crypto.ts        # AES-256-GCM encryption
│   │   │   │   ├── seal.ts          # SEAL secret sharing
│   │   │   │   ├── vault.ts         # Sui blockchain ops
│   │   │   │   ├── walrus.ts        # Walrus storage
│   │   │   │   └── credentials.ts   # Credential management
│   │   │   ├── pages/               # React components
│   │   │   ├── server/              # Express API server
│   │   │   └── services/            # Business logic
│   │   └── package.json             # Dependencies
│   └── extension/                   # Browser extension
│       ├── src/
│       │   ├── background/          # Service worker
│       │   ├── content/             # Content scripts  
│       │   ├── popup/               # Extension popup
│       │   └── services/            # Extension services
│       └── public/manifest.json     # Extension manifest
```

## **Getting Started**

### **Prerequisites**
```bash
Node.js 18+ 
npm or yarn
Chrome/Edge browser (for extension)
```

### **Installation & Development**

1. **Clone Repository**:
```bash
git clone https://github.com/abdushakurob/SafeKeyApp.git
cd SafeKeyApp/safekey_client
```

2. **Web App Setup**:
```bash
cd web-app
npm install
cp .env.example .env  # Configure environment variables
npm run dev           # Start dev server (http://localhost:5173)
npm run server        # Start API server (http://localhost:3001)
```

3. **Extension Setup**:
```bash
cd ../extension
npm install  
npm run build         # Build extension to dist/

# Load in Chrome:
# 1. Go to chrome://extensions/
# 2. Enable "Developer mode"
# 3. Click "Load unpacked" 
# 4. Select the dist/ folder
```

### **Environment Configuration**

**Web App** (`.env`):
- check `.env.example`

## **Complete User Flow**

### **First-Time Setup**
1. User visits SafeKey web app
2. Clicks "Login with Google" 
3. OAuth authentication completes
4. Enoki wallet creates zkLogin proof + ephemeral keys
5. SEAL derives master key from proof + entropy
6. Smart contract creates user vault on Sui (sponsored transaction)
7. Extension syncs session data for auto-fill

### **Saving Credentials**
1. User enters website credentials in web app
2. Domain hash computed: `HMAC-SHA256(master_key, domain)`
3. Session key derived: `HKDF-SHA256(master_key, session_nonce)`
4. Credentials encrypted: `AES-256-GCM(data, session_key)`
5. Encrypted data stored on Walrus (returns blob_id)
6. Metadata stored on Sui blockchain (domain_hash → blob_id)
7. Extension cache updated for instant access

### **Auto-fill Experience**  
1. User visits website (detected by extension content script)
2. Extension detects login form fields
3. Queries local cache for domain match
4. If found: Shows auto-fill button
5. User clicks button → Credentials filled automatically
6. If not cached: Syncs with web app to fetch from blockchain

## **Performance & Scalability**


### **Extension Performance**
- **Form Detection**: <50ms per page load
- **Cache Lookup**: <10ms per domain
- **Auto-fill**: <100ms per form
- **Background Sync**: Every 20 seconds

## **Security Features**

### **Cryptographic Standards**
- **Encryption**: AES-256-GCM (NIST approved)
- **Key Derivation**: HKDF-SHA256 (RFC 5869)
- **Domain Hashing**: HMAC-SHA256 (RFC 2104)
- **Random Generation**: Crypto.getRandomValues()


### **Privacy Protection**
**Domain Privacy**: Domains hashed, not stored in plaintext  
**Zero Knowledge**: SEAL servers can't decrypt user data  
**Forward Secrecy**: Session keys rotated per login  
