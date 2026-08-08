# Krypta Domain Model

## 1. Purpose

Krypta finds **conversion paths** that turn a user’s money from one form into another — typically fiat → crypto — while maximizing net proceeds and remaining **legally compliant under Belarusian law**.

This document defines the domain model that must cleanly support:

- Funding **sources** and **targets** (fiat or crypto; often the same catalog of assets)
- Simple exchange **services** (e.g. Whitebird): quoted rate + service fee + network fee
- Full **exchanges** (e.g. Bynex, Free2Ex): order books, maker/taker fees, spread, deposits/withdrawals, payment rails
- Generation of **actionable step-by-step plans** the user can follow to execute a conversion

No implementation details here — only the conceptual model.

---

## 2. Product Goals (domain-facing)

| Goal | Domain implication |
|------|--------------------|
| Best net return | Paths are scored by **effective output** after all fees, spreads, and slippage — not by headline rate |
| Legal compliance | Every path is filtered/annotated by a **compliance policy** before ranking |
| Comparable venues | Heterogeneous venues (services vs exchanges) share one **Quote / Leg / Path** abstraction |
| Executable guidance | A ranked path can be expanded into an ordered **Plan** of human-followable **Steps** |
| Source ↔ target symmetry | Assets and endpoints are not “deposit-only” or “withdraw-only” by type; direction is a property of the conversion request |

---

## 3. Core Mental Model

A conversion is a **path** through one or more **legs**. Each leg moves value between **endpoints** via a **venue**, under a **funding method**, producing a **quote** (or quote estimate) and eventually a **plan**.

```
Request (from Asset/Endpoint → to Asset/Endpoint, amount, constraints)
    │
    ▼
Candidate Paths  (sequence of Legs)
    │
    ▼
Compliance filter + cost model
    │
    ▼
Ranked Paths (by net yield / risk / complexity)
    │
    ▼
Selected Path → Plan (ordered Steps for the user)
```

**Key idea:** Whitebird and Bynex look different operationally, but both are venues that can answer: *“If I put A in, what do I get out, under which fees and constraints, and what must the user do?”*

---

## 4. Ubiquitous Language

| Term | Meaning |
|------|---------|
| **Asset** | A fungible value unit: fiat currency or crypto coin/token (optionally network-specific) |
| **Endpoint** | Where value sits or should sit for the user (wallet, bank account, exchange balance, cash, etc.) |
| **Venue** | An organization or service that can convert, hold, deposit, or withdraw assets (exchange or exchange service) |
| **Instrument** | A tradable pair or rate product on a venue (e.g. `USDT/BYN` spot market, or Whitebird `BTC→BYN` quote product) |
| **Funding method** | How money enters or leaves a venue (card, bank transfer, cash desk, on-chain deposit, internal transfer, etc.) |
| **Fee** | A cost component with a known structure (fixed, percent, tiered, side-dependent, etc.) |
| **Liquidity source** | How a venue prices a trade (flat quote vs order book vs RFQ) |
| **Leg** | One atomic conversion or transfer segment in a path |
| **Path** | Ordered sequence of legs from source to target |
| **Quote** | Snapshot of expected economics for a leg or path at a point in time |
| **Plan** | User-facing, ordered steps derived from a path |
| **Compliance constraint** | Rule that may forbid, warn, or require documentation for a path/leg |

---

## 5. Asset Model

### 5.1 Asset

An **Asset** is identified independently of where it is held.

```
Asset
  id
  kind                 // FIAT | CRYPTO
  symbol               // BYN, USD, EUR, BTC, USDT, …
  displayName
  decimals
  networks[]           // for crypto: BTC, TRC20, ERC20, … (empty/irrelevant for pure fiat)
  legalClassification  // jurisdiction-specific tags used by compliance
```

Notes:

- **USDT on TRC20** and **USDT on ERC20** are the same *economic asset family* but different **transfer rails**. The model should treat “asset + network” as the unit of transfer feasibility, while allowing UI grouping by symbol.
- Fiat assets may still have **rails** (SEPA, local bank wire, cash) — modeled as funding methods, not networks.

