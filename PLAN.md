# clawmark — Blockchain Core Plan

**Role:** Core blockchain node, consensus engine, cryptographic primitives, ledger, and SDK
**Build Priority:** 1 (all other repos depend on this)

---

## 1. Workspace Structure

```
clawmark/
├── Cargo.toml                      # workspace root
├── CLAWMARK_Whitepaper_v0.1.md
├── PLAN.md
├── proto/                          # protobuf API definitions (canonical contract)
│   ├── node.proto
│   ├── wallet.proto
│   ├── contract.proto
│   └── mint.proto
├── crates/
│   ├── cmk-types/                  # foundational domain types
│   ├── cmk-crypto/                 # PQC cryptographic primitives
│   ├── cmk-mint/                   # CCSI generation, coin minting/burning
│   ├── cmk-ledger/                 # chain state, CCSI registry, accounts
│   ├── cmk-payload/                # encrypted block payload, selective disclosure
│   ├── cmk-contracts/              # CMK Work Contract runtime
│   ├── cmk-zkp/                    # zk-SNARK proof generation/verification
│   ├── cmk-consensus/              # HotStuff BFT consensus engine
│   ├── cmk-wallet/                 # wallet management (human + agent)
│   ├── cmk-net/                    # P2P networking between validators
│   ├── cmk-api/                    # gRPC + REST API server
│   ├── cmk-sdk/                    # client SDK for downstream repos
│   └── cmk-node/                   # node binary (ties everything together)
└── tests/                          # integration tests
    ├── mint_test.rs
    ├── transfer_test.rs
    └── consensus_test.rs
```

---

## 2. Crate Details

### 2.1 `cmk-types` — Domain Types

**Dependencies:** `serde`, `bincode`, `blake3`, `hex`

Core types shared across all crates:

```rust
// Cryptographic Coin Serial ID — ML-DSA signature (~2,420 bytes)
pub struct Ccsi(pub Vec<u8>);

// SHA3-256 commitment of a CCSI (32 bytes, stored on-chain)
pub struct CcsiCommitment(pub [u8; 32]);

// A single indivisible coin
pub struct Coin {
    pub ccsi: Ccsi,
    pub commitment: CcsiCommitment,
    pub mint_timestamp_ns: u128,
    pub mint_counter: u128,
    pub status: CoinStatus, // Active, Escrowed, Burned
}

// Wallet address — BLAKE3 hash of public key
pub struct WalletAddress(pub [u8; 32]);

pub struct Transaction {
    pub id: TransactionId,
    pub sender: WalletAddress,
    pub receiver: WalletAddress,
    pub coin_commitments: Vec<CcsiCommitment>,
    pub timestamp_ns: u128,
    pub signature: Vec<u8>,  // ML-DSA signature
}

pub struct Block {
    pub height: u64,
    pub timestamp_ns: u128,
    pub prev_hash: [u8; 32],        // SHA3-256
    pub merkle_root: [u8; 32],      // BLAKE3 Merkle root of commitments
    pub encrypted_payload: Vec<u8>, // AES-256-GCM encrypted
    pub zk_proof: Option<Vec<u8>>,  // Groth16 proof (Phase 2)
    pub validator_signatures: Vec<ValidatorSignature>,
}

pub enum ContractStatus {
    Draft, Funded, Accepted, Completed, Settled, Disputed,
}

pub struct WorkContract {
    pub id: ContractId,
    pub requester: WalletAddress,
    pub worker: Option<WalletAddress>,
    pub escrowed_commitments: Vec<CcsiCommitment>,
    pub work_description_hash: [u8; 32],
    pub deadline_ns: u128,
    pub status: ContractStatus,
}
```

### 2.2 `cmk-crypto` — Cryptographic Primitives

**Dependencies:** `pqcrypto-dilithium`, `pqcrypto-kyber`, `pqcrypto-sphincsplus`, `pqcrypto-traits`, `blake3`, `sha3`, `aes-gcm`, `p256`, `ecdsa`, `rand`

Public API:

