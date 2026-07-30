---
name: linqgg-run-tournament-economy
description: Debit tournament entry fees and credit prizes against a player's LinQ game account without double-spending, then reconcile the ledger.
api: LinQ Wallet Public API
generated: '2026-07-19'
method: generated
source: https://docs.linq.gg/modules/money
operations:
  - linq.money.accounts.v1.AccountsService.GetActualBalance
  - linq.money.accounts.v1.AccountsService.GetAllAccounts
  - linq.money.accounts.v2.AccountsService.GetActualBalances
  - linq.money.accounts.v1.AccountsService.GetMoney
  - linq.money.accounts.v1.AccountsService.PutMoney
  - linq.money.accounts.v2.AccountsService.ApplyCustomReward
  - linq.money.operations.v1.OperationsService.GetOrdersHistory
---

# Run a tournament economy on LinQ

This is the in-game money loop: take an entry fee, run the tournament, pay the
prize, then reconcile. Every step is idempotent by construction, and it must be —
these calls move real money.

## Read the balance first

`AccountsService.GetActualBalance` takes a `currency` and returns `number`,
`currency` and `balance`. Omit `currency` and it defaults to `LNQ`, the wallet
account.

A player holds a wallet account (`LNQ`) plus one account per game. To avoid two
round trips, fetch everything at once and filter client-side:

```
const accounts = await accountsService.getAllAccounts({}, getAuthorization(authToken));
const walletBalance = accounts.response.accounts.filter(v => v.currency == "LNQ").pop()?.balance || 0;
const gamingBalance = accounts.response.accounts.filter(v => v.currency == "GSC").pop()?.balance || 0;
```

Balances for other games are never returned — that is deliberate, not a bug.

`balance` is an integer in coins. Divide by 100 for display: `500` is `$5.00`.

If you need the available-versus-current split, use
`linq.money.accounts.v2.AccountsService.GetActualBalances`, which returns
`Balance { current, available }` per account. Note that v2 does **not** carry
`PutMoney` / `GetMoney` — those live only in v1, and v1 is not deprecated. Both
versions are in active use.

## Take the entry fee

`AccountsService.GetMoney` withdraws from the player's game account.

```
await accountsService.getMoney({
  idempotencyKey: 'ik-124',
  amount: 500,
  reason: 'Tournament entry fee',
  extra: { tournament_id: 't1' },
}, getAuthorization(authToken));
```

- `idempotency_key` is **required** and is a message field, not a header. Derive
  it deterministically from something stable — for example
  `entry:{tournament_id}:{game_user_id}` — so a retry after a timeout cannot
  charge the player twice.
- `amount` is in coins, excluding fee. Set `fee` separately if one applies.
- `reason` is free text shown in reconciliation.
- `extra` is a JSON string for correlation data.
- `reference` is your own external id; it is echoed on every order response and
  history entry.

The response is a `MoneyResponse` with the order `id`, `status`, `amount`, the
resulting `account`, and `reference`. Treat `status: completed` as final.

## Pay the prize

`AccountsService.PutMoney` deposits, with the identical message shape:

```
await accountsService.putMoney({
  idempotencyKey: 'ik-123',
  amount: 500,
  reason: 'Tournament reward',
  extra: { tournament_id: 't1' },
}, getAuthorization(authToken));
```

Use a prize-specific idempotency key such as `prize:{tournament_id}:{game_user_id}`.
Paying out is the operation where a duplicate is most expensive — never reuse the
entry-fee key and never generate a random key per attempt.

For rewards outside the tournament loop (welcome bonuses, goodwill credits) use
`linq.money.accounts.v2.AccountsService.ApplyCustomReward` with `amount`,
a `reason` that "has to be explained", and an optional `reference`. It works only
if enabled for your game, and it returns an `AppliedRewardOrder`.

## Reconcile

`OperationsService.GetOrdersHistory` returns every order the player created, for
your side of the ledger or for a player-facing transaction list.

Filter with `types` (`charge`, `entry_fee`, `prize`, `transfer`), `statuses`
(`pending`, `awaiting`, `processing`, `accepted`, `declined`, `completed`,
`cancelled`, `terminated`), and `start_date` / `end_date` timestamps. The
singular `type` and `status` fields are the one-value forms of the same filters.
Ignore `assets` — the contract documents it as "do not use, has no effect".

Paginate with a cursor: send `limit`, read `next_id` from the response, send it
back as `next_id` on the next call, and stop when it is absent. `total` gives the
overall count.

Each `HistoryEntry` carries `is_incoming`, which encodes direction: `charge` and
`prize` are true, `entry_fee` is false, and a `transfer` is true when it lands in
the current app and false when it leaves. `transfer_source_app` and
`transfer_destination_app` name the other side.

**Reconciliation key:** when you did not set `reference`, `HistoryEntry.reference`
falls back to returning the `idempotency_key`. Deterministic idempotency keys
therefore double as your join key between the LinQ ledger and your own — another
reason not to randomize them.

## Rules that keep this safe

- One deterministic idempotency key per logical money movement. Reuse it on
  every retry of that movement; never share it across movements.
- Coins, not currency units. Multiply by 100 on the way in, divide on the way out.
- Anonymous players cannot transfer out and are capped at a total deposit of
  20.00. Entry fees and prizes still work for them.
- There are no webhooks. Anything asynchronous is discovered by polling.
- Errors are native gRPC statuses with no structured detail payload; failures
  surface as exceptions or as a terminal order status.