### 5.2 AssetInstance (optional refinement)

When precision matters for routing:

```
AssetInstance
  asset
  network?             // required for on-chain crypto movement
```

Sources and targets in a request refer to `AssetInstance` (or Asset + optional network).

---

## 6. Endpoints: Sources and Targets

Sources and targets are the **same conceptual list**: places value can come from or go to.

```
Endpoint
  id
  kind                 // USER_WALLET | BANK_ACCOUNT | CASH | VENUE_BALANCE | EXTERNAL_SERVICE | …
  assetCapabilities[]  // which AssetInstances this endpoint can hold/send/receive
  ownerScope           // USER | VENUE | THIRD_PARTY
  venue?               // set when the endpoint is a balance on an exchange/service
  metadata             // IBAN mask, wallet address book ref, etc. (PII kept out of core pricing)
```

### 6.1 Source / Target roles

A **ConversionRequest** assigns roles; it does not create separate type hierarchies:

```
ConversionRequest
  sourceEndpoint       // or: implied “user holds asset X somehow”
  sourceAsset          // AssetInstance
  targetEndpoint
  targetAsset          // AssetInstance
  amount
  amountSide           // SOURCE (spend exactly) | TARGET (receive exactly)
  preferences          // max steps, allowed venues, urgency, self-custody required, …
  complianceProfile    // user residency, KYC level, allowed product set
```

Why the same list for source and target:

- “I have BYN in my bank” and “I want BYN in my bank” are the same endpoint kind used in opposite directions.
- “I have USDT on Bynex” can be a **source** (sell) or **target** (buy then leave on exchange).
- Crypto self-custody wallets and venue balances are both endpoints; venues are not a special case outside this model.

### 6.2 Endpoint vs Venue balance

A venue balance is an endpoint *hosted by* a venue. Depositing BYN to Bynex is a **transfer leg** into `Endpoint(VENUE_BALANCE, venue=Bynex, asset=BYN)`, not a conversion by itself.

---

## 7. Venue Taxonomy

All conversion providers are **Venues**. Capabilities differ.

```
Venue
  id
  name                 // Whitebird, Bynex, Free2Ex, …
  kind                 // EXCHANGE_SERVICE | EXCHANGE | BROKER | P2P_PLATFORM | …
  jurisdictions[]
  capabilities
  instruments[]
  fundingMethods[]
  feeSchedule
  liquidityModel       // QUOTED_RATE | ORDER_BOOK | RFQ | HYBRID
  kycRequirements
  status               // ACTIVE | DEGRADED | DISABLED
```

### 7.1 Exchange service (e.g. Whitebird)

Characteristics:

- User typically does **not** trade an order book
- Venue publishes (or returns via API) a **quoted rate** for an asset pair
- Costs collapse to a small set: **service fee**, **network/withdrawal fee**, sometimes a payment-method surcharge
- Operational flow is usually: create order → pay via funding method → receive crypto/fiat

Native modeling:

```
QuotedRateInstrument
  venue
  fromAsset
  toAsset
  rateSource           // API mid/offer, scraped page, manual ops feed
  feeComponents[]      // SERVICE, NETWORK, PAYMENT_SURCHARGE, …
  minAmount / maxAmount
  eta
  settlementStyle      // INSTANT | DELAYED | MANUAL_REVIEW
```

### 7.2 Exchange (e.g. Bynex, Free2Ex)

Characteristics:

- Spot (or other) markets with **bids/asks**
- Fees depend on **maker vs taker**, VIP tier, and sometimes payment rail
- User may **take liquidity** (market / best ask) or **make liquidity** (limit at a reasonable price)
- Moving value in/out has **deposit** and **withdrawal** fees and constraints
- Path economics must include **spread** and **slippage**, not only fee %

Native modeling:

```
SpotMarket
  venue
  baseAsset
  quoteAsset
  orderBookDepth       // live or cached
  feePolicy            // maker/taker, tiers
  tickSize / lotSize
  tradingRules
```

An exchange conversion leg is usually *not* one atomic “BYN → BTC” magic step. It is often a **bundle** of sub-operations (see §10).

