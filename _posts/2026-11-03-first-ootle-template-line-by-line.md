---
layout: post
title: "Your first Ootle template, line by line: components, vaults, access rules, tests, and what a publish costs"
date: 2026-11-03 05:30:00 +0530
tags: [tari, ootle, rust, wasm, smart-contracts]
---

**TL;DR:** An Ootle template is a Rust crate compiled to WASM. The first `pub struct` in a `#[template]` module becomes your component, its fields are on-chain state, and a panic rolls back the whole transaction. Resources (tokens and NFTs) are native objects that sit in vaults, and by default only the creator can call your methods. Publishing a ~72 KiB binary costs about 0.61 TARI, and a 1 MiB one about 89 TARI, so binary size matters more than you'd expect. Here's the official guessing-game walkthrough, condensed, with the numbers.

## Scaffold it

The docs recommend `cargo-generate` over a bare `cargo new`:

    cargo install cargo-generate
    cargo generate https://github.com/tari-project/wasm-template wasm_templates/empty -din_cargo_workspace=false --name guessing_game

That gives you `Cargo.toml`, `src/lib.rs` for the template and `tests/test.rs`. Running `cargo test` straight away compiles the empty template to WASM and runs it through the real engine.

## The shape of a template

    use tari_template_lib::prelude::*;

    #[template]
    mod template {
        use super::*;

        pub struct GuessingGame {
            prize_vault: Vault,
            guesses: HashMap<RistrettoPublicKeyBytes, Guess>,
            round_number: u32,
        }

        impl GuessingGame {
            pub fn new() -> Component<Self> { /* ... */ }
            pub fn guess(&mut self, guess: u8, payout_to: ComponentAddress) { /* ... */ }
        }
    }

The rules that matter:

- **The first `pub struct` (or enum) is the component**, and it names the template. Any other structs in the module are plain data types.
- **Fields are persisted state.** They have to be CBOR-encodable. The macro derives `minicbor` encoding for everything declared inside the module, so you don't need serde.
- **`&mut self` = method, no `self` = function.** Transactions call methods with `CallMethod` and functions with `CallFunction`. Private `fn`s stay internal.
- **Any function that returns `Self` is a constructor.** Calling it creates a new component on-chain. One template can back many components, each with its own state.
- **Errors are panics.** `assert!` or `panic!` fails the transaction, and nothing it touched gets committed.
- **No state at all?** `#[template(stateless)]` gives you a pure function library. The compiler rejects `&self` methods and `-> Self` constructors in that mode.

## Resources and vaults

Assets on Ootle aren't balances in a contract's mapping. They're native resources that can be minted and moved but not copied or accidentally destroyed. There are three kinds:

- **Public fungible:** like ERC-20, with visible amounts.
- **Non-fungible:** like ERC-721, with visible metadata and owner.
- **Stealth fungible:** blinded UTXOs that hide both amount and owner.

The game's constructor makes an NFT resource and an empty vault to hold the prize:

    let prize_resource = ResourceBuilder::non_fungible()
        .metadata("name", "Guessing Game Prize")
        .with_token_symbol("🎲")
        .build();                       // returns a ResourceAddress
    let prize_vault = Vault::new_empty(prize_resource);

Things that will bite you:

- **A vault holds exactly one resource.** If you deposit anything else, the transaction fails.
- **A new vault must end up in a component's state before the function returns**, or the transaction fails.
- **Only the resource owner can mint by default.** You can widen that with `.mintable(rule![...], UpdateRule)`.

To pay a winner, the method withdraws from the vault into a `Bucket` and calls `deposit` on the player's account:

    guess.payout_to.invoke("deposit", args![prize]);

If nobody wins, `prize.burn()` destroys it.

## Access rules

By default, only the component's creator can call its methods. To let anyone play, the game opens up one method:

    let access_rules = ComponentAccessRules::new()
        .method("guess", rule![allow_all]);
    Component::new(Self { /* ... */ })
        .with_access_rules(access_rules)
        .create()

