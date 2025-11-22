# SafeKey - Smart Contract Documentation

## Overview
SafeKey Vault is a Sui Move smart contract that provides secure credential storage metadata on the Sui blockchain. Each user owns a personal vault where they can store references to encrypted credentials that are stored on Walrus decentralized storage.

**Key Security Features:**
- All credential data is encrypted client-side before being stored on Walrus
- Only metadata and Walrus blob references are stored on-chain  
- Only the vault owner can access their entries
- Encryption keys never touch the blockchain
- Each entry is indexed by a hashed domain identifier

**Storage Architecture:**
- **On-Chain (Sui)**: Domain hashes, Walrus blob IDs, timestamps (metadata only)
- **Off-Chain (Walrus)**: Encrypted credential data, session nonces, encryption IVs

---

## Contract Address
```
Package ID: [TO BE FILLED AFTER DEPLOYMENT]
Module: safekey::vault
```

---

## Data Structures

### UserVault
The main container object that holds all user credentials.

```move
public struct UserVault has key {
    id: UID,
    owner: address
}
```

**Fields:**
- `id`: Unique identifier for the vault
- `owner`: Address of the vault owner

**Ownership:** Owned by the user who created it

---

### VaultEntry
Individual credential entry metadata stored within a vault.

```move
public struct VaultEntry has key, store {
    id: UID,
    owner: address,
    domain_hash: vector<u8>,
    data: vector<u8>,
    entry_nonce: vector<u8>,
    session_nonce: vector<u8>,
    created_at: u64
}
```

**Fields:**
- `id`: Unique identifier for the entry
- `owner`: Address of the entry owner (same as vault owner)
- `domain_hash`: HMAC-SHA256 hash of the domain/service name (used as lookup key)
- `data`: **JSON array of Walrus blob IDs** (not encrypted credential data)
- `entry_nonce`: **Empty in current implementation** (nonces stored in Walrus blobs)
- `session_nonce`: **Empty in current implementation** (nonces stored in Walrus blobs)
- `created_at`: Timestamp in milliseconds when entry was created/last updated

**Storage:** Stored as dynamic fields on the UserVault object

**Important Notes:**
- **Current Implementation**: The `data` field contains a JSON array of Walrus blob IDs (e.g., `["blob_id_1", "blob_id_2"]`)
- **Legacy Support**: Old entries may still contain encrypted data directly on-chain
- **Nonces**: Session and entry nonces are stored within the Walrus blob payload, not on-chain
- **Walrus Blob Format**: `[sessionNonceLen(1)][sessionNonce][ivLen(1)][iv][ciphertext]`

---

## Error Codes

| Code | Constant | Description |
|------|----------|-------------|
| 0 | `ENotAuthorized` | Caller is not the vault owner |
| 1 | `EEntryAlreadyExists` | An entry with this domain_hash already exists |
| 2 | `EEntryNotFound` | No entry found for the given domain_hash |

---

## Functions

### 1. create_vault
Creates a new user vault and transfers ownership to the caller.

**Signature:**
```move
public fun create_vault(ctx: &mut TxContext)
```

**Parameters:**
- `ctx`: Transaction context (automatically provided)

**Returns:** None (transfers `UserVault` object to caller)

**Usage Example (TypeScript SDK):**
```typescript
const tx = new Transaction();
tx.moveCall({
    target: `${PACKAGE_ID}::vault::create_vault`,
});

const result = await signAndExecuteTransaction({
    transaction: tx,
    chain: 'sui:mainnet',
});
```

**Gas Estimate:** ~0.001 SUI

---

### 2. add_entry
Adds a new credential entry metadata to the vault.

**Signature:**
```move
public fun add_entry(
    vault: &mut UserVault,
    domain_hash: vector<u8>,
    data: vector<u8>,
    entry_nonce: vector<u8>,
    session_nonce: vector<u8>,
    clock: &Clock,
    ctx: &mut TxContext
)
```