### 7.3 Capability flags

Rather than hard-coding venue kinds everywhere, expose capabilities:

```
VenueCapabilities
  canQuoteExactPair(from, to)
  hasOrderBook(pair)
  supportsFunding(method, asset, direction)
  supportsWithdrawal(asset, network)
  supportsInternalTransfer
  requiresKycFor(amount, assets)
```

---

## 8. Funding Methods (payment & settlement rails)

Exchanges and services accept value in different ways with different fees. Funding methods are first-class.

```
FundingMethod
  id
  venue
  direction            // INBOUND | OUTBOUND | BOTH
  rail                 // BANK_TRANSFER | CARD | ERIP | CASH | ON_CHAIN | INTERNAL | …
  asset                // AssetInstance supported on this rail
  feePolicy
  limits
  settlementTime
  userRequirements[]   // verified card, same-name bank account, …
  stepTemplateRef      // how to explain this rail in a Plan
```

Examples:

| Venue | Method | Direction | Typical cost impact |
|-------|--------|-----------|---------------------|
| Whitebird | Bank transfer / card (as offered) | IN | payment surcharge or included in service fee |
| Whitebird | On-chain delivery | OUT | network fee |
| Bynex | BYN bank deposit | IN | deposit fee or free + hold time |
| Bynex | Spot trade USDT/BYN | n/a (trade) | maker/taker |
| Bynex | USDT TRC20 withdraw | OUT | withdrawal fee |
| Free2Ex | Card top-up | IN | percent fee |

Funding methods attach to **transfer legs**, while trading fees attach to **trade legs**. Keeping them separate avoids mixing “how I pay the venue” with “how the venue prices the pair.”

---

## 9. Fee Model

Fees must be composable. A path’s cost is the sum (and interaction) of fee components across legs — never a single opaque percentage.

### 9.1 FeeComponent

```
FeeComponent
  id
  name                 // "taker fee", "TRC20 withdrawal", "card inbound"
  appliesTo            // TRADE | DEPOSIT | WITHDRAWAL | SERVICE | NETWORK | FX_MARGIN | …
  structure            // FIXED | PERCENT | PERCENT_PLUS_FIXED | TIERED | BALANCE_DEPENDENT
  assetOfFee           // which asset the fee is charged in
  params               // e.g. 0.2%, 1 USDT, tiers[]
  sideConditions       // MAKER | TAKER | ANY
  fundingMethod?       // when fee depends on rail
  thresholdRules[]     // min fee, free above volume, etc.
```

### 9.2 Effective pricing inputs

For ranking, each leg produces an **economic projection**:

```
LegEconomics
  grossInput
  grossOutput
  feeBreakdown[]       // concrete FeeApplication rows
  spreadCost           // vs mid, if order-book based
  slippageEstimate     // for size-taking on book
  netOutput
  confidence           // HARD_QUOTE | ESTIMATE | STALE
  validUntil
```

### 9.3 Spread and order choice (exchanges)

For order-book venues, the model distinguishes **execution strategies**:

```
ExecutionStrategy
  kind                 // TAKE_BEST | LIMIT_AT | LIMIT_NEAR_MID | TWAP_HINT | MANUAL
  limitPrice?
  maxslippageBps?
  assumedFillModel     // FULL_IMMEDIATE | PARTIAL_OK | DEPTH_WALK
```

Implications:

- **TAKE_BEST**: walk asks/bids; include taker fee + realized spread/slippage for the size
- **LIMIT_AT / LIMIT_NEAR_MID**: better fee or price possible, but quote becomes **probabilistic** (may not fill). Path ranking must surface fill risk, not pretend it is a firm quote
- Maker/taker is not a global venue constant — it is a property of the chosen execution strategy (and sometimes of whether the order rests or crosses)

### 9.4 Deposit & withdrawal

```
CustodyMovement
  venue
  asset
  direction            // DEPOSIT | WITHDRAWAL
  networkOrRail
  fee
  minConfirmations? / settlementDelay
  availability         // when funds become tradable or spendable
```

A “buy BTC on Bynex with bank BYN” path must account for: deposit → (optional wait) → trade → withdraw, each with its own fee and delay.

