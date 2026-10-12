# receipt.md — PLAN

> "Every order the swarm takes becomes art."
> Status: **step 1 (data discovery) done as far as the sandbox allows. Waiting for approval before writing code.**

## 0. Where this lives

This repo (`oopsitsfay/ROMA`) holds Sentient's ROMA Python framework, which has nothing to do with receipt.md. I'm proposing to build
the whole project in a self-contained `receipt-md/` folder (pnpm workspace) and leave ROMA untouched. A separate repo would
be cleaner for the judges. See Q1.

## 1. Data discovery: what I verified and what I couldn't

### 1.1 Reachability from the build sandbox

| Source | Result |
|---|---|
| `api.imd.fun/health`, `/swarm` | **blocked** (proxy 403; WebFetch: DNS not found) |
| `imd.fun/docs`, `pool4.imd.fun/docs`, `explorer.imd.fun` | **blocked** |
| Public RPCs (publicnode, drpc, 1rpc, Robinhood RPC, Sepolia RPCs) | **blocked** |
| `github.com/identity-md/worker` (git clone) | ✅ read the README and the 3.3 MB `dist/` bundle |
| npm registry | ✅ (deps install fine) |

So I've **re-confirmed nothing from the live `/health` endpoint**. Every address in the brief is still marked unverified,
except where the worker bundle confirms it (below). To verify live data I need `api.imd.fun`, `imd.fun`,
`explorer.imd.fun` and one RPC host per chain added to this environment's allowed domains (Q2).

### 1.2 Confirmed from the worker source (`Identity-md/worker`, `dist/client.js`)

- **Default control-plane server is `https://api.imd.fun`** (`DEFAULT_SERVER`, can be overridden with `IDENTITYMD_SERVER`). The worker
  connects over WSS. The only REST path in the bundle is `/health`.
- **Chain 4663 = "Robinhood Chain"** (mainnet). RPCs: `https://robinhood-rpc.publicnode.com`, `https://rpc.mainnet.chain.robinhood.com`.
  Explorer: `https://robin.etherscan.io`.
- **IMD token, mainnet `0xd34a99bc0f67ae1bbd63c660e6d0b0dd03e263b7`**: `symbol:"IMD", name:"Identity.md", decimals:18`.
  It appears as a Uniswap v4 pair token. ✅ matches the brief.
- **IMD token, chain 4663 `0x5f7bb59365ce557c26dbcaa4ee9d39a4b95b7127`** also appears in the chain config. ✅
- Sepolia (11155111) is configured with explorer `sepolia.etherscan.io`.
- **Intake protocol (the on-chain side of x402 orders).** The worker can act as a "deployer" that settles intake requests:
  - `IntakeQuoteTerms { chainId, intake, asset, amount, expiresAt, quoteHash, signature }`: a signed price quote, payable in `asset` (IMD).
  - `IntakeCompletionTask { completionId, chainId, intake, requestId: bytes32, status: 0|1|2, resultHash: bytes32, uri, args, txHash }`.
    This means **every order has an on-chain `requestId` (bytes32) on the intake contract, and a completion tx that writes `status`
    + `resultHash` + `uri`**. My working guess is that the 3 status codes are something like completed / failed / refunded. That needs confirming
    against the contract ABI.
  - `IntakeDeliveryTask { delivery, sourceChainId, sourceIntake, requestId, target, selector, args }`: cross-chain delivery
    of a result to a target contract.
  - `IntakeResultPin*`: the result is pinned to IPFS (CID) and bound to `resultHash`.
- **Worker job lifecycle**: phases `preparing, planning, working, checking, bundling, uploading`, then outcome `accepted | rejected | error`.
  Submissions carry `wallet`, `tokenId` (the seat), `startedAt`, `endedAt`, `outcome`, `submissionHash`.
- **`receipt-archive` task**: batches of `receipt.json` plus `jobs/<sha256>.json` get pinned to IPFS. The network already archives
  per-job receipts, so a public CID index may exist (to confirm via the docs or explorer).
- **Not in the worker** (they live on the server side): the user-facing action names (`job.open`, `oracle.request`, …), the x402
  price, the intake contract ABI, and any per-order REST endpoint.

### 1.3 Field availability matrix (current best knowledge)