**Parameters:**
- `vault`: Reference to the user's vault object
- `domain_hash`: HMAC-SHA256 hash of the domain (e.g., `HMAC-SHA256(master_key, "github.com")`)
- `data`: **JSON array of Walrus blob IDs as bytes** (e.g., `["blob_id_1"]`)
- `entry_nonce`: **Empty array** in current implementation (stored in Walrus)
- `session_nonce`: **Empty array** in current implementation (stored in Walrus)
- `clock`: Reference to shared Clock object (`0x6`)
- `ctx`: Transaction context

**Errors:**
- `ENotAuthorized`: If caller is not the vault owner
- `EEntryAlreadyExists`: If an entry with this domain_hash already exists

**Usage Example (TypeScript SDK):**
```typescript
import { hashDomain } from './crypto';

// Client-side encryption and storage flow
const domain = "github.com";
const domainHash = await hashDomain(domain, masterKey); // HMAC-SHA256
const domainHashBytes = Uint8Array.from(atob(domainHash), c => c.charCodeAt(0));

// 1. Encrypt credentials and store on Walrus
const blobId = await storeBlob(encryptedCredentialData, signer);

// 2. Store blob ID reference on-chain
const blobIds = [blobId];
const blobIdsJson = JSON.stringify(blobIds);
const blobIdsBytes = new TextEncoder().encode(blobIdsJson);

const tx = new Transaction();
tx.moveCall({
    target: `${PACKAGE_ID}::vault::add_entry`,
    arguments: [
        tx.object(vaultId),                        // vault
        tx.pure.vector('u8', domainHashBytes),     // domain_hash  
        tx.pure.vector('u8', blobIdsBytes),        // data (JSON array of blob IDs)
        tx.pure.vector('u8', new Uint8Array(0)),   // entry_nonce (empty)
        tx.pure.vector('u8', new Uint8Array(0)),   // session_nonce (empty)
        tx.object('0x6'),                          // clock
    ],
});

await signAndExecuteTransaction({ transaction: tx });
```

**Gas Estimate:** ~0.002-0.005 SUI (depends on number of blob IDs)

---

### 3. get_entry_info
Retrieves credential metadata for a specific domain.

**Signature:**
```move
public fun get_entry_info(
    vault: &UserVault,
    domain_hash: vector<u8>,
    ctx: &TxContext
): (address, vector<u8>, vector<u8>, vector<u8>, vector<u8>, u64)
```

**Parameters:**
- `vault`: Reference to the user's vault object
- `domain_hash`: HMAC-SHA256 hash of the domain to retrieve
- `ctx`: Transaction context

**Returns:** Tuple containing:
1. `owner` (address)
2. `domain_hash` (vector<u8>)
3. `data` (vector<u8>) - **JSON array of Walrus blob IDs**
4. `entry_nonce` (vector<u8>) - **Empty in current implementation**
5. `session_nonce` (vector<u8>) - **Empty in current implementation**
6. `created_at` (u64) - timestamp in milliseconds

**Errors:**
- `ENotAuthorized`: If caller is not the vault owner
- `EEntryNotFound`: If no entry exists for this domain_hash

**Usage Example (TypeScript SDK):**
```typescript
const tx = new Transaction();
tx.moveCall({
    target: `${PACKAGE_ID}::vault::get_entry_info`,
    arguments: [
        tx.object(vaultId),
        tx.pure.vector('u8', domainHashBytes),
    ],
});

// Use devInspectTransactionBlock for read-only operations
const result = await suiClient.devInspectTransactionBlock({
    sender: walletAddress,
    transactionBlock: tx,
});

// Parse blob IDs and retrieve from Walrus
const dataBytes = result.results[0].returnValues[2][0];
const blobIdsJson = new TextDecoder().decode(new Uint8Array(dataBytes));
const blobIds = JSON.parse(blobIdsJson);

// Retrieve and decrypt credentials from Walrus
const credentials = await retrieveAndDecryptCredentials(blobIds, masterKey);
```

**Gas Estimate:** Free (read-only, use `devInspectTransactionBlock`)

---

### 4. update_entry
Updates an existing credential entry with new Walrus blob references.

**Signature:**
```move
public fun update_entry(
    vault: &mut UserVault,
    domain_hash: vector<u8>,
    new_data: vector<u8>,
    new_entry_nonce: vector<u8>,
    new_session_nonce: vector<u8>,
    clock: &Clock,
    ctx: &TxContext
)
```

