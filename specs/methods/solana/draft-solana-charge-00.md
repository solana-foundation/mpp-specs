---
title: Solana Charge Intent for HTTP Payment Authentication
abbrev: Solana Charge
docname: draft-solana-charge-00
version: 00
category: info
ipr: trust200902
submissiontype: independent
consensus: false

author:
  - name: Ludo Galabru
    ins: L. Galabru
    email: ludo.galabru@solana.org
    org: Solana Foundation

  - name: Ilan Gitter
    ins: I. Gitter
    email: ilan.gitter@solana.org
    org: Solana Foundation

normative:
  RFC2119:
  RFC3339:
  RFC4648:
  RFC8174:
  RFC8259:
  RFC8785:
  RFC9457:
  I-D.payment-intent-charge:
    title: "'charge' Intent for HTTP Payment Authentication"
    target: https://datatracker.ietf.org/doc/draft-payment-intent-charge/
    author:
      - name: Jake Moxey
      - name: Brendan Ryan
      - name: Tom Meagher
    date: 2026
  I-D.httpauth-payment:
    title: "The 'Payment' HTTP Authentication Scheme"
    target: https://datatracker.ietf.org/doc/draft-ryan-httpauth-payment/
    author:
      - name: Jake Moxey
    date: 2026-01

informative:
  SOLANA-DOCS:
    title: "Solana Documentation"
    target: https://solana.com/docs
    author:
      - org: Solana Foundation
    date: 2026
  SPL-TOKEN:
    title: "SPL Token Program"
    target: https://solana.com/docs/tokens
    author:
      - org: Solana Foundation
    date: 2026
  SPL-TOKEN-2022:
    title: "SPL Token-2022 Program"
    target: https://solana.com/docs/tokens/extensions 
    author:
      - org: Solana Foundation
    date: 2026
  CONFIDENTIAL-TRANSFER:
    title: "Token-2022 Confidential Transfer Extension"
    target: https://www.solana-program.com/docs/confidential-balances
    author:
      - org: Solana Foundation
    date: 2026
  ZK-ELGAMAL-PROOF:
    title: "ZK ElGamal Proof Program"
    target: https://docs.anza.xyz/runtime/zk-elgamal-proof
    author:
      - org: Anza
    date: 2026
  BASE58:
    title: "Base58 Encoding Scheme"
    target: https://datatracker.ietf.org/doc/html/draft-msporny-base58-03
    author:
      - name: Manu Sporny
    date: 2023
---

--- abstract

This document defines the "charge" intent for the "solana" payment
method within the Payment HTTP Authentication Scheme
{{I-D.httpauth-payment}}. The client constructs and signs a native SOL
or SPL token transfer on the Solana blockchain; the server verifies the
payment and presents the transaction signature as proof of payment.

Two credential types are supported: `type="transaction"` (default),
where the client sends the signed transaction to the server for
broadcast, and `type="signature"` (fallback), where the client
broadcasts the transaction itself and presents the on-chain transaction
signature for server verification.

This document also defines an optional confidential charge profile, in
which the transferred amount is encrypted on-chain using the Token-2022
Confidential Transfer extension {{CONFIDENTIAL-TRANSFER}}. Because a
confidential transfer spans multiple transactions, this profile adds a
third credential type, `type="bundle"`, and requires the mint to
designate an auditor so the server can verify the paid amount without
the amount appearing in cleartext on-chain.

--- middle

# Introduction

HTTP Payment Authentication {{I-D.httpauth-payment}} defines a
challenge-response mechanism that gates access to resources behind
payments. This document registers the "charge" intent for the
"solana" payment method.

Solana is a high-throughput blockchain with sub-second finality
and low transaction fees {{SOLANA-DOCS}}. This specification
supports payments in both native SOL and SPL tokens (including
Token-2022 {{SPL-TOKEN-2022}}), making it suitable for
micropayment use cases where fast confirmation and low overhead
are important.

## Pull Mode (Default) {#pull-mode}

The default flow, called "pull mode", uses `type="transaction"`
credentials. The client signs the transaction and the server
"pulls" it for broadcast to the Solana network:

~~~
   Client                     Server              Solana Network
      |                          |                        |
      |  (1) GET /resource       |                        |
      |----------------------->  |                        |
      |                          |                        |
      |  (2) 402 Payment Required|                        |
      |      (recipient, amount, |                        |
      |       feePayerKey?)      |                        |
      |<-----------------------  |                        |
      |                          |                        |
      |  (3) Build tx, set fee   |                        |
      |      payer, sign         |                        |
      |                          |                        |
      |  (4) Authorization:      |                        |
      |      Payment <credential>|                        |
      |      (signed tx bytes)   |                        |
      |----------------------->  |                        |
      |                          |  (5) Co-sign (if fee   |
      |                          |      payer) + send     |
      |                          |----------------------> |
      |                          |  (6) Confirmation      |
      |                          |<---------------------- |
      |                          |                        |
      |  (7) 200 OK + Receipt    |                        |
      |<-----------------------  |                        |
      |                          |                        |
~~~

In this model the server controls transaction broadcast, enabling
fee sponsorship ({{fee-sponsorship}}) and server-side retry logic.
When `feePayer` is `true`, the challenge includes `feePayerKey`
so the client sets the server as fee payer. The server co-signs
with its fee payer key before broadcasting.

## Push Mode (Fallback) {#push-mode}

The fallback flow, called "push mode", uses `type="signature"`
credentials. The client "pushes" the transaction to the network
itself and presents the confirmed signature. The client
broadcasts the transaction itself and presents the confirmed
transaction signature:

~~~
   Client                     Server              Solana Network
      |                          |                        |
      |  (1) GET /resource       |                        |
      |----------------------->  |                        |
      |                          |                        |
      |  (2) 402 Payment Required|                        |
      |      (recipient, amount) |                        |
      |<-----------------------  |                        |
      |                          |                        |
      |  (3) Build & sign tx     |                        |
      |                          |                        |
      |  (4) Send transaction    |                        |
      |----------------------------------------------->   |
      |  (5) Confirmation        |                        |
      |<-----------------------------------------------   |
      |                          |                        |
      |  (6) Authorization:      |                        |
      |      Payment <credential>|                        |
      |      (tx signature)      |                        |
      |----------------------->  |                        |
      |                          |  (7) getTransaction    |
      |                          |----------------------> |
      |                          |  (8) Parsed tx data    |
      |                          |<---------------------- |
      |                          |                        |
      |  (9) 200 OK + Receipt    |                        |
      |<-----------------------  |                        |
      |                          |                        |
~~~

This flow is useful when the client cannot or does not wish to
delegate broadcast to the server. The server verifies the payment
by fetching and inspecting the on-chain transaction via RPC.

## Relationship to the Charge Intent

This document inherits the shared request semantics of the
"charge" intent from {{I-D.payment-intent-charge}}. It defines
only the Solana-specific `methodDetails`, `payload`, and
verification procedures for the "solana" payment method.

# Requirements Language

{::boilerplate bcp14-tagged}

# Terminology

Transaction Signature
: A base58-encoded {{BASE58}} unique identifier for a Solana
  transaction, produced by the first signer. Serves as both
  the transaction identifier and proof of payment in this
  specification.

SPL Token
: A fungible token on Solana conforming to the SPL Token
  program {{SPL-TOKEN}} or the Token-2022 program
  {{SPL-TOKEN-2022}}.

Associated Token Account (ATA)
: A deterministically derived token account for a given
  owner and mint, per the Associated Token Program. The
  address is a Program Derived Address (PDA) seeded by
  the owner's public key, the token mint, and the token
  program ID.

Lamports
: The smallest unit of native SOL. 1 SOL = 1,000,000,000
  lamports.

Base Units
: The smallest transferable unit of an SPL token, determined
  by the token's decimal precision. For example, USDC uses
  6 decimals, so 1 USDC = 1,000,000 base units.

Fee Payer
: An account that pays Solana transaction fees. When the server
  acts as fee payer, it adds its signature to the transaction
  before broadcasting, covering the transaction fee on behalf
  of the client.

Pull Mode
: The default settlement flow where the client signs the
  transaction and the server broadcasts it
  (`type="transaction"`). The server "pulls" the signed
  transaction from the credential. Enables fee sponsorship
  and server-side retry logic.