| Field | Planned source | Confidence |
|---|---|---|
| order id | intake `requestId` (bytes32) from the request event | high (schema confirmed), event name unknown |
| payer | `from` of the IMD `Transfer` to the intake contract, or a field of the request event | medium |
| action type | request event args/`uri`, or API | **unknown**. May need the API. |
| tx hash, timestamp | logs plus `getBlock` | high |
| amount paid | IMD `Transfer` value | high |
| seat (tokenId) that did the work | API `/swarm` / explorer / receipt-archive JSON | **unknown** at per-order level |
| status paid → working → accepted | paid = request log, accepted/void = completion log `status`; **working** only from the API/explorer | partial |
| result URI / CID | completion `uri` / `resultHash` | high |

### 1.4 Ingestion strategy (adapter pattern so we can swap once the data is verified)

`OrderSource` interface with three implementations, tried in this order:
1. **`ImdApiSource`**: per-order or per-job endpoint, if the docs or explorer expose one (to verify first once the network is opened).
2. **`ChainSource`** (fallback): viem `getLogs` on the intake contract (`0xa43e…de82`) and IMD `Transfer(to=intake)` on 1 / 4663 / Sepolia,
   joined with `/swarm` for the seat and the "working" state. Event ABIs get filled in once I can read the verified contract.
3. **`FixtureSource`**: recorded real responses, used for tests, samples and offline demos. Clearly labelled, never presented as live.

The cache is SQLite via Drizzle (`orders`, `order_events`, `mints`). The indexer polls every 20 s.

## 2. Architecture

```
receipt-md/
  packages/art/        # pure TS SVG generator + vitest determinism tests  (step 2)
  contracts/           # Foundry: ReceiptMD.sol, tests, deploy scripts       (step 3)
  apps/web/            # Next.js App Router, wagmi/viem/RainbowKit, Tailwind, Drizzle/SQLite (step 4)
  scripts/render-samples.ts  -> samples/*.svg (20 real orders)
  README.md  DEMO.md  .env.example
```

**Generator**: `render(order, stage) -> svg string`. sfc32 is seeded from the tx-hash bytes. Composition depends on the action, the palette
balance on the payer, brightness on the UTC hour, and the barcode on the seat. Stages are sketch, inking, final and void. Tests assert
byte-identical output for the same input, different output for different seeds, and valid 1000×1000 SVG for every action and stage.

**Contract `ReceiptMD`**: OZ ERC721 + EIP712 + Ownable. The voucher is
`{orderId, payer, seed, action, seatId, seatOwner, tokenURI, deadline}`, signed by the backend signer, and the voucher type includes `status` = accepted.
`mint` checks the sig, deadline, `msg.sender == payer` and `!minted[orderId]`, then pulls the fee with `transferFrom`. It sends `seatShareBps`
to the seat owner and burns the rest. The owner can only set signer, fee and split (split capped at 100%). Seed and traits are stored on-chain. tokenId =
`uint256(orderId)`, so ids stay unique and readable.

**Burn choice**: still open. I couldn't read the IMD token bytecode. The plan is to check for `burn(uint256)` in the verified
source once RPC or Etherscan is reachable. If it exists, the contract pulls the fee to itself and calls `burn`. If it doesn't, it does `transferFrom(payer, 0x…dEaD)`.
I'll pick one at deploy time via a constructor flag, check it in a fork test, and write the choice up in the README. The Sepolia test token will be our own
mock unless the real Sepolia IMD is confirmed.

## 3. Questions for you

1. **Location**: build in `receipt-md/` inside ROMA on `claude/focused-sagan-oi9fvt`, or do you want a new repo?
2. **Network access** (blocking for verified data): please add `api.imd.fun`, `imd.fun`, `pool4.imd.fun`, `explorer.imd.fun`,
   `ethereum-sepolia-rpc.publicnode.com`, `ethereum-rpc.publicnode.com`, `robinhood-rpc.publicnode.com` and `api.etherscan.io`
   to the environment's allowed domains (environment settings → Network access; docs: https://code.claude.com/docs/en/cloud-environments#network-access).
   Or paste me `/health` and `/swarm` JSON plus one or two real order tx hashes, and I'll work from those.
3. Until access is available, can I build steps 2–3 (generator, contract) against the schema above with clearly-labelled fixture
   orders? Samples from **real** orders would wait for data access.
4. Sepolia deploy needs a funded deployer key and the signer key in env. Will you provide them, or should I hand you the commands to run?