```rust
// ML-DSA (Dilithium) — primary signatures
pub fn mldsa_keygen() -> (MldsaPublicKey, MldsaSecretKey);
pub fn mldsa_sign(sk: &MldsaSecretKey, msg: &[u8]) -> MldsaSignature;
pub fn mldsa_verify(pk: &MldsaPublicKey, msg: &[u8], sig: &MldsaSignature) -> bool;

// ML-KEM (Kyber) — key encapsulation for payload encryption
pub fn mlkem_keygen() -> (MlkemPublicKey, MlkemSecretKey);
pub fn mlkem_encapsulate(pk: &MlkemPublicKey) -> (SharedSecret, Ciphertext);
pub fn mlkem_decapsulate(sk: &MlkemSecretKey, ct: &Ciphertext) -> SharedSecret;

// AES-256-GCM — symmetric payload encryption
pub fn aes_encrypt(key: &[u8; 32], nonce: &[u8; 12], plaintext: &[u8]) -> Vec<u8>;
pub fn aes_decrypt(key: &[u8; 32], nonce: &[u8; 12], ciphertext: &[u8]) -> Result<Vec<u8>>;

// Hashing
pub fn blake3_hash(data: &[u8]) -> [u8; 32];
pub fn sha3_256_hash(data: &[u8]) -> [u8; 32];

// Hybrid signatures (ML-DSA + ECDSA P-256) — transition period
pub fn hybrid_sign(mldsa_sk: &MldsaSecretKey, ecdsa_sk: &EcdsaSecretKey, msg: &[u8]) -> HybridSignature;
pub fn hybrid_verify(mldsa_pk: &MldsaPublicKey, ecdsa_pk: &EcdsaPublicKey, msg: &[u8], sig: &HybridSignature) -> bool;

// SLH-DSA (SPHINCS+) — backup non-lattice signatures
pub fn slhdsa_keygen() -> (SlhdsaPublicKey, SlhdsaSecretKey);
pub fn slhdsa_sign(sk: &SlhdsaSecretKey, msg: &[u8]) -> SlhdsaSignature;
pub fn slhdsa_verify(pk: &SlhdsaPublicKey, msg: &[u8], sig: &SlhdsaSignature) -> bool;
```

### 2.3 `cmk-mint` — CCSI Generation & Minting

**Dependencies:** `cmk-types`, `cmk-crypto`

Implements the CCSI generation protocol from Whitepaper Section 3.2:

1. Combine: mint timestamp (ns) + 256-bit nonce + 128-bit counter + authority pubkey fingerprint
2. BLAKE3 hash the combined input → raw serial candidate
3. ML-DSA sign the hash with Authority key → this signature IS the CCSI (~2,420 bytes)
4. SHA3-256 hash the CCSI → on-chain commitment (32 bytes)

```rust
pub struct MintEngine {
    authority_sk: MldsaSecretKey,
    authority_pk: MldsaPublicKey,
    counter: AtomicU128,
}

impl MintEngine {
    pub fn mint_coin(&self) -> Coin;
    pub fn mint_batch(&self, count: u64) -> Vec<Coin>;
    pub fn burn_coin(&self, commitment: &CcsiCommitment) -> BurnReceipt;
    pub fn verify_ccsi(authority_pk: &MldsaPublicKey, ccsi: &Ccsi, commitment: &CcsiCommitment) -> bool;
}
```

### 2.4 `cmk-ledger` — Chain State

**Dependencies:** `cmk-types`, `cmk-crypto`, `rocksdb`, `serde`, `bincode`

```rust
pub struct Ledger {
    db: rocksdb::DB,
}

impl Ledger {
    // Block operations
    pub fn append_block(&mut self, block: Block) -> Result<()>;
    pub fn get_block(&self, height: u64) -> Result<Option<Block>>;
    pub fn get_latest_block(&self) -> Result<Option<Block>>;
    pub fn chain_height(&self) -> u64;

    // CCSI registry
    pub fn register_coin(&mut self, commitment: CcsiCommitment, owner: WalletAddress) -> Result<()>;
    pub fn get_coin_owner(&self, commitment: &CcsiCommitment) -> Result<Option<WalletAddress>>;
    pub fn transfer_coin(&mut self, commitment: &CcsiCommitment, new_owner: WalletAddress) -> Result<()>;
    pub fn is_double_spend(&self, commitment: &CcsiCommitment) -> bool;

    // Account balances
    pub fn get_balance(&self, wallet: &WalletAddress) -> u64;
    pub fn get_coins_for_wallet(&self, wallet: &WalletAddress) -> Vec<CcsiCommitment>;

    // Escrow
    pub fn escrow_coins(&mut self, commitments: &[CcsiCommitment], contract_id: &ContractId) -> Result<()>;
    pub fn release_escrow(&mut self, contract_id: &ContractId, to: WalletAddress) -> Result<()>;
}
```

