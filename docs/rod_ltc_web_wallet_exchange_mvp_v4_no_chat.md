# ROD/LTC Non-Custodial Exchange MVP v4
## Lean plan: web-wallet-first, ROD DB order book, no chat/direct messages

Status: development planning draft  
Target pair: `ROD_LTC`  
Primary objective: prove one simple non-custodial all-or-nothing trade flow between ROD and Litecoin using the ROD web wallet as the wallet for both assets.

---

## 0. Core decision

This MVP should not use human chat, direct chat messages, or a private negotiation UI between counterparties.

The system should use:

```text
ROD on-chain DB
= order book, take records, trade state, trade history, current-price source

ROD/LTC web wallet
= local key ownership, local signing, transaction construction, broadcast

Democrit-JS
= local order validation, orderbook construction, trade state machine, HTLC flow

SpeXFeed/Nostr
= optional live event mirror after the ROD DB version works
```

For MVP v4, the simplest path is:

```text
No RODPay
No Gatevia
No CEX API
No central exchange server
No partial fills
No private chat
No direct counterparty messaging
No SpeXFeed dependency for settlement
```

SpeXFeed can be added later as a speed layer, but the MVP source of truth should be ROD DB.

---

## 1. Why this design is leaner

Earlier Democrit used public order broadcasts and direct private protocol messages. That makes sense for an online game marketplace, but for the first ROD/LTC exchange MVP it adds moving parts.

The new MVP removes direct off-chain negotiation.

Instead of:

```text
Maker publishes order
Taker sends direct take message
Seller sends data
Buyer sends PSBT
Seller returns PSBT
```

use:

```text
Maker writes order record to ROD DB
Taker writes take record to ROD DB
Maker funds HTLC and writes funding record
Taker funds counter-HTLC and writes funding record
Maker claims taker HTLC
Taker claims maker HTLC
Final trade history is written to ROD DB
```

No chat messages. No direct messages. Coordination is through signed on-chain records plus native chain transactions.

---

## 2. MVP scope

The MVP supports one pair only:

```text
ROD/LTC
```

The MVP supports one fill type only:

```text
all_or_nothing
```

The MVP supports two order sides:

```text
sell_rod_for_ltc
buy_rod_with_ltc
```

But implementation can start with one side first:

```text
Phase A: maker sells ROD for LTC
Phase B: maker buys ROD with LTC
```

Reason: both directions use the same HTLC pattern with the maker funding first, but they invert the chain of the first HTLC.

---

## 3. MVP components

### 3.1 ROD/LTC Web Wallet

The wallet must support both network profiles:

```text
ROD
LTC
```

Minimum wallet capabilities:

```text
Generate/import seed
Derive ROD address
Derive LTC address
Show ROD balance
Show LTC balance
List ROD UTXOs
List LTC UTXOs
Build raw transaction
Sign raw transaction locally
Broadcast raw transaction
Track tx confirmation
Create HTLC funding transaction
Create HTLC claim transaction
Create HTLC refund transaction
Decode/verify HTLC script
```

The private keys never leave the browser.

---

### 3.2 Chain API backends

Because the browser wallet does not run full nodes, it needs public or self-hosted APIs.

ROD backend must provide:

```text
get block height
get name record
list/query Democrit order records
get UTXOs for address
broadcast raw tx
get tx status
get fee estimate
write name_update / order record
```

LTC backend must provide:

```text
get block height
get UTXOs for address
broadcast raw tx
get tx status
get fee estimate
get raw tx / tx details
```

For early development, APIs can be centralized endpoints, but they must not custody funds. They are read/broadcast infrastructure only.

---

### 3.3 Democrit-JS

Democrit-JS is the port/adaptation of only the useful Democrit logic.

Port conceptually:

```text
Order model
Order validation
Orderbook construction
Bid/ask sorting
Order locking / take semantics
Trade states
Trade lifecycle
Timeout / abandoned logic
Failure detection
Success finalization
Trade history format
```

