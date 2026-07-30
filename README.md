# GhostSwap

**A privacy layer for DEX swaps, built on iExec Nox.**

GhostSwap sits in front of an existing decentralized exchange and hides swap details — token pair, amount, direction, slippage — until the moment a trade settles. It does not fork, wrap, or modify the underlying DEX. It calls the DEX's existing router interface, the same way any other integrator would.

**Live contract (Sepolia testnet):** [`0xEdE85514187514e36B429b259E5b87Ee9532Bc23`](https://sepolia.etherscan.io/address/0xEdE85514187514e36B429b259E5b87Ee9532Bc23)
**Verified on Sourcify:** yes

---

## The Problem

Every swap submitted to Uniswap (and structurally identical AMMs) sits in the public mempool before it executes. Anyone running a searcher bot can read the pending transaction, see the exact token pair and amount, and act on that information before the trade lands — front-running or sandwiching it.

This isn't a bug in any specific DEX. It's a property of how public blockchains order and broadcast transactions. That's also why it can't be fixed by forking a DEX or launching a new pool — the fix has to sit at the layer where the *request* is made, before it ever touches a public mempool in readable form.

## The Solution

GhostSwap splits a swap into two separate steps, submitted as two separate transactions:

1. **Request.** The trader submits a swap request as an encrypted payload plus a commitment hash (`keccak256` of the payload). At this stage, an outside observer sees a transaction to the GhostSwap contract, but the payload is opaque — no token addresses, amounts, or direction can be read from it directly.
2. **Settle.** A separate transaction — submitted by an authorized relayer — provides the decrypted swap parameters (token in, token out, amount in, minimum amount out) along with an attestation. The contract verifies the attestation and calls the underlying DEX's router (`exactInputSingle` on Uniswap V3 in this build) to execute the actual swap.

Only at the settle step does the trade become public. Before that, nothing meaningful can be extracted from on-chain data.

## What's Real vs. What's Simulated

This is a hackathon build, and being precise about scope matters more than making it sound finished. Here's exactly where the line is:

**Real and working:**
- The Solidity contract (`GhostSwap.sol`) is deployed on Sepolia testnet at the address above and verified on Sourcify.
- A live transaction was submitted and confirmed on-chain: request ID `0x72b58ed2c2d45e65e3d93c3b80983bec0653bb1f0acb865f1445abc7714a4450`, block `11379050`. You can look this up directly on Sepolia Etherscan.
- The frontend (`ghostswap-app.html`) connects a real wallet via `ethers.js`, builds a real commitment hash client-side, and submits a real `requestSwap` transaction to the deployed contract.
- The settlement call (`settleSwap`) is real Solidity logic that calls Uniswap V3's router — it is not a mock function. In this deployment, the contract's `noxRelayer` role is set to the deployer's own wallet, so the full request → settle flow can be exercised end-to-end without external infrastructure.

**Simulated, and clearly labeled as such in the code and UI:**
- The actual decryption and matching computation that would run inside an iExec Nox Trusted Execution Environment is not wired up in this build. Doing that requires iExec's Nox SDK and TEE infrastructure access, which is the direct next step, not something faked to look real. The frontend's "Run simulated Nox match" step exists specifically to show *where* that computation would run and *what* it would do, without pretending it already does.
- The attestation passed to `settleSwap` in this build is a placeholder (`ethers.randomBytes(32)`), not a genuine cryptographic proof from a TEE. The contract has a stub `_verifyAttestation` function with a comment marking exactly what needs to be replaced.

If you read the contract, this distinction is visible directly in the code comments — nothing here is dressed up to look more complete than it is.

## Why This Doesn't Break Composability

GhostSwap never modifies token contracts, forks a DEX, or introduces a new pool. The settlement transaction calls Uniswap's router through its standard public interface. Liquidity, pricing, and execution all still happen on the real DEX. The only thing GhostSwap changes is *when* the details of a trade become visible on-chain — not *how* the trade is settled or priced.

## Trust Model

GhostSwap's privacy guarantee, once real Nox integration is added, rests on a Trusted Execution Environment: hardware-enforced isolation, not a zero-knowledge proof. This is a meaningfully different trust assumption than a fully trustless system — a TEE vendor and its attestation infrastructure are part of the trust boundary. This is the same model used by other TEE-based confidential compute systems (Oasis, Secret Network, and iExec's own Nox network). It is not being presented here as trustless, because it isn't.

## Repository Contents

| File | Purpose |
|---|---|
| `GhostSwap.sol` | The router contract — `requestSwap`, `settleSwap`, `cancelRequest`, relayer/owner access control |
| `ghostswap-app.html` | Wallet-connected frontend. Submits real transactions to the deployed contract via `ethers.js` |
| `ghostswap-demo.html` | A self-contained visual walkthrough of the full request → match → settle flow, used for demo purposes |
| `DEPLOYMENT-GUIDE.md` | Exact steps to deploy `GhostSwap.sol` yourself (Remix + MetaMask + Sepolia), including constructor arguments |
| `pitch-demo-script.md` | Talking points used for the project's demo video |

## Contract Interface

```solidity
function requestSwap(bytes32 commitment, bytes calldata encryptedPayload) 
    external returns (bytes32 requestId);

function settleSwap(
    bytes32 requestId,
    address tokenIn,
    address tokenOut,
    uint24 poolFee,
    uint256 amountIn,
    uint256 minAmountOut,
    bytes calldata attestation
) external; // onlyNoxRelayer

function cancelRequest(bytes32 requestId) external; // only original trader, only if still Submitted
```

Constructor takes two arguments at deploy time:
- `_dexRouter` — address of the target DEX's router (Uniswap V3 router used in this deployment: `0xE592427A0AEce92De3Edee1F18E0157C05861564`)
- `_noxRelayer` — the address authorized to call `settleSwap` (in production, this would be a Nox-controlled address; in this build, it's the deployer's own wallet for demonstration purposes)

## Running It Yourself

1. Deploy `GhostSwap.sol` following `DEPLOYMENT-GUIDE.md` (Remix + MetaMask, Sepolia testnet, ~15 minutes).
2. Open `ghostswap-app.html` in a browser with a Web3 wallet extension, or in a wallet app's built-in browser (note: MetaMask cannot inject into plain mobile Safari — use MetaMask's own in-app browser on mobile).
3. Connect your wallet, paste in your deployed contract address.
4. Submit a swap request — this is a real, confirmable on-chain transaction.
5. Run the simulated Nox match step to see the intended enclave computation flow.
6. Call settlement — since your wallet is the contract's `noxRelayer`, this executes the real settlement path.

## Roadmap

- Replace the placeholder attestation and `_verifyAttestation` stub with real iExec Nox SDK integration (encryption, TEE-side matching, genuine attestation verification).
- Support batching multiple pending requests into a single settlement to reduce gas and strengthen privacy through aggregation.
- Multi-DEX routing computed privately inside the enclave (best execution across multiple pools, not just one).
- On-chain attestation registry so settlement proofs are independently verifiable by anyone, not just the relayer.

## Originality

GhostSwap was built from scratch for this hackathon. It does not reuse code, contracts, or assets from any prior submission, including the earlier VIBE Coding Hackathon. It integrates with Uniswap's existing, unmodified router interface and is designed to plug into iExec Nox's actual infrastructure once SDK access is available.
