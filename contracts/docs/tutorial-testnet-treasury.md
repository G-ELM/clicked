# Tutorial: create and run a group treasury on testnet

You will deploy the `group_treasury` contract to Stellar testnet, add three members with an approval threshold of two, deposit a test asset, propose a withdrawal, approve it, and see what happens when you do the wrong thing. You will finish by pointing the app at the deployed contract.

Every command and output below was run on testnet on 2026-10-20 with `stellar 25.2.0`, `cargo 1.94.0` and `soroban-sdk 22.0.0`. Addresses, contract IDs and transaction hashes will differ in your run. Long transaction URLs and event payloads are shortened with `…`.

## Read this first: what differs from a clean run

Running this tutorial against the repository as it stands surfaced three problems in `scripts/deploy_group_treasury.sh` and one gap in the contract. Each is handled in the step where it appears, and each is a fix worth making in the repo:

| #   | Where                      | Problem                                                                                                                                                                                  |
| --- | -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | script, build step         | `cargo build --target wasm32-unknown-unknown` with a current Rust toolchain emits WebAssembly that testnet rejects (`reference-types not enabled`). Build with `stellar contract build`. |
| 2   | script, initialise step    | `initialize` takes a required `--threshold`; the script does not pass it, so it fails with `Missing required argument 'threshold'`.                                                      |
| 3   | script, admin address step | `stellar keys public-key --secret-key …` is not accepted by `stellar 25.2.0` (`unexpected argument '--secret-key'`).                                                                     |
| 4   | contract                   | There is **no execute entrypoint**. Reaching the threshold marks a proposal `Passed`; nothing moves funds. See [Step 8](#step-8-see-what-passed-does-and-does-not-do).                   |

The tutorial therefore runs the script's own steps by hand (upload, deploy, initialize) with the corrections above. Once the script is fixed, steps 3 to 5 collapse into one script call.

## Prerequisites

- `stellar` CLI 25 or newer: `stellar --version`
- Rust with the `wasm32v1-none` target: `rustup target add wasm32v1-none`
- A checkout of this repository, with commands run from `contracts/`

```bash
cd contracts
```

## Step 1: create and fund four testnet identities

You need an admin (deployer) and three members. `--fund` asks the testnet friendbot to give each account test XLM.

```bash
for n in tut-admin tut-alice tut-bob tut-carol; do
  stellar keys generate $n --network testnet --fund
done
```

```
✅ Key saved with alias tut-admin in "…/stellar/identity/tut-admin.toml"
✅ Account tut-admin funded on "Test SDF Network ; September 2015"
… (same two lines for tut-alice, tut-bob, tut-carol)
```

Save the addresses in shell variables:

```bash
ADMIN=$(stellar keys address tut-admin)
ALICE=$(stellar keys address tut-alice)
BOB=$(stellar keys address tut-bob)
CAROL=$(stellar keys address tut-carol)
echo $ALICE
```

```
GAEH7OD6EIE6OJNM3W3PCXETE6U4CWNW7MATSCHB7BCLMQY7QN6BB3LL
```

These are throwaway testnet keys. Never reuse a key you have funded on mainnet.

## Step 2: get a test token contract address

The treasury holds any SEP-41 token. The easiest on testnet is the built-in contract for the native asset (XLM), which your funded accounts already hold:

```bash
TOKEN=$(stellar contract id asset --asset native --network testnet)
echo $TOKEN
```

```
CDLZFC3SYJYDZT7K67VZ75HPJVIEUVNIXF47ZG2FB2RMQQVU2HHGCYSC
```

That ID is the same for everyone on testnet. Amounts are in stroops: 1 XLM = 10,000,000 stroops.

If you would rather use your own token, issue a classic asset and wrap it with `stellar contract asset deploy --asset CODE:ISSUER --source-account tut-admin --network testnet`; the command prints the contract ID.

## Step 3: build the contract

The script's own build line (problem 1) produces a file testnet refuses, so build with the CLI:

```bash
stellar contract build --package group_treasury
```

```
…
   • reject_withdraw
   • remove_member
   • withdraw
✅ Build Complete
```

The output is `target/wasm32v1-none/release/group_treasury.wasm`.

To see problem 1 for yourself, run the script with the identity's secret:

```bash
DEPLOYER_SECRET=$(stellar keys secret tut-admin) TOKEN_CONTRACT_ID=$TOKEN ./scripts/deploy_group_treasury.sh
```

```
==> Uploading WASM to testnet...
ℹ️  Simulating transaction…
❌ error: transaction simulation failed: HostError: Error(WasmVm, InvalidAction)
…"reference-types not enabled: zero byte expected"…
```

## Step 4: upload and deploy

These are the script's `upload` and `deploy` calls, pointed at the correct file.

```bash
WASM_HASH=$(stellar contract upload --network testnet --source tut-admin \
  --wasm target/wasm32v1-none/release/group_treasury.wasm)
echo $WASM_HASH
```

```
✅ Transaction submitted successfully!
🔗 https://stellar.expert/explorer/testnet/tx/8f4e4f91…
f9c5809d38f2ce24edb1e31d0be801535bf5c7cb46645f13aa7eb3950affcb1b
```

(The hash is the last line; the command prints the status lines to stderr.)

```bash
CID=$(stellar contract deploy --network testnet --source tut-admin --wasm-hash $WASM_HASH)
echo $CID
```

```
CDLFIQKSJLWHQPCQPATYYLUE67ZWM3CCK2V6ZBRHEIFSFZH56PK7LSEK
```

Define a short helper so the rest reads cleanly:

```bash
inv() { local src=$1; shift; stellar contract invoke --network testnet --source $src --id $CID -- "$@"; }
```

## Step 5: initialize with a threshold of two

First, the script's `initialize` call (problem 2):

```bash
inv tut-admin initialize --admin $ADMIN --token $TOKEN
```

```
❌ error: Missing required argument 'threshold' of type u32 (unsigned 32-bit integer)
```

Add the threshold:

```bash
inv tut-admin initialize --admin $ADMIN --token $TOKEN --threshold 2
inv tut-admin get_threshold
```

```
✅ Transaction submitted successfully!
2
```

`initialize` does not require the admin's signature, so deploy and initialize in the same sitting; anyone could initialize an uninitialized contract. See [api-group-treasury.md](./api-group-treasury.md#initialize).

## Step 6: add three members

Only the admin can add members. Note that the admin is not a member and cannot vote.

```bash
for m in $ALICE $BOB $CAROL; do inv tut-admin add_member --member $m; done
inv tut-admin get_members
```

```
✅ Transaction submitted successfully!
📅 CDLF…SEK - Success - Event: [{"symbol":"member_added"}] = {"map":[{"key":{"symbol":"added_by"},…
… (once per member)
["GAEH7OD6…QN6BB3LL","GBN4LAAQ…KMDLEVWQS","GBUUCHFH…GKOBXTMBW"]
```

## Step 7: deposit, propose, approve

Alice deposits 100 XLM (1,000,000,000 stroops). Depositing does not require membership.

```bash
inv tut-alice deposit --from $ALICE --token $TOKEN --amount 1000000000
inv tut-alice balance --token $TOKEN
```

```
✅ Transaction submitted successfully!
📅 CDLZ…YSC - Success - Event: [{"symbol":"transfer"},…] = {"i128":"1000000000"}
📅 CDLF…SEK - Success - Event: [{"symbol":"deposit"}] = {"map":[{"key":{"symbol":"amount"},"val":{"i128":"1000000000"}},…
"1000000000"
```

Alice proposes sending 10 XLM to the admin account. `ttl_ledgers` is how long the proposal stays open (about 5 seconds per ledger, so 17280 is about a day).

```bash
inv tut-alice propose_withdraw --proposer $ALICE --to $ADMIN --token $TOKEN \
  --amount 100000000 --ttl_ledgers 17280
inv tut-alice get_proposal --proposal_id 0
```

```
✅ Transaction submitted successfully!
📅 CDLF…SEK - Success - Event: [{"symbol":"proposal_created"}] = {…"id":{"u32":0}…}
0
{"amount":"100000000","approvals":1,"expires_at":1792581837,"id":0,"proposer":"GAEH…","rejections":0,"status":"Active","to":"GATF…","token":"CDLZ…"}
```

The new proposal has `approvals: 1`. **The proposer's vote is counted automatically**, so with a threshold of two, one more member is enough, not two. This is the main thing to understand about the "approve it twice" step in the original design: Alice's proposal already counts as her approval.

Bob approves:

```bash
inv tut-bob approve_withdraw --approver $BOB --proposal_id 0
inv tut-bob get_proposal --proposal_id 0
```

```
✅ Transaction submitted successfully!
📅 CDLF…SEK - Success - Event: [{"symbol":"proposal_approved"}] = {"map":[{"key":{"symbol":"approvals"},"val":{"u32":2}},{"key":{"symbol":"id"},"val":{"u32":0}},{"key":{"symbol":"threshold"},"val":{"u32":2}}]}
📅 CDLF…SEK - Success - Event: [{"symbol":"withdraw_vote"}] = {…"approve":{"bool":true}…}
{"amount":"100000000","approvals":2,"expires_at":1792581837,"id":0,…,"status":"Passed",…}
```

`proposal_approved` fired and the status is `Passed`.

## Step 8: see what `Passed` does and does not do

Check the treasury balance and the pending list:

```bash
inv tut-bob balance --token $TOKEN
inv tut-bob get_pending_proposals
```

```
"1000000000"
[]
```

The balance is unchanged: 100 XLM, not 90. `group_treasury` has no function that executes a passed proposal (the `execute` tests in `group_treasury/src/test.rs` are `#[ignore]`d for this reason). Two ways funds actually leave today:

1. **Admin `withdraw`.** The admin can send any amount to any address, regardless of proposals or votes:

   ```bash
   inv tut-admin withdraw --to $ADMIN --token $TOKEN --amount 100000000
   inv tut-admin balance --token $TOKEN
   ```

   ```
   ✅ Transaction submitted successfully!
   📅 CDLZ…YSC - Success - Event: [{"symbol":"transfer"},…] = {"i128":"100000000"}
   📅 CDLF…SEK - Success - Event: [{"symbol":"withdraw"}] = {"map":[{"key":{"symbol":"amount"},…
   "900000000"
   ```

2. **The `proposals` contract.** Its `execute_withdraw` calls the treasury's `withdraw`, which requires the treasury admin's authorization, and it runs its own create, vote and finalize flow. That is a separate tutorial (see [concepts-proposal-lifecycle.md](./concepts-proposal-lifecycle.md) and `scripts/deploy_proposals.sh`).

The honest summary: the three-member vote records consent on-chain, but it is the admin key that moves the money. Read [concepts-treasury-multisig-model.md](./concepts-treasury-multisig-model.md) before trusting a treasury with real funds.

## Step 9: failure cases to try

These are the failures a learner should see once. Release builds strip panic text, so the CLI reports `UnreachableCodeReached` and names the function; the table below says which check fired, from [contracts-errors.md](./contracts-errors.md). The workspace also defines a `release-with-logs` profile for debugging; this tutorial did not use it.

### Approving twice from the same member

Alice proposed, so she has already voted:

```bash
inv tut-alice approve_withdraw --approver $ALICE --proposal_id 0
```

```
❌ error: transaction simulation failed: HostError: Error(WasmVm, InvalidAction)
Event log (newest first):
   0: [Diagnostic Event] contract:CDLF…SEK, topics:[error, Error(WasmVm, InvalidAction)], data:["VM call trapped: UnreachableCodeReached", approve_withdraw]
```

On a proposal that is still `Active`, the check that fires is `"already voted"`. Here the proposal is already `Passed` (Step 7), and the status check runs first, so it is `"proposal is not pending"`. To see `"already voted"` itself, create a second proposal (Alice proposes, then Alice approves) before anyone else votes.

### Voting after the threshold is reached

Carol arrives late:

```bash
inv tut-carol approve_withdraw --approver $CAROL --proposal_id 0
```

```
❌ error: transaction simulation failed: HostError: Error(WasmVm, InvalidAction)
   0: … data:["VM call trapped: UnreachableCodeReached", approve_withdraw]
```

Same trap, check `"proposal is not pending"`: a `Passed` proposal accepts no further votes.

### "Executing" below the threshold

There is nothing to call, because there is no execute function (Step 8). The closest real experiment is to try the admin path with a non-admin key:

```bash
inv tut-alice withdraw --to $ALICE --token $TOKEN --amount 100000000
```

```
❌ error: Missing signing key for account GATFU5UCUXUCYT6W6SJUA3IWDQZVZRN3ATSE2EHAAS2FD2JT6DHKR5MC
```

The CLI refuses before submitting, because `withdraw` needs the admin's signature and `tut-alice` is not the admin.

### Proposing more than the treasury holds

```bash
inv tut-alice propose_withdraw --proposer $ALICE --to $ADMIN --token $TOKEN \
  --amount 99999999999 --ttl_ledgers 17280
```

```
❌ error: transaction simulation failed: HostError: Error(WasmVm, InvalidAction)
   0: … data:["VM call trapped: UnreachableCodeReached", propose_withdraw]
```

The check is `"insufficient funds"`.

## Step 10: point the app at the deployed contract

The treasury contract ID is read by the **backend**, not the web app:

```bash
# apps/backend/.env
GROUP_TREASURY_CONTRACT_ID=CDLFIQKSJLWHQPCQPATYYLUE67ZWM3CCK2V6ZBRHEIFSFZH56PK7LSEK
STELLAR_RPC_URL=https://soroban-testnet.stellar.org
```

With both set, the Stellar listener watches the contract and syncs its events into the database (`apps/backend/src/index.ts`). Without `GROUP_TREASURY_CONTRACT_ID`, the treasury route reports contract `stub`.

For the web app, set the network variables to match testnet (they are the defaults, so this is only needed if you changed them). All `NEXT_PUBLIC_*` values are compiled into the bundle and visible to every visitor; never put a secret in one:

```bash
# apps/web/.env.local
NEXT_PUBLIC_SOROBAN_RPC_URL=https://soroban-testnet.stellar.org
NEXT_PUBLIC_NETWORK_PASSPHRASE="Test SDF Network ; September 2015"
NEXT_PUBLIC_NETWORK=test
```

**There is currently no `NEXT_PUBLIC_*` variable for the group treasury contract.** The only contract ID the web app reads is `NEXT_PUBLIC_TOKEN_TRANSFER_CONTRACT` (`apps/web/src/lib/soroban.ts`), which is for the `token_transfer` contract, not this one. If a frontend variable for the treasury is added (the original request referenced #373), update this step. The full variable list and defaults are in [environment-variables.md](../../docs/environment-variables.md).

Restart the backend and web dev servers after changing env files.

## Clean up

Testnet state is free to leave behind. To remove the local identities:

```bash
for n in tut-admin tut-alice tut-bob tut-carol; do stellar keys rm $n; done
```