Do not port directly:

```text
XMPP MUC transport
Private XMPP messages
C++ RPC stubs
gloox/JID authentication
protobuf as required public format
same-chain game-asset PSBT assumptions
```

Replace with:

```text
JSON records
ROD DB coordination
ROD/LTC HTLC adapters
browser wallet signing
optional SpeXFeed mirror later
```

---

## 4. Democrit logic to preserve

### 4.1 Order model

Use a simplified Democrit order schema.

```json
{
  "type": "democrit.order.v1",
  "pair": "ROD_LTC",
  "side": "sell_rod_for_ltc",
  "maker": "maker.rod",
  "orderId": "000001",
  "amountRod": "10000.00000000",
  "amountLtc": "0.00010000",
  "priceLtcPerRod": "0.00000001",
  "fillPolicy": "all_or_nothing",
  "status": "open",
  "createdHeight": 123000,
  "expiresHeight": 124000,
  "makerRodRefundPubKey": "...",
  "makerLtcClaimPubKey": "...",
  "makerRodRefundAddress": "...",
  "makerLtcReceiveAddress": "...",
  "nonce": "...",
  "signature": "..."
}
```

For the opposite side:

```json
{
  "type": "democrit.order.v1",
  "pair": "ROD_LTC",
  "side": "buy_rod_with_ltc",
  "maker": "maker.rod",
  "orderId": "000002",
  "amountRod": "10000.00000000",
  "amountLtc": "0.00010000",
  "priceLtcPerRod": "0.00000001",
  "fillPolicy": "all_or_nothing",
  "status": "open",
  "createdHeight": 123000,
  "expiresHeight": 124000,
  "makerLtcRefundPubKey": "...",
  "makerRodClaimPubKey": "...",
  "makerLtcRefundAddress": "...",
  "makerRodReceiveAddress": "...",
  "nonce": "...",
  "signature": "..."
}
```

Important: the maker must sign the order. Wallets must verify the signature before displaying or accepting the order.

---

### 4.2 Orderbook construction

Orderbook is reconstructed locally from ROD DB records.

Filter active orders:

```text
type == democrit.order.v1
pair == ROD_LTC
status == open
expiresHeight > currentRodHeight
not already taken
not already filled
signature valid
amounts valid
price valid
```

Sort:

```text
asks = sell_rod_for_ltc, lowest price first
bids = buy_rod_with_ltc, highest price first
```

For ties:

```text
createdHeight
maker
orderId
```

Current price is derived locally:

```text
bestAsk = lowest sell_rod_for_ltc price
bestBid = highest buy_rod_with_ltc price
midPrice = (bestAsk + bestBid) / 2
lastPrice = latest successful trade price from ROD DB trade history
spread = bestAsk - bestBid
```

---

## 5. No-chat trade coordination

The key change in v4:

```text
No ProcessingMessage direct messaging.
No private counterparty chat.
No off-chain negotiation dependency.
```

The old Democrit `taking_order`, `seller_data`, and `psbt` messages become public/signed ROD DB records or native chain transactions.

Mapping:

```text
Old Democrit taking_order
→ ROD DB take record

Old Democrit seller_data
→ maker funding record / HTLC descriptor

Old Democrit psbt exchange
→ deterministic HTLC transactions built locally from public records

Old Democrit trade archive
→ ROD DB trade history record
```

---

## 6. Trade records

### 6.1 Take record

When a taker accepts an order, the taker writes a take record to ROD DB.

```json
{
  "type": "democrit.take.v1",
  "pair": "ROD_LTC",
  "orderRef": "dem/order/maker.rod/000001",
  "tradeId": "trade-abc123",
  "taker": "taker.rod",
  "amountRod": "10000.00000000",
  "amountLtc": "0.00010000",
  "takerRodClaimPubKey": "...",
  "takerLtcRefundPubKey": "...",
  "takerRodReceiveAddress": "...",
  "takerLtcRefundAddress": "...",
  "createdHeight": 123010,
  "expiresHeight": 123100,
  "nonce": "...",
  "signature": "..."
}
```