RocksDB column families:
- `blocks` — height → serialized Block
- `ccsi_registry` — CcsiCommitment → (WalletAddress, CoinStatus)
- `accounts` — WalletAddress → list of CcsiCommitments
- `contracts` — ContractId → WorkContract
- `metadata` — chain height, genesis hash, etc.

### 2.5 `cmk-payload` — Encrypted Block Payloads

**Dependencies:** `cmk-types`, `cmk-crypto`

Implements Whitepaper Section 5.2 — hybrid encryption with selective disclosure:

```rust
pub struct PayloadBuilder;

impl PayloadBuilder {
    /// Encrypt a transaction payload for sender, receiver, and authority
    pub fn encrypt(
        transactions: &[Transaction],
        sender_pk: &MlkemPublicKey,
        receiver_pk: &MlkemPublicKey,
        authority_pk: &MlkemPublicKey,
    ) -> EncryptedPayload;

    /// Decrypt a payload using a recipient's secret key
    pub fn decrypt(
        payload: &EncryptedPayload,
        recipient_sk: &MlkemSecretKey,
    ) -> Result<Vec<Transaction>>;
}

pub struct EncryptedPayload {
    pub ciphertext: Vec<u8>,                // AES-256-GCM encrypted
    pub sender_encapsulation: Vec<u8>,      // ML-KEM ciphertext for sender
    pub receiver_encapsulation: Vec<u8>,    // ML-KEM ciphertext for receiver
    pub authority_encapsulation: Vec<u8>,   // ML-KEM ciphertext for authority
    pub nonce: [u8; 12],                    // AES-GCM nonce
}
```

### 2.6 `cmk-contracts` — Work Contract Runtime (Phase 2)

**Dependencies:** `cmk-types`, `cmk-ledger`, `cmk-crypto`

State machine for the CMK Work Contract lifecycle:

```
Draft → Funded → Accepted → Completed → Settled
                                     ↘ Disputed → Settled
```

```rust
pub struct ContractEngine {
    ledger: Arc<Mutex<Ledger>>,
}

impl ContractEngine {
    pub fn create_contract(&self, draft: ContractDraft) -> Result<ContractId>;
    pub fn fund_contract(&self, id: &ContractId, coins: &[CcsiCommitment], sig: &MldsaSignature) -> Result<()>;
    pub fn accept_contract(&self, id: &ContractId, worker: WalletAddress, sig: &MldsaSignature) -> Result<()>;
    pub fn submit_completion(&self, id: &ContractId, proof: CompletionProof) -> Result<()>;
    pub fn settle_contract(&self, id: &ContractId) -> Result<()>;
    pub fn dispute_contract(&self, id: &ContractId, reason: DisputeReason, sig: &MldsaSignature) -> Result<()>;
}
```

### 2.7 `cmk-zkp` — Zero-Knowledge Proofs (Phase 2)

**Dependencies:** `cmk-types`, `ark-groth16`, `ark-bn254`, `ark-relations`, `ark-std`

Generates Groth16 zk-SNARK proofs per block proving:
- All CCSIs are legitimately issued
- No CCSI appears more than once (no double-spend)
- Incoming and outgoing coin values balance
- All party signatures are valid

```rust
pub struct ZkProver { /* trusted setup params */ }
pub struct ZkVerifier { /* verification key */ }

impl ZkProver {
    pub fn prove_block_validity(&self, block: &Block, witness: &BlockWitness) -> Result<Proof>;
}

impl ZkVerifier {
    pub fn verify_block_proof(&self, block_header: &BlockHeader, proof: &Proof) -> bool;
}
```