Push Mode
: The fallback settlement flow where the client broadcasts
  the transaction itself and presents the confirmed
  signature (`type="signature"`). The client "pushes" the
  transaction to the network directly. Cannot be used with
  fee sponsorship.

Confidential Transfer
: A Token-2022 transfer in which the transferred amount is
  encrypted on-chain using the Confidential Transfer
  extension {{CONFIDENTIAL-TRANSFER}}. The amount is hidden
  from public observers; only the sender, the receiver, and a
  designated auditor can recover it.

Confidential Token Account
: A Token-2022 associated token account that has been
  configured for confidential transfers via the
  `ConfigureAccount` operation, holding an encrypted
  available balance, an encrypted pending balance, and the
  owner's ElGamal public key. On mints that do not
  auto-approve, the account MUST also be approved by the
  mint's confidential-transfer authority before it can send
  or receive.

ElGamal Public Key
: A twisted-ElGamal public key bound to a confidential token
  account. Transfer amounts are encrypted under the sender's,
  the receiver's, and (when configured) the auditor's ElGamal
  public keys.

Auditor
: A party designated by the mint's `ConfidentialTransferMint`
  configuration whose ElGamal public key is included in every
  confidential transfer. The holder of the corresponding
  ElGamal secret key can decrypt transferred amounts. In this
  specification the verifying server acts as, or is delegated
  by, the auditor.

Pending Balance
: The portion of a confidential account's balance that has
  received incoming confidential transfers but is not yet
  spendable. The account owner converts pending balance into
  spendable available balance with the `ApplyPendingBalance`
  operation; no other party can perform this step.

Proof Context State Account
: A short-lived account into which the ZK ElGamal Proof
  Program {{ZK-ELGAMAL-PROOF}} records a verified proof so
  that a later instruction in the same or a subsequent
  transaction can reference it. Because confidential-transfer
  proofs are too large to share a single transaction with the
  transfer itself, they are verified into context state
  accounts first and closed afterward to reclaim rent.

# Intent Identifier

The intent identifier for this specification is "charge".
It MUST be lowercase.

# Intent: "charge"

The "charge" intent represents a one-time payment gating access
to a resource. The client builds and signs a Solana transfer
transaction, then either sends the signed transaction bytes to
the server for broadcast (`type="transaction"`) or broadcasts the
transaction itself and sends the on-chain signature
(`type="signature"`). The server verifies the transfer details
and returns a receipt.

# Encoding Conventions {#encoding}

All JSON {{RFC8259}} objects carried in auth-params or HTTP
headers in this specification MUST be serialized using the JSON
Canonicalization Scheme (JCS) {{RFC8785}} before encoding. JCS
produces a deterministic byte sequence, which is required for
any digest or signature operations defined by the base spec
{{I-D.httpauth-payment}}.

The resulting bytes MUST then be encoded using base64url
{{RFC4648}} Section 5 without padding characters (`=`).
Implementations MUST NOT append `=` padding when encoding,
and MUST accept input with or without padding when decoding.

This encoding convention applies to: the `request` auth-param
in `WWW-Authenticate`, the credential token in `Authorization`,
and the receipt token in `Payment-Receipt`.

# Request Schema

## Shared Fields

The `request` auth-param of the `WWW-Authenticate: Payment`
header contains a JCS-serialized, base64url-encoded JSON
object (see {{encoding}}). The following shared fields are
included in that object:

amount
: REQUIRED. The payment amount in base units, encoded as a
  decimal string. For native SOL, the amount is in lamports.
  For SPL tokens, the amount is in the token's smallest unit
  (e.g., for USDC with 6 decimals, "1000000" represents
  1 USDC). The value MUST be a positive integer that fits
  in a 64-bit unsigned integer (max 18,446,744,073,709,551,615).

currency
: REQUIRED. For native SOL, MUST be the lowercase
  string `"sol"`. For SPL tokens, MUST be the base58-encoded
  mint address (e.g.,
  `"EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"`
  for USDC). The mint address uniquely identifies the token
  and is used by the client to construct the transfer
  instruction. MUST NOT exceed 128 characters.

description
: OPTIONAL. A human-readable memo describing the resource or
  service being paid for. MUST NOT exceed 256 characters.

recipient
: REQUIRED. The base58-encoded public key of the account
  receiving the payment. For native SOL transfers, this is the
  destination account. For SPL token transfers, this is the
  owner of the destination associated token account, not the
  ATA address itself.

externalId
: OPTIONAL. Merchant's reference (e.g., order ID, invoice
  number), per {{I-D.payment-intent-charge}}. May be used
  for reconciliation or idempotency. MUST NOT exceed 566
  bytes (Solana Memo Program limit). When present, clients
  SHOULD include this value as a Memo Program instruction
  in the transaction, making it visible on-chain for
  auditing and reconciliation. Servers MAY verify the memo
  matches the `externalId` from the challenge.

## Method Details

The following fields are nested under `methodDetails` in
the request JSON:

network
: OPTIONAL. Identifies which Solana cluster the payment
  should be made on. MUST be one of "mainnet",
  "devnet", or "localnet". Defaults to "mainnet"
  if omitted. Clients MUST reject challenges whose
  network does not match their configured cluster.

decimals
: Conditionally REQUIRED. The number of decimal places
  for the token (0–9). MUST be present when `currency`
  is a mint address; MUST be absent when `currency` is
  `"sol"`. Used by the client to construct a
  `TransferChecked` instruction.

tokenProgram
: OPTIONAL. The base58-encoded program ID of the token
  program governing the token. MUST be either the
  Token Program
  (`TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA`) or
  the Token-2022 Program
  (`TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb`).
  If omitted, clients MUST determine the correct token
  program by fetching the mint account from the network
  and inspecting its owner program. If that lookup
  fails, returns an unexpected owner, or cannot be
  verified, clients MUST reject the challenge rather
  than falling back to the Token Program. Servers
  SHOULD include this field as a hint to avoid the
  extra RPC lookup. MUST NOT be present when
  `currency` is `"sol"`.

feePayer
: OPTIONAL. A boolean indicating whether the server will
  pay transaction fees on behalf of the client. Defaults
  to `false` if omitted. When `true`, the `feePayerKey`
  field MUST also be present. See {{fee-sponsorship}}.

feePayerKey
: Conditionally REQUIRED. The base58-encoded public key
  of the server's fee payer account. MUST be present when
  `feePayer` is `true`; MUST be absent when `feePayer` is
  `false` or omitted. The client uses this key as the
  transaction fee payer when constructing the transaction.

splits
: OPTIONAL. An array of at most 8 additional payment
  splits. Each entry is a JSON object with the following
  fields:

  - `recipient` (REQUIRED): Base58-encoded public key of
    the split recipient.
  - `amount` (REQUIRED): Amount in the same base units
    and asset as the primary `amount`.
  - `memo` (OPTIONAL): Human-readable label for this
    split (e.g., "platform fee", "referral"). MUST NOT
    exceed 566 bytes (Solana Memo Program limit).

  When present, the client MUST include a transfer
  instruction for each split in addition to the primary
  transfer to `recipient`. All splits use the same asset
  as the primary payment (native SOL or the token from `currency`).

  The top-level `amount` is the total the client pays.
  The sum of all split amounts MUST NOT exceed `amount`.
  The primary `recipient` receives `amount` minus the
  sum of all split amounts; this remainder MUST be
  greater than zero. Servers MUST reject challenges
  where splits consume the entire amount. Servers MUST
  verify each split transfer on-chain during credential
  verification. If the same recipient appears more than
  once in `splits`, each occurrence is a distinct
  payment leg and MUST be verified separately; servers
  MUST NOT implicitly aggregate such entries.

  This mechanism is a Solana-specific extension to the
  base `charge` intent. It can be used for fee payer cost
  recovery, platform fees, revenue sharing, or referral
  commissions.

recentBlockhash
: OPTIONAL. A base58-encoded recent blockhash for the
  client to use when constructing the transaction. When
  provided, clients SHOULD use this blockhash instead of
  fetching one from an RPC node. This avoids an extra
  RPC round-trip and ensures the server can verify
  blockhash freshness. This field is advisory and
  short-lived; it MUST NOT be assumed to remain valid for
  the full lifetime of the payment challenge. If omitted,
  clients MUST fetch a recent blockhash themselves.

