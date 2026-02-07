# Testing Patterns

Soroban SDK provides comprehensive testing utilities through the `testutils` feature.

## Basic Test Structure

```rust
#[cfg(test)]
mod test {
    use super::*;
    use soroban_sdk::Env;

    #[test]
    fn test_basic() {
        let env = Env::default();
        let contract_id = env.register(MyContract, ());
        let client = MyContractClient::new(&env, &contract_id);

        let result = client.my_function(&arg);
        assert_eq!(result, expected);
    }
}
```

**Key components:**

- `Env::default()` - Creates test environment
- `env.register(Contract, ())` - Registers contract, returns ID
- `ContractClient::new()` - Auto-generated client for calling contract

## Test Environment Setup

### Register Multiple Contracts

```rust
#[test]
fn test_multi_contract() {
    let env = Env::default();

    let contract1_id = env.register(Contract1, ());
    let contract2_id = env.register(Contract2, ());

    let client1 = Contract1Client::new(&env, &contract1_id);
    let client2 = Contract2Client::new(&env, &contract2_id);

    // Test interactions
    client1.call_other(&contract2_id);
}
```

### Deploy from WASM

```rust
#[test]
fn test_from_wasm() {
    let env = Env::default();

    // Load compiled WASM
    let wasm = include_bytes!("../target/wasm32v1-none/release/contract.wasm");
    let contract_id = env.register_contract_wasm(None, wasm);

    // Use contract
    let client = MyContractClient::new(&env, &contract_id);
}
```

## Mock Authorization

Authorization is automatically checked in tests. Use `mock_all_auths()` to bypass:

```rust
#[test]
fn test_with_auth() {
    let env = Env::default();
    let contract_id = env.register(MyContract, ());
    let client = MyContractClient::new(&env, &contract_id);

    let user = Address::generate(&env);

    // Mock all authorizations
    env.mock_all_auths();

    // This will succeed even though user hasn't actually signed
    client.transfer(&user, &recipient, &100);
}
```

### Selective Authorization Mocking

```rust
#[test]
fn test_specific_auth() {
    let env = Env::default();
    let contract_id = env.register(MyContract, ());
    let client = MyContractClient::new(&env, &contract_id);

    let user1 = Address::generate(&env);
    let user2 = Address::generate(&env);

    // Mock only user1's auth
    env.mock_auths(&[user1.clone()]);

    // This succeeds
    client.transfer(&user1, &user2, &100);

    // This would fail if user2 needs auth
    // client.transfer(&user2, &user1, &50);
}
```

### Verify Authorization

```rust
#[test]
fn test_auth_verification() {
    let env = Env::default();
    let contract_id = env.register(MyContract, ());
    let client = MyContractClient::new(&env, &contract_id);

    let user = Address::generate(&env);
    env.mock_all_auths();

    client.transfer(&user, &recipient, &100);

    // Verify what was authorized
    assert_eq!(
        env.auths(),
        std::vec![(
            user.clone(),
            AuthorizedInvocation {
                function: AuthorizedFunction::Contract((
                    contract_id,
                    symbol_short!("transfer"),
                    (user.clone(), recipient.clone(), 100_i128).into_val(&env)
                )),
                sub_invocations: std::vec![]
            }
        )]
    );
}
```

## Ledger State Control

### Set Ledger Sequence and Timestamp

```rust
#[test]
fn test_time_dependent() {
    let env = Env::default();

    // Set to specific ledger
    env.ledger().set_sequence_number(100);
    env.ledger().set_timestamp(1000000);

    let contract_id = env.register(MyContract, ());
    let client = MyContractClient::new(&env, &contract_id);

    // Contract sees ledger 100, timestamp 1000000
    let result = client.time_based_function();

    // Advance ledger
    env.ledger().set_sequence_number(200);
    env.ledger().set_timestamp(2000000);

    let result2 = client.time_based_function();
}
```

### Jump Forward in Time

```rust
#[test]
fn test_ttl_expiration() {
    let env = Env::default();
    let contract_id = env.register(MyContract, ());
    let client = MyContractClient::new(&env, &contract_id);

    env.mock_all_auths();

    // Store with short TTL
    client.store_temporary(&key, &value);

    // Data should exist
    assert!(client.has_data(&key));

    // Jump 1000 ledgers forward
    env.ledger().set_sequence_number(
        env.ledger().sequence() + 1000
    );

    // Data should be expired
    assert!(!client.has_data(&key));
}
```

## Budget and Resource Testing

### Set Budget

```rust
#[test]
fn test_budget() {
    let env = Env::default();

    // Set CPU and memory budget
    env.budget().reset_limits(100_000, 100_000);

    let contract_id = env.register(MyContract, ());
    let client = MyContractClient::new(&env, &contract_id);

    client.expensive_function();

    // Check budget usage
    let cpu_used = env.budget().cpu_instruction_cost();
    let mem_used = env.budget().memory_bytes_cost();

    println!("CPU: {}, Memory: {}", cpu_used, mem_used);
}
```

### Print Budget