This is the global lock attempt.

For MVP, the first valid take record by block height / tx order wins. All other take records for the same order are ignored.

---

### 6.2 Maker funding record

After seeing the winning take record, the maker wallet creates the maker-side HTLC and writes a funding record.

If maker sells ROD:

```json
{
  "type": "democrit.htlc.maker_funded.v1",
  "tradeId": "trade-abc123",
  "maker": "maker.rod",
  "taker": "taker.rod",
  "fundingChain": "ROD",
  "fundingTxid": "...",
  "fundingVout": 0,
  "amountRod": "10000.00000000",
  "hashlock": "sha256(secret)",
  "claimPubKey": "takerRodClaimPubKey",
  "refundPubKey": "makerRodRefundPubKey",
  "refundHeight": 124000,
  "scriptHash": "...",
  "createdHeight": 123020,
  "signature": "..."
}
```

If maker buys ROD with LTC, the same record is used but `fundingChain` is `LTC` and `amountLtc` is set.

---

### 6.3 Taker funding record

After verifying maker's HTLC, the taker funds the counter-HTLC and writes:

```json
{
  "type": "democrit.htlc.taker_funded.v1",
  "tradeId": "trade-abc123",
  "maker": "maker.rod",
  "taker": "taker.rod",
  "fundingChain": "LTC",
  "fundingTxid": "...",
  "fundingVout": 1,
  "amountLtc": "0.00010000",
  "hashlock": "same-sha256-secret",
  "claimPubKey": "makerLtcClaimPubKey",
  "refundPubKey": "takerLtcRefundPubKey",
  "refundHeight": 123200,
  "scriptHash": "...",
  "createdHeight": 123030,
  "signature": "..."
}
```

If maker buys ROD, taker funding is on ROD.

---

### 6.4 Claim records

When maker claims taker's HTLC, the secret is revealed on-chain. The wallet can read it from the claim transaction, but the claiming wallet should also write a compact record:

```json
{
  "type": "democrit.claim.v1",
  "tradeId": "trade-abc123",
  "claimer": "maker.rod",
  "chain": "LTC",
  "claimTxid": "...",
  "revealedSecretHash": "sha256(secret)",
  "createdHeight": 123040,
  "signature": "..."
}
```

Then taker uses the revealed secret to claim maker's HTLC.

Final claim:

```json
{
  "type": "democrit.claim.v1",
  "tradeId": "trade-abc123",
  "claimer": "taker.rod",
  "chain": "ROD",
  "claimTxid": "...",
  "revealedSecretHash": "sha256(secret)",
  "createdHeight": 123050,
  "signature": "..."
}
```

---

### 6.5 Final trade record

Once both claim transactions confirm:

```json
{
  "type": "democrit.trade.v1",
  "pair": "ROD_LTC",
  "tradeId": "trade-abc123",
  "orderRef": "dem/order/maker.rod/000001",
  "maker": "maker.rod",
  "taker": "taker.rod",
  "side": "sell_rod_for_ltc",
  "amountRod": "10000.00000000",
  "amountLtc": "0.00010000",
  "priceLtcPerRod": "0.00000001",
  "rodTxid": "...",
  "ltcTxid": "...",
  "makerClaimTxid": "...",
  "takerClaimTxid": "...",
  "status": "success",
  "completedRodHeight": 123060,
  "completedLtcHeight": 4000000,
  "signature": "..."
}
```

This record powers:

```text
last price
public trade history
user trade history
basic volume stats
```

---

## 7. HTLC design

### 7.1 Maker sells ROD for LTC

Sequence:

```text
1. Maker creates order in ROD DB.
2. Taker writes take record with taker ROD claim pubkey and LTC refund pubkey.
3. Maker creates ROD HTLC:
   - taker can claim ROD with secret S
   - maker can refund ROD after long timeout
4. Maker writes maker_funded record.
5. Taker verifies ROD HTLC.
6. Taker creates LTC HTLC:
   - maker can claim LTC with same secret S
   - taker can refund LTC after shorter timeout
7. Taker writes taker_funded record.
8. Maker claims LTC and reveals S.
9. Taker extracts S and claims ROD.
10. Final trade record is written to ROD DB.
```

Timeout rule:

```text
Maker first-funded HTLC timeout must be longer.
Taker second-funded HTLC timeout must be shorter.
```

Example:

```text
ROD HTLC refund: current ROD height + 144 blocks
LTC HTLC refund: current LTC height + 24 blocks
```

The exact values must be tuned for block times and confirmation policy.

---

### 7.2 Maker buys ROD with LTC

Sequence:

```text
1. Maker creates buy_rod_with_ltc order.
2. Taker writes take record with taker LTC claim pubkey and ROD refund pubkey.
3. Maker creates LTC HTLC:
   - taker can claim LTC with secret S
   - maker can refund LTC after long timeout
4. Maker writes maker_funded record.
5. Taker verifies LTC HTLC.
6. Taker creates ROD HTLC:
   - maker can claim ROD with same secret S
   - taker can refund ROD after shorter timeout
7. Taker writes taker_funded record.
8. Maker claims ROD and reveals S.
9. Taker extracts S and claims LTC.
10. Final trade record is written to ROD DB.
```

---

## 8. Trade state machine

Use these states:

```text
OPEN
TAKEN
MAKER_FUNDED
TAKER_FUNDED
MAKER_CLAIMED
TAKER_CLAIMED
SUCCESS
REFUNDABLE
REFUNDED
FAILED
EXPIRED
```

State derivation should be deterministic from ROD DB records and chain state.

```text
OPEN
= valid order, no winning take

TAKEN
= valid winning take exists

MAKER_FUNDED
= valid maker HTLC exists and confirmed

TAKER_FUNDED
= valid taker HTLC exists and confirmed

MAKER_CLAIMED
= maker claim tx exists, secret visible

TAKER_CLAIMED
= taker claim tx exists

SUCCESS
= both assets claimed by correct parties

REFUNDABLE
= timeout passed before claim

REFUNDED
= refund tx confirmed

FAILED
= invalid tx, wrong script, wrong amount, double spend, or invalid secret

EXPIRED
= no valid take before order expiry
```

---

## 9. Local browser UI

The user opens:

```text
http://localhost or static web wallet page
```

Screens:

```text
1. Market
   - best bid
   - best ask
   - mid price
   - last price
   - spread
   - order book

2. Create Order
   - sell ROD for LTC
   - buy ROD with LTC
   - amount
   - price
   - expiry

3. My Orders
   - open
   - taken
   - funded
   - completed
   - expired
   - refunded

4. Trade History
   - public trades
   - my trades
   - txids
   - price
   - status

5. Wallet
   - ROD balance
   - LTC balance
   - claim/refund alerts
```

No chart in MVP.

No candle data.

No partial fill UI.

---

## 10. ROD DB namespace proposal

Use compact names.

Example namespace:

```text
dem/o/<maker>/<orderId>     active order
dem/t/<tradeId>             take + trade state
dem/h/<tradeId>             final history record
```

If ROD DB values are size-limited, keep records compact and use hashes for verbose data.

Example compact order value:

```json
{
  "v": 1,
  "type": "o",
  "pair": "ROD_LTC",
  "side": "sell",
  "rod": "10000.00000000",
  "ltc": "0.00010000",
  "p": "0.00000001",
  "fill": "aon",
  "exp": 124000,
  "sig": "..."
}
```

---

## 11. Where SpeXFeed fits later

SpeXFeed is not required for the lean no-chat MVP.

