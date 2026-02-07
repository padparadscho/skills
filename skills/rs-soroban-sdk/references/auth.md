# Authorization Patterns

Soroban provides built-in authorization mechanisms for secure contract interactions.

## Basic Authorization

The simplest pattern uses `Address::require_auth()`:

```rust
pub fn transfer(env: Env, from: Address, to: Address, amount: i128) {
    // Requires from to have signed this invocation
    from.require_auth();
    
    // Now safe to proceed
    let balance = get_balance(&env, &from);
    assert!(balance >= amount);
    set_balance(&env, &from, balance - amount);
    set_balance(&env, &to, get_balance(&env, &to) + amount);
}
```

**How it works:**
- Checks that `from` has authorized this specific invocation
- Verifies signature or sub-authorization
- Panics if authorization is missing

## Authorization Context

The auth context includes:
1. Contract address being called
2. Function name
3. Function arguments

```rust
// When user calls:
client.transfer(&from, &to, &100);

// Auth context checked:
// - Contract: <this contract address>
// - Function: "transfer"
// - Args: [from, to, 100]
```

## Multi-signature Authorization

Require multiple addresses to authorize:

```rust
pub fn multi_sig_transfer(
    env: Env,
    signer1: Address,
    signer2: Address,
    to: Address,
    amount: i128
) {
    signer1.require_auth();
    signer2.require_auth();
    
    // Both have authorized, proceed
    // ...
}
```

## Custom Authorization Logic

For complex auth patterns, use the `auth` module:

```rust
use soroban_sdk::auth::{Context, CustomAccountInterface};

#[contracttype]
pub enum DataKey {
    Admins,
}

pub fn admin_only_function(env: Env, admin: Address) {
    admin.require_auth();
    
    // Verify admin is in admin list
    let admins: Vec<Address> = env.storage()
        .instance()
        .get(&DataKey::Admins)
        .unwrap();
    
    assert!(admins.contains(&admin), "Not an admin");
    
    // Proceed with admin operation
}
```

## Delegated Authorization

Allow contracts to act on behalf of users:

```rust
pub fn authorized_transfer(env: Env, from: Address, to: Address, amount: i128) {
    // from must have authorized this contract to act on their behalf
    from.require_auth();
    
    // Contract can now call other contracts on behalf of from
    let token = TokenClient::new(&env, &token_address);
    token.transfer(&from, &to, &amount); // from auth carries over
}
```

## Authorization Best Practices

### ⚠️ SECURITY NOTE

Authorization vulnerabilities are the #1 source of smart contract exploits. Always follow these patterns.

### 1. ✅ CRITICAL: Always Authorize Before State Changes

```rust
// ✓ Good: Authorize before changing state
pub fn withdraw(env: Env, user: Address, amount: i128) {
    user.require_auth();
    let balance = get_balance(&env, &user);
    assert!(balance >= amount);
    set_balance(&env, &user, balance - amount);
}

// ✗ Bad: Check after state change
pub fn withdraw(env: Env, user: Address, amount: i128) {
    let balance = get_balance(&env, &user);
    set_balance(&env, &user, balance - amount);
    user.require_auth(); // Too late!
}
```

### 2. Authorize the Right Address

```rust
// ✓ Good: Authorize the address transferring funds
pub fn transfer(env: Env, from: Address, to: Address, amount: i128) {
    from.require_auth();
    // ...
}

// ✗ Bad: Authorizing recipient
pub fn transfer(env: Env, from: Address, to: Address, amount: i128) {
    to.require_auth(); // Wrong! Anyone can receive
    // ...
}
```

### 3. Use Authorization for All Sensitive Operations

```rust
pub fn set_admin(env: Env, new_admin: Address) {
    let current_admin = get_admin(&env);
    current_admin.require_auth(); // Only current admin can change admin
    
    env.storage().instance().set(&symbol_short!("ADMIN"), &new_admin);
}
```

### 4. Don't Mix Auth with Business Logic

```rust
// ✓ Good: Clear separation
pub fn transfer(env: Env, from: Address, to: Address, amount: i128) {
    from.require_auth();
    
    // Validation
    assert!(amount > 0, "Invalid amount");
    let balance = get_balance(&env, &from);
    assert!(balance >= amount, "Insufficient balance");
    
    // State changes
    set_balance(&env, &from, balance - amount);
    set_balance(&env, &to, get_balance(&env, &to) + amount);
}
```

## Common Patterns

### Admin-Only Functions

