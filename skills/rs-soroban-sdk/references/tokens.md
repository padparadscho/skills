# Token Integration

Soroban provides built-in support for tokens, including the Stellar Asset Contract (SAC).

## Token Client

Use `TokenClient` to interact with any token contract:

```rust
use soroban_sdk::token::{TokenClient, TokenInterface};

pub fn use_token(env: Env, token_address: Address, from: Address, to: Address, amount: i128) {
    let token = TokenClient::new(&env, &token_address);
    
    // Transfer tokens
    token.transfer(&from, &to, &amount);
    
    // Check balance
    let balance = token.balance(&from);
    
    // Approve spender
    token.approve(&from, &spender, &amount, &expiration_ledger);
}
```

## Stellar Asset Contract (SAC)

SAC is the standard token interface on Stellar. All Stellar assets (native XLM, issued assets) are automatically wrapped as SAC.

### Standard SAC Functions

```rust
pub trait TokenInterface {
    // Transfer tokens
    fn transfer(env: Env, from: Address, to: Address, amount: i128);
    
    // Transfer from allowance
    fn transfer_from(env: Env, spender: Address, from: Address, to: Address, amount: i128);
    
    // Approve spender
    fn approve(env: Env, from: Address, spender: Address, amount: i128, expiration_ledger: u32);
    
    // Get balance
    fn balance(env: Env, id: Address) -> i128;
    
    // Get allowance
    fn allowance(env: Env, from: Address, spender: Address) -> i128;
    
    // Get decimals
    fn decimals(env: Env) -> u32;
    
    // Get name
    fn name(env: Env) -> String;
    
    // Get symbol
    fn symbol(env: Env) -> String;
}
```

### Admin Functions (StellarAssetClient)

For Stellar assets, admin functions are available:

```rust
use soroban_sdk::token::{StellarAssetClient};

pub fn manage_token(env: Env, token_address: Address, admin: Address) {
    let token_admin = StellarAssetClient::new(&env, &token_address);
    
    // Mint new tokens (admin only)
    token_admin.mint(&recipient, &amount);
    
    // Set admin
    token_admin.set_admin(&new_admin);
    
    // Burn tokens
    token_admin.burn(&from, &amount);
    
    // Freeze/unfreeze account
    token_admin.set_authorized(&account, &true);
    
    // Clawback tokens
    token_admin.clawback(&from, &amount);
}
```

## Common Token Patterns

### Deposit Tokens to Contract

```rust
pub fn deposit(env: Env, user: Address, token: Address, amount: i128) {
    user.require_auth();
    
    let token_client = TokenClient::new(&env, &token);
    
    // Transfer from user to contract
    token_client.transfer(&user, &env.current_contract_address(), &amount);
    
    // Update user's deposit balance
    let balance_key = (symbol_short!("DEPOSIT"), user.clone(), token.clone());
    let current = env.storage().persistent().get(&balance_key).unwrap_or(0);
    env.storage().persistent().set(&balance_key, &(current + amount));
    env.storage().persistent().extend_ttl(&balance_key, 100, 518_400);
}
```

### Withdraw Tokens from Contract

```rust
pub fn withdraw(env: Env, user: Address, token: Address, amount: i128) {
    user.require_auth();
    
    // Check user's deposit balance
    let balance_key = (symbol_short!("DEPOSIT"), user.clone(), token.clone());
    let current: i128 = env.storage()
        .persistent()
        .get(&balance_key)
        .expect("No deposits");
    
    assert!(current >= amount, "Insufficient deposit");
    
    // Update balance
    env.storage().persistent().set(&balance_key, &(current - amount));
    env.storage().persistent().extend_ttl(&balance_key, 100, 518_400);
    
    // Transfer tokens to user
    let token_client = TokenClient::new(&env, &token);
    token_client.transfer(&env.current_contract_address(), &user, &amount);
}
```

### Use Allowance Pattern

```rust
pub fn spend_allowance(
    env: Env,
    spender: Address,
    token: Address,
    from: Address,
    amount: i128
) {
    spender.require_auth();
    
    let token_client = TokenClient::new(&env, &token);
    
    // Check allowance
    let allowance = token_client.allowance(&from, &spender);
    assert!(allowance >= amount, "Insufficient allowance");
    
    // Use transfer_from
    token_client.transfer_from(&spender, &from, &env.current_contract_address(), &amount);
}
```

### Multi-Token Vault

```rust
#[contracttype]
pub enum DataKey {
    SupportedToken(Address),
    UserBalance(Address, Address), // (user, token)
}

pub fn add_supported_token(env: Env, admin: Address, token: Address) {
    admin.require_auth();
    require_admin(&env, &admin);
    
    let key = DataKey::SupportedToken(token.clone());
    env.storage().persistent().set(&key, &true);
}

pub fn deposit_any(env: Env, user: Address, token: Address, amount: i128) {
    user.require_auth();
    
    // Verify token is supported
    let supported_key = DataKey::SupportedToken(token.clone());
    assert!(
        env.storage().persistent().get(&supported_key).unwrap_or(false),
        "Token not supported"
    );
    
    // Transfer tokens
    let token_client = TokenClient::new(&env, &token);
    token_client.transfer(&user, &env.current_contract_address(), &amount);
    
    // Update balance
    let balance_key = DataKey::UserBalance(user.clone(), token.clone());
    let current = env.storage().persistent().get(&balance_key).unwrap_or(0);
    env.storage().persistent().set(&balance_key, &(current + amount));
}
```