After ROD DB coordination works, SpeXFeed can mirror events:

```text
order created
order taken
maker funded
taker funded
claim seen
trade completed
refund available
```

Purpose:

```text
faster UI updates
notifications
multi-relay discovery
activity feed
mobile alerts
```

But settlement authority remains:

```text
ROD DB records
ROD chain transactions
LTC chain transactions
signatures
```

SpeXFeed should never be the only source of truth.

---

## 12. What must be developed first

### Phase 1: Wallet foundation

```text
ROD network profile
LTC network profile
UTXO lookup
raw tx builder
local signing
broadcast
confirmation watcher
```

Exit condition:

```text
Wallet can send/receive ROD and LTC independently.
```

---

### Phase 2: ROD DB order book

```text
create order record
cancel order record
read active orders
sort bids/asks
show current price
show trade history shell
```

Exit condition:

```text
Local UI displays a real ROD/LTC orderbook from ROD DB.
```

---

### Phase 3: HTLC primitives

Implement for both chains:

```text
create HTLC script
fund HTLC
verify HTLC
claim HTLC
refund HTLC
extract secret from claim tx
```

Exit condition:

```text
Manual test can lock, claim, and refund on testnet/regtest.
```

---

### Phase 4: No-chat order fill

```text
take record
maker_funded record
taker_funded record
claim records
state machine
refund handling
```

Exit condition:

```text
One complete sell_rod_for_ltc order can settle without any direct messages.
```

---

### Phase 5: Full MVP

```text
support buy_rod_with_ltc
public trade history
my trade history
claim/refund alerts
basic validation hardening
```

Exit condition:

```text
Users can create and fill all-or-nothing ROD/LTC orders from the web wallet.
```

---

## 13. Main engineering risks

### Risk 1: Generic static order cannot include taker claim address

Mitigation:

```text
Use ROD DB take record.
```

The take record provides taker-specific claim/refund keys before maker funds the HTLC.

---

### Risk 2: Maker must be online after an order is taken

Mitigation:

```text
MVP requires maker wallet open / watcher running.
Later add optional maker watcher mode.
```

No custody is introduced. The watcher only uses maker-owned keys.

---

### Risk 3: Chain API reliability

Mitigation:

```text
support multiple ROD/LTC backends
verify txs independently
never trust API for signing
```

---

### Risk 4: On-chain DB speed

Mitigation:

```text
acceptable for MVP
SpeXFeed mirror later
```

---

### Risk 5: HTLC script compatibility

Mitigation:

```text
first prove ROD HTLC and LTC HTLC on testnet/regtest
only then integrate orderbook
```

This is the hardest technical dependency.

---

## 14. Exact lean MVP definition

The MVP is complete when:

```text
1. User can open a web wallet.
2. User can hold ROD and LTC locally.
3. User can view ROD/LTC orderbook from ROD DB.
4. User can see best bid, best ask, mid, last price.
5. User can create one all-or-nothing order.
6. Another user can take the whole order.
7. The two wallets settle via HTLCs.
8. No chat/direct messages are exchanged.
9. Trade history appears in ROD DB.
10. Failed trades can be refunded.
```

---

## 15. Final architecture

```text
ROD DB
  ├── orders
  ├── take records
  ├── HTLC funding records
  ├── claim/refund records
  └── trade history

ROD/LTC Web Wallet
  ├── ROD wallet mode
  ├── LTC wallet mode
  ├── HTLC builder
  ├── local signer
  ├── chain watcher
  └── local browser UI

Democrit-JS
  ├── order validation
  ├── orderbook construction
  ├── current price derivation
  ├── trade state machine
  └── no-chat HTLC coordination

SpeXFeed/Nostr
  └── optional event mirror after MVP
```

This is the leanest version consistent with the new requirement:

```text
No chat. No direct messages. No custodian. No central exchange. Whole-order ROD/LTC trades only.