### 2.8 `cmk-consensus` — HotStuff BFT

**Dependencies:** `cmk-types`, `cmk-crypto`, `cmk-net`, `tokio`

Linear-complexity BFT consensus (O(n) messages):
- Deterministic finality — no forks, no reorgs
- Tolerates f Byzantine failures in 3f+1 validators
- Sub-second block times with 7-21 validators
- Leader rotation per view

```rust
pub struct HotStuffNode {
    validator_id: ValidatorId,
    validators: Vec<ValidatorInfo>,
    ledger: Arc<Mutex<Ledger>>,
    network: Arc<Network>,
}

impl HotStuffNode {
    pub async fn run(&mut self);  // main consensus loop
    pub async fn propose_block(&self, transactions: Vec<Transaction>) -> Result<Block>;
    pub async fn vote(&self, block: &Block) -> Vote;
    pub async fn on_proposal(&mut self, proposal: Proposal) -> Result<()>;
    pub async fn on_vote(&mut self, vote: Vote) -> Result<()>;
}
```

Phases per round: PREPARE → PRE-COMMIT → COMMIT → DECIDE

### 2.9 `cmk-wallet` — Wallet Management

**Dependencies:** `cmk-types`, `cmk-crypto`, `serde`, `zeroize`

```rust
pub struct Wallet {
    pub address: WalletAddress,
    pub mldsa_pk: MldsaPublicKey,
    mldsa_sk: MldsaSecretKey,      // zeroized on drop
    pub mlkem_pk: MlkemPublicKey,
    mlkem_sk: MlkemSecretKey,       // zeroized on drop
    pub wallet_type: WalletType,    // Human or Agent
    pub principal_chain: Option<Vec<WalletAddress>>, // agent → operator chain
}

impl Wallet {
    pub fn generate_human() -> Self;
    pub fn generate_agent(operator: WalletAddress) -> Self;
    pub fn sign_transaction(&self, tx: &Transaction) -> MldsaSignature;
    pub fn decrypt_payload(&self, payload: &EncryptedPayload) -> Result<Vec<Transaction>>;
    pub fn export_public_info(&self) -> WalletPublicInfo;
}
```

### 2.10 `cmk-net` — P2P Networking

**Dependencies:** `cmk-types`, `libp2p`, `tokio`, `serde`

```rust
pub struct Network {
    swarm: libp2p::Swarm<CmkBehaviour>,
}

impl Network {
    pub async fn start(config: NetworkConfig) -> Self;
    pub async fn broadcast_proposal(&self, proposal: Proposal);
    pub async fn send_vote(&self, peer: PeerId, vote: Vote);
    pub async fn broadcast_transaction(&self, tx: Transaction);
    pub async fn recv_message(&mut self) -> NetworkMessage;
}
```

### 2.11 `cmk-api` — gRPC + REST Server

**Dependencies:** `cmk-types`, `cmk-ledger`, `cmk-wallet`, `cmk-contracts`, `cmk-mint`, `tonic`, `axum`, `tower`

- gRPC services: `NodeService`, `WalletService`, `ContractService`, `MintService`
- REST endpoints via Axum for admin dashboard and simple queries
- Implements the proto definitions from `proto/`

### 2.12 `cmk-sdk` — Client SDK

**Dependencies:** `cmk-types`, `tonic`, `tokio`

Published crate consumed by claw-bridge, clawmark-org, and clawmark-net:

```rust
pub struct ClawmarkClient {
    node: NodeServiceClient<Channel>,
    wallet: WalletServiceClient<Channel>,
    contract: ContractServiceClient<Channel>,
    mint: MintServiceClient<Channel>,
}

impl ClawmarkClient {
    pub async fn connect(endpoint: &str) -> Result<Self>;
    pub async fn get_balance(&self, wallet: &WalletAddress) -> Result<u64>;
    pub async fn transfer(&self, from: &Wallet, to: &WalletAddress, amount: u64) -> Result<TransactionId>;
    pub async fn create_contract(&self, draft: ContractDraft) -> Result<ContractId>;
    pub async fn fund_contract(&self, wallet: &Wallet, contract_id: &ContractId, amount: u64) -> Result<()>;
    pub async fn get_block(&self, height: u64) -> Result<Block>;
    pub async fn stream_blocks(&self) -> Result<impl Stream<Item = Block>>;
    pub async fn get_circulating_supply(&self) -> Result<u64>;
}
```

