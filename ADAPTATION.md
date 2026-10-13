# Robinhood Chain RemoteStacker adaptation

This is the contracts-only launch on chain 4663. The existing `launch.json` is unchanged and deploys only `RemoteStacker` with these four static constructor arguments, in order:

1. Robinhood IMD: `0x5f7bb59365ce557c26dbcaa4ee9d39a4b95b7127`.
2. Live Ethereum StackReceiver, launch #1529: `0x1999a54f573dac75ce8ef62a555dba418c9b3d10`.
3. Ethereum sIMD vault, used only to exclude it as a recipient: `0x9efa934d9fad4ae28c998a40195646b965a97247`.
4. Ethereum LayerZero endpoint id: `30101`.

No constructor adaptation was necessary: it is nonpayable, makes no external calls, requires no deployed dependency code, and configures all four immutables immediately. There is no owner, initializer, proxy, or linked external library. IMD's deployment service supplies the eventual address; this work signs and broadcasts nothing.

## Changes and reasons

| File | Change and requirement |
|---|---|
| `src/multichain/RemoteStacker.sol` | Fix the two reproduced low-severity findings below: reject the Ethereum IMD token and Ethereum LayerZero endpoint in both recipient entry points, and round the weighted arrival clock up using the already-vendored OpenZeppelin `Math.ceilDiv`. Constants preserve the exact four-argument constructor and avoid cross-chain reads. These are the only production-code changes. |
| `test/RemoteStackerAuditRegression.t.sol` | Add regressions for both destination checks, unchanged balances/events on rejection, preservation and clearing of wallet registrations, sub-second clock shifts at capped and uncapped ages, post-flush clock reset, and the bridge-refusal trust assumption. A fuzz property checks that the clock never rounds against the trader or into the future, and that fresh cashback contributes no immediate age-based tip. |
| `test/StackReceiverLaunch.t.sol` | Skip its two receiver-manifest tests when the active manifest does not deploy `StackReceiver`. Keep all strict receiver assertions for receiver launches. This fixes the reproduced full-suite failure without altering the correct RemoteStacker manifest. |
| `test/Deployment.t.sol` | Pin the RemoteStacker's receiver to the exact live launch #1529 address instead of accepting any nonzero address. |
| `test/RemoteStackerLaunch.t.sol` | Rehearse the exact manifest through a local CREATE2 factory on empty chain 4663. Check all arguments, one creation, no external constructor operations or child contracts, no owner, zero deployment value, CREATE2 prediction, init-code and EIP-170 limits, and absence of DELEGATECALL, CALLCODE and SELFDESTRUCT in executable runtime. Skip this rehearsal on other launch branches. |
| `test/RobinhoodFork.t.sol` | Use the actual live receiver address in the existing bridge tests. Add a fork regression that removes the Ethereum peer as the live OFT owner, verifies quote/send failure and atomic custody, then restores the peer and successfully flushes. These writes affect only the local fork. |
| `README.md` | Add the pending Robinhood deployments table and exact wiring, explain the audit corrections and bridge dependency, and correct this branch's manifest/build descriptions. The confirmed RemoteStacker address and transaction must be filled in after IMD deploys it; no address was invented. |
| `ADAPTATION.md` | Record the changes, audit dispositions, launch rehearsal and verification results required by the assignment. |

`LayerZero.sol`, the live `StackReceiver.sol`, all other production contracts, `launch.json`, `foundry.toml`, remappings and vendored dependencies remain unchanged. No token, registry, workflow, deployment script, or new dependency was added.

## Imported audit: reproduction and disposition

### Medium — receiver-only manifest assertions

Finding `99e328ec170fec2793bc538b1dc6c8342e506df3767e8c077948bbbd02e17913` reproduced. Before modification, `forge test --match-path test/StackReceiverLaunch.t.sol` failed both tests with `RemoteStacker != StackReceiver`. The original full offline suite reported **296 passed, 2 failed, 23 skipped (321 total)**. The receiver tests now skip this manifest individually. The strengthened deployment check and new RemoteStacker factory rehearsal validate the active launch instead.

