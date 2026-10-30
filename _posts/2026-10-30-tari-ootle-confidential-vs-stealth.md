---
layout: post
title: "Confidential vs Stealth on Tari Ootle: What Is Actually Hidden, and What Still Leaks"
date: 2026-10-30 12:30:00 +0530
tags: [tari, ootle, privacy, cryptography]
---

"Private by default" is the kind of phrase every chain likes to use. Tari's Ootle layer 2 is more honest about it. Ootle doesn't use one privacy switch. It has two separate privacy layers that stack, and its own docs list exactly what each one hides and what still leaks. This piece walks through that model in plain language.

## Ootle in one paragraph

Ootle is Tari's smart contract layer. It settles against the Minotari proof-of-work base layer, which acts as its registrar and clock, but it runs its own consensus, networking and execution. Validator nodes are organised into committees, each covering a slice of the state, and they reach HotStuff-based BFT consensus. Smart contracts are called templates. They compile to WebAssembly, and a running instance of a template is a component.

## The four resource types

Every asset on Ootle is a resource, and its type is the single most important privacy decision a builder makes. There are exactly four:

- Public Fungible: interchangeable tokens. Balances and owners are visible, and funds live in vaults.
- Public Non-Fungible: NFTs. Each item is visible, and funds live in vaults.
- Confidential: amounts are hidden, but the vault holding them (and its component) is visible.
- Stealth: amounts and owners are both hidden. Funds live as individual UTXOs, not in vaults.

The native token, TARI, is a stealth resource.

## Layer one: Confidential, which hides the number

A confidential balance is stored as a Pedersen commitment: `C = mask·G + v·H`. In plain terms, the value v is locked behind a random blinding factor (the "mask"). An observer can see that a vault holds something, but can't tell a commitment to 5 from a commitment to 5,000.

Commitments can also be added together. That's what lets the network check that inputs equal outputs without opening any of them. To stop someone faking that check with a negative number, every output carries a Bulletproofs+ range proof, which shows the hidden value is valid without revealing it. Ootle batches these proofs, so a many-output transfer stays cheap to verify.

The limit: a confidential balance still sits at a known address. The vault can be watched, even if its contents can't be read.

## Layer two: Stealth, which hides the place too

Stealth transfers remove that address entirely. For each output, the sender does a Diffie-Hellman exchange with the recipient's public key and derives a one-time address. Only the recipient, using their private key and a nonce published with the output, can recompute the shared secret, recognise the output as theirs and spend it. An outside observer sees the one-time key and the nonce, and can't link either back to the recipient.

Each stealth output also carries an encrypted payload (XChaCha20-Poly1305) holding the value, the mask the recipient needs to spend it later, and an optional memo of up to 255 bytes.

The result: an observer can see that outputs exist, but not how much they hold or who owns them. With a confidential balance there's an address to watch. With stealth, there's nothing to watch in the first place.

## Privacy with an audit door

Sometimes someone needs to see amounts, such as an auditor or an asset issuer. Ootle supports this with view keys. A resource can attach an ElGamal-encrypted balance to each output, and the view-key holder can read the value while everyone else sees only that a valid value exists. Crucially, a view key can't spend. Disclosure is a choice, not a weakening of the cryptography.

## The honest part: what still leaks

This is the section most chains skip. Ootle's own threat model says privacy is a property of how a transaction is built, not a blanket guarantee.

**Hidden:**

- Confidential and stealth amounts
- Stealth ownership

**Revealed by design:**

- Transaction fees, which must be paid in a known amount
- Any input or output a transaction chooses to reveal

**Leaks you can cause yourself:**

- Signing with a public key. Calling a component method that checks your identity links the transaction to you.
- Amount correlation. Withdrawing exactly the amount you then send as a revealed input makes the two linkable.
- Graph analysis. Which inputs were spent and which outputs were created is still public.
- Script-path spends, which publish the condition they use, and that might name a key or a deadline.

The guidance that follows: spend stealth inputs where you can, keep revealed amounts to the fee, and avoid identity-checked calls when privacy matters.

## Why this matters for builders

On most smart contract chains, privacy is bolted on later, through mixers or separate shielded pools. On Ootle, it's part of the asset definition. A builder choosing between `ResourceBuilder::confidential()` and `ResourceBuilder::stealth()` is choosing what the world can learn about their users. Confidential suits apps where a public vault is fine but balances shouldn't be, like a payroll or treasury component. Stealth suits money that should move like cash.

## Where XTM and TARI fit

Tari has two tokens. XTM is mined on the base layer. The only way to create TARI on Ootle is to burn XTM 1:1, a one-way path with no peg-out. Part of Ootle's transaction fees is burned, which Tari calls the Turbine model, and it's designed to keep overall supply in balance.

## The takeaway

Ootle's privacy is real cryptography, not marketing: Pedersen commitments, Bulletproofs+ range proofs, one-time stealth addresses and optional view keys. Just as useful is that the docs tell you plainly where the edges are. If you're building on Ootle, pick your resource type deliberately and build transactions with the leak list in mind.

Sources: Tari Ootle docs (ootle.tari.com: Privacy; Templates and Assets; State and Execution), the Tari RFC library (rfc.tari.com/ootle), and Tari Tokenomics (tari.com/tokenomics).

