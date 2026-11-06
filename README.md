# Anchor Vault (q3_26_vault)

A small Solana program built with Anchor. Each user gets their own SOL vault,
a PDA owned by the System Program, that only they can deposit into and
withdraw from.

Built as a Turbin3 Q3 2026 starter exercise.

## How it works

Two PDAs are derived per user:

| Account | Seeds | Purpose |
|---|---|---|
| `vault_state` | `["state", user]` | Stores the two bumps (`vault_bump`, `state_bump`) |
| `vault` | `["vault", user]` | Holds the SOL (system account, no data) |

Because the vault is owned by the System Program, the program cannot move SOL
out of it directly. It signs the transfer as the PDA with
`CpiContext::new_with_signer` and the seeds `["vault", user, vault_bump]`.

## Instructions

| Instruction | What it does |
|---|---|
| `initialize` | Creates `vault_state` and funds the vault with the rent-exempt minimum |
| `deposit(amount)` | Transfers `amount` lamports from the user to the vault |
| `withdraw(amount)` | Transfers `amount` lamports from the vault to the user. Fails if it would leave the vault below rent-exempt (`InsufficientFunds`) or if `amount` is 0 (`InvalidAmount`) |
| `close` | Sends the whole vault balance back to the user and closes `vault_state` |

Flow: `initialize -> deposit -> withdraw -> close`

## Project layout

```
programs/q3_26_vault/
  src/
    lib.rs                  program entrypoints
    constants.rs            PDA seeds
    state.rs                VaultState account
    error.rs                custom errors
    instructions/
      initialize.rs
      deposit.rs
      withdraw.rs
      close.rs
  tests/test_initialize.rs  LiteSVM test of the full flow
```

## Requirements

- Rust 1.89.0 (pinned in `rust-toolchain.toml`)
- Solana CLI 4.x
- Anchor CLI 1.1.x

## Build and test

```bash
anchor keys sync                 # make declare_id! match your local keypair
anchor build -- --arch v0        # builds target/deploy/q3_26_vault.so
cargo test
```

The test loads `target/deploy/q3_26_vault.so` into LiteSVM, so the program
must be built before running `cargo test`.

Notes from my setup (Solana CLI 4.3.0):

- `anchor build` failed with "Platform tools v1.52 is incompatible with
  SBPFv3". Passing `--arch v0` fixed the program build.
- With `-- --arch v0`, the IDL step then errors because `cargo test` does not
  accept `--arch`. The `.so` is still produced, which is all the test needs.
  Build the IDL separately with `anchor idl build`.

## Devnet

The program is deployed on devnet and its IDL is uploaded:

- Program ID: `5aGDPMPvqemfAAXwFwsvs1FmkX9XiWKD5ZyrSLkcmf49`
- [View on Solana Explorer](https://explorer.solana.com/address/5aGDPMPvqemfAAXwFwsvs1FmkX9XiWKD5ZyrSLkcmf49?cluster=devnet)

Deploy your own copy:

```bash
solana config set --url devnet
solana program deploy target/deploy/q3_26_vault.so \
  --program-id target/deploy/q3_26_vault-keypair.json
anchor idl build -o target/idl/q3_26_vault.json
anchor idl init <PROGRAM_ID> -f target/idl/q3_26_vault.json --provider.cluster devnet
```

Your own deploy gets a different program ID from your own keypair, so run
`anchor keys sync` first.

## Credits

Starter project by the Turbin3 cohort mentors. `withdraw` and `close` were
implemented as the exercise.