**Parameters:**
- `vault`: Reference to the user's vault object
- `domain_hash`: HMAC-SHA256 hash of the domain to update
- `new_data`: **New JSON array of Walrus blob IDs** (not encrypted data)
- `new_entry_nonce`: **Empty array** in current implementation
- `new_session_nonce`: **Empty array** in current implementation
- `clock`: Reference to shared Clock object (`0x6`)
- `ctx`: Transaction context

**Errors:**
- `ENotAuthorized`: If caller is not the vault owner
- `EEntryNotFound`: If no entry exists for this domain_hash

**Usage Example (TypeScript SDK):**
```typescript
// 1. Store new encrypted credential on Walrus
const newBlobId = await storeBlob(newEncryptedCredentialData, signer);

// 2. Update blob ID array (append to existing or replace)
const existingBlobIds = await getExistingBlobIds(vaultId, domainHashBytes);
const updatedBlobIds = [...existingBlobIds, newBlobId];
const blobIdsJson = JSON.stringify(updatedBlobIds);
const blobIdsBytes = new TextEncoder().encode(blobIdsJson);

const tx = new Transaction();
tx.moveCall({
    target: `${PACKAGE_ID}::vault::update_entry`,
    arguments: [
        tx.object(vaultId),
        tx.pure.vector('u8', domainHashBytes),
        tx.pure.vector('u8', blobIdsBytes),        // Updated blob IDs
        tx.pure.vector('u8', new Uint8Array(0)),   // Empty entry nonce
        tx.pure.vector('u8', new Uint8Array(0)),   // Empty session nonce
        tx.object('0x6'),
    ],
});

await signAndExecuteTransaction({ transaction: tx });
```

**Note:** The `created_at` timestamp is updated to the current time on each update.

**Gas Estimate:** ~0.002-0.005 SUI (depends on number of blob IDs)

---

### 5. delete_entry
Permanently deletes a credential entry from the vault.

**Signature:**
```move
public fun delete_entry(
    vault: &mut UserVault,
    domain_hash: vector<u8>,
    ctx: &TxContext
)
```

**Parameters:**
- `vault`: Reference to the user's vault object
- `domain_hash`: HMAC-SHA256 hash of the domain to delete
- `ctx`: Transaction context

**Errors:**
- `ENotAuthorized`: If caller is not the vault owner
- `EEntryNotFound`: If no entry exists for this domain_hash

**Usage Example (TypeScript SDK):**
```typescript
const tx = new Transaction();
tx.moveCall({
    target: `${PACKAGE_ID}::vault::delete_entry`,
    arguments: [
        tx.object(vaultId),
        tx.pure.vector('u8', domainHashBytes),
    ],
});

await signAndExecuteTransaction({ transaction: tx });
```

**Important:** This only deletes the on-chain metadata. Walrus blobs containing the actual encrypted credential data will remain until their epoch expires. Consider implementing blob cleanup if needed.

**Gas Estimate:** ~0.001-0.002 SUI

---

### 6. entry_exists
Checks if an entry exists for a given domain without retrieving its data.

**Signature:**
```move
public fun entry_exists(vault: &UserVault, domain_hash: vector<u8>): bool
```

**Parameters:**
- `vault`: Reference to the user's vault object
- `domain_hash`: HMAC-SHA256 hash of the domain to check

**Returns:** `true` if entry exists, `false` otherwise

**Usage Example (TypeScript SDK):**
```typescript
const tx = new Transaction();
const [exists] = tx.moveCall({
    target: `${PACKAGE_ID}::vault::entry_exists`,
    arguments: [
        tx.object(vaultId),
        tx.pure.vector('u8', domainHash),
    ],
});

const result = await suiClient.devInspectTransactionBlock({
    sender: walletAddress,
    transactionBlock: tx,
});

const entryExists = result.results[0].returnValues[0][0] === 1;
```

**Gas Estimate:** Free (read-only)

---

## **Storage Architecture**

SafeKey implements a **hybrid storage model** that separates metadata (on Sui blockchain) from encrypted data (on Walrus decentralized storage) for optimal security, cost, and scalability.

### **What's Stored On-Chain (Sui Blockchain)**

The smart contract stores **only metadata and references**:

```typescript
// VaultEntry on-chain data
{
  id: UID,                               // Unique entry ID
  owner: address,                        // Vault owner address  
  domain_hash: vector<u8>,               // HMAC-SHA256(master_key, domain)
  data: vector<u8>,                      // JSON array of Walrus blob IDs
  entry_nonce: vector<u8>,               // EMPTY (legacy field)
  session_nonce: vector<u8>,             // EMPTY (legacy field) 
  created_at: u64                        // Timestamp in milliseconds
}
```

**Example on-chain `data` field:**
```json
["blob_id_1a2b3c...", "blob_id_4d5e6f..."]
```

### **What's Stored Off-Chain (Walrus)**

The actual encrypted credential data is stored on Walrus with the following format:

```typescript
// Walrus blob format
[sessionNonceLen(1 byte)][sessionNonce][ivLen(1 byte)][iv][ciphertext]
```

**Example blob content:**
```
0x10                           // sessionNonce length (16 bytes)
a1b2c3d4e5f6...               // sessionNonce (16 bytes)  
0x0C                          // IV length (12 bytes)
f1e2d3c4b5a6...              // IV (12 bytes)
9f8e7d6c5b4a...              // AES-256-GCM ciphertext
```

**Encrypted ciphertext contains:**
```json
{
  "domain": "github.com",
  "username": "john@example.com", 
  "password": "super_secure_password_123"
}
```

### **Benefits of Hybrid Storage**

1. **Lower Costs**: Walrus storage is cheaper than on-chain for large data
2. **Better Privacy**: Domain names are hashed, not stored in plaintext
3. **Scalability**: Unlimited credential storage capacity  
4. **Security**: Each credential has independent encryption nonces
5. **Efficiency**: On-chain queries only retrieve small metadata

### **Storage Flow**

**Save Credential:**
1. Generate unique session nonce per credential
2. Encrypt credential data with AES-256-GCM
3. Format blob: `[nonce_len][nonce][iv_len][iv][ciphertext]`
4. Store blob on Walrus → get `blob_id`
5. Store `[blob_id]` array on-chain with domain hash

**Retrieve Credential:**
1. Query on-chain with domain hash → get blob IDs array
2. Retrieve blobs from Walrus using blob IDs
3. Parse blob format to extract nonce, IV, ciphertext
4. Decrypt credential data using master key + session nonce

### **Legacy Compatibility**

The system supports both storage formats:

- **Current**: Walrus blob references (empty on-chain nonces)
- **Legacy**: Direct on-chain encrypted storage (populated nonces)

The client automatically detects the format based on nonce field content.

---

## Client-Side Encryption Guide

### Current Implementation (Walrus Storage)

**Before storing credentials:**
1. User authenticates with Google OAuth via zkLogin  
2. SEAL derives master key from zkLogin proof + entropy
3. Generate unique session nonce per credential
4. Derive session key: `HKDF-SHA256(master_key, session_nonce)`
5. Encrypt credentials using AES-256-GCM with session key
6. Hash domain: `HMAC-SHA256(master_key, domain)`
7. Store encrypted blob on Walrus → get blob ID
8. Store blob ID array on-chain with domain hash

**When retrieving credentials:**
1. Query on-chain with domain hash → get blob IDs
2. Retrieve encrypted blobs from Walrus  
3. Extract session nonce and IV from blob
4. Derive session key from master key + session nonce
5. Decrypt credential data and display to user

### Example Implementation (TypeScript)