```rust
const ADMIN_KEY: Symbol = symbol_short!("ADMIN");

fn get_admin(env: &Env) -> Address {
    env.storage().instance().get(&ADMIN_KEY).unwrap()
}

fn require_admin(env: &Env, caller: &Address) {
    caller.require_auth();
    assert_eq!(caller, &get_admin(env), "Unauthorized");
}

pub fn pause_contract(env: Env, admin: Address) {
    require_admin(&env, &admin);
    env.storage().instance().set(&symbol_short!("PAUSED"), &true);
}
```

### Allowance Pattern (ERC20-style)

```rust
pub fn approve(env: Env, owner: Address, spender: Address, amount: i128) {
    owner.require_auth();
    
    let key = (symbol_short!("ALLOW"), owner.clone(), spender.clone());
    env.storage().persistent().set(&key, &amount);
    env.storage().persistent().extend_ttl(&key, 100, 518_400);
}

pub fn transfer_from(
    env: Env,
    spender: Address,
    from: Address,
    to: Address,
    amount: i128
) {
    spender.require_auth();
    
    // Check allowance
    let allowance_key = (symbol_short!("ALLOW"), from.clone(), spender.clone());
    let allowance: i128 = env.storage()
        .persistent()
        .get(&allowance_key)
        .unwrap_or(0);
    
    assert!(allowance >= amount, "Insufficient allowance");
    
    // Deduct allowance
    env.storage().persistent().set(&allowance_key, &(allowance - amount));
    
    // Transfer
    let from_balance = get_balance(&env, &from);
    assert!(from_balance >= amount, "Insufficient balance");
    set_balance(&env, &from, from_balance - amount);
    set_balance(&env, &to, get_balance(&env, &to) + amount);
}
```

### Time-locked Operations

```rust
pub fn propose_change(env: Env, proposer: Address, new_value: u32) {
    proposer.require_auth();
    
    let proposal = (new_value, env.ledger().timestamp() + 86400); // 24h timelock
    env.storage().instance().set(&symbol_short!("PROPOSAL"), &proposal);
}

pub fn execute_change(env: Env, executor: Address) {
    executor.require_auth();
    
    let (new_value, unlock_time): (u32, u64) = env.storage()
        .instance()
        .get(&symbol_short!("PROPOSAL"))
        .expect("No proposal");
    
    assert!(
        env.ledger().timestamp() >= unlock_time,
        "Timelock not expired"
    );
    
    env.storage().instance().set(&symbol_short!("VALUE"), &new_value);
}
```

### Role-Based Access Control

```rust
#[contracttype]
#[derive(Clone)]
pub enum Role {
    Admin,
    Operator,
    User,
}

#[contracttype]
pub enum DataKey {
    UserRole(Address),
    AdminCount,
}

pub fn grant_role(env: Env, admin: Address, user: Address, role: Role) {
    require_role(&env, &admin, Role::Admin);
    
    let key = DataKey::UserRole(user);
    env.storage().persistent().set(&key, &role);
}

fn require_role(env: &Env, user: &Address, required_role: Role) {
    user.require_auth();
    
    let key = DataKey::UserRole(user.clone());
    let user_role: Role = env.storage()
        .persistent()
        .get(&key)
        .expect("No role assigned");
    
    assert_eq!(user_role, required_role, "Insufficient permissions");
}

pub fn admin_function(env: Env, caller: Address) {
    require_role(&env, &caller, Role::Admin);
    // Admin-only logic
}
```

## Testing Authorization

```rust
#[test]
fn test_authorization() {
    let env = Env::default();
    let contract_id = env.register(MyContract, ());
    let client = MyContractClient::new(&env, &contract_id);
    
    let user = Address::generate(&env);
    
    // Mock authorization for testing
    env.mock_all_auths();
    
    client.transfer(&user, &other, &100);
    
    // Verify auth was required
    assert_eq!(
        env.auths(),
        std::vec![(
            user.clone(),
            AuthorizedInvocation {
                function: AuthorizedFunction::Contract((
                    contract_id.clone(),
                    symbol_short!("transfer"),
                    (user.clone(), other.clone(), 100_i128).into_val(&env)
                )),
                sub_invocations: std::vec![]
            }
        )]
    );
}
```

## Security Considerations

1. **Always require_auth first**: Before any state changes or validation
2. **Authorize the source**: Not the destination of transfers/changes  
3. **Check role/permissions**: After authorization but before actions
4. **Use specific contexts**: Auth is tied to exact function args
5. **Test authorization paths**: Verify both authorized and unauthorized cases
6. **Avoid auth bypass**: Don't provide alternative paths without auth