### Token Swap

```rust
pub fn swap(
    env: Env,
    user: Address,
    token_in: Address,
    token_out: Address,
    amount_in: i128,
    min_amount_out: i128
) -> i128 {
    user.require_auth();
    
    let token_in_client = TokenClient::new(&env, &token_in);
    let token_out_client = TokenClient::new(&env, &token_out);
    
    // Calculate output amount (simplified - real DEX would check reserves)
    let amount_out = calculate_output(&env, &token_in, &token_out, amount_in);
    assert!(amount_out >= min_amount_out, "Slippage too high");
    
    // Transfer input tokens from user to contract
    token_in_client.transfer(&user, &env.current_contract_address(), &amount_in);
    
    // Transfer output tokens from contract to user
    token_out_client.transfer(&env.current_contract_address(), &user, &amount_out);
    
    // Emit event
    env.events().publish(
        (symbol_short!("swap"), user.clone()),
        (token_in, token_out, amount_in, amount_out)
    );
    
    amount_out
}
```

## Creating Custom Tokens

Implement the `TokenInterface` trait:

```rust
use soroban_sdk::token::TokenInterface;

#[contract]
pub struct MyToken;

#[contractimpl]
impl TokenInterface for MyToken {
    fn transfer(env: Env, from: Address, to: Address, amount: i128) {
        from.require_auth();
        
        // Get balances
        let from_balance = get_balance(&env, &from);
        let to_balance = get_balance(&env, &to);
        
        // Validate
        assert!(amount > 0, "Invalid amount");
        assert!(from_balance >= amount, "Insufficient balance");
        
        // Update balances
        set_balance(&env, &from, from_balance - amount);
        set_balance(&env, &to, to_balance + amount);
        
        // Emit event
        env.events().publish((symbol_short!("transfer"), from, to), amount);
    }
    
    fn balance(env: Env, id: Address) -> i128 {
        get_balance(&env, &id)
    }
    
    // Implement other required functions...
}

// Helper functions
fn get_balance(env: &Env, address: &Address) -> i128 {
    let key = (symbol_short!("BALANCE"), address.clone());
    env.storage().persistent().get(&key).unwrap_or(0)
}

fn set_balance(env: &Env, address: &Address, amount: i128) {
    let key = (symbol_short!("BALANCE"), address.clone());
    env.storage().persistent().set(&key, &amount);
    env.storage().persistent().extend_ttl(&key, 100, 518_400);
}
```

## Testing with Tokens

```rust
#[test]
fn test_deposit_token() {
    let env = Env::default();
    env.mock_all_auths();
    
    // Create test token
    let admin = Address::generate(&env);
    let token_id = env.register_stellar_asset_contract_v2(admin.clone());
    let token = token::Client::new(&env, &token_id);
    let token_admin = token::StellarAssetClient::new(&env, &token_id);
    
    // Mint tokens to user
    let user = Address::generate(&env);
    token_admin.mint(&user, &1000);
    
    // Register vault contract
    let vault_id = env.register(VaultContract, ());
    let vault = VaultContractClient::new(&env, &vault_id);
    
    // Test deposit
    vault.deposit(&user, &token_id, &500);
    
    // Verify balances
    assert_eq!(token.balance(&user), 500);
    assert_eq!(token.balance(&vault_id), 500);
}
```

## Token Standards Compliance

When implementing custom tokens, follow these standards:

1. **Events**: Emit transfer events for all balance changes
2. **Authorization**: Always `require_auth()` on the `from` address
3. **Validation**: Check amounts are positive, balances sufficient
4. **Decimals**: Use consistent decimals (typically 7 for Stellar assets)
5. **Allowances**: Implement approve/transfer_from pattern
6. **Metadata**: Provide name, symbol, decimals

## Security Best Practices

⚠️ **Token operations are high-risk**: Always validate thoroughly.

1. **Validate amounts**: Check for positive, non-zero amounts (prevent overflow)
2. **Check balances**: Verify sufficient balance before transfers (prevent underflow)
3. **Authorize sender**: Not the recipient (critical authorization error)
4. **Extend TTL**: On all balance updates (prevent data loss)
5. **Emit events**: For all state changes (auditability)
6. **Handle rounding**: Be careful with division in swaps/fees (precision loss)
7. **Prevent reentrancy**: Soroban doesn't have reentrancy issues, but be aware of cross-contract calls
8. **Whitelist tokens**: Don't trust arbitrary token addresses (malicious contracts)
9. **Check return values**: Token operations may fail silently
10. **Use slippage protection**: For swaps and AMM operations (front-running prevention)

```rust
// ✓ Good: Validate then transfer
pub fn transfer(env: Env, from: Address, to: Address, amount: i128) {
    from.require_auth();
    assert!(amount > 0, "Invalid amount");
    
    let balance = get_balance(&env, &from);
    assert!(balance >= amount, "Insufficient balance");
    
    set_balance(&env, &from, balance - amount);
    set_balance(&env, &to, get_balance(&env, &to) + amount);
}

// ✗ Bad: No validation
pub fn transfer(env: Env, from: Address, to: Address, amount: i128) {
    from.require_auth();
    set_balance(&env, &from, get_balance(&env, &from) - amount); // Can underflow!
    set_balance(&env, &to, get_balance(&env, &to) + amount);
}
```