```typescript
import { hashDomain, encrypt, decrypt, deriveKS, generateSessionNonce } from './crypto'
import { storeBlob, retrieveBlobs } from './walrus'

// Save credential flow
async function saveCredential(
  domain: string, 
  username: string, 
  password: string,
  masterKey: string,
  signer: Signer
) {
  // 1. Generate session nonce for this credential
  const sessionNonce = generateSessionNonce()
  
  // 2. Derive session key  
  const sessionKey = await deriveKS(masterKey, sessionNonce)
  
  // 3. Encrypt credential data
  const credentialData = JSON.stringify({ domain, username, password })
  const encryptedData = await encrypt(credentialData, sessionKey)
  const [ivB64, ciphertextB64] = encryptedData.split('.')
  
  // 4. Format blob: [snLen][sessionNonce][ivLen][iv][ciphertext] 
  const sessionNonceBytes = Uint8Array.from(atob(sessionNonce), c => c.charCodeAt(0))
  const ivBytes = Uint8Array.from(atob(ivB64), c => c.charCodeAt(0)) 
  const ciphertextBytes = Uint8Array.from(atob(ciphertextB64), c => c.charCodeAt(0))
  
  const blobData = new Uint8Array(
    1 + sessionNonceBytes.length + 1 + ivBytes.length + ciphertextBytes.length
  )
  let offset = 0
  blobData[offset++] = sessionNonceBytes.length
  blobData.set(sessionNonceBytes, offset)
  offset += sessionNonceBytes.length
  blobData[offset++] = ivBytes.length  
  blobData.set(ivBytes, offset)
  offset += ivBytes.length
  blobData.set(ciphertextBytes, offset)
  
  // 5. Store on Walrus
  const blobId = await storeBlob(blobData, signer)
  
  // 6. Store blob ID reference on-chain
  const domainHash = await hashDomain(domain, masterKey) // HMAC-SHA256
  const domainHashBytes = Uint8Array.from(atob(domainHash), c => c.charCodeAt(0))
  
  const blobIds = [blobId]
  const blobIdsJson = JSON.stringify(blobIds)
  const blobIdsBytes = new TextEncoder().encode(blobIdsJson)
  
  // Call smart contract add_entry
  const tx = new Transaction()
  tx.moveCall({
    target: `${PACKAGE_ID}::vault::add_entry`,
    arguments: [
      tx.object(vaultId),
      tx.pure.vector('u8', Array.from(domainHashBytes)),
      tx.pure.vector('u8', Array.from(blobIdsBytes)),
      tx.pure.vector('u8', []), // empty entry_nonce
      tx.pure.vector('u8', []), // empty session_nonce  
      tx.object('0x6'),
    ],
  })
  
  return await signAndExecute({ transaction: tx })
}

// Retrieve credential flow  
async function getCredential(domain: string, masterKey: string): Promise<Credential[]> {
  // 1. Query on-chain for blob IDs
  const domainHash = await hashDomain(domain, masterKey)
  const domainHashBytes = Uint8Array.from(atob(domainHash), c => c.charCodeAt(0))
  
  const info = await getCredentialInfoFromDynamicField(vaultId, domainHashBytes)
  if (!info) return null
  
  // 2. Parse blob IDs from on-chain data
  const blobIdsJson = new TextDecoder().decode(info.data)
  const blobIds = JSON.parse(blobIdsJson) as string[]
  
  // 3. Retrieve blobs from Walrus
  const blobMap = await retrieveBlobs(blobIds)
  
  // 4. Decrypt each blob
  const credentials: Credential[] = []
  
  for (const [blobId, blobData] of blobMap.entries()) {
    // Parse blob format
    let offset = 0
    const snLen = blobData[offset++]
    const sessionNonceBytes = blobData.slice(offset, offset + snLen)
    offset += snLen
    const ivLen = blobData[offset++]  
    const ivBytes = blobData.slice(offset, offset + ivLen)
    offset += ivLen
    const ciphertextBytes = blobData.slice(offset)
    
    // Derive session key and decrypt
    const sessionNonceB64 = btoa(String.fromCharCode(...sessionNonceBytes))
    const sessionKey = await deriveKS(masterKey, sessionNonceB64)
    
    const ivB64 = btoa(String.fromCharCode(...ivBytes))
    const ciphertextB64 = btoa(String.fromCharCode(...ciphertextBytes))
    const encryptedData = `${ivB64}.${ciphertextB64}`
    
    const decryptedData = await decrypt(encryptedData, sessionKey)
    const credentialData = JSON.parse(decryptedData)
    
    credentials.push(credentialData)
  }
  
  return credentials
}

// Decrypt credentials
function decryptCredentials(
    ciphertext: Uint8Array,
    nonce: Uint8Array,
    masterKey: Uint8Array
): { username: string; password: string } {
    const cipher = xchacha20poly1305(masterKey, nonce);
    const plaintext = cipher.decrypt(ciphertext);
    
    return JSON.parse(new TextDecoder().decode(plaintext));
}

// Hash domain for lookup
function hashDomain(domain: string): Uint8Array {
    return sha256(domain);
}
```

