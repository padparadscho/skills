# Soroban Security Best Practices

This reference covers common security vulnerabilities and best practices for Soroban smart contracts.

## Table of Contents
- [Authorization & Access Control](#authorization--access-control)
- [Input Validation](#input-validation)
- [Initialization Security](#initialization-security)
- [Arithmetic Safety](#arithmetic-safety)
- [Storage Safety](#storage-safety)
- [Timestamp Dependence](#timestamp-dependence)
- [Front-Running Prevention](#front-running-prevention)
- [Token Integration Security](#token-integration-security)
- [Upgrade Security](#upgrade-security)
- [Testing & Auditing](#testing--auditing)

## Authorization & Access Control

### ❌ AVOID: Missing Authorization Checks

```rust
// BAD: No authorization check
pub fn set_admin(env: Env, new_admin: Address) {
    env.storage().instance().set(&DataKey::Admin, &new_admin);
}

// BAD: Authorization after state change
pub fn withdraw(env: Env, user: Address, amount: i128) {
    transfer_tokens(&env, &user, &amount);
    user.require_auth(); // TOO LATE!
}
```

### ✅ BEST PRACTICE: Always Authorize First

```rust
// GOOD: Authorize before any state changes
pub fn set_admin(env: Env, caller: Address, new_admin: Address) {
    caller.require_auth();
    let current_admin = get_admin(&env);
    assert_eq!(caller, current_admin, "Unauthorized");
    
    env.storage().instance().set(&DataKey::Admin, &new_admin);
}

// GOOD: Authorization and validation before operations
pub fn withdraw(env: Env, user: Address, amount: i128) {
    user.require_auth();
    
    let balance = get_balance(&env, &user);
    assert!(balance >= amount, "Insufficient balance");
    
    transfer_tokens(&env, &user, &amount);
}
```

### ❌ AVOID: Incorrect Authorization Target

```rust
// BAD: Authorizing recipient instead of sender
pub fn transfer(env: Env, from: Address, to: Address, amount: i128) {
    to.require_auth(); // WRONG! Anyone can receive
    // ...
}

// BAD: Authorizing wrong party in delegated operations
pub fn transfer_from(env: Env, from: Address, to: Address, amount: i128) {
    from.require_auth(); // WRONG! Should be spender
    // ...
}
```

### ✅ BEST PRACTICE: Authorize the Correct Address

```rust
// GOOD: Authorize the sender
pub fn transfer(env: Env, from: Address, to: Address, amount: i128) {
    from.require_auth();
    // ...
}

// GOOD: Authorize the spender in delegated transfers
pub fn transfer_from(
    env: Env,
    spender: Address,
    from: Address,
    to: Address,
    amount: i128
) {
    spender.require_auth();
    check_allowance(&env, &from, &spender, amount);
    // ...
}
```

## Input Validation

### ❌ AVOID: Missing Input Validation

```rust
// BAD: No validation on amounts
pub fn deposit(env: Env, user: Address, amount: i128) {
    user.require_auth();
    let balance = get_balance(&env, &user);
    set_balance(&env, &user, balance + amount); // Could overflow!
}

// BAD: No validation on addresses
pub fn set_config(env: Env, admin: Address, config_address: Address) {
    admin.require_auth();
    env.storage().instance().set(&DataKey::Config, &config_address);
    // What if config_address is invalid/zero?
}
```

### ✅ BEST PRACTICE: Validate All Inputs

```rust
// GOOD: Validate amounts
pub fn deposit(env: Env, user: Address, amount: i128) {
    user.require_auth();
    assert!(amount > 0, "Invalid amount");
    assert!(amount <= i128::MAX / 2, "Amount too large"); // Prevent overflow
    
    let balance = get_balance(&env, &user);
    set_balance(&env, &user, balance + amount);
}

// GOOD: Validate addresses and parameters
pub fn set_config(env: Env, admin: Address, config_address: Address) {
    admin.require_auth();
    require_admin(&env, &admin);
    
    // Validate config address exists/is valid
    assert!(is_valid_address(&env, &config_address), "Invalid address");
    
    env.storage().instance().set(&DataKey::Config, &config_address);
}

// GOOD: Validate array lengths match
pub fn batch_transfer(
    env: Env,
    from: Address,
    recipients: Vec<Address>,
    amounts: Vec<i128>
) {
    from.require_auth();
    assert_eq!(recipients.len(), amounts.len(), "Length mismatch");
    assert!(recipients.len() > 0, "Empty batch");
    assert!(recipients.len() <= 100, "Batch too large");
    
    // ...
}
```

### ❌ AVOID: Unbounded Loops or Operations

```rust
// BAD: Unbounded loop
pub fn process_all(env: Env) {
    let users: Vec<Address> = env.storage().persistent().get(&DataKey::Users).unwrap();
    for user in users.iter() { // Could run out of gas!
        process_user(&env, &user.unwrap());
    }
}
```

### ✅ BEST PRACTICE: Use Pagination or Limits

```rust
// GOOD: Paginated processing
pub fn process_batch(env: Env, start: u32, count: u32) {
    assert!(count <= 50, "Batch too large");
    
    let users: Vec<Address> = env.storage().persistent().get(&DataKey::Users).unwrap();
    let end = (start + count).min(users.len());
    
    for i in start..end {
        let user = users.get(i).unwrap();
        process_user(&env, &user);
    }
}
```

## Initialization Security

### ❌ AVOID: Reinitialization Vulnerabilities

```rust
// BAD: Can be called multiple times
pub fn initialize(env: Env, admin: Address) {
    env.storage().instance().set(&DataKey::Admin, &admin);
}

// BAD: Partial initialization check
pub fn initialize(env: Env, admin: Address, token: Address) {
    if env.storage().instance().has(&DataKey::Admin) {
        panic!("Already initialized");
    }
    env.storage().instance().set(&DataKey::Admin, &admin);
    env.storage().instance().set(&DataKey::Token, &token);
    // What if this is called again with only token check?
}
```

### ✅ BEST PRACTICE: Proper Initialization Guards

```rust
// GOOD: Single initialization flag
#[contracttype]
pub enum DataKey {
    Initialized,
    Admin,
    Token,
}

pub fn initialize(env: Env, admin: Address, token: Address) {
    if env.storage().instance().has(&DataKey::Initialized) {
        panic!("Already initialized");
    }
    
    // Validate all initialization parameters
    assert!(is_valid_address(&env, &admin), "Invalid admin");
    assert!(is_valid_address(&env, &token), "Invalid token");
    
    env.storage().instance().set(&DataKey::Admin, &admin);
    env.storage().instance().set(&DataKey::Token, &token);
    env.storage().instance().set(&DataKey::Initialized, &true);
    env.storage().instance().extend_ttl(100, 518_400);
}

fn require_initialized(env: &Env) {
    if !env.storage().instance().has(&DataKey::Initialized) {
        panic!("Not initialized");
    }
}

// Call at start of all public functions
pub fn some_function(env: Env, param: u32) {
    require_initialized(&env);
    // ...
}
```

### ❌ AVOID: Missing Validation at Initialization

```rust
// BAD: No validation of fee parameters
pub fn initialize(env: Env, admin: Address, fee_rate: u32) {
    env.storage().instance().set(&DataKey::Admin, &admin);
    env.storage().instance().set(&DataKey::FeeRate, &fee_rate);
    // What if fee_rate is 100,000 (100%+)?
}
```

### ✅ BEST PRACTICE: Validate Configuration at Init

```rust
// GOOD: Validate all configuration parameters
pub fn initialize(env: Env, admin: Address, fee_rate: u32) {
    if env.storage().instance().has(&DataKey::Initialized) {
        panic!("Already initialized");
    }
    
    assert!(fee_rate <= 10_000, "Fee rate must be <= 100% (10000 bps)");
    assert!(fee_rate >= 0, "Fee rate must be non-negative");
    
    env.storage().instance().set(&DataKey::Admin, &admin);
    env.storage().instance().set(&DataKey::FeeRate, &fee_rate);
    env.storage().instance().set(&DataKey::Initialized, &true);
}
```

## Arithmetic Safety

### ❌ AVOID: Unchecked Arithmetic

```rust
// BAD: Potential overflow
pub fn add_balance(env: Env, user: Address, amount: i128) {
    let balance = get_balance(&env, &user);
    set_balance(&env, &user, balance + amount); // Can overflow!
}

// BAD: Potential underflow
pub fn subtract_balance(env: Env, user: Address, amount: i128) {
    let balance = get_balance(&env, &user);
    set_balance(&env, &user, balance - amount); // Can underflow!
}
```

### ✅ BEST PRACTICE: Checked Arithmetic

```rust
// GOOD: Check before arithmetic operations
pub fn add_balance(env: Env, user: Address, amount: i128) {
    assert!(amount >= 0, "Amount must be non-negative");
    
    let balance = get_balance(&env, &user);
    let new_balance = balance.checked_add(amount)
        .expect("Balance overflow");
    
    set_balance(&env, &user, new_balance);
}

// GOOD: Validate before subtraction
pub fn subtract_balance(env: Env, user: Address, amount: i128) {
    assert!(amount >= 0, "Amount must be non-negative");
    
    let balance = get_balance(&env, &user);
    assert!(balance >= amount, "Insufficient balance");
    
    set_balance(&env, &user, balance - amount);
}
```

### ✅ BEST PRACTICE: Safe Division and Rounding

```rust
// GOOD: Check for division by zero and rounding
pub fn calculate_fee(amount: i128, fee_rate_bps: u32) -> i128 {
    assert!(fee_rate_bps <= 10_000, "Invalid fee rate");
    
    // Safe calculation: (amount * fee_rate) / 10_000
    let numerator = amount.checked_mul(fee_rate_bps as i128)
        .expect("Fee calculation overflow");
    
    numerator / 10_000 // Integer division rounds down
}

// GOOD: Be explicit about rounding direction
pub fn calculate_fee_rounded_up(amount: i128, fee_rate_bps: u32) -> i128 {
    let numerator = amount.checked_mul(fee_rate_bps as i128)
        .expect("Fee calculation overflow");
    
    // Round up: (numerator + divisor - 1) / divisor
    (numerator + 9_999) / 10_000
}
```

## Storage Safety

### ❌ AVOID: Storage Without TTL Management

```rust
// BAD: No TTL extension
pub fn set_balance(env: Env, user: Address, amount: i128) {
    env.storage().persistent().set(&(DataKey::Balance, user), &amount);
    // Data will expire!
}
```

### ✅ BEST PRACTICE: Always Extend TTL

```rust
// GOOD: Extend TTL on all persistent storage operations
pub fn set_balance(env: Env, user: Address, amount: i128) {
    let key = (DataKey::Balance, user);
    env.storage().persistent().set(&key, &amount);
    env.storage().persistent().extend_ttl(&key, 100, 518_400);
}
```

### ❌ AVOID: Wrong Storage Type

```rust
// BAD: User balances in temporary storage
pub fn set_balance(env: Env, user: Address, amount: i128) {
    env.storage().temporary().set(&(DataKey::Balance, user), &amount);
    // Will expire in ~5-10 ledgers!
}

// BAD: Frequently changing data in persistent storage
pub fn cache_query(env: Env, query_hash: BytesN<32>, result: Vec<u8>) {
    env.storage().persistent().set(&query_hash, &result);
    // Expensive for temporary cache!
}
```

### ✅ BEST PRACTICE: Choose Appropriate Storage

```rust
// GOOD: Critical data in Persistent
pub fn set_balance(env: Env, user: Address, amount: i128) {
    let key = (DataKey::Balance, user);
    env.storage().persistent().set(&key, &amount);
    env.storage().persistent().extend_ttl(&key, 100, 518_400);
}

// GOOD: Temporary data in Temporary storage
pub fn cache_query(env: Env, query_hash: BytesN<32>, result: Vec<u8>) {
    env.storage().temporary().set(&query_hash, &result);
    env.storage().temporary().extend_ttl(&query_hash, 0, 100);
}

// GOOD: Contract config in Instance storage
pub fn set_fee_rate(env: Env, admin: Address, fee_rate: u32) {
    admin.require_auth();
    require_admin(&env, &admin);
    
    env.storage().instance().set(&DataKey::FeeRate, &fee_rate);
    env.storage().instance().extend_ttl(100, 518_400);
}
```

## Timestamp Dependence

### ❌ AVOID: Relying on Exact Timestamps

```rust
// BAD: Exact timestamp comparison
pub fn claim_reward(env: Env, user: Address) {
    let unlock_time: u64 = get_unlock_time(&env, &user);
    if env.ledger().timestamp() == unlock_time { // Too precise!
        // User might miss their exact second!
    }
}
```

### ✅ BEST PRACTICE: Use Greater/Less Than Comparisons

```rust
// GOOD: Range-based time checks
pub fn claim_reward(env: Env, user: Address) {
    user.require_auth();
    
    let unlock_time: u64 = get_unlock_time(&env, &user);
    assert!(
        env.ledger().timestamp() >= unlock_time,
        "Still locked"
    );
    
    // Process reward
}
```

### ❌ AVOID: Assuming Block Time Precision

```rust
// BAD: Assuming blocks are exactly 5 seconds apart
pub fn time_based_action(env: Env) {
    let last_action = get_last_action_time(&env);
    let current = env.ledger().timestamp();
    let blocks_passed = (current - last_action) / 5; // Imprecise!
}
```

### ✅ BEST PRACTICE: Use Ledger Sequence or Time Windows

```rust
// GOOD: Use ledger sequence for block-based logic
pub fn check_cooldown(env: Env, user: Address) -> bool {
    let last_call_ledger: u32 = env.storage()
        .temporary()
        .get(&(DataKey::LastCall, user.clone()))
        .unwrap_or(0);
    
    let current_ledger = env.ledger().sequence();
    current_ledger - last_call_ledger >= 100 // 100 ledgers cooldown
}

// GOOD: Use time windows for time-based logic
pub fn is_time_window_passed(env: Env, start_time: u64, window_seconds: u64) -> bool {
    env.ledger().timestamp() >= start_time + window_seconds
}
```

## Front-Running Prevention

### ❌ AVOID: Predictable State Changes

```rust
// BAD: No slippage protection
pub fn swap(env: Env, user: Address, token_in: Address, amount_in: i128) {
    user.require_auth();
    
    let amount_out = calculate_output(&env, &token_in, amount_in);
    // Attacker can front-run and change the exchange rate!
    
    execute_swap(&env, &user, &token_in, amount_in, amount_out);
}
```

### ✅ BEST PRACTICE: Use Slippage Protection

```rust
// GOOD: Require minimum output amount
pub fn swap(
    env: Env,
    user: Address,
    token_in: Address,
    amount_in: i128,
    min_amount_out: i128 // Slippage protection
) {
    user.require_auth();
    
    let amount_out = calculate_output(&env, &token_in, amount_in);
    assert!(amount_out >= min_amount_out, "Slippage too high");
    
    execute_swap(&env, &user, &token_in, amount_in, amount_out);
}
```

### ✅ BEST PRACTICE: Commit-Reveal for Sensitive Operations

```rust
// Phase 1: Commit
pub fn commit_action(env: Env, user: Address, commitment: BytesN<32>) {
    user.require_auth();
    
    let commit_time = env.ledger().timestamp();
    env.storage().temporary().set(
        &(DataKey::Commitment, user.clone()),
        &(commitment, commit_time)
    );
}

// Phase 2: Reveal (after delay)
pub fn reveal_action(env: Env, user: Address, secret: BytesN<32>) {
    user.require_auth();
    
    let (commitment, commit_time): (BytesN<32>, u64) = env.storage()
        .temporary()
        .get(&(DataKey::Commitment, user.clone()))
        .expect("No commitment found");
    
    // Require time delay
    assert!(
        env.ledger().timestamp() >= commit_time + 60,
        "Must wait before reveal"
    );
    
    // Verify commitment
    let hash = env.crypto().sha256(&secret);
    assert_eq!(hash, commitment, "Invalid reveal");
    
    // Execute action
}
```

## Token Integration Security

### ❌ AVOID: Unchecked Token Operations

```rust
// BAD: No balance check before transfer
pub fn withdraw(env: Env, user: Address, token: Address, amount: i128) {
    user.require_auth();
    
    let token_client = TokenClient::new(&env, &token);
    token_client.transfer(&env.current_contract_address(), &user, &amount);
    // What if contract doesn't have enough tokens?
}
```

### ✅ BEST PRACTICE: Verify Token Balances

```rust
// GOOD: Check balance before transfer
pub fn withdraw(env: Env, user: Address, token: Address, amount: i128) {
    user.require_auth();
    
    let token_client = TokenClient::new(&env, &token);
    let contract_balance = token_client.balance(&env.current_contract_address());
    
    assert!(contract_balance >= amount, "Insufficient contract balance");
    
    token_client.transfer(&env.current_contract_address(), &user, &amount);
}
```

### ❌ AVOID: Trusting External Token Contracts

```rust
// BAD: No validation of token address
pub fn add_liquidity(env: Env, user: Address, token: Address, amount: i128) {
    user.require_auth();
    
    let token_client = TokenClient::new(&env, &token);
    token_client.transfer(&user, &env.current_contract_address(), &amount);
    // What if token is malicious?
}
```

### ✅ BEST PRACTICE: Whitelist Tokens

```rust
// GOOD: Only allow whitelisted tokens
pub fn add_liquidity(env: Env, user: Address, token: Address, amount: i128) {
    user.require_auth();
    
    // Check token is whitelisted
    assert!(
        is_token_whitelisted(&env, &token),
        "Token not supported"
    );
    
    let token_client = TokenClient::new(&env, &token);
    token_client.transfer(&user, &env.current_contract_address(), &amount);
}

fn is_token_whitelisted(env: &Env, token: &Address) -> bool {
    env.storage()
        .persistent()
        .get(&(DataKey::WhitelistedToken, token.clone()))
        .unwrap_or(false)
}
```

## Upgrade Security

### ❌ AVOID: Unrestricted Upgrades

```rust
// BAD: Anyone can upgrade
pub fn upgrade(env: Env, new_wasm_hash: BytesN<32>) {
    env.deployer().update_current_contract_wasm(new_wasm_hash);
}
```

### ✅ BEST PRACTICE: Restricted Upgrade with Timelock

```rust
// GOOD: Admin-only upgrade with optional timelock
pub fn propose_upgrade(env: Env, admin: Address, new_wasm_hash: BytesN<32>) {
    admin.require_auth();
    require_admin(&env, &admin);
    
    let unlock_time = env.ledger().timestamp() + 86400; // 24h timelock
    
    env.storage().instance().set(
        &DataKey::PendingUpgrade,
        &(new_wasm_hash, unlock_time)
    );
}

pub fn execute_upgrade(env: Env, admin: Address) {
    admin.require_auth();
    require_admin(&env, &admin);
    
    let (new_wasm_hash, unlock_time): (BytesN<32>, u64) = env.storage()
        .instance()
        .get(&DataKey::PendingUpgrade)
        .expect("No pending upgrade");
    
    assert!(
        env.ledger().timestamp() >= unlock_time,
        "Timelock not expired"
    );
    
    env.deployer().update_current_contract_wasm(new_wasm_hash);
    
    // Clear pending upgrade
    env.storage().instance().remove(&DataKey::PendingUpgrade);
}
```

## Testing & Auditing

### ✅ BEST PRACTICE: Comprehensive Test Coverage

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // Test authorization
    #[test]
    #[should_panic(expected = "Unauthorized")]
    fn test_unauthorized_access() {
        let env = Env::default();
        let contract_id = env.register(MyContract, ());
        let client = MyContractClient::new(&env, &contract_id);
        
        let attacker = Address::generate(&env);
        env.mock_all_auths();
        
        // Should panic
        client.admin_only_function(&attacker);
    }
    
    // Test overflow prevention
    #[test]
    #[should_panic(expected = "overflow")]
    fn test_overflow_protection() {
        let env = Env::default();
        let contract_id = env.register(MyContract, ());
        let client = MyContractClient::new(&env, &contract_id);
        
        env.mock_all_auths();
        
        // Should panic on overflow
        client.add_balance(&user, &i128::MAX);
        client.add_balance(&user, &1);
    }
    
    // Test initialization
    #[test]
    #[should_panic(expected = "Already initialized")]
    fn test_double_initialization() {
        let env = Env::default();
        let contract_id = env.register(MyContract, ());
        let client = MyContractClient::new(&env, &contract_id);
        
        env.mock_all_auths();
        
        client.initialize(&admin);
        client.initialize(&admin); // Should panic
    }
}
```

### ✅ BEST PRACTICE: Security Checklist

Before deploying:

1. **Authorization**: All state-changing functions have `require_auth()`
2. **Input Validation**: All parameters are validated
3. **Arithmetic**: All math operations use checked arithmetic
4. **Initialization**: Contract can only be initialized once
5. **Storage**: Correct storage types with TTL management
6. **Access Control**: Admin/privileged functions properly protected
7. **Token Operations**: All token transfers validated
8. **Upgrades**: Upgrade mechanism is secure and restricted
9. **Test Coverage**: All critical paths tested
10. **Documentation**: Security assumptions documented

## Common Vulnerability Patterns

| Vulnerability | Risk | Prevention |
|--------------|------|------------|
| Missing authorization | Critical | Always call `require_auth()` first |
| Integer overflow/underflow | High | Use checked arithmetic |
| Reinitialization | High | Use initialization flag |
| Missing input validation | High | Validate all inputs |
| Wrong storage type | Medium | Choose appropriate storage |
| Missing TTL extension | Medium | Extend TTL on all writes |
| Timestamp manipulation | Low-Medium | Use >= comparisons, ledger sequence |
| Front-running | Medium | Use slippage protection, commit-reveal |
| Unprotected upgrade | Critical | Restrict to admin with timelock |

## Resources

- Official Soroban Security: https://sorobansecurity.com/
- Stellar Smart Contract Docs: https://developers.stellar.org/docs/build/smart-contracts
- Soroban SDK: https://docs.rs/soroban-sdk/latest/soroban_sdk/
