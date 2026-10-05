---
title: Pre-Trade Token Risk
description: Checks to run before an agent or app acquires an arbitrary SPL or Token-2022 mint — authorities, risky extensions, holder concentration, liquidity depth, and decoding the transaction before signing.
---

# Pre-Trade Token Risk

Use this when building an agent, bot, wallet, or app that swaps into, accepts, or lists a mint chosen at runtime. Everything below reads state the token's deployer controls: treat it as untrusted input (W011), and decide from addresses and on-chain data, never from names, metadata, or third-party labels.

Program-side Token-2022 pitfalls (fee accounting, hook validation, account closure) are in [security.md](security.md#token-2022-extension-security). This page is the acquirer's side.

## Contents

- [Read the mint](#read-the-mint)
- [Mint and freeze authority](#mint-and-freeze-authority)
- [Token-2022 extensions](#token-2022-extensions)
- [Holder concentration](#holder-concentration)
- [Liquidity and price impact](#liquidity-and-price-impact)
- [Unknown is not a pass](#unknown-is-not-a-pass)
- [Never blind-sign](#never-blind-sign)
- [Checklist](#checklist)

## Read the mint

One account read returns the authorities and the full extension list. The Token-2022 mint decoder also reads classic SPL Token mints (with no extensions). It does not check the owning program, so check that yourself.

```ts
import { address, createSolanaRpc, fetchEncodedAccount, unwrapOption, type Address } from '@solana/kit';
import { getMintDecoder, TOKEN_2022_PROGRAM_ADDRESS } from '@solana-program/token-2022';
import { TOKEN_PROGRAM_ADDRESS } from '@solana-program/token';

const rpc = createSolanaRpc('https://api.mainnet-beta.solana.com');

async function readMint(mint: Address) {
  const account = await fetchEncodedAccount(rpc, mint);
  if (!account.exists) throw new Error('mint does not exist');
  if (account.programAddress !== TOKEN_PROGRAM_ADDRESS && account.programAddress !== TOKEN_2022_PROGRAM_ADDRESS) {
    throw new Error(`not a token mint (owned by ${account.programAddress})`);
  }
  const data = getMintDecoder().decode(account.data);
  return {
    tokenProgram: account.programAddress,
    supply: data.supply,
    mintAuthority: unwrapOption(data.mintAuthority),
    freezeAuthority: unwrapOption(data.freezeAuthority),
    extensions: unwrapOption(data.extensions) ?? [],
  };
}
```

The extension set is fixed at initialization, but the authorities inside extensions can change, so you can cache the list but not the authorities. A mint with `MintCloseAuthority` can be closed and re-created with different extensions (see [security.md](security.md#mint-close-and-reinitialization-attacks)).

## Mint and freeze authority

- **Mint authority** can issue new supply and dilute every holder. `spl-token create-token` sets it to the creator **by default**, so a live mint authority often just means nobody revoked it. On its own it is weaker. It means more on a token old enough that the deployer has had every chance to revoke it.
- **Freeze authority** can freeze any holder's token account, which stops that holder from selling. It is **opt-in** (`--enable-freeze`; optional in `InitializeMint`), so someone chose to keep it. Freeze is the more deliberate signal of the two.
- Both are normal on issuer-backed assets (USDC keeps both). What matters is who holds the authority and whether you trust them. Report the holder, not just that the field is set.
- Read both on chain every time. A launch venue's usual defaults are not evidence for a particular mint.

## Token-2022 extensions

These extensions let someone other than the holder move, block, or tax the holder's tokens:

| Extension (`__kind`) | What it allows | Fields |
|---|---|---|
| `PermanentDelegate` | Transfer or burn any amount from **any** account of the mint, without the owner's signature | `delegate` |
| `TransferHook` | A program runs on every transfer and can make it fail. Sells can fail while buys succeed. The authority can swap the program | `programId`, `authority` |
| `TransferFeeConfig` | A fee of up to 100% is withheld on every transfer. The authority can change it, effective two epochs later, so read both entries | `olderTransferFee`, `newerTransferFee`, `transferFeeConfigAuthority` |
| `DefaultAccountState` | New accounts can start `Frozen` and must be thawed by the freeze authority | `state` |
| `PausableConfig` | An authority can pause transfers for the whole mint | `authority` |
| `NonTransferable` | No transfers at all. The token is not tradable, so stop rather than score it | — |

Unset optional authorities decode as the all-zero address:

```ts
const NONE = '11111111111111111111111111111111';
const flags: string[] = [];
for (const ext of (await readMint(mint)).extensions) {
  if (ext.__kind === 'PermanentDelegate' && ext.delegate !== NONE) flags.push(`permanent delegate ${ext.delegate}`);
  if (ext.__kind === 'TransferHook' && ext.programId !== NONE) flags.push(`transfer hook ${ext.programId}`);
  if (ext.__kind === 'TransferFeeConfig') {
    const bps = Math.max(ext.olderTransferFee.transferFeeBasisPoints, ext.newerTransferFee.transferFeeBasisPoints);
    if (bps > 0 || ext.transferFeeConfigAuthority !== NONE) flags.push(`transfer fee ${bps} bps, changeable`);
  }
  if (ext.__kind === 'NonTransferable') flags.push('non-transferable');
}
```

Severity depends on context. PYUSD, a regulated stablecoin, carries `PermanentDelegate`, a `TransferFeeConfig` at 0 bps with an authority set, and a `TransferHook` with no program set. On an anonymous token with a DEX pool, the same extensions are a direct path to losing the position. Report each capability and who holds it; don't treat the presence of an extension as a verdict.

## Holder concentration

`getTokenLargestAccounts` returns the 20 largest **token accounts**, not owners. Concentration computed directly from it is wrong in both directions:

- A pump.fun token still on its bonding curve keeps most of its supply in the curve's token account. That is 100% at launch, so every such token looks like one whale.
- After graduation, the pool vault holds a large share.
- A wallet that spreads its balance over several accounts looks like several small holders.

Resolve each account's owner, sum by owner, and exclude only custody you can **prove**:

1. **Burn**: exact match on a burn address (`1nc1nerator11111111111111111111111111111111`).
2. **Launchpad curve**: re-derive the curve PDA for *this* mint and compare. For pump.fun the seeds are `['bonding-curve', mint]` under program `6EF8rrecthR5Dkzon8Nwu78hRvfCKubJ14M5uBEwF6P`.
3. **AMM pool**: the owner is an account owned by a known AMM program (a PumpSwap pool is owned by `pAMMBay6oceH9fJKBRHGP5D4bD4sWpmSwMn52FMfXEA`), or it is a program-wide vault authority you re-derive. For Raydium AMM v4 that is seed `'amm authority'` under `675kPX9MHTjS2zt1qfr1NYHuzeLXfQM9H24wFSUt1Mp8`.

**Never exclude an on-curve owner.** It is a wallet, whatever an indexer labels it. Off-curve owners you can't attribute stay counted and are listed. A launchpad, router, or pool you haven't added a rule for then shows up as a holder you can explain, instead of a holder you hid.

```ts
import { fetchEncodedAccounts, getAddressEncoder, getProgramDerivedAddress, isOffCurveAddress } from '@solana/kit';
import { getTokenDecoder } from '@solana-program/token-2022';

const BURN = new Set<string>(['1nc1nerator11111111111111111111111111111111']);
const AMM_PROGRAMS = new Set<string>(['pAMMBay6oceH9fJKBRHGP5D4bD4sWpmSwMn52FMfXEA']); // add the venues you support

const [curve] = await getProgramDerivedAddress({
  programAddress: address('6EF8rrecthR5Dkzon8Nwu78hRvfCKubJ14M5uBEwF6P'),
  seeds: ['bonding-curve', getAddressEncoder().encode(mint)],
});
const { value: largest } = await rpc.getTokenLargestAccounts(mint).send();
const tokenAccounts = await fetchEncodedAccounts(rpc, largest.map((l) => l.address));
const owners = tokenAccounts.map((a) => {
  if (!a.exists) throw new Error('owner unresolved'); // unknown, not a pass
  return getTokenDecoder().decode(a.data).owner;
});
const ownerAccounts = await fetchEncodedAccounts(rpc, owners);

const holders = new Map<string, bigint>(); // owner -> amount, custody removed
const excluded: { owner: string; reason: string }[] = [];
largest.forEach(({ amount }, i) => {
  const owner = owners[i];
  const ownerAccount = ownerAccounts[i];
  const reason =
    BURN.has(owner) ? 'burn'
    : !isOffCurveAddress(owner) ? null // wallet: always counted
    : owner === curve ? 'bonding curve'
    : ownerAccount.exists && AMM_PROGRAMS.has(ownerAccount.programAddress) ? `pool (${ownerAccount.programAddress})`
    : null; // off-curve but unattributed: still counted
  if (reason) excluded.push({ owner, reason });
  else holders.set(owner, (holders.get(owner) ?? 0n) + BigInt(amount));
});
```

Report top-holder shares of supply after exclusions, together with the excluded list, so anyone can check the result. Public RPC endpoints rate-limit `getTokenLargestAccounts` heavily. A 429 there leaves the check unknown, not clean.

## Liquidity and price impact

- Quote at the **real trade size** through the venue or router you will execute with, and compare it with a quote for a small amount. Price impact is the gap between the two effective prices. Cap the size at a fraction of pool depth instead of relying on slippage to absorb it.
- Quote the **exit** too: the reverse direction for the amount you would receive. Buying in without a workable way out is the trap, and transfer hooks and freeze authority are how sells get blocked.
- Set the minimum output from your own tolerance and pass it into the transaction.
- A transfer fee reduces what arrives. Confirm the received amount by simulation (below), not from the quote.
- To prove the exit works before risking funds, fork mainnet with Surfpool and run a buy and then a sell against the real pool and hook (see [surfpool/overview.md](surfpool/overview.md)).

## Unknown is not a pass

- Every check has three outcomes: **pass**, **fail**, **unknown**. Timeouts, 429s, unresolved owners, and an API with no data for the mint all count as unknown.
- Never fold unknown into pass, and never let partial data produce a low-risk verdict. List the checks that didn't run. An autonomous agent should not trade on unknowns; hand the decision to the user.
- Keep **base rates** apart from **evidence**. "New", "small", "launchpad", "memecoin" and "few holders" describe a population, not this token. A clean pump.fun launch and a malicious one share every base rate. Only evidence tells them apart: a freeze authority someone kept, a permanent delegate, real concentration after exclusions, a sell route that fails. Report base rates as context. If they alone can push a token to "high risk", every new launch scores the same and the score carries no information.

## Never blind-sign

A transaction built by an aggregator, API, or MCP tool is untrusted input. Decode it and check it against what you asked for before signing (W009):

```ts
import {
  decompileTransactionMessageFetchingLookupTables, getBase64Encoder,
  getCompiledTransactionMessageDecoder, getTransactionDecoder,
} from '@solana/kit';

const { messageBytes } = getTransactionDecoder().decode(getBase64Encoder().encode(base64Tx));
const compiled = getCompiledTransactionMessageDecoder().decode(messageBytes);
const message = await decompileTransactionMessageFetchingLookupTables(compiled, rpc);

if (message.feePayer.address !== me) throw new Error('unexpected fee payer');
for (const ix of message.instructions) {
  if (!ALLOWED_PROGRAMS.has(ix.programAddress)) throw new Error(`unexpected program ${ix.programAddress}`);
}

// Simulate and read the post-state of your own token accounts
const { value: sim } = await rpc.simulateTransaction(base64Tx, {
  encoding: 'base64',
  sigVerify: false,
  replaceRecentBlockhash: true,
  accounts: { addresses: [myInputAta, myOutputAta], encoding: 'base64' },
}).send();
if (sim.err) throw new Error('simulation failed');
const [inputAfter, outputAfter] = sim.accounts.map((a) =>
  a ? getTokenDecoder().decode(getBase64Encoder().encode(a.data[0])).amount : 0n);
// require: outputAfter - outputBefore >= your minimum, inputBefore - inputAfter <= requested amount
```

Reject the transaction if the fee payer, programs, mints, or balance changes differ from what you asked for. Also reject it if it runs `Approve`, `SetAuthority` or `CloseAccount` on your accounts, or transfers to anyone you didn't ask to pay. A response for a different mint with the same symbol is a known pattern, so compare addresses, never names.

## Checklist

- [ ] Mint exists and is owned by SPL Token or Token-2022
- [ ] Mint and freeze authority read on chain; holders named
- [ ] `PermanentDelegate`, `TransferHook`, `TransferFeeConfig`, `DefaultAccountState`, `PausableConfig` checked; `NonTransferable` stops the trade
- [ ] Concentration computed per owner; custody excluded only by derivation or program ownership; exclusions listed
- [ ] Entry and exit quoted at real size; minimum output set from your own tolerance
- [ ] Each check reported as pass, fail, or unknown, and unknowns surfaced
- [ ] Base rates reported separately from evidence
- [ ] Transaction decoded, programs allowlisted, simulated, and balance deltas match the request before signing