Two defaults are worth knowing: `OwnerRule::OwnedBySigner`, and `UpdateRule::Locked`, which means the component's template can never be swapped.

For identity, use `CallerContext::transaction_signer_public_key()`. The docs say outright that you should **never take the player's key as a method argument**, because anyone can spoof it.

## Randomness is deterministic

`tari_template_lib::rand::random_bytes` gets its entropy from the transaction plus an internal counter. That's hard to game but not impossible, so the docs say not to use it where you need cryptographic security. The `rand` crate can't work inside a template at all, because every validator has to get the same result. Requests are capped at 1 KiB.

## Testing against the real engine

`tari_template_test_tooling` (dev-dependency, `0.39` in the docs) compiles your crate and runs transactions through the same engine the network uses:

    let mut test = TemplateTest::my_crate();
    let (player, _proof, player_key) = test.create_funded_account();
    let reason = test.execute_expect_failure(
        test.transaction()
            .call_method(game, "guess", args![100u8, player])
            .build_and_seal(&player_key),
        vec![],
    );
    assert_reject_reason(reason, "Template error: Guess must be from 0 to 10");

Useful details:

- **Fees are off by default.** Call `test.enable_fees()` when fees are part of what you're testing.
- **`extract_component_value(game, "$.2")` reads state by field index, not name.** If you reorder struct fields, your test paths change.
- **Signer badges only reach the top-level call.** A lent proof reaches one hop. A method that passes when called directly can be denied when it's reached through another template.
- **Speed up tests.** Setting `opt-level = 2` for the wasmer and cranelift packages in your dev profile makes tests about 10x faster.

## Publishing, and why size is money

Build with `cargo build --target wasm32-unknown-unknown --release`, then publish the `.wasm` from the wallet web UI ("Publish Template", then Estimate Fee) or with `tari publish`. The wallet daemon runs `wasm-opt` for you.

From the docs' fee table:

| Binary | Publish charge | `TemplateLoad` per caller |
|---|---|---|
| 72 KiB | ~0.61 TARI | 240 µT |
| 128 KiB | ~0.87 TARI | 430 µT |
| 256 KiB | ~3.6 TARI | 870 µT |
| 1 MiB (hard cap) | ~89 TARI | 3,500 µT |

The storage charge grows quadratically past a 96 KiB allowance (100 µT × KiB² beyond it). Every caller also pays `TemplateLoad`, 10 µT per 3 KB, each time a transaction loads your template.

The docs' recommended release profile is `opt-level='s'`, `lto=true`, `codegen-units=1`, `panic='abort'`, `strip=true`. It took their test template from 95 KiB to 78 KiB after wasm-opt. Adding `no_std` plus immediate-abort panics got it to 55 KiB, 42% smaller. Keep `strip=true` too: the engine rejects binaries with any custom section other than `tari_tdef`, and the toolchain emits three by default.

## What this means if you're building

- **Start from `cargo generate`, not a blank crate.** You get the test harness wired in from day one.
- **Design around vaults, not mappings.** Every asset your component holds lives in a vault field, one vault per resource.
- **Lock down by default, open up per method.** Creator-only is already the default, so add `allow_all` only where you mean it.
- **Assert on the reject message, not just the failure.** A test that fails for the wrong reason (wrong key, no funds) is a false green.
- **Watch the 96 KiB line in CI.** Below it you pay ordinary storage. Above it the premium grows quadratically, and your users pay a per-call load fee for as long as the template is used.
- **Share logic by calling other templates.** Calling a published library with `TemplateManager::get(address).call(...)` means you pay the publish premium once, not once per template.

Open question for other Ootle builders: has anyone gone fully `no_std` + `talc` on a real template yet, and was the size saving worth the extra friction while you were still iterating?

*Sources: ootle.tari.com (Guides: Templates Overview, Building a Guessing Game, Testing Your Template, Publishing Templates; Reference: Limits, Reducing Template Size), github.com/tari-project/wasm-template.*
