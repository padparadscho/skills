# Common Soroban Contract Patterns

This reference provides battle-tested patterns for common contract scenarios.

## Table of Contents
- [Initialization Pattern](#initialization-pattern)
- [Access Control](#access-control)
- [Upgradeable Contracts](#upgradeable-contracts)
- [Pausable Contracts](#pausable-contracts)
- [Fee Collection](#fee-collection)
- [Timelock](#timelock)
- [Rate Limiting](#rate-limiting)
- [Batch Operations](#batch-operations)
- [Oracle Integration](#oracle-integration)
- [Event Publishing](#event-publishing)

## Initialization Pattern

⚠️ **Security Critical**: Reinitialization vulnerabilities allow attackers to reset contract state.

Ensure contract can only be initialized once:

```rust
#[contracttype]
pub enum DataKey {
    Initialized,
    Admin,
}

pub fn initialize(env: Env, admin: Address) {
    // Check not already initialized
    if env.storage().instance().has(&DataKey::Initialized) {
        panic!("Already initialized");
    }
    
    // Set admin
    env.storage().instance().set(&DataKey::Admin, &admin);
    env.storage().instance().set(&DataKey::Initialized, &true);
    env.storage().instance().extend_ttl(100, 518_400);
}

fn require_initialized(env: &Env) {
    if !env.storage().instance().has(&DataKey::Initialized) {
        panic!("Not initialized");
    }
}
```

## Access Control

### Single Admin

```rust
#[contracttype]
pub enum DataKey {
    Admin,
}

pub fn set_admin(env: Env, current_admin: Address, new_admin: Address) {
    current_admin.require_auth();
    require_admin(&env, &current_admin);
    
    env.storage().instance().set(&DataKey::Admin, &new_admin);
    env.storage().instance().extend_ttl(100, 518_400);
}

fn get_admin(env: &Env) -> Address {
    env.storage().instance().get(&DataKey::Admin).unwrap()
}

fn require_admin(env: &Env, caller: &Address) {
    let admin = get_admin(env);
    assert_eq!(caller, &admin, "Unauthorized");
}
```

### Multi-Admin

```rust
#[contracttype]
pub enum DataKey {
    Admins,
}

pub fn add_admin(env: Env, caller: Address, new_admin: Address) {
    caller.require_auth();
    require_admin(&env, &caller);
    
    let mut admins: Vec<Address> = env.storage()
        .instance()
        .get(&DataKey::Admins)
        .unwrap_or(Vec::new(&env));
    
    if !admins.contains(&new_admin) {
        admins.push_back(new_admin);
        env.storage().instance().set(&DataKey::Admins, &admins);
        env.storage().instance().extend_ttl(100, 518_400);
    }
}

fn require_admin(env: &Env, caller: &Address) {
    let admins: Vec<Address> = env.storage()
        .instance()
        .get(&DataKey::Admins)
        .expect("No admins set");
    
    assert!(admins.contains(caller), "Not an admin");
}
```

### Role-Based Access Control (RBAC)

```rust
#[contracttype]
#[derive(Clone, Copy, PartialEq, Eq)]
pub enum Role {
    Admin = 0,
    Operator = 1,
    User = 2,
}

#[contracttype]
pub enum DataKey {
    UserRole(Address),
}

pub fn grant_role(env: Env, admin: Address, user: Address, role: Role) {
    admin.require_auth();
    require_role(&env, &admin, Role::Admin);
    
    let key = DataKey::UserRole(user);
    env.storage().persistent().set(&key, &role);
    env.storage().persistent().extend_ttl(&key, 100, 518_400);
}

fn get_role(env: &Env, user: &Address) -> Option<Role> {
    let key = DataKey::UserRole(user.clone());
    env.storage().persistent().get(&key)
}

fn require_role(env: &Env, user: &Address, required: Role) {
    let role = get_role(env, user).expect("No role assigned");
    assert_eq!(role, required, "Insufficient permissions");
}
```

## Upgradeable Contracts

Store WASM hash and allow admin to upgrade:

```rust
#[contracttype]
pub enum DataKey {
    Admin,
    WasmHash,
}

pub fn upgrade(env: Env, admin: Address, new_wasm_hash: BytesN<32>) {
    admin.require_auth();
    require_admin(&env, &admin);
    
    // Update WASM
    env.deployer().update_current_contract_wasm(new_wasm_hash);
    
    // Store hash for reference
    env.storage().instance().set(&DataKey::WasmHash, &new_wasm_hash);
}

pub fn get_wasm_hash(env: Env) -> BytesN<32> {
    env.storage().instance().get(&DataKey::WasmHash).unwrap()
}
```

## Pausable Contracts

Add emergency pause functionality:

```rust
#[contracttype]
pub enum DataKey {
    Admin,
    Paused,
}

pub fn pause(env: Env, admin: Address) {
    admin.require_auth();
    require_admin(&env, &admin);
    
    env.storage().instance().set(&DataKey::Paused, &true);
}

pub fn unpause(env: Env, admin: Address) {
    admin.require_auth();
    require_admin(&env, &admin);
    
    env.storage().instance().set(&DataKey::Paused, &false);
}

fn require_not_paused(env: &Env) {
    let paused: bool = env.storage()
        .instance()
        .get(&DataKey::Paused)
        .unwrap_or(false);
    
    assert!(!paused, "Contract is paused");
}

pub fn transfer(env: Env, from: Address, to: Address, amount: i128) {
    require_not_paused(&env);
    from.require_auth();
    
    // Transfer logic...
}
```

## Fee Collection

Charge fees on operations:

```rust
#[contracttype]
pub enum DataKey {
    FeeRate,      // Basis points (e.g., 30 = 0.3%)
    FeeRecipient,
    CollectedFees(Address), // Per-token collected fees
}

pub fn set_fee_rate(env: Env, admin: Address, fee_rate_bps: u32) {
    admin.require_auth();
    require_admin(&env, &admin);
    
    assert!(fee_rate_bps <= 10_000, "Fee rate too high"); // Max 100%
    env.storage().instance().set(&DataKey::FeeRate, &fee_rate_bps);
}

fn calculate_fee(amount: i128, fee_rate_bps: u32) -> i128 {
    (amount * fee_rate_bps as i128) / 10_000
}

pub fn transfer_with_fee(
    env: Env,
    from: Address,
    to: Address,
    token: Address,
    amount: i128
) {
    from.require_auth();
    
    // Get fee rate
    let fee_rate: u32 = env.storage()
        .instance()
        .get(&DataKey::FeeRate)
        .unwrap_or(0);
    
    let fee = calculate_fee(amount, fee_rate);
    let net_amount = amount - fee;
    
    // Transfer tokens
    let token_client = TokenClient::new(&env, &token);
    token_client.transfer(&from, &to, &net_amount);
    
    if fee > 0 {
        let fee_recipient: Address = env.storage()
            .instance()
            .get(&DataKey::FeeRecipient)
            .unwrap();
        
        token_client.transfer(&from, &fee_recipient, &fee);
        
        // Track collected fees
        let fee_key = DataKey::CollectedFees(token.clone());
        let total_fees = env.storage().persistent().get(&fee_key).unwrap_or(0);
        env.storage().persistent().set(&fee_key, &(total_fees + fee));
    }
}
```

## Timelock

Delay execution of critical operations:

```rust
#[contracttype]
pub struct Proposal {
    pub action: Symbol,
    pub data: Bytes,
    pub proposer: Address,
    pub unlock_time: u64,
    pub executed: bool,
}

#[contracttype]
pub enum DataKey {
    Proposal(u32),
    ProposalCount,
    TimelockDelay,
}

pub fn set_timelock_delay(env: Env, admin: Address, delay_seconds: u64) {
    admin.require_auth();
    require_admin(&env, &admin);
    
    env.storage().instance().set(&DataKey::TimelockDelay, &delay_seconds);
}

pub fn propose(env: Env, proposer: Address, action: Symbol, data: Bytes) -> u32 {
    proposer.require_auth();
    require_admin(&env, &proposer);
    
    let delay: u64 = env.storage()
        .instance()
        .get(&DataKey::TimelockDelay)
        .unwrap_or(86400); // Default 24h
    
    let proposal_id: u32 = env.storage()
        .instance()
        .get(&DataKey::ProposalCount)
        .unwrap_or(0);
    
    let proposal = Proposal {
        action,
        data,
        proposer,
        unlock_time: env.ledger().timestamp() + delay,
        executed: false,
    };
    
    let key = DataKey::Proposal(proposal_id);
    env.storage().persistent().set(&key, &proposal);
    env.storage().persistent().extend_ttl(&key, 100, 518_400);
    
    env.storage().instance().set(&DataKey::ProposalCount, &(proposal_id + 1));
    
    proposal_id
}

pub fn execute(env: Env, executor: Address, proposal_id: u32) {
    executor.require_auth();
    
    let key = DataKey::Proposal(proposal_id);
    let mut proposal: Proposal = env.storage()
        .persistent()
        .get(&key)
        .expect("Proposal not found");
    
    assert!(!proposal.executed, "Already executed");
    assert!(
        env.ledger().timestamp() >= proposal.unlock_time,
        "Timelock not expired"
    );
    
    // Mark as executed
    proposal.executed = true;
    env.storage().persistent().set(&key, &proposal);
    
    // Execute action (implementation specific)
    execute_action(&env, &proposal.action, &proposal.data);
}
```

## Rate Limiting

Limit operations per time period:

```rust
#[contracttype]
pub struct RateLimit {
    pub last_call: u64,
    pub count: u32,
}

#[contracttype]
pub enum DataKey {
    RateLimit(Address),
}

pub fn rate_limited_function(env: Env, user: Address) {
    user.require_auth();
    
    let current_time = env.ledger().timestamp();
    let window = 3600; // 1 hour window
    let max_calls = 10;
    
    let key = DataKey::RateLimit(user.clone());
    let mut rate_limit: RateLimit = env.storage()
        .temporary()
        .get(&key)
        .unwrap_or(RateLimit {
            last_call: 0,
            count: 0,
        });
    
    // Reset if outside window
    if current_time - rate_limit.last_call > window {
        rate_limit.count = 0;
    }
    
    assert!(rate_limit.count < max_calls, "Rate limit exceeded");
    
    rate_limit.count += 1;
    rate_limit.last_call = current_time;
    
    env.storage().temporary().set(&key, &rate_limit);
    
    // Function logic...
}
```

## Batch Operations

Process multiple operations efficiently:

```rust
pub fn batch_transfer(
    env: Env,
    from: Address,
    recipients: Vec<Address>,
    amounts: Vec<i128>
) {
    from.require_auth();
    
    assert_eq!(recipients.len(), amounts.len(), "Length mismatch");
    
    let from_balance = get_balance(&env, &from);
    let total: i128 = amounts.iter().map(|a| a).sum();
    
    assert!(from_balance >= total, "Insufficient balance");
    
    // Deduct total from sender
    set_balance(&env, &from, from_balance - total);
    
    // Credit each recipient
    for i in 0..recipients.len() {
        let recipient = recipients.get(i).unwrap();
        let amount = amounts.get(i).unwrap();
        
        let balance = get_balance(&env, &recipient);
        set_balance(&env, &recipient, balance + amount);
        
        // Emit event per transfer
        env.events().publish(
            (symbol_short!("transfer"), from.clone(), recipient.clone()),
            amount
        );
    }
}
```

## Oracle Integration

Query external data sources:

```rust
#[contracttype]
pub struct PriceData {
    pub price: i128,
    pub timestamp: u64,
    pub decimals: u32,
}

#[contracttype]
pub enum DataKey {
    OracleContract,
    PriceCache(Symbol),
}

pub fn set_oracle(env: Env, admin: Address, oracle_address: Address) {
    admin.require_auth();
    require_admin(&env, &admin);
    
    env.storage().instance().set(&DataKey::OracleContract, &oracle_address);
}

pub fn get_price(env: Env, asset: Symbol) -> i128 {
    // Check cache first
    let cache_key = DataKey::PriceCache(asset.clone());
    if let Some(cached): Option<PriceData> = env.storage().temporary().get(&cache_key) {
        // Use cached if fresh (< 5 minutes old)
        if env.ledger().timestamp() - cached.timestamp < 300 {
            return cached.price;
        }
    }
    
    // Query oracle
    let oracle_address: Address = env.storage()
        .instance()
        .get(&DataKey::OracleContract)
        .expect("Oracle not set");
    
    let oracle = OracleClient::new(&env, &oracle_address);
    let price_data = oracle.get_price(&asset);
    
    // Cache result
    env.storage().temporary().set(&cache_key, &price_data);
    
    price_data.price
}
```

## Event Publishing

Structured event logging for off-chain indexing:

```rust
#[contracttype]
#[derive(Clone)]
pub struct TransferEvent {
    pub from: Address,
    pub to: Address,
    pub amount: i128,
}

pub fn transfer(env: Env, from: Address, to: Address, amount: i128) {
    from.require_auth();
    
    // Transfer logic...
    
    // Publish structured event
    env.events().publish(
        (symbol_short!("transfer"), from.clone(), to.clone()),
        amount
    );
    
    // Or with custom event type
    let event = TransferEvent {
        from: from.clone(),
        to: to.clone(),
        amount,
    };
    env.events().publish((symbol_short!("transfer"),), event);
}

pub fn emit_state_change(env: Env, old_state: u32, new_state: u32) {
    env.events().publish(
        (symbol_short!("state_change"),),
        (old_state, new_state)
    );
}
```

## Best Practices Summary

1. **Initialization**: Always guard with initialization check
2. **Access Control**: Implement appropriate admin/role patterns
3. **Upgradeability**: Plan for upgrades from the start
4. **Emergency Stop**: Add pause functionality for critical contracts
5. **Fees**: Make fee rates configurable by admin
6. **Timelocks**: Use for high-impact operations
7. **Rate Limits**: Prevent abuse with temporary storage
8. **Batching**: Optimize gas for bulk operations
9. **Events**: Emit events for all state changes
10. **Validation**: Always validate inputs and state before operations
