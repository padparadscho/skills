# Stellar JS SDK Code Examples

## Table of Contents

- [Setup & Installation](#setup--installation)
- [Account Management](#account-management)
- [Payments](#payments)
- [Path Payments](#path-payments)
- [Asset Issuance](#asset-issuance)
- [Trustlines](#trustlines)
- [DEX Trading](#dex-trading)
- [Streaming](#streaming)
- [Querying Horizon](#querying-horizon)
- [Fee-Bump Transactions](#fee-bump-transactions)
- [Claimable Balances](#claimable-balances)
- [Sponsored Reserves](#sponsored-reserves)
- [Multisig Transactions](#multisig-transactions)
- [RPC Interactions](#rpc-interactions)
- [SEP-10 Authentication](#sep-10-authentication)
- [Federation](#federation)
- [Error Handling](#error-handling)

## Setup & Installation

```bash
npm install @stellar/stellar-sdk
# or
yarn add @stellar/stellar-sdk
```

```typescript
import {
  Keypair,
  Horizon,
  TransactionBuilder,
  Networks,
  Operation,
  Asset,
  Memo,
  BASE_FEE,
} from "@stellar/stellar-sdk";

const server = new Horizon.Server("https://horizon-testnet.stellar.org");
```

## Account Management

### Create and Fund a Testnet Account

```typescript
import { Keypair } from "@stellar/stellar-sdk";

const pair = Keypair.random();
console.log("Public Key:", pair.publicKey());
console.log("Secret Key:", pair.secret());

// Fund via Friendbot (testnet only)
const response = await fetch(
  `https://friendbot.stellar.org?addr=${pair.publicKey()}`,
);
const data = await response.json();
console.log("Account funded:", data);
```

### Load Account Details

```typescript
const account = await server.loadAccount("G...");
console.log("Sequence:", account.sequence);
console.log("Balances:");
account.balances.forEach((balance) => {
  if (balance.asset_type === "native") {
    console.log(`  XLM: ${balance.balance}`);
  } else {
    console.log(`  ${balance.asset_code}: ${balance.balance}`);
  }
});
```

### Create Account On-Chain

```typescript
const sourceAccount = await server.loadAccount(sourceKeypair.publicKey());
const fee = await server.fetchBaseFee();

const tx = new TransactionBuilder(sourceAccount, {
  fee: fee.toString(),
  networkPassphrase: Networks.TESTNET,
})
  .addOperation(
    Operation.createAccount({
      destination: newKeypair.publicKey(),
      startingBalance: "10", // minimum ~1 XLM for base reserve
    }),
  )
  .setTimeout(30)
  .build();

tx.sign(sourceKeypair);
const result = await server.submitTransaction(tx);
```

## Payments

### Send XLM

```typescript
const sourceAccount = await server.loadAccount(sourceKeypair.publicKey());

const tx = new TransactionBuilder(sourceAccount, {
  fee: BASE_FEE,
  networkPassphrase: Networks.TESTNET,
})
  .addOperation(
    Operation.payment({
      destination: "GAIRISXKPLOWZBMFRPU5XRGUUX3VMA3ZEWKBM5MSNRU3CHV6P4PYZ74D",
      asset: Asset.native(),
      amount: "350.1234567", // up to 7 decimal places
    }),
  )
  .addMemo(Memo.text("Test payment"))
  .setTimeout(30)
  .build();

tx.sign(sourceKeypair);

try {
  const result = await server.submitTransaction(tx);
  console.log("Success:", result.hash);
} catch (e) {
  console.error("Error:", e.response?.data?.extras?.result_codes);
}
```

### Send Custom Asset

```typescript
const usd = new Asset("USD", "GISSUER...");

const tx = new TransactionBuilder(sourceAccount, {
  fee: BASE_FEE,
  networkPassphrase: Networks.TESTNET,
})
  .addOperation(
    Operation.payment({
      destination: receiverPublicKey,
      asset: usd,
      amount: "25.00",
    }),
  )
  .setTimeout(30)
  .build();

tx.sign(sourceKeypair);
await server.submitTransaction(tx);
```

### Multiple Operations in One Transaction

```typescript
const tx = new TransactionBuilder(sourceAccount, {
  fee: (parseInt(BASE_FEE) * 3).toString(), // fee per op
  networkPassphrase: Networks.TESTNET,
})
  .addOperation(
    Operation.payment({
      destination: "G_ALICE...",
      asset: Asset.native(),
      amount: "10",
    }),
  )
  .addOperation(
    Operation.payment({
      destination: "G_BOB...",
      asset: Asset.native(),
      amount: "20",
    }),
  )
  .addOperation(
    Operation.payment({
      destination: "G_CAROL...",
      asset: Asset.native(),
      amount: "30",
    }),
  )
  .setTimeout(30)
  .build();

tx.sign(sourceKeypair);
await server.submitTransaction(tx);
```

## Path Payments

### Strict Send Path Payment

```typescript
// Send exactly 10 XLM, receive at least 9 USD
const tx = new TransactionBuilder(sourceAccount, {
  fee: BASE_FEE,
  networkPassphrase: Networks.TESTNET,
})
  .addOperation(
    Operation.pathPaymentStrictSend({
      sendAsset: Asset.native(),
      sendAmount: "10",
      destination: receiverPublicKey,
      destAsset: new Asset("USD", "G_ISSUER..."),
      destMin: "9",
      path: [], // let the network find the path
    }),
  )
  .setTimeout(30)
  .build();
```

### Strict Receive Path Payment

```typescript
// Receive exactly 100 USD, send at most 110 XLM
const tx = new TransactionBuilder(sourceAccount, {
  fee: BASE_FEE,
  networkPassphrase: Networks.TESTNET,
})
  .addOperation(
    Operation.pathPaymentStrictReceive({
      sendAsset: Asset.native(),
      sendMax: "110",
      destination: receiverPublicKey,
      destAsset: new Asset("USD", "G_ISSUER..."),
      destAmount: "100",
      path: [],
    }),
  )
  .setTimeout(30)
  .build();
```

### Find Payment Paths

```typescript
// Strict send paths
const sendPaths = await server
  .strictSendPaths(Asset.native(), "10", [new Asset("USD", "G_ISSUER...")])
  .call();

// Strict receive paths
const receivePaths = await server
  .strictReceivePaths([Asset.native()], new Asset("USD", "G_ISSUER..."), "100")
  .call();

sendPaths.records.forEach((path) => {
  console.log(
    `${path.source_amount} ${path.source_asset_code || "XLM"} -> ` +
      `${path.destination_amount} ${path.destination_asset_code || "XLM"}`,
  );
});
```

## Asset Issuance

### Issue a Custom Asset

```typescript
const issuingKeypair = Keypair.random();
const distributionKeypair = Keypair.random();

// 1. Fund both accounts (testnet)
await fetch(`https://friendbot.stellar.org?addr=${issuingKeypair.publicKey()}`);
await fetch(
  `https://friendbot.stellar.org?addr=${distributionKeypair.publicKey()}`,
);

const myAsset = new Asset("MYTOKEN", issuingKeypair.publicKey());

// 2. Distribution account trusts the issuer
const distAccount = await server.loadAccount(distributionKeypair.publicKey());
const trustTx = new TransactionBuilder(distAccount, {
  fee: BASE_FEE,
  networkPassphrase: Networks.TESTNET,
})
  .addOperation(Operation.changeTrust({ asset: myAsset }))
  .setTimeout(30)
  .build();

trustTx.sign(distributionKeypair);
await server.submitTransaction(trustTx);

// 3. Issuer sends tokens to distribution account
const issuerAccount = await server.loadAccount(issuingKeypair.publicKey());
const issueTx = new TransactionBuilder(issuerAccount, {
  fee: BASE_FEE,
  networkPassphrase: Networks.TESTNET,
})
  .addOperation(
    Operation.payment({
      destination: distributionKeypair.publicKey(),
      asset: myAsset,
      amount: "1000000",
    }),
  )
  .setTimeout(30)
  .build();

issueTx.sign(issuingKeypair);
await server.submitTransaction(issueTx);

// 4. (Optional) Lock the issuer account
const lockTx = new TransactionBuilder(issuerAccount, {
  fee: BASE_FEE,
  networkPassphrase: Networks.TESTNET,
})
  .addOperation(Operation.setOptions({ masterWeight: 0 }))
  .setTimeout(30)
  .build();

lockTx.sign(issuingKeypair);
await server.submitTransaction(lockTx);
```

## Trustlines

### Add a Trustline

```typescript
const account = await server.loadAccount(keypair.publicKey());
const tx = new TransactionBuilder(account, {
  fee: BASE_FEE,
  networkPassphrase: Networks.TESTNET,
})
  .addOperation(
    Operation.changeTrust({
      asset: new Asset("USD", "G_ISSUER..."),
      limit: "10000", // max amount to hold; omit for max
    }),
  )
  .setTimeout(30)
  .build();

tx.sign(keypair);
await server.submitTransaction(tx);
```

### Remove a Trustline

```typescript
// Balance must be 0 and no open offers for this asset
const tx = new TransactionBuilder(account, {
  fee: BASE_FEE,
  networkPassphrase: Networks.TESTNET,
})
  .addOperation(
    Operation.changeTrust({
      asset: new Asset("USD", "G_ISSUER..."),
      limit: "0",
    }),
  )
  .setTimeout(30)
  .build();
```

## DEX Trading

### Place a Sell Offer

```typescript
const tx = new TransactionBuilder(account, {
  fee: BASE_FEE,
  networkPassphrase: Networks.TESTNET,
})
  .addOperation(
    Operation.manageSellOffer({
      selling: Asset.native(),
      buying: new Asset("USD", "G_ISSUER..."),
      amount: "100", // selling 100 XLM
      price: "0.25", // 0.25 USD per XLM
      offerId: "0", // new offer
    }),
  )
  .setTimeout(30)
  .build();
```

### Cancel an Offer

```typescript
Operation.manageSellOffer({
  selling: Asset.native(),
  buying: new Asset("USD", "G_ISSUER..."),
  amount: "0", // amount 0 = delete
  price: "1", // price doesn't matter for deletion
  offerId: "12345", // existing offer ID
});
```

### Query the Orderbook

```typescript
const orderbook = await server
  .orderbook(Asset.native(), new Asset("USD", "G_ISSUER..."))
  .call();

console.log("Bids:", orderbook.bids);
console.log("Asks:", orderbook.asks);
```

## Streaming

### Stream Payments

```typescript
const close = server
  .payments()
  .forAccount("G...")
  .cursor("now")
  .stream({
    onmessage: (payment) => {
      if (payment.type === "payment") {
        console.log(
          `Received ${payment.amount} ${payment.asset_code || "XLM"}`,
        );
      }
    },
    onerror: (error) => {
      console.error("Stream error:", error);
    },
  });

// Later: close the stream
close();
```

### Stream Transactions

```typescript
const close = server
  .transactions()
  .forAccount("G...")
  .cursor("now")
  .stream({
    onmessage: (tx) => {
      console.log("New transaction:", tx.hash);
    },
  });
```

### Stream Orderbook

```typescript
const close = server
  .orderbook(Asset.native(), new Asset("USD", "G_ISSUER..."))
  .cursor("now")
  .stream({
    onmessage: (orderbook) => {
      console.log("Orderbook updated");
    },
  });
```

## Querying Horizon

### Paginated Results

```typescript
let page = await server
  .transactions()
  .forAccount("G...")
  .order("desc")
  .limit(25)
  .call();

while (page.records.length > 0) {
  page.records.forEach((tx) => console.log(tx.hash));
  page = await page.next();
}
```

### Following Links (HAL)

```typescript
const payments = await server.payments().limit(1).call();
// Follow the transaction link from a payment
const tx = await payments.records[0].transaction();
console.log(tx.hash);
```

### Decode XDR from Responses

```typescript
import { xdr } from "@stellar/stellar-sdk";

const txResponse = await server.transactions().limit(1).call();
const record = txResponse.records[0];

const envelope = xdr.TransactionEnvelope.fromXDR(record.envelope_xdr, "base64");
const result = xdr.TransactionResult.fromXDR(record.result_xdr, "base64");
const meta = xdr.TransactionMeta.fromXDR(record.result_meta_xdr, "base64");
```

## Fee-Bump Transactions

```typescript
import { TransactionBuilder, Keypair, Networks } from "@stellar/stellar-sdk";

// innerTx is a already-signed Transaction
const feeBumpTx = TransactionBuilder.buildFeeBumpTransaction(
  feePayerKeypair,
  "5000", // new higher fee in stroops
  innerTransaction,
  Networks.TESTNET,
);

feeBumpTx.sign(feePayerKeypair);
const result = await server.submitTransaction(feeBumpTx);
```

## Claimable Balances

### Create a Claimable Balance

```typescript
import { Claimant } from "@stellar/stellar-sdk";

const tx = new TransactionBuilder(sourceAccount, {
  fee: BASE_FEE,
  networkPassphrase: Networks.TESTNET,
})
  .addOperation(
    Operation.createClaimableBalance({
      asset: Asset.native(),
      amount: "100",
      claimants: [
        new Claimant(recipientPublicKey, Claimant.predicateUnconditional()),
        new Claimant(
          sourceKeypair.publicKey(),
          Claimant.predicateBeforeRelativeTime("86400"), // 24h reclaim window
        ),
      ],
    }),
  )
  .setTimeout(30)
  .build();
```

### Query and Claim

```typescript
const balances = await server
  .claimableBalances()
  .claimant(recipientPublicKey)
  .call();

for (const cb of balances.records) {
  const tx = new TransactionBuilder(recipientAccount, {
    fee: BASE_FEE,
    networkPassphrase: Networks.TESTNET,
  })
    .addOperation(Operation.claimClaimableBalance({ balanceId: cb.id }))
    .setTimeout(30)
    .build();

  tx.sign(recipientKeypair);
  await server.submitTransaction(tx);
}
```

## Sponsored Reserves

```typescript
// Sponsor creates a trustline for the sponsored account
const tx = new TransactionBuilder(sponsorAccount, {
  fee: BASE_FEE,
  networkPassphrase: Networks.TESTNET,
})
  .addOperation(
    Operation.beginSponsoringFutureReserves({
      sponsoredId: sponsoredPublicKey,
      source: sponsorKeypair.publicKey(),
    }),
  )
  .addOperation(
    Operation.changeTrust({
      asset: new Asset("USD", "G_ISSUER..."),
      source: sponsoredPublicKey,
    }),
  )
  .addOperation(
    Operation.endSponsoringFutureReserves({
      source: sponsoredPublicKey,
    }),
  )
  .setTimeout(30)
  .build();

// Both parties must sign
tx.sign(sponsorKeypair);
tx.sign(sponsoredKeypair);
await server.submitTransaction(tx);
```

## Multisig Transactions

### Set Up Multisig

```typescript
const tx = new TransactionBuilder(account, {
  fee: BASE_FEE,
  networkPassphrase: Networks.TESTNET,
})
  .addOperation(
    Operation.setOptions({
      signer: { ed25519PublicKey: signer2PublicKey, weight: 1 },
    }),
  )
  .addOperation(
    Operation.setOptions({
      masterWeight: 1,
      lowThreshold: 1,
      medThreshold: 2,
      highThreshold: 2,
    }),
  )
  .setTimeout(30)
  .build();

tx.sign(masterKeypair);
await server.submitTransaction(tx);
```

### Sign with Multiple Keys

```typescript
const tx = new TransactionBuilder(account, {
  fee: BASE_FEE,
  networkPassphrase: Networks.TESTNET,
})
  .addOperation(
    Operation.payment({
      destination: "G...",
      asset: Asset.native(),
      amount: "100",
    }),
  )
  .setTimeout(30)
  .build();

// Both signers sign
tx.sign(signer1Keypair);
tx.sign(signer2Keypair);

await server.submitTransaction(tx);
```

### XDR Round-Trip for Offline Signing

```typescript
// Signer 1 builds and partially signs
const xdrString = tx.toEnvelope().toXDR("base64");

// Send xdrString to signer 2 (e.g., via API or file)

// Signer 2 deserializes, signs, and submits
const tx2 = TransactionBuilder.fromXDR(xdrString, Networks.TESTNET);
tx2.sign(signer2Keypair);
await server.submitTransaction(tx2);
```

## RPC Interactions

### Simulate and Submit Smart Contract Transaction

```typescript
import {
  rpc,
  TransactionBuilder,
  Networks,
  Operation,
  xdr,
  Account,
} from "@stellar/stellar-sdk";

const rpcServer = new rpc.Server("https://soroban-testnet.stellar.org");

// Build the transaction
const account = await rpcServer.getAccount(publicKey);
const tx = new TransactionBuilder(account, {
  fee: BASE_FEE,
  networkPassphrase: Networks.TESTNET,
})
  .addOperation(/* your invokeHostFunction op */)
  .setTimeout(30)
  .build();

// Simulate to get resource requirements
const simResult = await rpcServer.simulateTransaction(tx);

// Prepare adds resources and fees from simulation
const preparedTx = await rpcServer.prepareTransaction(tx);

// Sign and send
preparedTx.sign(keypair);
const sendResult = await rpcServer.sendTransaction(preparedTx);
console.log("Hash:", sendResult.hash);

// Poll for completion
let getResult;
do {
  await new Promise((r) => setTimeout(r, 2000));
  getResult = await rpcServer.getTransaction(sendResult.hash);
} while (getResult.status === "NOT_FOUND");

console.log("Status:", getResult.status);
```

### Query Contract Events

```typescript
const events = await rpcServer.getEvents({
  startLedger: 100000,
  filters: [
    {
      type: "contract",
      contractIds: ["C..."],
      topics: [["AAAADwAAAAh0cmFuc2Zlcg==", "*", "*"]],
    },
  ],
  limit: 100,
});

events.events.forEach((event) => {
  console.log("Ledger:", event.ledger, "Value:", event.value);
});
```

## SEP-10 Authentication

```typescript
import { WebAuth, Keypair, Networks } from "@stellar/stellar-sdk";

// Server: build challenge
const challengeXDR = WebAuth.buildChallengeTx(
  serverKeypair,
  clientPublicKey,
  "yourdomain.com",
  300,
  Networks.TESTNET,
  "yourdomain.com",
);

// Client: read, sign, return
const { tx } = WebAuth.readChallengeTx(
  challengeXDR,
  serverKeypair.publicKey(),
  Networks.TESTNET,
  "yourdomain.com",
  "yourdomain.com",
);
tx.sign(clientKeypair);
const signedXDR = tx.toEnvelope().toXDR("base64");

// Server: verify
const signers = WebAuth.verifyChallengeTxSigners(
  signedXDR,
  serverKeypair.publicKey(),
  Networks.TESTNET,
  0,
  [clientPublicKey],
  "yourdomain.com",
  "yourdomain.com",
);
```

## Federation

```typescript
import { Federation } from "@stellar/stellar-sdk";

// Resolve a federated address
const result = await Federation.Server.resolve("alice*example.com");
console.log("Account:", result.account_id);
console.log("Memo:", result.memo);
```

## Error Handling

### Horizon Submission Errors

```typescript
try {
  const result = await server.submitTransaction(tx);
  console.log("Success:", result.hash);
} catch (error) {
  if (error.response?.data?.extras) {
    const extras = error.response.data.extras;
    console.error("Transaction failed");
    console.error("Result codes:", extras.result_codes);
    // e.g., { transaction: 'tx_failed', operations: ['op_underfunded'] }
    console.error("Envelope XDR:", extras.envelope_xdr);
  } else {
    console.error("Network error:", error.message);
  }
}
```

### Common Error Codes

| Transaction Code               | Meaning                       |
| ------------------------------ | ----------------------------- |
| `tx_failed`                    | One or more operations failed |
| `tx_too_early` / `tx_too_late` | Outside time bounds           |
| `tx_bad_seq`                   | Wrong sequence number         |
| `tx_bad_auth`                  | Not enough valid signatures   |
| `tx_insufficient_fee`          | Fee too low                   |
| `tx_no_source_account`         | Source account doesn't exist  |

| Operation Code      | Meaning                           |
| ------------------- | --------------------------------- |
| `op_underfunded`    | Not enough balance                |
| `op_no_trust`       | Missing trustline                 |
| `op_line_full`      | Trustline limit exceeded          |
| `op_low_reserve`    | Below minimum XLM reserve         |
| `op_no_destination` | Destination account doesn't exist |

### AccountRequiresMemoError

```typescript
import { AccountRequiresMemoError } from "@stellar/stellar-sdk";

try {
  await server.submitTransaction(tx);
} catch (e) {
  if (e instanceof AccountRequiresMemoError) {
    console.log("Destination requires a memo:", e.accountId);
  }
}
```