### 2.13 `cmk-node` — Node Binary

**Dependencies:** all crates, `clap`, `config`, `toml`, `tracing-subscriber`

```
cmk-node --config node.toml

  --role validator|observer
  --listen-addr /ip4/0.0.0.0/tcp/9000
  --rpc-addr 0.0.0.0:50051
  --rest-addr 0.0.0.0:8080
  --data-dir /var/lib/clawmark
```

---

## 3. Build Order

```
Phase 1 (Months 3-9):
  cmk-types → cmk-crypto → cmk-mint → cmk-ledger → cmk-payload
  → cmk-wallet → cmk-consensus → cmk-net → cmk-api → cmk-sdk → cmk-node

Phase 2 (Months 9-15):
  cmk-contracts → cmk-zkp
```

---

## 4. Key Dependencies

| Crate | Version | Purpose |
|-------|---------|---------|
| `pqcrypto-dilithium` | latest | ML-DSA signatures |
| `pqcrypto-kyber` | latest | ML-KEM key encapsulation |
| `pqcrypto-sphincsplus` | latest | SLH-DSA backup signatures |
| `pqcrypto-traits` | latest | Common PQC traits |
| `blake3` | 1.x | Fast hashing |
| `sha3` | 0.10.x | SHA3-256 commitments |
| `aes-gcm` | 0.10.x | Symmetric encryption |
| `p256` | 0.13.x | ECDSA P-256 (hybrid transition) |
| `ecdsa` | 0.16.x | ECDSA traits |
| `ark-groth16` | 0.5.x | zk-SNARK proofs |
| `ark-bn254` | 0.5.x | BN254 curve for Groth16 |
| `ark-relations` | 0.5.x | Constraint system |
| `rocksdb` | 0.22.x | Persistent KV storage |
| `libp2p` | 0.54.x | P2P networking |
| `tonic` | 0.12.x | gRPC server/client |
| `prost` | 0.13.x | Protobuf codegen |
| `axum` | 0.8.x | HTTP/REST |
| `tokio` | 1.x | Async runtime |
| `serde` | 1.x | Serialization |
| `bincode` | 1.x | Binary encoding |
| `clap` | 4.x | CLI parsing |
| `tracing` | 0.1.x | Structured logging |
| `zeroize` | 1.x | Secure memory cleanup |
| `rand` | 0.8.x | Cryptographic RNG |
| `proptest` | 1.x | Property-based testing |
| `criterion` | 0.5.x | Benchmarks |

---

## 5. Verification

1. **Unit tests:** every crate has `#[cfg(test)]` modules
   - `cmk-crypto`: roundtrip sign/verify, encrypt/decrypt, hash consistency
   - `cmk-mint`: CCSI generation uniqueness, verification, batch minting
   - `cmk-ledger`: block append/get, double-spend detection, balance tracking
   - `cmk-payload`: encrypt/decrypt roundtrip for all three recipients
   - `cmk-consensus`: state machine transitions, vote counting

2. **Integration tests** (`tests/`):
   - Spin up single node, mint 100 coins, transfer between wallets, verify CCSIs
   - Multi-node consensus: 4 nodes, propose blocks, verify agreement
   - Encrypted payload: encrypt, transmit, decrypt by each party

3. **Benchmarks** (`benches/`):
   - ML-DSA sign/verify throughput
   - CCSI generation rate
   - BLAKE3 vs SHA3-256 hashing throughput
   - Block serialization/deserialization
   - Target: 10,000+ TPS for transaction processing

4. **CI:** GitHub Actions
   - `cargo fmt --check`
   - `cargo clippy -- -D warnings`
   - `cargo test --workspace`
   - `cargo bench` (regression tracking)

---

*clawmark PLAN v0.1 — April 2026*