---

## 10. Liquidity & Quoting

### 10.1 LiquidityModel

```
LiquidityModel
  kind                 // QUOTED_RATE | ORDER_BOOK | RFQ
```

**QUOTED_RATE** (Whitebird-like):

- `Quote = f(amount, direction, fundingMethod?)`
- Firmness often high for a short TTL
- Spread is usually **embedded** in the quoted rate (and/or service margin), not computed from a book

**ORDER_BOOK** (exchange-like):

- `Quote = simulate(orderBook, amount, executionStrategy, feePolicy)`
- Must expose mid, best bid/ask, depth consumed, slippage
- Optional: suggest a **reasonable limit price** (e.g. mid − X bps) as an alternative quote lane

**RFQ**:

- Request a dealer quote; treat response like QUOTED_RATE with explicit counterparty TTL

### 10.2 Quote

```
Quote
  id
  subject              // Leg or Path
  inputAmount / outputAmount
  economics            // LegEconomics / aggregated
  executionStrategy?
  assumptions[]        // "taker", "depth as of T", "KYC tier 1"
  sourcedAt
  ttl
```

Path quotes aggregate leg quotes carefully (output of leg N is input of leg N+1), carrying forward confidence and TTL as the minimum along the path.

---

## 11. Legs, Paths, and Graphs

### 11.1 Leg kinds

Unified leg taxonomy:

| Leg kind | What it does | Typical venue |
|----------|--------------|---------------|
| `TRANSFER_IN` | User → venue balance via funding method | any |
| `TRADE` | Exchange asset A for B on venue | EXCHANGE |
| `SERVICE_CONVERT` | Service quotes and settles A→B | EXCHANGE_SERVICE |
| `TRANSFER_OUT` | Venue → user endpoint | any |
| `NETWORK_TRANSFER` | On-chain or inter-wallet move between endpoints | wallets / multi-venue |
| `INTERNAL_SWAP` | Optional venue-native convert product | some exchanges |

```
Leg
  id
  kind
  venue?
  fromEndpoint
  toEndpoint
  fromAsset
  toAsset
  fundingMethod?
  instrument?          // SpotMarket or QuotedRateInstrument
  executionStrategy?
  constraints
```

### 11.2 Path

```
Path
  id
  requestRef
  legs[]
  aggregateQuote
  complianceResult
  complexityScore      // number of steps, accounts needed, manual ops
  riskFlags[]          // FILL_UNCERTAINTY, STALE_BOOK, MANUAL_SETTLEMENT, …
```

### 11.3 Why paths, not single hops only

Best return may require multi-hop routes, for example:

1. Bank BYN → Whitebird → USDT (TRC20) to self-custody  
2. Bank BYN → Bynex deposit → buy USDT → withdraw TRC20  
3. Bank BYN → Bynex → buy BTC → withdraw on-chain  
4. (Future) BYN → Venue A → USDT → Venue B → rarer token  

The router explores a **conversion graph**:

- Nodes: `(Endpoint capability × AssetInstance)` or normalized `(VenueBalance|External) × AssetInstance`
- Edges: possible legs with fee/liquidity adapters

Legal constraints prune edges before cost optimization (or score them as infeasible).

---

## 12. From Path to Plan (user steps)

Economics alone are not enough: users need **easy steps to follow**.

### 12.1 Plan

```
Plan
  path
  steps[]
  prerequisites[]      // KYC on venue, install app, have IBAN, gas dust, …
  totalEstimatedTime
  warnings[]
```

### 12.2 Step

```
Step
  ordinal
  title                // "Deposit BYN to Bynex by bank transfer"
  kind                 // ACTION | WAIT | VERIFY | DECIDE
  actor                // USER | SYSTEM | VENUE
  relatedLeg
  instructions         // structured rich text / checklist
  parameters           // amount, address, order price, payment refs
  expectedResult       // "USDT balance ≥ X"
  feeReminder?
  complianceNotes?
  uiHints              // deep links, copy-to-clipboard fields, QR
```

### 12.3 Step generation rules