---

## Integration Workflow

### 1. Initial Setup (New User)
```typescript
// Step 1: Create vault
const createTx = new Transaction();
createTx.moveCall({
    target: `${PACKAGE_ID}::vault::create_vault`,
});

const result = await signAndExecuteTransaction({ transaction: createTx });

// Step 2: Extract vault object ID from transaction effects
const vaultId = result.effects.created[0].reference.objectId;

// Step 3: Store vaultId in local storage
localStorage.setItem('safekey_vault_id', vaultId);
```

### 2. Adding Credentials
```typescript
async function addPassword(domain: string, username: string, password: string) {
    // Get master key (from user input)
    const masterPassword = prompt('Enter master password:');
    const salt = getUserSalt(); // Retrieve or generate user-specific salt
    const masterKey = await deriveMasterKey(masterPassword, salt);
    
    // Encrypt credentials
    const { data, nonce: entryNonce } = encryptCredentials(username, password, masterKey);
    const sessionNonce = randomBytes(24);
    
    // Hash domain
    const domainHash = Array.from(hashDomain(domain));
    
    // Add to blockchain
    const vaultId = localStorage.getItem('safekey_vault_id');
    const tx = new Transaction();
    tx.moveCall({
        target: `${PACKAGE_ID}::vault::add_entry`,
        arguments: [
            tx.object(vaultId),
            tx.pure.vector('u8', domainHash),
            tx.pure.vector('u8', Array.from(data)),
            tx.pure.vector('u8', Array.from(entryNonce)),
            tx.pure.vector('u8', Array.from(sessionNonce)),
            tx.object('0x6'),
        ],
    });
    
    await signAndExecuteTransaction({ transaction: tx });
}
```

### 3. Retrieving Credentials
```typescript
async function getPassword(domain: string) {
    const vaultId = localStorage.getItem('safekey_vault_id');
    const domainHash = Array.from(hashDomain(domain));
    
    // Check if entry exists first
    const existsTx = new Transaction();
    existsTx.moveCall({
        target: `${PACKAGE_ID}::vault::entry_exists`,
        arguments: [
            existsTx.object(vaultId),
            existsTx.pure.vector('u8', domainHash),
        ],
    });
    
    const existsResult = await suiClient.devInspectTransactionBlock({
        sender: walletAddress,
        transactionBlock: existsTx,
    });
    
    if (!existsResult.results[0].returnValues[0][0]) {
        throw new Error('No credentials found for this domain');
    }
    
    // Get entry info
    const tx = new Transaction();
    tx.moveCall({
        target: `${PACKAGE_ID}::vault::get_entry_info`,
        arguments: [
            tx.object(vaultId),
            tx.pure.vector('u8', domainHash),
        ],
    });
    
    const result = await suiClient.devInspectTransactionBlock({
        sender: walletAddress,
        transactionBlock: tx,
    });
    
    // Parse results
    const data = new Uint8Array(result.results[0].returnValues[2][0]);
    const entryNonce = new Uint8Array(result.results[0].returnValues[3][0]);
    
    // Decrypt
    const masterPassword = prompt('Enter master password:');
    const salt = getUserSalt();
    const masterKey = await deriveMasterKey(masterPassword, salt);
    
    return decryptCredentials(data, entryNonce, masterKey);
}
```

### 4. Updating Credentials
```typescript
async function updatePassword(domain: string, newUsername: string, newPassword: string) {
    const vaultId = localStorage.getItem('safekey_vault_id');
    const domainHash = Array.from(hashDomain(domain));
    
    // Get master key
    const masterPassword = prompt('Enter master password:');
    const salt = getUserSalt();
    const masterKey = await deriveMasterKey(masterPassword, salt);
    
    // Encrypt new credentials
    const { data, nonce: entryNonce } = encryptCredentials(newUsername, newPassword, masterKey);
    const sessionNonce = randomBytes(24);
    
    // Update on blockchain
    const tx = new Transaction();
    tx.moveCall({
        target: `${PACKAGE_ID}::vault::update_entry`,
        arguments: [
            tx.object(vaultId),
            tx.pure.vector('u8', domainHash),
            tx.pure.vector('u8', Array.from(data)),
            tx.pure.vector('u8', Array.from(entryNonce)),
            tx.pure.vector('u8', Array.from(sessionNonce)),
            tx.object('0x6'),
        ],
    });
    
    await signAndExecuteTransaction({ transaction: tx });
}
```

