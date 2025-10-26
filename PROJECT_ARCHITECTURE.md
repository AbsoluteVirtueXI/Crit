# Crit - Ledger-Based Cryptocurrency Protocol

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Core Components](#core-components)
  - [Transaction Layer](#transaction-layer)
  - [Mempool Layer](#mempool-layer)
  - [State Layer](#state-layer)
  - [Account Layer](#account-layer)
  - [Wallet Layer](#wallet-layer)
  - [Currency Layer](#currency-layer)
  - [Cryptography Layer](#cryptography-layer)
  - [Network Layer](#network-layer)
- [Data Flow](#data-flow)
- [Transaction Lifecycle](#transaction-lifecycle)
- [Design Patterns](#design-patterns)
- [Project Structure](#project-structure)

---

## Overview

**Crit** is a Rust implementation of a ledger-based cryptocurrency protocol featuring:

- **Transaction validation** with two-layer pipeline (stateless + stateful)
- **State management** with in-memory account snapshots
- **Transaction mempool** for buffering pending operations
- **Ed25519 signatures** for authentication
- **BLAKE3 hashing** for transaction IDs
- **Three-pool balance model**: available, staked, and reward balances
- **Network isolation** preventing cross-chain replay attacks
- **0.1% transaction fees** with anti-spam minimum

### Transaction Types

1. **Transfer**: Send funds between accounts
2. **Stake**: Lock funds for validator participation
3. **Unstake**: Recover staked funds to available balance
4. **ClaimReward**: Convert accrued rewards to spendable balance

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Crit Ledger System                   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │     Transaction Layer (tx.rs)                    │   │
│  │  • Tx: Transaction primitives                    │   │
│  │  • TxKind: Transfer, Stake, Unstake, ClaimReward │   │
│  │  • TransactionCollector: Stateless validation    │   │
│  └──────────────────────────────────────────────────┘   │
│                          ↓                              │
│  ┌──────────────────────────────────────────────────┐   │
│  │     Mempool Layer (mempool.rs)                   │   │
│  │  • Mempool<S>: Stateful transaction pool         │   │
│  │  • State-aware validation (balances, nonces)     │   │
│  └──────────────────────────────────────────────────┘   │
│                          ↓                              │
│  ┌──────────────────────────────────────────────────┐   │
│  │     State Layer (state.rs)                       │   │
│  │  • State: In-memory account snapshots            │   │
│  │  • AccountStateView: Read-only interface         │   │
│  └──────────────────────────────────────────────────┘   │
│                          ↓                              │
│  ┌──────────────────────────────────────────────────┐   │
│  │     Account Layer (account.rs)                   │   │
│  │  • Account: Stateful account with wallet         │   │
│  │  • Nonce management                              │   │
│  └──────────────────────────────────────────────────┘   │
│                          ↓                              │
│  ┌──────────────────────────────────────────────────┐   │
│  │     Wallet Layer (wallet.rs)                     │   │
│  │  • Wallet: Balance tracking                      │   │
│  │  • Three pools: available, staked, reward        │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │     Supporting Modules                           │   │
│  │  • crypto.rs: Ed25519, BLAKE3 hashing            │   │
│  │  • currency.rs: Crit denomination, fees          │   │
│  │  • network.rs: MAIN_NET, TEST_NET                │   │
│  │  • random.rs: OS-backed RNG                      │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Core Components

### Transaction Layer

**File**: `src/tx.rs` (430 lines)

#### `TxKind` - Transaction Variants

```rust
pub enum TxKind {
    Transfer { to: AccountId, amount: Crit },
    Stake { amount: Crit },
    Unstake { amount: Crit },
    ClaimReward { amount: Crit },
}
```

#### `Tx` - Transaction Structure

```rust
pub struct Tx {
    pub network_id: NetId,           // Prevents cross-chain replay
    pub from: AccountId,             // Sender's Ed25519 public key (32 bytes)
    pub nonce: u64,                  // Prevents replay, orders transactions
    pub kind: TxKind,                // Transaction payload
    pub signature: Option<Signature>, // Ed25519 signature (64 bytes)
}
```

**Key Methods**:

- `Tx::sign(signing_key)` - Signs transaction using Ed25519 over BLAKE3 hash of `(network_id, from, nonce, kind)`
- `Tx::verify_signature()` - Verifies embedded signature
- `Tx::tx_id()` - Computes BLAKE3 hash over entire serialized transaction
- `Tx::fee()` - Computes 0.1% fee (minimum 1 centcrit)

**Design Note**: Signature is excluded from signed payload to avoid circular dependencies. The transaction ID includes the signature for immutability proof.

#### `TransactionCollector` - Stateless Validation

Buffers transactions and performs checks:

- Network ID matches
- Signature verification
- Non-zero amounts
- No self-transfers
- Valid recipient keys

```rust
let mut collector = TransactionCollector::new(MAIN_NET);
collector.push(signed_tx)?; // Validates before accepting
```

---

### Mempool Layer

**File**: `src/mempool.rs` (295 lines)

#### `Mempool<S>` - Transaction Pool

Generic over `AccountStateView` for flexible state sources.

```rust
pub struct Mempool<S> {
    state: S,                           // Account snapshot provider
    entries: HashMap<[u8; 32], Tx>,     // Keyed by tx_id
}
```

**Validation Checks**:

1. Account exists in state view
2. Nonce matches expected value (prevents gaps)
3. Sufficient balances to cover amount + fee:
   - **Transfer/Stake**: `available ≥ amount + fee`
   - **Unstake**: `staked ≥ amount` AND `available ≥ fee`
   - **ClaimReward**: `reward ≥ amount` AND `available ≥ fee`
4. Transaction not already in pool (duplicate detection)

**Key Methods**:

```rust
let mut pool = Mempool::new(state);
pool.insert(tx)?;           // Validates and enqueues
pool.iter();                // Returns all buffered transactions
pool.clear();               // Drains all transactions
```

**Design Rationale**: Generic design allows mempool to work with any state source (in-memory, persistent storage, mock for testing).

---

### State Layer

**File**: `src/state.rs` (192 lines)

#### `AccountSnapshot` - Read-Only Account View

```rust
pub struct AccountSnapshot {
    pub nonce: u64,          // Next expected nonce
    pub available: Crit,     // Spendable balance
    pub staked: Crit,        // Locked in validator stake
    pub reward: Crit,        // Accrued rewards
}
```

#### `AccountStateView` - Trait for State Access

```rust
pub trait AccountStateView {
    fn network_id(&self) -> NetId;
    fn account_snapshot(&self, account_id: &AccountId) -> Option<AccountSnapshot>;
    fn nonce(&self, account_id: &AccountId) -> Option<u64>;
    fn available_balance(&self, account_id: &AccountId) -> Option<Crit>;
}
```

#### `State` - In-Memory Implementation

```rust
pub struct State {
    network_id: NetId,
    accounts: HashMap<AccountKey, AccountSnapshot>,
}
```

**Key Methods**:

```rust
let mut state = State::new(MAIN_NET);
state.sync_account(&account_id, &account);      // Sync from Account object
state.insert_snapshot(account_id, snapshot);    // Update snapshot
state.load_from_iter(entries);                  // Bulk hydration
```

**Design Rationale**: Trait-based design allows mempool to work with any state source. Snapshots are copyable for cheap passing to validators.

---

### Account Layer

**File**: `src/account.rs` (348 lines)

#### `Account` - Stateful Account

```rust
pub struct Account {
    network_id: NetId,
    nonce: u64,          // Monotonic counter
    wallet: Wallet,      // Three-pool balance
}
```

**Key Methods** (all increment nonce only on success):

```rust
account.deposit(amount)?;          // Credit available balance
account.withdraw(amount)?;         // Debit available balance
account.stake(amount)?;            // Move available → staked
account.unstake(amount)?;          // Move staked → available
account.receive_reward(amount)?;   // Credit reward balance
account.claim_reward(amount)?;     // Move reward → available
```

**Atomic Nonce Updates**:

```rust
fn apply_with_nonce<F>(&mut self, op: F) -> Result<(), AccountError>
where
    F: FnOnce(&mut Wallet) -> Result<(), WalletError>,
{
    let new_nonce = self.next_nonce()?;  // Check overflow first
    op(&mut self.wallet)?;                // Apply operation
    self.nonce = new_nonce;               // Only increment if both succeed
    Ok(())
}
```

**Design Rationale**: Nonce incremented only after wallet operation succeeds, preventing nonce gaps from failed operations.

---

### Wallet Layer

**File**: `src/wallet.rs` (296 lines)

#### `Wallet` - Three-Pool Balance Model

```rust
pub struct Wallet {
    available: Crit,    // Spendable funds
    staked: Crit,       // Locked in validator participation
    reward: Crit,       // Accrued but unclaimed rewards
}
```

**Balance Operations**:

```rust
wallet.credit_available(amount)?;   // Add to available
wallet.debit_available(amount)?;    // Remove from available
wallet.credit_staked(amount)?;      // Move available → staked
wallet.debit_staked(amount)?;       // Move staked → available
wallet.credit_reward(amount)?;      // Add to reward
wallet.debit_reward(amount)?;       // Move reward → available
```

**Design Rationale**:

- **available**: Supports transfers and fee payment
- **staked**: Supports validator participation (consensus mechanism)
- **reward**: Tracks incentive distribution separately
- All operations check for overflow/underflow and are atomic (fail with no mutation)

---

### Currency Layer

**File**: `src/currency.rs` (240 lines)

#### `Crit` - Fixed-Precision Amount

```rust
pub struct Crit(u128); // Raw centcrit units

const DECIMAL: u32 = 8;                    // 8 decimal places
const UNIT: u128 = 100_000_000;            // 1 CRIT = 10^8 centcrit
```

**Constants**:

- `Crit::ZERO`, `Crit::ONE`
- `Crit::NAME = "CRIT"`

**Safe Arithmetic**:

```rust
crit.checked_add(amount)?;    // Returns Result<Crit, CurrencyError>
crit.checked_sub(amount)?;
crit.add_assign(amount)?;     // In-place mutation
crit.sub_assign(amount)?;
```

#### Fee Calculation

```rust
pub fn compute_fee(amount: Crit) -> Result<Crit, CurrencyError> {
    // Formula: amount * 1 / 1_000 (0.1%)
    // Minimum: 1 centcrit for any non-zero amount
}
```

**Design Rationale**:

- `u128` avoids overflow for typical transaction amounts
- Fixed-point avoids floating-point precision issues
- 0.1% fee matches common cryptocurrency conventions
- Minimum fee prevents zero-fee spam attacks

---

### Cryptography Layer

**File**: `src/crypto.rs` (103 lines)

#### Functions

```rust
pub fn generate_keypair() -> (VerifyingKey, SigningKey) {
    // Creates Ed25519 keypair using OS RNG
}

pub fn blake3_hash(input: &[u8]) -> Blake3Hash {
    // Returns 32-byte BLAKE3 hash
}
```

**Design Rationale**:

- **Ed25519**: Fast, high-security signature scheme (32-byte keys, 64-byte signatures)
- **BLAKE3**: Fast cryptographic hash with 32-byte output
- **OS RNG**: Suitable for secure keypair generation

---

### Network Layer

**File**: `src/network.rs` (16 lines)

#### Network Identifiers

```rust
pub type NetId = u8;

pub const MAIN_NET: NetId = 1;  // Production blockchain
pub const TEST_NET: NetId = 2;  // Public testnet
```

**Design Rationale**:

- Transactions include `network_id` in signed payload
- Prevents replaying testnet transactions on mainnet or vice versa
- Compact (1 byte) for serialization efficiency

---

## Data Flow

### Transaction Lifecycle

```
1. CREATION & SIGNING
   ├─ generate_account_keys()          [crypto.rs]
   ├─ Tx::new(network_id, from, nonce, kind)
   └─ tx.sign(signing_key)             [uses blake3_hash]

2. COLLECTION (STATELESS VALIDATION)
   ├─ TransactionCollector::new(network_id)
   └─ collector.push(tx)
        ├─ check_network()              [verify tx.network_id matches]
        ├─ check_signature()            [verify Ed25519 signature]
        ├─ check_amount()               [ensure tx.kind.amount() > 0]
        └─ check_transfer_rules()       [no self-transfer, valid recipient]

3. POOLING (STATE-AWARE VALIDATION)
   ├─ State::load_from_iter(entries)   [hydrate from storage]
   ├─ Mempool::new(state)
   └─ mempool.insert(tx)
        ├─ account_snapshot(tx.from)   [lookup account in state]
        ├─ verify nonce matches
        ├─ verify fee + amount ≤ balance
        └─ prevent duplicates

4. EXECUTION (Future Implementation)
   ├─ Apply transaction to accounts
   ├─ Update Account state
   ├─ Increment Account.nonce
   └─ Sync State snapshots with updated Account
```

### Validation Layers

| Layer | Validation |
|-------|-----------|
| **TransactionCollector** | Network match, signature valid, amount > 0, no self-transfer, valid recipient key |
| **Mempool** | Account exists, nonce matches expected, available ≥ amount + fee (or appropriate balance for tx type), tx_id unique |
| **(Future Execution)** | Transaction effects (balance updates), reward distribution, validator state changes |

### Component Dependencies

```
main.rs
 └─ (placeholder, prints "Hello")

tx.rs
 ├─ account.rs (AccountId, Account)
 ├─ crypto.rs (Blake3Hash, blake3_hash)
 ├─ currency.rs (Crit, CurrencyError)
 ├─ network.rs (NetId)
 └─ (borsh, ed25519_dalek, thiserror)

mempool.rs
 ├─ tx.rs (Tx, TxError)
 ├─ state.rs (AccountStateView)
 ├─ currency.rs (CurrencyError)
 └─ (std::collections::HashMap)

state.rs
 ├─ account.rs (Account, AccountId, AccountSnapshot)
 ├─ currency.rs (Crit)
 ├─ network.rs (NetId)
 └─ (std::collections::HashMap)

account.rs
 ├─ crypto.rs (generate_keypair)
 ├─ currency.rs (Crit)
 ├─ network.rs (NetId)
 ├─ wallet.rs (Wallet, WalletError)
 └─ (ed25519_dalek)

wallet.rs
 └─ currency.rs (Crit, CurrencyError)

crypto.rs
 ├─ random.rs (crypto_rng)
 └─ (blake3, ed25519_dalek, rand_core)

currency.rs
 └─ (borsh for serialization)

network.rs
 └─ (no dependencies)

random.rs
 └─ (rand_core::OsRng)
```

---

## Transaction Lifecycle

### Example: Alice Sends Funds to Bob

```rust
use crit::{Account, Tx, TxKind, TransactionCollector, Mempool, State, Crit, MAIN_NET};

// 1. Create identities
let (alice_id, alice_key) = Account::generate_account_keys();
let (bob_id, bob_key) = Account::generate_account_keys();

// 2. Create transaction
let tx = Tx::new(
    MAIN_NET,
    alice_id,
    0,  // nonce
    TxKind::transfer(&bob_id, Crit::from_units(1_000_000))  // 0.01 CRIT
);

// 3. Sign
let signed_tx = tx.sign(&alice_key)?;

// 4. Stateless validation
let mut collector = TransactionCollector::new(MAIN_NET);
collector.push(signed_tx.clone())?;  // Validates signature, network, amount

// 5. Prepare state
let mut alice_account = Account::new(MAIN_NET);
alice_account.deposit(Crit::from_units(2_000_000))?;  // 0.02 CRIT

let mut state = State::new(MAIN_NET);
state.sync_account(&alice_id, &alice_account);

// 6. Mempool insertion (state-aware validation)
let mut pool = Mempool::new(state);
pool.insert(signed_tx)?;  // Validates nonce, balance

// 7. Ready for execution
assert_eq!(pool.len(), 1);
```

---

## Design Patterns

### 1. Two-Layer Validation Pipeline

```
TransactionCollector (stateless)
         ↓
      Mempool (stateful)
         ↓
    Execution Layer (future)
```

**Rationale**: Separates concerns — stateless checks (signatures, format) happen before expensive state lookups.

---

### 2. Trait-Based State Access

```rust
pub trait AccountStateView {
    fn network_id(&self) -> NetId;
    fn account_snapshot(&self, account_id: &AccountId) -> Option<AccountSnapshot>;
}
```

**Benefits**:

- `Mempool<S>` is generic over any `AccountStateView` implementation
- Allows in-memory `State` for testing
- Supports future persistent store implementations
- Enables mock states for unit tests

---

### 3. Atomic Nonce + Balance Updates

All `Account` mutation methods use this pattern:

```rust
fn apply_with_nonce<F>(&mut self, op: F) -> Result<(), AccountError>
where
    F: FnOnce(&mut Wallet) -> Result<(), WalletError>,
{
    let new_nonce = self.next_nonce()?;  // Check overflow first
    op(&mut self.wallet)?;                // Apply operation
    self.nonce = new_nonce;               // Only increment if both succeed
    Ok(())
}
```

**Rationale**: Prevents nonce gaps. If a wallet operation fails, nonce remains unchanged.

---

### 4. Three-Pool Balance Model

```
Wallet {
    available: Spendable funds
    staked:    Locked balance earning rewards
    reward:    Accrued rewards awaiting claim
}
```

**Rationale**:

- `available`: Supports transfers and fee payment
- `staked`: Supports validator participation (consensus mechanism)
- `reward`: Tracks incentive distribution separately

---

### 5. Fee Calculation with Minimum

```rust
pub fn compute_fee(amount: Crit) -> Result<Crit, CurrencyError> {
    if amount == Crit::ZERO {
        return Ok(Crit::ZERO);
    }
    let numerator = amount.0.checked_mul(FEE_NUMERATOR)?;
    let mut fee_units = numerator / FEE_DENOMINATOR;  // 0.1%

    // Round up
    if numerator % FEE_DENOMINATOR != 0 {
        fee_units = fee_units.checked_add(1)?;
    }

    // Enforce minimum
    if fee_units < MIN_FEE_UNITS {
        fee_units = MIN_FEE_UNITS;  // 1 centcrit
    }

    Ok(Crit(fee_units))
}
```

**Rationale**:

- 0.1% incentivizes larger transactions (economies of scale)
- Minimum 1 centcrit prevents zero-fee spam
- Rounding up biases in favor of protocol (prevents undercharging)

---

### 6. Borsh Serialization

All domain types derive `BorshSerialize`/`BorshDeserialize`:

- Transactions and signatures
- Currency amounts
- Account state snapshots

**Rationale**:

- Schema evolution is explicit
- Deterministic byte format (critical for signatures)
- Efficient and simple

---

### 7. Result Types for Error Handling

Each module defines its own error enum:

```rust
TxError          // Signing, verification, encoding
MempoolError     // Pool insertion failures
WalletError      // Balance operations
AccountError     // Nonce/wallet mutations
CurrencyError    // Arithmetic overflow/underflow
CollectError     // Transaction collection validation
```

**Rationale**:

- Context-specific error messages
- Callers can handle domain-specific failures
- Propagation via `#[from]` in `thiserror`

---

### 8. Network Scoping

Every major component is bound to a `NetId`:

```rust
Account { network_id: NetId, ... }
State { network_id: NetId, ... }
TransactionCollector { network_id: NetId, ... }
```

**Rationale**: Prevents accidental cross-network operations. Network ID is also included in transaction signed payload.

---

## Project Structure

```
/Crit/
├── Cargo.toml                    # Project manifest
│   ├── thiserror 2.0.17          # Error handling macros
│   ├── ed25519-dalek 2.2.0       # Ed25519 signatures
│   ├── rand_core 0.6             # RNG trait & OsRng
│   ├── borsh 1.5.7               # Serialization
│   └── blake3 1.8.2              # Hashing
│
└── src/
    ├── lib.rs                    # Public module exports
    ├── main.rs                   # Entry point (placeholder)
    │
    ├── tx.rs                     # Transactions (430 lines)
    │   ├── TxKind (enum)
    │   ├── Tx (struct)
    │   ├── TransactionCollector (struct)
    │   └── 20+ tests
    │
    ├── mempool.rs                # Transaction pool (295 lines)
    │   ├── Mempool<S> (generic)
    │   ├── MempoolError (enum)
    │   └── 7 tests
    │
    ├── state.rs                  # Account snapshots (192 lines)
    │   ├── AccountSnapshot (struct)
    │   ├── State (struct)
    │   ├── AccountStateView (trait)
    │   └── 3 tests
    │
    ├── account.rs                # Full accounts (348 lines)
    │   ├── Account (struct)
    │   ├── AccountId (type alias)
    │   ├── AccountError (enum)
    │   └── 14 tests
    │
    ├── wallet.rs                 # Balance tracking (296 lines)
    │   ├── Wallet (struct)
    │   ├── WalletError (enum)
    │   └── 12 tests
    │
    ├── currency.rs               # Denomination (240 lines)
    │   ├── Crit (struct)
    │   ├── CurrencyError (enum)
    │   └── 12 tests
    │
    ├── crypto.rs                 # Signatures & hashing (103 lines)
    │   ├── generate_keypair()
    │   ├── blake3_hash()
    │   └── 6 tests
    │
    ├── network.rs                # Network identifiers (16 lines)
    │   ├── NetId (type alias)
    │   ├── MAIN_NET constant
    │   └── TEST_NET constant
    │
    ├── random.rs                 # Randomness (30 lines)
    │   ├── crypto_rng()
    │   └── 1 test
    │
    └── storage.rs                # (Placeholder, commented)
        └── Future persistent storage

Total: ~2,100 lines of code + 80+ tests
```

---

## Key Data Structures

| Type | Module | Purpose |
|------|--------|---------|
| `Tx` | `tx` | Complete transaction with all fields and optional signature |
| `TxKind` | `tx` | Enum specifying transaction action (Transfer, Stake, Unstake, ClaimReward) |
| `TxId` | `tx` (alias) | BLAKE3 hash of entire transaction (32 bytes) |
| `TransactionCollector` | `tx` | Stateless validation buffer for incoming transactions |
| `Mempool<S>` | `mempool` | Generic transaction pool requiring `AccountStateView` |
| `MempoolError` | `mempool` | Enum of pool insertion failures |
| `AccountSnapshot` | `state` | Copyable account metadata (nonce, balances) |
| `State` | `state` | In-memory account snapshot store |
| `AccountStateView` | `state` | Trait for read-only account access |
| `Account` | `account` | Mutable account with nonce + wallet |
| `AccountId` | `account` (alias) | Ed25519 VerifyingKey (32 bytes) |
| `Wallet` | `wallet` | Three-pool balance tracker |
| `Crit` | `currency` | Fixed-precision amount (u128 centcrit) |
| `CurrencyError` | `currency` | Enum: Overflow \| Underflow |
| `NetId` | `network` (alias) | u8 network identifier |

---

## Safety Features

1. **Ed25519 signature verification** - All transactions must be signed
2. **Overflow/underflow protection** - All arithmetic uses checked operations
3. **Atomic nonce + balance updates** - Prevents state inconsistencies
4. **Two-layer validation** - Stateless (TransactionCollector) + Stateful (Mempool)
5. **Network-scoped transactions** - Prevents cross-chain replay attacks
6. **Duplicate transaction detection** - Via tx_id in mempool
7. **Nonce ordering** - Ensures sequential execution per account
8. **Fee enforcement** - 0.1% with 1 centcrit minimum prevents spam

---

## Future Work

Based on the current implementation, these components are planned:

1. **Persistent Storage** (`storage.rs` is currently a placeholder)
   - Database integration for accounts, transactions, blocks
   - State persistence and recovery

2. **Execution Layer**
   - Apply validated transactions to account state
   - Update balances and nonces
   - Emit execution receipts/events

3. **Consensus Mechanism**
   - Block production using staked validators
   - Reward distribution logic
   - Fork resolution

4. **Network Layer**
   - P2P transaction propagation
   - Block gossip protocol
   - Peer discovery

5. **RPC/API Layer**
   - Query account balances
   - Submit transactions
   - Monitor transaction status

---

## Testing

Run the comprehensive test suite:

```bash
cargo test
```

Test coverage includes:

- Transaction signing and verification
- Mempool insertion validation
- Account balance mutations
- Wallet three-pool operations
- Currency arithmetic edge cases
- Nonce sequencing
- Network isolation
- Fee calculation

---

## Dependencies

| Crate | Version | Purpose |
|-------|---------|---------|
| `thiserror` | 2.0.17 | Derive macros for error types |
| `ed25519-dalek` | 2.2.0 | Ed25519 signatures and verification |
| `rand_core` | 0.6 | RNG trait and OS random source |
| `borsh` | 1.5.7 | Binary serialization |
| `blake3` | 1.8.2 | Cryptographic hashing |

---

## License

This project is an educational implementation of a cryptocurrency ledger system.

---

*Documentation generated as of October 23, 2025*
