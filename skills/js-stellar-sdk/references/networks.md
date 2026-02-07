# Stellar Networks & Configuration

## Networks

| Network       | Passphrase                                       | Horizon URL                             | RPC URL                               | Friendbot                                 |
| ------------- | ------------------------------------------------ | --------------------------------------- | ------------------------------------- | ----------------------------------------- |
| **Mainnet**   | `Public Global Stellar Network ; September 2015` | Multiple providers                      | Third-party only                      | No                                        |
| **Testnet**   | `Test SDF Network ; September 2015`              | `https://horizon-testnet.stellar.org`   | `https://soroban-testnet.stellar.org` | `https://friendbot.stellar.org`           |
| **Futurenet** | `Test SDF Future Network ; October 2022`         | `https://horizon-futurenet.stellar.org` | `https://rpc-futurenet.stellar.org`   | `https://friendbot-futurenet.stellar.org` |

SDK constants:

```typescript
import { Networks } from "@stellar/stellar-sdk";

Networks.PUBLIC; // 'Public Global Stellar Network ; September 2015'
Networks.TESTNET; // 'Test SDF Network ; September 2015'
Networks.FUTURENET; // 'Test SDF Future Network ; October 2022'
Networks.STANDALONE; // 'Standalone Network ; February 2017'
```

## Server Setup

### Horizon (REST API for classical Stellar data)

```typescript
import { Horizon } from "@stellar/stellar-sdk";

// Testnet
const testServer = new Horizon.Server("https://horizon-testnet.stellar.org");

// Mainnet (use a provider — SDF does not provide free mainnet Horizon)
const mainServer = new Horizon.Server("https://horizon.stellar.org");
```

### RPC (JSONRPC for smart contracts & modern features)

```typescript
import { rpc } from "@stellar/stellar-sdk";

const testRpc = new rpc.Server("https://soroban-testnet.stellar.org");
```

## Friendbot (Testnet/Futurenet Only)

Fund test accounts with 10,000 XLM:

```typescript
// Simple fetch
await fetch(`https://friendbot.stellar.org?addr=${publicKey}`);

// Or use the SDK
const response = await fetch(
  `https://friendbot.stellar.org?addr=${keypair.publicKey()}`,
);
```

For Futurenet: `https://friendbot-futurenet.stellar.org?addr=G...`

## Switching from Testnet to Mainnet

Change three things:

1. **Network passphrase**: `Networks.TESTNET` → `Networks.PUBLIC`
2. **Horizon URL**: testnet URL → your mainnet Horizon provider
3. **RPC URL** (if using): testnet RPC → your mainnet RPC provider

```typescript
// Testnet
const server = new Horizon.Server("https://horizon-testnet.stellar.org");
const networkPassphrase = Networks.TESTNET;

// Mainnet
const server = new Horizon.Server("https://your-horizon-provider.com");
const networkPassphrase = Networks.PUBLIC;
```

Transactions signed for one network are invalid on another (passphrases are part of the transaction hash).

## Testnet Resets

Testnet resets quarterly. All accounts, balances, trustlines, offers, and contract data are wiped. Automate test data recreation with scripts using Friendbot + transaction submissions.

## Key Amounts & Limits

| Parameter                      | Value                                               |
| ------------------------------ | --------------------------------------------------- |
| Minimum account balance        | 1 XLM (2 base reserves × 0.5 XLM)                   |
| Per subentry reserve           | 0.5 XLM (trustlines, offers, signers, data entries) |
| Maximum subentries per account | 1,000                                               |
| Minimum transaction fee        | 100 stroops (0.00001 XLM)                           |
| Fee per operation              | `base_fee × number_of_operations`                   |
| 1 XLM                          | 10,000,000 stroops                                  |
| Max operations per transaction | 100 (1 for smart contract transactions)             |
| Max signers per account        | 20                                                  |
| Asset code length              | 1-4 chars (alphanum4) or 5-12 chars (alphanum12)    |
| Memo text max length           | 28 bytes                                            |
| Amount precision               | 7 decimal places                                    |
| Max path payment hops          | 5 intermediate assets                               |

## CLI: Generate TypeScript Contract Bindings

```bash
# From a deployed contract
npx @stellar/stellar-sdk generate \
  --contract-id CABC...XYZ \
  --network testnet \
  --output-dir ./my-contract-client

# From a WASM file
npx @stellar/stellar-sdk generate \
  --wasm ./path/to/contract.wasm \
  --output-dir ./my-contract-client \
  --contract-name my-contract

# From a WASM hash on network
npx @stellar/stellar-sdk generate \
  --wasm-hash <hex-hash> \
  --network testnet \
  --output-dir ./my-contract-client \
  --contract-name my-contract

# Mainnet (requires --rpc-url)
npx @stellar/stellar-sdk generate \
  --contract-id CABC...XYZ \
  --rpc-url https://your-rpc-provider.com \
  --network mainnet \
  --output-dir ./my-contract-client
```

CLI options: `--overwrite`, `--allow-http`, `--timeout <ms>`, `--headers <json>`.

## Browser Usage

CDN:

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/stellar-sdk/{version}/stellar-sdk.js"></script>
<script>
  console.log(StellarSdk);
  const server = new StellarSdk.Horizon.Server(
    "https://horizon-testnet.stellar.org",
  );
</script>
```

Custom builds (smaller bundles):

- `stellar-sdk-no-axios.js` — no axios (no HTTP requests from browser)
- `stellar-sdk-no-eventsource.js` — no streaming
- `stellar-sdk-minimal.js` — no axios, no eventsource

## React Native Setup

1. `yarn add --dev rn-nodeify`
2. Postinstall: `yarn rn-nodeify --install url,events,https,http,util,stream,crypto,vm,buffer --hack --yarn`
3. Uncomment `require('crypto')` in shim.js
4. `react-native link react-native-randombytes`
5. Add `import "./shim";` to entry point
6. `yarn add @stellar/stellar-sdk`

For Expo: use `expo-random` to create keypairs since `Keypair.random()` won't work.

## stellar-sdk vs stellar-base

- `stellar-sdk` — high-level: Horizon client, RPC client, transaction building + submission
- `stellar-base` — low-level: XDR, key encoding, operation construction

**Always use `stellar-sdk`.** It re-exports everything from `stellar-base`. Never install both — version mismatches cause subtle bugs.

For production Node.js: ensure `sodium-native` installs successfully for faster Ed25519 signing. Check via `StellarSdk.FastSigning` (returns `true` if using native crypto).
