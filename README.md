# GhostSwap
### A privacy-preserving DEX router powered by iExec Nox

**Positioning:** We're not replacing DEXs. We're adding privacy to them.

## The Problem
Every swap on Uniswap (and similar AMMs) is visible in the mempool before it executes. MEV bots and searchers watch pending transactions and front-run or sandwich large trades. This is a structural problem with public blockchains, not a flaw in any one DEX — which is exactly why it can't be fixed by forking the DEX itself.

## The Solution
GhostSwap sits *in front of* an existing DEX as a thin privacy layer:

1. **Request** — A user submits a swap request as an encrypted blob (token pair, amount, side, slippage) plus a commitment hash. On-chain, this looks like opaque data — no observer can extract trade details.
2. **Compute (Nox / TEE)** — iExec Nox decrypts the request inside a Trusted Execution Environment, reads live pool state from the target DEX, and computes the exact swap parameters.
3. **Settle** — Nox calls back into the GhostSwap contract with the computed instruction plus an attestation proving correct execution. The contract verifies the attestation and executes the real swap on the unmodified DEX.
4. **Reveal** — The trade becomes public only at the moment of settlement — never before.

## Why This Doesn't Break Composability
GhostSwap never forks, wraps token logic, or modifies Uniswap's contracts. It calls the DEX's existing router interface (`exactInputSingle`, etc.) exactly like any other integrator would. Liquidity, pricing, and settlement all still happen on the real DEX. GhostSwap only changes *when* trade details become visible — not *how* the trade settles.

## Trust Model
GhostSwap relies on a Trusted Execution Environment (hardware-based confidential compute), not a fully trustless zero-knowledge proof. This is the same trust assumption used by other TEE-based confidential compute platforms (e.g. Oasis, Secret Network). The attestation mechanism lets anyone verify that Nox computed the swap correctly and didn't tamper with the request — without needing to trust iExec's operators directly.

## MVP Scope (hackathon build)
- Single relayer-authorized settlement path (`onlyNoxRelayer`)
- One DEX integration target (Uniswap V3 style router)
- Single swap request → single settlement (no batching)
- Attestation verification stubbed for demo purposes — flagged clearly in code, real Nox SDK call would replace it

## Roadmap (v2, for the pitch)
- Batch matching across multiple pending requests (reduces gas, adds further privacy via aggregation)
- Multi-DEX routing (best execution across Uniswap, Curve, etc., computed privately inside Nox)
- On-chain attestation registry for public verifiability of every settlement
- Support for limit-style private orders, not just market swaps

## Files
- `GhostSwap.sol` — the router contract
- `ghostswap-demo.html` — interactive visual demo (used for the submission video)
- `ghostswap-app.html` — functional frontend (connects a real wallet, submits real on-chain requests, calls real settlement — see "What's Real vs Simulated" below)
- `DEPLOYMENT-GUIDE.md` — step-by-step, no-coding deployment instructions using Remix + MetaMask + Sepolia testnet

## Deployment Status
## Deployment Status
✅ **Live on Sepolia testnet.** Contract address: `0xEdE85514187514e36B429b259E5b87Ee9532Bc23`

You can verify this directly on [Sepolia Etherscan](https://sepolia.etherscan.io/address/0xEdE85514187514e36B429b259E5b87Ee9532Bc23).

## Setup, Deployment & Usage
Full step-by-step instructions are in `DEPLOYMENT-GUIDE.md`. Short version:
1. Get a MetaMask wallet + Sepolia testnet ETH (free faucet).
2. Deploy `GhostSwap.sol` via [Remix](https://remix.ethereum.org) or [thirdweb](https://thirdweb.com/dashboard) (paste code, click Deploy) — requires a desktop browser.
3. Open `ghostswap-app.html`, connect your wallet, paste in the deployed contract address.
4. Submit a swap request (real on-chain transaction), run the simulated Nox match, then settle (real on-chain transaction).

## What's Real vs. Simulated
Being transparent about scope, as this is a hackathon MVP:
- **Real:** wallet connection, on-chain request submission, on-chain settlement call, the Solidity contract logic itself, the interface with Uniswap's router.
- **Simulated:** the actual Nox TEE decryption and matching computation. This requires iExec's Nox SDK and enclave access, which is the direct next integration step post-hackathon. In this demo, the "Nox relayer" role is played by the deployer's own wallet so the full request → match → settle flow can be demonstrated end-to-end.

## Originality & Existing Work
GhostSwap was built from scratch during this hackathon. It does not reuse or modify any code from the previous VIBE Coding Hackathon or any other prior submission. It integrates with Uniswap's existing, unmodified router interface (no fork, no contract changes on their side) and is designed to plug into iExec Nox once full SDK access is available.