- Each leg compiles to one or more steps via **templates** bound to venue + funding method + leg kind
- Exchange trade legs may expand to: open market → place order (market/limit) → wait for fill → confirm balance
- When execution strategy is `LIMIT_*`, insert a **DECIDE** step: accept suggested price or edit, with clear impact on fill probability and net output
- Parallelism is rare; default is strict sequence with explicit WAIT steps for deposits/confirmations

This keeps Whitebird “create order → pay → receive” and Bynex “deposit → trade → withdraw” in the same planning machinery.

---

## 13. Compliance (Belarus-focused constraint layer)

Compliance is not bolted on after routing; it is a **filter/annotator** on legs and paths.

```
CompliancePolicy
  jurisdiction         // BY default for Krypta v1
  rules[]

ComplianceRule
  id
  appliesWhen          // assets, venues, amounts, user profile, funding rails
  effect               // FORBID | REQUIRE_KYC | REQUIRE_DISCLOSURE | WARN | LIMIT_AMOUNT
  explanation          // user-visible reason
```

```
ComplianceResult
  status               // ALLOWED | ALLOWED_WITH_REQUIREMENTS | BLOCKED
  requirements[]
  warnings[]
  matchedRules[]
```

Examples of rule inputs (illustrative, not legal advice):

- Whether a venue is permitted for residents
- Whether a given fiat on-ramp method is allowed
- KYC tier required above thresholds
- Disclosure that a path uses P2P vs licensed exchange service
- Asset restrictions (where applicable)

The router only ranks paths that are `ALLOWED` or `ALLOWED_WITH_REQUIREMENTS` (with requirements surfaced in the Plan prerequisites).

---

## 14. Worked Comparisons

### 14.1 Whitebird-style service (simple)

**Request:** BYN (bank) → BTC (self-custody)

**Path (typical):**

1. `SERVICE_CONVERT` on Whitebird: BYN → BTC  
2. Implicit or explicit payout rail: BTC on specified network to `USER_WALLET`

**Economics:**

- Quoted rate (margin included or explicit)
- Service fee
- Network fee
- Optional payment-method fee

**Plan sketch:**

1. Create Whitebird order for amount X  
2. Pay via the selected funding method (exact amount / reference)  
3. Wait for confirmation  
4. Receive BTC at wallet address  

**Model fit:** one `QuotedRateInstrument` + funding method fees + network `FeeComponent`. No order book, no maker/taker.

### 14.2 Exchange-style (complex)

**Request:** BYN (bank) → USDT TRC20 (self-custody) via Bynex

**Path:**

1. `TRANSFER_IN` — bank BYN deposit to Bynex (`FundingMethod=BANK_TRANSFER`)  
2. `TRADE` — buy USDT for BYN on `USDT/BYN` with strategy `TAKE_BEST` *or* `LIMIT_NEAR_MID`  
3. `TRANSFER_OUT` — withdraw USDT TRC20 to user wallet  

**Economics:**

- Deposit fee / hold  
- Spread + slippage (if taking) or expected improvement + non-fill risk (if making)  
- Maker or taker trading fee  
- Withdrawal fee on TRC20  

**Plan sketch:**

1. Complete KYC if required  
2. Deposit BYN (show bank details, reference, amount)  
3. Wait until balance is tradable  
4. Place order — system proposes market take **or** a reasonable limit; user confirms  
5. Verify USDT balance  
6. Withdraw to TRC20 address; confirm fee and arrive-amount  
7. Wait for network confirmations  

**Model fit:** same Path/Plan machinery; richer `SpotMarket`, `ExecutionStrategy`, and custody movements.

---

## 15. Ranking Objective

Paths are ranked by a transparent score, defaulting to economic optimality under constraints:

```
Score = f(
  netTargetAssetOut,          // primary
  confidence,                 // firm quote vs estimate
  fillRisk,                   // for limit strategies
  totalTime,
  complexityScore,            // fewer fragile manual steps preferred if close in price
  complianceRequirements      // heavier KYC may be de-prioritized if user asked for simplicity
)
```

Users may choose lanes:

- **Best net** — maximize expected output (limit strategies may appear with risk labels)
- **Best firm quote** — prefer executable taker/service quotes only
- **Simplest** — minimize steps / venues
- **Fastest** — minimize settlement time

---

## 16. Domain Boundaries & Adapters

Keep core model venue-agnostic; isolate integrations:

| Adapter | Responsibility |
|---------|----------------|
| Venue catalog adapter | Static capabilities, fee schedules, funding methods |
| Market data adapter | Order books, quoted rates, TTLs |
| Fee engine | Evaluate `FeeComponent`s against amount/tier/side |
| Router | Build candidate paths on the conversion graph |
| Compliance engine | Apply `CompliancePolicy` |
| Planner | Compile path → `Plan` steps via templates |
| Quote oracle | Normalize venue payloads into `Quote` / `LegEconomics` |

Whitebird’s “simple rate” and Bynex’s “book + fees” are **adapters behind the same interfaces**, not parallel product models.

---

## 17. Minimal Entity Map

```
Asset / AssetInstance
Endpoint
Venue
  ├─ FundingMethod
  ├─ FeeSchedule (FeeComponent[])
  ├─ QuotedRateInstrument        // services
  └─ SpotMarket                  // exchanges
ConversionRequest
Leg
Path
Quote / LegEconomics
ExecutionStrategy
CompliancePolicy / ComplianceResult
Plan / Step
```

Relationships (conceptual):

- Request binds source/target endpoints + assets + amount  
- Router emits Paths of Legs against Venue instruments and funding methods  
- Each Path has Quotes and a ComplianceResult  
- Planner turns a Path into a Plan of Steps  

---

## 18. Design Principles

1. **One conversion graph** — services and exchanges are different edge types, not different apps  
2. **Fees are data** — never hard-code “0.2% taker” into path logic; declare fee components  
3. **Execution strategy is explicit** — taking the ask vs making a bid changes economics *and* plan steps  
4. **Transfer ≠ trade** — deposits/withdrawals are legs with their own rails and fees  
5. **Source/target symmetry** — same endpoint and asset catalogs in both roles  
6. **Quotes carry confidence** — do not rank a hopeful limit as if it were a Whitebird firm rate  
7. **Plans are derived** — user guidance is a projection of the domain path, not a separate script system  
8. **Compliance gates routing** — illegal or disallowed paths never win on price  

---

## 19. Out of Scope (for this document)

- API shapes, DB schemas, and UI wireframes  
- Concrete legal conclusions for Belarus (policy content is configurable; this doc only places the hook)  
- Portfolio tracking, tax lots, or accounting  
- Automatic trade execution / custodial order placement (Krypta may remain advisory + step guidance initially)  
- Exact venue integrations for Bynex, Free2Ex, Whitebird  

---

## 20. Open Questions

1. **Advisory vs execution:** Does v1 only generate plans, or also place exchange orders via API where available?  
2. **Limit-order ranking:** How aggressively should expected-but-unfilled maker quotes compete with firm taker/service quotes?  
3. **Multi-network defaults:** When target is “USDT” without network, which networks are default-allowed for Belarus users?  
4. **Identity of endpoints:** Are bank accounts fully modeled in-app, or treated as abstract “user fiat rail” until later?  
5. **P2P venues:** If included later, do they appear as `VenueKind=P2P_PLATFORM` with counterparty risk flags, or stay out of v1 graph?  
6. **Amount sides:** Is “I want exactly 1000 USDT” (TARGET amount) required in v1, or only SOURCE spend amounts?  
7. **Stale data:** What TTL and refresh rules apply before a Plan must be regenerated?

---

## 21. Summary

Krypta’s domain should treat conversion as **routing over a graph of assets and endpoints**, where **venues** expose either **quoted-rate instruments** or **order-book markets**, **funding methods** describe how value enters and exits, and **composable fees + execution strategies** produce honest economics. The same path object then compiles into a **user Plan** — whether the underlying venue is as simple as Whitebird or as involved as a full exchange deposit–trade–withdraw sequence — while a **compliance layer** ensures only Belarus-law-compatible routes are offered.