confidential
: OPTIONAL. A boolean indicating that the charge MUST be
  settled as a Token-2022 confidential transfer
  ({{confidential}}), in which the transferred amount is
  encrypted on-chain. Defaults to `false` if omitted. When
  `true`: `currency` MUST be the mint address of a Token-2022
  mint whose `ConfidentialTransferMint` extension is enabled
  and configures an auditor; `tokenProgram`, if present, MUST
  be the Token-2022 Program; `auditorElgamalPubkey` MUST be
  present; the credential MUST use `type="bundle"`
  ({{bundle-payload}}); and `splits` MUST NOT be present.
  Servers MUST reject a confidential challenge that violates
  any of these constraints. MUST NOT be `true` when `currency`
  is `"sol"`.

auditorElgamalPubkey
: Conditionally REQUIRED. The base64-encoded
  ({{encoding}} notwithstanding, this value is raw base64 of
  the 32-byte key) twisted-ElGamal public key of the mint's
  confidential-transfer auditor. MUST be present when
  `confidential` is `true`; MUST be absent otherwise. Clients
  MUST verify this value matches the `auditorElgamalPubkey`
  recorded in the mint's `ConfidentialTransferMint` extension
  on-chain, and MUST reject the challenge if it does not match
  or if the mint configures no auditor. The client encrypts an
  auditor handle of the transferred amount under this key so
  the server (acting as, or delegated by, the auditor) can
  verify the amount during settlement.

recipientElgamalPubkey
: OPTIONAL. The base64-encoded twisted-ElGamal public key of
  the recipient's confidential token account, supplied as a
  hint to save an RPC lookup. When present, clients MUST
  verify it matches the recipient's on-chain confidential
  account state before use. Regardless of this hint, clients
  MUST confirm the recipient has a configured (and, on mints
  that do not auto-approve, approved) confidential token
  account; if it does not, the client MUST reject the
  challenge, because confidential transfers cannot be received
  by an unconfigured account ({{prerequisites}}). MUST be
  absent when `confidential` is `false` or omitted.

### Native SOL Example

~~~json
{
  "amount": "10000000",
  "currency": "sol",
  "recipient": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
  "description": "Weather API access",
  "methodDetails": {
    "network": "mainnet"
  }
}
~~~

This requests a transfer of 0.01 SOL (10,000,000 lamports).

### SPL Token Example

~~~json
{
  "amount": "1000000",
  "currency": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  "recipient": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
  "description": "Premium API call",
  "methodDetails": {
    "network": "mainnet",
    "decimals": 6,
    "tokenProgram": "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA"
  }
}
~~~

This requests a transfer of 1 USDC (1,000,000 base units).

### Fee Sponsorship Example

~~~json
{
  "amount": "10000000",
  "currency": "sol",
  "recipient": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
  "description": "Weather API access",
  "methodDetails": {
    "network": "mainnet",
    "feePayer": true,
    "feePayerKey": "9aE3Fg7HjKLmNpQr5TuVwXyZ2AbCdEf8GhIjKlMnOp1R"
  }
}
~~~

This requests a transfer of 0.01 SOL where the server pays
transaction fees.

### Payment Splits Example

~~~json
{
  "amount": "1050000",
  "currency": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  "recipient": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
  "description": "Marketplace purchase",
  "methodDetails": {
    "network": "mainnet",
    "decimals": 6,
    "splits": [
      { "recipient": "3pF8Kg2aHbNvJkLMwEqR7YtDxZ5sGhJn4UV6mWcXrT9A", "amount": "50000", "memo": "platform fee" }
    ]
  }
}
~~~

This requests a total payment of 1.05 USDC. The platform
receives 0.05 USDC and the primary recipient (seller)
receives 1.00 USDC.

### Confidential Transfer Example

~~~json
{
  "amount": "1000000",
  "currency": "HVWf8JmLoHs99Lw8Psf3fyqAtA4crWxCPkrmSdNjhNH3",
  "recipient": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
  "description": "Confidential API call",
  "methodDetails": {
    "network": "mainnet",
    "decimals": 6,
    "tokenProgram": "TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb",
    "feePayer": true,
    "feePayerKey": "9aE3Fg7HjKLmNpQr5TuVwXyZ2AbCdEf8GhIjKlMnOp1R",
    "confidential": true,
    "auditorElgamalPubkey": "GCJ+UreNo+YOlsWHCswYmm7+Phb90ionwJkBsIS4OUo="
  }
}
~~~

This requests a confidential transfer of 1 token (1,000,000
base units). The on-chain transfer encrypts the amount; the
server recovers and verifies it using the auditor ElGamal
secret corresponding to `auditorElgamalPubkey`. See
{{confidential}}.

# Credential Schema

The `Authorization` header carries a single base64url-encoded
JSON token (no auth-params). The decoded object contains the
following top-level fields:

challenge
: REQUIRED. An echo of the challenge auth-params from the
  `WWW-Authenticate` header: `id`, `realm`, `method`,
  `intent`, `request`, and (if present) `expires`. This
  binds the credential to the exact challenge that was
  issued.

source
: OPTIONAL. A payer identifier string, as defined by
  {{I-D.httpauth-payment}}. Solana implementations MAY
  use the payer's base58-encoded public key or a DID.

payload
: REQUIRED. A JSON object containing the Solana-specific
  credential fields. The `type` field determines which
  additional fields are present. Three payload types are
  defined: `"transaction"` (default) and `"signature"`
  (fallback) for ordinary transfers, and `"bundle"` for
  confidential transfers ({{bundle-payload}}).

## Transaction Payload — Pull Mode {#transaction-payload}

In pull mode (`type="transaction"`), the client sends the
signed transaction bytes to the server for broadcast. The
`transaction` field contains the base64-encoded serialized
signed transaction.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | REQUIRED | `"transaction"` |
| `transaction` | string | REQUIRED | Base64-encoded serialized signed transaction bytes (max 1232 bytes decoded) |

The transaction MUST be a valid Solana versioned transaction
that does not exceed the 1232-byte transaction size limit.
containing the transfer instruction(s) matching the challenge
parameters. The client MUST sign the transaction with the
transfer authority key. When `feePayer` is `false` or absent,
the client MUST also be the fee payer and the transaction MUST
be fully signed. When `feePayer` is `true`, the transaction
MUST set the server's `feePayerKey` as fee payer, and the
client signs only as transfer authority; the server adds the
fee payer signature before broadcasting (see
{{fee-sponsorship}}).

Example (decoded):

~~~json
{
  "challenge": {
    "id": "kM9xPqWvT2nJrHsY4aDfEb",
    "realm": "api.example.com",
    "method": "solana",
    "intent": "charge",
    "request": "eyJ...",
    "expires": "2026-03-15T12:05:00Z"
  },
  "payload": {
    "type": "transaction",
    "transaction": "AQAAAA...base64-encoded-signed-tx..."
  }
}
~~~

## Signature Payload — Push Mode {#signature-payload}

In push mode (`type="signature"`), the client has already
broadcast the transaction to the Solana network. The
`signature` field contains the base58-encoded transaction
signature for the server to verify on-chain.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | REQUIRED | `"signature"` |
| `signature` | string | REQUIRED | Base58-encoded Solana transaction signature |

Example (decoded):

~~~json
{
  "challenge": {
    "id": "kM9xPqWvT2nJrHsY4aDfEb",
    "realm": "api.example.com",
    "method": "solana",
    "intent": "charge",
    "request": "eyJ...",
    "expires": "2026-03-15T12:05:00Z"
  },
  "payload": {
    "type": "signature",
    "signature": "5UfDuX7hXbPjGUpTmt9PHRLsNGJe4dEny..."
  }
}
~~~

## Bundle Payload — Confidential Transfers {#bundle-payload}

When `methodDetails.confidential` is `true`, the credential
MUST use `type="bundle"`. A confidential transfer cannot be
expressed as a single transaction: its validity, equality, and
range proofs are too large to share a transaction with the
transfer instruction, so they are first verified into proof
context state accounts ({{confidential}}). The `transactions`
field carries the ordered list of signed transactions that
implement the transfer.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | REQUIRED | `"bundle"` |
| `transactions` | array | REQUIRED | Ordered, non-empty array of base64-encoded serialized signed transactions. Each element MUST individually satisfy the 1232-byte transaction size limit. |

