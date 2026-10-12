# receipt.md — PLAN

> "Every order the swarm takes becomes art."
> Status: **step 1 (data discovery) done as far as the sandbox allows. Waiting for approval before writing code.**

## 0. Where this lives

**Decided:** a self-contained `receipt-md/` folder (pnpm workspace) inside `oopsitsfay/ROMA`. ROMA itself stays untouched.

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

### 1.3 Confirmed from public IMD swarm outputs (round 2)

The swarm publishes its own work as public repos under `github.com/identity-md-launches`. Several of them are IMD API clients and
ship **live API responses captured on 2026-10-02** as test fixtures. I also read [mtezy/imd-writeup](https://github.com/mtezy/imd-writeup)
(a route map dated 24 Sep 2026).

- **Public read routes** (no auth): `/jobs`, `/jobs/:id`, `/jobs/:id/submissions`, `/jobs/:id/result`, `/workflows/:id`,
  `/oracle/requests/:id`, `/launches/:id`, `/schedules`, `/swarm`, `/seats/:tokenId`, `/seats/owners`, `/wallets/:address/earnings`,
  `/agents/by-token/:tokenId.json`.
- **Paid-request routes**: `GET /requests/capabilities`, `POST /requests/quote`, `POST /requests/{id}/submit`, `GET /requests/{id}`.
  `GET /requests/{id}` returns `payment.transactionHash` plus `admission` (the job, oracle or workflow it created), **but only to the
  requester's own bearer token**. We can't read other people's orders this way.
- **Live payment terms** (`capabilities.json`): network `eip155:1`, asset IMD `0xd34a…63b7`, amount `500000000000000000` (0.5 IMD),
  **payTo `0x4e0fa57bde726079356537e2f34d671e9f41adbc`**. The scheme is x402 v2 `exact` over **Permit2**, and the IMD server wallet submits the tx.
  On chain, each paid order is therefore an IMD `Transfer(payer → 0x4e0f…adbc, 0.5e18)`, sent via Permit2.
- **Order lifecycle** (`GET /requests/{id}`): `quoted, expired, payment_pending, payment_failed, admission_pending, admitted`.
- **A real live `/jobs/:id` response** (job `ae3c9745-7363-4bd2-bfaf-dc8944649cd8`, an `audit` template job from 2026-10-02):
  - `paidBy: 0xccbea3abfcb77d8ce532ec2250e9be74fd2778e8`, `state: completed`, `createdAt` and `updatedAt`
  - `nodes[]` each have a `state` (`accepted`, …) and **`seat: {tokenId, agentId}`**. This job was worked by seats 1602, 617, 1548, 1299 and 270.
  - `reviews[]: {chainId: 1, txHash, blockNumber, entries[{agentId, value}]}`. These are the **ERC-8004 reputation writes**.
  - `delivery: {repoUrl, commit, deliveredAt}`.
  - Job states seen elsewhere: `completed`, `executing`, `cancelled`.
- **Intake on Robinhood Chain**: a swarm-built consumer uses `IIntake.request(bytes32 action, bytes body, Callback, asset, amount)
  returns (bytes32 requestId)` and `priceOf(action, asset)`, with action ids like `bytes32("oracle.request@oracle-1")`. **Address mismatch to
  resolve:** that repo's 4663 Intake is `0x1397434cd35e8a9c8ac312a61d3a285eb31dea56`, while the brief has `0xa43e…de82` for all chains.
  The repo itself marks its address "not independently verified". Contract callers ("on-chain orders") are a second, smaller order stream.

### 1.4 Field availability: final answer

| Field | Source | Status |
|---|---|---|
| order id | `job.id` (UUID) from `/jobs` | ✅ public |
| payer | `job.paidBy` | ✅ public |
| action type | `job.template` + `launch.requested` / `oracleRequestId` / `workflow` → maps to `job.open` / `launch.open` / `oracle.request` / `workflow.open`; `/schedules` gives `schedule.*` | ✅ derived |
| status | `job.state` + `nodes[].state` → paid (no nodes running) / working (`executing`) / accepted (`completed` with accepted nodes) / void (`cancelled`, failed) | ✅ public |
| seat that did the work | `nodes[].seat.tokenId` / `agentId`. Several seats per job, so the barcode signs them all and the "lead" seat is the judge node. | ✅ public |
| seat owner (for the mint split) | `/seats/owners` or `ownerOf(tokenId)` on the seat collection | ✅ |
| **payment tx hash** | **not public via API**. Joined on chain: IMD `Transfer(from=paidBy, to=payTo)` on mainnet, nearest block before `job.createdAt`. Falls back to the job id hash if there's no match. | ⚠️ needs a mainnet RPC |
| IMD paid | Transfer value, or `capabilities` price | ✅ |
| reputation tx | `reviews[].txHash` | ✅ public (once sent) |

**Seed** = payment tx hash when joined, else `keccak256(job.id)`. The source is recorded in the metadata, so it's honest about which one was used.

### 1.5 Real tx hashes: status

I can't pull them from this sandbox. Every RPC and explorer host is blocked (Etherscan included, even through WebFetch), and the one API route that
returns them is requester-private. I do now have **real job ids and payers** (above). With **one** mainnet RPC host allowed (e.g.
`ethereum-rpc.publicnode.com`) the indexer finds the hashes itself, via `getLogs(Transfer, to=0x4e0f…adbc)`. Alternatively, copy the
hashes from Etherscan → IMD token → filter by `0x4e0fa57bde726079356537e2f34d671e9f41adbc`.

### 1.6 Ingestion strategy

`OrderSource` interface with three implementations, swappable by env:
1. **`ImdJobsSource`** (primary): poll `GET /jobs` and `/jobs/:id` every 20 s, plus `/schedules` and `/swarm` for the hero counters.
2. **`ChainJoin`**: viem `getLogs` for IMD `Transfer(to=payTo)` on mainnet, joined to jobs by `paidBy` and time, to get the real tx hash and block time.
   Intake `request` events on 4663 are added once the address and ABI are confirmed.
3. **`FixtureSource`**: the real captured responses above (job `ae3c9745…`, capabilities, openapi), used for tests and offline demo,
   labelled as captured on 2026-10-02.

The cache is SQLite via Drizzle (`orders`, `order_events`, `mints`).

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

1. ~~Location~~: decided, `receipt-md/` in this repo.
2. **Network access** (needed for live data and real tx hashes): please add `api.imd.fun`, `imd.fun`, `pool4.imd.fun`, `explorer.imd.fun`,
   `ethereum-sepolia-rpc.publicnode.com`, `ethereum-rpc.publicnode.com`, `robinhood-rpc.publicnode.com` and `api.etherscan.io`
   to the environment's allowed domains (environment settings → Network access; docs: https://code.claude.com/docs/en/cloud-environments#network-access).
   Or paste me `/health` and `/swarm` JSON plus one or two real order tx hashes, and I'll work from those.
3. Until access is available, can I build steps 2–3 (generator, contract) against the schema above with clearly-labelled fixture
   orders? Samples from **real** orders would wait for data access.
4. Sepolia deploy needs a funded deployer key and the signer key in env. Will you provide them, or should I hand you the commands to run?