### 5. Deleting Credentials
```typescript
async function deletePassword(domain: string) {
    const vaultId = localStorage.getItem('safekey_vault_id');
    const domainHash = Array.from(hashDomain(domain));
    
    const tx = new Transaction();
    tx.moveCall({
        target: `${PACKAGE_ID}::vault::delete_entry`,
        arguments: [
            tx.object(vaultId),
            tx.pure.vector('u8', domainHash),
        ],
    });
    
    await signAndExecuteTransaction({ transaction: tx });
}
```

---

## Best Practices

### Security
1. **Never store master password**: Master password should only exist in memory during encryption/decryption
2. **Use strong KDF**: Use PBKDF2 with 100,000+ iterations or Argon2id
3. **Random nonces**: Always generate cryptographically secure random nonces
4. **Clear sensitive data**: Zero out encryption keys and plaintext passwords after use
5. **Salt storage**: Store user-specific salt securely (can be derived from wallet address)

### Gas Optimization
1. **Batch operations**: If adding multiple entries, consider batching in a single transaction
2. **Data size**: Keep credential data compact (only store essential information)
3. **Check existence**: Use `entry_exists` before `add_entry` to provide better UX

### Error Handling
```typescript
try {
    await addPassword(domain, username, password);
} catch (error) {
    if (error.message.includes('EEntryAlreadyExists')) {
        console.error('Credentials already exist for this domain. Use update instead.');
    } else if (error.message.includes('ENotAuthorized')) {
        console.error('You do not own this vault.');
    } else if (error.message.includes('EEntryNotFound')) {
        console.error('No credentials found for this domain.');
    } else {
        console.error('Transaction failed:', error);
    }
}
```

---

## Querying User's Vault

To find a user's vault object:

```typescript
async function findUserVault(ownerAddress: string): Promise<string | null> {
    const objects = await suiClient.getOwnedObjects({
        owner: ownerAddress,
        filter: {
            StructType: `${PACKAGE_ID}::vault::UserVault`
        },
    });
    
    return objects.data[0]?.data?.objectId || null;
}
```

---

## Testing

### Unit Tests (Move)
```bash
sui move test
```

### Integration Tests (TypeScript)
```typescript
describe('SafeKey Vault', () => {
    it('should create a vault', async () => {
        // Test vault creation
    });
    
    it('should add and retrieve credentials', async () => {
        // Test add/get flow
    });
    
    it('should update existing credentials', async () => {
        // Test update flow
    });
    
    it('should delete credentials', async () => {
        // Test deletion
    });
    
    it('should prevent unauthorized access', async () => {
        // Test authorization
    });
});
```

---

## FAQ

**Q: Can I have multiple vaults?**
A: Yes, you can create multiple vaults, but typically one vault per user is recommended for simplicity.

**Q: What happens if I lose my master password?**
A: There is no password recovery. All data is encrypted client-side, so losing the master password means losing access to all credentials permanently.

**Q: Can someone else read my encrypted data?**
A: The encrypted data is publicly readable on the blockchain, but without your master password, it's cryptographically impossible to decrypt.

**Q: What's the maximum size for credential data?**
A: While there's no hard limit, keeping data under 10KB is recommended for gas efficiency.

**Q: How do I list all my saved domains?**
A: Currently, you need to maintain a local index of domain hashes. The contract stores entries as dynamic fields which aren't easily enumerable. Consider storing a list of domains in your frontend's local storage.

---

## Support & Resources

- **Contract Source**: [GitHub Repository]
- **Sui Documentation**: https://docs.sui.io
- **Sui TypeScript SDK**: https://sdk.mystenlabs.com/typescript
- **Report Issues**: [GitHub Issues]

---

**Version:** 1.0.0  
**Last Updated:** October 2025  
**License:** MIT
```