The transactions MUST be ordered for sequential submission:
each proof context state account MUST be created and its proof
verified before the transaction that consumes it, and any
context-account close instructions MUST appear no earlier than
the transaction that last references the account. The final
transaction in the array MUST contain the confidential
`Transfer` (or `TransferWithFee`) instruction. The server
submits the transactions in array order, waiting for the
required commitment level on each before submitting the next
(see {{confidential-verification}} and {{bundle-settlement}}).

When `feePayer` is `true`, every transaction in the bundle
MUST set the server's `feePayerKey` as fee payer, and the
client signs each only as the transfer authority; the server
co-signs each transaction before broadcast. The rent for proof
context state accounts MUST be funded by the client, and each
close instruction MUST return the reclaimed lamports to the
client, so that fee sponsorship does not expose the server to
rent drain (see {{confidential-rent}}).

Example (decoded):

~~~json
{
  "challenge": {
    "id": "kM9xPqWvT2nJrHsY4aDfEb",
    "realm": "api.example.com",
    "method": "solana",
    "intent": "charge",
    "request": "eyJ...",
    "expires": "2026-03-15T12:05:00Z"
  },
  "payload": {
    "type": "bundle",
    "transactions": [
      "AQAAAA...proof-context-setup-tx...",
      "AQAAAA...confidential-transfer-tx..."
    ]
  }
}
~~~

## Limitations of Push Mode {#signature-limitations}

The `type="signature"` credential has the following limitations:

- MUST NOT be used when `feePayer` is `true` in the challenge
  request. Since the client has already broadcast the
  transaction, the server cannot add its fee payer signature.
  Servers MUST reject `type="signature"` credentials when
  the challenge specifies `feePayer: true`.

- The server cannot modify or enhance the transaction (e.g.,
  add priority fees, adjust compute units, or retry with
  different parameters).

# Fee Sponsorship {#fee-sponsorship}

When a challenge includes `feePayer: true` in `methodDetails`,
the server commits to paying Solana transaction fees on behalf
of the client. This section describes the fee sponsorship
mechanism.

## Server-Paid Fees

When `feePayer` is `true`:

1. **Client constructs transaction**: The client builds the
   transfer transaction with the server's `feePayerKey` set
   as the transaction fee payer. The client's account is the
   transfer authority but NOT the fee payer.

2. **Client partially signs**: The client signs the transaction
   with only its own key (the transfer authority). The fee
   payer signature slot remains empty.

3. **Client sends credential**: The client sends the partially
   signed transaction as a `type="transaction"` credential.

4. **Server adds fee payer signature**: The server verifies the
   transaction contents, then signs with the fee payer key to
   complete the transaction.

