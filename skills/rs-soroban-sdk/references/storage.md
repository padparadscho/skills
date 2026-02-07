# Storage Patterns

Soroban provides three storage types with different characteristics:

## Storage Types Overview

| Type | Lifetime | Cost | Use Case |
|------|----------|------|----------|
| **Persistent** | Long (user-managed TTL) | Higher | User balances, critical state |
| **Temporary** | Short (5-10 ledgers) | Lower | Caching, temporary locks |
| **Instance** | Tied to contract | Medium | Contract config, metadata |

## Persistent Storage

Use for data that must survive long-term. Requires explicit TTL management.

```rust
const BALANCE_KEY: Symbol = symbol_short!("BALANCE");

pub fn set_balance(env: Env, user: Address, amount: i128) {
    let key = (BALANCE_KEY, user);
    env.storage().persistent().set(&key, &amount);
    
    // Extend TTL for 30 days worth of ledgers (~518,400 ledgers at 5s/ledger)
    env.storage().persistent().extend_ttl(&key, 100, 518_400);
}

pub fn get_balance(env: Env, user: Address) -> i128 {
    let key = (BALANCE_KEY, user);
    env.storage().persistent().get(&key).unwrap_or(0)
}
```

**TTL Management:**
- `extend_ttl(key, threshold, extend_to)` - Extend if TTL < threshold
- `bump_ttl(key, extend_to)` - Always extend TTL
- Call on reads and writes to keep data alive

## Temporary Storage

Use for short-lived data that doesn't need persistence.

```rust
const CACHE_KEY: Symbol = symbol_short!("CACHE");

pub fn cache_result(env: Env, id: u32, value: Vec<u8>) {
    let key = (CACHE_KEY, id);
    env.storage().temporary().set(&key, &value);
    
    // Optional: extend for 100 ledgers (~8 minutes)
    env.storage().temporary().extend_ttl(&key, 0, 100);
}

pub fn get_cached(env: Env, id: u32) -> Option<Vec<u8>> {
    let key = (CACHE_KEY, id);
    env.storage().temporary().get(&key)
}
```

**Characteristics:**
- Auto-expires after 5-10 ledgers by default
- 10x cheaper than Persistent
- No persistent guarantees

## Instance Storage

Use for contract-wide configuration and metadata. Lifetime tied to contract.

```rust
const ADMIN_KEY: Symbol = symbol_short!("ADMIN");
const INITIALIZED: Symbol = symbol_short!("INIT");

pub fn initialize(env: Env, admin: Address) {
    if env.storage().instance().has(&INITIALIZED) {
        panic!("Already initialized");
    }
    
    env.storage().instance().set(&ADMIN_KEY, &admin);
    env.storage().instance().set(&INITIALIZED, &true);
    
    // Extend instance storage TTL
    env.storage().instance().extend_ttl(100, 518_400);
}

pub fn get_admin(env: Env) -> Address {
    env.storage().instance().get(&ADMIN_KEY).unwrap()
}
```

**Characteristics:**
- Shared across all contract invocations
- Cannot be deleted while contract exists
- Good for admin addresses, config, flags

## Storage Best Practices

### ⚠️ SECURITY: Storage Type Selection

Choosing the wrong storage type can lead to data loss or excessive costs.

### 1. Choose the Right Storage Type

```rust
// ✓ Good: User balances in Persistent
env.storage().persistent().set(&(symbol_short!("BAL"), user), &balance);

// ✗ Bad: User balances in Temporary (will expire!)
env.storage().temporary().set(&(symbol_short!("BAL"), user), &balance);

// ✓ Good: Contract admin in Instance
env.storage().instance().set(&symbol_short!("ADMIN"), &admin);

// ✓ Good: Query cache in Temporary
env.storage().temporary().set(&(symbol_short!("CACHE"), query_hash), &result);
```

### 2. ⚠️ CRITICAL: Always Manage TTL for Persistent Storage

**Security Risk**: Missing TTL extension causes data to expire and be lost permanently.

```rust
pub fn update_balance(env: Env, user: Address, amount: i128) {
    let key = (symbol_short!("BALANCE"), user);
    env.storage().persistent().set(&key, &amount);
    
    // ✅ CRITICAL: Always extend TTL after writes
    env.storage().persistent().extend_ttl(&key, 100, 518_400);
}
```

### 3. Use Composite Keys

```rust
// Good: Hierarchical keys
let user_balance_key = (symbol_short!("BALANCE"), user_address);
let user_allowance_key = (symbol_short!("ALLOW"), owner, spender);

// Works but less organized
let user_balance_key = Symbol::new(&env, "balance_user1");
```

### 4. Check Existence Before Get

```rust
// ✓ Good: Handle missing keys
pub fn get_balance(env: Env, user: Address) -> i128 {
    let key = (symbol_short!("BALANCE"), user);
    env.storage().persistent().get(&key).unwrap_or(0)
}

// ✗ Bad: Will panic if key doesn't exist
pub fn get_balance(env: Env, user: Address) -> i128 {
    let key = (symbol_short!("BALANCE"), user);
    env.storage().persistent().get(&key).unwrap()
}
```

### 5. Use Custom Types with #[contracttype]

```rust
#[contracttype]
#[derive(Clone, Debug, Eq, PartialEq)]
pub struct UserData {
    pub balance: i128,
    pub last_updated: u64,
    pub is_active: bool,
}

pub fn save_user(env: Env, user: Address, data: UserData) {
    let key = (symbol_short!("USER"), user);
    env.storage().persistent().set(&key, &data);
}
```

## Storage Patterns by Use Case

### User Balances
```rust
const BALANCE: Symbol = symbol_short!("BALANCE");

pub fn set_balance(env: Env, user: Address, amount: i128) {
    env.storage().persistent().set(&(BALANCE, user), &amount);
    env.storage().persistent().extend_ttl(&(BALANCE, user), 100, 518_400);
}
```

### Allowances (ERC20-style)
```rust
const ALLOWANCE: Symbol = symbol_short!("ALLOW");

pub fn set_allowance(env: Env, owner: Address, spender: Address, amount: i128) {
    let key = (ALLOWANCE, owner, spender);
    env.storage().persistent().set(&key, &amount);
    env.storage().persistent().extend_ttl(&key, 100, 518_400);
}
```

### Rate Limiting
```rust
pub fn check_rate_limit(env: Env, user: Address) -> bool {
    let key = (symbol_short!("RATE"), user);
    let current_ledger = env.ledger().sequence();
    
    if let Some(last_call) = env.storage().temporary().get(&key) {
        if current_ledger - last_call < 100 { // 100 ledgers cooldown
            return false;
        }
    }
    
    env.storage().temporary().set(&key, &current_ledger);
    true
}
```

### Configuration
```rust
#[contracttype]
pub struct Config {
    pub admin: Address,
    pub fee_rate: u32,
    pub paused: bool,
}

pub fn set_config(env: Env, config: Config) {
    env.storage().instance().set(&symbol_short!("CONFIG"), &config);
    env.storage().instance().extend_ttl(100, 518_400);
}
```

## Storage Cost Optimization

1. **Use Temporary for ephemeral data**: 10x cheaper than Persistent
2. **Batch TTL extensions**: Extend TTL for multiple keys in one call
3. **Remove old data**: Delete unused keys to reclaim rent
4. **Use smaller keys**: Symbol constants are cheaper than String keys
5. **Compress data**: Store hashes instead of full data when possible

```rust
// ✓ Efficient: Use Symbol constants
const KEY: Symbol = symbol_short!("DATA");

// ✗ Less efficient: Create new Symbols each time
let key = Symbol::new(&env, "data");
```