### Low — Ethereum infrastructure accepted as recipients

Finding `3a8e5fe63673d16e1e273b74e9b117f3b3c5a26da43bd68dbc46da38f937883b` reproduced: the new rejection regressions failed on the original contract because the calls succeeded. Both `credit` and `setEthereumRecipient` now reject Ethereum IMD `0xd34a99bc0f67ae1bbd63c660e6d0b0dd03e263b7` and the Ethereum endpoint `0x1a44076050125825900e736c501f859c50fE728c`, even without code at those addresses on Robinhood Chain. The original exclusions, zero-trader error, and zero recipient used to clear a registration are retained. No fifth constructor argument is needed.

### Low — weighted clock rounding against the trader

Finding `aaed61fe7a2acbcb6cb0f276f06e0f0e549c5df28e5618c7f4a804a27fb55d61` reproduced exactly: after 1,000,000 IMD waited 12 hours, adding 0.001 IMD increased the immediate quoted tip from 20,000 IMD to 20,000.00002 IMD. The uncapped six-hour case and the fuzz rounding property also failed on the original source.

Ceiling division now rounds arrival forward by less than one second relative to the exact weighted average, so waiting time and tips round in the trader's favor. At a basis-point threshold this can reduce the prior quote; that conservative rounding is intentional. The existing 12-hour cap, amount weighting, step-rounded tips, dust custody and atomic send behavior remain intact. All new regressions pass with the fix.

### Informational — IMD's bridge owner controls availability

Finding `b5a69a21c2f9158a0b11112c7062f7feab53203b1a01bf95adee1a680d619151` reproduced as a trust assumption, with no production behavior change. The mock refusal test verifies that balances, clock, tips and native funds remain unchanged after failed sends, and that credits continue to be accepted. On a Robinhood fork, removing peer 30101 as the OFT owner makes both `quoteFlush` and `flush` revert with `NoPeer(30101)`; restoring the peer permits delivery. A read at the supplied Robinhood block 87,284,919 confirmed the owner as `0x047F606fD5b2BaA5f5C6c4aB8958E45CB6B054B7`. Ethereum receiver getters at block 26,181,997 confirmed the IMD token and endpoint used above.

The fixed route depends on IMD's OFT owner and LayerZero configuration remaining operational. No rescue, pause, redirect, or alternate exit was introduced: those would change the requested custody model. All imported findings reproduced; none was dismissed as unreproducible.

## Verification

Foundry 1.8.3, using the unchanged project configuration: solc 0.8.26, Cancun, 10,000 optimizer runs, `bytecode_hash = "none"`, configured fuzz and invariant settings.

- `forge build`: passed.
- `forge test` with all three RPC variables unset: **303 passed, 0 failed, 26 skipped (329 total)**. Skips are 22 fork tests and four tests for other launch manifests.
- `forge test` with `ETH_RPC_URL`, `SEPOLIA_RPC_URL` and `ROBINHOOD_RPC_URL` set: **325 passed, 0 failed, 4 skipped (329 total)**. All 22 fork tests passed; only the two Stacker-manifest and two StackReceiver-manifest checks skipped.
- `forge test --match-path test/RemoteStackerAuditRegression.t.sol`: six regressions passed, including the configured 256-run clock fuzz test.
- `forge test --match-path test/RemoteStackerLaunch.t.sol`: passed the exact-manifest CREATE2 rehearsal. Runtime is **8,182 bytes**, below the 24,576-byte EIP-170 limit; no forbidden executable opcodes.
- Robinhood fork tests: **5 passed**, including peer removal and restoration, two- and fifty-recipient sends, fee failure atomicity and live OFT facts.

RPC source identities are [Ethereum PublicNode](https://ethereum-rpc.publicnode.com), [Robinhood PublicNode](https://robinhood-rpc.publicnode.com), and [Sepolia PublicNode](https://ethereum-sepolia-rpc.publicnode.com); supplied contributor routing was used for Ethereum and Robinhood. Fork actions are local simulations, not live transactions. Slither and Mythril were not run.
