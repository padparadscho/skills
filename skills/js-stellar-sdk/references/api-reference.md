# Stellar JS SDK API Reference

Package: `@stellar/stellar-sdk` (v14.x, requires Node 20+)

## Table of Contents

- [Module Structure](#module-structure)
- [Core Classes](#core-classes)
- [Horizon Module](#horizon-module)
- [RPC Module](#rpc-module)
- [Operations](#operations)
- [Assets](#assets)
- [Key Utilities](#key-utilities)
- [Federation & StellarToml](#federation--stellartoml)
- [WebAuth](#webauth)
- [Contract Module](#contract-module)

## Module Structure

The SDK exports several modules. Import only what you need for tree-shaking:

```typescript
// Named imports (preferred)
import {
  Keypair,
  TransactionBuilder,
  Operation,
  Asset,
  Networks,
  Memo,
  Account,
  StrKey,
  BASE_FEE,
  MuxedAccount,
  Horizon,
  rpc,
  contract,
  Federation,
  StellarToml,
  WebAuth,
} from "@stellar/stellar-sdk";

// Sub-path imports (if your environment supports package exports)
import { Server } from "@stellar/stellar-sdk/rpc";
import { Server as HorizonServer } from "@stellar/stellar-sdk/horizon";
```

**Do NOT install `@stellar/stellar-base` separately.** The SDK re-exports everything from `stellar-base`. Mismatched versions cause bugs.

## Core Classes

### Keypair

Create and manage Ed25519 key pairs.

```typescript
// Generate random keypair
const pair = Keypair.random();
pair.publicKey(); // G... address
pair.secret(); // S... secret key

// From existing secret
const pair2 = Keypair.fromSecret("S...");

// From public key (verify-only, cannot sign)
const pair3 = Keypair.fromPublicKey("G...");

// Sign and verify
const signature = pair.sign(dataBuffer);
pair.verify(dataBuffer, signature); // boolean
```

### TransactionBuilder

Build transactions with one or more operations.

```typescript
const account = await server.loadAccount(sourcePublicKey);
const fee = await server.fetchBaseFee();

const transaction = new TransactionBuilder(account, {
  fee: fee.toString(),
  networkPassphrase: Networks.TESTNET,
})
  .addOperation(
    Operation.payment({
      destination: "G...",
      asset: Asset.native(),
      amount: "10",
    }),
  )
  .addMemo(Memo.text("payment"))
  .setTimeout(30) // seconds; highly recommended
  .build();

transaction.sign(keypair);
```

Key methods:

- `.addOperation(op)` — add an operation (max 100 per tx, 1 for smart contract ops)
- `.addMemo(memo)` — attach a memo
- `.setTimeout(seconds)` — set time bounds (use 30 for typical, `TimeoutInfinite` for pre-auth)
- `.build()` — finalize and return a `Transaction`
- `.setNetworkPassphrase(passphrase)` — override network

### Transaction

```typescript
transaction.sign(keypair); // Sign with Keypair
transaction.toEnvelope().toXDR("base64"); // Serialize
transaction.hash(); // Transaction hash

// Deserialize
const tx = TransactionBuilder.fromXDR(xdrString, Networks.TESTNET);
```

### FeeBumpTransaction

Wrap an existing transaction with a higher fee:

```typescript
const feeBump = TransactionBuilder.buildFeeBumpTransaction(
  feeSourceKeypair, // account paying the new fee
  "1000", // new fee (in stroops)
  innerTransaction, // original transaction
  Networks.TESTNET,
);
feeBump.sign(feeSourceKeypair);
```

### Account

Represents a Stellar account with sequence number tracking:

```typescript
const account = new Account("G...", "123456"); // public key + sequence number
account.sequenceNumber(); // current sequence
account.incrementSequenceNumber();
```

### MuxedAccount

Virtual accounts sharing a single Stellar account (identified by a 64-bit ID):

```typescript
const muxed = new MuxedAccount(account, "12345"); // base account + muxed ID
muxed.accountId(); // M... address
muxed.baseAccount();
muxed.id(); // the 64-bit ID string
```

### Memo

```typescript
Memo.none();
Memo.text("hello"); // up to 28 bytes
Memo.id("12345"); // 64-bit unsigned int
Memo.hash(buffer); // 32-byte hash
Memo.return(buffer); // 32-byte hash (refund reference)
```

## Horizon Module

### Horizon.Server

REST API client for Horizon endpoints.

```typescript
const server = new Horizon.Server("https://horizon-testnet.stellar.org");

// Load account
const account = await server.loadAccount("G...");
console.log(account.balances);
console.log(account.sequence);

// Base fee
const fee = await server.fetchBaseFee();

// Submit transaction
const result = await server.submitTransaction(transaction);

// Submit async (returns immediately with PENDING/TRY_AGAIN_LATER/ERROR)
const asyncResult = await server.submitAsyncTransaction(transaction);
```

### Call Builders (Builder Pattern)

Chain methods to build queries, then call `.call()` or `.stream()`:

```typescript
// Transactions for an account
const page = await server
  .transactions()
  .forAccount("G...")
  .order("desc")
  .limit(10)
  .call();

console.log(page.records);
const nextPage = await page.next();

// Payments
await server
  .payments()
  .forAccount("G...")
  .join("transactions") // include tx details
  .call();

// Operations
await server.operations().forTransaction("txHash...").call();

// Effects
await server.effects().forAccount("G...").call();

// Orderbook
await server.orderbook(new Asset("USD", "G_ISSUER..."), Asset.native()).call();

// Trade aggregations
await server
  .tradeAggregation(
    Asset.native(),
    new Asset("USD", "G_ISSUER..."),
    startTime,
    endTime,
    resolution,
  )
  .call();

// Offers
await server.offers().forAccount("G...").call();

// Ledgers
await server.ledgers().order("desc").limit(1).call();

// Assets
await server.assets().forCode("USD").call();

// Accounts
await server.accounts().forSigner("G...").call();

// Claimable balances
await server.claimableBalances().claimant("G...").call();

// Fee stats
await server.feeStats();

// Strict send paths
await server
  .strictSendPaths(sourceAsset, sourceAmount, [destinationAsset])
  .call();

// Strict receive paths
await server
  .strictReceivePaths([sourceAsset], destinationAsset, destinationAmount)
  .call();

// Liquidity pools
await server.liquidityPools().forAssets(asset1, asset2).call();
```

### Streaming

Many endpoints support Server-Sent Events:

```typescript
const es = server
  .payments()
  .forAccount("G...")
  .cursor("now")
  .stream({
    onmessage: (payment) => {
      console.log("New payment:", payment.amount, payment.asset_type);
    },
    onerror: (error) => {
      console.error("Stream error:", error);
    },
  });

// Stop streaming
es(); // call the returned function to close
```

Streamable endpoints: `transactions()`, `operations()`, `payments()`, `effects()`, `ledgers()`, `offers()`, `orderbook()`, `trades()`.

## RPC Module

### rpc.Server

JSONRPC client for Stellar RPC (formerly Soroban RPC).

```typescript
const rpcServer = new rpc.Server("https://soroban-testnet.stellar.org");

// Health check
const health = await rpcServer.getHealth();

// Get account/ledger entry
const account = await rpcServer.getAccount("G...");

// Get latest ledger
const ledger = await rpcServer.getLatestLedger();

// Get transaction by hash
const tx = await rpcServer.getTransaction("txHash...");

// Get transactions
const txs = await rpcServer.getTransactions({
  startLedger: 1000,
  limit: 10,
});

// Get events (contract events, system events)
const events = await rpcServer.getEvents({
  startLedger: 1000,
  filters: [{ type: "contract", contractIds: ["C..."] }],
});

// Get network info
const network = await rpcServer.getNetwork();

// Simulate transaction (dry-run, no fees)
const simResult = await rpcServer.simulateTransaction(transaction);

// Prepare transaction (adds resource info from simulation)
const preparedTx = await rpcServer.prepareTransaction(transaction);

// Send transaction
const sendResult = await rpcServer.sendTransaction(transaction);

// Get version info
const version = await rpcServer.getVersionInfo();

// Get ledger entries (raw)
const entries = await rpcServer.getLedgerEntries([ledgerKey1, ledgerKey2]);

// Get fee stats
const feeStats = await rpcServer.getFeeStats();
```

## Operations

All available via `Operation.<name>(params)`. Each returns an xdr.Operation.

### Account Operations

```typescript
Operation.createAccount({ destination: "G...", startingBalance: "100" });
Operation.accountMerge({ destination: "G..." });
Operation.setOptions({
  homeDomain: "example.com",
  masterWeight: 1,
  lowThreshold: 1,
  medThreshold: 2,
  highThreshold: 3,
  signer: { ed25519PublicKey: "G...", weight: 1 },
  setFlags: AuthRequiredFlag | AuthRevocableFlag,
});
Operation.bumpSequence({ bumpTo: "12345" });
Operation.manageData({ name: "key", value: Buffer.from("value") }); // or value: null to delete
```

### Payment Operations

```typescript
Operation.payment({
  destination: "G...",
  asset: Asset.native(), // or new Asset('USD', 'G_ISSUER...')
  amount: "100.50",
});

Operation.pathPaymentStrictSend({
  sendAsset: Asset.native(),
  sendAmount: "10",
  destination: "G...",
  destAsset: new Asset("USD", "G_ISSUER..."),
  destMin: "9",
  path: [new Asset("EUR", "G_ISSUER2...")], // intermediate assets
});

Operation.pathPaymentStrictReceive({
  sendAsset: Asset.native(),
  sendMax: "11",
  destination: "G...",
  destAsset: new Asset("USD", "G_ISSUER..."),
  destAmount: "10",
  path: [],
});
```

### Trust & Asset Operations

```typescript
Operation.changeTrust({
  asset: new Asset("USD", "G_ISSUER..."),
  limit: "1000", // omit or '0' to remove trustline
});

Operation.setTrustLineFlags({
  trustor: "G...",
  asset: new Asset("USD", "G_ISSUER..."),
  flags: { authorized: true },
});

Operation.clawback({
  from: "G...",
  asset: new Asset("USD", "G_ISSUER..."),
  amount: "50",
});
```

### DEX Operations

```typescript
Operation.manageSellOffer({
  selling: Asset.native(),
  buying: new Asset("USD", "G_ISSUER..."),
  amount: "100",
  price: "0.5", // or { n: 1, d: 2 }
  offerId: "0", // '0' = new offer, existing ID = update
});

Operation.manageBuyOffer({
  selling: Asset.native(),
  buying: new Asset("USD", "G_ISSUER..."),
  buyAmount: "100",
  price: "0.5",
  offerId: "0",
});

Operation.createPassiveSellOffer({
  selling: Asset.native(),
  buying: new Asset("USD", "G_ISSUER..."),
  amount: "100",
  price: "0.5",
});
```

### Claimable Balance Operations

```typescript
Operation.createClaimableBalance({
  asset: Asset.native(),
  amount: "100",
  claimants: [
    new Claimant("G...", Claimant.predicateUnconditional()),
    new Claimant("G...", Claimant.predicateBeforeAbsoluteTime("1700000000")),
  ],
});

Operation.claimClaimableBalance({ balanceId: "00000000..." });
```

### Sponsorship Operations

```typescript
Operation.beginSponsoringFutureReserves({ sponsoredId: "G..." });
Operation.endSponsoringFutureReserves({});
Operation.revokeAccountSponsorship({ account: "G..." });
Operation.revokeTrustlineSponsorship({
  account: "G...",
  asset: new Asset("USD", "G..."),
});
```

### Liquidity Pool Operations

```typescript
const poolAsset = new LiquidityPoolAsset(
  Asset.native(),
  new Asset("USD", "G..."),
  LiquidityPoolFeeV18,
);

Operation.changeTrust({ asset: poolAsset }); // join pool

Operation.liquidityPoolDeposit({
  liquidityPoolId: "abc123...",
  maxAmountA: "100",
  maxAmountB: "200",
  minPrice: "0.5",
  maxPrice: "2.0",
});

Operation.liquidityPoolWithdraw({
  liquidityPoolId: "abc123...",
  amount: "50",
  minAmountA: "25",
  minAmountB: "45",
});
```

### Smart Contract Operations

```typescript
Operation.invokeHostFunction({ func: hostFunction, auth: [] });
Operation.extendFootprintTtl({ extendTo: 1000 });
Operation.restoreFootprint({});
```

## Assets

```typescript
// XLM (native asset)
Asset.native();

// Custom asset
const usd = new Asset("USD", "G_ISSUER_PUBLIC_KEY...");
usd.getCode(); // 'USD'
usd.getIssuer(); // 'G...'
usd.isNative(); // false
usd.getAssetType(); // 'credit_alphanum4' or 'credit_alphanum12'

// Asset codes: 1-4 chars = credit_alphanum4, 5-12 chars = credit_alphanum12
```

## Key Utilities

### StrKey

Encode/decode Stellar keys:

```typescript
StrKey.isValidEd25519PublicKey("G..."); // boolean
StrKey.isValidEd25519SecretSeed("S..."); // boolean
StrKey.isValidMed25519PublicKey("M..."); // muxed account
StrKey.isValidContract("C..."); // contract address

StrKey.encodeEd25519PublicKey(rawBytes);
StrKey.decodeEd25519PublicKey("G...");
```

### Address

```typescript
const addr = new Address("G..."); // or 'C...' for contracts
addr.toString();
addr.toScVal(); // convert to ScVal for contract calls
```

### XDR Helpers

```typescript
// Native JS <-> ScVal conversion
import { nativeToScVal, scValToNative } from "@stellar/stellar-sdk";

nativeToScVal("hello", { type: "string" }); // ScVal
nativeToScVal(100n, { type: "i128" }); // ScVal
nativeToScVal(true); // ScVal
scValToNative(scVal); // JS native

// XDR decode
xdr.TransactionEnvelope.fromXDR(base64String, "base64");
xdr.TransactionResult.fromXDR(base64String, "base64");
```

### SorobanDataBuilder

Build resource footprints for smart contract transactions:

```typescript
const sorobanData = new SorobanDataBuilder()
  .setReadOnly([ledgerKey1])
  .setReadWrite([ledgerKey2])
  .setResourceFee(100000)
  .build();
```

## Federation & StellarToml

### Federation

Resolve federated addresses:

```typescript
const result = await Federation.Server.resolve("user*example.com");
// { account_id: 'G...', memo_type: 'text', memo: '...' }

// Direct federation server
const fedServer = new Federation.Server("https://example.com/federation");
const record = await fedServer.resolveAddress("user*example.com");
```

### StellarToml

Fetch and parse `stellar.toml` files:

```typescript
const toml = await StellarToml.Resolver.resolve("example.com");
console.log(toml.CURRENCIES);
console.log(toml.TRANSFER_SERVER);
```

## WebAuth

SEP-10 authentication:

```typescript
import { WebAuth } from "@stellar/stellar-sdk";

// Build challenge (server-side)
const challenge = WebAuth.buildChallengeTx(
  serverKeypair, // server signing key
  clientPublicKey, // client to authenticate
  "example.com", // home domain
  300, // timeout in seconds
  Networks.TESTNET,
  "example.com", // web auth domain
);

// Read and verify challenge (client-side)
const { tx, clientAccountID, matchedHomeDomain } = WebAuth.readChallengeTx(
  challengeXDR,
  serverPublicKey,
  Networks.TESTNET,
  "example.com",
  "example.com",
);

// Verify challenge signatures
WebAuth.verifyChallengeTxSigners(
  challengeXDR,
  serverPublicKey,
  Networks.TESTNET,
  threshold,
  signersSummary,
  "example.com",
  "example.com",
);
```

## Contract Module

### contract.Client

Interact with deployed Soroban smart contracts:

```typescript
import { contract, Networks, Keypair } from "@stellar/stellar-sdk";
import { basicNodeSigner } from "@stellar/stellar-sdk/contract";

const client = new contract.Client({
  contractId: "C...",
  networkPassphrase: Networks.TESTNET,
  rpcUrl: "https://soroban-testnet.stellar.org",
  publicKey: keypair.publicKey(),
  ...basicNodeSigner(keypair, Networks.TESTNET),
});

// Call contract methods (auto-generated from contract spec)
const result = await client.myMethod({ arg1: "value" });

// Deploy a contract
const tx = await contract.Client.deploy(
  { constructorArg: value },
  {
    networkPassphrase: Networks.TESTNET,
    rpcUrl: "https://soroban-testnet.stellar.org",
    wasmHash: Buffer.from("abcdef...", "hex"),
    publicKey: keypair.publicKey(),
    ...basicNodeSigner(keypair, Networks.TESTNET),
  },
);
```

### contract.Spec

Parse and work with contract specifications:

```typescript
const spec = new contract.Spec([xdrSpecEntry1, xdrSpecEntry2]);
spec.funcs(); // list contract functions
```

### contract.AssembledTransaction

Low-level transaction assembly for contract calls. Used internally by `contract.Client`.

### contract.SentTransaction

Represents a submitted contract transaction with polling:

```typescript
const sent = await assembledTx.signAndSend();
// Polls RPC until resolved
console.log(sent.result); // parsed result
```

## Error Classes

```typescript
import {
  AccountRequiresMemoError,
  BadRequestError,
  BadResponseError,
  NetworkError,
  NotFoundError,
} from "@stellar/stellar-sdk";
```

- `AccountRequiresMemoError` — destination account requires a memo
- `BadRequestError` — 400-level Horizon error
- `BadResponseError` — unexpected Horizon response
- `NetworkError` — connectivity issue
- `NotFoundError` — 404 from Horizon

## Constants

```typescript
import { BASE_FEE, Networks, TimeoutInfinite } from "@stellar/stellar-sdk";

BASE_FEE; // '100' (minimum fee in stroops)
Networks.PUBLIC; // mainnet passphrase
Networks.TESTNET; // testnet passphrase
Networks.FUTURENET; // futurenet passphrase
Networks.STANDALONE; // standalone passphrase
TimeoutInfinite; // 0 (no timeout)
```

## Account Flags

```typescript
import {
  AuthRequiredFlag, // 0x1
  AuthRevocableFlag, // 0x2
  AuthImmutableFlag, // 0x4
  AuthClawbackEnabledFlag, // 0x8
} from "@stellar/stellar-sdk";
```