```rust
#[test]
fn test_with_budget_print() {
    let env = Env::default();
    env.budget().reset_unlimited();

    let contract_id = env.register(MyContract, ());
    let client = MyContractClient::new(&env, &contract_id);

    client.function();

    // Print budget breakdown
    env.budget().print();
}
```

## Error Testing

### Test Panics

```rust
#[test]
#[should_panic(expected = "Insufficient balance")]
fn test_insufficient_balance() {
    let env = Env::default();
    let contract_id = env.register(MyContract, ());
    let client = MyContractClient::new(&env, &contract_id);

    env.mock_all_auths();

    // Should panic
    client.transfer(&from, &to, &999999);
}
```

### Test Custom Errors

```rust
#[test]
fn test_custom_error() {
    let env = Env::default();
    let contract_id = env.register(MyContract, ());
    let client = MyContractClient::new(&env, &contract_id);

    env.mock_all_auths();

    let result = client.try_invalid_operation();

    // Check error
    assert_eq!(
        result,
        Err(Ok(Error::InvalidOperation))
    );
}
```

## Event Testing

### Verify Events

```rust
#[test]
fn test_events() {
    let env = Env::default();
    let contract_id = env.register(MyContract, ());
    let client = MyContractClient::new(&env, &contract_id);

    env.mock_all_auths();

    client.transfer(&from, &to, &100);

    // Get all events
    let events = env.events().all();

    // Verify transfer event
    assert_eq!(events.len(), 1);
    let (contract, topics, data) = &events[0];
    assert_eq!(contract, &contract_id);

    // Verify event details
    // ...
}
```

## Storage Testing

### Inspect Storage

```rust
#[test]
fn test_storage_state() {
    let env = Env::default();
    let contract_id = env.register(MyContract, ());
    let client = MyContractClient::new(&env, &contract_id);

    env.mock_all_auths();

    client.set_value(&key, &value);

    // Direct storage access for verification
    let stored: i128 = env.as_contract(&contract_id, || {
        env.storage().persistent().get(&key).unwrap()
    });

    assert_eq!(stored, value);
}
```

### Test Storage TTL

```rust
#[test]
fn test_storage_ttl() {
    let env = Env::default();
    let contract_id = env.register(MyContract, ());
    let client = MyContractClient::new(&env, &contract_id);

    env.mock_all_auths();

    // Store value
    client.store_with_ttl(&key, &value);

    // Check TTL was set correctly
    // Note: TTL inspection APIs may vary by SDK version
}
```

## Token Testing

### Use Test Token

```rust
use soroban_sdk::token;

#[test]
fn test_with_token() {
    let env = Env::default();
    env.mock_all_auths();

    // Create test token
    let token_admin = Address::generate(&env);
    let token_id = env.register_stellar_asset_contract_v2(token_admin.clone());
    let token = token::Client::new(&env, &token_id);
    let token_admin_client = token::StellarAssetClient::new(&env, &token_id);

    // Mint tokens
    token_admin_client.mint(&user, &1000);

    // Test contract with token
    let contract_id = env.register(MyContract, ());
    let client = MyContractClient::new(&env, &contract_id);

    client.deposit_token(&token_id, &100);

    // Verify balance
    assert_eq!(token.balance(&user), 900);
}
```

## Integration Testing Patterns

### Multi-Contract Interaction

```rust
#[test]
fn test_contract_integration() {
    let env = Env::default();
    env.mock_all_auths();

    // Setup multiple contracts
    let token_id = env.register_stellar_asset_contract_v2(admin.clone());
    let vault_id = env.register(VaultContract, ());
    let strategy_id = env.register(StrategyContract, ());

    let vault = VaultContractClient::new(&env, &vault_id);
    let strategy = StrategyContractClient::new(&env, &strategy_id);

    // Initialize
    vault.initialize(&token_id, &strategy_id);

    // Test full flow
    vault.deposit(&user, &1000);
    strategy.execute(&vault_id);
    vault.withdraw(&user, &500);
}
```

## Test Organization

### Use Test Fixtures

```rust
fn setup_test<'a>() -> (Env, Address, MyContractClient<'a>) {
    let env = Env::default();
    env.mock_all_auths();

    let contract_id = env.register(MyContract, ());
    let client = MyContractClient::new(&env, &contract_id);

    (env, contract_id, client)
}

#[test]
fn test_function_a() {
    let (env, contract_id, client) = setup_test();
    // Test logic
}

#[test]
fn test_function_b() {
    let (env, contract_id, client) = setup_test();
    // Test logic
}
```

## Best Practices

1. **Mock auth in tests**: Use `env.mock_all_auths()` for convenience
2. **Test error paths**: Verify panics and custom errors
3. **Verify events**: Check that events are published correctly
4. **Test TTL**: Advance ledger to verify expiration
5. **Use fixtures**: Reduce duplication in test setup
6. **Test budget**: Ensure functions don't exceed resource limits
7. **Integration tests**: Test multi-contract interactions
8. **Check storage**: Verify storage state directly when needed