5. **Server broadcasts**: The fully signed transaction
   (containing both the client's transfer authority signature
   and the server's fee payer signature) is broadcast to the
   Solana network.

## Client-Paid Fees

When `feePayer` is `false` or omitted, the client MUST set
itself as the fee payer and fully sign the transaction. The
server broadcasts the transaction as-is without adding any
signatures.

## Server Requirements

When acting as fee payer, servers:

- MUST maintain sufficient SOL balance in the fee payer
  account to cover transaction fees
- MUST verify the transaction contents before signing
  (see {{transaction-verification}})
- SHOULD implement rate limiting to mitigate fee
  exhaustion attacks (see {{fee-payer-risks}})

## Client Requirements

- When `feePayer` is `true`: clients MUST set `feePayerKey`
  from `methodDetails` as the transaction fee payer and MUST
  sign only with the transfer authority key. Clients MUST
  use `type="transaction"` credentials.
- When `feePayer` is `false` or omitted: clients MUST set
  themselves as the fee payer and fully sign the transaction.
  Clients MAY use either `type="transaction"` or
  `type="signature"` credentials.

# Confidential Transfers {#confidential}

When a challenge sets `methodDetails.confidential` to `true`,
the charge MUST be settled as a Token-2022 Confidential
Transfer {{CONFIDENTIAL-TRANSFER}}. The transferred amount is
encrypted on-chain under twisted-ElGamal public keys and does
not appear in cleartext in the transaction. This profile
applies only to Token-2022 SPL tokens; it MUST NOT be used for
native SOL.

## Overview

A confidential transfer differs from an ordinary
`transferChecked` in three ways that this specification must
accommodate:

1. **The amount is encrypted.** The server cannot read the
   transferred amount from parsed transaction data. Instead, it
   recovers the amount from the auditor ciphertext handle using
   the auditor ElGamal secret key (see {{confidential-auditor}}).

2. **The transfer spans multiple transactions.** The transfer
   requires a ciphertext-validity proof, a
   ciphertext-ciphertext equality proof, and a range proof
   (and, on mints with a confidential transfer fee, additional
   fee proofs). These do not fit in a single transaction and
   are verified into proof context state accounts first. The
   credential therefore carries a transaction bundle
   ({{bundle-payload}}).

3. **Delivery is two-phase.** Funds received by a confidential
   transfer land in the recipient's pending balance and become
   spendable only after the recipient applies them. The receipt
   reflects this (see {{confidential-receipt}}).

## Prerequisites {#prerequisites}

A confidential charge can only succeed when all of the
following hold. Servers MUST NOT issue a confidential challenge,
and clients MUST reject one, unless they are satisfied:

- The mint identified by `currency` is owned by the Token-2022
  Program and has the `ConfidentialTransferMint` extension
  enabled.

- The mint configures an auditor ElGamal public key
  ({{confidential-auditor}}). A mint with no auditor MUST NOT be
  used for the charge intent, because the server would be unable
  to verify the paid amount.

- The sender (client) holds a configured confidential token
  account for the mint, with sufficient confidential available
  balance to cover the amount.

- The recipient holds a configured confidential token account
  for the mint. On a mint that does not auto-approve new
  accounts, that account MUST already be approved by the mint's
  confidential-transfer authority. The server or client cannot
  configure or approve the recipient's account on its behalf:
  the recipient owns the ElGamal secret, and approval is the
  mint authority's prerogative.

Account configuration (`ConfigureAccount`), approval
(`ApproveAccount`), deposit, and `ApplyPendingBalance` are
account-lifecycle operations outside the scope of the charge
intent. They are NOT carried in the charge credential and MUST
NOT appear in the bundle's transactions; the bundle is limited
to proof setup, the transfer, and proof-context cleanup
({{confidential-verification}}).

## Auditor Requirement {#confidential-auditor}

Every confidential transfer in this profile MUST encrypt an
auditor handle of the amount under the mint's auditor ElGamal
public key, and the grouped validity proof MUST bind the
sender, receiver, and auditor ciphertexts to the same amount.

The verifying server MUST be configured with the auditor
ElGamal secret key corresponding to the mint's
`auditorElgamalPubkey`, or MUST delegate amount verification to
a party that holds it. The server uses this key to decrypt the
auditor handle and confirm the transferred amount equals the
challenge `amount` ({{confidential-verification}}). Servers
MUST store the auditor secret with protection at least
equivalent to the fee payer signing key, and SHOULD isolate it
from the request-handling path (see {{confidential-key}}).

## Proof Context State Accounts

The client builds the bundle so that, for each required proof,
a proof context state account is created and the proof is
verified into it by the ZK ElGamal Proof Program
{{ZK-ELGAMAL-PROOF}} before the transfer instruction that
references it. After the transfer, the bundle MUST close those
context state accounts and return their rent to the client.

Clients SHOULD minimize the number of transactions by batching
proof verifications subject to the transaction size limit, but
MUST NOT exceed it. Servers MUST treat the bundle as opaque in
count but MUST verify its structure per
{{confidential-verification}}.

## Pending Balance and Delivery Semantics

A confidential transfer credits the recipient's pending
balance, not its available balance. The funds are delivered but
not yet spendable; only the recipient can convert them with
`ApplyPendingBalance`. A successful confidential charge
therefore attests that the encrypted amount was transferred to
the recipient's confidential account, not that it is
immediately spendable by the recipient. The receipt signals
this with `delivery: "pending"` ({{confidential-receipt}}).

## Restrictions

- `splits` MUST NOT be combined with `confidential`. Each
  confidential split would require its own proof set and
  transfer; this is out of scope for `draft-00`. Servers MUST
  reject a challenge that sets both.

- Confidential charges MUST use `type="bundle"`. `type="signature"`
  and a bare `type="transaction"` MUST NOT be used, and servers
  MUST reject them when `confidential` is `true`.

# Verification Procedure {#verification}

Upon receiving a request with a credential, the server MUST:

1. Decode the base64url credential and parse the JSON.

2. Verify that `payload.type` is present and is one of
   `"transaction"`, `"signature"`, or `"bundle"`. If the
   challenge sets `confidential: true`, the type MUST be
   `"bundle"`; otherwise it MUST be `"transaction"` or
   `"signature"`.

3. Look up the stored challenge using
   `credential.challenge.id`. If no matching challenge
   is found, reject the request.

4. Verify that all fields in `credential.challenge`
   exactly match the stored challenge auth-params.

5. If `payload.type` is `"signature"` and the challenge
   specifies `feePayer: true`, reject the request (see
   {{signature-limitations}}).

6. Proceed with type-specific verification:
   - For `type="transaction"`: see {{transaction-verification}}.
   - For `type="signature"`: see {{signature-verification}}.
   - For `type="bundle"`: see {{confidential-verification}}.

## Pull Mode Verification {#transaction-verification}

For credentials with `type="transaction"`:

1. Decode the base64 `payload.transaction` value.

2. Deserialize the transaction and verify that it
   structurally matches the challenge request:
   - the fee payer matches the challenge policy;
   - the transfer authority is signed by the client;
   - the transaction contains only expected transfer,
     ATA-creation, memo, and compute-budget instructions;
   - the payment semantics match the challenge request,
     as described in {{sol-verification}} or
     {{spl-verification}}.

3. If `feePayer` is `true`, add the server's fee payer
   signature using the `feePayerKey` and re-serialize.
   The transaction MUST have the server's `feePayerKey`
   set as the fee payer account.

4. If `feePayer` is `true`, simulate the transaction
   using the `simulateTransaction` RPC method. The
   server MUST reject the credential if simulation
   fails. If `feePayer` is `false` or omitted, the
   server SHOULD simulate the transaction before
   broadcast and SHOULD reject the credential if
   simulation indicates the transaction will fail.
   This catches invalid transactions without spending
   fees, which is especially important in fee payer
   mode (see {{fee-payer-risks}}).

5. Broadcast the transaction to the Solana network using
   `sendTransaction`.

6. Wait for confirmation at the required commitment level.

7. Fetch the confirmed transaction using `getTransaction`
   with `jsonParsed` encoding and verify the transfer
   details still match the challenge request, as described in
   {{sol-verification}} or {{spl-verification}}.

8. Record the transaction signature as consumed to
   prevent replay (see {{replay-protection}}).

9. Return the resource with a Payment-Receipt header.

## Push Mode Verification {#signature-verification}

For credentials with `type="signature"`:

1. Verify that `payload.signature` is present and is a
   valid base58-encoded string.

2. Verify the transaction signature has not been
   previously consumed (see {{replay-protection}}).

3. Fetch the transaction from the Solana network using
   the RPC `getTransaction` method with `jsonParsed`
   encoding and the `confirmed` commitment level.

4. Verify the transaction was successful (no error in
   the transaction metadata).

5. Verify the transfer details match the challenge
   request, as described in {{sol-verification}} or
   {{spl-verification}}.

6. Mark the transaction signature as consumed to
   prevent replay.

7. Return the resource with a Payment-Receipt header.

Note: both credential types reuse the same on-chain
transfer verification logic defined in
{{sol-verification}} and {{spl-verification}}.

## Native SOL Verification {#sol-verification}

For native SOL payments (`currency` is `"sol"`),
the server MUST:

1. Compute the primary payment amount as the top-level
   `amount` minus the sum of all `splits`, if any.

2. Locate a System Program `transfer` instruction in the
   transaction's parsed instructions whose `destination`
   matches the top-level `recipient` and whose `lamports`
   field matches that primary payment amount.

3. For each split in `splits`, if any, locate an
   additional System Program `transfer` instruction whose
   `destination` and `lamports` fields match that split.

   Each required payment leg MUST be matched to a
   distinct transfer instruction. A single transfer
   instruction MUST NOT satisfy more than one required
   payment leg, even if multiple legs share the same
   recipient.

If any required transfer instruction is missing, the
server MUST reject the credential.

## SPL Token Verification {#spl-verification}

For SPL token payments (`currency` is a mint address,
not `"sol"`), the server MUST:

1. Compute the primary payment amount as the top-level
   `amount` minus the sum of all `splits`, if any.

2. Locate a `transferChecked` instruction from the
   appropriate token program (Token Program or
   Token-2022) in the transaction's parsed instructions
   whose `mint` field matches the top-level `currency`
   field from the challenge request.

3. Derive the expected destination associated token
   account for the top-level `recipient` from the
   `recipient`, `currency`, and `tokenProgram` in the
   challenge request. Verify that at least one matching
   `transferChecked` instruction uses that derived ATA
   as `destination` and has `tokenAmount.amount` equal
   to the primary payment amount.

4. For each split in `splits`, if any, derive the
   expected destination ATA for that split recipient and
   verify that at least one additional `transferChecked`
   instruction uses that ATA as `destination` and has
   `tokenAmount.amount` equal to the split amount.

   Each required payment leg MUST be matched to a
   distinct `transferChecked` instruction. A single
   instruction MUST NOT satisfy more than one required
   payment leg, even if multiple legs resolve to the
   same destination ATA.

If any required `transferChecked` instruction is missing,
the server MUST reject the credential.

## Confidential Transfer Verification {#confidential-verification}

For credentials with `type="bundle"` (confidential charges),
the server MUST:

1. Decode each element of `payload.transactions` and
   deserialize it. Reject the credential if any element is not
   a valid transaction or exceeds the transaction size limit.

2. Verify the bundle contains only expected instructions:
   proof context state account creation, ZK ElGamal Proof
   Program {{ZK-ELGAMAL-PROOF}} verification instructions, the
   confidential `Transfer` or `TransferWithFee` instruction on
   the Token-2022 Program, context-account close instructions,
   and optionally memo and compute-budget instructions. Any
   other instruction — in particular `ConfigureAccount`,
   `ApproveAccount`, deposit, withdraw, or `ApplyPendingBalance`
   — MUST cause rejection ({{prerequisites}}).

3. Verify the confidential transfer instruction:
   - operates on the mint identified by `currency`;
   - uses, as its destination, the recipient's confidential
     token account derived from the top-level `recipient` and
     the mint;
   - references the proof context state accounts created
     earlier in the bundle;
   - includes an auditor ciphertext handle encrypted under the
     mint's auditor ElGamal public key, matching
     `auditorElgamalPubkey`.

4. If `feePayer` is `true`, verify every transaction sets the
   server's `feePayerKey` as fee payer and that proof-context
   rent is funded by, and returned to, the client
   ({{confidential-rent}}); then add the server's fee payer
   signature to each transaction.

5. If `feePayer` is `true`, simulate each transaction before
   broadcast and reject the credential on simulated failure.
   Otherwise the server SHOULD simulate before broadcast.

6. Submit the transactions in array order, waiting for at least
   the `confirmed` commitment level on each before submitting
   the next. If any transaction fails to land, the server MUST
   reject the credential and MUST NOT return a success receipt,
   even if earlier transactions in the bundle have landed.

7. Recover the transferred amount by decrypting the auditor
   ciphertext handle of the confirmed transfer using the
   auditor ElGamal secret key, and verify it equals the
   top-level `amount`. If the decrypted amount does not match,
   the server MUST reject the credential.

8. Record the signature of the final (transfer) transaction as
   consumed to prevent replay ({{replay-protection}}).

9. Return the resource with a Payment-Receipt header
   ({{confidential-receipt}}).

Because the proofs are verified on-chain by the ZK ElGamal
Proof Program, the server does not re-verify the
zero-knowledge proofs itself; it relies on the proofs having
been accepted by the program as a precondition for the transfer
instruction succeeding. The server's independent check is the
auditor decryption in step 7, which binds the on-chain
encrypted amount to the challenged `amount`.

## Replay Protection {#replay-protection}

Servers MUST maintain a set of consumed transaction
signatures. Before accepting a credential, the server
MUST check whether the signature has already been
consumed. After successful verification, the server
MUST atomically mark the signature as consumed.

The transaction signature is globally unique on the
Solana network, making it a natural replay prevention
token. A signature that has been consumed MUST NOT be
accepted again, even if presented with a different
challenge ID.

For `type="transaction"` credentials, the transaction
signature is derived after broadcast. For
`type="signature"` credentials, the signature is
provided directly by the client.

# Settlement Procedure

Two settlement flows are supported, corresponding to
the two credential types.

## Pull Mode Settlement (type="transaction")

For `type="transaction"` credentials, the client signs
the transaction and sends it to the server. The server
optionally adds a fee payer signature and broadcasts:

~~~
   Client                        Server                   Solana Network
      |                             |                           |
      |  (1) Authorization:         |                           |
      |      Payment <credential>   |                           |
      |      (signed tx bytes)      |                           |
      |-------------------------->  |                           |
      |                             |                           |
      |                             |  (2) If feePayer: true,   |
      |                             |      co-sign as fee payer |
      |                             |                           |
      |                             |  (3) simulateTransaction  |
      |                             |------------------------>  |
      |                             |  (4) Simulation OK        |
      |                             |<------------------------  |
      |                             |                           |
      |                             |  (5) sendTransaction      |
      |                             |------------------------>  |
      |                             |  (6) Confirmation         |
      |                             |<------------------------  |
      |                             |                           |
      |                             |  (7) getTransaction       |
      |                             |      (verify transfer)    |
      |                             |------------------------>  |
      |                             |  (8) Parsed tx data       |
      |                             |<------------------------  |
      |                             |                           |
      |  (9) 200 OK + Receipt       |                           |
      |<--------------------------  |                           |
      |                             |                           |
~~~

1. Client submits credential containing signed transaction
   bytes.
2. If `feePayer` is `true`, the server co-signs with its
   fee payer key.
3. Server simulates the transaction to catch failures
   without spending fees.
4. Server broadcasts the transaction to Solana.
5. Transaction reaches the required commitment level.
6. Server fetches the confirmed transaction and verifies
   the transfer details match the challenge request.
7. Server records the signature as consumed and returns
   the resource with a Payment-Receipt header whose
   `reference` field is the transaction signature.

## Push Mode Settlement (type="signature")

For `type="signature"` credentials, the client broadcasts
the transaction itself and presents the confirmed signature:

~~~
   Client                     Server              Solana Network
      |                          |                        |
      |  (1) Build & sign tx     |                        |
      |                          |                        |
      |  (2) sendTransaction     |                        |
      |----------------------------------------------->   |
      |                          |                        |
      |  (3) Poll confirmation   |                        |
      |----------------------------------------------->   |
      |  (4) Confirmed           |                        |
      |<-----------------------------------------------   |
      |                          |                        |
      |  (5) Authorization:      |                        |
      |      Payment <credential>|                        |
      |      (tx signature)      |                        |
      |----------------------->  |                        |
      |                          |  (6) getTransaction    |
      |                          |----------------------> |
      |                          |  (7) Verified          |
      |                          |<---------------------- |
      |                          |                        |
      |  (8) 200 OK + Receipt    |                        |
      |<-----------------------  |                        |
~~~

1. Client builds a transfer transaction and signs it.
2. Client sends the transaction to the Solana network.
3. Client polls for confirmation status.
4. Transaction reaches `confirmed` commitment level.
5. Client presents the transaction signature as the
   credential.
6. Server fetches the transaction via RPC and verifies
   transfer details.
7. Server confirms the payment matches the challenge.
8. Server returns the resource with a Payment-Receipt.

## Bundle Settlement (type="bundle") {#bundle-settlement}

For `type="bundle"` credentials (confidential charges), the
client submits an ordered set of signed transactions and the
server settles them sequentially:

~~~
   Client                        Server                   Solana Network
      |                             |                           |
      |  (1) Authorization:         |                           |
      |      Payment <credential>   |                           |
      |      (signed tx bundle)     |                           |
      |-------------------------->  |                           |
      |                             |                           |
      |                             |  (2) Verify bundle        |
      |                             |      structure            |
      |                             |                           |
      |                             |  (3) For each tx in order:|
      |                             |      co-sign (if fee      |
      |                             |      payer), simulate,    |
      |                             |      send, await confirm  |
      |                             |------------------------>  |
      |                             |<------------------------  |
      |                             |                           |
      |                             |  (4) getTransaction on    |
      |                             |      final transfer       |
      |                             |------------------------>  |
      |                             |<------------------------  |
      |                             |                           |
      |                             |  (5) Decrypt auditor      |
      |                             |      handle; verify       |
      |                             |      amount == challenge  |
      |                             |                           |
      |  (6) 200 OK + Receipt       |                           |
      |<--------------------------  |                           |
      |                             |                           |
~~~

1. Client submits the credential containing the ordered
   transaction bundle.
2. Server verifies the bundle structure
   ({{confidential-verification}}).
3. Server co-signs (when fee payer), simulates, broadcasts, and
   confirms each transaction in array order. If any transaction
   fails to land, settlement aborts and no success receipt is
   returned.
4. Server fetches the confirmed final transfer transaction.
5. Server decrypts the auditor handle and verifies the
   transferred amount equals the challenge `amount`.
6. Server records the final transfer signature as consumed and
   returns the resource with a Payment-Receipt header whose
   `reference` is that signature and whose `delivery` is
   `"pending"` ({{confidential-receipt}}).

## Client Transaction Construction

### Native SOL

The client MUST construct a transaction containing a
System Program `transfer` instruction with:

- `source`: the client's signing account
- `destination`: the `recipient` from the challenge
- `lamports`: the `amount` from the challenge

### SPL Tokens

The client MUST construct a transaction containing:

1. An idempotent Associated Token Account creation
   instruction for the recipient's ATA, ensuring
   payment succeeds even if the recipient has never
   held the token. The payer covers the rent-exempt
   minimum (~0.002 SOL) if the account does not exist.

2. A `transferChecked` instruction on the appropriate
   token program with:
   - `source`: the client's associated token account
   - `mint`: the `currency` field
   - `destination`: the recipient's derived ATA
   - `authority`: the client's signing account
   - `amount`: the `amount` from the challenge
   - `decimals`: the `decimals` from `methodDetails`

### Fee Payer Configuration

When `feePayer` is `true` in the challenge:

- The client MUST set the server's `feePayerKey` as the
  transaction fee payer.
- The client MUST sign the transaction only with its own
  key (transfer authority).
- The fee payer signature slot MUST be left empty for the
  server to fill.

When `feePayer` is `false` or absent:

- The client MUST set itself as the transaction fee payer.
- The client MUST fully sign the transaction.

Clients SHOULD set a compute unit limit and priority
fee appropriate for current network conditions.

## Confirmation Requirements

For `type="signature"` credentials, clients MUST wait for
at least the `confirmed` commitment level before presenting
the credential. Servers MUST fetch the transaction with at
least `confirmed` commitment. Servers MAY require
`finalized` commitment for high-value transactions.

For `type="transaction"` credentials, the server controls
the broadcast and confirmation process. Servers MUST wait
for at least `confirmed` commitment before returning the
receipt.

## Finality

Solana provides two commitment levels relevant to
payment verification:

- `confirmed`: optimistic confirmation from a
  supermajority of validators (~400ms). Sufficient
  for most payment use cases.
- `finalized`: deterministic finality after ~31 slots
  (~12 seconds). Required for high-value transactions
  where rollback risk is unacceptable.

In theory, a `confirmed` transaction could be rolled
back if validators shift consensus to a competing fork
that excludes the confirmed block. In practice, this
has never occurred on Solana mainnet. The `confirmed`
level is RECOMMENDED as the default for payment
verification to minimize latency.

## Receipt Generation

Upon successful verification, the server MUST include
a `Payment-Receipt` header in the 200 response.

The receipt payload for Solana charge:

| Field | Type | Description |
|-------|------|-------------|
| `method` | string | `"solana"` |
| `challengeId` | string | The challenge `id` from `WWW-Authenticate` |
| `reference` | string | The transaction signature (base58-encoded) |
| `status` | string | `"success"` |
| `timestamp` | string | {{RFC3339}} verification time |

Example (decoded):

~~~json
{
  "method": "solana",
  "challengeId": "kM9xPqWvT2nJrHsY4aDfEb",
  "reference": "5UfDuX7hXbPjGUpTmt9PHRLsNGJe4dEny...",
  "status": "success",
  "timestamp": "2026-03-10T21:00:00Z"
}
~~~

## Confidential Charge Receipt {#confidential-receipt}

For a confidential charge (`type="bundle"`), the `reference`
is the signature of the final confidential transfer
transaction, and the receipt includes one additional field:

| Field | Type | Description |
|-------|------|-------------|
| `delivery` | string | `"pending"` — the transferred amount was credited to the recipient's pending balance and becomes spendable only after the recipient applies it (see {{confidential}}). |

The receipt MUST NOT include the cleartext transferred amount;
the amount is encrypted on-chain, and the server learns it only
through the auditor key for verification purposes.

Example (decoded):

~~~json
{
  "method": "solana",
  "challengeId": "kM9xPqWvT2nJrHsY4aDfEb",
  "reference": "5UfDuX7hXbPjGUpTmt9PHRLsNGJe4dEny...",
  "status": "success",
  "delivery": "pending",
  "timestamp": "2026-03-10T21:00:00Z"
}
~~~

# Error Responses

When rejecting a credential, the server MUST return HTTP
402 (Payment Required) with a fresh
`WWW-Authenticate: Payment` challenge per
{{I-D.httpauth-payment}}. The server SHOULD include a
response body conforming to RFC 9457 {{RFC9457}} Problem
Details, with `Content-Type: application/problem+json`.
Servers MUST use the standard problem types defined in
{{I-D.httpauth-payment}}: `malformed-credential`,
`invalid-challenge`, and `verification-failed`. The
`detail` field SHOULD contain a human-readable
description of the specific failure (e.g., "Transaction
not found", "Amount mismatch", "Signature already
consumed").

All error responses MUST include a fresh challenge in
`WWW-Authenticate`.

Example error response body:

~~~json
{
  "type": "https://paymentauth.org/problems/verification-failed",
  "title": "Transfer Mismatch",
  "status": 402,
  "detail": "Destination token account does not belong to expected recipient"
}
~~~

# Security Considerations

## Transport Security

All communication MUST use TLS 1.2 or higher. Solana
credentials MUST only be transmitted over HTTPS
connections.

## Replay Protection Considerations

Servers MUST track consumed transaction signatures and
reject any signature that has already been accepted.
The check-and-consume operation MUST be atomic to
prevent race conditions where concurrent requests
present the same signature. Transaction signatures are
globally unique on the Solana network (derived from the
signer's key and the blockhash), making them natural
replay prevention tokens.

## Client-Side Verification

Clients MUST verify the challenge before signing:

1. `amount` is reasonable for the service
2. `currency` matches the expected asset
3. `recipient` is the expected party
4. If `currency` is a mint address, verify it is a known token
5. `splits`, if present, contain expected recipients
   and amounts — malicious servers could add splits
   to redirect funds
6. `feePayerKey`, if present, is the expected server
7. If `confidential` is `true`, `auditorElgamalPubkey`
   matches the mint's on-chain `ConfidentialTransferMint`
   auditor key, and the recipient has a configured (and,
   where required, approved) confidential token account

Malicious servers could request excessive amounts,
direct payments to unexpected recipients, or add
hidden splits. A malicious server could also supply an
attacker-controlled `auditorElgamalPubkey`; verifying it
against the on-chain mint configuration prevents the
amount from being encrypted to an unintended auditor.

## RPC Trust

The server relies on its Solana RPC endpoint to
provide accurate transaction data for on-chain
verification. A compromised RPC could return
fabricated transaction data, causing the server to
accept payments that were never made. Servers SHOULD
use trusted RPC providers or run their own nodes.

## Front-running (Push Mode)

In push mode, the client broadcasts the transaction
before presenting the credential, making it visible
on-chain. A party monitoring the chain could attempt
to present the same signature to the server. The
challenge binding (the credential echoes the challenge
`id`, which is HMAC-verified) and single-use signature
enforcement mitigate this: only the party that received
the challenge can construct a valid credential.

Push mode does not require the on-chain transaction to
carry a challenge-specific marker. It proves that a
payment matching the challenged terms was made, but not
necessarily that the payment was created for one unique
challenge instance. If multiple valid challenges have
identical terms, the same confirmed transaction could
satisfy any one of them, and the first accepted
presentation wins.

Requiring an on-chain marker such as a Memo carrying
the challenge `id` would provide stronger binding, but
would also reveal extra correlation metadata on chain.
This specification does not require such a marker in
the base flow, but implementations MAY define a
backward-compatible profile that does.

Pull mode is not susceptible to front-running because
the transaction is not broadcast until the server
receives and validates the credential.

## Fee Payer Risks {#fee-payer-risks}

Servers acting as fee payers accept financial risk in
exchange for providing a seamless payment experience.

Denial of Service via Bad Transactions
: Malicious clients could submit transactions that
  fail on-chain (insufficient balance, invalid
  instructions), causing the server to pay ~5,000
  lamports per failed transaction. Mitigations:

  - **Transaction simulation**: `simulateTransaction`
    catches most failures before broadcast, without
    spending fees. Servers MUST simulate fee-sponsored
    pull mode transactions before broadcasting.
    Servers SHOULD simulate non-fee-sponsored pull
    mode transactions before broadcasting.
  - **Rate limiting**: per client address, per IP, or
    per time window.
  - **Balance verification**: check the client's
    balance covers the transfer amount before signing.
  - **Client authentication**: require API keys or
    OAuth tokens before accepting fee-sponsored
    transactions.

ATA Rent Drain
: When the fee payer funds creation of an Associated
  Token Account (ATA), it pays ~0.002 SOL in rent.
  The recipient can close the ATA to reclaim rent,
  then the next payment re-creates it at the fee
  payer's expense. Servers SHOULD verify the
  recipient's ATA exists before co-signing, or factor
  rent cost into the payment amount via `splits`.

Fee Payer Balance Exhaustion
: Servers MUST monitor fee payer balance and reject
  new fee-sponsored requests when insufficient. The
  server SHOULD return a 402 with `feePayer: false`,
  allowing the client to pay its own fees as fallback.

## Transaction Payload Security

In pull mode, the server receives raw transaction
bytes from the client. A malicious client could craft
a transaction that transfers funds FROM the server's
fee payer account rather than simply paying fees.

Servers MUST verify that the transaction contains
only the expected instructions: transfer instruction(s)
matching the challenge parameters, ATA creation
(idempotent), and optionally compute budget
instructions. Any unexpected instructions MUST cause
rejection.

## Blockhash Freshness

When the server provides `recentBlockhash` in the
challenge, clients SHOULD verify it is plausible
(not obviously stale). A malicious server could
provide an expired blockhash, causing the client to
sign a transaction that will never land — wasting
the signing effort. However, since the transaction
is not broadcast by the client in pull mode, the
practical risk is limited to a failed payment
attempt that the client can retry.

## Auditor Key Custody (Confidential) {#confidential-key}

Confidential charge verification depends on the server holding
the auditor ElGamal secret key. This key can decrypt the
amount of every confidential transfer on the mint. It is a
high-value secret distinct from the fee payer signing key and
is not a key type natively supported by typical key-management
services. Servers MUST protect it with at least the same rigor
as the fee payer key, SHOULD isolate decryption behind a
narrow internal interface rather than exposing the raw secret
to request handlers, and SHOULD support rotation of the mint's
auditor key. Compromise of the auditor secret defeats the
confidentiality of all transfers on the mint, though it does
not by itself permit theft of funds.

## Proof Context Rent Drain (Confidential) {#confidential-rent}

Proof context state accounts are rent-bearing. If the fee payer
were also the rent payer, a malicious client could induce the
server to fund context accounts and then withhold the closing
transaction, draining the server's balance — analogous to the
ATA rent drain in {{fee-payer-risks}}. Therefore, when the
server is fee payer, the client MUST fund proof-context rent
and the bundle MUST close every context account back to the
client. Servers MUST verify this funding and close structure
before co-signing, and MUST reject bundles that fund
proof-context rent from the fee payer account.

## Partial Bundle Settlement (Confidential)

A confidential transfer is settled across multiple
transactions. If settlement aborts after some transactions have
landed, on-chain state may include created proof context
accounts without a completed transfer. The server MUST NOT
return a success receipt in this case. Because the close
instructions return rent to the client, the financial impact of
an aborted bundle falls on the client, not the server; clients
SHOULD be prepared to reclaim rent from any context accounts
left open by a failed attempt. Servers SHOULD bound the time
and number of transactions they will settle for a single
credential to limit resource exhaustion.

## Amount Privacy Scope (Confidential)

The confidential profile hides the transferred amount from
public on-chain observers. It does not hide the amount from the
challenge exchange itself: the `amount` field travels in the
challenge and credential over TLS, and the server learns the
amount both from the challenge and by decrypting the auditor
handle. Confidentiality is therefore relative to third-party
chain observers, not to the counterparties or the auditor.
Sender and recipient account identities, and the fact that a
confidential transfer occurred, remain visible on-chain.

# IANA Considerations

## Payment Method Registration

This document requests registration of the following
entry in the "HTTP Payment Methods" registry established
by {{I-D.httpauth-payment}}:

| Method Identifier | Description | Reference |
|-------------------|-------------|-----------|
| `solana` | Solana blockchain native SOL and SPL token transfer | This document |

## Payment Intent Registration

This document requests registration of the following
entry in the "HTTP Payment Intents" registry established
by {{I-D.httpauth-payment}}:

| Intent | Applicable Methods | Description | Reference |
|--------|-------------------|-------------|-----------|
| `charge` | `solana` | One-time SOL or SPL token transfer | This document |

--- back

# Examples

The following examples illustrate the complete HTTP exchange
for each flow. Base64url values are shown with their decoded
JSON below.

## Native SOL Charge (Pull Mode)

A 0.01 SOL charge for weather API access.

**1. Challenge (402 response):**

~~~http
HTTP/1.1 402 Payment Required
WWW-Authenticate: Payment id="kM9xPqWvT2nJrHsY4aDfEb",
  realm="api.example.com",
  method="solana",
  intent="charge",
  request="eyJhbW91bnQiOiIxMDAwMDAwMCIsImN1cnJlbmN5Ij
    oiU09MIiwiZGVzY3JpcHRpb24iOiJXZWF0aGVyIEFQSSBhY2
    Nlc3MiLCJtZXRob2REZXRhaWxzIjp7Im5ldHdvcmsiOiJtYWl
    ubmV0LWJldGEiLCJyZWZlcmVuY2UiOiJmNDdhYzEwYi01OGNj
    LTQzNzItYTU2Ny0wZTAyYjJjM2Q0NzkifSwicmVjaXBpZW50I
    joiN3hLWHRnMkNXODdkOTdUWEpTRHBiRDVqQmtoZVRxQTgzVF
    pSdUpvc2dBc1UifQ",
  expires="2026-03-15T12:05:00Z"
Cache-Control: no-store
~~~

Decoded `request`:

~~~json
{
  "amount": "10000000",
  "currency": "sol",
  "recipient": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
  "description": "Weather API access",
  "methodDetails": {
    "network": "mainnet"
  }
}
~~~

**2. Credential (retry with signed transaction):**

~~~http
GET /weather HTTP/1.1
Host: api.example.com
Authorization: Payment <base64url-encoded credential>
~~~

Decoded credential:

~~~json
{
  "challenge": {
    "id": "kM9xPqWvT2nJrHsY4aDfEb",
    "realm": "api.example.com",
    "method": "solana",
    "intent": "charge",
    "request": "<base64url-encoded request>",
    "expires": "2026-03-15T12:05:00Z"
  },
  "payload": {
    "type": "transaction",
    "transaction": "<base64-encoded signed transaction>"
  }
}
~~~

**3. Response (with receipt):**

~~~http
HTTP/1.1 200 OK
Payment-Receipt: <base64url-encoded receipt>
Content-Type: application/json

{"temperature": 72, "condition": "sunny"}
~~~

Decoded receipt:

~~~json
{
  "method": "solana",
  "challengeId": "kM9xPqWvT2nJrHsY4aDfEb",
  "reference": "4vJ9YFuPzUgdLkWYJf3KqfNM8cTnBp3jXx...",
  "status": "success",
  "timestamp": "2026-03-15T12:04:58Z"
}
~~~

## SPL Token (USDC) Charge with Fee Sponsorship

A 1 USDC charge where the server sponsors transaction fees
and includes a `recentBlockhash` to eliminate client RPC
dependency.

Decoded `request`:

~~~json
{
  "amount": "1000000",
  "currency": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  "recipient": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
  "description": "Premium API call",
  "methodDetails": {
    "network": "mainnet",
    "decimals": 6,
    "tokenProgram": "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA",
    "feePayer": true,
    "feePayerKey": "Gh9ZwEmdLJ8DscKNTkTqPbNwLNNBjuSzaG9Vp2KGtKJr",
    "recentBlockhash": "EkSnNWid2cvwEVnVx9aBqawnmiCNiDgp3gUdkDPTKN1N"
  }
}
~~~

The client uses `recentBlockhash` from the challenge (no RPC
call needed), sets `feePayerKey` as the transaction fee payer,
and partially signs with its own key only. The server
verifies the transaction contents, co-signs as fee payer,
and broadcasts.

Decoded credential:

~~~json
{
  "challenge": { "..." : "echoed challenge" },
  "payload": {
    "type": "transaction",
    "transaction": "<base64-encoded partially-signed tx>"
  }
}
~~~

## Push Mode (type="signature")

The client broadcasts the transaction itself and presents
the confirmed signature. Cannot be used with fee sponsorship.

Decoded credential:

~~~json
{
  "challenge": { "..." : "echoed challenge" },
  "payload": {
    "type": "signature",
    "signature": "4vJ9YFuPzUgdLkWYJf3KqfNM8cTnBp3jXx..."
  }
}
~~~

## Payment Splits

A marketplace charge of 1.05 USDC where 0.05 USDC goes to
the platform as a fee.

Decoded `request`:

~~~json
{
  "amount": "1050000",
  "currency": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  "recipient": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
  "description": "Marketplace purchase",
  "methodDetails": {
    "network": "mainnet",
    "decimals": 6,
    "splits": [
      {
        "recipient": "3pF8Kg2aHbNvJkLMwEqR7YtDxZ5sGhJn4UV6mWcXrT9A",
        "amount": "50000",
        "memo": "platform fee"
      }
    ]
  }
}
~~~

The client builds a transaction with two transfers: 1,000,000
base units to the primary recipient and 50,000 to the platform.
The total paid remains 1,050,000 base units, matching the
top-level `amount`.

## Confidential Charge (Bundle)

A 1-token confidential charge with server-sponsored fees. The
amount is encrypted on-chain; the server verifies it via the
auditor key.

Decoded `request`:

~~~json
{
  "amount": "1000000",
  "currency": "HVWf8JmLoHs99Lw8Psf3fyqAtA4crWxCPkrmSdNjhNH3",
  "recipient": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
  "description": "Confidential API call",
  "methodDetails": {
    "network": "mainnet",
    "decimals": 6,
    "tokenProgram": "TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb",
    "feePayer": true,
    "feePayerKey": "9aE3Fg7HjKLmNpQr5TuVwXyZ2AbCdEf8GhIjKlMnOp1R",
    "confidential": true,
    "auditorElgamalPubkey": "GCJ+UreNo+YOlsWHCswYmm7+Phb90ionwJkBsIS4OUo="
  }
}
~~~

The client confirms the recipient has an approved confidential
token account, verifies `auditorElgamalPubkey` against the
mint, builds the proof-context-setup and confidential-transfer
transactions (each with `feePayerKey` as fee payer and signed
only by the transfer authority), and presents them as a bundle.

Decoded credential:

~~~json
{
  "challenge": { "..." : "echoed challenge" },
  "payload": {
    "type": "bundle",
    "transactions": [
      "<base64-encoded proof-context-setup tx>",
      "<base64-encoded confidential-transfer tx>"
    ]
  }
}
~~~

Decoded receipt:

~~~json
{
  "method": "solana",
  "challengeId": "kM9xPqWvT2nJrHsY4aDfEb",
  "reference": "5UfDuX7hXbPjGUpTmt9PHRLsNGJe4dEny...",
  "status": "success",
  "delivery": "pending",
  "timestamp": "2026-03-15T12:04:58Z"
}
~~~

# Acknowledgements

The authors thank the Tempo team for their input on this 
specification.
